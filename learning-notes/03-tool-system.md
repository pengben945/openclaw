# OpenClaw 工具系统深度分析

## 1. 系统概述

OpenClaw 的工具系统是一个 **多阶段流水线（Pipeline）架构**，负责将零散的工具定义（核心工具、Shell 工具、插件工具、MCP 工具）组装成最终呈现给 LLM 的 `ToolDefinition[]` 数组。核心思路是：先收集、再过滤、最后增强。

### 架构图

```
+------------------------------------------------------------------+
|                   工具装配中心 (agent-tools.ts)                    |
|     createOpenClawCodingTools() - 主入口函数                       |
+------------------------------------------------------------------+
                              |
         +--------------------+--------------------+
         |                    |                    |
         v                    v                    v
  基础编码工具           Shell 工具          OpenClaw 工具
  (read/write/edit)    (exec/process)     (message/sessions/...)
         |                    |                    |
         |              +-----+-----+              |
         |              |           |              |
         |         Plugin 工具   MCP 工具          |
         |        (extensions)  (外部服务器)       |
         |                    |                    |
         +---------+----------+--------------------+
                   |
        +----------+-----------+
        |     策略过滤 Pipeline  |
        | (tool-policy-pipeline) |
        +----------+-----------+
                   |
        +----------+-----------+
        |   Schema 规范化       |
        | (按 provider 调整)    |
        +----------+-----------+
                   |
        +----------+-----------+
        |   Before-Tool-Call   |
        |   Hook 包装层         |
        | (循环检测/审批/诊断)  |
        +----------+-----------+
                   |
        +----------+-----------+
        |   AbortSignal 包装    |
        +----------+-----------+
                   |
                   v
          ToolDefinition[]
          (最终给 LLM runtime)
```

---

## 2. 工具类型（ToolDescriptor 体系）

`src/tools/types.ts` 定义了核心的类型体系：

### ToolDescriptor（工具描述符）
```typescript
type ToolDescriptor = {
  name: string;
  description: string;
  inputSchema: JsonObject;
  owner: ToolOwnerRef;        // 谁拥有的
  executor?: ToolExecutorRef; // 谁执行的
  availability?: ToolAvailabilityExpression; // 可用性条件
};
```

### 拥有者类型 (ToolOwnerRef)
- `{ kind: "core" }` - 核心内置工具
- `{ kind: "plugin"; pluginId }` - 插件工具
- `{ kind: "channel"; channelId }` - 频道工具
- `{ kind: "mcp"; serverId }` - MCP 服务器工具

### 执行者类型 (ToolExecutorRef)
- `{ kind: "core"; executorId }`
- `{ kind: "plugin"; pluginId; toolName }`
- `{ kind: "channel"; channelId; actionId }`
- `{ kind: "mcp"; serverId; toolName }`

### 可用性表达式 (ToolAvailabilityExpression)
支持逻辑组合：
- `{ kind: "always" }` - 始终可用
- `{ kind: "auth"; providerId }` - 需要某个认证
- `{ kind: "config"; path }` - 需要某配置项
- `{ kind: "env"; name }` - 需要环境变量
- `{ kind: "plugin-enabled"; pluginId }` - 需要某插件启用
- `{ kind: "context"; key; equals? }` - 需要上下文匹配
- `{ allOf: [...] }` - 全部满足
- `{ anyOf: [...] }` - 任一满足

---

## 3. 工具注册与发现机制

### 3.1 核心工具

核心工具由 `src/agents/agent-tools.ts` 中的 `createOpenClawCodingTools()` 函数装配。这是一个 600+ 行的巨型函数，其内部顺序决定了工具的生命周期：

```typescript
// 1. 基础编码工具
createCodingTools(codingRoot)  // 产生 read/write/edit/bash/touch/create/etc.
  -> 沙箱模式下替换为 createSandboxedReadTool/createSandboxedWriteTool
  -> workspaceOnly 模式下包裹 wrapToolWorkspaceRootGuard

// 2. Shell 工具
createLazyExecTool()   // "exec" - 懒加载，只有调用时才 import bash-tools.js
createLazyProcessTool() // "process" - 后台进程管理
createApplyPatchTool()  // "apply_patch" - 差异化补丁

// 3. 频道工具
listChannelAgentTools() // 频道插件注册的工具（如登录、配置等）

// 4. OpenClaw 工具
createOpenClawTools()  // message/sessions_spawn/sessions_list/sessions_send/gateway etc.

// 5. 插件工具
resolveOpenClawPluginToolsForOptions() // 所有启用的插件工具

// 6. MCP 工具
// 通过 MCP 会话运行时动态注入

// 7. Tool Search 工具
createToolSearchTools() // 工具搜索/调用控制工具
```

### 3.2 插件工具注册

插件通过 `src/plugin-sdk/tool-plugin.ts` 定义：

```typescript
// 定义方式
defineToolPlugin({
  id: "my-plugin",
  name: "My Plugin",
  description: "...",
  tools: (tool) => [
    tool({
      name: "my_tool",
      description: "...",
      parameters: Type.Object({...}),
      execute: async (params, config, { api, signal }) => {
        // 工具执行逻辑
      },
    }),
  ],
});
```

插件工具注册有两种模式：

1. **标准 execute 模式**：直接提供 `execute` 函数，在插件注册时自动通过 `api.registerTool()` 注册。
2. **工厂模式（factory）**：提供 `factory` 函数，返回 `AnyAgentTool` 或数组，允许动态创建工具。

### 3.3 插件工具的运行时发现

在 `src/agents/openclaw-plugin-tools.ts` 中的 `resolveOpenClawPluginToolsForOptions()` 函数发现并实例化插件工具。传入 `pluginToolAllowlist` 和 `pluginToolDenylist` 进行过滤。

---

## 4. 工具执行流程

### 执行流程图

```
LLM 请求工具调用
        |
        v
splitToolExecuteArgs()
  - 处理新/旧两种参数顺序兼容
  - 提取 toolCallId, params, signal, onUpdate
        |
        v
=== 前置阶段 (如果未被 before-tool-call hook 包装) ===
prepareToolParamsBeforeHook()
  - 调用工具的 prepareBeforeToolCallParams
  - 例如：exec 工具需要准备命令执行环境
        |
        v
normalizeCodeModeExecBeforeHookParams()
  - 规范化 code mode 控制工具参数
        |
        v
runBeforeToolCallHook()  // <-- 核心钩子系统
  |
  +-- 循环检测 (detectToolCallLoop)
  |    检测重复调用模式
  |    如果 stuck + critical -> 返回 blocked veto
  |    如果 stuck + warning -> 发出警告
  |    记录每次调用到 session state
  |
  +-- 可信策略 (runTrustedToolPolicies)
  |    插件级 before_tool_call 策略
  |    可以 block 或 requireApproval
  |
  +-- 全局 hook (hookRunner.runBeforeToolCall)
  |    运行所有已注册的 before_tool_call hooks
  |    可以 block 或 requireApproval
  |
  +-- 审批流程 (requestPluginToolApproval)
       通过 gateway 发起审批请求
       等待用户决策 (allow/deny/timeout)
       支持 defer 模式 (延迟到后续处理)
        |
        v
reconcileCodeModeExecBeforeHookParams()
finalizeToolParamsBeforeExecute()
  - 调用工具的 finalizeBeforeToolCallParams
        |
        v
=== 执行阶段 ===
tool.execute(toolCallId, executeParams, signal, onUpdate)
        |
        v
=== 后置阶段 ===
normalizeToolExecutionResult()
  - 确保 result.content[] 格式正确
  - 处理非标准返回值 (兜底包装)
        |
        v
返回 AgentToolResult<unknown> 给 LLM runtime
```

### 错误处理

```
工具执行错误处理:
  if (signal?.aborted) -> 直接抛出原始错误 (中止优先)
  if (isBeforeToolCallBlockedError) -> 返回 blocked 结果
  else -> 
    1. 记录详细错误日志 (含 sanitized 参数)
    2. 返回 { status: "error", tool, error: message }
```

关键设计点：

- **参数 sanitization**：exec 命令参数会做摘要哈希后才记录日志，避免敏感信息泄露
- **exec env sanitization**：环境变量值在日志中统一替换为 `[omitted exec env value]`
- **两种 execute 签名兼容**：`(toolCallId, params, signal, onUpdate)` 和旧版 `(toolCallId, params, onUpdate, ctx, signal)`

---

## 5. 工具 Schema 生成与规范化

`src/agents/agent-tools-parameter-schema.ts` 是整个系统中最复杂的 Schema 处理模块。

### 处理流程

```
原始 JSON Schema
        |
        v
normalizeOpenApiSchemaKeywords()
  - 去除 OpenAPI 专有关键字 (nullable, discriminator, xml...)
  - 将 nullable: true 转换为 type: [type, "null"]
        |
        v
inlineLocalToolSchemaRefs()
  - 递归展开 $ref 引用
  - 解析 $defs/definitions 中的定义
  - 内联到 schema 本体
  - 处理循环引用 (用空对象打断)
        |
        v
normalizeArraySchemasMissingItems()
  - 为缺少 items 的 array 类型补上 items: {}
        |
        v
stripEmptyArrayItemsFromArraySchemas() (如果有配置)
  - 去除无意义的空 items schema
        |
        v
=== Provider 特殊处理 ===
Google Gemini -> cleanSchemaForGemini()  (去除不兼容关键字)
xAI -> stripUnsupportedSchemaKeywords()  (去除验证约束关键字)
OpenAI -> 需要顶层 type: "object" (触发 union flattening)
Anthropic -> 保持完整的 JSON Schema 2020-12
        |
        v
=== Union Flattening (当顶层有 anyOf/oneOf 时) ===
合并所有变体的 properties
合并 required 字段 (所有变体都需要的才升级为顶层)
保留 additionalProperties
        |
        v
最终 TSchema
```

### 缓存策略

使用 `WeakMap<object, Array<{ key: string; value: TSchema }>>` 对 Schema 进行缓存，每个 Schema 对象最多缓存 8 个变体（按 provider/model 组合）。使用 WeakMap 确保 Schema 对象不被缓存长期持有。

---

## 6. MCP 工具集成方式

MCP 集成是独立于主工具流水线的子系统，由 `${serverName}.${toolName}` 命名空间前缀避免冲突。

### 架构分层

```
agent-bundle-mcp-runtime.ts          -- 会话级 MCP 运行时管理器
agent-bundle-mcp-materialize.ts      -- MCP 工具物化
agent-bundle-mcp-tools.ts            -- 导出公共 API
agent-bundle-mcp-types.ts            -- 类型定义
agent-bundle-mcp-names.ts            -- 工具名规范化 (provider-safe)
```

### SessionMcpRuntime 生命周期

```
getOrCreateSessionMcpRuntime()
        |
        v
createSessionMcpRuntime()
  - 加载 MCP 配置 (mcpServers) 
  - 生成配置指纹 (configFingerprint)
  - 建立 BundleMcpSession (Client + Transport)
        |
        v
getCatalog()  -- 首次调用时连接所有 MCP 服务器
  - 遍历所有配置的 server
  - resolveMcpTransport() 解析传输类型 (stdio/SSE/streamable-http)
  - 创建 MCP SDK Client
  - connectWithTimeout() 连接 (带超时)
  - listAllToolsBestEffort() 获取工具列表
  - 应用 toolFilter (include/exclude 匹配)
  - 构建 McpToolCatalog
  - 添加到集合 Sessions 映射
        |
        v
materializeBundleMcpToolsForRun()
  - buildBundleMcpToolsFromCatalog()
    对每个 MCP 工具:
      - 生成 provider-safe 名称 (sanitizeServerName)
      - 规范化 inputSchema
      - 设置 PluginToolMeta (标记为 bundle-mcp)
      - 绑定 execute 回调 -> callTool()
    额外注入 utility 工具:
      - resources_list / resources_read (如果服务器支持)
      - prompts_list / prompts_get (如果服务器支持)
```

### MCP 特性亮点

1. **故障隔离** - 使用 `BUNDLE_MCP_FAILURE_THRESHOLD=3` + `BUNDLE_MCP_FAILURE_COOLDOWN_MS=60000` 的退避机制，单个 MCP 服务器故障不影响其他服务器。
2. **空闲回收** - 默认 10 分钟空闲 TTL，60 秒扫描间隔。
3. **配置变更检测** - 通过 configFingerprint（SHA1 哈希）检测 MCP 配置变化，自动重建运行时。
4. **Schema 验证** - 使用自定义 `createBundleMcpJsonSchemaValidator()`，支持 Draft 2020-12 Schema，并通过 TypeBox 编译进行高性能验证。
5. **工具列表变更通知** - 通过 MCP SDK 的 `listChanged` 回调机制，自动失效缓存。
6. **命名冲突解决** - 使用 `sanitizeServerName()` 和 `buildSafeToolName()` 确保工具名在 provider 环境中安全。

---

## 7. 工具审批与安全

### 多层级策略管道 (`src/agents/tool-policy-pipeline.ts`)

```
buildDefaultToolPolicyPipelineSteps() 返回的顺序:
  1. Profile 策略 (tools.profile)
  2. byProvider Profile 策略
  3. 全局策略 (tools.allow)
  4. 全局 byProvider 策略
  5. Agent 级策略
  6. Agent 级 byProvider 策略
  7. 群组策略 (channel groups)
  8. 发送者策略 (per-user)
  9. Sandbox 策略 (沙箱)
  10. Subagent 策略 (子代理)
  11. 继承策略 (父会话)
```

每个步骤都在执行 `filterToolsByPolicy()`，使用 `isToolAllowedByPolicyName()` 判断是否保留工具。

### 策略匹配规则 (`src/agents/tool-policy-match.ts`)

```
isToolAllowedByPolicyName(name, policy):
  1. 先检查 deny 列表 (glob 匹配)
  2. 如果匹配 deny -> 拒绝
  3. 如果 allow 列表为空 -> 允许所有
  4. 如果 allow 列表匹配 -> 允许
  5. 特殊规则: apply_patch 跟随 write 的 allow 状态
```

### 审批流程 (`src/agents/agent-tools.before-tool-call.ts`)

插件和策略可以通过 `requireApproval` 触发审批流程：

```
Plugin 请求审批:
  requireApproval: {
    pluginId, title, description, severity,
    allowedDecisions, timeoutMs, twoPhase
  }
        |
        v
requestPluginToolApproval():
  1. 通过 gateway 发送 plugin.approval.request
  2. 如果有即时决策 -> 直接处理
  3. 否则等待 plugin.approval.waitDecision
  4. AbortSignal 支持: 如果 run 被中止，提前退出等待
  5. 结果:
     - ALLOW_ONCE/ALLOW_ALWAYS -> 允许执行
     - DENY -> 返回 blocked
     - TIMEOUT -> 根据 timeoutBehavior (默认 deny)
```

### 审批模式

- `request` - 立刻发起审批请求
- `report` - 报告需要审批但不等待（返回 blocked）
- `defer` - 延迟到后续处理（用于两阶段审批）

### Subagent 安全限制

`src/agents/agent-tools.policy.ts`:

- **全局禁止**：gateway, agents_list, session_status, cron, sessions_send
- **Leaf 子代理额外禁止**：subagents, sessions_list, sessions_history, sessions_spawn

---

## 8. 错误处理与重试

### 工具执行错误处理

`toToolDefinitions()` 中的 execute 包装：

```
try {
  const rawResult = await tool.execute(...);
  const result = normalizeToolExecutionResult(rawResult);
  return result;
} catch (err) {
  // 1. AbortError -> 直接抛出（上层处理）
  if (signal?.aborted) throw err;
  
  // 2. BeforeToolCallBlockedError -> 返回 blocked result
  if (isBeforeToolCallBlockedError(err)) {
    return buildBlockedToolResult({ reason });
  }
  
  // 3. 普通错误 -> 记录日志 + 返回结构化 error result
  logError(`[tools] ${name} failed: ${message} raw_params=...`);
  return { content: [{ type: "text", text }], details: { status: "error" } };
}
```

### 日志安全

- exec 命令参数用 SHA256 前 16 字符替代
- exec 环境变量值统一替换为 `[omitted exec env value]`
- 超过 600 字符的参数会被截断

### 循环检测

`detectToolCallLoop()` 来自懒加载的 `agent-tools.before-tool-call.runtime.ts`：

- 记录每次工具调用的参数和结果
- 检测重复模式（相同参数/相同结果）
- 级别：warning -> 仅记录日志；critical -> 直接 block
- 警告频率：每 10 次触发一次（桶计数），避免日志泛滥

### MCP 错误处理

- 连接超时：`connectWithTimeout()` 使用独立定时器
- 工具调用失败：`runGuardedServerRequest()` 记录失败次数，达到阈值后暂停服务器 60 秒
- 释放超时：`disposeSession()` 使用 5 秒兜底超时

---

## 9. 关键设计决策

### 9.1 为什么使用单一巨大函数 `createOpenClawCodingTools()`？

**背景**：这个函数 600+ 行，不符合单一职责原则。

**理由**：工具装配是一个天然的线性 Pipeline，每一步的中间结果（政策对象、沙箱上下文、策略列表）需要在后续步骤中共享。拆成多个小函数会导致大量参数传递。在注释中已经标注了每个阶段的边界（`options?.recordToolPrepStage?.("stage-name")`），方便性能分析。

**代价**：
- 可测试性差（但已经通过 `testing` 导出关键子函数弥补）
- 理解负担重
- 代码变更冲突概率高

### 9.2 Before-Tool-Call Hook 包装的双重性

**设计**：工具既可以由 `wrapToolWithBeforeToolCallHook()` 包装，也可以在 `toToolDefinitions()` 中由 `runBeforeToolCallHook()` 直接调用。

**原因**：`toToolDefinitions()` 处理的是 **ClientToolDefinition**（来自 OpenAI Responses API 等外部宿主），这些工具没有标准的 `AnyAgentTool` 接口，必须在 adapter 级别注入 hook。

**隐患**：如果某个工具在两个地方都被 hook，会导致 hook 执行两次。代码通过 `isToolWrappedWithBeforeToolCallHook()` 的 `BEFORE_TOOL_CALL_WRAPPED` Symbol 标记来检测是否已经包装。

### 9.3 Schema 规范化 vs 原始 Schema 存储

**设计**：工具定义使用完整 JSON Schema，但在传递给 provider 时按需规范化。

**原因**：
- Anthropic 接受完整 JSON Schema 2020-12
- OpenAI 需要 `type: "object"` 顶层（触发 union flattening）
- Gemini 拒绝许多约束关键字（需要 `cleanSchemaForGemini()`）
- xAI 拒绝 minLength/maxLength 等验证关键字

**实现**：每个工具在 Pipeline 末尾统一调用 `normalizeToolParameters()`，缓存结果避免重复计算。

### 9.4 MCP 的 on-demand 初始化

**设计**：MCP 服务器连接延迟到首次需要时才执行，且有会话级别的运行时管理器。

**原因**：MCP 服务器可能启动慢（尤其是 stdio 进程），且很多 session 可能根本不需要 MCP 工具。

**坑点**：`getCatalog()` 是 async 的，这意味着 `createOpenClawCodingTools()` 在构造工具列表时不能同步包含 MCP 工具。MCP 工具需要通过另外的路径注入 `materializeBundleMcpToolsForRun()`。

### 9.5 ToolPolicy Pipeline 的复合过滤

**设计**：多个策略层叠过滤，而非优先匹配。

**规则**：工具必须通过 **所有** 策略的过滤才能保留。每个策略都可以 deny 工具，但只有 **至少一个** 策略的 allow 列表匹配时才保留（当 allow 不为空时）。

**实现**：在 `tool-policy-match.ts::isToolAllowedByPolicyName()` 中：
```
1. 如果任意 deny 匹配 -> 拒绝
2. 如果 allow 为空 -> 通过（允许所有）
3. 如果 allow 匹配 -> 通过
```

---

## 10. 生产环境评估

### 潜在问题

1. **`createOpenClawCodingTools()` 的热路径性能**：每次 AI 请求都会调用这个函数重建工具列表。600+ 行的同步执行 + Schema 规范化 + MCP getCatalog() 的异步等待可能成为瓶颈。

2. **MCP catalog 的竞态条件**：当并发请求触发 catalog 重建时，存在 `catalogInFlight` 去重机制，但如果 `catalogInvalidationGeneration` 在构建过程中被多次递增，可能导致缓存不一致。

3. **内存泄漏风险**：
   - `adjustedParamsByToolCallId` Map 有上限 1024，但旧条目使用 `Map.keys().next()` 清理（FIFO），可能导致热路径上的高频工具调 ID 被提前清理。
   - MCP Session 在 idle sweep 中通过 `lastUsedAt` 判断，但如果 `markUsed()` 没有被正确调用（比如工具调用失败路径），可能导致会话被过早回收。

4. **Schema 缓存的内存增长**：`WeakMap` 的每个条目最多 8 个变体，但如果工具数量大（MCP + 插件），累计的 Schema 缓存可能显著。

### 面试题

#### Q1: OpenClaw 的工具政策是如何实现多层级覆写的？如果有冲突（global allow=exec, agent deny=exec），最终是 allow 还是 deny？

**参考答案**：

工具体系采用的是 **"deny overrides allow" + "all policies must pass"** 的组合机制。在 `tool-policy-match.ts::isToolAllowedByPolicyName()` 中：

1. 首先检查 deny 列表，如果匹配则立即拒绝
2. 然后检查 allow 列表：如果 allow 为空（意味着允许所有），则通过；如果 allow 非空但不匹配，则拒绝

在多层 Pipeline 中（global -> agent -> group -> sender -> sandbox -> subagent），每一层都独立执行这个逻辑。工具必须通过 **所有层** 的过滤。

所以 global allow=exec 不会产生冲突：agent deny=exec 会让工具被拒绝，因为 agent 层的 deny 匹配了 exec。

但如果反过来，global deny=exec 而 agent allow=exec，结果仍然是 deny（deny 优先于 allow 的逻辑）。

**例外**：`alsoAllow` 机制可以覆盖 deny 列表，但它作用于 profile 层面，不是直接在 policy 层面。

#### Q2: 描述 OpenClaw 中 MCP 工具从配置到执行的完整生命周期。它如何处理 MCP 服务器崩溃？

**参考答案**：

1. **配置加载**：`loadEmbeddedAgentMcpConfig()` 从 OpenClaw 配置中读取 `mcpServers`，生成 SHA1 指纹。
2. **运行时创建**：`createSessionMcpRuntime()` 创建 `BundleMcpSession`（每个 server 一个 MCP SDK Client）。
3. **目录获取**：`getCatalog()` 延迟连接到所有服务器，调用 `client.listTools()`，应用 `toolFilter`。
4. **物化**：`materializeBundleMcpToolsForRun()` 执行 `buildBundleMcpToolsFromCatalog()`，将 MCP 工具转换为 `AnyAgentTool`，每个工具有 `pluginId: "bundle-mcp"` 的 PluginToolMeta 标记。
5. **执行**：LLM 调用 `serverName.toolName` 时，通过 `runtime.callTool()` 转发到对应 MCP 服务器的 `client.callTool()`。

**服务器崩溃处理**：

- 使用 `runGuardedServerRequest()` 包装所有 MCP 请求
- 失败时调用 `recordServerToolFailure()` 递增失败计数
- 连续失败 3 次后，服务器被暂停 60 秒（`BUNDLE_MCP_FAILURE_COOLDOWN_MS`）
- 暂停期间尝试调用会抛出明确错误："bundle-mcp server is paused after repeated tool failures"
- 成功一次后，`serverBackoff.delete()` 重置计数器
- 如果 `listChanged` 回调触发，`catalogInvalidationGeneration` 递增，下次 `getCatalog()` 重建目录

#### Q3: `normalizeToolParameterSchema()` 中的 union flattening 做了什么？为什么要这样做？

**参考答案**：

当工具的顶层 schema 包含 `anyOf` 或 `oneOf` 时，normalize 函数会执行 **union flattening**：

1. 提取所有变体的 `properties` 字典
2. 合并相同名称的属性（通过 `mergePropertySchemas()` 合并 enum 值）
3. 计算跨变体都需要的 `required` 字段
4. 构建一个 `{ type: "object", properties: mergedProperties, required: mergedRequired }` 的扁平 schema

**原因**：

- **OpenAI** 的函数调用 API 要求顶层必须是 `type: "object"`，有 `anyOf` 的 schema 会被直接拒绝
- **Gemini** 不支持顶层 `type` 与 `anyOf` 共存
- **Anthropic** 能处理 `anyOf`，但扁平化后的 schema 对 LLM 更友好

**代价**：合并后的 schema 丢失了变体间的互斥语义。例如 `{ anyOf: [{ x: string }, { y: number }] }` 会变成 `{ x?: string, y?: number }`，失去了"只能选其一"的约束。

#### Q4: `toToolDefinitions()` 如何处理两种不同的 tool.execute 参数顺序？

**参考答案**：

代码通过 `splitToolExecuteArgs()` 进行兼容性适配。其核心逻辑：

1. 定义两种签名：`(toolCallId, params, signal, onUpdate)`（新版）和 `(toolCallId, params, onUpdate, ctx, signal)`（旧版）
2. 通过检查第 3 个参数是否为 function（`typeof args[2] === "function"`）来判断调用方式
3. 检查第 5 个参数是否为 AbortSignal

这是因为系统中的不同 Runtime（如 Anthropic 的 OAuth 传输 vs 本地 code mode）使用不同的执行接口定义。这个兼容层确保所有 Runtime 可以共享同一套工具定义。

#### Q5: 如果一个工具执行时抛出异常，OpenClaw 怎么保证 LLM 不会因此挂起？异常中的敏感信息如何保护？

**参考答案**：

**防挂起机制**：

`toToolDefinitions()` 的 execute 函数用 try-catch 包裹。任何异常都不会传播到 LLM runtime：

1. 如果是 `AbortError`，说明是 run 被中止，直接抛出（由上层处理）
2. 如果是 `BeforeToolCallBlockedError`，返回 `buildBlockedToolResult()`，包含友好的原因描述
3. 其他所有异常都会被 catch，记录日志后返回 `{ status: "error", error: message }` 结构

**敏感信息保护**：

- 对于 `exec` 工具：参数中 `command` 字段用 SHA256 哈希（前 16 字符）替代原始命令
- `env` 字段的所有值替换为 `[omitted exec env value]`
- 所有参数在日志中截断到 600 字符
- 使用 `redactToolDetail()` 进一步脱敏

这意味着即使 exec 执行了包含密码的命令，日志中也看不到明文。

#### Q6: 当一个插件工具需要用户审批时，OpenClaw 如何将这个审批请求同步回用户？如果用户一直没有响应会发生什么？

**参考答案**：

审批流程通过 gateway 的 `plugin.approval.request` 和 `plugin.approval.waitDecision` 两个 gateway tool 实现：

1. **发起审批**：`requestPluginToolApproval()` 调用 `callGatewayTool("plugin.approval.request", ...)`
   - 包含 `title`, `description`, `severity`, `allowedDecisions` 等信息
   - `twoPhase: true` 表示需要两阶段（先请求，再等待决策）
   - 设置 `timeoutMs`（从插件配置或默认值）

2. **等待决策**：
   - 如果请求返回了即时决策（`hasImmediateDecision`），直接处理
   - 否则，调用 `callGatewayTool("plugin.approval.waitDecision", { id })` 等待用户决策
   - 支持 `AbortSignal` 中断等待：如果 agent run 被取消，审批等待也会提前退出

3. **超时处理**：
   - 超时行为由 `approval.timeoutBehavior` 配置，默认为 `"deny"`
   - 可选 `"allow"` 超时自动放行
   - Gateway timeout 比审批 timeout 多 10 秒缓冲，确保 gateway 能先清理再超时

4. **决策结果**：
   - `ALLOW_ONCE` / `ALLOW_ALWAYS` -> 允许执行，`ALLOW_ALWAYS` 会记录到 approvel resolution
   - `DENY` -> 返回 "Denied by user"
   - 超时/取消 -> 根据配置 deny 或 allow
