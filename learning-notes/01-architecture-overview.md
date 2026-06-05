# OpenClaw 系统架构与学习路线

> 更新日期：2026-06-04

---

## 一、项目概览

**OpenClaw** 是一个个人 AI 助手平台，运行在你自己的设备上。它通过你日常使用的消息渠道（Telegram、Discord、Slack、WhatsApp、iMessage 等）与你交互，提供一个"本地、快速、始终在线"的 AI 助手体验。

- **GitHub**: https://github.com/openclaw/openclaw
- **文档**: https://docs.openclaw.ai
- **项目愿景**: 见 `VISION.md`

---

## 二、系统架构

### 2.1 高层架构图

```
┌──────────────────────────────────────────────────────────────┐
│                        用户交互层                              │
│  ┌─────────┐ ┌──────────┐ ┌──────┐ ┌────────┐ ┌──────┐     │
│  │ Telegram │ │ Discord  │ │ Slack│ │WhatsApp│ │CLI/TUI│ ... │
│  └────▲─────┘ └▲─────────┘ └──▲───┘ └───▲────┘ └──▲────┘     │
│       │         │              │         │         │          │
├───────┼─────────┼──────────────┼─────────┼─────────┼──────────┤
│       │         │              │         │         │          │
│  ┌────┴─────────┴──────────────┴─────────┴─────────┴────┐    │
│  │                    Gateway (网关)                      │    │
│  │  ┌──────────────────────────────────────────────────┐ │    │
│  │  │  HTTP/WS Server  │ Auth │ Sessions │ Routing    │ │    │
│  │  └──────────────────────────────────────────────────┘ │    │
│  └────────────────────────┬─────────────────────────────┘    │
│                           │                                   │
├───────────────────────────┼───────────────────────────────────┤
│                           │                                   │
│  ┌────────────────────────▼─────────────────────────────┐    │
│  │                  Event Loop / Agent Loop              │    │
│  │  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌───────────┐  │    │
│  │  │ Provider │ │  Plugin  │ │ Memory │ │   Tools   │  │    │
│  │  │ Runtime  │ │  System  │ │ Engine │ │  & Skills │  │    │
│  │  └──────────┘ └──────────┘ └────────┘ └───────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
│                           │                                   │
├───────────────────────────┼───────────────────────────────────┤
│                           │                                   │
│  ┌────────────────────────▼──────────────────────────────┐   │
│  │              LLM Provider Layer                        │   │
│  │  OpenAI │ Claude │ Gemini │ DeepSeek │ ollama ...     │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Storage Layer                            │   │
│  │  SQLite (state) │ Config (JSON) │ Transcripts │ FS   │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 核心概念

| 概念 | 说明 |
|------|------|
| **Gateway** | 控制面的核心 HTTP/WS 服务器，处理认证、会话、消息路由、插件管理 |
| **Agent** | AI 助手的运行实例，管理 LLM 调用、工具执行、记忆、上下文 |
| **Channel** | 消息渠道适配层，将各种 IM 协议的 inbound/outbound 消息统一化 |
| **Provider** | LLM 提供商的抽象层（OpenAI、Anthropic、Google 等） |
| **Plugin** | 通过 Plugin SDK 扩展功能的模块（渠道插件、能力插件等） |
| **Extension** | 同 Plugin，在代码结构中指 `extensions/` 下的独立包 |
| **Session** | 一次对话的生命周期管理 |
| **Skill** | 预定义的 AI 执行技能（slash command 形式 `/skill`） |
| **Node** | Gateway 的远程节点（如 Mac 桌面端、移动端） |

### 2.3 系统特性

- **单用户设计**: 专注于单用户个人助手，不是 SaaS 多租户
- **多渠道聚合**: 统一管理主流 IM 消息渠道
- **Provider 无关**: 通过抽象层支持多种 LLM 提供商，可自由切换
- **插件化架构**: 通过 Plugin SDK 扩展能力
- **多模态支持**: 文字、图片、语音、视频、文件的理解与生成
- **记忆系统**: 持久化记忆引擎，支持长期上下文
- **本地部署**: 数据保留在自己设备上

---

## 三、代码结构总览

### 3.1 顶层目录

| 目录/文件 | 说明 |
|-----------|------|
| `src/` | 核心源码（TypeScript） |
| `extensions/` | 插件/扩展（渠道、Provider等，独立包） |
| `packages/` | 共享的内部包 |
| `ui/` | 前端 UI（Control UI / Canvas） |
| `docs/` | 文档源文件 |
| `apps/` | 平台应用（iOS / Android / Desktop 入口） |
| `scripts/` | 构建、部署、开发脚本 |
| `test/` | 测试基础设施与 Vitest 配置 |
| `config/` | 项目配置模板 |
| `qa/` | QA 测试辅助 |
| `skills/` | 预置 AI Skills |
| `openclaw.mjs` | CLI 入口点 |
| `pnpm-workspace.yaml` | Monorepo 工作区定义 |

### 3.2 核心源码 (`src/`) 详细结构

#### 入口与启动

| 路径 | 说明 |
|------|------|
| `src/bootstrap/` | 启动前置（CA 证书、环境检查） |
| `src/cli/program.ts` | CLI 程序主入口，命令路由 |
| `src/cli/run-main.ts` | CLI 主运行流程 |
| `src/gateway/boot.ts` | Gateway 启动 |
| `src/gateway/client.ts` | Gateway 客户端 |

#### 网关层 (Gateway)

| 路径 | 说明 |
|------|------|
| `src/gateway/` | 网关核心（约 400+ 文件，最大的模块） |
| `src/gateway/server/` | HTTP/WS 服务器实现 |
| `src/gateway/server-*.ts` | 服务端各子模块拆分 |
| `src/gateway/server.ts` | 服务端主入口 |
| `src/gateway/server-http.ts` | HTTP 服务器与路由 |
| `src/gateway/boot.ts` | 启动序列 |
| `src/gateway/auth*.ts` | 认证系统（device auth、token、OAuth） |
| `src/gateway/session*.ts` | 会话管理（生命周期、存储、迁移） |
| `src/gateway/hooks*.ts` | 插件钩子系统 |
| `src/gateway/events.ts` | 事件系统 |
| `src/gateway/config*.ts` | 配置加载/重载/热更新 |
| `src/gateway/probe*.ts` | 健康探测 |
| `src/gateway/startup-*.ts` | 各阶段启动逻辑 |
| `src/gateway/control-ui*.ts` | 控制台 UI 后端 |

#### Agent 引擎

| 路径 | 说明 |
|------|------|
| `src/agents/` | Agent 引擎（LLM 调用、工具执行、对话循环） |
| `src/agents/agent-command.ts` | Agent 对话主循环 |
| `src/agents/agent-scope.ts` | Agent 作用域/配置 |
| `src/agents/agent-steering-queue.ts` | Agent 消息队列调度 |
| `src/agents/agent-settings.ts` | Agent 设置管理 |
| `src/agents/acp-spawn.ts` | ACP (Agent Communication Protocol) 子 agent 生成 |
| `src/agents/agent-tools-*.ts` | 工具调用相关 |
| `src/agents/agent-bundle-mcp*.ts` | MCP 工具集成 |
| `src/agents/agent-hooks/` | Agent 钩子 |
| `src/agents/agent-runtime-*.ts` | Agent 运行时配置 |

#### 渠道系统 (Channel)

| 路径 | 说明 |
|------|------|
| `src/channels/` | 渠道抽象层 |
| `src/channels/channel-config.ts` | 渠道配置 |
| `src/channels/session.ts` | 渠道会话绑定 |
| `src/channels/typing.ts` | 输入状态指示 |
| `src/channels/targets.ts` | 渠道目标寻址 |
| `src/channels/mention-gating.ts` | @提及门控 |
| `src/channels/plugins/` | 渠道插件相关 |
| `src/channels/transport/` | 传输层适配 |
| `src/channels/message/` | 消息处理 |

#### Provider 系统

| 路径 | 说明 |
|------|------|
| `src/provider-runtime/` | Provider 运行时（当前只有重试逻辑） |
| `src/plugins/providers*.ts` | Provider 注册与发现 |
| `src/plugins/provider-auth*.ts` | Provider 认证 |
| `src/plugins/provider-catalog.ts` | Provider 目录 |
| `src/plugins/provider-runtime.ts` | Provider 运行时 |
| `src/plugins/provider-discovery.ts` | Provider 发现机制 |
| `src/plugins/provider-model-*.ts` | Provider 模型管理 |
| `src/llm/providers/` | LLM 提供商具体实现 |

#### 插件系统 (Plugin)

| 路径 | 说明 |
|------|------|
| `src/plugins/` | 插件系统的核心（约 300+ 文件） |
| `src/plugins/loader.ts` | 插件加载器 |
| `src/plugins/install.ts` | 插件安装 |
| `src/plugins/runtime.ts` | 插件运行时管理 |
| `src/plugins/manifest*.ts` | 插件清单解析 |
| `src/plugins/bundled*.ts` | 内置插件管理 |
| `src/plugins/config*.ts` | 插件配置 |
| `src/plugins/hooks.ts` | 插件钩子注册 |
| `src/plugins/slots.ts` | 插件插槽 |
| `src/plugins/registry*.ts` | 插件注册表 |
| `src/plugin-sdk/` | **Plugin SDK** — 插件开发者使用的公共 API |

#### Plugin SDK (`src/plugin-sdk/`)

这是插件开发者面向的公共 API 层，包含:

| 文件 | 说明 |
|------|------|
| `index.ts` | SDK 主要导出 |
| `core.ts` | 核心类型 |
| `agent-core.ts` | Agent 运行时接口 |
| `agent-runtime.ts` | Agent 运行时实现 |
| `channel-core.ts` | 渠道核心接口 |
| `channel-runtime.ts` | 渠道运行时 |
| `channel-lifecycle.ts` | 渠道生命周期 |
| `channel-message.ts` | 渠道消息处理 |
| `channel-streaming.ts` | 渠道流式输出 |
| `provider-entry.ts` | Provider 入口规范 |
| `provider-stream.ts` | Provider 流式调用 |
| `provider-auth.ts` | Provider 认证 |
| `runtime.ts` | SDK 运行时基类 |
| `browser-*.ts` | 浏览器相关 |
| `memory-core*.ts` | 记忆引擎 SDK |
| `config-runtime.ts` | 配置运行时 |
| `fetch-runtime.ts` | 网络请求运行时 |
| `approval*.ts` | 审批流程 |
| `tool-plugin.ts` | 工具插件 |
| `conversation-binding.ts` | 对话绑定 |
| `secret-input.ts` | 密钥输入 |

#### 配置系统

| 路径 | 说明 |
|------|------|
| `src/config/` | 配置加载、验证、合并 (Zod Schemas) |
| `src/config/schema.ts` | JSON Schema 生成 |
| `src/config/io.ts` | 配置文件 I/O |
| `src/config/merge-patch.ts` | 配置合并与补丁 |
| `src/config/defaults.ts` | 默认值 |
| `src/config/zod-schema*.ts` | Zod 类型定义 (各模块) |
| `src/config/types.*.ts` | 配置类型定义 |

#### 其他核心模块

| 路径 | 说明 |
|------|------|
| `src/sessions/` | 会话管理与生命周期 |
| `src/routing/` | 消息路由（channel → agent/account） |
| `src/commands/` | CLI 命令实现（`agents.*`、`channels.*` 等） |
| `src/chat/` | 聊天界面/流式渲染 |
| `src/memory/` | 记忆引擎核心 |
| `src/mcp/` | MCP (Model Context Protocol) 支持 |
| `src/secrets/` | 密钥管理 |
| `src/tools/` | 工具定义与注册 |
| `src/tasks/` | 任务管理 |
| `src/flows/` | 流程编排 |
| `src/skills/` | Core 技能管理 |
| `src/cron/` | 定时任务 |
| `src/hooks/` | 钩子系统 |
| `src/llm/` | LLM 抽象（模型注册、流式、OAuth） |
| `src/context-engine/` | 上下文引擎 |
| `src/talk/` | 语音对话 |
| `src/tts/` | 文本转语音 |
| `src/media/` | 媒体文件处理 |
| `src/media-generation/` | 媒体生成 |
| `src/media-understanding/` | 媒体理解 |
| `src/image-generation/` | 图片生成 |
| `src/video-generation/` | 视频生成 |
| `src/music-generation/` | 音乐生成 |
| `src/web-fetch/` | 网页抓取 |
| `src/web-search/` | 网页搜索 |
| `src/link-understanding/` | 链接理解 |
| `src/realtime-transcription/` | 实时语音转文字 |
| `src/acp/` | Agent Communication Protocol |
| `src/state/` | 状态管理 (SQLite) |
| `src/i18n/` | 国际化 |
| `src/web/` | Web 功能 |
| `src/wizard/` | 引导流程 |
| `src/daemon/` | 守护进程 |
| `src/pairing/` | 设备配对 |
| `src/credentials/` | 凭证管理 |
| `src/status/` | 状态监控 |
| `src/logging/` | 日志系统 |
| `src/security/` | 安全检查 |
| `src/compat/` | 兼容性层 |
| `src/infra/` | 基础设施 |
| `src/interactive/` | 交互模式 |
| `src/bindings/` | 绑定管理 |
| `src/transcripts/` | 对话转录 |
| `src/trajectory/` | Agent 运行轨迹 |
| `src/commitments/` | 承诺/规约 |
| `src/auto-reply/` | 自动回复 |
| `src/crestodian/` | 文件系统守护 |
| `src/docs/` | 文档生成功能 |
| `src/node-host/` | Node.js 主机环境 |
| `src/memory-host-sdk/` | 记忆宿主 SDK |
| `src/plugin-state/` | 插件状态管理 |
| `src/proxy-capture/` | 代理捕获 |
| `src/process/` | 进程管理 |
| `src/shared/` | 共享工具函数 |
| `src/types/` | 通用类型定义 |
| `src/utils/` | 通用工具函数 |
| `src/test-helpers/` | 测试辅助 |
| `src/test-utils/` | 测试工具 |

### 3.3 内部包 (`packages/`)

| 包名 | 说明 |
|------|------|
| `agent-core` | Agent 运行时核心定义 |
| `plugin-sdk` | Plugin SDK 包（同 `src/plugin-sdk/`，构建产物） |
| `sdk` | OpenClaw SDK |
| `gateway-protocol` | Gateway 通信协议定义 |
| `gateway-client` | Gateway 客户端 |
| `llm-core` | LLM 核心类型 |
| `llm-runtime` | LLM 运行时 |
| `model-catalog-core` | 模型目录核心 |
| `media-core` | 媒体核心 |
| `media-generation-core` | 媒体生成核心 |
| `media-understanding-common` | 媒体理解公共 |
| `memory-host-sdk` | 记忆宿主 SDK |
| `net-policy` | 网络策略 |
| `speech-core` | 语音核心 |
| `terminal-core` | 终端核心 |
| `web-content-core` | 网页内容核心 |
| `tool-call-repair` | 工具调用修复 |
| `normalization-core` | 标准化核心 |
| `acp-core` | ACP 核心 |
| `markdown-core` | Markdown 核心 |
| `plugin-package-contract` | 插件包契约 |

### 3.4 插件/扩展 (`extensions/`)

141 个扩展包，主要分为以下几类：

**渠道插件（消息平台）**:
`discord`, `telegram`, `slack`, `whatsapp`, `signal`, `discord`, `google-chat`, `imessage`, `mattermost`, `matrix`, `feishu`, `line`, `wechat`, `qq`, `zalo`, `irc`, `msteams`, `nostr`, `twitch`, `webchat` ...

**Provider 插件（LLM 提供商）**:
`anthropic`, `openai`, `google-gemini`, `deepseek`, `openrouter`, `amazon-bedrock`, `cerebras`, `groq`, `perplexity`, `together`, `fireworks`, `github-copilot` ...

**功能插件**:
`browser` (浏览器自动化), `memory` (记忆), `canvas`, `diffs`, `codex`, `cron`, `image-generation`, `tts`, `text-to-speech`, `web-fetch` ...

### 3.5 前端 (`ui/`)

Control UI 的 Vite + React 前端，提供 Web 管理界面和 Canvas 交互界面。

---

## 四、学习路线

### 第一阶段：快速上手（1-2 天）

**目标**: 本地启动并体验核心功能

1. **安装依赖并启动**
   ```bash
   pnpm install
   pnpm openclaw onboard   # 交互式配置向导
   pnpm dev                # 开发模式启动
   ```

2. **体验 CLI 命令**
   ```bash
   pnpm openclaw --help
   pnpm openclaw status
   pnpm openclaw channels list
   pnpm openclaw agents list
   pnpm openclaw models list
   ```

3. **理解数据流**: 看一个消息从 Channel → Gateway → Agent → LLM → 返回的全路径

### 第二阶段：理解核心架构（1 周）

**顺序阅读源码，建议按以下顺序**:

#### Step 1: CLI 入口 → Gateway 启动
- `openclaw.mjs` - CLI 启动脚本
- `src/cli/program.ts` - 命令路由
- `src/cli/run-main.ts` - 主流程
- `src/gateway/boot.ts` - Gateway 启动
- `src/gateway/startup-*.ts` - 各启动阶段

#### Step 2: 配置系统
- `src/config/defaults.ts` - 默认配置
- `src/config/io.ts` - 配置加载
- `src/config/schema.ts` - Schema 生成
- `src/config/zod-schema*.ts` - 类型定义
- `src/config/merge-patch.ts` - 合并逻辑

#### Step 3: 认证与会话
- `src/gateway/auth*.ts` - 认证
- `src/gateway/connection-auth.ts` - 连接认证
- `src/sessions/` - 会话管理
- `src/gateway/server-chat.ts` - 聊天请求处理

#### Step 4: Agent 引擎（最核心）
- `src/agents/agent-command.ts` - Agent 主循环（核心！）
- `src/agents/agent-scope.ts` - 作用域配置
- `src/agents/agent-steering-queue.ts` - 消息队列
- `src/plugin-sdk/agent-core.ts` - Agent SDK 接口

#### Step 5: Channel 系统
- `src/channels/channel-config.ts` - 渠道配置
- `src/channels/session.ts` - 会话绑定
- `src/channels/transport/` - 传输适配
- `extensions/discord/src/` - 选一个具体渠道深入

#### Step 6: Plugin 系统
- `src/plugins/loader.ts` - 加载器
- `src/plugins/install.ts` - 安装
- `src/plugins/manifest.ts` - 清单
- `src/plugins/hooks.ts` - 钩子系统
- `src/plugins/runtime.ts` - 运行时
- `src/plugin-sdk/` - SDK 设计

#### Step 7: Provider 系统
- `src/plugins/provider-runtime.ts` - Provider 运行时
- `src/plugins/provider-catalog.ts` - 目录
- `src/plugins/provider-auth*.ts` - 认证
- `src/plugin-sdk/provider-entry.ts` - Provider SDK
- `src/llm/providers/` - 具体实现

### 第三阶段：深入专题（2-4 周）

根据兴趣选择深入方向:

| 方向 | 关键源码 | 前置知识 |
|------|----------|----------|
| **Agent 循环** | `agent-command.ts`, `agent-scope.ts` | 阶段二基础 |
| **工具调用** | `agent-tools-*.ts`, `src/tools/` | Agent 循环 |
| **记忆系统** | `src/memory/`, `plugin-sdk/memory-core*.ts` | Agent、Plugin |
| **MCP 协议** | `src/mcp/`, `agent-bundle-mcp*.ts` | Agent、Plugin |
| **流式输出** | `channel-streaming.ts`, `src/chat/` | Channel |
| **插件开发** | `src/plugin-sdk/`, `extensions/*/src/` | Plugin 系统 |
| **Gateway 协议** | `packages/gateway-protocol/` | 网络编程 |
| **配置与 Schema** | `src/config/`, Zod | TypeScript |
| **技能系统** | `skills/`, `src/skills/` | 插件系统 |
| **多模态** | `media-generation/`, `src/media/` | Provider |
| **语音对话** | `src/talk/`, `src/tts/` | Provider |
| **前端 (Control UI)** | `ui/` | React, Vite |
| **测试体系** | `test/`, `vitest.config.ts` | Vitest |

### 第四阶段：参与贡献

1. 阅读 `CONTRIBUTING.md` 和 `AGENTS.md`
2. 从 Good First Issue 开始
3. 理解测试规范 (`docs/reference/test.md`)
4. 掌握 CI/CD 流程
5. 学习 ClawSweeper (代码审查机器人) 的工作方式

---

## 五、本地启动与调试

### 5.1 环境要求

```bash
# Node.js 22.19+ (推荐 Node 24)
node --version

# 包管理器
npm install -g pnpm

# 克隆项目
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

### 5.2 安装依赖

```bash
pnpm install          # 安装所有依赖
```

### 5.3 构建

```bash
pnpm build            # 构建全部
pnpm build:core       # 仅构建核心
```

### 5.4 开发模式

```bash
pnpm dev              # 开发模式启动（自动检测变更重启）
```

### 5.5 配置向导

```bash
pnpm openclaw onboard           # 交互式配置（推荐）
# 或者直接手动配置 openclaw.json
```

### 5.6 常用命令

```bash
pnpm openclaw status                     # 查看运行状态
pnpm openclaw channels add               # 添加消息渠道
pnpm openclaw agents list                # 查看 Agent
pnpm openclaw models list                # 查看可用模型
pnpm openclaw skills list                # 查看技能
pnpm openclaw doctor                     # 系统诊断
```

### 5.7 运行测试

```bash
pnpm test                               # 完整测试
pnpm test:changed                       # 仅测试变更文件
pnpm test src/gateway/server.test.ts    # 单文件测试
pnpm test:coverage                      # 测试覆盖率
pnpm test:serial                        # 串行执行测试
pnpm test:extensions                    # 插件测试
pnpm test:live                          # 需要 API Key 的实网测试
```

### 5.8 代码质量

```bash
pnpm format:check          # 格式化检查
pnpm format:fix            # 自动格式化
pnpm lint:check            # Lint 检查
pnpm check:import-cycles   # 导入循环检查
pnpm check                 # 全量代码质量检查
```

### 5.9 调试技巧

1. **日志级别**: 设置 `OPENCLAW_LOG_LEVEL=debug` 环境变量获取详细日志
2. **单元测试调试**: 直接运行单文件测试 `pnpm test <path>`
3. **VSCode 调试配置**:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug OpenClaw",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["dev"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

4. **Gateway 调试**: `pnpm openclaw gateway start --log-level debug`
5. **测试调试**: 使用 Vitest VSCode 扩展直接运行单测

---

## 六、关键数据流

### 6.1 消息接收流（Inbound）

```
Channel (Telegram Webhook)
  → extensions/telegram/src/index.ts
    → Gateway HTTP endpoint
      → server-http.ts (路由)
        → auth (认证/鉴权)
          → server-channels.ts (渠道消息处理)
            → channels/ (消息规范化)
              → server-chat.ts (聊天请求)
                → agents/agent-command.ts (Agent 主循环)
                  → LLM Provider (模型调用)
                    → 工具调用 / 插件执行
                      → 回复生成
```

### 6.2 消息发送流（Outbound）

```
Agent 回复
  → channels/reply-pipeline.ts (回复流水线)
    → channel-streaming.ts (流式输出)
      → transport/ (渠道传输适配)
        → IM Platform (Telegram API)
```

---

## 七、核心设计理念

从 AGENTS.md 中提炼的关键设计哲学:

1. **插件解耦**: 核心保持插件无关，插件通过 SDK 契约交互
2. **配置即契约**: 配置 Schema 严格，避免向后兼容的兼容层
3. **SQLite 优先**: 运行时状态优先使用 SQLite，避免文件碎片
4. **热路径优化**: 运行时缓存准备好的数据结构，避免重复发现
5. **确定性排序**: Prompt 缓存中保持 Map/Set/插件列表的顺序确定性
6. **窄接口**: 只导出当前调用方需要的类型，保持 API 面小
7. **减少 fallback**: Fallback 是产品决策而非实现便利
8. **数据迁移优先 Doctor**: 配置/数据迁移通过 `openclaw doctor --fix` 完成，运行时不做兼容

---

> **下一步建议**: 打开 `src/gateway/boot.ts` 和 `src/agents/agent-command.ts`，从启动流程和 Agent 主循环开始逐行阅读。
