# 引擎 Runtime 抽象层分析

> 本文档为系列分析文档的**第二篇**，深入分析 LobsterAI 与引擎之间的抽象解耦层。
> 前置文档：[架构总览](./architecture-overview.md)
> 配套文档：[存储层分析](./storage-layer.md)

---

## 一、设计目标

两个引擎（OpenClaw Gateway 和内置 Claude Agent SDK）需要能被**无缝替换**，同时渲染进程（UI 层）对引擎选择完全无感知。

实现路径：**接口契约 + 门面路由 + 事件多路复用**

---

## 二、核心接口：`CoworkRuntime`

**文件**: `src/main/libs/agentEngine/types.ts`

```typescript
export type CoworkAgentEngine = 'openclaw' | 'yd_cowork';

export interface CoworkRuntimeEvents {
  message: (sessionId: string, message: CoworkMessage) => void;
  messageUpdate: (sessionId: string, messageId: string, content: string) => void;
  permissionRequest: (sessionId: string, request: PermissionRequest) => void;
  complete: (sessionId: string, claudeSessionId: string | null) => void;
  error: (sessionId: string, error: string) => void;
  sessionStopped: (sessionId: string) => void;
}

export interface CoworkRuntime {
  // EventEmitter 继承
  on<U extends keyof CoworkRuntimeEvents>(event: U, listener: CoworkRuntimeEvents[U]): this;
  off<U extends keyof CoworkRuntimeEvents>(event: U, listener: CoworkRuntimeEvents[U]): this;

  // 核心操作
  startSession(sessionId: string, prompt: string, options?: CoworkStartOptions): Promise<void>;
  continueSession(sessionId: string, prompt: string, options?: CoworkContinueOptions): Promise<void>;
  stopSession(sessionId: string): void;
  stopAllSessions(): void;
  respondToPermission(requestId: string, result: PermissionResult): void;
  isSessionActive(sessionId: string): boolean;
  getSessionConfirmationMode(sessionId: string): 'modal' | 'text' | null;
  onSessionDeleted?(sessionId: string): void;
}
```

**设计要点**:
- 所有事件通过 `EventEmitter` 接口，主进程只监听一个对象即可接收任意引擎的事件
- `respondToPermission` 是实现精确路由的关键：permission request 有 `requestId`，可唯一追踪
- `CoworkRuntimeEvents` 中每个事件第一个参数都是 `sessionId`，方便路由层建立 `session → engine` 映射

---

## 三、路由层：`CoworkEngineRouter`

**文件**: `src/main/libs/agentEngine/coworkEngineRouter.ts`

### 3.1 类结构

```typescript
export class CoworkEngineRouter extends EventEmitter implements CoworkRuntime {
  private readonly runtimeByEngine: Record<CoworkAgentEngine, CoworkRuntime>;
  private readonly sessionEngine    = new Map<string, CoworkAgentEngine>(); // session → 引擎
  private readonly requestEngine    = new Map<string, CoworkAgentEngine>(); // requestId → 引擎
  private readonly requestSession   = new Map<string, string>();             // requestId → sessionId
  private currentEngine: CoworkAgentEngine;
}
```

### 3.2 三层 Map 的精确路由

| Map | 键 | 值 | 用途 |
|-----|----|----|------|
| `sessionEngine` | `sessionId` | `engine` | 知道某个会话属于哪个引擎 |
| `requestEngine` | `requestId` | `engine` | 知道某个权限请求属于哪个引擎 |
| `requestSession` | `requestId` | `sessionId` | 知道权限请求对应的会话 |

### 3.3 事件绑定——透明多路复用

```typescript
private bindRuntimeEvents(engine: CoworkAgentEngine, runtime: CoworkRuntime): void {
  // 每次收到 message，自动标记来源引擎
  runtime.on('message', (sessionId, message) => {
    this.sessionEngine.set(sessionId, engine); // 建立映射
    this.emit('message', sessionId, message);  // 向上转发
  });

  // permissionRequest 同时建立 requestId → engine 映射
  runtime.on('permissionRequest', (sessionId, request) => {
    this.sessionEngine.set(sessionId, engine);
    this.requestEngine.set(request.requestId, engine);  // 精确路由的关键
    this.requestSession.set(request.requestId, sessionId);
    this.emit('permissionRequest', sessionId, request);
  });

  // complete / error 时清理映射
  runtime.on('complete', (sessionId) => {
    this.sessionEngine.delete(sessionId);
    this.clearRequestEngineBySession(sessionId);
    this.emit('complete', sessionId);
  });
}
```

### 3.4 权限响应精确路由

```typescript
respondToPermission(requestId: string, result: PermissionResult): void {
  const engine = this.requestEngine.get(requestId);
  if (engine) {
    // 精确路由：直接发给来源引擎
    this.runtimeByEngine[engine].respondToPermission(requestId, result);
    if (result.behavior === 'allow' || result.behavior === 'deny') {
      this.requestEngine.delete(requestId);
      this.requestSession.delete(requestId);
    }
    return;
  }
  // 降级：两个引擎都发（适用于并发场景）
  this.runtimeByEngine.openclaw.respondToPermission(requestId, result);
  this.runtimeByEngine.yd_cowork.respondToPermission(requestId, result);
}
```

### 3.5 引擎切换处理

```typescript
handleEngineConfigChanged(nextEngine: CoworkAgentEngine): void {
  if (nextEngine === this.currentEngine) return;
  this.currentEngine = nextEngine;
  this.stopAllSessions(); // 停止旧引擎所有会话
  activeSessionIds.forEach((sessionId) => {
    this.emit('error', sessionId, ENGINE_SWITCHED_CODE); // 通知 UI 切换提示
  });
}
```

---

## 四、OpenClaw 协议适配器：`OpenClawRuntimeAdapter`

**文件**: `src/main/libs/agentEngine/openclawRuntimeAdapter.ts`

这是整个抽象层中最复杂的组件，负责把 OpenClaw Gateway 的 WebSocket 协议转换为 `CoworkRuntime` 事件。

### 4.1 核心数据结构

```typescript
export class OpenClawRuntimeAdapter extends EventEmitter implements CoworkRuntime {
  private gatewayClient: GatewayClientLike | null = null;   // WebSocket 客户端
  private gatewayReadyPromise: Promise<void> | null = null; // 并发初始化锁
  private activeTurns = new Map<string, ActiveTurn>();       // sessionId → Turn 状态
}
```

**`GatewayClientLike`** — 抽象 WebSocket 客户端接口：

```typescript
type GatewayClientLike = {
  start: () => void;
  stop: () => void;
  request: <T>(method: string, params?: unknown, opts?: { expectFinal?: boolean }) => Promise<T>;
};
```

**`ActiveTurn`** — 单个会话的并发状态机：

```typescript
type ActiveTurn = {
  sessionId: string;
  sessionKey: string;           // OpenClaw 内部 session key
  runId: string;
  assistantMessageId: string | null;  // 当前 assistant 消息 ID
  committedAssistantText: string;     // 工具调用前已提交的文本
  currentText: string;                  // 当前流式文本
  currentContentText: string;           // 当前内容
  currentContentBlocks: string[];      // 当前内容块
  toolUseMessageIdByToolCallId: Map<string, string>;  // tool_call_id → 消息 ID
  toolResultMessageIdByToolCallId: Map<string, string>;
  stopRequested: boolean;
  pendingUserSync: boolean;      // 渠道会话预取锁
  bufferedChatPayloads: BufferedChatEvent[];   // 缓冲（预取锁期间）
  bufferedAgentPayloads: BufferedAgentEvent[];
  timeoutTimer?: ReturnType<typeof setTimeout>; // 客户端超时看门狗
};
```

### 4.2 `handleGatewayEvent` — 网关事件分发

```typescript
private handleGatewayEvent(event: GatewayEventFrame): void {
  if (event.event === 'tick') {
    this.lastTickTimestamp = Date.now();  // 心跳，用于存活检测
    return;
  }

  if (event.event === 'chat') {
    this.handleChatEvent(event.payload as ChatEventPayload, event.seq);
  }

  if (event.event === 'agent') {
    // 优先处理 assistant 文本流（因可能因 session 未就绪而入队）
    this.processAgentAssistantText(event.payload as AgentEventPayload);
    this.handleAgentEvent(event.payload as AgentEventPayload, event.seq);
  }

  if (event.event === 'exec.approval.requested') {
    this.handleApprovalRequested(event.payload as ExecApprovalRequestedPayload);
  }

  if (event.event === 'exec.approval.resolved') {
    this.handleApprovalResolved(event.payload as ExecApprovalResolvedPayload);
  }
}
```

### 4.3 `handleChatEvent` — 内容流处理

`chat` 事件有四种状态（`delta` / `final` / `aborted` / `error`）：

```typescript
private handleChatEvent(payload: ChatEventPayload, seq?: number): void {
  const turn = this.activeTurns.get(payload.sessionKey ?? '');
  if (!turn) return; // 可能已被 stopSession 清理

  switch (payload.state) {
    case 'delta':
      // 流式增量文本
      this.processChatDelta(turn, payload);
      break;
    case 'final':
      // 最终确认，触发 completion
      this.finalizeTurn(turn, payload);
      break;
    case 'aborted':
      this.abortTurn(turn);
      break;
    case 'error':
      this.emitTurnError(turn, payload.errorMessage ?? 'unknown');
      break;
  }
}
```

### 4.4 流式文本合并策略

OpenClaw 同时发送 `chat.delta`（内容流）和 `agent.assistant`（模型原始文本）两种事件。两者都是**全量替换式**的，适配器实现智能合并：

```typescript
const mergeStreamingText = (
  previousText: string,
  incomingText: string,
  mode: TextStreamMode,
): { text: string; mode: TextStreamMode } => {
  if (mode === 'delta') {
    // 检测是否只是后缀追加
    if (incomingText.startsWith(previousText))
      return { text: incomingText, mode: 'snapshot' };

    // 检测前后缀重叠实现增量追加（处理边界情况）
    const overlap = computeSuffixPrefixOverlap(previousText, incomingText);
    return { text: previousText + incomingText.slice(overlap), mode };
  }
  // snapshot 模式：直接替换
  return { text: incomingText, mode };
};
```

### 4.5 工具事件三阶段处理

```typescript
private handleAgentToolEvent(turn: ActiveTurn, data: unknown): void {
  const { phase, toolCallId, text } = parseToolEvent(data);

  if (phase === 'start') {
    // 阶段 1: 添加 tool_use 消息
    const msg = this.store.addMessage(turn.sessionId, {
      type: 'tool_use',
      toolName,
      toolInput,
      toolCallId,
    });
    turn.toolUseMessageIdByToolCallId.set(toolCallId, msg.id);
    this.emit('message', turn.sessionId, msg);
  }

  if (phase === 'update') {
    // 阶段 2: 流式更新（增量合并）
    const msgId = turn.toolUseMessageIdByToolCallId.get(toolCallId);
    const currentText = turn.toolResultTextByToolCallId.get(toolCallId) ?? '';
    const merged = mergeStreamingText(currentText, text, mode);
    turn.toolResultTextByToolCallId.set(toolCallId, merged.text);
    this.emit('messageUpdate', turn.sessionId, msgId!, merged.text);
  }

  if (phase === 'result') {
    // 阶段 3: 标记最终状态
    turn.toolResultMessageIdByToolCallId.set(toolCallId, resultMessageId);
  }
}
```

### 4.6 权限请求映射

OpenClaw `exec.approval.requested` → `CoworkRuntimeEvents.permissionRequest`：

```typescript
private handleApprovalRequested(payload: ExecApprovalRequestedPayload): void {
  const request = payload.request!;
  const pending: PendingApprovalEntry = {
    requestId: payload.id!,        // 使用 OpenClaw 的 approval ID
    sessionId: this.resolveSessionIdBySessionKey(request.sessionKey),
    allowAlways: false,
  };

  const dangerLevel = getCommandDangerLevel(request.command ?? '');
  if (dangerLevel === 'high') {
    pending.allowAlways = false; // 高危命令必须每次确认
  }

  this.pendingApprovals.set(pending.requestId, pending);
  this.emit('permissionRequest', pending.sessionId, {
    requestId: pending.requestId,
    toolName: 'exec',
    toolInput: {
      command: request.command,
      cwd: request.cwd,
    },
  });
}
```

### 4.7 重连与心跳机制

```typescript
const GATEWAY_RECONNECT_DELAYS    = [2_000, 5_000, 10_000, 15_000, 30_000]; // 指数退避
const GATEWAY_RECONNECT_MAX_ATTEMPTS = 10;
const TICK_WATCHDOG_INTERVAL_MS   = 60_000; // 心跳检查间隔
const TICK_TIMEOUT_MS              = 90_000; // 3 个 tick 无响应 → 连接死亡
```

**Tick 心跳 watchdog**:

```typescript
private startTickWatchdog(): void {
  this.tickWatchdogTimer = setInterval(() => {
    const elapsed = Date.now() - this.lastTickTimestamp;
    if (elapsed > TICK_TIMEOUT_MS) {
      // OpenClaw 未主动关闭但实际已死亡 → 强制重连
      void this.attemptGatewayReconnect();
    }
  }, TICK_WATCHDOG_INTERVAL_MS);
}
```

### 4.8 懒初始化与版本校验

```typescript
private async ensureGatewayClientReady(): Promise<void> {
  if (this.gatewayReadyPromise) return this.gatewayReadyPromise;

  this.gatewayReadyPromise = this.initGatewayClient();

  // 版本变化时重建客户端
  const needsNewClient = this.lastKnownGatewayVersion !== connectionInfo.version;
  if (needsNewClient) {
    this.disposeGatewayClient();
    this.gatewayReadyPromise = this.initGatewayClient();
  }

  await this.gatewayReadyPromise;
}
```

### 4.9 节流保护

```typescript
const MESSAGE_UPDATE_THROTTLE_MS = 200;  // IPC 发射节流
const STORE_UPDATE_THROTTLE_MS   = 250;  // SQLite 写入节流

// 使用 lodash throttle 或手写：leading + trailing 模式
private throttledEmitMessageUpdate = throttle(
  (sessionId: string, messageId: string, content: string) => {
    this.emit('messageUpdate', sessionId, messageId, content);
  },
  MESSAGE_UPDATE_THROTTLE_MS,
  { leading: true, trailing: true }
);
```

---

## 五、OpenClaw 进程管理：`OpenClawEngineManager`

**文件**: `src/main/libs/openclawEngineManager.ts`

### 5.1 状态机

```
┌─────────────┐   安装完成    ┌───────────┐
│not_installed│────────────→ │  ready    │
└─────────────┘              └─────┬─────┘
                                    │ startGateway()
┌─────────────┐   startGateway()    ▼
│   error    │←──────────────────────────┐  ┌───────────┐
└─────────────┘                          │  │ starting  │
    ↑            stopGateway()           │  └─────┬─────┘
    │                ┌──────────────────┘        │
    │                ↓                           ↓ isGatewayHealthy()
    └───────────────  ←←←←←←←←←←←←←←←←←←←←←  ┌───────────┐
                                             │  running  │
                                             └───────────┘
```

### 5.2 平台差异：进程启动方式

| 平台 | 方式 | 原因 |
|------|------|------|
| macOS / Linux | `utilityProcess.fork()` | Electron 内置进程管理，安全的沙箱隔离 |
| Windows | `child_process.spawn()` + `ELECTRON_RUN_AS_NODE=1` | Benchmark: `utilityProcess` 在 Windows 上冷启动 ESM 有 ~5× 开销（163s vs 34s）|

### 5.3 关键环境变量注入

```typescript
const env = {
  // OpenClaw 运行时路径
  OPENCLAW_HOME: runtimeRoot,
  OPENCLAW_STATE_DIR: this.stateDir,
  OPENCLAW_CONFIG_PATH: this.configPath,
  OPENCLAW_GATEWAY_TOKEN: token,
  OPENCLAW_GATEWAY_PORT: String(port),
  OPENCLAW_NO_RESPAWN: '1',          // 禁止 OpenClaw 自我重启（由 LobsterAI 管理）

  // Node 编译缓存
  NODE_COMPILE_CACHE: compileCacheDir,

  // LobsterAI 信息
  LOBSTERAI_ELECTRON_PATH: electronNodeRuntimePath,
  LOBSTERAI_OPENCLAW_ENTRY: openclawEntry,

  // 技能路径（OpenClaw 和 LobsterAI 各一份）
  SKILLS_ROOT: skillsRoot,
  LOBSTERAI_SKILLS_ROOT: skillsRoot,

  // 秘密变量：${VAR} 占位符的实际值（不写入磁盘）
  ...this.secretEnvVars,

  // 系统代理
  ...(isSystemProxyEnabled() ? { http_proxy, https_proxy } : {}),
};
```

### 5.4 端口管理

```typescript
// 启动时扫描可用端口（从默认 18789 开始，最多扫描 80 个）
// 连接信息持久化：
//   stateDir/gateway-port.json  → { port, version }
//   stateDir/gateway-token      → 认证 token

const availablePort = await scanAvailablePort(DEFAULT_GATEWAY_PORT, GATEWAY_PORT_SCAN_LIMIT);
```

### 5.5 健康检测与重启策略

```typescript
const GATEWAY_MAX_RESTART_ATTEMPTS = 5;
const GATEWAY_RESTART_DELAYS       = [3_000, 5_000, 10_000, 20_000, 30_000];

private async doStartGateway(): Promise<void> {
  const healthy = await this.isGatewayHealthy(port);
  if (healthy) {
    this.setStatus({ phase: 'running', ... });
    return;
  }
  // 健康检测失败 → 停止旧进程 → 重新启动
  await this.stopGatewayProcess();
  await this.startGatewayProcess();
}
```

---

## 六、配置同步层：`OpenClawConfigSync`

**文件**: `src/main/libs/openclawConfigSync.ts`

### 6.1 同步矩阵

| LobsterAI 配置 | OpenClaw 配置 |
|---------------|--------------|
| Provider API Key（加密存储）| `agents.providers[].apiKey`（`${LOBSTER_APIKEY_XXX}` 占位符）|
| Agent 定义（模型/工具/技能）| `agents.entries[]` |
| 技能列表（来自 `skills.config.json`）| `agents.entries[].skills[]` |
| MCP 服务器 | `mcp.servers[]` |
| 执行模式（local/auto/sandbox）| `sandbox.mode` |
| 系统提示（记忆策略、命令安全策略）| 注入到 system prompt |
| 渠道插件配置（Telegram/DingTalk/飞书等）| `channels` |

### 6.2 受管理的策略注入

**记忆策略**（注入 system prompt）:

```typescript
const MANAGED_MEMORY_POLICY_PROMPT = `## Memory Policy
**Write before you confirm.** When the user expresses any intent to persist information...
- Only say "记住了" AFTER the write tool call succeeds.`;
```

**命令安全策略**:

```typescript
const MANAGED_EXEC_SAFETY_PROMPT = `## Command Execution Policy
- Before executing delete operations, check if AskUserQuestion tool is available...`;
```

### 6.3 配置热同步策略

```typescript
if (hasActiveGatewayWorkloads()) {
  scheduleDeferredGatewayRestart(reason);  // 有活跃会话 → 延迟重启
  return { success: true, changed: true, status };
} else {
  // 无活跃会话 → 硬重启
  openClawRuntimeAdapter.disconnectGatewayClient();
  await manager.stopGateway();
  await manager.startGateway();
}
```

---

## 七、IPC 事件转发：`bindCoworkRuntimeForwarder`

**文件**: `src/main/main.ts`

渲染进程从不直接订阅引擎事件，而是通过 `BrowserWindow.webContents.send()` 推送：

```typescript
const bindCoworkRuntimeForwarder = (): void => {
  const runtime = getCoworkEngineRouter();

  runtime.on('message', (sessionId, message) => {
    const safeMessage = sanitizeCoworkMessageForIpc(message);
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('cowork:stream:message', { sessionId, message: safeMessage });
    });
  });

  runtime.on('messageUpdate', (sessionId, messageId, content) => {
    const safeContent = truncateIpcString(content, IPC_UPDATE_CONTENT_MAX_CHARS);
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('cowork:stream:messageUpdate', { sessionId, messageId, content: safeContent });
    });
  });

  runtime.on('permissionRequest', (sessionId, request) => {
    // text 确认模式下跳过（由 UI 直接处理）
    if (runtime.getSessionConfirmationMode(sessionId) === 'text') return;
    const safeRequest = sanitizePermissionRequestForIpc(request);
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('cowork:stream:permission', { sessionId, request: safeRequest });
    });
  });

  runtime.on('complete', (sessionId, claudeSessionId) => {
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('cowork:stream:complete', { sessionId, claudeSessionId });
    });
  });

  runtime.on('error', (sessionId, error) => {
    BrowserWindow.getAllWindows().forEach((win) => {
      win.webContents.send('cowork:stream:error', { sessionId, error });
    });
  });
};
```

---

## 八、完整事件流

```
┌─────────────────────────────────────────────────────────┐
│               OpenClaw Gateway 子进程                    │
│   WebSocket ws://127.0.0.1:port                          │
│   推送事件: tick | chat.delta/final/aborted/error         │
│            agent.assistant | agent.tool_*                │
│            exec.approval.requested/resolved              │
└──────────────────────────┬──────────────────────────────┘
                           │ ws.onmessage()
                           ▼
┌─────────────────────────────────────────────────────────┐
│          OpenClawRuntimeAdapter (协议适配器)               │
│  handleGatewayEvent()                                    │
│  ├─ handleChatEvent()      → 流式文本 / 最终确认 / 中止    │
│  ├─ handleAgentEvent()     → 工具调用三阶段                │
│  ├─ processAgentAssistantText() → 原始文本流合并          │
│  ├─ handleApprovalRequested() → permissionRequest        │
│  └─ handleApprovalResolved()  → 权限响应                  │
│                                                          │
│  ActiveTurn 状态机                                        │
│  throttledEmitMessageUpdate() → 节流保护                  │
└──────────────┬──────────────────────────────────────────┘
               │ this.emit('message'|'messageUpdate'|'complete'|...)
               ▼
┌─────────────────────────────────────────────────────────┐
│           CoworkEngineRouter (引擎路由门面)               │
│  sessionEngine Map 标记来源引擎                          │
│  requestEngine Map 精确路由权限响应                       │
│  向所有 BrowserWindow 广播（来源透明）                     │
└──────────────┬──────────────────────────────────────────┘
               │ runtime.on('message', ...)
               ▼
┌─────────────────────────────────────────────────────────┐
│        bindCoworkRuntimeForwarder (main.ts)              │
│  sanitizeCoworkMessageForIpc() 序列化                    │
│  win.webContents.send('cowork:stream:message', {...})  │
└──────────────┬──────────────────────────────────────────┘
               │ ipcRenderer.on('cowork:stream:message', ...)
               ▼
┌─────────────────────────────────────────────────────────┐
│            渲染进程 (React + Redux)                       │
│  coworkSlice 接收 action → 更新 UI 状态                  │
└─────────────────────────────────────────────────────────┘
```

---

## 九、设计决策总结

| 决策 | 实现方式 | 目的 |
|------|---------|------|
| **统一运行时接口** | `CoworkRuntime` 接口定义 | 内置引擎和 OpenClaw 完全可互换 |
| **门面模式路由** | `CoworkEngineRouter` 组合两个 `CoworkRuntime` | 对外暴露单一接口，内部多路复用 |
| **事件驱动解耦** | `EventEmitter` + IPC `send` 广播 | 渲染进程与引擎完全异步 |
| **协议转换集中化** | `OpenClawRuntimeAdapter` | OpenClaw 特有逻辑（WS/流式/重连）隔离在此 |
| **精确路由** | `requestEngine` Map | 权限响应准确路由到来源引擎 |
| **配置不落盘密钥** | `${VAR}` 占位符 + `secretEnvVars` | API Key 通过进程环境变量注入 |
| **进程隔离** | `utilityProcess.fork()` / `spawn ELECTRON_RUN_AS_NODE` | 网关崩溃不影响主进程 |
| **懒初始化** | `ensureGatewayClientReady` | 最小化资源占用 |
| **版本校验** | `needsNewClient` 比较 | 避免协议版本错配 |
| **流式文本双源合并** | `chat.delta` + `agent.assistant` 合并 | 处理 OpenClaw 同时发送两种流事件的冗余 |
| **心跳 watchdog** | Tick timestamp + 90s 超时 | 检测 OpenClaw 假死连接 |
| **节流保护** | `throttledEmitMessageUpdate` | 防止高频流式事件压垮 IPC |
| **软/硬重启策略** | 活跃会话时延迟重启 | 不打断用户工作 |
| **平台差异化启动** | macOS/Linux 用 `utilityProcess`，Windows 用 `spawn` | 规避 Windows ESM 冷启动性能问题 |
