
以下是 LobsterAI 系统中所有数据的完整清单，按存储位置分类：

---

## 一、SQLite 数据库（`lobsterai.sqlite`）

路径：`{userDataPath}/lobsterai.sqlite`

所有表都在**同一个数据库文件**中，启用 **WAL 模式**（写并发优化）：

### 1.1 `kv` 表 — 全局键值配置

```sql
CREATE TABLE kv (key TEXT PRIMARY KEY, value TEXT NOT NULL, updated_at INTEGER);
```

| 典型 Key | 内容 |
|---------|------|
| `auth_tokens` | 登录令牌 `{accessToken, refreshToken, expiresAt}` |
| `theme` | UI 主题 |
| `language` | 语言设置 |
| `windowState` | 窗口位置/大小状态 |

### 1.2 `cowork_sessions` 表 — 协作会话元数据

```sql
CREATE TABLE cowork_sessions (
  id TEXT PRIMARY KEY,
  title TEXT, pinned INTEGER, status TEXT,
  cwd TEXT, system_prompt TEXT, execution_mode TEXT,
  created_at INTEGER, updated_at INTEGER
);
```

### 1.3 `cowork_messages` 表 — 协作消息

```sql
CREATE TABLE cowork_messages (
  id TEXT PRIMARY KEY, session_id TEXT,
  type TEXT,           -- 'user' | 'assistant' | 'system' | 'tool_use' | 'tool_result'
  content TEXT, metadata TEXT,
  sequence INTEGER,
  created_at INTEGER,
  FOREIGN KEY (session_id) REFERENCES cowork_sessions(id) ON DELETE CASCADE
);
```

### 1.4 `cowork_config` 表 — 协作配置

```sql
CREATE TABLE cowork_config (
  key TEXT PRIMARY KEY, value TEXT, updated_at INTEGER
);
```

| 典型 Key | 内容 |
|---------|------|
| `agentEngine` | `'openclaw'` 或 `'yd_cowork'` |
| `workingDirectory` | 工作目录路径 |
| `memoryEnabled` | `1` / `0` |
| `memoryImplicitUpdateEnabled` | `1` / `0` |
| `memoryLlmJudgeEnabled` | `1` / `0` |
| `memoryGuardLevel` | `'strict'` / `'standard'` / `'relaxed'` |

### 1.5 `user_memories` 表 — 记忆条目元数据

```sql
CREATE TABLE user_memories (
  id TEXT PRIMARY KEY,
  text TEXT,              -- 记忆文本内容
  fingerprint TEXT,       -- 去重指纹
  confidence REAL,         -- 置信度
  is_explicit INTEGER,   -- 是否显式记忆
  status TEXT,            -- 'created' | 'archived'
  created_at INTEGER, updated_at INTEGER, last_used_at INTEGER
);
```

### 1.6 `user_memory_sources` 表 — 记忆来源追踪

```sql
CREATE TABLE user_memory_sources (
  id TEXT PRIMARY KEY, memory_id TEXT,
  session_id TEXT, message_id TEXT,
  role TEXT, is_active INTEGER,
  FOREIGN KEY (memory_id) REFERENCES user_memories(id) ON DELETE CASCADE
);
```

### 1.7 `agents` 表 — 多 Agent 配置

```sql
CREATE TABLE agents (
  id TEXT PRIMARY KEY, name TEXT, description TEXT,
  system_prompt TEXT, identity TEXT, model TEXT,
  icon TEXT, skill_ids TEXT, enabled INTEGER,
  is_default INTEGER, source TEXT, preset_id TEXT,
  created_at INTEGER, updated_at INTEGER
);
```

### 1.8 `mcp_servers` 表 — MCP 服务器配置

```sql
CREATE TABLE mcp_servers (
  id TEXT PRIMARY KEY, name TEXT UNIQUE,
  description TEXT, enabled INTEGER,
  transport_type TEXT,           -- 'stdio' | 'sse' | 'http'
  config_json TEXT,              -- {command, args, env, url, headers...}
  created_at INTEGER, updated_at INTEGER
);
```

### 1.9 `im_config` 表 — IM 渠道配置

```sql
CREATE TABLE im_config (
  key TEXT PRIMARY KEY,  -- 'telegram' | 'feishu' | 'dingtalk' | 'settings' | ...
  value TEXT, updated_at INTEGER
);
```

各渠道配置 JSON 存储在此，包括 bot token、webhook secret、平台特有配置。

### 1.10 `im_session_mappings` 表 — 渠道会话 ↔ 本地会话映射

```sql
CREATE TABLE im_session_mappings (
  im_conversation_id TEXT, platform TEXT,
  cowork_session_id TEXT, agent_id TEXT,
  created_at INTEGER, last_active_at INTEGER,
  PRIMARY KEY (im_conversation_id, platform)
);
```

### 1.11 `scheduled_task_meta` 表 — 定时任务元数据

```sql
CREATE TABLE scheduled_task_meta (
  task_id TEXT PRIMARY KEY,
  origin TEXT,   -- JSON.stringify(TaskOrigin)
  binding TEXT   -- JSON.stringify(ExecutionBinding)
);
```

> **说明**：OpenClaw Gateway 的 cron API 不支持自定义字段，所以将 origin/binding 信息持久化在本地。

---

## 二、文件系统

### 2.1 `{userDataPath}/openclaw/` — OpenClaw 运行时数据

```
{userDataPath}/openclaw/
├── state/
│   ├── openclaw.json          # LobsterAI 生成的 OpenClaw 配置
│   ├── gateway-token          # WebSocket 认证 Token
│   ├── gateway-port.json      # {port, version}
│   └── credentials/           # IM 渠道配对/白名单文件
│       ├── telegram-pairing.json      # 配对请求列表
│       ├── telegram-allowFrom.json   # 允许来源列表
│       ├── feishu-allowFrom.json
│       └── {channel}-{accountId}-allowFrom.json  # 多实例
├── logs/
│   └── gateway.log            # OpenClaw 网关日志（electron-log）
└── resources/                # 运行时目录（版本化）
    └── cfmind-{version}/     # OpenClaw 二进制/资源
```

### 2.2 `{userDataPath}/openclaw/workspace/` — OpenClaw 工作区

```
{userDataPath}/openclaw/workspace/
├── AGENTS.md                 # LobsterAI 生成的系统提示
├── SOUL.md                   # Agent 人格
├── USER.md                   # 用户画像
├── IDENTITY.md               # Agent 身份
├── MEMORY.md                 # 持久化记忆主文件（LLM 直接读写）
├── TOOLS.md                  # 工具说明
├── HEARTBEAT.md             # 心跳文件
└── memory/
    └── YYYY-MM-DD.md         # 每日记忆笔记
```

### 2.3 `{userDataPath}/SKILLs/` — 技能目录

```
{userDataPath}/SKILLs/
├── skills.config.json         # 技能启用/顺序配置
├── docx/
│   ├── SKILL.md              # 技能定义
│   ├── AGENTS.md            # Agent 接口声明
│   ├── package.json         # 依赖
│   ├── .env                 # 环境变量（可含密钥）
│   └── ...（资源文件）
├── xlsx/
├── pptx/
└── {skill-name}/
```

### 2.4 electron-log 日志

```
{userDataPath}/logs/
├── main.log                  # 主进程日志（每日轮转）
├── renderer.log             # 渲染进程日志
└── gateway.log               # OpenClaw 网关日志
```

### 2.5 OpenClaw 打包资源

```
resources/cfmind/             # macOS 生产打包
  └── cfmind-{version}/

vendor/openclaw-runtime/current/  # 开发模式
```

### 2.6 应用日志导出

```
{documentsDir}/LobsterAI/logs/  # 用户主动导出的日志
```

---

## 三、OpenClaw 自管理的存储

OpenClaw Gateway **自己管理**的数据（路径在 `{userDataPath}/openclaw/state/` 下，OpenClaw 直接读写）：

| 数据 | 路径 | 内容 |
|------|------|------|
| Session 历史 | `agents/{agentId}/sessions/` | JSONL 格式的会话消息记录 |
| Session 索引 | `agents/{agentId}/sessions.json` | 所有会话 key 的列表 |
| Cron 任务 | OpenClaw 内部管理 | 通过 cron API 创建，LobsterAI 只记录元数据 |
| 渠道 WebSocket 连接 | OpenClaw 内部管理 | 钉钉/飞书/Telegram 等的连接状态 |
| 归档会话 | `{sessionId}.jsonl.deleted.{timestamp}` | 被 OpenClaw 维护逻辑归档的会话 |

---

## 四、凭证存储架构

```
┌─────────────────────────────────────────────────────────────┐
│  IM 渠道凭证                                                 │
│  ─────────────────────────────────────────────────────────  │
│  文件: {stateDir}/credentials/{channel}-pairing.json       │
│  内容: { version: 1, requests: [{id, code, createdAt}] }   │
│  管理方: LobsterAI（通过 imPairingStore 直接读写 JSON）      │
│  用途: 配对请求缓存                                          │
│                                                             │
│  文件: {stateDir}/credentials/{channel}-allowFrom.json    │
│  内容: { version: 1, allowFrom: ['...'] }                  │
│  用途: 消息来源白名单（防止消息伪造）                          │
│                                                             │
│  API Keys / Bot Tokens / Secrets                           │
│  ─────────────────────────────────────────────────────────  │
│  存储位置: IM 渠道配置的 `im_config` 表（JSON 字段）           │
│  注入方式: → openclawConfigSync.collectSecretEnvVars()       │
│             → 注入到 OpenClaw 网关进程环境变量                 │
│  不落盘: API Key 永远不写入 openclaw.json                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、数据总览图

```
{userDataPath}/
├── lobsterai.sqlite                      # SQLite（主数据库）
│   ├── kv                                # 全局配置（auth_tokens, theme, language...）
│   ├── cowork_sessions                   # 协作会话
│   ├── cowork_messages                   # 协作消息
│   ├── cowork_config                     # 协作配置（引擎、记忆、执行模式）
│   ├── user_memories                     # 记忆条目
│   ├── user_memory_sources               # 记忆来源
│   ├── agents                            # 多 Agent 配置
│   ├── mcp_servers                       # MCP 服务器
│   ├── im_config                         # IM 渠道配置（JSON 存凭证）
│   ├── im_session_mappings               # 渠道会话 ↔ 本地会话映射
│   └── scheduled_task_meta               # 定时任务元数据
│
├── openclaw/                             # OpenClaw 运行时
│   ├── state/
│   │   ├── openclaw.json                # LobsterAI 生成（不含密钥）
│   │   ├── gateway-token                # Token（不含密钥）
│   │   ├── gateway-port.json            # {port, version}
│   │   └── credentials/                  # 配对/白名单 JSON
│   ├── logs/gateway.log                  # OpenClaw 网关日志
│   ├── workspace/                        # OpenClaw 工作区（LLM 直接读写）
│   │   ├── MEMORY.md / memory/          # LLM 记忆
│   │   ├── USER.md / SOUL.md            # 用户/Agent 画像
│   │   └── AGENTS.md                    # LobsterAI 生成的系统提示
│   └── resources/cfmind-{version}/      # OpenClaw 二进制
│
├── SKILLs/                               # 技能（技能定义、代码、资源）
│   ├── skills.config.json
│   └── {skill-name}/
│
├── logs/                                 # electron-log
│   ├── main.log / renderer.log
│
├── locales/                              # i18n 翻译文件
└── crashpad/                             # Electron crash reports
```
