# OpenClaw 插件系统深度分析

> 分析日期：2026-06-05
> 分析范围：`src/plugins/`、`src/plugin-sdk/`
> 文档目的：梳理插件系统的完整架构、加载流程、manifest 规范、钩子系统、SDK API 设计、内置 vs 外部插件机制、安全模型

---

## 目录

1. [插件系统架构总览](#1-插件系统架构总览)
2. [插件发现与加载流程](#2-插件发现与加载流程)
3. [Manifest 规范与解析](#3-manifest-规范与解析)
4. [钩子系统详解](#4-钩子系统详解)
5. [Plugin SDK API 设计分析](#5-plugin-sdk-api-设计分析)
6. [内置插件 vs 外部插件](#6-内置插件-vs-外部插件)
7. [权限与安全模型](#7-权限与安全模型)
8. [关键设计决策](#8-关键设计决策)
9. [生产环境预演](#9-生产环境预演)
10. [面试模拟](#10-面试模拟)
11. [建议添加注释的位置](#11-建议添加注释的位置)

---

## 1. 插件系统架构总览

### 1.1 核心概念

OpenClaw 的插件系统是一个 **manifest-driven（清单驱动）**、**分层加载**、**基于注册表**的插件架构。核心思想是：

- **Manifest 优先**：一切元数据（id、configSchema、kind、dependencies）从 `openclaw.plugin.json` 读取，无需加载模块即可知道插件能做什么。
- **延迟加载（Lazy Loading）**：Runtime 模块使用 Proxy 实现按需实例化，避免启动时加载所有运行时依赖。
- **分层注册表（Registry）**：所有插件注册的结果（tools、hooks、channels、providers 等）都汇聚到一个 `PluginRegistry` 对象中。
- **多级缓存**：manifest 加载缓存、registry 缓存、module loader 缓存，避免重复 I/O。

### 1.2 分层架构

```
┌────────────────────────────────────────────────────┐
│                   Plugin SDK                        │
│  (src/plugin-sdk/)                                  │
│  definePluginEntry / defineChannelPluginEntry       │
│  PluginRuntime / OpenClawPluginApi                  │
└──────────────┬─────────────────────────────────────┘
               │ 插件编写者使用 SDK 暴露 register()
               ▼
┌────────────────────────────────────────────────────┐
│               Plugin Loader                         │
│  (src/plugins/loader.ts)                            │
│  发现 -> Manifest 加载 -> 模块加载 -> register()    │
└──────────────┬─────────────────────────────────────┘
               │ 所有注册结果汇聚到 PluginRegistry
               ▼
┌────────────────────────────────────────────────────┐
│             Plugin Registry                         │
│  (src/plugins/registry.ts + registry-types.ts)      │
│  tools / hooks / channels / providers / services    │
└──────────────┬─────────────────────────────────────┘
               │ 运行时通过 getActivePluginRegistry() 读取
               ▼
┌────────────────────────────────────────────────────┐
│              Hook Runner                            │
│  (src/plugins/hooks.ts)                             │
│  按优先级执行插件的生命周期钩子                     │
│  void / modifying / claiming 三种模式               │
└────────────────────────────────────────────────────┘
```

### 1.3 数据流

```
配置 (openclaw.json)
     │
     ▼
发现 (discovery.ts) ──扫描目录──► candidates[] ──根据 manifest 过滤──► 排序、去重
     │
     ▼
Manifest 加载 (manifest.ts + manifest-registry.ts)
     │  读取 openclaw.plugin.json + package.json
     │  校验 config schema、kind、provider 声明
     ▼
模块加载 (loader.ts + plugin-module-loader-cache.ts)
     │  使用 jiti / native resolver 加载 JS/TS 入口
     │  解析模块导出 (resolvePluginModuleExport)
     ▼
注册 (registry.ts)
     │  执行插件 register(api)
     │  插件通过 api.registerTool/registerChannel 等注入能力
     ▼
激活 (runtime.ts)
     │  设置进程级全局注册表
     │  同步 hook runner
     ▼
运行时消费
     getActivePluginRegistry() -> tools/hooks/channels/providers...
```

---

## 2. 插件发现与加载流程

### 2.1 发现（Discovery）

`src/plugins/discovery.ts` 中的 `discoverOpenClawPlugins()` 是入口。发现有三层来源：

| 来源 | 优先级 | 说明 |
|------|--------|------|
| `stock`（内置） | 最高 | 由 `resolveBundledPluginsDir()` 解析的 dist/extensions 或 dist-runtime/extensions |
| `global`（全局） | 次之 | `~/.openclaw/extensions/`（用户级安装） |
| `workspace`（工作区） | 最低 | `{workspaceDir}/.openclaw/extensions/`（项目级） |

此外，`plugins.load.paths` 配置可以指定额外搜索路径，使用 `origin="config"` 标记。

**发现逻辑的关键点：**

- 每个目录先检查是否有 `package.json`，如果有则解析 `openclaw.extensions` 数组来定位入口文件
- 如果有 `openclaw.plugin.json`，用 manifest 中的 `id` 作为插件 ID
- bundle 格式插件通过 `detectBundleManifestFormat()` 检测
- 每个文件路径经过安全检查：`isUnsafePluginCandidate()` 验证路径是否逃逸根目录、是否 world-writable、所有权是否可疑
- 安装记录（install records）中的路径优先于自动扫描

### 2.2 Manifest 加载

`src/plugins/manifest.ts` 中的 `loadPluginManifest()`：

1. 查找 `openclaw.plugin.json`（仅此文件名，无变体）
2. 通过 `openRootFileSync` 安全地打开文件（验证路径不逃逸根目录）
3. 读取并解析 JSON（支持 JSON5）
4. 逐个字段校验/规范化（通过 `normalizeOptionalString`、`normalizeTrimmedStringList` 等）
5. 缓存结果到 LRU cache（最多 512 条，基于 `(dev, ino, size, mtimeMs, ctimeMs)` 的复合键）

**关键校验规则：**
- `id` 必须存在且非空
- `configSchema` 必须存在且为 JSON Schema Object
- 所有字符串列表经过 trim 和过滤空值
- `kind` 可以是一个字符串或字符串数组（表示多角色）
- `enabledByDefault` 可选，配合 `enabledByDefaultOnPlatforms` 实现平台感知的默认启用

### 2.3 模块加载

`src/plugins/loader.ts` 中的 `createPluginModuleLoader()`：

1. **SDK 别名解析**：通过 `resolvePluginSdkScopedAliasMap()` 创建 import 别名映射，确保插件中 `import { ... } from "openclaw/plugin-sdk"` 正确解析到宿主版本的 SDK
2. **Module Loader Cache**：使用 LRU cache 缓存加载器实例，避免重复创建 jiti 转换器
3. **入口路径检查**：通过 `openRootFileSync` 确保模块路径不会逃逸插件根目录
4. **解析模块导出**：`resolvePluginModuleExport()` 智能处理多种导出模式：
   - 直接导出 `register` 函数
   - 导出 `definePluginEntry()` 对象（含 `register`）
   - 导出 `{ default: ... }` 或 `{ module: ... }` 嵌套结构

### 2.4 注册与激活

`src/plugins/loader.ts` 中的 `loadOpenClawPlugins()`：

1. 解析配置上下文（`resolvePluginLoadCacheContext`）
2. 检查缓存是否可用
3. 如果缓存可用，直接恢复之前注册的状态（包括 commands、agent harnesses 等）
4. 如果缓存不可用，开始加载：
   a. 排序候选插件（按 duplicate 优先级）
   b. 遍历每个候选，解析激活状态
   c. 跳过禁用/重复/不匹配 Scope 的插件
   d. 加载模块入口 -> 解析 export -> 执行 `register()`
   e. 每一步都在 registry 快照的保护下进行，出错时回滚
5. 缓存结果
6. 激活注册表：`activatePluginRegistry()` 设置全局状态

### 2.5 注册模式（Registration Mode）

`PluginRegistrationMode` 控制插件在什么场景下注册：

| 模式 | 说明 |
|------|------|
| `full` | 完整激活，执行所有注册 |
| `discovery` | 仅发现/探测模式，不激活运行时能力 |
| `setup-only` | 仅加载 setup entry，用于渠道插件的预监听启动 |
| `setup-runtime` | 加载 setup entry + runtime entry 的合并 |
| `tool-discovery` | 仅工具发现 |
| `cli-metadata` | 仅 CLI 命令注册 |

### 2.6 缓存机制

插件系统实现了**三层 LRU 缓存**：

| 缓存 | 最大条目 | 位置 | 关键 |
|------|---------|------|------|
| Manifest Load Cache | 512 | `manifest.ts` | 基于 `(dev, ino, size, mtimeMs, ctimeMs)` |
| Registry Cache | 128 | `loader.ts` | 基于配置指纹（`buildCacheKey`） |
| Module Loader Cache | 由 LRU 控制 | `plugin-module-loader-cache.ts` | 基于模块路径 |

**缓存失效策略：**
- Registry cache 的 key 包含：workspace 路径、所有插件配置、激活元数据 hash、install records、env、devSourceRoot、模式标志等 20+ 维度
- 任何配置变化都会导致缓存键变化

---

## 3. Manifest 规范与解析

### 3.1 清单文件

每个插件必须有 `openclaw.plugin.json`，位置在插件根目录。核心字段：

```jsonc
{
  "id": "plugin-id",                    // 必填，全局唯一
  "configSchema": { ... },              // 必填，JSON Schema
  "kind": "memory",                     // 可选，插件角色（memory | context-engine）
  "channels": ["telegram"],             // 可选，拥有的渠道
  "providers": ["openai"],              // 可选，拥有的模型提供商
  "requiresPlugins": ["memory-core"],   // 可选，依赖的插件
  "enabledByDefault": true,            // 可选，默认启用
  "enabledByDefaultOnPlatforms": ["darwin"],  // 可选，平台感知默认
  "activation": {                       // 可选，条件激活规则
    "onStartup": true,
    "onProviders": ["openai"],
    "onChannels": ["telegram"]
  },
  "setup": {                           // 可选，安装/设置元数据
    "providers": [{ "id": "openai", "authMethods": ["api-key"] }]
  },
  "contracts": {                       // 可选，声明的能力所有权
    "tools": ["web_search"],
    "embeddingProviders": ["openai"]
  },
  "commandAliases": [                  // 可选，命令别名
    { "name": "/gpt", "provider": "openai", "model": "gpt-5.4" }
  ]
}
```

### 3.2 package.json 扩展字段

插件也可通过 `package.json` 的 `"openclaw"` 字段声明元数据：

```jsonc
{
  "name": "@org/my-plugin",
  "openclaw": {
    "extensions": ["./dist/index.js"],   // 入口文件列表
    "setupEntry": "./dist/setup-entry.js", // 可选的 setup 入口
    "runtimeSetupEntry": "...",          // 可选的 runtime setup 入口
    "channel": {                         // 渠道配置
      "id": "my-channel",
      "label": "My Channel"
    },
    "plugin": {
      "id": "my-plugin",
      "label": "My Plugin"
    },
    "compat": {                          // SDK 兼容性声明
      "pluginApi": ">=1.0.0"
    },
    "install": {                         // 安装配置
      "npmSpec": "@org/my-plugin",
      "minHostVersion": ">=1.0.0"
    }
  }
}
```

### 3.3 规范化流程

`src/plugins/manifest.ts` 中的 `loadPluginManifest()` 读取清单后，通过**大量细粒度的规范化函数**对每个字段进行消毒：

- 字符串列表去掉空值、trimmed
- 对象键通过 `isBlockedObjectKey()` 过滤原型污染风险
- 模型 ID 通过 `normalizeModelCatalogProviderId()` 标准化
- 数字范围通过 `normalizeManifestPositiveInteger()` 做边界检查
- 废弃/旧字段（如 `providerAuthEnvVars`）触发 deprecation warning 但仍然兼容

---

## 4. 钩子系统详解

### 4.1 架构

`src/plugins/hooks.ts` 中的 `createHookRunner()` 创建钩子运行器，负责执行插件注册的各类生命周期钩子。

**执行模型有三种：**

| 模式 | 说明 | 执行方式 | 典型用途 |
|------|------|----------|----------|
| **Void**（Fire-and-forget） | 观察者模式，不返回值 | 并行执行 | `agent_end`, `message_received`, `gateway_start` |
| **Modifying**（顺序合并） | 可修改事件数据 | 按优先级顺序，结果合并 | `before_prompt_build`, `message_sending` |
| **Claiming**（优先胜出） | 首个 `handled: true` 胜出 | 按优先级顺序，第一个处理就停止 | `inbound_claim`, `before_agent_reply` |

### 4.2 钩子类别

钩子按触发时机分为六类：

#### Agent 生命周期钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `before_agent_run` | Modifying | 最严格的 gate 决策，返回 `outcome: "pass" | "block"` |
| `before_model_resolve` | Modifying | 覆盖 provider/model 选择 |
| `before_prompt_build` | Modifying | 注入 system prompt / prependContext |
| `before_agent_start` | Modifying | 废弃，组合 before_model_resolve + before_prompt_build |
| `agent_turn_prepare` | Modifying | 注入 turn 级别的上下文 |
| `before_agent_reply` | Claiming | 拦截消息并返回合成回复 |
| `model_call_started` | Void | 观察模型调用开始 |
| `model_call_ended` | Void | 观察模型调用结束 |
| `llm_input` | Void | 观察 LLM 输入 |
| `llm_output` | Void | 观察 LLM 输出 |
| `before_agent_finalize` | Modifying | 请求一次额外的模型传递 |
| `agent_end` | Void | 分析已完成会话 |
| `before_compaction` | Void | 压缩前 |
| `after_compaction` | Void | 压缩后 |
| `before_reset` | Void | /reset 或 /new 清空会话前 |

#### 消息钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `inbound_claim` | Claiming | 在命令/Agent 调度前声明入站事件 |
| `message_received` | Void | 消息到达 |
| `before_dispatch` | Claiming | 在模型调度前处理消息 |
| `reply_dispatch` | Claiming | 拥有回复调度 |
| `reply_payload_sending` | Modifying | 修改或取消回复 payload |
| `message_sending` | Modifying | 修改或取消发送中的消息 |
| `message_sent` | Void | 消息已发送 |

#### 会话钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `session_start` | Void | 会话开始 |
| `session_end` | Void | 会话结束 |

#### Subagent 钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `subagent_spawning` | Modifying | 子 Agent 创建（已废弃） |
| `subagent_delivery_target` | Modifying | 子 Agent 投递路由 |
| `subagent_spawned` | Void | 子 Agent 已创建 |
| `subagent_ended` | Void | 子 Agent 已结束 |

#### Gateway 钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `gateway_start` | Void | Gateway 启动 |
| `gateway_stop` | Void | Gateway 停止 |
| `heartbeat_prompt_contribution` | Modifying | 心跳 prompt 贡献 |
| `cron_changed` | Void | Cron 生命周期变化 |

#### 工具钩子

| 钩子名 | 模式 | 说明 |
|--------|------|------|
| `before_tool_call` | Modifying | 修改或拦截工具调用 |
| `after_tool_call` | Void | 工具调用后 |
| `tool_result_persist` | Sync-Modifying | 同步运行，不能返回 Promise |
| `before_message_write` | Sync-Modifying | 同步运行，消息写入前 |

### 4.3 优先级

钩子按 `priority` 字段排序（数值越大优先级越高）。高优先级的 modifying hook 先执行，其结果可能影响后续 hook 的输入。Void hook 虽然并行执行，但优先级影响注册时的顺序。

### 4.4 超时策略

- **void hooks**：根据 `DEFAULT_VOID_HOOK_TIMEOUT_MS_BY_HOOK`，`agent_end`、`before_compaction`、`after_compaction` 默认 30 秒超时
- **modifying hooks**：`before_agent_run`、`before_agent_start`、`before_prompt_build` 默认 15 秒
- 超时后 runner 通过 `fail-open` 策略继续执行（catch 并 log）
- `before_agent_run` 默认 `fail-closed`（钩子失败会被 throw）

### 4.5 同步钩子的设计考量

`tool_result_persist` 和 `before_message_write` 是唯二使用同步运行的钩子。在 `src/plugins/hooks.ts:1316-1358` 和 `1376-1430` 中，它们：
- 不 await 任何异步操作
- 检测到返回 Promise 时会 log warning 并跳过
- 在 session transcript 的热路径上顺序执行

---

## 5. Plugin SDK API 设计分析

### 5.1 SDK 入口结构

`src/plugin-sdk/index.ts` 是 SDK 的主入口，但**故意保持极小**。其设计原则（参见 `src/plugin-sdk/CLAUDE.md`）：

> "Keep public SDK entrypoints cheap at module load."

主入口只 export type 和轻量帮助函数。实际运行时能力通过专用子路径暴露：

| 子路径 | 内容 | 加载成本 |
|--------|------|----------|
| `openclaw/plugin-sdk` | 类型 + 轻量 helpers | 低 |
| `openclaw/plugin-sdk/runtime` | 运行时工具（logger、exit、backup） | 中 |
| `openclaw/plugin-sdk/core` | 渠道插件定义 + 聊天渠道 builder | 高 |

### 5.2 插件定义方式

**非渠道插件**使用 `definePluginEntry()`（`src/plugin-sdk/plugin-entry.ts:307`）：

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Does something",
  configSchema: buildPluginConfigSchema({ ... }),
  register: (api) => {
    api.registerTool({ ... });
    api.registerHook({ ... });
  },
});
```

**渠道插件**使用 `defineChannelPluginEntry()`（`src/plugin-sdk/core.ts:540`）：

```typescript
export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "A messaging channel",
  plugin: { ... },
  registerFull: (api) => {
    // 完整激活时额外注册
  },
});
```

**Setup-only 渠道插件**使用 `defineSetupPluginEntry()`（`src/plugin-sdk/core.ts:594`）：

```typescript
export default defineSetupPluginEntry({
  plugin: { ... }
});
```

### 5.3 PluginRuntime 设计

`src/plugins/runtime/types.ts` 定义的 `PluginRuntime` 是一个**轻量代理对象**：

```typescript
type PluginRuntime = PluginRuntimeCore & {
  subagent: { run, waitForRun, getSessionMessages, deleteSession };
  nodes: { list, invoke };
  channel: PluginRuntimeChannel;
};
```

在 `src/plugins/loader.ts:1863` 中，runtime 通过 Proxy 实现**延迟实例化**：

```typescript
const runtime = new Proxy({} as PluginRuntime, {
  get(_target, prop, receiver) {
    return Reflect.get(resolveRuntime(), prop, receiver);
  },
  // ...
});
```

这意味着一系列"懒反射键"（lazyRuntimeReflectionKeys = version, config, agent, subagent, ...）在首次访问时才触发 `createPluginRuntime()` 的完整调用链。

### 5.4 OpenClawPluginApi 设计

`src/plugins/types.ts` 定义的 `OpenClawPluginApi` 是插件与宿主交互的唯一通道。它暴露 50+ 个注册方法，涵盖：

| 类别 | 方法 | 数量 |
|------|------|------|
| Tool | registerTool | 1 |
| Hook | registerHook, on | 2 |
| Provider | registerProvider, registerModelCatalogProvider, ... | 20+ |
| Channel | registerChannel | 1 |
| Service | registerService, registerGatewayDiscoveryService | 2 |
| Command | registerCommand, registerCli | 2 |
| Agent | registerAgentHarness, registerAgentEventSubscription | 4 |
| Session | registerSessionExtension, registerSessionSchedulerJob, ... | 5 |
| Memory | registerMemoryCapability, registerMemoryEmbeddingProvider, ... | 6 |
| Runtime | registerReload, registerRuntimeLifecycle | 2 |
| Security | registerSecurityAuditCollector | 1 |

**API 守卫（Guard）机制**（`src/plugins/loader.ts:504-536`）：

```typescript
function createGuardedPluginRegistrationApi(api: OpenClawPluginApi) {
  let closed = false;
  const guardedApi = new Proxy(api, {
    get(target, prop, receiver) {
      if (typeof value !== "function") return value;
      if (isLateCallableMethod) return (...args) => Reflect.apply(value, target, args);
      return (...args) => {
        if (closed) return undefined;  // register 完成后所有调用变为 noop
        return Reflect.apply(value, target, args);
      };
    },
  });
  return { api: guardedApi, close: () => { closed = true; } };
}
```

这意味着：
- `register()` 返回后，API 立即失效（`closed = true`）
- `isLateCallablePluginApiMethod()` 白名单内的方法（如 `registerReload`）仍然可用
- 这种设计防止插件在 `register()` 之外意外调用 API 导致副作用

### 5.5 SDK 类型系统

SDK 大量导出类型接口，让插件可以 TypeScript 类型安全地定义自己。关键类型包括：

- `OpenClawPluginDefinition`：插件的完整定义（含 id、register、reload、nodeHostCommands）
- `OpenClawPluginApi`：注册 API 接口
- `PluginRuntime`：运行时能力接口
- `ProviderPlugin`：统一模型提供商插件接口（含 30+ 生命周期方法）
- `ChannelPlugin`：渠道插件的统一接口

---

## 6. 内置插件 vs 外部插件

### 6.1 分类依据（PluginOrigin）

`src/plugins/plugin-origin.types.ts` 定义了四种来源：

| Origin | 路径来源 | 特点 |
|--------|----------|------|
| `bundled` | `dist/extensions/` 或 `dist-runtime/extensions/` | 官方内置，高信任 |
| `global` | `~/.openclaw/extensions/` | 用户安装的外部插件 |
| `workspace` | `{workspaceDir}/.openclaw/extensions/` | 项目级特有插件 |
| `config` | `plugins.load.paths` 配置指定 | 显式声明的插件路径 |

### 6.2 处理差异

内置插件和外部插件在以下方面有差异：

1. **安全检查**：内置插件不会被检查文件所有权（`discovery.ts:175`），外部插件会检查 uid
2. **硬链接检测**：通过 `shouldRejectHardlinkedPluginFiles()` 决定是否拒绝硬链接文件
3. **内置插件边界**：内置插件的 `package.json` 读取使用 `readTrustedPackageManifest()`（直接 `tryReadJsonSync`），外部插件使用 `readPackageManifest()`（通过 `openRootFileSync` 做路径逃逸检查）
4. **启动优化**：内置插件可以优先加载预编译的 JS 产物而非源码 TS（`preferBuiltPluginArtifacts` 选项）
5. **插件 SDK 版本检查**：外部插件跳过 `shouldSkipIncompatiblePackagePluginApi()` 时做 `satisfiesPluginApiRange` 检测

### 6.3 内置插件的特殊路径解析

`src/plugins/bundled-dir.ts` 中的 `resolveBundledPluginsDir()` 负责解析内置插件目录：

1. 检查环境变量 `OPENCLAW_DISABLE_BUNDLED_PLUGINS` 可关闭内置插件
2. 检查环境变量 `OPENCLAW_BUNDLED_PLUGINS_DIR` 可覆盖路径
3. 自动探测路径顺序：`dist-runtime/extensions/` > `dist/extensions/` > `extensions/`（源码检出）
4. 使用进程级缓存（`bundledPluginsDirCache`）避免重复解析

### 6.4 内置插件 vs 外部插件的 runtime 加载优化

`loader.ts` 中的 `resolvePreferredBuiltRuntimeArtifact()` 为内置插件执行特殊优化：

- 对于内置插件，优先从 `dist-runtime/` 或 `dist/` 加载编译产物
- 对于外部插件，优先从包内 `dist/` 加载
- 避免加载 source map、测试文件、TypeScript 源码

---

## 7. 权限与安全模型

### 7.1 文件系统安全

`src/plugins/discovery.ts` 实现了多层文件级安全检查：

**路径逃逸检查**（`checkSourceEscapesRoot`）：
- 使用 `realpath` 解析符号链，验证入口文件在插件根目录下
- 防止插件通过符号链接读取宿主文件系统

**权限检查**（`checkPathStatAndPermissions`）：
- 检查目录和文件是否 world-writable（0402 模式）
- 非内置插件检查文件 uid 是否与当前进程 uid 一致（或 root）
- 内置插件 world-writable 会尝试修复（chmod），修复失败则阻止加载

### 7.2 内存插槽排他性

`src/plugins/slots.ts` 实现了"插槽排他"机制：

- `memory` 和 `context-engine` 两种插槽类型（`SLOT_BY_KIND`）
- 一个插槽只能有一个活跃插件
- 选择新的内存插件时，自动禁用旧插槽中的其他插件
- 提供"做梦引擎"特例：`memory-core` 即使不是选定插槽也可以加载，支持 sidecar 模式

### 7.3 配置安全

- **Config Schema 验证**：插件必须提供 JSON Schema，运行时使用 `validateJsonSchemaValue` 校验 `plugins.entries.<id>.config`
- **危险配置标志**：`PluginManifestDangerousConfigFlag` 机制允许插件标记某些配置值为危险的
- **SecretRef**：配置中的密码引用通过 `PluginManifestSecretInputPath` 声明，宿主统一解析

### 7.4 激活策略

`src/plugins/config-state.ts` 的激活决策模型：

```
插件启用决策 = 用户配置 ? 用户决定 : (默认启用 ? 开启 : (条件激活 ? 开启 : 关闭))
```

- 用户可通过 `plugins.enabled`、`plugins.entries.<id>.enabled`、`plugins.allow`/`plugins.deny` 控制
- manifest 中的 `enabledByDefault` 和 `enabledByDefaultOnPlatforms` 声明默认行为
- 条件激活通过 `activation.onProviders` / `activation.onChannels` 等声明
- `autoEnableWhenConfiguredProviders` 声明：当用户在配置中引用某 provider 时自动启用此插件

### 7.5 Bundle 插件限制

Bundle 格式插件（Claude Code / Codex / Cursor 等外部格式）仅支持有限的能力：

- `skills`、`mcpServers`、`settings`、`commands`、`agents`、`outputStyles`、`lspServers`
- 不支持 `tools`、`hooks`（除 codex/claude 的 hooks）
- 不执行 JS/TS 代码，仅读取元数据

---

## 8. 关键设计决策

参考 `src/plugins/CLAUDE.md` 和 `src/plugin-sdk/CLAUDE.md`，以下是最重要的设计原则：

### 8.1 Manifest 优先（Manifest-First）

> "Preserve manifest-first behavior: discovery, config validation, and setup should work from metadata before plugin runtime executes."

这意味着：
- 插件清单（`openclaw.plugin.json`）在模块加载前就已读取
- Config Schema 在 `register()` 前就校验
- Provider 声明、渠道声明、命令别名等都在不执行任何插件代码的情况下可用

### 8.2 控制面与运行面分离

> "Keep control-plane and runtime-plane concerns separate."

- **控制面**：发现、清单解析、配置验证、激活规划
- **运行面**：插件代码执行、provider 能力、渠道适配

控制面永远不会主动导入运行面模块。

### 8.3 懒加载惰性原则

> "Preserve laziness in discovery and activation flows."

- Runtime 通过 Proxy 延迟实例化
- Module loader 只在有活跃插件需要加载时才创建
- channel plugins 的 full runtime 可以 defer 到 listen 之后（`deferConfiguredChannelFullLoadUntilAfterListen`）

### 8.4 不在核心硬编码插件策略

> "Core stays plugin-agnostic. No bundled ids/defaults/policy in core when manifest/registry/capability contracts work."

- 核心不假设任何插件 ID 的存在（唯一例外是 memory slot 的 default `memory-core`）
- 所有启用策略来自 manifest 和用户配置
- 插件自身的配置只能通过 `plugins.entries.<id>.config` 访问，核心不会直接读取

### 8.5 全局状态是兼容性支架

> "Treat mutable global runtime registry state as compatibility scaffolding."

- `getActivePluginRegistry()` 返回的全局注册表应该被视为兼容性过渡方案
- 新增的运行时路径应该优先使用不可变或 request-scoped 的句柄

---

## 9. 生产环境预演

### 9.1 潜在风险与改进建议

#### 风险 1：全局注册表单点故障

`src/plugins/runtime.ts` 中的 `state` 对象是进程级单例。在高并发场景下，`setActivePluginRegistry()` 和 `getActivePluginRegistry()` 之间没有锁：

```
// 风险：Thread A 调用 setActivePluginRegistry(newRegistry)
// 同时 Thread B 调用 getActivePluginRegistry() -> 可能得到空指针
```

实际上 Node.js 是单线程事件循环，这不构成竞态风险。但**如果未来迁移到 Worker Threads**，全局 `state` 会跨线程共享，需要改为 `SharedArrayBuffer` 或 thread-local storage。

#### 风险 2：插件间 state 冲突

`clearActivatedPluginRuntimeState()` 会清除所有插件的全局注册状态（agent harnesses、commands、embedding providers 等）。如果两个 registry 加载过程重叠：

```typescript
// loader.ts:1793
if (shouldActivate) {
  clearActivatedPluginRuntimeState();  // 可能清空另一个正在使用的 registry
}
```

虽然 loader 有 `isPluginRegistryLoadInFlight()` 防止递归，但并发的 registry 加载请求仍然可能互相破坏对方的状态。

**建议**：registry 切换使用引用计数或快照事务。

#### 风险 3：内存泄漏

- `bundledPackageCacheIdentityByStockRoot`（`loader.ts:858`）是一个不会自动清理的永久 Map
- `packageManifestProcessCache`（`discovery.ts:61`）虽然有 512 条限制，但 LRU 策略仅删除最早的条目
- `pluginManifestLoadCache` 使用 LRU，但监听了 `(dev, ino, size, mtimeMs, ctimeMs)` 五个维度，键空间可能膨胀

**建议**：
- 为 `bundledPackageCacheIdentityByStockRoot` 添加 WeakRef 或定期清理机制
- Manifest cache 如果遇到文件系统变更频繁的场景（docker + volume mount），`mtimeMs` 精度可能不足，导致键不命中。可考虑加入 `mtimeNs`。

#### 风险 4：钩子超时导致 hang

`src/plugins/hooks.ts` 中的 void hooks 默认 timeout 15-30 秒。如果一个插件 hang 住了：

- Void hook 的超时不会取消底层操作，只是跳过结果
- 这意味着插件内部可能永远运行着一个计时器或子进程

**建议**：超过超时后使用 `AbortController` 实际取消插件的工作。

#### 风险 5：插件 API 回退不完整

`createGuardedPluginRegistrationApi` 在 `register()` 结束后关闭 API，但如果插件持有 API 引用（例如闭包中的 `api.runtime`），调用 `register()` 后修改 runtime 属性的操作会被静默忽略：

```typescript
// 插件代码
const myRuntime = api.runtime;
setTimeout(() => {
  myRuntime.subagent.run(...)  // 这没问题，因为 runtime 不是 API 的一部分
}, 1000);
api.runtime.something = "value"; // 被静默忽略??? 不，runtime 是 property 不是 method
```

守卫只拦截方法调用，属性访问不受影响。但如果 `isLateCallablePluginApiMethod` 未正确配置，某些调用可能被静默忽略。

### 9.2 高并发场景建议

1. **Registry 读取优化**：考虑将 `PluginRegistry` 设计为 `Object.freeze()` 后的不可变对象
2. **钩子 Runner 隔离**：每个会话应该有自己的 hook runner 副本，避免相互影响
3. **模块加载器缓存独立**：考虑为不同的 `devSourceRoot` 或 `pluginSdkResolution` 创建独立的缓存实例

---

## 10. 面试模拟

### Q1: 请描述 OpenClaw 插件系统的整体加载流程（从扫描文件系统到插件激活）

**参考答案**：

加载流程分为四个阶段：

1. **发现**（`discovery.ts`）：按优先级扫描三个目录（bundled > global > workspace），并处理 `plugins.load.paths` 中的自定义路径。每个目录检查 `package.json`（解析 `openclaw.extensions`）和 `openclaw.plugin.json`（读取 manifest）。生成 `PluginCandidate[]`。

2. **Manifest 注册**（`manifest-registry.ts`）：对每个候选加载其 `openclaw.plugin.json`，校验必填字段（`id`、`configSchema`），规范化所有可选字段，生成 `PluginManifestRegistry`。

3. **加载与注册**（`loader.ts`）：对每个启用的插件，按优先级排序、去重。使用 `createPluginModuleLoader()` 加载模块入口，通过 `resolvePluginModuleExport()` 解析 export，然后调用 `runPluginRegisterSync(register, api)` 执行插件的 `register()` 函数。整个过程在 registry 快照保护下，出错时自动回滚。

4. **激活**（`runtime.ts`）：所有插件注册完成后，调用 `activatePluginRegistry()` 设置全局注册表，同步 hook runner，缓存结果。

### Q2: Plugin SDK 设计中的"Lazy Runtime Proxy"是如何实现的？解决了什么问题？

**参考答案**：

`src/plugins/loader.ts:1848-1888` 中创建了一个 `PluginRuntime` 的 Proxy：

- 维护一个 `LAZY_RUNTIME_REFLECTION_KEYS` 数组（`["version", "config", "agent", "subagent", ...]`）
- Proxy 的 `get`/`set`/`has` 等 trap 只在首次访问这些 key 时触发 `resolveRuntime()` 的完整初始化
- 实际的 `createPluginRuntime()` 是在访问 `runtime.version` 或 `runtime.config` 时才调用的

解决的问题：
- **启动时间**：Gateway 启动时只加载 manifest，不需要实例化运行时上下文
- **依赖图**：不触发渠道、provider 等重型模块的递归导入
- **测试友好**：测试中可以通过 `runtime = {} as PluginRuntime` 注入简单 mock

### Q3: Hook 系统设计了三种执行模式（Void / Modifying / Claiming），请分别说明其适用场景和实现机制

**参考答案**：

1. **Void（Fire-and-forget）**：观察者模式，所有 handler 并行执行，不关心返回值。
   - 适用：日志、监控、统计
   - 实现：`Promise.all(hooks.map(async hook => { ... }))`
   - 超时处理：fail-open，超时后 log 并继续

2. **Modifying（顺序合并）**：按优先级降序执行，handler 可以修改事件数据，结果合并后传递给下一个 handler。
   - 适用：修改 prompt、注入上下文、拦截消息
   - 实现：`for (const hook of sortedHooks) { result = merge(result, await handler(event, ctx)) }`
   - 合并策略：`mergeResults` 函数由各钩子自定义（firstDefined / lastDefined / concat 等）
   - 可提前终止：`shouldStop` 函数判断是否跳过剩余 handler

3. **Claiming（优先胜出）**：按优先级降序执行，首个返回 `{ handled: true }` 的 handler 胜出并停止。
   - 适用：入站消息路由、回复拦截
   - 实现：遍历 hooks，`if (handlerResult?.handled) return handlerResult`
   - 竞态安全：高优先级的插件优先处理

### Q4: 什么是 Manifest-First 设计？它带来了哪些架构优势？

**参考答案**：

Manifest-First 意味着插件系统可以在**不加载任何插件代码**的情况下，仅通过 `openclaw.plugin.json` 的元数据就决定哪些插件应该启用、如何配置、依赖什么。

优势：
1. **快速启动筛选**：启动时不需要 import 所有插件模块就能知道哪些要激活
2. **配置提前校验**：manifest 中声明的 `configSchema` 可以在注册前就验证用户配置
3. **轻量级探测**：CLI 可以运行 `--validate` 模式只对 manifest 做校验而不加载模块
4. **声明式能力路由**：manifest 中的 `contracts`、`activation` 等信息可以用于条件激活，无需执行插件代码
5. **依赖图提前解析**：`requiresPlugins`、`autoEnableWhenConfiguredProviders` 等依赖关系可以在加载前构建

这一设计体现在 `loader.ts:2274` 的逻辑中：当 `shouldLoadModules` 为 false 时，loader 可以做"manifest-only"的轻量级加载，跳过 module import。

### Q5: 插件系统的缓存方案是如何设计的？缓存失效的策略是什么？

**参考答案**：

插件系统有三层 LRU 缓存：

1. **Manifest Load Cache**（`manifest.ts`）：最多 512 条目。键基于 `(dev, ino, size, mtimeMs, ctimeMs)`，确保文件内容变化时缓存失效。

2. **Registry Cache**（`loader.ts`）：最多 128 条目。键通过 `buildCacheKey()` 生成，包含 20+ 维度：workspace 路径、所有插件配置、激活元数据 hash、install records、env、devSourceRoot、运行时子 Agent 模式等。任何配置变化都会导致缓存键变化。

3. **Module Loader Cache**（`plugin-module-loader-cache.ts`）：基于模块路径的 LRU cache，缓存 jiti 编译器实例。

缓存还能**复用**：`getReusableCachedPluginRegistry()` 尝试在 "gateway-bindable" 和 "default" 模式之间共享缓存。例如 gateway 启动时以 "gateway-bindable" 模式加载，后续工具发现请求可以使用 "default" 模式的缓存，只要配置指纹匹配。

---

## 11. 建议添加注释的位置

以下位置需要添加/补充中文注释以帮助后来者理解：

| 文件 | 行号 | 建议 |
|------|------|------|
| `src/plugins/loader.ts` | 504-536 | `createGuardedPluginRegistrationApi` - 解释 why API 需要在 register 后关闭 |
| `src/plugins/loader.ts` | 1848-1862 | `lazyRuntimeReflectionKeySet` 和 `resolveLazyRuntimeDescriptor` - 解释 Proxy 延迟初始化机制 |
| `src/plugins/loader.ts` | 1169-1247 | `resolvePluginRegistrationPlan` - 解释为什么需要这么多注册模式，各自的决策树 |
| `src/plugins/hooks.ts` | 186-205 | `HookRunnerOptions` - 解释 fail-open vs fail-closed 的设计考量 |
| `src/plugins/hooks.ts` | 207-233 | DEFAULT_VOID_HOOK_TIMEOUT_MS 和 DEFAULT_MODIFYING_HOOK_TIMEOUT_MS - 解释超时值为什么这么选 |
| `src/plugins/discovery.ts` | 1426-1629 | `discoverOpenClawPlugins` - 解释 scoped 和 shared 两个阶段的差异，为什么这样分离 |
| `src/plugins/manifest.ts` | 1698-1883 | `loadPluginManifest` - 解释 cache 键的选择（dev, ino）和文件参数检查 |
| `src/plugins/registry.ts` | 编写位置 | `createPluginRegistry` - 解释 registry 中每个数组的作用 |
| `src/plugins/runtime.ts` | 24-52 | `state` 初始化 - 解释为什么使用 `globalThis` 做进程级单例 |
| `src/plugins/slots.ts` | 12-21 | `SLOT_BY_KIND` 和 `DEFAULT_SLOT_BY_KEY` - 解释为什么 memory slot 默认是 memory-core |
| `src/plugin-sdk/plugin-entry.ts` | 307-332 | `definePluginEntry` - 解释 configSchema 为什么使用 `createCachedLazyValueGetter` |
| `src/plugin-sdk/core.ts` | 540-586 | `defineChannelPluginEntry` - 解释 setup-runtime 和 full 模式的差异 |
| `src/plugins/package-compat.ts` | N/A | 查找 `satisfiesPluginApiRange` 调用处，解释版本兼容性检测 |

---

## 附录：关键文件索引

| 文件 | 角色 |
|------|------|
| `src/plugins/loader.ts` | 主加载器，3221 行，包含加载、注册、激活的核心逻辑 |
| `src/plugins/discovery.ts` | 插件发现，扫描文件系统生成候选 |
| `src/plugins/manifest.ts` | Manifest 定义与加载，2059 行 |
| `src/plugins/manifest-registry.ts` | Manifest 注册表聚合 |
| `src/plugins/hooks.ts` | Hook 运行器，1685 行，3 种执行模式 |
| `src/plugins/runtime.ts` | 运行时状态管理，进程级单例注册表 |
| `src/plugins/registry.ts` | 注册表创建、API 构建、工具/钩子/渠道注册 |
| `src/plugins/registry-types.ts` | 注册表类型定义，502 行 |
| `src/plugins/slots.ts` | 插件插槽排他性控制 |
| `src/plugins/config-state.ts` | 配置规范化、启用决策、激活状态 |
| `src/plugins/bundled-dir.ts` | 内置插件目录解析 |
| `src/plugins/roots.ts` | 根目录解析（stock / global / workspace） |
| `src/plugins/types.ts` | 插件核心类型定义 |
| `src/plugin-sdk/index.ts` | SDK 主入口，保持极小的类型导出 |
| `src/plugin-sdk/core.ts` | SDK 核心，848 行，渠道插件定义 |
| `src/plugin-sdk/runtime.ts` | SDK 运行时工具 |
| `src/plugin-sdk/plugin-entry.ts` | definePluginEntry 实现 |
