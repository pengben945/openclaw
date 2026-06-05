# OpenClaw MCP 集成深度分析

> 分析日期: 2026-06-05
> 目标版本: openclaw `main` (8f6f2617ec)

---

## 1. MCP 整体架构

OpenClaw 中的 MCP 集成存在 **两个正交的角色**，不可混淆：

| 角色 | 目录 | 方向 | 说明 |
|------|------|------|------|
| **MCP 服务端** (Server) | `src/mcp/` | OpenClaw 作为 MCP 服务端 | 对外暴露 OpenClaw 频道能力为 MCP 工具 |
| **MCP 客户端** (Client) | `src/agents/` | OpenClaw 作为 MCP 客户端 | 连接外部 MCP 服务器，将其工具桥接到 Agent 系统 |

这两个角色共享的公共资产仅仅是 `@modelcontextprotocol/sdk` 这个 npm 依赖；两者的代码路径、生命周期、配置来源完全独立。

### 1.1 MCP 服务端 (Server Mode)

```
外界 MCP 客户端 (Claude Desktop / Claude Code)
        |
        | MCP stdio
        v
  +------------------+
  | McpServer        |  <-- channel-server.ts
  +------------------+
        |
  +------------------+
  | OpenClawChannelBridge  |  <-- channel-bridge.ts
  +------------------+
        |
  +------------------+
  | GatewayClient    |  <--> OpenClaw Gateway
  +------------------+
```

- 通过 stdio transport 对外暴露 MCP Server
- 场景：Claude Desktop 将 OpenClaw 作为 MCP 服务器添加，从而在 Claude 对话中直接操作 OpenClaw 会话
- 暴露的工具：`conversations_list`, `conversation_get`, `messages_read`, `messages_send`, `attachments_fetch`, `events_poll`, `events_wait`, `permissions_list_open`, `permissions_respond`

### 1.2 MCP 客户端 (Client Mode)

```
Agent Runtime
        |
  +------------------+
  | SessionMcpRuntimeManager  |  <-- 全局单例，管理所有会话的 MCP 运行时
  +------------------+
        |
  +------------------+
  | SessionMcpRuntime |  <-- 每个 agent session 一个
  +------------------+
        |
  +--------+---------+
        |
  +------+------+   +------+------+
  | MCP Client  |   | MCP Client  |
  | (MCP SDK)   |   | (MCP SDK)   |
  +------+------+   +------+------+
        |                   |
  +----+----+         +----+----+
  | Transport|         | Transport|
  | (stdio)  |         | (HTTP)   |
  +---------+          +---------+
        |                   |
  External MCP         External MCP
  Server (stdio)       Server (HTTP)
```

- 场景：用户配置了外部 MCP 服务器（如 `mcpServers` 中的 Filesystem MCP Server），OpenClaw Agent 需要调用这些服务器上的工具
- 配置来源：插件 manifest、用户 `openclaw.json` 中的 `mcp.servers`、`.mcp.json` 文件

---

## 2. MCP 服务器注册与发现

### 2.1 配置层次

MCP 服务器配置来自三个层次，由 `loadMergedBundleMcpConfig` 合并：

```typescript
// src/agents/bundle-mcp-config.ts:47
export function loadMergedBundleMcpConfig(params) {
  // 1. 插件注册的 MCP 服务器
  const bundleMcp = loadEnabledBundleMcpConfig({ ... });

  // 2. 用户 openclaw.json 配置
  const configuredMcp = normalizeConfiguredMcpServers(cfg?.mcp?.servers);

  // 3. 合并策略：用户配置覆盖插件配置
  //    - enabled: false 的服务器会被排除
  return { mcpServers: { ...enabledBundleMcp, ...enabledConfiguredMcp } };
}
```

### 2.2 插件 MCP 注册流程

```
插件安装目录
  |
  +-- manifest.json (声明 mcpServers 路径)
  |     |-- mcpServers: [".mcp.json"]
  |     |-- mcpServers: { inline config }
  |
  +-- .mcp.json (MCP 服务器定义文件)
        |
        +-- mcpServers: {
        |     "my-server": {
        |       command: "node",
        |       args: ["server.js"]
        |     }
        |   }
```

关键路径：

1. `loadEnabledBundleMcpConfig` (`src/plugins/bundle-mcp.ts:291`) -- 遍历所有启用的插件
2. 每个插件读取其 `manifest.json` 中的 `mcpServers` 字段
3. 支持三种配置方式：文件引用 (`.mcp.json`)、内联配置、CLI type 别名
4. 路径被展开到绝对路径，`${CLAUDE_PLUGIN_ROOT}` 占位符被替换
5. 使用 `applyMergePatch` 支持深度合并（JSON Merge Patch 语义）

### 2.3 用户配置格式

通过 `openclaw.json` 中的 `mcp.servers` 配置：

```json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"],
        "enabled": true
      },
      "remote-server": {
        "url": "https://mcp.example.com/sse",
        "transport": "sse",
        "headers": { "Authorization": "Bearer xxx" }
      }
    },
    "sessionIdleTtlMs": 600000
  }
}
```

完整配置类型见 `src/config/types.mcp.ts:23` 的 `McpServerConfig`。

### 2.4 CLI 类型别名兼容

为了兼容 Claude Code / Gemini CLI 的 `type` 字段，OpenClaw 实现了别名转换：

```
CLI type       -> OpenClaw transport
"http"         -> "streamable-http"
"streamable-http" -> "streamable-http"
"sse"          -> "sse"
"stdio"        -> "stdio"
```

见 `src/config/mcp-config-normalize.ts:6` 的 `CLI_MCP_TYPE_TO_OPENCLAW_TRANSPORT` 映射。

---

## 3. MCP 传输层实现

### 3.1 传输类型

三种传输方式，在 `mcp-transport.ts` 的 `resolveMcpTransport` 中解析：

| 传输类型 | 类 | 使用场景 |
|---------|-----|---------|
| stdio | `OpenClawStdioClientTransport` | 本地进程，子进程 spawn |
| SSE | `SSEClientTransport` (MCP SDK) | 传统 HTTP 长连接 |
| Streamable HTTP | `StreamableHTTPClientTransport` (MCP SDK) | 新式 HTTP，支持会话管理 |

### 3.2 Stdio 传输实现

`OpenClawStdioClientTransport` (`src/agents/mcp-stdio-transport.ts:28`) 是自定义实现，关键设计：

```typescript
// 进程管理
const child = spawn(command, args, {
  detached: process.platform !== "win32",  // 允许进程树管理
  shell: false,                            // 禁止 shell 注入
});
```

- **进程树终结**：`close()` 方法使用分级终结策略 (line 113-136)
  1. `stdin.end()` 优雅关闭
  2. 等待 2s 后 `killProcessTree` (SIGTERM)
  3. 再等 2s 后 `signalProcessTree` SIGKILL
- **stderr 日志**：通过 `attachStderrLogging` 将子进程 stderr 重定向到 OpenClaw 日志系统
- **背压处理**：`send()` 方法在 `write` 回调中解决 promise，避免异步 EPIPE 逃逸为 uncaughtException

### 3.3 HTTP 传输实现

HTTP 传输 (SSE / Streamable HTTP) 的封装在 `mcp-transport.ts`，关键增强：

- **重定向追踪**：`buildStreamableHttpFetch` 实现手动重定向处理，支持最多 20 跳
- **跨域安全**：跨域重定向时 `retainSafeHeadersForCrossOriginRedirect` 清理敏感 header
- **TLS 自签名证书**：通过 `buildMcpHttpFetch` 支持 `sslVerify: false`、mTLS 客户端证书
- **OAuth 支持**：通过 `createMcpOAuthClientProvider` 集成 MCP 标准 OAuth 流程

### 3.4 OAuth 认证

`src/agents/mcp-oauth.ts` 实现了完整的 MCP OAuth 客户端提供者：

```typescript
export function createMcpOAuthClientProvider(params) {
  return {
    // 标准的 OAuthClientProvider 接口
    redirectUrl,
    clientMetadata,
    state,
    clientInformation,
    tokens,
    redirectToAuthorization,
    // ...
  };
}
```

- 令牌持久化：存储到 `state/mcp-oauth/<server>-<sha256>.json`
- 文件权限：`0o600`（仅 owner 可读写）
- 支持 `openclaw mcp login` 命令行认证
- 回调地址默认 `http://127.0.0.1:8989/oauth/callback`

---

## 4. MCP --> Agent 工具桥接流程

这是整个客户端模式的核心价值——将外部 MCP 服务器的工具转化为 Agent 可调用的 `AnyAgentTool`。

### 4.1 整体流程

```
配置加载
  |
  v
loadMergedBundleMcpConfig()
  |
  v
createSessionMcpRuntime()
  |-- 为每个会话创建一个 SessionMcpRuntime
  |-- 在 getCatalog() 中懒连接 MCP 服务器
  |
  v
getCatalog()
  |-- 遍历所有配置的 MCP 服务器
  |-- 对每台服务器：
  |     1. resolveMcpTransport() - 解析传输配置
  |     2. Client.connect() - 建立 MCP 连接
  |     3. client.listTools() - 列举工具
  |     4. 应用 toolFilter include/exclude 过滤
  |-- 构建 McpToolCatalog（服务器元数据 + 工具列表）
  |
  v
materializeBundleMcpToolsForRun()
  |-- 调用 buildBundleMcpToolsFromCatalog()
  |-- 为每个 MCP 工具创建 AnyAgentTool
  |-- 工具执行时委托回 SessionMcpRuntime.callTool()
  |
  v
Agent 工具系统
  |-- 工具被注入到 agent 的 tool_use block 中
  |-- agent 选择工具 -> 执行 -> 结果返回
```

### 4.2 工具命名和安全

`buildSafeToolName` (`src/agents/agent-bundle-mcp-names.ts:46`) 确保工具名在 provider 之间安全和唯一：

```
服务器名 "my-file-server" + 工具名 "read_file"
  -> "my-file-server__read_file" (使用 __ 分隔)

服务器名 "my-file-server" + 工具名 "read_file" (已存在)
  -> "my-file-server__read_file-2" (冲突自动编号)
```

关键约束：
- 总长度不超过 64 字符
- 服务器名前缀不超过 30 字符
- 只允许 `[A-Za-z0-9_-]` 字符
- 必须以字母开头

### 4.3 工具过滤

每台服务器可通过 `toolFilter` 配置 include/exclude 模式：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node server.js",
      "toolFilter": {
        "include": ["read_*"],
        "exclude": ["write_*", "delete_*"]
      }
    }
  }
}
```

使用 glob 匹配（只支持 `*` 通配符），见 `globMatches()` 函数。

### 4.4 资源/提示的桥接

MCP Resources 和 Prompts 不是直接暴露给 Agent 的，而是通过**虚拟工具**桥接：

| MCP 能力 | 虚拟工具名 | 说明 |
|---------|-----------|------|
| resources.list | `<server>__resources_list` | 列出资源 |
| resources.read | `<server>__resources_read` | 读取资源（需要 uri 参数） |
| prompts.list | `<server>__prompts_list` | 列出提示 |
| prompts.get | `<server>__prompts_get` | 获取提示（需要 name 参数） |

这些虚拟工具在 `buildBundleMcpToolsFromCatalog` 中自动生成，且同样受 `toolFilter` 约束。

### 4.5 结果处理

`toAgentToolResult` (`src/agents/agent-bundle-mcp-materialize.ts:22`) 处理 MCP 调用结果：

```typescript
// 结构化内容优先
if (result.structuredContent 存在) {
  // 只返回 structuredContent 块，避免提示膨胀
}
// 否则返回原始 content
// 错误时自动添加 mcpServer/mcpTool 元数据
```

### 4.6 错误重试与熔断

`runGuardedServerRequest` (`src/agents/agent-bundle-mcp-runtime.ts:521`) 实现服务器级熔断：

- 阈值：连续 3 次失败 (`BUNDLE_MCP_FAILURE_THRESHOLD = 3`)
- 冷却：60s (`BUNDLE_MCP_FAILURE_COOLDOWN_MS`)
- 熔断期间对调用方抛出明确错误信息
- 成功调用自动重置熔断状态

---

## 5. 生命周期管理

### 5.1 会话 MCP 运行时

`SessionMcpRuntime` 是每个 agent session 的 MCP 运行时单元：

```
创建:
  createSessionMcpRuntime()
    |-- 加载 MCP 配置
    |-- 首次 getCatalog() 时连接所有服务器
    |-- 生成 configFingerprint 用于变更检测

使用:
  getCatalog() - 列举工具（懒连接）
  callTool()   - 调用 MCP 工具
  listResources() / listPrompts() / readResource() / getPrompt()

回收:
  dispose() - 关闭所有连接、清理进程

空闲回收:
  sweepIdleRuntimes() - 超过 idleTtl (默认 10min) 自动回收
```

### 5.2 全局管理器

`SessionMcpRuntimeManager` 是一个全局单例（通过 `Symbol.for("openclaw.sessionMcpRuntimeManager")` 注册）：

```typescript
export function getSessionMcpRuntimeManager(): SessionMcpRuntimeManager {
  return resolveGlobalSingleton(SESSION_MCP_RUNTIME_MANAGER_KEY, createSessionMcpRuntimeManager);
}
```

核心功能：
- **去重创建**：`getOrCreate()` 确保同一 sessionId 不会创建多个运行时
- **配置变更检测**：`configFingerprint` 缓存失效机制
- **创建中消歧**：`createInFlight` map 防止同 sessionId 并发创建
- **空闲回收定时器**：每 60s 扫描一次
- **泄漏租约**：`acquireLease()` / release callback 机制防止在用时被回收

### 5.3 连接生命周期

```
start()
  |-- 首次连接：connectWithTimeout
  |     |-- 默认 30s 超时
  |-- 失败回退：根据错误类型
  |     |-- GatewayClientRequestError.retryable -> 重试
  |     |-- 其他 -> 拒绝 readyPromise
  |
  已连接
  |-- 接收事件：onEvent
  |     |-- session.message -> 通知 MCP 客户端
  |     |-- exec.approval.* -> 跟踪审批状态
  |     |-- plugin.approval.* -> 跟踪审批状态
  |
  disconnect
  |-- 自动重连（如果 retryable）
  |-- 非 retryable -> 网关断开导致 readyPromise 拒绝
```

### 5.4 关闭顺序

```
close()
  |-- 标记 closed
  |-- 清理 pending 定时器
  |-- 解析所有 pending waiter（返回 null）
  |-- 清理 pending approvals
  |-- 停止 GatewayClient
  |-- 清理认证缓存（OAuth 令牌持久化是分开的）
```

---

## 6. MCP 服务端 (Server 模式) 实现细节

### 6.1 服务器创建

`createOpenClawChannelMcpServer` (`src/mcp/channel-server.ts:28`)：

```typescript
const server = new McpServer(
  { name: "openclaw", version: VERSION },
  capabilities ? { capabilities } : undefined,
);
const bridge = new OpenClawChannelBridge(cfg, { gatewayUrl, /* ... */ });
bridge.setServer(server);
// 注册 Claude Permission 通知处理器
server.server.setNotificationHandler(ClaudePermissionRequestSchema, handler);
// 注册所有频道工具
registerChannelMcpTools(server, bridge);
```

### 6.2 工具清单

`registerChannelMcpTools` (`src/mcp/channel-tools.ts`) 注册以下工具：

| 工具名 | 功能 | 后台方法 |
|-------|------|---------|
| `conversations_list` | 列出会话 | `bridge.listConversations()` |
| `conversation_get` | 获取单会话 | `bridge.getConversation()` |
| `messages_read` | 读取消息历史 | `bridge.readMessages()` |
| `attachments_fetch` | 获取附件 | `bridge.readMessages()` + 本地过滤 |
| `events_poll` | 轮询事件队列 | `bridge.pollEvents()` |
| `events_wait` | 等待事件 | `bridge.waitForEvent()` |
| `messages_send` | 发送消息 | `bridge.sendMessage()` |
| `permissions_list_open` | 列出待审批 | `bridge.listPendingApprovals()` |
| `permissions_respond` | 响应审批 | `bridge.respondToApproval()` |

所有工具都通过 Gateway 协议调用，操作的是 OpenClaw 网关。

### 6.3 Claude Channel 模式

`ClaudeChannelMode` 控制 MCP 服务器是否向 Claude 发送频道通知：

| 模式 | `getChannelMcpCapabilities` | `shouldEmitClaudeChannel` |
|------|---------------------------|--------------------------|
| `off` | 返回 undefined | 始终 false |
| `on` | 声明 `claude/channel` 能力 | role==="user" 且 conversation 存在 |
| `auto` | 同 `on` | 同 `on` |

### 6.4 权限处理

MCP 服务器处理 Claude 权限请求（当 Claude Desktop 使用工具需要用户批准时）：

```typescript
// 接收 Claude 的权限请求通知
server.server.setNotificationHandler(ClaudePermissionRequestSchema, handler);

// 通过网关消息监听用户回复
// 匹配 "yes <requestId>" 或 "no <requestId>" 模式
const permissionMatch = text ? CLAUDE_PERMISSION_REPLY_RE.exec(text) : null;
```

### 6.5 插件工具 MCP 服务器

`plugin-tools-serve.ts` 提供一个独立的 MCP 服务器，暴露所有已注册的插件工具（如 memory 工具），供 ACP 会话（Claude Code）使用。

这个服务器使用的是 `tools-stdio-server.ts` 中的通用 `createToolsMcpServer` 模式：

```typescript
const server = new Server(
  { name: "openclaw-plugin-tools", version: VERSION },
  { capabilities: { tools: {} } },
);
server.setRequestHandler(ListToolsRequestSchema, handlers.listTools);
server.setRequestHandler(CallToolRequestSchema, handlers.callTool);
```

---

## 7. 关键设计决策

### 7.1 为什么使用自定义 Stdio 传输而不是 MCP SDK 默认实现？

OpenClaw 实现了自己的 `OpenClawStdioClientTransport`（而不是使用 MCP SDK 的 `StdioClientTransport`），原因：

1. **进程树终结**：MCP SDK 默认只 kill 子进程本身，OpenClaw 需要 `killProcessTree` 确保子进程创建的子进程也被清理
2. **分级关闭策略**：SIGTERM -> 2s -> SIGKILL 的多级关闭
3. **stderr 日志集成**：将子进程 stderr 接入 OpenClaw 的 `logDebug` 系统
4. **背压处理**：stdin write 的 promise 化处理

### 7.2 为什么 MCP 客户端是 session 级别的，而不是全局的？

`SessionMcpRuntime` 是按 agent session 创建的，每个 `getOrCreate` 调用都绑定到一个 `sessionId`。这是为了：

1. **隔离性**：不同 agent session 可能有不同的 MCP 配置（不同的工作空间目录）
2. **生命周期对齐**：MCP 服务器生命周期与 agent session 对齐
3. **配置变更**：不同 session 可能使用不同的 `workspaceDir`，从而加载不同的 `.mcp.json`
4. **工具过滤**：不同 session 可能有不同的 `toolFilter` 规则

### 7.3 为什么资源/提示要用虚拟工具而不是直接暴露？

MCP 的 Resources 和 Prompts 是模型上下文协议的原生概念，但 OpenClaw 的 Agent 工具系统是基于 `tool_use` 的。为了统一接口：

1. Resources/Prompts 在 Agent 侧表现为普通工具（使用 `__` 命名空间前缀）
2. Agent 可以像调用工具一样列出资源、读取资源
3. 虚拟工具同样受 `toolFilter` 约束
4. 返回结果标记为 `untrustedMcpOutput: true`，提醒消费方输出来自外部服务器

### 7.4 为什么 MCP 服务端和客户端是完全独立的代码路径？

虽然两者都使用 `@modelcontextprotocol/sdk`，但 OpenClaw 的 MCP 服务端和客户端在架构上没有共享代码：

- **MCP 服务端** (Server 模式) 是网关协议的 MCP 包装器，让外部 MCP 客户端能控制 OpenClaw
- **MCP 客户端** (Client 模式) 是 MCP 工具的消费方，让 OpenClaw Agent 能使用外部 MCP 服务器
- 两者面向不同的用户场景和配置来源

### 7.5 JSON Schema 兼容性策略

MCP 使用 draft-2020-12 JSON Schema，而 OpenClaw 的工具系统使用 TypeBox。`createBundleMcpJsonSchemaValidator` (`src/agents/agent-bundle-mcp-runtime.ts:155`) 实现了桥接：

1. 识别 draft-2020-12 schema（检查 `$schema` 字段）
2. 移除 `format` 关键字（TypeBox 不支持）
3. 将 `type: ["string", "number"]` 数组形式展开为 `anyOf`
4. 使用 TypeBox 的 `Compile` 进行验证
5. 非 draft-2020-12 schema 回退到 MCP SDK 的 `AjvJsonSchemaValidator`

### 7.6 速率限制与服务器熔断

`runGuardedServerRequest` 实现了针对不稳定 MCP 服务器的保护：

- 连续 3 次失败后进入 60s 冷却
- 冷却期内所有调用立即抛出，不实际连接服务器
- 成功调用重置计数器
- 这是一个**客户端侧**的熔断，对应用户配置的外部服务器

---

## 8. 生产环境与面试视角

### 8.1 潜在的生产环境风险

| 问题 | 场景 | 风险等级 |
|------|------|---------|
| 子进程泄漏 | MCP stdio 服务器 `close()` 分级终结仍有窗口期 | 中 |
| OAuth 令牌文件权限 | `mcp-oauth/*.json` 文件权限 `0o600`，但多用户系统下需额外注意 | 低 |
| 内存泄漏 | `pendingWaiters` 如果 resolve 从未被调用，waiter 对象一直被 Set 持有 | 中 |
| configFingerprint 碰撞 | SHA1 hash 用于检测配置变更，碰撞概率低但理论上可能 | 极低 |
| 大面积熔断风暴 | 多个 session 使用同一不稳定 MCP 服务器，各自独立熔断 | 低 |
| stderr 缓存堆积 | `OpenClawStdioClientTransport` 的 stderr stream 如果不消费可能背压 | 低 |

### 8.2 面试模拟题

**Q1: OpenClaw 同时实现了 MCP 服务端和客户端，两者在代码层面有什么根本区别？**

参考答案：两者的职责完全相反。MCP 服务端 (`src/mcp/`) 将 OpenClaw Gateway 能力封装为 MCP Server，供外部调用；MCP 客户端 (`src/agents/agent-bundle-mcp-runtime.ts`) 消费外部 MCP Server 的工具，注入到 Agent 工具系统中。服务端使用 `McpServer` + `StdioServerTransport`，客户端使用 `Client` + 多种 Transport。两者不共享业务代码。

**Q2: 描述 MCP 工具名 `my-server__read_file-2` 是如何生成的？**

参考答案：使用 `buildSafeToolName` 函数 (`src/agents/agent-bundle-mcp-names.ts:46`)。服务器名 "my-server" 被截断到 30 字符，工具名 "read_file" 被安全化（只含 `[A-Za-z0-9_-]`），合并为 "my-server__read_file"。如果冲突，递增编号 "my-server__read_file-2"。总长度不超过 64 字符。

**Q3: 如果配置的 MCP 服务器连续崩溃，OpenClaw 会怎么处理？熔断机制如何实现？**

参考答案：`runGuardedServerRequest` 函数 (`src/agents/agent-bundle-mcp-runtime.ts:521`) 记录连续失败次数。达到 3 次后进入 60s 冷却期，期间所有请求立即抛出错误，不实际连接服务器。成功调用重置计数器。这是客户端侧的保护机制。

**Q4: MCP Resources 和 Prompts 在 OpenClaw Agent 中是如何暴露的？**

参考答案：它们不作为 MCP 原生概念暴露，而是通过虚拟工具桥接。当服务器在其 capabilities 中声明支持 resources 或 prompts 时，`buildBundleMcpToolsFromCatalog` 自动生成 `<server>__resources_list`、`<server>__resources_read`、`<server>__prompts_list`、`<server>__prompts_get` 四个虚拟工具，同样受 `toolFilter` 约束。

**Q5: 简述 MCP stdio 子进程的关闭策略，为什么如此设计？**

参考答案：`OpenClawStdioClientTransport.close()` 使用三级关闭策略：
1. `stdin.end()` 优雅通知子进程
2. 2s 后 `killProcessTree(pid)` 发送 SIGTERM 给整个进程树
3. 再 2s 后 `signalProcessTree(pid, "SIGKILL")` 强制终结
这种设计是因为 MCP 服务器可能是脚本或进程树（如 `npx -y package` 可能启动多个进程），需要确保完整清理。三级策略平衡了优雅关闭和防止僵尸进程的需求。

---

## 9. 建议添加注释的位置列表

以下位置代码逻辑复杂或涉及重要的架构决策，建议添加中文注释：

| 文件 | 行号/区域 | 原因 |
|------|----------|------|
| `src/mcp/channel-bridge.ts:49` | `OpenClawChannelBridge` class | 核心桥接类，职责横跨 Gateway 和 MCP，领域边界不明显 |
| `src/mcp/channel-bridge.ts:526-588` | `handleSessionMessageEvent` | 权限匹配与事件路由的混合逻辑 |
| `src/mcp/channel-bridge.ts:604-617` | `shouldRetryInitialMcpGatewayConnect` | 重试策略的边界条件 |
| `src/mcp/tools-stdio-server.ts:24-48` | `connectToolsMcpServerToStdio` | stdio shutdown 的竞态处理 |
| `src/agents/agent-bundle-mcp-runtime.ts:381-414` | `disposeSession` | 多级超时关闭策略 |
| `src/agents/agent-bundle-mcp-runtime.ts:509-537` | `runGuardedServerRequest` | 服务器熔断逻辑 |
| `src/agents/agent-bundle-mcp-runtime.ts:544-744` | `getCatalog` | 核心的 catalog 构建与缓存失效 |
| `src/agents/agent-bundle-mcp-runtime.ts:863-1071` | `createSessionMcpRuntimeManager` | 全局管理器的 `createInFlight` 和 `sweepIdleRuntimes` 并发设计 |
| `src/agents/agent-bundle-mcp-materialize.ts:22-71` | `toAgentToolResult` | 结构化内容 vs 原始内容的取舍逻辑 |
| `src/agents/agent-bundle-mcp-materialize.ts:194-347` | `buildBundleMcpToolsFromCatalog` | 工具/资源/提示的三合一投影 |
| `src/agents/mcp-stdio-transport.ts:113-136` | `close()` | 三级进程终结策略，与 MCP SDK 默认行为的差异 |
| `src/agents/mcp-transport.ts:119-174` | `buildStreamableHttpFetch` | 自定义重定向追踪，弥补 MCP SDK 的不足 |
| `src/agents/mcp-oauth.ts:78-169` | `createMcpOAuthClientProvider` | OAuth 状态持久化设计 |
| `src/agents/bundle-mcp-config.ts:47-89` | `loadMergedBundleMcpConfig` | 三层配置合并（插件默认 + 用户配置 + 禁用覆盖） |
| `src/mcp/channel-tools.ts:24-189` | `registerChannelMcpTools` | MCP 工具定义与 OpenClaw 频道的映射 |
| `src/agents/agent-bundle-mcp-runtime.ts:155-187` | `createBundleMcpJsonSchemaValidator` | TypeBox 与 MCP draft-2020-12 schema 的互操作桥接 |

---

## 10. 文件引用索引

| 路径 | 角色 |
|------|------|
| `src/mcp/channel-server.ts` | MCP 服务端入口，创建并启动 MCP Server |
| `src/mcp/channel-bridge.ts` | 服务端桥接层，连接 MCP <-> Gateway |
| `src/mcp/channel-shared.ts` | 服务端共享类型和工具函数 |
| `src/mcp/channel-tools.ts` | 服务端 MCP 工具注册 |
| `src/mcp/tools-stdio-server.ts` | 通用工具型 MCP 服务器的 stdio 启动 |
| `src/mcp/plugin-tools-serve.ts` | 暴露插件工具的独立 MCP 服务器 |
| `src/mcp/plugin-tools-handlers.ts` | 插件工具的执行处理器 |
| `src/mcp/openclaw-tools-serve.ts` | 暴露内置 OpenClaw 工具的 MCP 服务器 |
| `src/agents/agent-bundle-mcp-runtime.ts` | **核心文件**：MCP 客户端运行时 |
| `src/agents/agent-bundle-mcp-types.ts` | MCP 客户端类型定义 |
| `src/agents/agent-bundle-mcp-materialize.ts` | MCP catalog -> Agent 工具投影 |
| `src/agents/agent-bundle-mcp-names.ts` | 工具名安全化与去重 |
| `src/agents/agent-bundle-mcp-tools.ts` | MCP 工具模块的重新导出 |
| `src/agents/mcp-transport.ts` | MCP 传输层解析和创建 |
| `src/agents/mcp-transport-config.ts` | 传输配置参数提取 |
| `src/agents/mcp-stdio.ts` | Stdio 传输配置 |
| `src/agents/mcp-http.ts` | HTTP 传输配置 |
| `src/agents/mcp-http-fetch.ts` | HTTP fetch 封装（SSL、mTLS、重定向） |
| `src/agents/mcp-oauth.ts` | MCP OAuth 认证 |
| `src/agents/mcp-stdio-transport.ts` | 自定义 Stdio 传输实现 |
| `src/agents/mcp-config-shared.ts` | 配置共享工具（env 安全、类型守卫） |
| `src/agents/bundle-mcp-config.ts` | 插件+用户配置合并 |
| `src/agents/embedded-agent-mcp.ts` | 嵌入 agent 的 MCP 配置加载 |
| `src/agents/codex-mcp-config.ts` | Codex MCP 配置投影 |
| `src/agents/cli-runner/bundle-mcp-codex.ts` | Codex CLI MCP 用户配置注入 |
| `src/plugins/bundle-mcp.ts` | 插件 MCP 配置加载 |
| `src/config/mcp-config-normalize.ts` | 配置别名规范化 |
| `src/config/types.mcp.ts` | MCP 配置类型定义 |
| `src/plugin-sdk/codex-mcp-projection.ts` | MCP 投影的 SDK 边界导出 |
