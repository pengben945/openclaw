# OpenClaw Gateway 深度分析

分析日期: 2026-06-05
分析者: Claude Opus 4.8 (1M context)

---

## 1. Gateway 架构总览

OpenClaw Gateway 是整个系统的**控制面核心**，它是一个同时承载 HTTP 和 WebSocket 的服务器进程，负责：

- **管理面**: 配置读取、认证授权、插件生命周期、渠道管理、健康监测
- **通信面**: WebSocket 实时双向通信、HTTP REST API、Webhook 钩子
- **路由面**: 将聊天请求分发给 Agent 运行时、将事件广播给连接的客户端
- **扩展面**: 插件 HTTP 路由注册、Gateway 方法注册、节点发现

### 架构层次

```
┌─────────────────────────────────────────────────┐
│                   CLI Layer                      │
│  (commands/gateway.ts -> server.ts -> impl)      │
├─────────────────────────────────────────────────┤
│              Gateway Core Layer                  │
│  ┌─────────────────────────────────────────┐    │
│  │        server-http.ts                   │    │
│  │   Ordered Stage Router (HTTP Pipeline)  │    │
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────┐    │
│  │     server/ws-connection.ts              │    │
│  │   WebSocket Connection Handler           │    │
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────┐ ┌───────────────────┐      │
│  │   auth.ts        │ │   hooks.ts        │      │
│  │   (认证系统)     │ │   (Webhook)       │      │
│  └─────────────────┘ └───────────────────┘      │
├─────────────────────────────────────────────────┤
│              Runtime Layer                       │
│  ┌──────────────┐ ┌────────────┐ ┌───────────┐  │
│  │ server-chat  │ │ channels   │ │ methods/   │  │
│  │ (聊天/事件)  │ │ (渠道管理) │ │ (RPC方法)  │  │
│  └──────────────┘ └────────────┘ └───────────┘  │
├─────────────────────────────────────────────────┤
│              Plugin Layer                        │
│  ┌──────────────┐ ┌────────────────────────┐    │
│  │ plugins/     │ │  server/plugins-http/  │    │
│  │ (插件注册表) │ │  (插件HTTP路由)        │    │
│  └──────────────┘ └────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

### 核心文件职责

| 文件 | 职责 |
|------|------|
| `server.ts` | 入口 facade，延迟加载 `server.impl.ts` |
| `server.impl.ts` | **核心实现** — 启动整个 Gateway 生命周期的编排者 (1784 行) |
| `server-http.ts` | HTTP 服务器、有序请求阶段路由器、WebSocket 升级处理 |
| `server-runtime-state.ts` | 创建运行时状态、HTTP Server、WS Server、广播器 |
| `auth.ts` / `auth-resolve.ts` | 认证模式解析和请求授权 |
| `connection-auth.ts` | 连接时凭据解析（含 Secrets Provider 集成） |
| `hooks.ts` | Webhook 系统 — 外部 HTTP POST 调用 Agent |
| `events.ts` | 事件常量定义 |
| `server-chat.ts` | Agent 事件处理器 — 将 agent 事件转为 chat 广播 |
| `server-channels.ts` | 渠道（channel）生命周期管理 |
| `client.ts` | Gateway 客户端（远程节点用） |
| `probe.ts` | 远程 Gateway 探测（健康诊断） |
| `startup-auth.ts` | 启动时认证配置合并 |
| `server-startup-config.ts` | 启动时配置加载与验证 |
| `server-startup-early.ts` | 早期运行时启动（Bonjour 发现等） |
| `server-startup-plugins.ts` | 启动时插件引导 |
| `server-startup-post-attach.ts` | HTTP 绑定后的运行时启动 |

---

## 2. 启动流程详解

Gateway 的启动是一个精心编排的多阶段过程，带有详细的性能追踪（`createGatewayStartupTrace`）。启动入口在 `server.impl.ts:540` 的 `startGatewayServer()` 函数。

### 启动阶段文本流程图

```
startGatewayServer(port, opts)
│
├─ 0. 前置准备
│    ├─ normalizeStateDirEnv()        # 标准化状态目录环境变量
│    ├─ bootstrapGatewayNetworkRuntime()  # 初始化网络运行时
│    └─ resumeGatewayRestartTrace()    # 恢复重启追踪
│
├─ 1. Config Snapshot ── [config.snapshot]
│    ├─ loadGatewayStartupConfigSnapshot()
│    │   ├─ readConfigFileSnapshotWithPluginMetadata()
│    │   ├─ 处理 legacyIssues
│    │   └─ 应用 pluginAutoEnable
│    │
│    └─ 返回 configSnapshot + pluginMetadataSnapshot
│
├─ 2. Auth Bootstrap ── [config.auth]
│    ├─ prepareGatewayStartupConfig()
│    │   ├─ mergeGatewayAuthConfig()        # 合并运行时 auth 覆盖
│    │   ├─ resolveGatewayAuth()            # 解析认证模式
│    │   ├─ assertGatewayAuthConfigured()   # 验证认证配置完整性
│    │   └─ 可能生成运行时 token
│    │
│    └─ 返回 cfgAtStart + generatedToken
│
├─ 3. Control UI Seed ── [control-ui.seed]
│    └─ maybeSeedControlUiAllowedOriginsAtStartup()
│
├─ 4. Plugin Bootstrap ── [plugins.bootstrap]
│    ├─ prepareGatewayPluginBootstrap()
│    │   ├─ runChannelPluginStartupMaintenance()
│    │   ├─ runStartupSessionMigration()
│    │   ├─ loadPluginLookUpTable()
│    │   └─ resolveGatewayPluginConfig()
│    │
│    └─ 返回 pluginRegistry + baseGatewayMethods
│
├─ 5. Runtime Config ── [runtime.config]
│    └─ resolveGatewayRuntimeConfig()    # 解析绑定地址、端口、TLS、控制UI等
│
├─ 6. TLS & State ── [tls.runtime]
│    ├─ loadGatewayTlsRuntime()
│    └─ createGatewayRuntimeState()      # 创建 HTTP Server、WS Server
│        ├─ createGatewayBroadcaster()
│        ├─ 创建懒加载的 hooks 处理器
│        ├─ 创建懒加载的 plugin HTTP 处理器
│        └─ attachGatewayUpgradeHandler()  # 附加 WS upgrade handler
│
├─ 7. Node Session Runtime
│    └─ createGatewayNodeSessionRuntime()
│
├─ 8. Live State ── [gateway.request-context]
│    ├─ createGatewayRequestContext()
│    ├─ createGatewayServerLiveState()
│    └─ setFallbackGatewayContextResolver()
│
├─ 9. Early Runtime ── [runtime.early]
│    ├─ startGatewayPluginDiscovery()   # Bonjour/mDNS 服务发现
│    ├─ startGatewayEarlyRuntime()
│    │   ├─ primeRemoteSkillsCache()
│    │   ├─ setSkillsRemoteRegistry()
│    │   └─ startGatewayMaintenanceTimers()
│    │
│    └─ 返回 bonjourStop + getActiveTaskCount
│
├─10. Runtime Subscriptions ── [runtime.subscriptions]
│    └─ startGatewayEventSubscriptions()
│
├─11. Runtime Services ── [runtime.services]
│    └─ startGatewayRuntimeServices()
│
├─12. Gateway Handlers ── [gateway.handlers]
│    └─ 创建 extraHandlers + coreGatewayHandlers
│
├─13. WS Attach ── [gateway.ws-attach]
│    └─ attachGatewayWsHandlers()
│        └─ attachGatewayWsConnectionHandler()
│
├─14. HTTP Listen ── [http.listen]
│    └─ listenGatewayHttpServer()
│
├─15. Post-Attach Runtime ── [runtime.post-attach]
│    ├─ startGatewayPostAttachRuntime()
│    │   ├─ Tailscale 服务暴露
│    │   ├─ 启动渠道 (startChannels)
│    │   ├─ 加载启动插件
│    │   ├─ 启动插件服务
│    │   └─ 回调: onSidecarsReady -> activateScheduledServices()
│    │
│    └─ 返回 stopGatewayUpdateCheck + tailscaleCleanup + pluginServices
│
├─16. Config Watcher
│    └─ startManagedGatewayConfigReloader()
│
├─17. Post-Ready Maintenance ── [ready]
│    ├─ scheduleGatewayPostReadyMaintenance()
│    │   ├─ tickInterval (周期性 tick)
│    │   ├─ healthInterval (健康检查)
│    │   └─ dedupeCleanup (去重清理)
│    │
│    └─ finishGatewayRestartTrace()
│
└─ return { close: async () => {...} }
```

### 启动追踪注入点

Gateway 启动阶段有详细的性能追踪，通过 `OPENCLAW_GATEWAY_STARTUP_TRACE` 环境变量启用。追踪信息可以通过两种机制查看：
- 标准错误输出: `[gateway] startup trace: ...`
- 诊断时间线事件: `emitDiagnosticsTimelineEvent()`

---

## 3. HTTP API 路由设计

Gateway 的 HTTP 路由是基于**有序阶段（Stage）管道**模型实现的，见 `server-http.ts:481` 的 `createGatewayHttpServer()`。

### 路由阶段顺序

请求进入后，按以下优先级依次尝试各阶段（`handleGatewayHttpRequestStages`）：

```
客户端请求
    │
    ▼
setDefaultSecurityHeaders()          # 设置安全头部 (CSP, HSTS 等)
    │
    ▼
parseGatewayRequestPath()            # 解析 URL pathname
    │
    ├── ❿ WebSocket Upgrade? ──→ 跳过 HTTP 处理，由 ws 处理
    │
    ▼
[Gateway Probes]                     # /health, /healthz, /ready, /readyz
    │  (GET/HEAD only, 返回 JSON)
    │
    ▼
[Hooks]                              # POST /hooks/... (Webhook)
    │  需 Bearer token 验证
    │
    ▼
[Models]                             # GET /v1/models, /v1/models/:id
    │  (OpenAI 兼容, 需启用 openAiChatCompletionsEnabled)
    │
    ▼
[Embeddings]                         # POST /v1/embeddings
    │
    ▼
[Tools Invoke]                       # POST /tools/invoke
    │
    ▼
[Sessions Kill]                      # POST /sessions/:id/kill
    │
    ▼
[Sessions History]                   # GET /sessions/:id/history
    │
    ▼
[OpenResponses]                      # POST /v1/responses
    │  (需启用 openResponsesEnabled)
    │
    ▼
[OpenAI Chat Completions]            # POST /v1/chat/completions
    │  (需启用 openAiChatCompletionsEnabled)
    │
    ▼
[Plugin Node Capability Auth]        # 节点能力认证
    │
    ▼
[Plugin Routes]                      # 插件注册的 HTTP 路由
    │  ├─ Plugin Auth (网关认证检查)
    │  └─ Plugin HTTP (分发到插件)
    │
    ▼
[Managed Image Attachments]          # GET /api/chat/media/outgoing/:id
    │
    ▼
[Control UI]                         # 控制界面
    ├─ Assistant Media
    ├─ Avatar
    └─ SPA 页面
    │
    ▼
404 Not Found                        # 无匹配
```

### 关键设计决策

1. **阶段短路**: 一旦某个 stage 返回 `true`，后续阶段不再执行。这确保核心内置路由优先级高于插件路由。
2. **插件容错**: 插件 HTTP 阶段配置了 `continueOnError: true`，即使插件抛出异常，控制 UI 等后续阶段仍然可达。
3. **懒加载**: 每个 HTTP handler 模块（models, embeddings, sessions 等）都是延迟加载的，`getXxxModule()` 模式确保只加载需要的模块。
4. **路径归一化**: 通过 `normalizePluginNodeCapabilityScopedUrl()` 在认证前将 scoped URL 转换为规范路径。

---

## 4. 认证与安全

Gateway 的认证系统是多模式的，从 `auth.ts` 和 `auth-resolve.ts` 实现。

### 认证模式

| 模式 | 说明 | 配置方式 |
|------|------|----------|
| `none` | 无认证 | `gateway.auth.mode: none` |
| `token` | Bearer Token | `gateway.auth.token` 或 `OPENCLAW_GATEWAY_TOKEN` |
| `password` | 密码认证 | `gateway.auth.password` 或 `OPENCLAW_GATEWAY_PASSWORD` |
| `trusted-proxy` | 信任反向代理身份 | `gateway.auth.trustedProxy` |
| `tailscale` | Tailscale 身份认证 | `gateway.auth.allowTailscale: true` (附加模式) |
| `device-token` | 设备令牌 | 设备配对流程后自动颁发 |
| `bootstrap-token` | 引导令牌 | 首次设置的临时令牌 |

### 认证流程 (`authorizeGatewayConnect`)

```
authorizeGatewayConnect(params)
    │
    ├── trusted-proxy mode?
    │    ├─ 验证 remoteAddr 在 trustedProxies 中
    │    ├─ 检查 requiredHeaders
    │    ├─ 读取 userHeader 获取用户身份
    │    └─ 可选 allowUsers 白名单
    │
    ├── none mode? ──→ 直接放行 (ok: true)
    │
    ├── 速率限制检查 (rejectIfRateLimited)
    │
    ├── Tailscale auth? (仅 ws-control-ui 表面)
    │    ├─ 读取 tailscale-user-login 等头部
    │    ├─ 调用 Tailscale Whois API 验证
    │    └─ 比对 login 是否一致
    │
    ├── token mode?
    │    └─ safeEqualSecret(connectToken, authToken)
    │
    └── password mode?
         └─ safeEqualSecret(connectPassword, authPassword)
```

### 速率限制

- 两种限流器: `authRateLimiter` (远程+HTTP)、`browserRateLimiter` (浏览器来源，无 loopback 豁免)
- 限流作用于共享秘密认证失败的尝试
- 使用 `withSerializedRateLimitAttempt` 防止 Tailscale 异步路径下的竞争条件

### 安全特性

- **常量时间比较**: `safeEqualSecret()` 防止时序攻击
- **速率限制**: 可配置的 `gateway.auth.rateLimit` 防止暴力破解
- **HSTS**: 通过 `strictTransportSecurityHeader` 配置
- **默认安全头部**: `setDefaultSecurityHeaders()` 设置 CSP 等
- **预认证连接预算**: `preauthConnectionBudget` 限制未认证连接数
- **Device Identity**: 非对称加密挑战-响应，防止设备冒充
- **已知弱密码检查**: `assertGatewayAuthNotKnownWeak()` 拒绝常见弱密码

### 认证表面（Auth Surface）

Gateway 区分两种认证表面：
- `http`: 标准 HTTP API 调用，禁用 Tailscale 头部认证
- `ws-control-ui`: 控制界面 WebSocket，启用 Tailscale 头部认证（允许无 token 的信任主机登录）

---

## 5. WebSocket 实时通信

WebSocket 是 Gateway 的核心通信机制，位于 `server/ws-connection.ts`。

### 连接生命周期

```
客户端                                   Gateway
  │                                        │
  │── HTTP Upgrade Request ──────────────→ │
  │     (携带认证 token)                    │
  │                                        │
  │◄── 101 Switching Protocols ─────────── │  (预认证连接预算消耗)
  │                                        │
  │── WebSocket Connect Frame ────────────→│
  │     (协议版本、客户端标识、认证)         │
  │                                        │
  │◄── Hello-OK Frame ─────────────────── │   (服务端特性声明)
  │     (methods, events, auth scopes)     │
  │                                        │
  │── Request/Response ──────────────────→│   (JSON-RPC 风格)
  │     (method, params, id)              │
  │                                        │
  │◄── Push Events ───────────────────────│
  │     (chat, agent, session events)      │
  │                                        │
  │── Close ──────────────────────────────→│   (清理、释放预算)
```

### 预认证连接预算 (`preauthConnectionBudget`)

这是一个重要机制，防止未经认证的连接耗尽服务器资源：

- 每个客户端 IP 在 upgrade 时获取一个预算槽
- budget 在连接认证完成后转移（认领）或关闭时释放
- 防止 DoS 攻击（大量未认证 WebSocket 连接）
- 实现文件: `server/preauth-connection-budget.ts`

### WebSocket 消息协议

消息格式基于包的 `gateway-protocol` 定义的帧类型：

```
请求:    { id, method, params }
响应:    { id, result } 或 { id, error }
推送:    { event, data }
系统:    TickEvent (心跳), ShutdownEvent (关闭通知)
```

### 广播系统

Gateway 使用 `createGatewayBroadcaster()` 创建广播器，支持三种广播模式：

1. **全局广播** (`broadcast`): 发送给所有连接的 WS 客户端
2. **选择性广播** (`broadcastToConnIds`): 发送给指定的连接 ID 集合
3. **节点发送** (`nodeSendToSession`): 发送给远程节点上的特定 session

---

## 6. 事件系统

Gateway 的事件系统处理 agent 运行过程中的各类事件，核心实现在 `server-chat.ts`。

### 事件类型

Agent 事件流 (`stream` 字段):
- `lifecycle`: start, end, error
- `assistant`: 助手文本增量
- `thinking`: 思考过程文本增量
- `tool`: 工具调用生命周期 (start, result, error)
- `item`: 通用项生命周期
- `status`: 状态更新

### 事件处理流程

```
AgentEvent
    │
    ▼
createAgentEventHandler()  (server-chat.ts:258)
    │
    ├── 事件排序 (seq gap 检测)
    ├── 文本节流 (150ms 缓冲区)
    │
    ├── Chat Delta (emitChatDelta)
    │   ├── normalizeLiveAssistantEventText()
    │   ├── 合并缓冲文本
    │   ├── 150ms 节流
    │   └── broadcast("chat", payload)
    │
    ├── Agent 事件 (sendAgentPayload)
    │   ├── 文本节流 (coalescing)
    │   ├── 广播到 Control UI
    │   └── 发送到节点 session
    │
    ├── Tool 事件
    │   ├── runToolRecipients 广播
    │   └── sessionEventSubscribers 广播
    │
    └── Lifecycle 结束
        ├── emitChatFinal("done" | "error")
        └── persistGatewaySessionLifecycleEvent()
```

### 订阅系统

- **sessionEventSubscribers**: 监听 session 变更事件（用于 Control UI）
- **sessionMessageSubscribers**: 监听会话消息事件（用于节点通信）
- **toolEventRecipients**: 按 runId 订阅工具事件（用于 Control UI 实时工具卡片）

### 心跳与节流

- 心跳文本在 `shouldHideHeartbeatChatOutput()` 中被抑制
- Chat delta 使用 150ms 节流窗口
- Agent 文本事件使用类似机制合并小增量

---

## 7. 远程节点管理

Gateway 通过 `createGatewayNodeSessionRuntime()` 管理远程节点。

### 节点注册表 (`nodeRegistry`)

```typescript
// server.impl.ts:927
const {
  nodeRegistry,
  nodePresenceTimers,
  sessionEventSubscribers,
  sessionMessageSubscribers,
  nodeSendToSession,
  nodeSendToAllSubscribed,
  nodeSubscribe,
  nodeUnsubscribe,
  hasTalkNodeConnected,
} = createGatewayNodeSessionRuntime({ broadcast });
```

### 节点通信

- **节点连接**: 远程节点通过 Gateway Client 连接到 Gateway 的 WebSocket
- **Session 订阅**: 节点可以订阅特定 session 的事件
- **Voice Wake**: 通过 `broadcastVoiceWakeChanged` 和 `broadcastVoiceWakeRoutingChanged` 管理语音唤醒路由
- **Skill 远程调用**: 通过 `skills/runtime/remote.ts` 注册远程节点可用的 skills

### 节点发现

- **Bonjour/mDNS**: `server-discovery-runtime.ts` 在局域网内广播 Gateway 服务
- **Tailscale**: 通过 Tailscale 暴露服务，自动进行 NAT 穿透
- **Wide Area Discovery**: 配置 `discovery.wideArea.enabled` 开启广域网发现

### 探针 (`probe.ts`)

`probeGateway()` 用于从 CLI 或其他进程探测远程 Gateway 状态：

1. 创建 GatewayClient (probe mode)
2. 连接并获取 Hello-OK
3. 依次请求 health, status, system-presence, config.get
4. 返回连接延迟、认证能力、健康状态等

---

## 8. Gateway 协议

协议定义在 `packages/gateway-protocol/` 和 `packages/gateway-client/` 中。

### 协议层级

```
┌──────────────────────────────────────┐
│        Gateway Client                │
│  (packages/gateway-client/)          │
│  自动重连、请求队列、挑战-响应      │
├──────────────────────────────────────┤
│        Gateway Protocol              │
│  (packages/gateway-protocol/)        │
│  帧类型、方法 Schema（TypeBox）     │
├──────────────────────────────────────┤
│        WebSocket (ws package)        │
│  传输层、二进制帧                    │
└──────────────────────────────────────┘
```

### 方法 Schema

协议使用 **TypeBox** 定义所有 RPC 方法的参数和结果 Schema，位于 `gateway-protocol/src/schema/`：

| Schema 文件 | 包含的方法 |
|-------------|-----------|
| `agent.ts` | agent 相关方法 |
| `sessions.ts` | 会话管理 |
| `channels.ts` | 渠道管理 |
| `config.ts` | 配置读写 |
| `cron.ts` | 定时任务 |
| `devices.ts` | 设备配对 |
| `nodes.ts` | 节点管理 |
| `exec-approvals.ts` | 执行审批 |
| `secrets.ts` | 密钥管理 |
| `commands.ts` | 命令管理 |
| `push.ts` | 推送通知 |
| `artifacts.ts` | 产物管理 |

### 客户端实现

`src/gateway/client.ts` 是对 `packages/gateway-client` 的封装，添加了 OpenClaw 运行时特有的功能：

- 设备身份管理 (`loadOrCreateDeviceIdentity`, `signDevicePayload`)
- 设备令牌持久化 (`loadDeviceAuthToken`, `storeDeviceAuthToken`)
- 代理生命周期管理 (`ensureInheritedManagedProxyRoutingActive`)
- 探针客户端 (`GatewayClient` with mode `PROBE`)
- 关闭码描述 (`describeGatewayCloseCode`)

### Connect 帧结构

```typescript
// packages/gateway-protocol/src/schema/frames.ts
ConnectParams = {
  minProtocol, maxProtocol,        // 协议版本协商
  client: { id, version, mode },   // 客户端标识
  auth?: { token, deviceToken },   // 认证凭据
  device?: { id, publicKey, signature, nonce },  // 设备身份
  caps, commands, scopes           // 能力声明
}
```

### Hello-OK 帧结构

```typescript
HelloOk = {
  protocol,                        // 选定的协议版本
  server: { version, connId },     // 服务端信息
  features: { methods, events },   // 支持的方法和事件列表
  auth: { role, scopes, deviceToken },  // 认证结果
  snapshot,                        // 状态快照
  pluginSurfaceUrls                // 插件 HTTP 端点 URL
}
```

---

## 9. 关键设计决策

### 9.1 有序阶段管道（HTTP Stage Pipeline）

`server-http.ts` 中采用 ordered stages 而非传统路由表。优点：
- 明确的优先级控制（核心路由 > 插件路由 > 控制 UI）
- 每个 stage 可独立容错（`continueOnError`）
- 轻松的测试和调试（每个 stage 命名且可独立 mock）
- 懒加载的 handler 模块，减少启动内存

### 9.2 插件 HTTP 路由的认证分离

插件 HTTP 请求认证被拆分为两个独立阶段（`buildPluginRequestStages`）：
1. `plugin-auth`: 检查是否需要认证并执行
2. `plugin-http`: 将包含认证上下文的分发到插件处理器

这种分离确保即使插件崩溃（`continueOnError`），其他核心路由仍然可达。

### 9.3 预认证连接预算（DoS 防护）

在未认证的 WebSocket 连接建立前就消耗预算槽，防止攻击者通过大量连接消耗文件描述符。设计类似 TCP 的 SYN cookie 概念。

### 9.4 多认证表面（Dual Auth Surface）

区分 `http` 和 `ws-control-ui` 两种认证表面，允许控制 UI WebSocket 使用 Tailscale 头部认证（无需提前配置 token），而 HTTP API 则要求显式凭证。

### 9.5 事件节流与合并

- Chat 文本使用 150ms 窗口合并增量广播
- Agent 文本事件使用 `mergeBufferedAgentPayload` 合并小增量
- 心跳文本根据配置选择性地抑制（`shouldHideHeartbeatChatOutput`）
- 避免在高速流式传输中产生过多消息

### 9.6 有状态重启和无缝升级

Gateway 支持通过 `restart-handoff` 机制实现有状态重启：
- 重启追踪（`restart-trace.ts`）记录各阶段耗时
- 配置变更后通过 `startManagedGatewayConfigReloader` 热重载
- Shared session generation 机制确保认证状态变更时 WS 连接被正确回收

### 9.7 Gateway 懒加载架构

整个 Gateway 大量使用动态 `import()` 和模块缓存模式：

```typescript
// 典型的懒加载模式
let someModulePromise: Promise<typeof import("./some-module.js")> | null = null;
function getSomeModule() {
  someModulePromise ??= import("./some-module.js");
  return someModulePromise;
}
```

这确保在 Gateway 启动时只加载真正需要的模块，减少了启动时间和内存占用。

---

## 10. 建议添加注释的位置

以下目录值得深入研究，并可以添加详细注释以辅助理解：

### 10.1 `src/gateway/server-http.ts`

- **`createGatewayHttpServer()`** (L481): 核心 HTTP 阶段管道设计，建议详细注释每个 stage 的优先级理由和 fallthrough 逻辑
- **`createGatewayHttpRequestStages()`** (L400): 插件 stage 的构建逻辑，建议解释为什么 plugin-auth 和 plugin-http 分离
- **`attachGatewayUpgradeHandler()`** (L826): WebSocket upgrade 的复杂认证流程和预算管理

### 10.2 `src/gateway/server.impl.ts`

- **`startGatewayServer()`** (L540): 整个启动流程的编排，建议添加阶段编号和依赖关系注释
- **`createGatewayStartupTrace()`** (L223): 启动性能追踪的实现，建议解释各 trace point 的含义
- **`reloadAttachedGatewayPlugins()`** (L1270): 插件热重载逻辑，涉及的编排相当复杂

### 10.3 `src/gateway/auth.ts`

- **`authorizeGatewayConnectCore()`** (L478): 认证决策树，建议添加每种模式的流转条件
- **`authorizeTrustedProxy()`** (L302): 信任代理认证的详细安全语义
- **`resolveVerifiedTailscaleUser()`** (L220): Tailscale 身份验证的 whois 校验逻辑

### 10.4 `src/gateway/server-chat.ts`

- **`createAgentEventHandler()`** (L258): Agent 事件处理器的核心事件循环，建议标注事件流的生命周期
- **`emitChatDelta()`** (L571): 增量文本广播的节流和合并机制
- **`finalizeLifecycleEvent()`** (L447): 生命周期结束时的清理和广播逻辑

### 10.5 `src/gateway/server/ws-connection.ts`

- **WS 连接建立流程**: 从 upgrade 到 hello-ok 的完整握手
- **`preauthConnectionBudget`**: 预算的获取、持有和释放时序

### 10.6 `packages/gateway-protocol/src/schema/frames.ts`

- Connect 和 Hello-OK 的协议帧结构说明
- 协议版本协商机制
- device identity 挑战-响应流程

### 10.7 `src/gateway/probe.ts`

- **`probeGateway()`**: 远程探针的完整生命周期，设备身份缓存的短路逻辑
- **`resolveGatewayProbeCapability()`**: 能力推导逻辑

### 10.8 `src/gateway/client.ts`

- `GatewayClient` 对 `BaseGatewayClient` 的封装职责
- hostDeps 注入的函数职责说明

---

## 附录: 关键配置项

```jsonc
{
  "gateway": {
    "port": 18789,                           // 默认端口
    "bind": "loopback",                      // loopback | lan | tailnet | auto
    "auth": {
      "mode": "token",                       // none | token | password | trusted-proxy
      "token": "xxx",                        // 共享秘密
      "allowTailscale": true,                // 允许 Tailscale 身份认证
      "rateLimit": {                         // 速率限制
        "maxAttempts": 5,
        "windowMs": 30000
      },
      "trustedProxy": {                      // 信任代理配置
        "userHeader": "X-Forwarded-User",
        "allowUsers": ["admin"]
      }
    },
    "tls": {
      "enabled": false,
      "cert": "...",
      "key": "..."
    },
    "controlUi": {
      "enabled": true,
      "basePath": "/"
    },
    "http": {
      "endpoints": {
        "chatCompletions": { "enabled": false },  // OpenAI 兼容 API
        "responses": { "enabled": false }         // OpenResponses API
      }
    },
    "trustedProxies": ["127.0.0.1"],         // 信任的反向代理 IP
    "handshakeTimeoutMs": 10000              // WS 握手超时
  }
}
```

---

## 附录: Gateway 文件地图

```
src/gateway/
├── boot.ts                          # BOOT.md 引导执行
├── client.ts                        # Gateway 客户端封装
├── connection-auth.ts               # 连接凭据解析
├── events.ts                        # 事件常量
├── hooks.ts                         # Webhook 钩子系统
├── probe.ts                         # 远程 Gateway 探针
├── server.ts                        # 入口 Facade
├── server.impl.ts                   # 核心实现 (1784 行)
├── server-http.ts                   # HTTP 服务器和阶段管道
├── server-runtime-state.ts          # 运行时状态创建
├── server-ws-runtime.ts             # WS 处理器包装
├── server-chat.ts                   # Agent 事件处理
├── server-channels.ts               # 渠道生命周期管理
├── server-close.runtime.ts          # 关闭处理
├── server-broadcast.ts              # 广播器
├── server-startup-config.ts         # 启动配置加载
├── server-startup-early.ts          # 早期运行时启动
├── server-startup-plugins.ts        # 插件引导
├── server-startup-post-attach.ts    # 绑定后启动
├── startup-auth.ts                  # 启动认证配置
├── startup-control-ui-origins.ts    # 控制 UI 来源配置
├── auth.ts / auth-resolve.ts        # 认证系统
├── auth-rate-limit.ts               # 速率限制
├── net.ts                           # 网络工具
├── server/
│   ├── ws-connection.ts             # WS 连接处理
│   ├── hooks.ts                     # Webhook 请求处理
│   ├── plugins-http.ts              # 插件 HTTP 路由
│   ├── plugins-http/                # 插件 HTTP 子模块
│   ├── readiness.ts                 # 就绪检查
│   ├── health-state.ts              # 健康状态
│   ├── preauth-connection-budget.ts # 预认证连接预算
│   ├── event-loop-health.ts         # 事件循环健康
│   ├── tls.ts                       # TLS 配置
│   └── ...
├── methods/                         # Gateway RPC 方法
│   ├── registry.ts                  # 方法注册表
│   ├── core-descriptors.ts          # 核心方法描述
│   └── ...
└── ...
```
