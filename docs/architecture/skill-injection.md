# Skill 注入 OpenClaw 机制

> 本文档描述 LobsterAI 如何将内置技能（SKILLs/ 目录）注入 OpenClaw 引擎。

## 概述

LobsterAI 通过两层机制将内置技能注入 OpenClaw：

1. **`openclaw.json` 配置** — 告知 OpenClaw 技能目录和启停状态
2. **环境变量** — `SKILLS_ROOT` 告知 OpenClaw 运行时去哪里扫描技能
3. **AGENTS.md 策略片段** — 嵌入 web-search / exec / memory 等运行时策略

## 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| SkillManager | `src/main/skillManager.ts` | 技能 CRUD、启停管理、变更通知 |
| OpenClawConfigSync | `src/main/libs/openclawConfigSync.ts` | 生成 `openclaw.json` 和 `AGENTS.md` |
| OpenClawEngineManager | `src/main/libs/openclawEngineManager.ts` | 启动 OpenClaw gateway 进程，注入环境变量 |
| main.ts | `src/main/main.ts` | 监听技能变更，触发重新同步 |

## 技能目录

技能物理存储在用户数据目录下，由 Electron `app.getPath('userData')` 决定：

| 平台 | 路径 |
|------|------|
| macOS | `~/Library/Application Support/LobsterAI/SKILLs/` |
| Windows | `%APPDATA%/LobsterAI/SKILLs/` |
| Linux | `~/.config/LobsterAI/SKILLs/` |

启动时 `syncBundledSkillsToUserData()` 将 app bundle 中打包的技能首次同步到用户目录；之后 `listSkills()` 扫描含 `SKILL.md` 的子目录，解析元数据（id、name、description、enabled 状态）。

`SKILLs/skills.config.json` 定义所有内置技能的默认启停状态和排序：

```json
{
  "defaults": {
    "docx":        { "order": 10,  "enabled": true },
    "web-search":  { "order": 15,  "enabled": true },
    "pdf":         { "order": 40,  "enabled": true },
    "skill-creator": { "order": 300, "enabled": true }
  }
}
```

## openclaw.json 中的 skills 配置

`OpenClawConfigSync.sync()` 生成完整的 `openclaw.json`，其中 `skills` 节点结构如下：

```jsonc
{
  "skills": {
    // 每项技能的启用/禁用覆盖
    "entries": {
      "docx":        { "enabled": true },
      "pdf":         { "enabled": true },
      "web-search":  { "enabled": true },
      // IM 渠道内置禁用列表
      "qqbot-cron":            { "enabled": false },
      "feishu-cron-reminder":  { "enabled": false }
    },
    // 告诉 OpenClaw 去哪里扫描 SKILL.md
    "load": {
      "extraDirs": ["~/Library/Application Support/LobsterAI/SKILLs"],
      "watch": true   // 热监控目录变化
    }
  }
}
```

关键生成逻辑在 `openclawConfigSync.ts`：

```typescript
// 构建每个技能的 enabled 状态
private buildSkillEntries(): Record<string, { enabled: boolean }> {
  const skills = this.getSkillsList?.() ?? [];
  const entries: Record<string, { enabled: boolean }> = {};
  for (const skill of skills) {
    entries[skill.id] = { enabled: skill.enabled };
  }
  return entries;
}

// 解析技能目录路径
private resolveSkillsExtraDirs(): string[] {
  const userDataSkillsDir = path.join(app.getPath('userData'), 'SKILLs');
  // ...
  return [userDataSkillsDir];
}
```

## 环境变量注入

`OpenClawEngineManager` 启动 OpenClaw gateway 进程时注入 `SKILLS_ROOT`：

```typescript
// openclawEngineManager.ts
const env: NodeJS.ProcessEnv = {
  ...process.env,
  SKILLS_ROOT:           skillsRoot,
  LOBSTERAI_SKILLS_ROOT: skillsRoot,
};
```

`getSkillsRoot()`（定义在 `coworkUtil.ts`）在打包环境下返回 `app.getPath('userData')/SKILLs`，开发环境下依次查找多个候选路径直到找到存在的目录。

## AGENTS.md 策略片段

`syncAgentsMd()` 在每次 `sync()` 调用时更新 OpenClaw 工作区的 `AGENTS.md`，写入内容包括：

- `MANAGED_WEB_SEARCH_POLICY_PROMPT` — web-search 策略
- `MANAGED_EXEC_SAFETY_PROMPT` — exec 安全约束
- `MANAGED_MEMORY_POLICY_PROMPT` — 内存策略
- 技能创建目录提示（告知模型在哪个路径创建新技能）
- 定时任务策略

> 注意：技能本身的路由提示已不再嵌入 AGENTS.md，改为由 OpenClaw 原生通过 `skills.load.extraDirs` 加载。

## 完整数据流

```
用户启停技能
    │
    ▼
SkillManager.notifySkillsChanged()
    │  向所有 BrowserWindow 广播 'skills:changed'
    │  调用所有已注册的 onSkillsChanged() 回调
    ▼
main.ts onSkillsChanged() 回调触发
    │
    ▼
syncOpenClawConfig({ reason: 'skills-changed' })
    │
    ▼
openClawConfigSync.sync()
    │
    ├── buildSkillEntries()
    │       → 从 SkillManager 拉取所有技能 { id, enabled }
    │
    ├── resolveSkillsExtraDirs()
    │       → 解析用户数据目录下的 SKILLs 路径
    │
    ├── 生成 openclaw.json
    │       skills.entries  = { ...buildSkillEntries(), ...MANAGED_SKILL_ENTRY_OVERRIDES }
    │       skills.load.extraDirs = [ ~/Library/.../LobsterAI/SKILLs ]
    │       skills.load.watch = true
    │
    └── syncAgentsMd()
            → 更新 OpenClaw 工作区 AGENTS.md
```

## 技能变更监听链路

`SkillManager` 通过 `changeListeners` 集合支持外部模块注册回调：

```typescript
// skillManager.ts
private notifySkillsChanged(): void {
  BrowserWindow.getAllWindows().forEach(win => {
    if (!win.isDestroyed()) {
      win.webContents.send('skills:changed');
    }
  });
  for (const listener of this.changeListeners) {
    try {
      listener();   // → 触发 main.ts 的 syncOpenClawConfig()
    } catch (error) {
      console.warn('[skills] onSkillsChanged listener error:', error);
    }
  }
}
```

`main.ts initApp()` 中注册回调：

```typescript
// main.ts:5060
manager.onSkillsChanged(() => {
  syncOpenClawConfig({ reason: 'skills-changed' }).catch((error) => {
    console.warn('[Main] Failed to sync OpenClaw config after skills change:', error);
  });
});
```

## 总结

LobsterAI 不通过代码直接注册技能，而是通过配置文件驱动：

1. **openclaw.json** — 声明技能启停 + 扫描目录，OpenClaw 原生技能系统读取
2. **环境变量 SKILLS_ROOT** — 进程级告知 OpenClaw 运行时技能根目录
3. **AGENTS.md** — 嵌入运行时策略和行为约束，不含技能路由逻辑

这种设计的优势：OpenClaw 自身具备完整的技能加载机制，LobsterAI 只需生成配置，无需感知技能的具体实现细节。
