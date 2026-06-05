# OpenClaw 渠道系统深度分析

## 渠道系统架构总览

OpenClaw 的"渠道"(Channel)是其与外部即时通讯(IM)平台通信的抽象层。每个 Channel 代表一个消息平台（Telegram、Slack、Discord、iMessage 等），系统采用**插件化架构**将平台差异封装在插件内部，核心只定义通用契约。

### 分层架构

```
  外部 IM 平台 (Telegram, Slack, Discord, 微信 ...)
       |
   [传输层] transport/         -- webhook / gateway 长连接 / 轮询
       |
   [渠道插件] plugins/         -- 每个平台一个 ChannelPlugin 实例
       |
   [入站事件管道]              -- 消息标准化、去重、路由
       |
   [Agent 引擎]               -- AI 推理、工具调用、回复生成
       |
   [出站消息管道]              -- 渲染、流式输出、耐用发送
       |
   [渠道插件] plugins/         -- 通过 message 适配器发送
       |
   [传输层] transport/
       |
  外部 IM 平台
```

### 核心目录结构

| 目录/文件 | 职责 |
|-----------|------|
| `src/channels/plugins/` | 渠道插件注册表、类型定义、适配器契约 |
| `src/channels/message/` | 消息收发核心：入站队列、出站发送、耐用性、流式预览 |
| `src/channels/transport/` | 传输层看门狗（超时检测） |
| `src/channels/streaming.ts` | 流式输出配置、进度草稿渲染 |
| `src/channels/mention-gating.ts` | @提及门控逻辑 |
| `src/channels/targets.ts` | 消息目标寻址（用户/频道解析） |
| `src/channels/session.ts` | 会话记录与路由跟踪 |
| `src/channels/typing.ts` | 输入状态指示器（正在输入...） |
| `src/channels/channel-config.ts` | 渠道配置匹配（slug 规范化） |
| `src/plugin-sdk/` | 对外 SDK 接口（插件开发者面向） |
| `src/gateway/server-channels.ts` | Gateway 侧的渠道生命周期管理 |

---

## 核心 Channel 接口设计

### ChannelPlugin -- 总接口

文件: `src/channels/plugins/types.plugin.ts:61-106`

```typescript
type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter<ResolvedAccount>;
  setup?: ChannelSetupAdapter;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  outbound?: ChannelOutboundAdapter;
  message?: ChannelMessageAdapterShape;
  messaging?: ChannelMessagingAdapter;
  actions?: ChannelMessageActionAdapter;
  threading?: ChannelThreadingAdapter;
  mentions?: ChannelMentionAdapter;
  streaming?: ChannelStreamingAdapter;
  // ... 还有十几个可选适配器
};
```

**设计亮点**：
- 每个 ChannelPlugin 被设计为**纯适配器对象的集合**，没有抽象基类或继承要求。这是组合优于继承的极致实践。
- 所有适配器都是**可选**的——插件只需实现其平台需要的部分。
- `ResolvedAccount` 泛型让每个插件定义自己的账户模型，核心不感知具体字段。

### 适配器分解

ChannelPlugin 将平台行为分解为 20+ 个独立的适配器接口，每个负责一个正交维度：

| 适配器 | 类型文件 | 职责 |
|--------|---------|------|
| `ChannelConfigAdapter` | `types.adapters.ts:116` | 账户列表、启用/禁用、配置状态 |
| `ChannelGatewayAdapter` | `types.adapters.ts:337` | 启动/停止长连接网关 |
| `ChannelMessageAdapterShape` | `message/types.ts:391` | 消息发送、耐用性、流式、接收确认 |
| `ChannelMessagingAdapter` | `types.core.ts:505` | 目标解析、会话路由、回复转换 |
| `ChannelMessageActionAdapter` | `types.core.ts:749` | message tool 的 action 发现与执行 |
| `ChannelSetupAdapter` | `types.adapters.ts:76` | CLI/交互式安装流程 |
| `ChannelThreadingAdapter` | `types.core.ts:422` | 线程绑定与回复模式 |
| `ChannelStreamingAdapter` | `types.core.ts:379` | 流式输出 coalesce 参数 |
| `ChannelSecurityAdapter` | `types.adapters.ts:838` | DM 策略、安全审计 |
| `ChannelDoctorAdapter` | `types.adapters.ts:506` | 配置修复与迁移 |
| `ChannelLifecycleAdapter` | `types.adapters.ts:541` | 配置变更钩子、启动维护 |

---

## 消息标准化流程

不同 IM 平台的消息格式差异巨大（文本、媒体、富文本、交互组件等），OpenClaw 通过分层抽象实现统一。

### 入站消息标准化

1. **平台原始负载**由渠道插件的 gateway/transport 接收
2. 通过 `buildChannelInboundEventContext()` (`src/channels/inbound-event/context.ts`) 组装成标准化 `MsgContext`
3. 标准化内容包括：
   - `Body` / `RawBody` / `BodyForAgent` -- 文本内容，区分原始与清洗后
   - `SenderName` / `SenderId` / `SenderUsername` -- 发送者身份
   - `ChatType` -- `"direct" | "group" | "channel"`
   - `WasMentioned` -- @提及检测
   - `Media` -- 媒体附件标准化
   - `SupplementalContext` -- 引用/转发/线程上下文

### 出站消息标准化

出站消息通过 `ReplyPayload` (`src/auto-reply/reply-payload.ts`) 统一表示，然后通过渠道的 `message` 适配器发送：

```
Agent 回复 (文本/媒体/交互)
    |
    v
ReplyPayload (标准化)
    |
    v
ChannelMessageSendAdapter
    |-- send.text()      -- 纯文本发送
    |-- send.media()     -- 媒体发送
    |-- send.payload()   -- 结构化负载发送
    |-- send.poll()      -- 投票发送
    |
    v
平台原生 API
```

### 消息耐用性 (Durable Final Delivery)

文件: `src/channels/message/types.ts`

系统定义了 `MessageDurabilityPolicy` 三种等级：
- `"required"` -- 必须成功发送，否则重试
- `"best_effort"` -- 尽力发送，失败可忽略
- `"disabled"` -- 不保证送达

适配器通过 `DurableFinalDeliveryRequirementMap` 声明自己支持的耐用能力（text、media、replyTo、thread、batch 等），核心通过 `deriveDurableFinalDeliveryRequirements()` (`capabilities.ts`) 计算出一条消息需要哪些能力，然后校验适配器是否满足。

---

## 入站消息处理链（含文本流程图）

```
IM Platform (Telegram / Slack / Discord)
    |
    | webhook / gateway 长连接 / 轮询
    v
[Ingress Queue]  src/channels/message/ingress-queue.ts
    | SQLite 持久化队列，去重防重复
    |
    v
[durable-receive] src/channels/message/durable-receive.ts
    | 耐用接收日志，标记完成/失败
    |
    v
[Inbound Event Classification] src/channels/inbound-event/classification.ts
    | 分类事件：user_request (需回复) vs room_event (静默)
    |
    v
[Inbound Event Context] src/channels/inbound-event/context.ts
    | 组装标准化 MsgContext
    | - 引用上下文适配 (quote/forward/thread)
    | - 媒体附件标准化
    | - 安全上下文过滤 (contextVisibility)
    |
    v
[Mention Gating] src/channels/mention-gating.ts
    | @提及门控决策：
    | - 是否需要 @提及才回复？
    | - 是否隐式提及（回复bot/引用bot）？
    | - 是否文本命令绕过？
    |
    v
[Command Gating] src/channels/command-gating.ts
    | 命令检测与授权
    |
    v
[Conversation Resolution] src/channels/conversation-resolution.ts
    | 会话绑定解析
    |
    v
[Session Recording] src/channels/session.ts
    | 记录入站会话 + 最后路由更新
    |
    v
[Agent Dispatch]
    | 进入 Agent 推理循环
    v
```

**关键设计点**：

1. **去重优先**：`ChannelIngressQueue` 使用 SQLite 持久化队列，每个入站事件有唯一 id，防止 webhook 重试导致重复处理
2. **阶段确认 (Ack Policy)**：`MessageReceiveContext` 支持 `after_receive_record` / `after_agent_dispatch` / `after_durable_send` / `manual` 四种确认策略，控制何时向平台确认消息已处理
3. **事件分类**：`classifyChannelInboundEvent()` 区分需要 Agent 处理的 `user_request` 和仅需记录的 `room_event`，避免群聊中每条消息都触发推理

---

## 出站消息处理链（含文本流程图）

```
Agent 完成推理，产生回复
    |
    v
[Reply Payload 组装]
    | - 文本内容
    | - 媒体附件
    | - 交互组件 (buttons, selects)
    | - channelData (平台特有数据)
    |
    v
[Channel Reply Pipeline] src/channels/message/reply-pipeline.ts
    | - 回复前缀 (sender attribution)
    | - 输入状态指示器 (typing indicator)
    | - ReplyPayload 转换 (插件可定制)
    |
    v
[Streaming / Progress Draft] src/channels/streaming.ts
    | 流式输出逻辑：
    | - block 模式：逐块发送
    | - progress 模式：发送进度草稿
    | - 本地流式：编辑已发送的预览消息
    |
    v
[Live Message Preview] src/channels/message/live.ts
    | Live Preview 最终化逻辑：
    | - 预览草稿编辑为最终消息
    | - 失败时回退到正常发送
    |
    v
[ChannelMessageSendAdapter] src/channels/message/types.ts
    | send.text / send.media / send.payload / send.poll
    |
    v
[Outbound Bridge] src/channels/message/outbound-bridge.ts
    | 向后兼容层：将旧 outbound 适配器桥接为 message 适配器
    |
    v
[Message Receipt] src/channels/message/receipt.ts
    | 标准化发送结果回执
    |
    v
[Durable Send Commit]
    | 耐用发送确认 + 状态持久化
    |
    v
[Platform API]
```

---

## 流式输出机制

流式输出是渠道系统最复杂的部分之一，代码集中在 `src/channels/streaming.ts` 和 `src/channels/message/live.ts`。

### 三种流式模式

由 `StreamingMode` 控制：

1. **partial** (分块)：文本逐步追加发送（如 Slack 的逐步更新消息）
2. **block** (块式)：每生成一段完整文本就发送一个独立消息块
3. **progress** (进度)：在工具调用期间显示进度状态（如"正在搜索..."、"正在执行..."）

### Progress Draft 渲染

`src/channels/streaming.ts` 实现了完整的进度草稿渲染管线：

- `buildChannelProgressDraftLine()` -- 将工具调用、命令执行、文件修改等事件渲染为可读文本行
- 支持 `event` 类型：`tool`、`item`、`plan`、`approval`、`command-output`、`patch`
- 支持丰富格式：emoji 前缀、状态标签、详细内容、截断逻辑
- `mergeChannelProgressDraftLine()` -- 去重合并，相同 id 的行替换而非追加
- `compactChannelProgressDraftLine()` -- 按最大字符数截断文本

### Live Preview 最终化

`live.ts` 的 `deliverFinalizableLivePreview()` 实现了关键的"预览->最终化"流程：

```
Final 阶段到来
    |
    ├─ draft.id() 有效?
    |  ├─ 是: buildFinalEdit() -> editFinal() 编辑预览消息为最终
    |  |   ├─ 编辑成功: markLiveMessageFinalized()
    |  |   ├─ 编辑失败: handlePreviewEditError()
    |  |   |   ├─ "retain": 保留预览为最终状态
    |  |   |   └─ "fallback": 丢弃预览，重新发送
    |  └─ 否: deliverNormally() 正常发送
    |
    └─ final 之前: deliverNormally() 直接发送
```

这个设计解决了 IM 平台上"先发预览再编辑为最终"的经典问题，同时处理了编辑失败、超时等边界。

### 流式配置读取

`streaming.ts` 包含大量 `resolveChannel*` 函数，从渠道配置中读取流式参数，支持：

- **多层配置合并**：插件配置层面有 `streaming` 字段，config 层面有 `ChannelStreamingConfig`
- **向后兼容**：支持旧的 `blockStreaming`、`draftChunk`、`nativeStreaming` 等字段名
- **coalesce 配置**：块式流式时，可配置最小字符数和空闲时间来决定何时发送下一个块

### Typing Indicator (输入状态指示器)

`src/channels/typing.ts` 实现了带安全机制的 typing indicator：

- **Keepalive 循环**：每 3 秒发送一次 typing 信号，保持聊天中的"正在输入"状态
- **TTL 安全机制**：默认 60 秒自动停止，防止 Agent 卡死导致永久"正在输入"
- **熔断保护**：连续失败 `maxConsecutiveFailures` 次后停止 keepalive
- **状态机**：`onReplyStart` -> keepalive -> `onIdle`/`onCleanup` -> 停止

---

## 传输层设计

### 现状

`src/channels/transport/` 当前只包含一个**看门狗 (Watchdog)**：

`stall-watchdog.ts` 实现了 `ArmableStallWatchdog`：
- `arm()` -- 启动监视，记录最后活动时间
- `touch()` -- 更新最后活动时间
- `disarm()` -- 解除监视
- 周期检查（默认 timeout/6，5s-250ms 范围内）
- 超过 `timeoutMs` 无活动则触发 `onTimeout` 回调
- 支持 `AbortSignal` 优雅停止

看门狗用于检测传输层卡死（如底层 WebSocket 静默断开、轮询无响应等）。

### 传输层架构决策

根据 AGENTS.md 的渠道原则：
- **传输/运行时策略与插件面向的助手对齐**：暴露给插件的传输相关功能通过 `plugin-sdk` 出口
- **渠道插件的 gateway 适配器** (`ChannelGatewayAdapter`) 负责实际的网络连接管理
- **StallWatchdog** 是核心提供的唯一传输层基础设施，插件使用它而非重新实现

---

## 目标寻址

文件: `src/channels/targets.ts`

### MessagingTarget 模型

```typescript
type MessagingTarget = {
  kind: "user" | "channel";
  id: string;
  raw: string;
  normalized: string; // "user:xxx" 或 "channel:xxx"
};
```

### 解析策略

系统支持多种目标解析方式：

1. **Mention 模式**：`@username` -> 匹配正则提取用户 ID
2. **前缀模式**：`user:xxx`、`channel:xxx` 等带前缀的显式目标
3. **@用户模式**：`@<id>` 格式

`parseMentionPrefixOrAtUserTarget()` (`targets.ts:106`) 按以下优先级解析：
1. `mentionPattern` 正则匹配（平台特有的 @提及格式）
2. 前缀列表匹配（`user:` / `channel:` / 平台特有前缀）
3. `@username` 格式

### 目标寻址在插件专有端的扩展

`ChannelMessagingAdapter` (`types.core.ts:505`) 提供了插件层面的目标解析扩展：
- `targetPrefixes` -- 插件特有的目标前缀
- `normalizeTarget` -- 自定义目标标准化
- `targetResolver` -- 目录查找失败后的后备解析器
- `resolveOutboundSessionRoute` -- 出站会话路由构建

---

## @提及与门控

文件: `src/channels/mention-gating.ts`

### 核心函数: `resolveInboundMentionDecision()`

```typescript
function resolveInboundMentionDecision(params: {
  facts: InboundMentionFacts;     // 事实层: 能否检测@提及、是否被提及、隐式提及类型
  policy: InboundMentionPolicy;    // 策略层: 是否是群聊、是否需要提及、文本命令策略
}): InboundMentionDecision
```

### 隐式提及 (Implicit Mention) -- 关键设计

系统不要求每个平台都能检测 @提及。对于无法检测@提及的平台（如某些 SMS 桥接），或者 Agent 已经参与的场景，定义了**隐式提及**：

```typescript
type InboundImplicitMentionKind =
  | "reply_to_bot"        // 回复了 bot 的消息
  | "quoted_bot"          // 引用了 bot 的消息
  | "bot_thread_participant" // bot 已参与的线程
  | "native";             // 平台原生 @提及 (兼容旧接口)
```

### 绕过决策 (Bypass)

`resolveInboundMentionDecision()` 内部的 `shouldBypassMention` 逻辑：

```
群聊 + 要求@提及 + 未被提及 + 无任何提及 + 允许文本命令 + 命令已授权 + 有控制命令
    => 绕过 @提及检查
```

这解决了"群聊中使用 /command 无需 @bot"的常见需求。

### 架构分层

1. **事实层 (Facts)**：由渠道插件提供（平台是否支持 @检测、消息中是否包含 @）
2. **策略层 (Policy)**：由核心配置和 Agent 配置决定（群聊策略、命令策略）
3. **决策层**：`resolveMentionDecisionCore()` 计算最终结果（`shouldSkip`）

### 兼容性设计

保留了旧的 `resolveMentionGating()` 和 `resolveMentionGatingWithBypass()`，标注 `@deprecated` 并指向新的 `resolveInboundMentionDecision()`，确保旧插件正常工作。

---

## 关键设计决策

以下决策参照了 `AGENTS.md` 中的渠道原则：

### 1. 纯适配器组合而非继承

**位置**: `types.plugin.ts:61`

每个 ChannelPlugin 是一个适配器对象集合，没有基类。这允许：
- 插件只实现自己需要的适配器
- 核心不依赖插件的内部实现
- 适配器之间没有隐式耦合

这与 AGENTS.md 的"Keep code small and clean"、"Prefer composition over inheritance"一致。

### 2. 渠道是纯传输层，不拥有业务逻辑

**原则**: "Message/channel plugins stay transport-only."

渠道插件不应实现产品命令树、插件策略或特性菜单。它们只负责：
- 消息格式映射 (transport formatting)
- 传输限制处理
- 原生回调映射

业务逻辑（如命令处理、权限检查）由核心层统一处理。

### 3. 消息 Tool 标准化

**原则**: "Portable command UI must use typed presentation actions."

渠道插件的 `message` 工具通过 `ChannelMessageActionAdapter.describeMessageTool()` 声明其能力（action、capability、schema），核心在 Agent tool 中统一暴露，不依赖渠道插件的私有实现。

### 4. 消息耐用性的显式契约

`DurableFinalDeliveryRequirementMap` 让渠道插件**显式声明**自己支持哪些发送能力，核心据此决定发送策略。这避免了：
- 核心猜测插件能力的隐含约定
- 运行时因能力不匹配而静默失败
- 插件遗漏实现必要功能

### 5. 入站/出站分离

入站处理（`inbound-event/`、`ingress-queue.ts`）和出站处理（`message/`、`streaming.ts`）在代码组织上明确分离，仅在 `session.ts` 记录会话时交叉。

### 6. 向后兼容性分层

多处可见 `@deprecated` 兼容模式：
- `channel-runtime.ts` -- 将所有老 API 重新导出，标注已弃用
- `outbound-bridge.ts` -- `createChannelMessageAdapterFromOutbound()` 将旧的 outbound 适配器包装为新的 message 适配器
- `mention-gating.ts` -- `resolveMentionGating()` 保留但弃用

这确保了升级路径平滑，新旧插件共存。

### 7. Gateway 生命周期管理

`server-channels.ts` 实现了自动重启、熔断和优雅关闭：
- 指数退避重试（初始 5s，最大 5min，因子 2，10% 抖动）
- 最大 10 次重启尝试
- 手动停止标记不自动重启（`manuallyStopped` Set）
- 带超时的优雅关闭（5s）

### 8. 热路径懒加载

根据 `src/channels/CLAUDE.md`：
- 渠道入口文件（`channel.ts`、`shared.ts`、`gateway.ts`、`outbound.ts`）不能静态引入异步负载
- 路由协议、镜像校验、设置流等放在专用的 `*.runtime.ts` 文件中
- `createChannelReplyPipeline()` 中的 `transformReplyPayload` 也是延迟加载插件，避免热路径引入插件注册表

---

## 建议添加注释的位置列表

以下位置对于代码维护者来说尤其需要注释，因为它们涉及非显而易见的跨路径状态、生命周期顺序、退避策略等：

1. **`src/channels/streaming.ts:144-149`** -- `isPotentialTruncatedFinal()` 的"截断检测"逻辑：说明为什么用 trailing ellipsis 检测 + 最小字符阈值来判定最终文本可能被截断
2. **`src/channels/streaming.ts:835-852`** -- `compactProgressLineDetail()` 的中间截断策略：说明为什么保留前 45% + 后若干字符而非简单地截断尾部
3. **`src/channels/mention-gating.ts:104-127`** -- `resolveMentionDecisionCore()` 的 `shouldSkip` 逻辑：说明 `requireMention && canDetectMention && !effectiveWasMentioned` 三者同时为真时跳过的原因
4. **`src/channels/typing.ts:68-80`** -- TTL 安全机制：说明为什么需要 60 秒默认超时和异常警告日志
5. **`src/gateway/server-channels.ts:29-36`** -- `CHANNEL_RESTART_POLICY` 的退避参数：解释指数退避 + 抖动的选择依据和最大重启次数
6. **`src/channels/inbound-event/context.ts:410-426`** -- `resolveUntrustedStructuredContext()` 中 `group_prompt_context` 的处理：说明为什么用户控制的群提示词不能进入 `GroupSystemPrompt`，只能作为不受信任上下文
7. **`src/channels/message/live.ts:153-201`** -- `deliverFinalizableLivePreview()` 中的 "retain" 分支：说明为什么编辑失败时可以选择"保留预览"而非总是回退
8. **`src/channels/channel-config.ts:44-46`** -- `normalizeChannelSlug()` 的规范化规则：说明为什么去掉 `#` 前缀、非字母数字转横线
9. **`src/channels/message/ingress-queue.ts`** 的 `ChannelIngressQueueClaim` 类型：说明 claim 机制如何防止多个 worker 处理同一条入站消息
10. **`src/channels/plugins/types.core.ts:79-80`** -- `ChannelMessageToolDiscovery.mediaSourceParams`：说明为什么按 action 限定 media source params 的范围
