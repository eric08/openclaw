# OpenClaw 技术分析报告

> 生成日期：2026-02-25
> 版本：2026.2.25（未发布）

---

## 1. 项目概述

**OpenClaw** 是一个开源的个人 AI 助手（Personal AI Assistant）网关，支持在用户自有设备上运行，并可通过主流即时通讯渠道（WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等）与 AI 模型交互。项目遵循 **MIT 许可证**，核心定位为"终端优先、隐私友好、可扩展"的 AI 编排系统。

- **仓库**：<https://github.com/openclaw/openclaw>
- **网站**：<https://openclaw.ai>
- **文档**：<https://docs.openclaw.ai>
- **版本策略**：日历版本（`YYYY.M.D`），稳定/Beta/Dev 三条发布通道

---

## 2. 技术栈

| 类别 | 技术选型 |
|---|---|
| 运行时 | Node.js ≥ 22（同时支持 Bun） |
| 语言 | TypeScript (ESM, strict 模式) |
| 包管理 | pnpm（workspace monorepo） |
| 构建工具 | tsdown（输出到 `dist/`） |
| Lint/格式化 | Oxlint + Oxfmt |
| 测试框架 | Vitest（V8 覆盖率） |
| CLI 框架 | Commander + @clack/prompts |
| 容器化 | Docker（多镜像：sandbox、sandbox-browser、生产） |
| 托管 | Fly.io、Render（`fly.toml` / `render.yaml`） |
| 移动端 | Android（Kotlin/Gradle）、iOS/macOS（Swift/SwiftUI） |

---

## 3. 项目结构

```
openclaw/
├── src/                  # 核心 TypeScript 源码（2 335 个 .ts 文件）
│   ├── agents/           # AI Agent 运行器、工具执行、子 Agent 编排
│   ├── gateway/          # HTTP + WebSocket 网关服务器
│   ├── channels/         # 频道路由、允许列表、命令门控
│   ├── config/           # 配置 Schema、读写、校验
│   ├── cli/              # CLI 命令解析与注入依赖
│   ├── commands/         # 具体命令实现
│   ├── plugins/          # 插件注册表与 Hook 运行器
│   ├── security/         # 安全审计、路径守卫、正则安全
│   ├── memory/           # 记忆管理（QMD、批量嵌入）
│   ├── infra/            # 基础设施工具（格式化、心跳、环境变量等）
│   ├── terminal/         # 终端输出（表格、主题色）
│   ├── media/            # 媒体处理管道
│   ├── tts/              # 文字转语音
│   └── ...（telegram、discord、slack、signal、imessage 等渠道模块）
├── extensions/           # 扩展插件包（38 个，独立 npm 工作区）
├── apps/
│   ├── android/          # Android 原生应用（Kotlin）
│   ├── ios/              # iOS 应用（SwiftUI）
│   └── macos/            # macOS 伴侣应用（SwiftUI）
├── ui/                   # Web 控制界面（Control UI）
├── docs/                 # Mintlify 文档（含 zh-CN 机器翻译）
├── packages/             # 共享 npm 包（shared 等）
├── skills/               # 捆绑技能（Markdown 文件）
└── scripts/              # 构建/发布辅助脚本
```

---

## 4. 核心架构

### 4.1 网关（Gateway）

网关是整个系统的控制平面，以 HTTP + WebSocket 双协议运行：

- **HTTP API**：配置读写、模型目录、会话管理、工具调用代理
- **WebSocket**：节点订阅、实时事件推送（助手流式回复、工具执行状态）
- **认证层**：基于令牌 + 速率限制（`auth-rate-limit.ts`）
- **配置热重载**：检测文件变化后无停机应用新配置（`config-reload.ts`）
- **健康监控**：各渠道周期性心跳探测（`channel-health-monitor.ts`）
- **执行审批**：沙盒外命令需经用户审批（`exec-approval-manager.ts`）

### 4.2 AI Agent 运行器

`src/agents/` 是系统最复杂的模块，包含：

- **pi-embedded-runner**：封装 Anthropic Claude SDK（`@anthropic-ai/sdk`）的嵌入式 Agent 运行器，支持流式输出、工具调用、压缩（Compaction）、会话持久化
- **模型故障转移**（`model-fallback.ts`）：多候选模型链式降级，支持 `rate_limit` / `cooling_down` / 未知错误的不同处理策略
- **认证配置轮换**（`auth-profiles.ts`）：多 API Key 轮换、最近使用排序、冷却期管理
- **子 Agent 编排**（`subagent-registry.ts`）：嵌套 Agent 树、深度限制、生命周期追踪
- **工具政策管道**（`tool-policy-pipeline.ts`）：工具调用前/后拦截、文件系统路径守卫
- **Bash 工具**（`bash-tools.ts`）：PTY/沙盒执行、脚本预检、后台进程管理
- **技能系统**（`skills.ts`）：从工作区加载自定义 Markdown 技能并注入系统提示

### 4.3 渠道系统

每个消息渠道作为独立适配器实现，通过统一的 Plugin SDK 接口注册：

**核心渠道（内置）**：WhatsApp（Baileys）、Telegram（Telegraf）、Slack（Bolt）、Discord、Signal、iMessage、LINE、WebChat

**扩展渠道（extensions/）**：
BlueBubbles、Matrix、Zalo、Zalo Personal、Microsoft Teams、Google Chat、IRC、Feishu、Synology Chat、Nextcloud Talk、Mattermost、Nostr、Tlon、Twitch、语音通话

每个渠道实现以下适配器接口（部分可选）：
`ChannelGatewayAdapter`、`ChannelAuthAdapter`、`ChannelCommandAdapter`、`ChannelGroupAdapter`、`ChannelHeartbeatAdapter`、`ChannelDirectoryAdapter`

### 4.4 插件 SDK

`src/plugin-sdk/` 暴露稳定的公共 API（独立导出路径 `openclaw/plugin-sdk`），包含：

- 渠道适配器类型定义
- 工具工厂接口
- 账户管理助手
- Hook 事件类型

插件打包为独立 npm 包，运行时通过 `jiti` 别名解析 `openclaw/plugin-sdk`。

### 4.5 记忆系统

`src/memory/` 实现可插拔记忆后端：

- **QMD Manager**（`qmd-manager.ts`，~1 900 行）：基于 Markdown 的量化记忆，支持批量嵌入检索
- **LanceDB 扩展**（`extensions/memory-lancedb`）：基于向量数据库的语义搜索

---

## 5. 支持的 AI 模型提供商

OpenClaw 通过模型目录（`model-catalog.ts`）和模型配置（`models-config.ts`）支持以下提供商：

| 提供商 | 说明 |
|---|---|
| Anthropic | Claude 系列（推荐 Opus 4.6） |
| OpenAI | GPT / Codex 系列 |
| Google Gemini | 含 Google AI Studio + Vertex |
| Ollama | 本地模型 |
| Bedrock | AWS Bedrock |
| HuggingFace | 推断 API |
| DouBao / VolcEngine | 字节跳动云 |
| MiniMax | MiniMax VLM |
| Moonshot | Kimi |
| Venice | Venice AI |
| Together | Together AI |
| OpenCode Zen | 编码专项 |
| GitHub Copilot | 代理令牌 |
| Kilocode | 编码模型 |
| Chutes | Chutes OAuth |
| Qwen（通义千问）| Alibaba Cloud |
| Qianfan（文心）| Baidu |
| Byteplus | BytePlus |

---

## 6. 测试覆盖

| 指标 | 数值 |
|---|---|
| `src/` 中的测试文件 | 1 405 个 |
| `extensions/` 中的测试文件 | 146 个 |
| 测试超时（默认） | 120 秒 |
| 覆盖率阈值（V8）| 70%（行/分支/函数/语句） |
| CI 并发工作线程（非 Windows）| 3 |
| 本地并发工作线程 | max(4, min(16, CPU 数)) |

测试类型：
- **单元测试**（`*.test.ts`）：大量并置单测，覆盖各模块独立逻辑
- **E2E 测试**（`*.e2e.test.ts`）：需启动实际 Gateway 的集成测试
- **Live 测试**（`*.live.test.ts`）：需真实 API Key，默认跳过
- **Docker 测试**：`pnpm test:docker:live-models` / `pnpm test:docker:live-gateway`

---

## 7. 安全设计

OpenClaw 的安全模型体现在以下几个维度：

1. **工具执行安全**
   - 文件系统路径守卫（`tool-fs-policy.ts`）：防止路径遍历
   - 沙盒策略（`sandbox.ts`）：可选容器化执行环境
   - 危险工具标记与审批流程（`dangerous-tools.ts`）
   - 危险配置标志检测（`dangerous-config-flags.ts`）

2. **渠道安全**
   - 允许列表机制（`channels/allowlists/`）：精确控制消息来源
   - DM 专用配对存储与群组允许列表严格隔离（防跨上下文授权）
   - Webhook 签名验证（各渠道各自实现，Nextcloud Talk 已修复前置验证漏洞）

3. **网关安全**
   - 速率限制（`auth-rate-limit.ts`）
   - CORS/CSP 保护（`control-ui-csp.ts`）
   - Origin 检查（`origin-check.ts`）
   - 控制平面速率限制（`control-plane-rate-limit.ts`）

4. **审计**
   - 同步/异步双层安全审计（`audit-extra.sync.ts` / `audit-extra.async.ts`，各约 1 200 行）
   - 密钥泄露扫描（`.detect-secrets.cfg` / `.secrets.baseline`）
   - `zizmor.yml`：GitHub Actions 工作流安全检查

---

## 8. 持续集成与 DevOps

```
.github/workflows/
├── ci.yml           # 主 CI：lint、tsgo、test
└── ...
```

- **Pre-commit hooks**：`prek install`（与 CI 相同检查）
- **死代码检测**：knip + ts-prune + ts-unused-exports
- **代码行数限制**：`pnpm check:loc`（单文件建议 ≤ 500 行，软性限制）
- **文档链接检查**：`pnpm docs:check-links`

---

## 9. 客户端应用

| 平台 | 技术 | 状态 |
|---|---|---|
| macOS | SwiftUI（`@Observable` 框架）+ Sparkle 自动更新 | 活跃 |
| iOS | SwiftUI | 活跃 |
| Android | Kotlin + Jetpack Compose | 活跃 |
| Web Control UI | TypeScript（`ui/`） | 活跃 |
| Canvas | A2UI 捆绑（`src/canvas-host/`） | 活跃 |

---

## 10. 关键指标汇总

| 指标 | 数值 |
|---|---|
| 核心 TypeScript 源文件（`src/`） | 2 335 个 |
| 最大单文件行数 | ~1 900 行（`qmd-manager.ts`） |
| 扩展插件数量 | 38 个 |
| 支持消息渠道总数 | ~20 个 |
| 支持 AI 模型提供商 | ~18 个 |
| pnpm workspace 包数 | 4 个顶层（`.`、`ui`、`packages/*`、`extensions/*`） |
| npm 包体积（发布文件） | `dist/`、`assets/`、`skills/`、`docs/`、`extensions/` |

---

## 11. 总结与建议

**优势**：
- 插件架构设计清晰，渠道、记忆、工具均可扩展
- 测试覆盖密度高（1 400+ 测试文件，并发 lint+type+test 的完整 CI）
- 安全意识强：工具政策、允许列表隔离、审计日志一应俱全
- 多模型提供商支持完善，故障转移机制健全

**潜在改进点**：
- 部分核心文件超过 1 900 行，可进一步拆分（如 `qmd-manager.ts`、`pi-embedded-runner/attempt.ts`）
- Agent 运行器深度耦合 Anthropic SDK（`pi-embedded`），如需深度支持其他提供商的原生流式 API 仍需适配工作
- 文档 i18n 依赖机器翻译脚本，质量参差

---

*本报告由自动化分析生成，基于代码库截止 2026-02-25 的状态。*
