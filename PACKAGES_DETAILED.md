# Pi 项目 `packages/` 完整文件路径与源码架构详解

本文档对 `packages/` 下 11 个核心子包的内部目录组织、源文件（`src/`）、测试（`test/`）以及重点模块的设计实现进行逐一深度解析。

---

## 目录索引

1. [`packages/coding-agent`（核心智能体 CLI 与运行时）](#1-packagescoding-agent)
2. [`packages/agent`（智能体核心循环驱动）](#2-packagesagent)
3. [`packages/ai`（统一大模型 API 与 Provider）](#3-packagesai)
4. [`packages/tui`（差量渲染终端用户界面）](#4-packagestui)
5. [`packages/mcp`（Model Context Protocol 客户端）](#5-packagesmcp)
6. [`packages/codemode`（QuickJS Wasm 沙箱执行器）](#6-packagescodemode)
7. [`packages/durable`（长持久化会话与任务运行时）](#7-packagesdurable)
8. [`packages/env`（SSH 远端宿主环境守护进程）](#8-packagesenv)
9. [`packages/telemetry`（中立遥测与指标度量）](#9-packagestelemetry)
10. [`packages/chord`（分布式契约与状态同步运行时）](#10-packageschord)
11. [`packages/evals`（智能体自动化评测套件）](#11-packagesevals)

---

## 1. `packages/coding-agent`

`@earendil-works/pi-coding-agent` 是面向开发者的核心交互入口，集成了会话管理、工具调度、TUI 视图、RPC 远程协议与扩展系统。

### 1.1 顶层入口
- `src/main.ts`：Node.js CLI 执行主入口，负责环境探测、全局未捕获异常处理与命令派发。
- `src/cli.ts`：命令行入口封装，包含参数解析与模式初始化。
- `src/rpc-entry.ts`：外部通过独立进程直接以 RPC 服务方式拉起 pi 时的专用入口。
- `src/config.ts`：统一资产与配置文件定位解析器（处理源码运行、npm 全局安装及独立打包不同场景的路径映射）。
- `src/migrations.ts`：本地用户数据、会话历史与配置 Schema 的升级迁移器。
- `src/package-manager-cli.ts`：管理通过 npm / git 安装的 pi 扩展包的 CLI 辅助。

### 1.2 `src/cli/`（命令行启动阶段）
- `args.ts`：基于轻量解析器提取 CLI 标志参数（`--mode`、`--model`、`--provider` 等）。
- `setup.ts` / `startup-ui.ts`：首次启动引导、检查依赖与欢迎界面呈现。
- `auth-command.ts` / `auth-check.ts`：`/login` 命令处理及运行前 API Key / OAuth 凭证有效性检查。
- `session-picker.ts`：启动时选择继续已有历史会话或新建会话的交互选择器。
- `project-trust.ts`：项目信任评估（安全策略：针对首次打开的非受信仓库限制危险工具自动执行）。
- `list-models.ts`：列出本地可用和已配置的所有模型列表。
- `file-processor.ts`：解析从 CLI 参数传入的文件路径或直接重定向的内容。

### 1.3 `src/core/`（核心智能体运行时体系）
- **会话与生命周期**：
  - `agent-session.ts` / `agent-session-runtime.ts` / `agent-session-services.ts`：Agent 会话的核心控制层，管理历史消息树、状态机流转、事件总线分发以及工具拦截器。
  - `session-manager.ts`：磁盘上的会话持久化、检索、分支管理。
  - `session-cwd.ts`：会话工作目录切换跟踪。
  - `session-export.ts`：导出当前会话为不同格式（如 Markdown、HTML 等）。
- **执行器与工具**：
  - `bash-executor.ts`：负责实际调用本地 shell 执行系统命令，支持信号监听、超时终止与管道分块。
  - `exec.ts`：底层子进程执行封装。
  - `src/core/tools/`：内置基础智能体工具：
    - `read.ts`：读取代码文件（支持指定行切片、精确范围及图片预览）。
    - `edit.ts` / `edit-diff.ts`：基于 Hashline 和精确锚点的安全文件编辑修改器。
    - `write.ts`：新建或完全重写文件。
    - `bash.ts` / `powershell.ts`：跨平台的 Shell 执行工具实现。
    - `grep.ts` / `find.ts` / `ls.ts`：文件搜索、模式正则匹配与目录检索工具。
    - `file-mutation-queue.ts`：防止多工具并发写冲突的写入队列。
    - `truncate.ts` / `output-accumulator.ts`：工具输出结果截断与超长缓冲保护。
    - `renderers/`：工具在终端上的专属渲染器（将 read/edit/bash 结果格式化为优美的 ANSI/UI 输出）。
- **上下文压缩与剪枝（`src/core/compaction/`）**：
  - `compaction.ts`：当上下文接近模型 token 阈值时自动触发压缩。
  - `branch-summarization.ts`：将历史轮次摘要为结构化总结，保留关键结论并丢弃工具的大量原始输出。
- **扩展加载机制（`src/core/extensions/`）**：
  - `loader.ts` / `jiti-loader.ts`：支持动态加载 TypeScript 扩展插件。
  - `runner.ts` / `wrapper.ts`：扩展的沙箱生命周期管理与事件钩子注入。
  - `virtual-modules.ts`：为扩展提供内置虚拟模块映射。
- **配置、模型与密钥**：
  - `model-registry.ts` / `model-resolver.ts` / `models-store.ts`：模型路由、别名映射与动态发现。
  - `auth-storage.ts` / `runtime-credentials.ts`：系统密钥环/本地加密文件凭据存储。
  - `settings-manager.ts` / `settings-schema.ts`：用户设置读写与类型校验。
  - `keybindings.ts` / `keybindings-schema.ts`：自定义快捷键配置与解析。
  - `system-prompt.ts` / `prompt-templates.ts`：系统 Prompt 构建模板与动态注入规则。
  - `skills.ts`：本地技能（Skills）的加载与解析。
  - `slash-commands.ts`：终端 Slash 命令（如 `/compact`, `/clear`, `/model`）定义与处理器。

### 1.4 `src/modes/`（输出模式解耦）
- `src/modes/interactive/`：TUI 交互模式主逻辑：
  - `interactive-mode.ts`：键盘监听、视口更新、交互循环管理。
  - `chat-viewport.ts`：聊天历史的视口滚动与差量重绘。
  - `tui-renderer.ts`：将消息与交互转换为 TUI 虚拟节点。
  - `components/`：丰富的 TUI 业务组件（`assistant-message.ts`, `diff.ts`, `model-selector.ts`, `login-dialog.ts`, `mermaid.ts` 等）。
  - `theme/`：支持自定义的主题机制与颜色语义 Token。
- `print-mode.ts`：适合管道重定向或无交互环境的纯文本流式输出模式。
- `json-event.ts`：按行输出结构化 JSON 事件流模式（便于第三方 IDE 或自动化脚本消费）。
- `src/modes/rpc/`：基于 JSON-RPC 的长连接控制模式，供远端协议调用。

### 1.5 `src/extensions/`（内置扩展模块）
- `codemode/`：将工具执行转入 QuickJS WebAssembly 沙箱的扩展。
- `mcp/`：通过配置文件连接外部 MCP 服务器并将其实时注入为工具。
- `llama/`：支持本地通过 Llama.cpp 运行的大模型 Provider。
- `tool-search/`：针对成百上千工具时的动态工具语义搜索扩展。

---

## 2. `packages/agent`

`@earendil-works/pi-agent-core` 提炼了不依赖任何 CLI 界面或具体模型的底层智能体循环与状态机抽象。

### 文件路径及功能
- `src/index.ts`：子包对外公共导出入口。
- `src/types.ts`：定义消息格式（User、Assistant、ToolCall、ToolResult）、状态机事件、Token 统计与流式事件接口。
- `src/agent-loop.ts`：**最核心的状态循环引擎**：
  - 组织 Prompt -> 触发模型请求 -> 接收流式返回 -> 提取并调用 Tools -> 收集 Tool 输出 -> 回填至历史 -> 判定是否结束或继续下一轮。
  - 严谨处理异常、中断、用户插话以及工具调用的重试机制。
- `src/agent.ts`：高层 Agent 实例，管理当前正在运行的生命周期、中止信号控制器（AbortController）及状态同步。
- `src/stream-fn.ts`：统一流式事件处理辅助函数，支持分块文本拼装与事件分发。
- `src/proxy.ts`：网络与代理配置层，支持通过企业级 HTTP/HTTPS 代理连接底层 API。

---

## 3. `packages/ai`

`@earendil-works/pi-ai` 提供了对所有主流大模型厂商的统一切换层，将各家各异的 JSON 协议抹平为标准的流式调用规范。

### 3.1 `src/api/`（底层网络请求与协议转换）
针对每个提供商的专属协议与流式响应转换器：
- `anthropic-messages.ts`：Anthropic Claude 的 Messages API 协议封装与 SSE 流解析。
- `openai-responses.ts` / `openai-completions.ts`：OpenAI 标准协议与兼容协议转换。
- `azure-openai-responses.ts`：Azure OpenAI 服务集成与版本参数适配。
- `google-generative-ai.ts` / `google-vertex.ts`：Google Gemini 及 Google Cloud Vertex AI 接入。
- `bedrock-converse-stream.ts`：AWS Bedrock ConverseStream 接口接入。
- `cloudflare-workers-ai-system-one.ts`：Cloudflare 边缘 AI 节点接入。
- `mistral-conversations.ts`：Mistral AI 接口支持。
- `constrained-sampling.ts`：结构化输出（JSON Schema）与受限采样能力实现。

### 3.2 `src/providers/`（模型提供商实现集）
覆盖了业界 30+ 种 Provider 的配置参数、端点与特征映射：
- `anthropic.ts` / `anthropic.models.ts`：Anthropic Claude 系列模型定义与参数。
- `openai.ts` / `openai.models.ts`：OpenAI GPT 系列模型。
- `deepseek.ts` / `deepseek.models.ts`：DeepSeek 原生接入与推理思考模式参数。
- `google.ts` / `google.models.ts`：Google Gemini 模型族。
- `amazon-bedrock.ts` / `amazon-bedrock.models.ts`：AWS Bedrock 托管模型。
- `openrouter.ts` / `openrouter.models.ts`：OpenRouter 聚合服务网关。
- `groq.ts`、`cerebras.ts`、`together.ts`、`fireworks.ts`：超高速推理云厂商。
- `minimax.ts`、`moonshotai.ts`、`kimi-coding.ts`、`zai.ts`、`xiaomi.ts`：国内主流大模型接入。
- `faux.ts`：测试用虚拟 Provider（不耗费真实 Token，用于单元测试自动化）。

### 3.3 `src/auth/`（认证与 OAuth 流程）
- `oauth/`：针对各大提供商的 OAuth 授权实现：
  - `callback-server.ts`：本地拉起临时 HTTP 服务接收 OAuth 回调。
  - `pkce.ts` / `device-code.ts`：PKCE 安全挑战码生成与设备码登录流。
  - 各专属提供商授权流程：`github-copilot.ts`, `anthropic.ts`, `openai-codex.ts`, `openrouter.ts` 等。
- `credential-store.ts`：凭据读写与过期刷新管理。

### 3.4 脚本与元数据生成
- `src/models.generated.ts`：**由脚本自动生成**的全量静态模型字典（包含模型 ID、上下文窗口大小、最大输出 Token、定价等）。
- `scripts/generate-models.ts`：拉取最新上游元数据并重生成 `models.generated.ts` 的离线脚本。

---

## 4. `packages/tui`

`@earendil-works/pi-tui` 是一个不依赖 React/Vue 的高性能终端差量渲染引擎，专为高频打字与流式渲染优化。

### 4.1 终端核心抽象
- `src/tui.ts` / `src/tui-main-screen.ts` / `src/tui-alt-screen.ts`：终端渲染总控，维护虚拟帧缓冲（Virtual Buffer），计算新旧两帧的 Diff 并仅向 stdout 输出变更 ANSI 序列。
- `src/terminal.ts`：底层 TTY 操作，包括 Raw Mode 设置、光标隐藏/移动、窗口尺寸监听（`SIGWINCH`）。
- `src/keys.ts` / `src/keybindings.ts`：键盘 ANSI 输入流解析，将原始字节解析为 `{ key: "enter", ctrl: true }` 等结构化对象。
- `src/layout.ts` / `src/layout-node.ts`：类 Flexbox 的轻量终端盒模型布局算法，计算宽、高、内边距与换行。
- `src/undo-stack.ts` / `src/kill-ring.ts`：文本编辑器的撤销重做栈与 Emacs 风格剪切环。

### 4.2 `src/components/`（基础 UI 组件）
- `editor.ts`：功能完善的多行代码/文本输入组件，内置光标移动、选区、行号与软换行。
- `markdown.ts`：终端 Markdown 渲染器（支持加粗、代码块、引用、列表排版）。
- `latex.ts`：数学公式终端字符排版。
- `image.ts` / `terminal-image.ts`：利用 Kitty 图形协议、iTerm2 协议或 ANSI 六色将图片直显于终端内。
- `select-list.ts` / `settings-list.ts`：交互式选项选择与开关切换列表。
- `scroll-view.ts`：带虚拟滚动条的长列表视口。
- `loader.ts` / `cancellable-loader.ts`：转轮 Spinner 动画与进度加载器。

### 4.3 `native/`（平台原生 C/Node-API 扩展）
为了提供极致的系统级体验，在跨平台剪贴板和快捷键上编写了原生 Node 插件：
- `native/win32/src/win32-platform.c`：Windows 原生剪贴板图片/文本交互及修饰键检测。
- `native/linux/src/linux-platform-x11.c`：Linux X11 剪贴板守护进程与读取器。
- `native/darwin/src/darwin-platform.m`：macOS NSPasteboard 原生集成。

---

## 5. `packages/mcp`

`@earendil-works/pi-mcp` 是一个轻量、零框架绑定的独立 Model Context Protocol 客户端。

### 文件路径及功能
- `src/client.ts`：MCP 协议客户端主入口，实现能力协商（Capabilities Negotiation）、列出工具（List Tools）、调用工具（Call Tool）、获取 Prompt 模板与订阅资源变更。
- `src/transports/`：多通道传输层抽象：
  - `transport.ts`：传输层基础接口规范。
  - `stdio.ts`：基于子进程的标准输入输出管道通信，用于拉起本地 MCP Server（如 `npx -y @modelcontextprotocol/server-everything`）。
  - `streamable-http.ts`：基于 HTTP 与 Server-Sent Events (SSE) 的长连接，连接远程 MCP 服务。
  - `in-memory.ts`：内存通信通道，供单元测试使用。
- `src/protocol/`：
  - `jsonrpc.ts`：JSON-RPC 2.0 请求编码、响应匹配与异常处理。
  - `types.ts` / `content.ts`：协议报文结构（TextContent, ImageContent, EmbeddedResource 等）。
- `src/oauth/`：连接受 OAuth 保护的远程 MCP 服务时的身份授权发现与 Token 刷新。

---

## 6. `packages/codemode`

`@earendil-works/pi-codemode` 允许智能体在一个隔离的 JavaScript 容器中执行逻辑，而**唯一能够影响外界的方式是调用注入的工具**。

### 文件路径及功能
- `src/wasm.ts`：将 QuickJS 编译为 WebAssembly 运行时的引擎封装。
- `src/source.ts` / `src/declarations.ts`：自动为注入的工具生成 TypeScript `.d.ts` 声明文件，使智能体能看懂可用工具签名。
- `src/identifier.ts`：工具标识符安全转义与重命名，防止命名污染。
- `src/runtime/`：
  - `vm.ts` / `execution.ts`：QuickJS 虚拟机实例创建，设置运行内存上限与 CPU 周期配额。
  - `worker.ts` / `inline.ts`：在独立的 Worker 线程中执行用户代码，确保死循环代码不会卡死主进程。
  - `protocol.ts`：Host 主进程与沙箱之间的请求/响应转发协议。

---

## 7. `packages/durable`

`@earendil-works/pi-durable` 实现了长期会话的事件溯源（Event Sourcing）、异步任务调度与上下文存储机制。

### 文件路径及功能
- `src/session/`：
  - `session.ts` / `transaction.ts`：基于事务的会话状态更新，保障意外断电或崩溃时状态无损。
  - `forks.ts`：支持从会话历史的任意检查点切出新分支并行探索。
  - `observation.ts`：智能体观察结果持久化。
- `src/storage/`：
  - `jsonl/`：轻量级单行 JSON 追加存储后端（适用于本地快速读取）。
  - `sqlite/`：基于 SQLite 的结构化关系型存储（提供全文索引、快速切片与大容量持久化）。
- `src/harness/`：
  - `harness.ts` / `scheduler.ts`：长任务生命周期调度器。
  - `task-graph.ts`：任务依赖有向无环图（DAG）构建与执行。
- `src/tool-search/`：
  - `tool-search.ts`：为大规模扩展包环境提供的工具动态检索与延迟注入能力。
- `src/images/`：对会话中交互图片的压缩、缩放（通过 Photon Wasm）、EXIF 方向矫正与离线缓存。

---

## 8. `packages/env`

`@earendil-works/pi-env` 构成了 Pi 的远程执行环境系统。允许本地 CLI 无缝连接到云端虚拟机、容器或开发机上执行命令与管理文件。

### 文件路径及功能
- `daemon/`（Rust 编写的远端轻量守护进程）：
  - `Cargo.toml` / `src/main.rs`：无第三方繁重依赖的纯静态编译二进制。
  - `src/sys/`：封装 Linux 下的 PTY（虚拟终端伪设备）、进程派生、信号传递与文件监听。
- `src/`（Node.js 端客户端接口）：
  - `ssh.ts`：建立 SSH 隧道，负责自动将预编译的守护进程推送到远端服务器并拉起。
  - `remote-env.ts`：在远程机器上抽象出同等语义的 `read`、`write`、`bash` 环境。
  - `list-files.ts` / `watch.ts`：远程文件系统遍历与增量文件变动监听。

---

## 9. `packages/telemetry`

`@earendil-works/pi-telemetry` 提供了统一且零强制依赖的监控度量接口。

### 文件路径及功能
- `src/index.ts`：公共 API 导出。
- `src/metrics.ts`：指标定义与收集器（计数器 Counter、直方图 Histogram、仪表盘 Gauge）。
- `src/memory.ts`：将度量指标保存在内存缓冲中，供退出时导出或调试诊断使用。
- `src/noop.ts`：空实现（No-op Provider），在未开启遥测时保证零运行时开销。

---

## 10. `packages/chord`

`@earendil-works/chord` 提供了基于强类型契约的微宿主系统，主要用于复杂扩展的跨进程通信、多客户端实时协作与状态副本差量同步。

### 文件路径及功能
- `src/host.ts` / `src/cluster.ts`：宿主生命周期管理与多节点集群发现。
- `src/contract.ts` / `src/declare.ts`：使用 TypeScript 定义跨进程服务契约接口。
- `src/delta/`：
  - `diff.ts` / `apply-immutable-trusted.ts`：结构化状态对象的不可变差量（Delta）计算与应用。
  - `tracker.ts` / `revision-validator.ts`：版本追溯与并发冲突校验。
- `src/sqlite/`：集成 SQLite 作为状态历史存储介质。

---

## 11. `packages/evals`

`@earendil-works/pi-evals` 承担了 Agent 代码能力、文档准确性与 Provider 稳定性的自动化回归评测。

### 文件路径及功能
- `src/harness.ts`：自动化测试基座，拉起隔离沙箱执行评测任务。
- `src/plan.ts` / `src/report.ts`：评测矩阵计划规划与多维度综合打分报告生成器。
- `evals/`：具体评测任务用例：
  - `smoke.eval.ts`：基础连通性冒烟测试。
  - `models.docs.eval.ts` / `extensions.docs.eval.ts`：文档与代码一致性自动验证。
  - `custom-provider.docs.eval.ts`：验证用户自定义 Provider 流程的正确性。
- `docker/`：用于在干净 Docker 容器中执行评测的镜像定义。

---

## 总结

各子包之间职责边界清晰、依赖层次严格递进：
- **`ai`** 专注模型通信；
- **`tui`** 专注终端渲染；
- **`codemode`** 与 **`mcp`** 专注工具执行安全与协议扩展；
- **`agent`** 专注智能体反馈循环；
- **`coding-agent`** 将上述底层包有机组装为功能强大的现代化开发者工具。
