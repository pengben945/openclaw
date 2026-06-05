# OpenClaw Config System 深度分析

## 1. 配置系统架构总览

OpenClaw 的配置系统是一个多层、高鲁棒性的配置框架，支持 JSON5 格式、`$include` 模块化、环境变量替换、Zod Schema 静态验证、插件感知验证、运行时热重载以及异常配置自愈。

### 核心文件与职责

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/config/io.ts` | 2857 | 配置 I/O 中枢：读取、解析、写入、健康状态跟踪 |
| `src/config/validation.ts` | 1927 | 验证引擎：Zod Schema + 插件 Schema + 跨字段规则 |
| `src/config/zod-schema.ts` | 1332 | 主 Zod Schema 定义（OpenClawSchema） |
| `src/config/zod-schema.core.ts` | 1062 | 核心类型：SecretRef、Secrets Provider、Model Config |
| `src/config/defaults.ts` | 551 | 运行时默认值注入 |
| `src/config/includes.ts` | 443 | `$include` 指令解析（带安全沙箱） |
| `src/config/paths.ts` | 347 | 配置文件路径解析（多候选、向后兼容） |
| `src/config/runtime-snapshot.ts` | 342 | 运行时配置快照管理 |
| `src/config/schema.ts` | 846 | JSON Schema 生成（Gateway API 用） |
| `src/config/env-substitution.ts` | 203 | `${VAR}` 环境变量替换 |
| `src/config/merge-patch.ts` | 97 | Merge Patch 实现（RFC 7396 风格） |
| `src/config/materialize.ts` | 87 | 默认值编排器 |
| `src/config/config-env-vars.ts` | 92 | 内联 env.vars 处理 |
| `src/gateway/config-reload.ts` | 420 | 文件监听 + 热重载引擎 |
| `src/gateway/config-diff.ts` | 33 | 配置对象差异分析 |
| `src/gateway/config-reload-plan.ts` | ~150 | 差异路径到重载动作的映射 |
| `src/plugin-sdk/plugin-config-runtime.ts` | 45 | 插件配置查询 SDK |

---

## 2. 配置加载流程

```
loadConfig()
  │
  ├── 1. 路径解析 (paths.ts)
  │   ├── OPENCLAW_CONFIG_PATH 环境变量（最高优先级）
  │   ├── OPENCLAW_STATE_DIR/openclaw.json
  │   ├── ~/.openclaw/openclaw.json（默认）
  │   └── 遗留兼容: ~/.clawdbot/clawdbot.json
  │
  ├── 2. 加载 dotenv (infra/dotenv.ts) —— 仅首次，仅对真实 process.env
  │
  ├── 3. 文件不存在？→ 返回空配置对象 {}，后续默认值填充
  │
  ├── 4. 读取文件 + JSON5 解析 (io.ts)
  │   ├── 先尝试 JSON.parse —— 快速路径（自写文件）
  │   └── 失败后回退 json5.parse —— 兼容用户手写
  │       └── 还会尝试修复 JSON 前缀垃圾（shebang 等）
  │
  ├── 5. $include 解析 (includes.ts)
  │   ├── 递归处理，最大深度 10
  │   ├── 安全检查：CWE-22 路径穿越防护
  │   ├── symlink 双重校验（词法路径 + realpath）
  │   └── 可选的 OPENCLAW_INCLUDE_ROOTS 白名单
  │
  ├── 6. 环境变量替换 (env-substitution.ts)
  │   ├── 替换 ${VAR_NAME} 模式（仅大写字母/数字/下划线）
  │   ├── escaped: $${VAR} → 保留字面量
  │   ├── 缺失变量默认 non-fatal（warn 而非 throw）
  │   └── 替换前先 apply config.env.vars → process.env
  │
  ├── 7. 插件安装记录迁移 (io.ts)
  │   ├── 将遗留的 plugins.installs 从配置中剥离
  │   └── 写入到持久化插件索引（state/openclaw.sqlite）
  │
  ├── 8. Zod Schema 验证 (validation.ts)
  │   ├── OpenClawSchema.safeParse(raw)
  │   ├── 目标：验证形状，不做默认值注入
  │   ├── 政策规则检查：
  │   │   ├── SecretRef 在不支持的字段上的使用
  │   │   ├── 重复的 agent 目录
  │   │   ├── identity.avatar 路径安全
  │   │   ├── gateway.tailscale 安全约束（bind + auth）
  │   │   └── 心跳目标有效性
  │   └── 插件级验证：
  │       ├── bundled channel AJV schema 验证
  │       ├── 插件 manifest schema 验证
  │       ├── 缺失/阻止的插件诊断
  │       └── 模型引用可用性检查
  │
  ├── 9. 运行时默认值注入 (materialize.ts)
  │   ├── applyMessageDefaults() - ackReactionScope
  │   ├── applyLoggingDefaults() - redactSensitive
  │   ├── applySessionDefaults() - 忽略 session.mainKey
  │   ├── applyAgentDefaults() - maxConcurrent, subagent
  │   ├── applyCronDefaults() - maxConcurrentRuns
  │   ├── applyContextPruningDefaults() - Anthropic 默认修剪
  │   ├── applyCompactionDefaults() - mode: "safeguard"
  │   ├── applyModelDefaults() - 模型 ID 规范化、成本、maxTokens 安全值
  │   ├── applyTalkConfigNormalization() - 语音配置规范化
  │   ├── normalizeConfigPaths() - 规范化目录路径
  │   └── normalizeExecSafeBinProfilesInConfig() - 可执行文件安全检查
  │
  ├── 10. 最终运行时处理
  │   ├── 发现重复 agent 目录 → 抛异常
  │   ├── 应用 config.env.vars → process.env
  │   ├── Shell env 回退（可选的 $SHELL -l 执行）
  │   ├── Owner display secret 生成
  │   └── 运行时覆盖 (applyConfigOverrides)
  │
  └── 11. 快照锁定 (runtime-snapshot.ts)
      └── loadPinnedRuntimeConfig() 将首次成功加载的快照锁定为进程级单例
```

### 配置材质化模式

`materialize.ts` 定义了三种材质化模式，每种模式应用不同的默认值集合：

| 模式 | 场景 | compaction defaults | context pruning | logging defaults | paths normalize |
|------|------|:---:|:---:|:---:|:---:|
| `load` | 正常加载 | YES | YES | YES | YES |
| `missing` | 插件缺失时 | YES | YES | NO | NO |
| `snapshot` | 快照读/验证 | NO | NO | YES | YES |

---

## 3. Schema 定义与验证

### 3.1 Zod Schema 层次结构

OpenClawSchema 定义在 `zod-schema.ts` 中，是一个深度嵌套的 z.object 树：

```
zod-schema.ts:
  OpenClawSchema
  ├── $schema: z.string().optional()
  ├── meta: z.object({ lastTouchedVersion, lastTouchedAt }).strict().optional()
  ├── env: z.object({ shellEnv?, vars? }).catchall(z.string()).optional()
  ├── wizard: z.object(...).optional()
  ├── diagnostics: z.object({ enabled, flags, otel, ... }).optional()
  ├── logging: z.object({ level, file, consoleLevel, redactSensitive, ... }).optional()
  ├── cli: z.object({ banner }).optional()
  ├── update: z.object({ channel, auto, proxy }).optional()
  ├── auth: z.object({ profiles, order, cooldowns, ... }).optional()
  ├── accessGroups: z.record(...).optional()
  ├── gateway: GatewaySchema (remote, tailscale, bind, reload, auth, ...)
  ├── nodeHost: NodeHostSchema.optional()
  ├── proxy: ProxyConfigSchema.optional()
  ├── security.z.object({ audit.suppressions }).optional()
  ├── approvals: ApprovalsSchema.optional()
  ├── browser: z.object({ snapshot, ... }).optional()
  ├── canvasHost: LegacyCanvasHostSchema.optional()
  ├── web: z.object(...).optional()
  ├── channels: ChannelsSchema.optional()
  ├── models: ModelsConfigSchema (providers, pricing)
  ├── agents: AgentsSchema (defaults, list, bindings)
  ├── plugins: PluginsConfigSchema (entries, allow, deny, slots, installs)
  ├── tools: ToolsSchema (web, code, editor, mcp)
  ├── skills: z.record(SkillEntrySchema).optional()
  ├── talk: TalkSchema.optional()
  ├── messages: MessagesSchema.optional()
  ├── session: SessionSchema.optional()
  ├── hook: HookMappingSchema.optional()
  ├── secrets: SecretsConfigSchema.optional()
  ├── memory: MemorySchema.optional()
  ├── mcp: z.object(...).optional()
  ├── commitments: CommitmentsSchema.optional()
  ├── crestodian: CrestodianSchema.optional()
  ├── cron: z.object({ maxConcurrentRuns }).optional()
  └── bindings: BindingsSchema.optional()
```

所有顶级字段都是 `.strict()` + `.optional()`。Zod 的 `.strict()` 确保没有意外键被静默忽略。

### 3.2 SecretRef 类型

`zod-schema.core.ts` 定义了三种 SecretRef 来源：

- **env** (`{ source: "env", provider: string, id: string }`) — 引用进程环境变量
- **file** (`{ source: "file", provider: string, id: string }`) — 引用文件中的 JSON 值
- **exec** (`{ source: "exec", provider: string, id: string }`) — 引用命令执行结果

`SecretInputSchema` = `z.union([z.string(), SecretRefSchema])`，许多敏感字段
（如 apiKey、token）使用 `.register(sensitive)` 标记，在日志/诊断中自动脱敏。

### 3.3 SuperRefine 跨字段校验

Zod 的 `superRefine` 用于跨字段校验：

- `TalkSchema.superRefine` — 验证 `talk.provider` 存在于 `talk.providers`
- `TalkRealtimeSchema.superRefine` — 同样模式
- 其他关系型约束（如 gateway.auth/tailscale 的依赖关系）放在 `validation.ts` 中

### 3.4 插件 Schema 验证

`validation.ts` 中的 `validateConfigObjectWithPluginsBase()` 负责插件级验证：

1. **通道 Schema 验证**：每个 configured bundled channel 通过 AJV 验证其 JSON Schema
2. **插件 Manifest Schema 验证**：每个激活的插件通过 AJV 验证 `plugins.entries[pluginId].config`
3. **插件状态推断**：通过 `resolveEffectivePluginActivationState` 决定插件是否真正激活
4. **Memory slot 冲突检测**：通过 `resolveMemorySlotDecision` 确保 memory slot 唯一
5. **WebSearch provider 验证**：检查 `tools.web.search.provider` 是否匹配已知 provider
6. **Model ref 抑制检测**：通过 `planManifestModelCatalogSuppressions` 检查移除的模型
7. **心跳目标验证**：`agents.*.heartbeat.target` 必须是已知 channel 或 "last"/"none"

### 3.5 验证错误处理

`validation.ts` 中的 `mapZodIssueToConfigIssue()` 将 Zod 抽象错误转换为用户友好的消息：

- 枚举值验证失败时，附带允许值列表（`(allowed: "stable", "beta", "dev")`）
- 数值越界时附加界限信息（`(maximum: 65535)`）
- Bindings union 分支展开，提取最具体的错误分支
- `collectAllowedValuesFromBundledChannelSchemaPath` 从通道 schema 恢复可接受值

---

## 4. 配置合并与优先级

### 4.1 优先级层级（从低到高）

```
1. 硬编码默认值 (defaults.ts 中的 DEFAULT_* 常量)
2. applyMessageDefaults / applyAgentDefaults 等默认注入
3. 文件配置 (openclaw.json / $include 合并结果)
4. ${VAR} 环境变量替换 (env-substitution.ts)
5. config.env.vars 内联变量 (config-env-vars.ts)
6. 运行时默认值注入 (materialize.ts)
7. 运行时覆盖 (runtime-overrides.ts)
```

### 4.2 Merge Patch 算法

`merge-patch.ts` 实现了 RFC 7396 风格的 merge patch：

```typescript
// 核心递归逻辑
function applyMergePatch(base, patch):
  - 如果 patch 不是 plain object：直接替换 base
  - 对每个 patch key：
    - value === null：从结果中删除该 key
    - 如果启用 mergeObjectArraysById 且双方都是数组：按 id 合并
    - 如果 value 是 plain object：递归合并
    - 其他情况：直接替换
```

特别的数组合并策略：当数组元素有 `id` 字段时，按 `id` 合并而非全量替换。这对于插件 entries 的增量更新非常关键。

### 4.3 写回时的 Source vs Runtime 分离

这是系统最关键的设计决策之一：

- **sourceConfig**: 用户实际写入文件的配置值（不含运行时默认值）
- **runtimeConfig**: sourceConfig + 运行时默认值注入后的完整配置

写回时只写入 sourceConfig 形状，绝不写回运行时注入的默认值（如 compaction、context pruning 等）。
这是通过 `io.write-prepare.ts` 中的 `resolvePersistCandidateForWrite` 实现的。

---

## 5. 热重载机制

### 5.1 架构

```
                   ┌─────────────────────┐
                   │    chokidar 文件监听   │
                   │   (awaitWriteFinish)  │
                   └───────┬─────────────┘
                           │ add/change/unlink
                           ▼
              ┌──────────────────────────┐
              │  startGatewayConfigReloader│
              │  (gateway/config-reload.ts)│
              └──────────┬───────────────┘
                         │ debounce + deduplicate
                         ▼
              ┌──────────────────────────┐
              │  readSnapshot() + diff    │
              │  (diffConfigPaths)        │
              └──────────┬───────────────┘
                         │ changed paths
                         ▼
              ┌──────────────────────────┐
              │  buildGatewayReloadPlan() │
              │  路径 → 动作映射          │
              └──────────┬───────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    onRestart()    onHotReload()     静默忽略
    (重启网关)     (热更新)          (no-op)
```

### 5.2 路径到重载动作的映射

`config-reload-plan.ts` 定义了完整的映射规则：

| 路径前缀 | 动作 | 副作用 |
|----------|------|--------|
| `gateway.remote` | `none` | 无 |
| `gateway.channelHealthCheckMinutes` | `hot` | restart-health-monitor |
| `hooks.gmail` | `hot` | restart-gmail-watcher |
| `hooks` | `hot` | reload-hooks |
| `agents.defaults.heartbeat` | `hot` | restart-heartbeat |
| `models` | `hot` | restart-heartbeat |
| `models.pricing` | `restart` | 网关重启 |
| `plugins.load` | `restart` | 网关重启 |
| `plugins.installs` | `restart` | 网关重启 |
| `cron` | `hot` | restart-cron |
| `mcp` | `hot` | dispose-mcp-runtimes |
| `diagnostics.stuckSessionWarnMs` | `none` | 无 |
| `meta` | `none` | 无 |
| `skills.*` | 特殊处理 | 会话快照失效 |
| 插件 config | 由插件声明 | 动态解析 |

### 5.3 三种重载模式

`gateway.reload.mode` 控制重载行为：

- **auto**（默认）：根据 changed paths 智能选择 hot/restart
- **hot**：强制热更新（但 restart 类变更会警告并忽略）
- **restart**：任何变更都重启网关
- **off**：关闭自动重载，仅手动 `openclaw gateway restart`

### 5.4 进程内写优化

当运行时通过 API/CLI 写入配置时（而非编辑器外部编辑），直接通过 `ConfigWriteNotification` listener
通知重载引擎，跳过 chokidar 的文件监听延迟。此优化避免了文件写入 → 监听触发 → 读取 → 差异分析的
完整周期，减少了 `plugins.entries.*` 变更可能触发的误重启。

---

## 6. 环境变量处理

### 6.1 三种环境变量来源

| 来源 | 位置 | 优先级 | 生效时机 |
|------|------|--------|----------|
| 系统环境变量 | `process.env` | 最高 | 始终 |
| config.env.vars | `openclaw.json` 中 `env.vars` | 中 | 仅当系统未设置 |
| config.env.* 快捷写法 | `openclaw.json` 中 `env.KEY` | 中 | 仅当系统未设置 |
| Shell env fallback | `$SHELL -l -c 'env -0'` | 最低 | 仅当显式启用 |

### 6.2 环境变量替换流程

```
config.env.vars → applyConfigEnvVars() → 写入 process.env（如未设置）
     │
     ▼
$include 解析完成后的配置对象
     │
     ▼
resolveConfigEnvVars() — 递归遍历所有字符串值
     │
     ├── 匹配 ${VAR_NAME} → 替换为 env[VAR_NAME]
     ├── 匹配 $${VAR} → 保留字面量 ${VAR}
     └── 缺失变量 → onMissing callback（收集为 warnings 而非 crash）
```

### 6.3 写回时的变量保留

写回配置文件时，系统会从原始文件内容（env 替换前）提取 `${VAR}` 引用，并将
env 替换后的值与原始模式比较。如果值匹配（即来自 env），则恢复为 `${VAR}` 占位符，
避免写回时将 `${ANTHROPIC_API_KEY}` 明文暴露为 `sk-ant-...`。

实现细节：`restoreEnvVarRefs()` 和 `restoreEnvRefsFromMap()` 分别在
`io.ts:2290-2300` 和 `io.write-prepare.ts` 中，使用 `envSnapshotForRestore`
来解决 TOCTOU 问题（加载时的 env 快照 vs 写回时的实时 env）。

---

## 7. 插件配置接入

### 7.1 配置结构

插件配置位于 `openclaw.json` 的 `plugins.entries` 下：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        enabled: true,
        config: {
          // 插件自定义配置
        }
      }
    }
  }
}
```

### 7.2 插件配置获取

`plugin-config-runtime.ts` 提供 SDK：

```typescript
// 从已加载的 config 对象中提取
resolvePluginConfigObject(config, pluginId)

// 或使用延迟加载器（运行时实时获取）
resolveLivePluginConfigObject(runtimeConfigLoader, pluginId)
```

### 7.3 插件配置验证流程

1. 插件 manifest 声明 `configSchema`（JSON Schema 对象）
2. 配置加载过程中，`validation.ts` 的 `validateConfigObjectWithPluginsBase()` 自动
   对每个激活的插件调用 AJV 验证
3. 插件配置中的类型错误会被映射为标准化的 `ConfigValidationIssue`
4. 验证通过的配置值（含 AJV 注入的默认值）会回写回 `mutatedConfig`

### 7.4 插件 Schema 参与 JSON Schema 生成

`schema.ts` 中的 `applyPluginSchemas()` 将每个已安装插件的 `configSchema` 合并到
全局 JSON Schema 中，使得 Gateway API（如 `/api/config/schema/lookup`）能够提供
完整的配置 UI 提示。

### 7.5 缺失插件处理

- **bundled plugins**：系统内置，始终可用
- **official external plugins**：如果缺失，给出 `openclaw plugins install <spec>` 提示
- **unrecognized plugins**：warn 而非 error（跨版本升级容错），标记为 stale config
- **blocked plugins**：根据 registry diagnostics 给出具体的阻塞原因

---

## 8. 关键设计决策

### 8.1 验证与默认值分离

Zod Schema 仅验证形状（类型、必须字段、模式），不注入任何默认值。
运行时默认值由 `materializeRuntimeConfig()` 在验证后注入。

**原因**：确保写回时不会将运行时默认值写入用户配置文件（`#6070`）。

### 8.2 Fail Closed 策略

无效配置抛出异常而非回退到宽松默认值。这是有意的安全选择——避免用户误以为配置
生效而实际上在运行不安全的默认值。

```typescript
// io.ts:1803-1806
if (error?.code === "INVALID_CONFIG") {
  // Fail closed so invalid configs cannot silently fall back to permissive defaults.
  throw err;
}
```

### 8.3 配置健康监控

`io.ts` 中的 `ConfigHealthState` 系统跟踪配置文件的指纹信息：

- SHA256 hash
- 文件大小、mtime、ctime
- 设备 ID、inode 号（硬链接检测）
- 是否有 `meta` 段、gateway mode

异常检测规则：
- 文件大小突然缩小 50%+ → `size-drop` 警告并审计
- 之前有 meta 节但新文件没有 → `missing-meta` 警告
- gateway mode 被移除 → 高风险，写操作被阻塞（需 `allowDestructiveWrite`）

### 8.4 安全敏感的写保护

配置写入前会检查可疑模式并阻止：

```typescript
const blockingReasons = resolveConfigWriteBlockingReasons(suspiciousReasons, options);
if (blockingReasons.length > 0 && options.allowDestructiveWrite !== true) {
  // 拒绝写入，将 payload 保存到 .rejected 文件
  throw err;
}
```

### 8.5 原子写入 + 备份轮转

`replaceFileAtomic` 通过临时文件 + rename 实现原子写入，`maintainConfigBackups`
维护配置文件的历史备份链。

### 8.6 JSON 前置内容自动恢复

`io.ts:1070-1090` 中的 `findJsonRootSuffix` 可以检测配置文件中的非 JSON 前缀内容
（如 shebang、shell 配置指令），自动提取第一个 JSON 对象并恢复。

---

## 9. 生产环境与面试视角

### 9.1 生产预演：潜在问题

**问题 1: 配置读取锁竞争**
- 场景：极端并发下多个进程/goroutines 同时读取 `openclaw.json`
- 风险：chokidar 可能触发多次 reload，竞争条件导致运行时状态不一致
- 缓解：`debounceMs`（默认 300ms）+ `running`/`pending` 状态机控制

**问题 2: SecretRef 解析延迟**
- 场景：`exec` 类型的 SecretRef 需要外部命令执行，可能超时或挂起
- 风险：网关启动被阻塞或 `preflightRuntimeSnapshotWrite` 失败导致写操作回滚
- 缓解：有明确的 `timeoutMs` 配置，但 exec 路径需要 `isSafeExecutableValue` + 绝对路径

**问题 3: JSON Schema 缓存膨胀**
- 场景：大量插件（>64）导致 `mergedSchemaCache` 频繁驱逐
- 缓解：`MERGED_SCHEMA_CACHE_MAX = 64` 和 `EXTENSION_SCHEMA_TOTAL_MAX_BYTES = 2MB` 限制

**问题 4: 环境变量替换的 TOCTOU**
- 场景：配置加载时 `env[API_KEY]` 存在，写入时 env 已变化
- 缓解：`envSnapshotForRestore` 保存加载时刻的 env 快照；但仍有可能在快照保存到
  写入之间出现不一致

**问题 5: Nix 模式下的只读配置**
- 风险：Nix 管理的配置不可写但 gateway 尝试写入会失败
- 缓解：`resolveIsNixMode()` + `assertConfigWriteAllowedInCurrentMode()` 在写入前检查

### 9.2 面试模拟

**Q1: OpenClaw 的配置加载流程中，`sourceConfig` 和 `runtimeConfig` 为什么是分离的？**
> 答：这是 OpenClaw 配置系统的关键设计。`sourceConfig` 表示用户实际写入配置文件的内容，
> 而 `runtimeConfig` 是 sourceConfig + 运行时默认值注入后的完整配置。分离的原因是：
> (1) 写回配置文件时只写回 sourceConfig，避免将运行时注入的默认值（如 compaction 模式、
> context pruning 阈值）持久化到磁盘；(2) 配置差异比较基于 sourceConfig，确保只有用户
> 主动修改的路径触发 reload，而非默认值变化。

**Q2: 如何防止 `$include` 指令的路径遍历攻击（CWE-22）？**
> 答：`includes.ts` 实现了多层防护：(1) 词法路径检查：验证包含路径在 config 目录内；
> (2) symlink realpath 二次检查：解析符号链接后重新验证；`openRootFileSync` 在文件打开时
> 再次校验文件描述符在允许的根目录内；(3) `OPENCLAW_INCLUDE_ROOTS` 白名单机制允许
> 受控的跨目录包含；(4) 空字节检查、路径长度限制（4096 字符）、最大深度限制（10 层）；
> (5) 大文件限制（2MB），防止 DoS。

**Q3: 配置热重载中如何处理"某些变更需要彻底重启"的问题？**
> 答：`config-reload-plan.ts` 中定义了一个声明式的路径→动作映射规则表。
> 例如 `plugins.load` 和 `models.pricing` 变更需要完全重启网关，而 `hooks` 或 `cron`
> 变更可以热更新。`gateway.reload.mode` 配置项提供三个级别：auto（智能判断）、
> hot（强制热更新，忽略重启类变更）、restart（任何变更都重启）。
> 当进程内写操作通过 API 触发时，还会携带 `afterWrite` 意图，允许调用方声明期望的
> 重载行为（auto/restart/none）。

**Q4: 同一份 Zod Schema 既用于验证配置又用于生成 JSON Schema，是如何实现的？**
> 答：`schema-base.ts` 将 Zod Schema（`OpenClawSchema`）通过递归遍历转换为 JSON Schema
> 对象，同时配合 `FIELD_LABELS` 和 `FIELD_HELP` 字典提供字段标题和描述。
> `schema.ts` 在此基础上合并已安装插件的 config schema 和 channel schema，构建完整的
> JSON Schema。UI hints 系统（`schema.hints.ts`）提供额外的标签、敏感标记、心跳
> 目标提示等元数据。最后通过 LRU 缓存（最多 64 个）加速重复请求。

**Q5: 如何保证高并发下配置文件写入的一致性？**
> 答：OpenClaw 使用原子写入模式——先写入临时文件再 rename。写入前通过
> `resolveConfigWriteSuspiciousReasons()` 检测异常模式（如文件大小骤降 50%+、
> gateway mode 丢失），高风险写入被拒绝（payload 保存到 `.rejected` 后缀文件）。
> 写入后通过 `observeConfigSnapshot()` 计算新指纹并与 last-known-good 比较，
> 异常变化触发审计日志记录。如果 `finalizeRuntimeSnapshotWrite` 阶段失败（如 SecretRef
> 解析失败），系统会自动回滚文件到写入前状态。

---

## 10. 代码注释建议

以下是值得添加中文注释的关键位置（repo-root 相对路径）：

| 文件位置 | 建议注释内容 |
|----------|------------|
| `src/config/io.ts:1250-1270` | `resolveConfigForRead` 中 env 替换前先 apply config.env.vars 的原因 |
| `src/config/io.ts:2272-2300` | 写回时还原 ${VAR} 引用的 TOCTOU 防护策略 |
| `src/config/io.ts:2434-2453` | 可疑写操作的封锁策略与 payload quarantine |
| `src/config/validation.ts:1016-1019` | `validateConfigObjectWithPluginsBase` 的 two-phase 验证架构 |
| `src/config/defaults.ts:108-121` | `applyMessageDefaults` 的向后兼容保证 |
| `src/config/materialize.ts:26-45` | 三种材质化模式的选择逻辑 |
| `src/config/merge-patch.ts:25-60` | `mergeObjectArraysById` 的按 id 合并约定 |
| `src/config/includes.ts:240-266` | 词法路径 + realpath 双重安全检查防护 symlink bypass |
| `src/config/runtime-snapshot.ts:264-271` | `loadPinnedRuntimeConfig` 进程级单例锁定的意图 |
| `src/gateway/config-reload.ts:104-118` | 防 `pendingInProcessConfig` 与文件 watcher 冲突的策略 |
| `src/gateway/config-reload.ts:370-397` | chokidar 参数选择的依据 |
| `src/gateway/config-reload-plan.ts:54-116` | 各路径前缀对应 reload 动作的设计原理 |
| `src/config/schema.ts:571-613` | mergedSchemaCache 的边界限制 |
| `src/config/env-substitution.ts:197-203` | 非致命 env 缺失的设计考量和降级策略 |
| `src/config/config-env-vars.ts:83-91` | 跳过含 ${} 未解析引用的原因 |
| `src/plugin-sdk/plugin-config-runtime.ts:36-44` | 延迟加载器 vs 启动时配置的选择理由 |
