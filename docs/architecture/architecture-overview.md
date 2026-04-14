# LobsterAI 项目架构总览

> 本文档为系列分析文档的**第一篇**，聚焦整体架构与模块划分。
> 配套文档：
> - [引擎抽象层分析](./engine-runtime-abstraction.md)
> - [存储层分析](./storage-layer.md)

---

## 一、技术栈

| 层次 | 技术选型 |
|------|---------|
| 桌面框架 | Electron |
| 前端框架 | React + TypeScript |
| 构建工具 | Vite |
| 进程间通信 | IPC (invoke/send) |
| 数据持久化 | SQLite (`better-sqlite3`) |
| Agent 引擎 | OpenClaw Gateway（主要）+ Claude Agent SDK（已废弃）|
| 协议通信 | WebSocket |
| 样式 | Tailwind CSS |

---

## 二、进程模型

```
┌──────────────────────────────────────────────────────────┐
│                    Main Process (主进程)                  │
│                                                           │
│  ┌─────────────────┐  ┌──────────────────┐               │
│  │  Window Manager  │  │  SQLite Store    │               │
│  │  (窗口生命周期)   │  │  (kv / cowork)    │               │
│  └─────────────────┘  └──────────────────┘               │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              CoworkEngineRouter                      │ │
│  │         (引擎路由层 — 门面模式)                       │ │
│  │  ┌──────────────────┐  ┌──────────────────────┐    │ │
│  │  │OpenClawRuntime    │  │ClaudeRuntimeAdapter  │    │ │
│  │  │Adapter            │  │(已废弃)               │    │ │
│  │  └────────┬─────────┘  └──────────────────────┘    │ │
│  │           │ WebSocket                               │ │
│  │           ▼                                         │ │
│  │  ┌──────────────────┐  ┌──────────────────┐         │ │
│  │  │OpenClawEngine    │  │CoworkRunner      │         │ │
│  │  │Manager (进程管理) │  │(内置 SDK Runner) │         │ │
│  │  └──────────────────┘  └──────────────────┘         │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐       │
│  │SkillManager│  │McpServer   │  │IM Gateways  │       │
│  │(技能管理)   │  │Manager     │  │(微信/钉钉/等)│       │
│  └────────────┘  └────────────┘  └──────────────┘       │
│                                                           │
└───────────────────────────┬───────────────────────────────┘
                            │ contextBridge (preload.ts)
                            │ IPC (invoke / send)
┌───────────────────────────▼───────────────────────────────┐
│                 Renderer Process (渲染进程)                │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                    React App                          │ │
│  │  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │ │
│  │  │ CoworkView  │  │ Artifacts  │  │ Settings    │  │ │
│  │  │(协作会话 UI) │  │ Panel       │  │ Panel        │  │ │
│  │  └────────────┘  └────────────┘  └─────────────┘  │ │
│  │                                                      │ │
│  │  ┌──────────────────────────────────────────────┐   │ │
│  │  │ Redux Store (coworkSlice / artifactSlice)   │   │ │
│  │  └──────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│             OpenClaw Gateway Process (独立子进程)          │
│  WebSocket (ws://127.0.0.1:port) ←─── OpenClawRuntimeAdapter
│                                                           │
│  职责：LLM 对话执行、工具调用、Telegram/DingTalk/飞书等渠道  │
└──────────────────────────────────────────────────────────┘
```

### 安全模型

- **Context Isolation**: 启用
- **Node Integration**: 禁用
- **Sandbox**: 启用
- 渲染进程与主进程完全隔离，通过 `contextBridge` 暴露安全的 `window.electron` API

---

## 三、目录结构

```
LobsterAI/
├── src/
│   ├── main/                          # Electron 主进程
│   │   ├── main.ts                    # 入口，IPC 注册，全局初始化
│   │   ├── preload.ts                 # contextBridge API 暴露
│   │   ├── coworkStore.ts             # SQLite CRUD（会话/消息/配置/记忆）
│   │   ├── sqliteStore.ts             # 底层 kv store
│   │   ├── skillManager.ts            # 技能安装/管理
│   │   ├── i18n.ts                    # 主进程 i18n
│   │   ├── logger.ts                  # electron-log 封装
│   │   │
│   │   ├── libs/
│   │   │   ├── agentEngine/          # ★ 引擎抽象层（核心）
│   │   │   │   ├── types.ts           # CoworkRuntime 接口定义
│   │   │   │   ├── coworkEngineRouter.ts    # 引擎路由（门面）
│   │   │   │   ├── openclawRuntimeAdapter.ts # OpenClaw 适配器
│   │   │   │   └── claudeRuntimeAdapter.ts  # 内置引擎适配器
│   │   │   ├── openclawEngineManager.ts     # OpenClaw 进程生命周期
│   │   │   ├── openclawConfigSync.ts        # LobsterAI 配置 → OpenClaw 配置
│   │   │   ├── openclawChannelSessionSync.ts # 渠道会话同步
│   │   │   ├── coworkMemoryExtractor.ts      # 记忆提取（规则引擎）
│   │   │   ├── coworkMemoryJudge.ts          # 记忆质量评估（规则 + LLM）
│   │   │   ├── mcpServerManager.ts          # MCP 服务器管理
│   │   │   ├── coworkRunner.ts              # 内置引擎执行器
│   │   │   ├── claudeSdk.ts                 # SDK 加载工具
│   │   │   ├── commandSafety.ts             # 命令安全评估
│   │   │   └── ...
│   │   │
│   │   ├── im/                       # IM 渠道网关
│   │   │   ├── types.ts
│   │   │   ├── imStore.ts
│   │   │   ├── feishu.ts / dingtalk.ts / wechat.ts / ...
│   │   │
│   │   └── shared/                   # 跨进程共享类型/工具
│   │       ├── platform.ts
│   │       └── providers.ts
│   │
│   └── renderer/                     # React 前端
│       ├── App.tsx
│       ├── services/
│       │   ├── api.ts               # LLM API（SSE 流式）
│       │   ├── cowork.ts             # Cowork IPC 封装
│       │   ├── artifactParser.ts   # Artifact 检测与解析
│       │   └── i18n.ts              # 渲染进程 i18n
│       ├── store/
│       │   └── slices/
│       │       ├── coworkSlice.ts   # Redux：会话/流式状态
│       │       └── artifactSlice.ts # Redux：Artifact 状态
│       ├── components/
│       │   ├── cowork/              # Cowork UI 组件
│       │   └── artifacts/           # Artifact 渲染器
│       └── types/
│           └── cowork.ts
│
├── SKILLs/                          # 技能定义目录
│   ├── skills.config.json           # 技能启用/顺序配置
│   ├── docx/ / xlsx/ / pptx/ ...
│   └── ...
│
├── openclaw-extensions/             # OpenClaw 渠道插件源码
│
├── resources/
│   └── cfmind/                      # 打包的 OpenClaw 运行时
│
└── docs/architecture/               # 本系列文档
    ├── architecture-overview.md     # 本文档
    ├── engine-runtime-abstraction.md # 引擎抽象层
    └── storage-layer.md             # 存储层
```

---

## 四、认证与令牌体系

```
登录流程:
  1. 打开系统浏览器 → Portal 登录页 → URS 登录
  2. 回调 lobsterai://auth/callback?code=<authCode>
  3. POST /api/auth/exchange → 获得 accessToken(2h) + refreshToken(30d)
  4. SQLite kv 表存储双 token，应用重启后自动恢复
  5. 被动刷新：收到 401 → 使用 refreshToken 刷新 → 重试原请求
  6. 主动刷新：accessToken 距 exp < 5 分钟 → 后台静默刷新
  7. 滚动续期：每次 refresh 签发新 refreshToken（30 天）
```

关键文件：
- 令牌存储与请求：`src/renderer/services/api.ts`（`fetchWithAuth()`）
- 登录流程：`src/main/main.ts`（deep link 处理）
- 持久化：`src/main/sqliteStore.ts`（kv 表）

---

## 五、Artifact 系统

支持富文本代码预览：

| 类型 | 渲染方式 | 安全沙箱 |
|------|---------|---------|
| HTML | sandboxed iframe | `allow-scripts`，禁止 `allow-same-origin` |
| SVG | DOMPurify + 内联渲染 | 去除所有 script |
| React/JSX | Babel 编译 + 隔离 iframe | 完全无网络访问 |
| Mermaid | Mermaid.js | `securityLevel: 'strict'` |
| Code | 语法高亮 + 行号 | — |

检测方式：
1. 显式标记：`` ```artifact:html title="..." ``
2. 启发式检测：分析代码块语言和内容模式

---

## 六、配置体系

所有配置存储在 SQLite `kv` 表（主配置）和 `cowork_config` 表（协作会话配置）中：

```sql
-- 主配置（kv 表）
INSERT INTO kv (key, value) VALUES ('theme', 'dark');

-- Cowork 配置
INSERT INTO cowork_config (key, value) VALUES ('agentEngine', 'openclaw');
INSERT INTO cowork_config (key, value) VALUES ('memoryEnabled', '1');
INSERT INTO cowork_config (key, value) VALUES ('memoryImplicitUpdateEnabled', '1');
```

数据库文件：`lobsterai.sqlite`，位于用户数据目录。

---

## 七、关键设计哲学

1. **进程隔离**: OpenClaw Gateway 运行在独立子进程中，崩溃不影响主进程
2. **事件驱动**: 所有引擎事件通过 `EventEmitter` → IPC `send` 异步推送到渲染进程
3. **零直接调用**: 渲染进程从不直接调用主进程模块，全部通过 IPC
4. **配置即代码**: OpenClaw 的 `openclaw.json` 由 `openclawConfigSync.ts` 从 LobsterAI 配置动态生成
5. **密钥不落盘**: API Key 通过 `${VAR}` 占位符写入配置，实际值注入到子进程环境变量
