# 存储层分析

> 本文档为系列分析文档的**第三篇**，分析 LobsterAI 中所有持久化存储的设计与实现。
> 前置文档：[架构总览](./architecture-overview.md)、[引擎抽象层](./engine-runtime-abstraction.md)

---

## 一、存储总览

LobsterAI 的存储分为**两个层次**：

```
SQLite 数据库层（src/main/coworkStore.ts + sqliteStore.ts）
    │
    ├── kv 表          — 全局键值配置（主题、语言、窗口状态等）
    ├── cowork_config  — 协作会话配置（引擎选择、记忆设置等）
    ├── cowork_sessions — 协作会话元数据
    ├── cowork_messages — 协作消息（user/assistant/system/tool_use/tool_result）
    ├── memory_entries — 记忆条目（文件路径 / 内容 / 质量分 / 搜索词）
    └── scheduled_task_meta — 定时任务元数据
            │
            └── OpenClaw 工作目录（文件系统）
                ├── MEMORY.md       — 持久化记忆主文件
                ├── memory/         — 每日记忆笔记
                │   └── YYYY-MM-DD.md
                ├── USER.md / SOUL.md — 用户画像 / Agent 人格
                └── .claude/        — OpenClaw 内部状态
```

**两种存储的定位**:
- **SQLite（结构化数据）**: 快速查询、原子更新、跨会话持久化。适合会话列表、配置、记忆元数据。
- **文件系统（文件内容）**: OpenClaw 直接读写，LLM 可直接编辑。适合 `MEMORY.md`（LLM 用 `write` 工具维护）。

---

## 二、SQLite 底层：`sqliteStore.ts`

### 2.1 表结构

```typescript
// kv 表 — 通用键值存储
db.exec(`
  CREATE TABLE IF NOT EXISTS kv (
    key TEXT PRIMARY KEY,
    value TEXT
  )
`);

// cowork_sessions 表
db.exec(`
  CREATE TABLE IF NOT EXISTS cowork_sessions (
    id TEXT PRIMARY KEY,
    title TEXT,
    pinned INTEGER DEFAULT 0,
    status TEXT,
    created_at INTEGER,
    updated_at INTEGER
  )
`);

// cowork_messages 表
db.exec(`
  CREATE TABLE IF NOT EXISTS cowork_messages (
    id TEXT PRIMARY KEY,
    session_id TEXT,
    type TEXT,          -- 'user' | 'assistant' | 'system' | 'tool_use' | 'tool_result'
    content TEXT,
    tool_name TEXT,
    tool_call_id TEXT,
    is_final INTEGER DEFAULT 0,
    is_streaming INTEGER DEFAULT 0,
    created_at INTEGER,
    FOREIGN KEY (session_id) REFERENCES cowork_sessions(id)
  )
`);

// cowork_config 表
db.exec(`
  CREATE TABLE IF NOT EXISTS cowork_config (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at INTEGER
  )
`);
```

### 2.2 核心 API

```typescript
class SqliteStore {
  get(key: string): string | null;
  set(key: string, value: string): void;
  has(key: string): boolean;
  delete(key: string): void;
  // ... 事务支持
}
```

---

## 三、会话与消息存储：`coworkStore.ts`

**文件**: `src/main/coworkStore.ts`

`CoworkStore` 封装了所有与协作会话相关的数据库操作。

### 3.1 核心 CRUD

```typescript
class CoworkStore {
  // 会话操作
  createSession(id: string, data: CreateSessionData): CoworkSession;
  getSession(id: string): CoworkSession | null;
  listSessions(options?: ListSessionsOptions): CoworkSession[];
  updateSession(id: string, data: Partial<UpdateSessionData>): void;
  deleteSession(id: string): void;
  deleteSessionBatch(ids: string[]): void;

  // 消息操作
  addMessage(sessionId: string, data: CreateMessageData): CoworkMessage;
  getMessages(sessionId: string, options?: GetMessagesOptions): CoworkMessage[];
  updateMessage(id: string, content: string): void;
  markMessageFinal(id: string): void;
  deleteMessagesBySession(sessionId: string): void;

  // 配置操作
  getConfig(): CoworkConfig;
  setConfig(config: Partial<SetCoworkConfigData>): void;
}
```

### 3.2 消息类型

```typescript
type MessageType = 'user' | 'assistant' | 'system' | 'tool_use' | 'tool_result';

interface CoworkMessage {
  id: string;
  sessionId: string;
  type: MessageType;
  content: string;
  toolName?: string;       // tool_use / tool_result
  toolCallId?: string;    // tool_use
  isFinal: boolean;
  isStreaming: boolean;
  createdAt: number;
}
```

### 3.3 消息历史构建（供引擎使用）

```typescript
getMessagesForHistory(
  sessionId: string,
  limit: number,
): Array<{ role: string; content: string }> {
  const messages = this.getMessages(sessionId, { limit });
  return messages.map((msg) => ({
    role: msg.type === 'user' ? 'user'
        : msg.type === 'assistant' ? 'assistant'
        : 'system',
    content: msg.content,
  }));
}
```

---

## 四、记忆系统

LobsterAI 的记忆系统由**两部分**组成：

```
┌──────────────────────────────────────────────────────────────┐
│                    SQLite memory_entries 表                   │
│  (记忆元数据：路径、搜索词、质量分、标签、活跃状态)              │
│  → 支持快速相似度查询、去重、活跃条目标记                        │
└──────────────────────────┬───────────────────────────────────┘
                           │ IPC: cowork:memory:*
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              OpenClaw 工作目录文件系统                         │
│  MEMORY.md / memory/YYYY-MM-DD.md / USER.md / SOUL.md        │
│  → LLM 通过 write 工具直接读写                                 │
└──────────────────────────────────────────────────────────────┘
```

### 4.1 记忆提取器：`coworkMemoryExtractor.ts`

**文件**: `src/main/libs/coworkMemoryExtractor.ts`

采用**规则引擎**从对话中提取记忆条目，无需 LLM 调用。

#### 显式记忆提取（高置信度 0.99）

```typescript
// 命中以下模式 → 提取为记忆
const EXPLICIT_ADD_RE = /(?:^|\n)\s*(?:请)?(?:记住|记下|保存到记忆|remember|store\s+in\s+memory)\s*[:：,，]?\s*(.+)$/gim;
const EXPLICIT_DELETE_RE = /(?:^|\n)\s*(?:删除记忆|从记忆中删除|忘掉|forget\s+this)\s*[:：,，]?\s*(.+)$/gim;
```

#### 隐式记忆提取（候选 → 质量评分 → 决定是否存入）

**候选信号**（高置信度指标）：

| 信号类型 | 正则 | 示例 |
|---------|------|------|
| 个人简介 | `PERSONAL_PROFILE_SIGNAL_RE` | "我叫张三"、"我住在上海" |
| 个人偏好 | `PERSONAL_PREFERENCE_SIGNAL_RE` | "我喜欢 Python"、"我习惯用 VSCode" |
| 个人所有权 | `PERSONAL_OWNERSHIP_SIGNAL_RE` | "我养了只猫"、"我的公司是..." |
| 简短事实 | `SHORT_FACT_SIGNAL_RE` | "我叫..."（短句开头） |

**过滤信号**（排除非记忆内容）：

| 信号类型 | 正则 | 原因 |
|---------|------|------|
| 小对话 | `SMALL_TALK_RE` | "好的"、"谢谢" |
| 问句 | `isQuestionLikeMemoryText()` | "怎么配置？" |
| 程序化 | `PROCEDURAL_CANDIDATE_RE` | "执行以下命令..." |
| 助手风格 | `ASSISTANT_STYLE_CANDIDATE_RE` | "使用 xlsx 技能" |
| 临时话题 | `TRANSIENT_SIGNAL_RE` | "今天天气..." |

#### 质量评分函数

```typescript
function scoreMemoryTextQuality(value: string): number {
  let score = normalized.length;  // 长度加权
  if (/^该用户|^the\s+user/i) score -= 12;  // 第三人称减分
  if (/^我|^i\s/i) score += 4;               // 第一人称加分
  return score;
}
```

### 4.2 记忆质量评估：`coworkMemoryJudge.ts`

**文件**: `src/main/libs/coworkMemoryJudge.ts`

对候选记忆进行**二级评估**，决定是否存入。

#### 两阶段评估

```
候选记忆文本
     │
     ├── 阶段 1：规则快速判断
     │   ├── 空文本 / 问句 / 程序化 → 拒绝
     │   └── 通过 → 进入阶段 2
     │
     └── 阶段 2：阈值判断
         ├── explicit: strict=0.7, standard=0.6, relaxed=0.52
         └── implicit: strict=0.8, standard=0.72, relaxed=0.62
             │
             ├── 边界情况（阈值 ±0.08）→ LLM 裁决（可选）
             └── 其他 → 直接接受/拒绝
```

#### LLM 边界裁决（可选）

```typescript
const LLM_BORDERLINE_MARGIN = 0.08;
const LLM_MIN_CONFIDENCE    = 0.55;
const LLM_TIMEOUT_MS        = 5000;
const LLM_CACHE_MAX_SIZE    = 256;   // LRU 缓存
const LLM_CACHE_TTL_MS       = 10 * 60 * 1000; // 10 分钟

// 缓存 key: `${guardLevel}|${isExplicit}|${text}`
const llmJudgeCache = new Map<string, CachedLlmJudgeResult>();
```

### 4.3 记忆配置

```typescript
interface CoworkConfig {
  memoryEnabled: boolean;           // 记忆总开关，默认 true
  memoryImplicitUpdateEnabled: boolean;  // 自动提取隐式记忆，默认 true
  memoryLlmJudgeEnabled: boolean;   // LLM 边界裁决，默认 false
  memoryGuardLevel: CoworkMemoryGuardLevel;  // 'strict' | 'standard' | 'relaxed'
  memoryUserMemoriesMaxItems: number;      // USER.md 最大条目数，默认 12
}
```

**Guard Level 阈值对比**：

| Level | 显式阈值 | 隐式阈值 | 适用场景 |
|-------|---------|---------|---------|
| strict | 0.7 | 0.8 | 高隐私、频繁遗忘 |
| standard | 0.6 | 0.72 | 平衡（默认）|
| relaxed | 0.52 | 0.62 | 积极记忆 |

### 4.4 记忆去重机制

```typescript
function scoreMemorySimilarity(left: string, right: string): number {
  return Math.max(
    // 短语包含率（长度比）
    phraseScore,
    // Token 重叠率
    scoreTokenOverlap(left, right),
    // 字符二元组 Dice 系数
    scoreCharacterBigramDice(left, right)
  );
}

// 阈值: 0.82 以上认为是重复
const MEMORY_NEAR_DUPLICATE_MIN_SCORE = 0.82;
```

---

## 五、OpenClaw 渠道会话同步

**文件**: `src/main/libs/openclawChannelSessionSync.ts`

将 OpenClaw 渠道（Telegram/DingTalk/飞书等）产生的会话与本地 SQLite 会话绑定。

### 5.1 SessionKey 格式

```
LobsterAI 原生会话:  agent:main:lobsterai:{sessionId}
OpenClaw 渠道会话:   agent:{agentId}:{platform}:{accountId}:{peerKind}:{peerId}
  例: agent:main:telegram:123456:user:987654
Cron 会话:           agent:{agentId}:cron:{jobId}
DingTalk HTTP:       agent:{agentId}:openai-user:{jsonContext}
```

### 5.2 轮询发现机制

```typescript
const CHANNEL_POLL_INTERVAL_MS = 10_000; // 每 10 秒轮询

startChannelPolling(): void {
  void this.pollChannelSessions();
  this.channelPollingTimer = setInterval(() => {
    void this.pollChannelSessions();
  }, CHANNEL_POLL_INTERVAL_MS);
}

private async pollChannelSessions(): Promise<void> {
  // 调用 sessions.list RPC 发现新渠道会话
  const sessions = await this.gatewayClient.request('sessions.list', {
    limit: CHANNEL_SESSION_DISCOVERY_LIMIT,
  });

  for (const session of sessions) {
    if (isManagedSessionKey(session.key)) continue; // 跳过原生会话
    const localSessionId = this.resolveOrCreateSession(session.key);
    if (localSessionId) {
      // 全量同步历史消息
      await this.syncFullChannelHistory(session.key, localSessionId);
    }
  }
}
```

### 5.3 渠道会话 → 本地会话映射

```typescript
resolveOrCreateSession(sessionKey: string): string | null {
  // 1. 跳过 LobsterAI 原生会话
  if (isManagedSessionKey(sessionKey)) return null;

  // 2. 查内存缓存
  if (this.syncedSessionKeys.has(sessionKey))
    return this.syncedSessionKeys.get(sessionKey);

  // 3. 解析渠道信息
  const { platform, conversationId } = parseChannelSessionKey(sessionKey);
  const agentId = resolveAgentBinding(this.bindings, platform, accountId);

  // 4. 创建本地 CoworkSession
  const localSessionId = this.coworkStore.createSession({ ... });

  // 5. 触发全量历史同步
  await this.syncFullChannelHistory(sessionKey, localSessionId);

  // 6. 缓存映射
  this.syncedSessionKeys.set(sessionKey, localSessionId);
  return localSessionId;
}
```

---

## 六、技能管理：`SkillManager`

**文件**: `src/main/skillManager.ts`

### 6.1 技能目录结构

```
SKILLs/                          # 根目录（SKILLS_ROOT）
├── skills.config.json           # 全局启用/顺序配置
├── docx/
│   ├── SKILL.md                 # 技能定义文件
│   ├── AGENTS.md               # Agent 接口声明
│   └── ...（技能资源）
├── xlsx/
├── pptx/
└── ...
```

### 6.2 `skills.config.json` 结构

```json
{
  "skills": [
    { "id": "docx", "enabled": true, "order": 1 },
    { "id": "xlsx", "enabled": true, "order": 2 }
  ]
}
```

### 6.3 技能发现与加载

```typescript
export class SkillManager {
  async discoverSkills(): Promise<SkillRecord[]> {
    const skillsDir = path.join(getAppRoot(), 'SKILLs');
    const entries = fs.readdirSync(skillsDir, { withFileTypes: true });
    const skills: SkillRecord[] = [];

    for (const entry of entries) {
      if (!entry.isDirectory()) continue;
      const skillFile = path.join(skillsDir, entry.name, 'SKILL.md');
      if (!fs.existsSync(skillFile)) continue;
      const content = fs.readFileSync(skillFile, 'utf-8');
      skills.push(this.parseSkillFile(entry.name, content));
    }

    return this.sortByConfigOrder(skills);
  }
}
```

### 6.4 技能变更 → 配置热同步

```typescript
// skillManager.ts
public onSkillsChanged(): void {
  // 通知配置同步层
  syncOpenClawConfig({ reason: 'skills-changed' });
}
```

### 6.5 渠道禁用技能策略

```typescript
// openclawConfigSync.ts 中，部分技能在特定渠道被禁用：
const DISABLED_MANAGED_SKILL_NAMES = [
  'qqbot-cron',         // 原因：使用渠道特定的 cron 封装，应使用 OpenClaw 内置 cron
  'feishu-cron-reminder',
];
```

---

## 七、MCP 服务器管理：`McpServerManager`

**文件**: `src/main/libs/mcpServerManager.ts`

### 7.1 MCP Bridge 架构

```
OpenClaw Gateway
    │ HTTP POST (带 callbackUrl)
    ▼
LobsterAI McpBridgeServer (Node.js 内置 HTTP 服务器)
    │
    ├── 路由：/mcp/callback/{serverName}/{toolName}
    │
    ▼
LobsterAI McpServerManager
    │
    ├── 查找对应 MCP Server
    │
    ▼
MCP Server (stdio 通信)
    例：文件系统 MCP / Git MCP / 数据库 MCP
```

### 7.2 MCP Bridge 配置

```typescript
export interface McpBridgeConfig {
  callbackUrl: string;        // LobsterAI 暴露的回调地址
  askUserCallbackUrl: string;  // 需要用户确认的回调地址
  secret: string;             // 签名密钥
  tools: McpToolManifestEntry[]; // 可用工具清单（tool name → server 映射）
}

export interface McpToolManifestEntry {
  server: string;   // MCP 服务器名
  name: string;     // 工具名
}
```

### 7.3 工具发现与清单同步

```typescript
class McpServerManager {
  async refreshToolManifest(): Promise<void> {
    this._toolManifest = [];
    for (const [name, server] of this.servers) {
      const tools = await server.listTools();
      for (const tool of tools) {
        this._toolManifest.push({
          server: name,
          name: tool.name,
        });
      }
    }
    // 通知 OpenClawConfigSync 更新配置
    syncOpenClawConfig({ reason: 'mcp-servers-changed' });
  }
}
```

---

## 八、SQLite Store（KV 层）

**文件**: `src/main/sqliteStore.ts`

### 8.1 表结构

```sql
CREATE TABLE IF NOT EXISTS kv (
  key TEXT PRIMARY KEY,
  value TEXT
);

-- 认证令牌（加密或明文存储）
INSERT INTO kv (key, value) VALUES ('auth_tokens', '{"accessToken":"...","refreshToken":"..."}');
```

### 8.2 Auth Token 持久化

```typescript
// 登录成功后
sqliteStore.set('auth_tokens', JSON.stringify({
  accessToken,
  refreshToken,
  expiresAt: Date.now() + 2 * 60 * 60 * 1000, // 2h
}));

// 应用启动时恢复
const tokens = JSON.parse(sqliteStore.get('auth_tokens'));
if (tokens && tokens.expiresAt > Date.now()) {
  // 使用 accessToken
} else {
  // 使用 refreshToken 刷新
}
```

---

## 九、OpenClaw 运行时文件存储

**文件**: `src/main/libs/openclawEngineManager.ts`

### 9.1 目录结构

```
{userDataPath}/openclaw/           # baseDir
├── state/                         # 状态目录
│   ├── openclaw.json              # 生成的 OpenClaw 配置
│   ├── gateway-port.json          # { port, version }
│   └── gateway-token              # 认证 token
├── logs/                          # 日志目录
│   └── gateway.log                # OpenClaw 网关日志
└── resources/                     # 运行时目录（版本化）
    └── cfmind-{version}/
        └── ...
```

### 9.2 版本管理

```typescript
// 从 package.json 读取 pinned 版本
const runtime = this.resolveRuntimeMetadata();
this.desiredVersion = runtime.version || DEFAULT_OPENCLAW_VERSION;

// 版本变化时触发重新安装
if (runtime.version !== this.desiredVersion) {
  await this.installRuntime(this.desiredVersion);
}
```

---

## 十、存储设计决策总结

| 决策 | 实现方式 | 目的 |
|------|---------|------|
| **SQLite 统一存储** | `better-sqlite3` 同步 API | 原子性、事务支持、快速查询 |
| **双层记忆系统** | SQLite 元数据 + 文件系统内容 | SQLite 快速检索，文件系统供 LLM 直接编辑 |
| **规则引擎提取** | `coworkMemoryExtractor.ts` 正则引擎 | 无需 LLM 调用的高效提取 |
| **LLM 边界裁决** | `coworkMemoryJudge.ts` 可选 LLM 调用 | 临界情况高精度判断，带 LRU 缓存 |
| **记忆去重** | Token 重叠率 + 字符 Dice 系数 | 阈值 0.82 以上认为是重复 |
| **渠道会话透明化** | `OpenClawChannelSessionSync` | IM 渠道会话与原生会话在 UI 表现一致 |
| **轮询发现渠道会话** | 每 10 秒调用 `sessions.list` RPC | 可靠发现渠道侧发起的会话 |
| **技能配置热同步** | `onSkillsChanged()` → `syncOpenClawConfig()` | 技能变更无需重启网关 |
| **MCP Bridge 架构** | LobsterAI 内置 HTTP 服务器 | OpenClaw 可调用任意 MCP 工具，无需每个 MCP 单独适配 |
| **配置不落盘密钥** | `${VAR}` 占位符 + 进程环境变量 | API Key 安全注入，不写入 `openclaw.json` |
| **认证令牌 SQLite 持久化** | `kv` 表 | 应用重启后自动恢复登录态 |
| **版本化管理运行时** | `cfmind-{version}` 目录 | 支持多版本共存，平滑升级 |
| **平台差异化进程启动** | macOS 用 `utilityProcess`，Windows 用 `spawn` | 规避 Windows ESM 性能问题 |
