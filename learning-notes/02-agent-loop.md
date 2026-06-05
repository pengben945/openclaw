# OpenClaw Agent 主循环深度分析

## 1. 顶层职责描述

`src/agents/agent-command.ts` 是整个 OpenClaw Agent 系统的**中枢调度器**。它负责：

- 接收用户/系统的消息请求（CLI 或网络 ingress）
- 解析配置、会话、模型选择、认证信息
- 编排 Agent 执行生命周期（准备 -> 执行 -> 持久化 -> 投递）
- 处理模型故障转移（fallback）和实时模型切换
- 管理会话持久化和重启恢复

这是 OpenClaw 中最核心、最复杂的编排文件，约 2269 行，横跨配置解析、会话管理、模型路由、技能发现、ACP 协议、投递等多个子系统。

---

## 2. 核心数据结构详解

### 2.1 `AgentCommandOpts` ( `src/agents/command/types.ts:42` )

这是 Agent 执行请求的**统一入口参数**，多达 50+ 个可选字段，按功能分组：

| 分组 | 关键字段 | 职责 |
|------|----------|------|
| 消息 | `message`, `transcriptMessage`, `images` | 用户输入内容 |
| 路由 | `to`, `sessionId`, `sessionKey`, `agentId` | 目标会话/Agent |
| 模型 | `provider`, `model`, `thinking`, `thinkingOnce` | 运行时模型选择 |
| 投递 | `deliver`, `replyTo`, `replyChannel`, `channel` | 结果投递配置 |
| 执行 | `runContext`, `abortSignal`, `timeout`, `lane` | 运行时控制 |
| 安全 | `senderIsOwner`, `allowModelOverride` | 鉴权 |
| 元数据 | `spawnedBy`, `groupId`, `workspaceDir`, `cwd` | 子Agent 继承 |

关键设计决策：**所有字段都是可选的**，系统通过 session 持久化、配置继承、自动发现来填充缺失值。这种设计使得无论是 CLI 一行命令（只传 `--message`），还是复杂的子Agent 调用，都能走同一个执行路径。

### 2.2 `SessionResolution` ( `src/agents/command/session.ts:35` )

```
SessionResolution {
  sessionId: string;           // 当前运行会话 UUID
  sessionKey?: string;         // 持久化存储的 Key
  sessionEntry?: SessionEntry; // 会话的持久化状态（可能为 undefined 表示新会话）
  sessionStore?: Record<string, SessionEntry>; // 整个会话存储文件的快照
  storePath: string;           // 会话文件路径
  isNewSession: boolean;       // 是否为新创建的会话
  persistedThinking?: ThinkLevel;   // 上次持久化的思考级别
  persistedVerbose?: VerboseLevel;  // 上次持久化的详情级别
}
```

### 2.3 `SessionEntry` ( `src/config/sessions/types.ts` )

会话持久化的核心单元，包含：

- **路由信息**: `channel`, `chatType`, `lastChannel`, `groupId`
- **模型状态**: `providerOverride`, `modelOverride`, `modelOverrideSource`
- **认证状态**: `authProfileOverride`, `authProfileOverrideSource`
- **生命周期**: `sessionId`, `updatedAt`, `sessionStartedAt`, `lastInteractionAt`
- **投递状态**: `pendingFinalDelivery*`, `restartRecoveryDeliveryContext`
- **技能快照**: `skillsSnapshot`
- **其他**: `thinkingLevel`, `verboseLevel`, `fastMode`, `spawnedBy`, `subject`

### 2.4 `EmbeddedAgentRunResult`

LLM 调用的结果，包含：

- `payloads: ReplyPayload[]` — 生成的回复内容
- `meta.agentMeta` — 模型信息（provider, model, usage）
- `meta.durationMs` — 耗时
- `meta.executionTrace` — 执行跟踪（决定转录持久化策略）

---

## 3. 主循环流程

### 3.1 入口点

两种入口：

```
agentCommand()                          // CLI/本地调用, 默认 senderIsOwner=true, allowModelOverride=true
  └─ withLocalGatewayRequestScope()
       └─ agentCommandInternal()

agentCommandFromIngress()               // 网络入口, 强制显式 allowModelOverride
  └─ agentCommandInternal()
```

设计意图：**本地 CLI 调用者是受信任的**，默认拥有模型覆盖权限；网络入口需要显式授权，防止未授权的远程调用篡改模型选择。

### 3.2 完整流程图（文本）

```
agentCommand / agentCommandFromIngress
│
├── [Phase 0] 依赖与上下文初始化
│   ├── resolveAgentCommandDeps()     // CLI 依赖懒加载
│   ├── prepareAgentCommandExecution() // 核心准备
│   │   ├── resolveAgentRuntimeConfig()  // 配置文件加载 + secret 解析
│   │   ├── 规范化 AgentId、SessionKey、Lane
│   │   ├── resolveSession()             // 会话解析
│   │   ├── 解析 thinking/verbose/timeout 覆盖
│   │   ├── 解析工作空间、技能元数据
│   │   ├── 构建模型清单上下文 (modelManifestContext)
│   │   └── ACP 会话初始化
│   │
├── [Phase 1]  ACP 路径（快速路径）
│   ├── 如果 acpResolution.kind === "ready":
│   │   ├── ACP 策略检查 (turn/dispatch/agent policy)
│   │   ├── acpManager.runTurn()  // 委托给 ACP 运行时
│   │   ├── 事件处理 (prompt_submitted, text_delta, done)
│   │   ├── 转录持久化
│   │   ├── buildAcpResult() + deliverAgentCommandResult()
│   │   └── return (跳过 CLI 路径)
│   │
├── [Phase 2] Skills 快照解析
│   ├── resolveEffectiveAgentSkillFilter()
│   ├── resolveReusableWorkspaceSkillSnapshot()
│   ├── 持久化 skillsSnapshot 到 session
│   │
├── [Phase 3] 初始会话持久化
│   ├── 持久化 thinkOverride / verboseOverride
│   └── 设置 lastInteractionAt
│
├── [Phase 4] 模型选择（核心路径）
│   ├── resolveDefaultModelForAgent()  // 从配置获取默认模型
│   ├── 覆盖解析优先级:
│   │   1. 显式 --provider/--model 覆盖
│   │   2. 会话持久化覆盖 (sessionEntry.providerOverride/modelOverride)
│   │   3. 频道模型覆盖 (channelModelOverride)
│   │   4. Auto-fallback primary probe
│   │   5. 可见性策略 (visibilityPolicy) 过滤
│   ├── 认证 Profile 验证
│   └── Harness 插件初始化
│
├── [Phase 5] 主执行循环 (for(;;))
│   ├── runWithModelFallback()     // 模型 fallback 框架
│   │   ├── resolveModelCandidateChain()  // 生成候选链
│   │   ├── for candidate in candidates:
│   │   │   ├── HarnessAuth precheck
│   │   │   ├── Cooldown/Auth 决策
│   │   │   ├── runAgentAttempt()        // 实际执行
│   │   │   │   ├── 选择运行时: CLI / Embedded Agent
│   │   │   │   ├── runCliAgent() 或 runEmbeddedAgent()  ← LLM 调用在这里
│   │   │   │   └── 返回 EmbeddedAgentRunResult
│   │   │   └── 分类结果 -> 成功/失败 -> fallback 或 break
│   │   └── 所有候选失败 -> 抛出 FallbackSummaryError
│   │
│   ├── LiveSessionModelSwitchError 处理:
│   │   ├── 更新 provider/model
│   │   ├── 清理状态 (autoFallbackPrimaryProbeInterruptedByLiveSwitch)
│   │   ├── retry (最多 5 次)
│   │   └── 重新进入 for(;;)
│   │
│   └── 成功后:
│       ├── emitLifecycleFinishing()
│       └── break
│
├── [Phase 6] 会话持久化更新
│   ├── updateSessionStoreAfterAgentRun()
│   │   ├── 更新 token 用量、模型信息
│   │   └── 更新 lastInteractionAt
│   │
├── [Phase 7] 转录持久化
│   ├── persistCliTurnTranscript()   // 写入 session 转录文件
│   └── runCliTurnCompactionLifecycle()  // 上下文压缩
│
├── [Phase 8] Pending Final Delivery 持久化
│   ├── 在投递前将 payload 持久化到 session store
│   └── 确保进程重启后不会丢失投递内容
│
├── [Phase 9] 结果投递
│   ├── deliverAgentCommandResult()
│   │   ├── 投递路由解析
│   │   ├── 回复 Payload 规范化
│   │   ├── sendDurableMessageBatch()   // 通过 channel 发送
│   │   └── 清除 pending final delivery
│   │
├── [Phase 10] 最终清理 (try/finally)
│   ├── 清除 restart recovery delivery context
│   └── clearAgentRunContext()
```

---

## 4. 消息队列与调度

### 4.1 Steering Queue (`src/agents/agent-steering-queue.ts`)

这是一个**子Agent 完成事件合并队列**，用于在主 Agent 的下一个 turn 中批量通知其子任务的完成状态。

关键机制：

- **租约机制** (Lease): `leasePendingAgentSteeringItemsFromSubagentRuns()` 将 pending 的条目标记为 `in_progress`，锁定给某个生命周期。
- **去重排序**: `listPendingAgentSteeringItemsFromSubagentRuns()` 按结束时间、创建时间排序。
- **大小限制**: `MAX_MERGED_STEERING_CHARS = 24000`，超出的条目不纳入当前轮次。
- **状态机**: `SubagentCompletionDeliveryState.status` 从 `pending` -> `in_progress` -> `delivered`（或 `failed`/`discarded`）。
- **陈旧租约恢复**: 超过 5 分钟的 `in_progress` 条目视为陈旧，可被重新租用。

设计亮点：这不是一个传统的消息队列，而是一个**内存中状态驱动的合并批量通知系统**。子Agent 的输出不通过独立通道回传，而是等待主 Agent 下一轮 LLM 调用时作为 steering prompt 注入。

### 4.2 与主循环的关系

Steering queue 在主 Agent 的下一次 `runAgentAttempt()` 调用前被消费——通过 `runContext` 和 skills snapshot 机制，子Agent 的完成事件被注入到上下文提示中，让主 Agent 感知到子任务的进度。

---

## 5. 状态生命周期

### 5.1 Agent 运行生命周期状态

定义在 `src/agents/command/attempt-callbacks.ts`：

```
AgentAttemptLifecycleState {
  currentTurnUserMessagePersisted: boolean;  // 当前轮用户消息是否已持久化
  lifecycleFinishing: boolean;               // 是否正在完成
  lifecycleEnded: boolean;                   // 是否已结束
}
```

通过 `createAgentAttemptLifecycleCallbacks()` 创建的**状态跟踪回调**被传递给 `runAgentAttempt()`，用于：

- 在 fallback 重试时避免重复持久化用户消息（`suppressPromptPersistenceOnRetry`）
- 防止 `emitLifecycleFinishing()` 和 `emitLifecycleEnd()` 重复触发

### 5.2 Agent Event 生命周期流

```
phase: "start"     -> agentCommandInternal 开始
  (fallback loop with multiple attempts)
phase: "finishing" -> 一次成功后才发射
phase: "end"       -> 所有后处理完成后
phase: "error"     -> 任意阶段失败
```

每个事件包含 `runId`, `startedAt`, `endedAt`, `stopReason`, `aborted` 等字段。

### 5.3 会话级别生命周期

```
SessionEntry {
  sessionStartedAt: timestamp     // 首次创建
  updatedAt: timestamp            // 每次更新
  lastInteractionAt: timestamp    // 最后交互
}
```

会话重置策略 (`resolveSessionResetPolicy`) 决定何时开始新会话（基于空闲超时、channel 切换等）。

### 5.4 模型 Fallback 生命周期

```
runWithModelFallback -> for each candidate:
  1. resolveModelCandidateChain()    生成有序候选列表
  2. HarnessAuth precheck            检查运行时兼容性
  3. Cooldown decision               是否跳过冷却中的 provider
  4. runFallbackAttempt()            尝试调用
  5. Result classification           分类结果（是否可恢复）
```

失败汇总以 `FallbackSummaryError` 形式抛出，包含所有尝试的详细信息。

---

## 6. 关键设计决策

### 6.1 为什么使用 `for(;;)` 外层循环 + `runWithModelFallback` 内层循环？

这是双层 fallback 架构：

- **外层** (`for(;;)` in `agentCommandInternal`): 处理 `LiveSessionModelSwitchError`——这是来自嵌入 Agent 运行的**模型切换指令**，需要在当前运行时切换 provider/model 后重试。
- **内层** (`runWithModelFallback`): 处理模型候选的**故障转移**——在相同的会话中尝试配置的 fallback 模型列表。

这种分层设计的原因是：LiveSessionModelSwitchError 可能发生在任何 fallback 尝试中，且切换后需要重新评估整个 fallback 链。

### 6.2 为什么 ACP 和 CLI 路径分离？

`agentCommandInternal` 中有一个显式的 `if (acpResolution.kind === "ready")` 分支：

- **ACP 路径**：使用 OpenAI Responses API 原生的会话管理，OpenClaw 只做事件监听和结果转发。这是**快速路径**。
- **CLI/Embedded 路径**：使用 OpenClaw 自己的 Agent 运行时（`runEmbeddedAgent` 或 `runCliAgent`），处理模型 fallback、技能注入、转录管理、上下文压缩等。这是**完整路径**。

两条路径共享相同的 `prepareAgentCommandExecution()` 准备阶段，但在执行和持久化阶段完全分离。

### 6.3 为什么模型选择优先级如此复杂？

模型选择的优先级（`agent-command.ts:1190-1425`）是一个多级覆盖链：

```
1. 显式 CLI 覆盖 (--provider/--model)
2. 会话持久化覆盖 (session store)
3. 频道模型覆盖 (channels.modelByChannel)
4. Auto-fallback primary probe
5. 可见性策略过滤 (agents.defaults.models)
```

每个级别都有自己的验证和清理逻辑（如 `repairProviderWrappedModelOverride`、`clearAutoFallbackPrimaryProbeSelection`）。这是为了支持：

- **跨 run 持久化**：用户可以通过 channel 对话选择一个模型，该选择持久化到 session store
- **动态 fallback**：当首选模型不可用时，自动降级到 fallback，并在会话中记录
- **安全围栏**：`visibilityPolicy` 确保子Agent 不能在父 Agent 配置的允许范围之外选择模型

### 6.4 为什么 Pending Final Delivery 需要两阶段持久化？

`agent-command.ts:2074-2178` 中的两阶段设计：

- **Phase 1**（投递前）：将 payload 持久化到 session store 的 `pendingFinalDelivery*` 字段
- **Phase 2**（投递后）：清除这些字段

这保证了：如果进程在 `sendDurableMessageBatch()` 执行过程中崩溃重启，重启后可以通过 `restartRecoveryDeliveryContext` 恢复投递，不会丢失用户应该收到的消息。

### 6.5 为什么使用 lazy import 加载运行时？

`agent-command.ts:241-329` 中的 12 个 lazy import loader：

```
attemptExecutionRuntimeLoader, acpManagerRuntimeLoader,
acpPolicyRuntimeLoader, deliveryRuntimeLoader, ...
```

这些运行时模块合计可能有数千行代码和复杂的依赖。懒加载确保：

- 快速 CLI 命令（如 `--help`）不触发这些加载
- ACP 路径只需要 `acp*RuntimeLoader`，不需要 `deliveryRuntimeLoader`
- 模块可以独立演进，减少循环依赖

---

## 7. 与其他模块的交互关系

```
                       ┌─────────────────────┐
                       │    CLI / Ingress     │
                       │  (agentCommand.ts)   │
                       └──────────┬──────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
   │  Config 系统      │  │  Session 管理     │  │  ACP 控制平面    │
   │ (config/io.ts)   │  │ (sessions/)      │  │ (acp/)          │
   │ resolveRuntime   │  │ resolveSession() │  │ acpManager      │
   │ Config()         │  │ loadSessionStore │  │ .runTurn()      │
   └─────────────────┘  └─────────────────┘  └─────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────┐
│  Model Fallback   │    │  Agent 执行      │    │  技能系统             │
│ (model-fallback   │◄───│  (attempt-      │    │  (skills/)           │
│ .ts)              │    │  execution.ts)  │    │  resolveReusable    │
│ runWithModel      │    │  run           │    │  WorkspaceSkill     │
│ Fallback()        │    │  AgentAttempt() │    │  Snapshot()          │
└─────────────────┘    └─────────────────┘    └─────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Embedded Agent   │
                    │ Runner          │
                    │ (embedded-      │
                    │ agent.ts)       │
                    │ → runCliAgent  │
                    │ → runEmbedded   │
                    │   Agent        │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 投递系统          │
                    │ (delivery.ts)    │
                    │ sendDurable      │
                    │ MessageBatch()   │
                    └─────────────────┘
```

---

## 8. 建议添加注释的位置列表

以下是在源码中添加注释可以极大降低理解成本的**关键位置**。建议使用 1-3 行的中文注释，说明 "为什么这么写" 和 "边界条件是什么"。

### `src/agents/agent-command.ts`

| 行号 | 建议注释内容 |
|------|-------------|
| 586-587 | 说明 `isRawModelRun` 的定义——modelRun 和 promptMode=none 都表示不经过完整 Agent 管道，直接调用模型 |
| 651-657 | 说明 subagent lane 的默认 timeout 为 0（无超时），因为子Agent 执行时间不可预测 |
| 702 | 说明工作空间继承策略——子Agent 可以继承父 Agent 的工作空间 |
| 750-751 | 说明 `ensureAgentWorkspace` 在准备阶段执行的原因：需要确保文件系统存在后再进行技能发现 |
| 860-861 | 说明 ACP stale 会话的处理——过期 ACP 会话在准备阶段报错，避免进入执行阶段浪费资源 |
| 906-1085 | 在 ACP 分支开始处注释：ACP 路径和 CLI 路径在此分叉，前者委托给 ACP 控制平面，后者走完整 OpenClaw 运行时 |
| 1156-1157 | 说明初始会话持久化的目的：在技能发现之前先持久化 thinking/verbose 覆盖，避免它们被后续步骤覆盖 |
| 1292-1316 | 说明 Legacy auto-fallback 清理逻辑——当会话中的 modelOverrideSource 为 "auto" 但缺少 fallback origin 信息时，需要用当前默认模型替换 |
| 1379-1393 | 说明 Auto-fallback primary probe 的概念：定期尝试用主模型恢复，看其是否从故障中恢复 |
| 1595-1655 | 说明生命周期状态回调的设计意图：防止 fallback retry 时重复发射 lifecycle 事件 |
| 1867-1951 | 说明 LiveSessionModelSwitchError 处理——这是一种"控制流错误"，用于下层运行时通知上层切换模型 |
| 2076-2077 | 说明两阶段 Pending Final Delivery 的设计意图：投递前持久化保证崩溃恢复 |
| 2218-2243 | 说明 `agentCommand` 和 `agentCommandFromIngress` 的安全模型差异：本地 CLI 受信任，网络入口需显式授权 |

### `src/agents/agent-scope.ts`

| 行号 | 建议注释内容 |
|------|-------------|
| 49-51 | 说明 `AUTO_FALLBACK_PRIMARY_PROBE_INTERVAL_MS = 5 分钟` 的 rationale——最小探测间隔防止在故障期频繁重试 |
| 140-141 | 说明 `resolveAutoFallbackPrimaryProbe` 的目的：创建一个"探测候选"，尝试用主模型运行，如果成功则切回主模型 |
| 160-168 | 说明 probe 返回 undefined 的条件：origin 和 primary 相同时不需要探测，因为已经在用主模型 |
| 178-190 | 说明探测频率限制逻辑 |

### `src/agents/agent-steering-queue.ts`

| 行号 | 建议注释内容 |
|------|-------------|
| 8-11 | 说明字符上限的 rationale——避免 steering prompt 占用过多上下文窗口 |
| 28-33 | 说明陈旧租约检测（5分钟超时）——防止子Agent 完成但主 Agent 不消费 |
| 140-157 | 说明 `selectPromptBoundedItems` 的贪心算法——优先加入更多条目，直到达到字符上限 |
| 249-258 | 说明 `prependAgentSteeringPrompt` 的使用方式——steering 信息放在用户消息之前 |

### `src/agents/agent-settings.ts`

| 行号 | 建议注释内容 |
|------|-------------|
| 85-98 | 说明小上下文模型的 reserve tokens 上限计算——防止默认的 20000 token 下限超过模型的整个上下文窗口 |
| 189-199 | 说明 `shouldDisableAgentAutoCompaction` 的条件——当 context engine 拥有压缩权时，OpenClaw 运行时不应干预 |

### `src/agents/model-fallback.ts`

| 行号 | 建议注释内容 |
|------|-------------|
| 181-193 | 说明 `isFallbackAbortError` 的严格检测——只承认显式 `AbortError` 名为用户中断 |
| 1293-1297 | 说明 Skip-known-bad 缓存的 rationale——避免在同一个会话中反复尝试已知会鉴权失败的候选模型 |
| 1550-1554 | 说明为何局部运行时协调错误（如会话写锁超时）不应触发 fallback |
| 1582-1593 | 说明 LiveSessionModelSwitchError 的 fallback 候选跳转优化 |

---

## 9. 生产环境与面试视角

### 9.1 潜在生产风险

| 风险 | 位置 | 说明 |
|------|------|------|
| **大对象内存泄漏** | Steering queue 的 Map 存储 | `autoFallbackPrimaryProbeState` 和 `fallbackCandidateCache` 都是静态 Map，虽然有大小限制和 TTL，但在极端高并发下可能成为内存瓶颈 |
| **文件锁竞争** | `persistTextTurnTranscript` 的 `acquireSessionWriteLock` | 同一 session 的并发写入可能导致写锁等待，在高频消息场景下可能成为性能瓶颈 |
| **进程重启丢失状态** | Steering queue 纯内存 | `AgentSteeringQueueItem` 全程在内存中，进程崩溃后所有 steering 事件丢失 |
| **无限 fallback 循环** | `agent-command.ts:1867-1951` | `LiveSessionModelSwitchError` 重试有 5 次上限，但如果 visibilityPolicy 持续拒绝切换，会导致每次都抛错。代码中有 `visibilityPolicy.allowsKey()` 检查，但如果在准备阶段（`prepareAgentHarnessRuntime`）抛异常，可能绕过检查 |
| **配置爆炸** | `AgentCommandOpts` 50+ 字段 | 大量可选字段的组合状态难以测试，边缘情况可能未覆盖 |
| **超大转录文件** | Session 转录文件持续增长 | 虽然 `cliCompactionRuntime` 会做压缩，但在极端长会话中，转录文件的读写会成为性能瓶颈 |

### 9.2 面试模拟题

#### Q1: `agentCommandInternal` 函数中有一个 `for(;;)` 无限循环。请解释这个循环的作用，以及它是如何终止的。

**参考答案**：

这个外层循环处理 `LiveSessionModelSwitchError`——这是嵌入 Agent 运行时抛出的一种"控制流异常"，用于通知上层调用者需要切换 provider/model。捕获到该异常后：

1. 验证新模型是否在可见性策略（`visibilityPolicy.allowsKey()`）的允许范围内
2. 更新 `provider`, `model`, `storedModelOverride` 等变量
3. 重置 `attemptLifecycleState.lifecycleEnded = false`
4. `continue` 重新进入内层 fallback 循环

正常终止有两种方式：
- 内层 `runWithModelFallback` 成功 -> `emitLifecycleFinishing -> break`
- 所有 model candidates 失败 -> `FallbackSummaryError` 抛出
- `LiveSessionModelSwitchError` 超过 5 次 -> 抛出最终错误

设计模式类比：这类似于 Go 语言中通过 `panic`/`recover` 实现栈展开控制流的模式，但这里用 `try/catch` + `for(;;)` 实现。下层运行时（embedded agent）无法直接修改上层执行引擎的状态，只能通过异常将控制流上浮。

#### Q2: 什么是 Auto-Fallback Primary Probe？它是如何工作的？

**参考答案**：

当一个 Agent 因为模型故障（如 rate limit、认证错误）从 primary model fallback 到 backup model 后，系统会定期尝试用 primary model 重新执行，检查其是否恢复。这就是 Auto-Fallback Primary Probe。

工作流程：
1. 在 `resolveAutoFallbackPrimaryProbe()` 中检查 session entry 是否标记了 `modelOverrideSource === "auto"` 和 fallback origin 信息
2. 如果有，构造一个 `AutoFallbackPrimaryProbe` 结构体，包含原模型和当前降级模型
3. 在 `runAgentAttempt()` 前，如果当前 candidate 是 primary probe 候选，调用 `markAutoFallbackPrimaryProbe()` 记录探测时间戳
4. 探测频率限制为 5 分钟（`AUTO_FALLBACK_PRIMARY_PROBE_INTERVAL_MS`），防止在故障期间频繁重试
5. 如果 primary 探测成功（模型恢复），自动清除 session 中的 fallback 覆盖，让后续 run 回到 primary
6. 如果 primary 探测失败，结果被 `classifyResult` 分类为故障，fallback loop 继续使用当前候选

这样设计的好处是：用户不需要手动切换回 primary model，系统自动在探测成功时恢复。

#### Q3: `agentCommandInternal` 中有两条完全不同的执行路径（ACP 和 CLI/Embedded）。为什么要这样设计？何时选择哪条路径？

**参考答案**：

选择路径的依据是 `acpResolution.kind` 的状态：
- `"ready"` -> ACP 路径
- `"stale"` -> 报错，不执行
- 其他（null 或未初始化）-> CLI/Embedded 路径

ACP（Agent Control Plane）路径适用于使用 OpenAI Responses API 的会话。ACP 拥有自己的会话状态管理，OpenClaw 只需：1) 注入 prompt, 2) 转发事件, 3) 持久化转录, 4) 投递结果。这是一个**薄层代理**。

CLI/Embedded 路径是 OpenClaw 自己的 Agent 运行时，需要处理：模型选择/覆盖/fallback、技能发现/注入、Auth profile 管理、上下文压缩、Steering queue 等。这是**完整 Agent 运行时**。

这种分离的设计意义包括：
- ACP 路径减少了对 ACP 运行时状态的干扰
- 两条路径共享 `prepareAgentCommandExecution()` 准备阶段，确保配置解析一致
- ACP 和 CLI/Embedded 的转录持久化策略不同（ACP 使用 `persistAcpTurnTranscript`，CLI 使用 `persistCliTurnTranscript`）
- 新的 Agent 运行时（如 Codex harness）可以各自实现自己的执行路径，但复用准备和投递逻辑

#### Q4: 解释 `agent-command.ts` 中 model selection 的优先级链。为什么需要这么复杂的覆盖机制？

**参考答案**：

模型选择的优先级链从高到低：

```
1. 显式 CLI 覆盖 (opts.provider / opts.model)
2. 会话 store 持久化的覆盖 (sessionEntry.providerOverride / modelOverride)
3. 频道级别的模型覆盖 (channels.modelByChannel)
4. Auto-fallback primary probe (用于恢复 primary model)
5. Agent 配置的 default model
6. 全局 DEFAULT_PROVIDER / DEFAULT_MODEL
```

每个层级都经过 `normalizeAgentCommandModelRef` 标准化和 `visibilityPolicy.allowsKey()` 安全过滤。

设计理由：
- **层级 1** 给 CLI 用户最高控制权——"这次运行就用这个模型"
- **层级 2** 支持跨 run 持久化——"上次选了这个模型，下次还用"
- **层级 3** 支持 channel 粒度配置——"这个 Discord 频道用 GPT-4，那个 Slack 频道用 Claude"
- **层级 4** 是自主恢复机制——"上次 fallback 了，现在试试能不能切回去"
- **安全围栏**（visibilityPolicy）确保子Agent 不能选择超出父 Agent 配置允许范围的模型

这种复杂度的来源是：模型选择不再是一个简单的配置项，而是**安全策略 + 故障恢复 + 用户偏好 + 频道路由**的交叉点。

#### Q5: 请描述 `pendingFinalDelivery` 两阶段持久化机制的完整流程和设计意图。

**参考答案**：

**Phase 1**（`agent-command.ts:2078-2113`，在投递前）：
1. 检查 `opts.deliver === true` 且 session 未标记为 subagent
2. 将 payloads 合并为一个文本块，执行 `sanitizePendingFinalDeliveryText`（清理敏感信息）
3. 将合并后的文本写入 session store 的 `pendingFinalDeliveryText` 字段，同时记录 `pendingFinalDeliveryContext`（路由信息）
4. 使用 `shouldPersistCurrentRunSessionCleanup` 防止在并发写入时覆盖其他 session 的状态

**设计意图**：如果进程在 Phase 1 和投递之间崩溃，重启后可以从 session store 读取 `pendingFinalDeliveryContext` + `restartRecoveryDeliveryContext`，恢复投递。这是**崩溃安全（crash-safe）**设计。

**Phase 2**（`agent-command.ts:2153-2177`，在投递后）：
1. 调用 `deliverAgentCommandResult()` 执行实际投递
2. 投递成功后，调用 `clearPendingFinalDeliveryFields()` 清除所有 `pendingFinalDelivery*` 字段
3. 使用 `shouldPersistCurrentRunSessionCleanup` 确保只清理当前 session 的 pending 状态

**边界情况**：
- 如果 payload 为空（`noPendingTextForThisRun`），但 session 中已经有 `pendingFinalDelivery=true`，也需要清理
- 对于 subagent session key，不执行 pending final delivery（子Agent 的结果由父 Agent 的 steering queue 汇聚）
- 如果 deliverySucceeded 为 `"partial"` 或 `false`，pending 状态保留

---

## 10. 学习要点

1. **控制流错误模式**：`LiveSessionModelSwitchError` 展示了如何用异常实现"下层通知上层切换状态"的模式，这是一种隐式的控制流反转。

2. **两阶段持久化**：pending delivery 的两阶段设计是分布式系统中常见的"先持久化后操作"模式，保证崩溃恢复。

3. **模块化懒加载**：通过 `createLazyImportLoader` 将 12 个运行时模块懒加载，这是大型 Node.js 应用优化启动性能的标准模式。

4. **防御性配置解析**：整个配置系统大量使用 `normalize*` 函数（`normalizeProviderId`, `normalizeAgentId`, `normalizeModelRef`），确保不同输入格式都能被一致处理。

5. **分层 fallback**：外层 fallback（模型切换）+ 内层 fallback（候选链）+ 会话级 skip cache（避免重复失败），三层防御机制覆盖了模型失败的各个维度。

6. **类型体操**：大量使用 `satisfies`、`ReturnType`、`Pick` 等类型操作，确保编译时类型安全的同时保持运行时灵活性。例如 `type AttemptExecutionRuntime = typeof import("./command/attempt-execution.runtime.js")` 避免了运行时模块的静态依赖。
