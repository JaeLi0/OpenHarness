<div align="center">

# 🔧 OpenHarness

**轻量级 Agent Harness 基础设施 — 为 LLM 提供手、眼、记忆和安全边界**

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](./LICENSE)
[![Version](https://img.shields.io/badge/Version-v0.1.7-orange)](./CHANGELOG.md)
[![Pydantic](https://img.shields.io/badge/Pydantic-v2-red?logo=pydantic)](https://docs.pydantic.dev/)
[![MCP](https://img.shields.io/badge/Protocol-MCP-purple)](https://modelcontextprotocol.io/)

*"模型是 Agent，代码是 Harness"*

[English](README.md) · [快速开始](#-快速开始) · [架构图](#-系统架构) · [我做了什么](#-我在原项目上做的改进)

</div>

![OpenHarness in action](docs/screenshot.png)

---

## 📖 项目简介

OpenHarness 是 [Claude Code](https://github.com/anthropics/claude-code) 的开源 Python 移植版，提供完整的 **Agent Harness 基础设施**，让任何 LLM 都能像 Claude Code 一样拥有工具调用能力、持久记忆、权限治理和多 Agent 协调。

本仓库是在 [novix-science/OpenHarness](https://github.com/HKUDS/OpenHarness) 基础上的改进版，修复了原项目中的 4 个进程挂死 Bug、2 个压缩边界问题、权限并发竞态，并新增对 10+ OpenAI 兼容 Provider 的支持。

---

## ✨ 核心功能

| 功能模块 | 描述 |
|---------|------|
| 🔄 **Agent Loop** | 流式工具调用循环，并行执行、指数退避重试、Token 计数与成本跟踪 |
| 🛠️ **43+ 工具** | 文件 I/O、Shell、搜索、Web、MCP 协议全覆盖，每个工具有 Pydantic 输入校验 |
| 🧠 **Memory 系统** | MEMORY.md 持久记忆，YAML frontmatter 结构化，支持中文关键词搜索 |
| 📦 **Skills 系统** | Markdown 知识按需加载，兼容 anthropics/skills 生态 |
| 🔌 **Plugin 系统** | 命令 + Hook + Agent 三合一插件架构，兼容 claude-code 插件 |
| 🔒 **权限治理** | 三级模式（default/auto/plan）、路径规则、命令拒绝列表、Pre/PostToolUse Hook |
| 🤖 **Swarm 协调** | 子 Agent 生成、后台任务生命周期、通知系统 |
| 🖥️ **React TUI** | 基于 React/Ink 的终端 UI，Markdown 渲染、权限对话框、命令选择器 |
| 🌐 **多 Provider** | Anthropic / DeepSeek / Qwen / Gemini / Groq / Ollama 等 10+ 后端 |
| 💾 **Auto-Compact** | 三层上下文压缩（无 LLM → 结构化摘要 → 全量 LLM），支持多天不间断会话 |

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                     用户入口层                            │
│   CLI (oh / openh)    React TUI    ohmo Personal Agent  │
│   Feishu / Slack / Telegram / Discord Channels          │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                    Runtime Bundle                        │
│        配置解析 → 认证 → Provider → 会话管理             │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  QueryEngine（Agent Loop）               │
│  ┌──────────────────────────────────────────────────┐   │
│  │  while True:                                     │   │
│  │    ① Auto-Compact 检查（三层压缩）                │   │
│  │    ② API 流式调用 → 文本增量事件                  │   │
│  │    ③ stop_reason == "tool_use"?                  │   │
│  │       ├─ Yes → 权限检查 → PreHook → 执行 →       │   │
│  │       │         PostHook → 结果注入 → 继续循环    │   │
│  │       └─ No  → Stop Hook → 结束                  │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Tool      │  │ Permission  │  │    Hook     │
│  Registry   │  │  Checker    │  │  Executor   │
│  43+ Tools  │  │ 三级模式/路径│  │  Pre/Post   │
└─────────────┘  └─────────────┘  └─────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                     工具执行层                            │
│  File I/O  │  Shell  │  Search  │  Web  │  Agent  │ MCP │
└─────────────────────────────────────────────────────────┘
```

### Auto-Compact 三层压缩策略

```
Token 超阈值（默认 167K / 200K 窗口）
  │
  ├─ 第一层：Microcompact（零 LLM 调用）
  │   清除旧工具结果 → [Old tool result content cleared]
  │   保留最近 5 条工具结果
  │
  ├─ 第二层：Session Memory（零 LLM 调用）
  │   将旧消息压缩为结构化文本摘要
  │   保留最近 12 条消息
  │
  └─ 第三层：Full LLM Compact（调用 LLM）
      生成 9 维度 <summary>（请求意图/技术概念/文件/错误/待办...）
      保留最近 6 条消息 + 结构化附件
```

---

## 🚀 我在原项目上做的改进

本仓库在原版 OpenHarness 基础上做了 **5 项针对性修复和扩展**：

### 1. 🔧 工具执行：修复 4 个进程挂死 Bug

**问题：**
- `grep`/`glob` 工具通过 `rg` 子进程执行时，`stderr=PIPE` 在大型仓库中会导致 OS 管道缓冲区（~64KB）被填满，子进程的 `write()` 永久阻塞，Agent Loop 卡死
- `bash` 工具超时后未关闭 `stdout` 流，导致资源泄漏
- `session_runner` 后台任务从不读取 `stderr` 管道，子进程写满缓冲区后死锁

**修复：** 将 `grep`/`glob` 的 `stderr` 重定向到 `DEVNULL`；给 `bash` 工具超时后的 `stdout` 加 `asyncio.wait_for(2s)` drain；`session_runner` 改为 `stderr=STDOUT` 合并输出流。

**结果：** 4 个挂死根因全部消除，大型仓库下 `grep`/`glob` 稳定运行。

---

### 2. 📉 上下文压缩：边界处理与视觉 Token 估算

**问题：** 超过 16K 字符的大型工具结果未被标记为可压缩，在压缩边界触发 Token 溢出；多模态对话中图片 Block 无 Token 估算，导致实际用量严重低估，频繁触发 `context_length_exceeded` API 错误。

**修复：** 确保超大工具输出写入 `tool_artifacts/` 并标记为 micro-compactable；旧 MCP 结果加入 `context_collapse` 剪枝链；新增每张图片 Token 估算（默认 3072 tokens/图，可通过 `OPENHARNESS_IMAGE_TOKEN_ESTIMATE` 配置）。

**结果：** 多模态长会话不再因 Token 低估被 API 拒绝；压缩边界正确处理大型工具输出。

---

### 3. 🔒 权限系统：序列化并发确认对话框

**问题：** LLM 在一轮返回多个 `tool_use` 时，所有工具通过 `asyncio.gather` 并发调度。若多个工具需要用户授权，`_ask_permission` 被并发调用，权限对话框互相覆盖，用户确认响应被静默丢弃。

**修复：** 在 `_ask_permission` 中引入 `asyncio.Lock`，将并发权限请求串行化，每次只有一个对话框激活，后续请求在 Lock 后排队。

**结果：** 并发工具执行场景下零丢失权限响应，每次工具调用获得独立、明确的确认交互。

---

### 4. 🧠 记忆系统：YAML 解析与中文搜索

**问题：** 记忆文件的 YAML frontmatter 通过手动分割 `---` 解析，在 block scalar（`>`、`|`）和带引号的值上解析出错；记忆搜索只匹配元数据字段（name/description/type），正文内容不可被搜索；中文查询因分词器只处理 ASCII 而无结果。

**修复：** 改用 `yaml.safe_load()` 完整解析 YAML spec；搜索评分扩展到正文内容（元数据权重 2×，正文权重 1×）；`_tokenize()` 中新增 Han 字符识别（`一-鿿`），每个 CJK 字符作为独立搜索 token。

**结果：** 复杂 YAML 结构的记忆文件正确解析；中文查询产生有意义的搜索结果；正文搜索覆盖率显著提升。

---

### 5. 🌐 Provider 层：OpenAI 兼容后端支持

**问题：** 原版只支持 Anthropic 和少数固定 Provider，使用 DeepSeek、Qwen、Groq、Ollama 等 OpenAI 兼容端点的开发者没有接入路径。

**修复：** 实现完整的 `OpenAICompatibleClient`，将 OpenAI 流式事件映射到 OpenHarness 内部 `StreamEvent` 类型；处理 Provider 特有的 `reasoning_content` 字段（DeepSeek R1 等），发出 `ThinkingDeltaEvent`，保持前端 Provider 无关；内置 Qwen（DashScope）、Gemini、MiniMax 等 Provider 配置。

**结果：** 10+ OpenAI 兼容后端开箱即用（DeepSeek、DashScope、GitHub Models、Groq、Ollama 等），切换 Provider 只需 `oh provider use <name>`，无需改代码。

---

## 🛠️ 技术栈

| 类别 | 技术 |
|------|------|
| **运行时** | Python 3.10+ / asyncio |
| **类型安全** | Pydantic v2 |
| **LLM 客户端** | Anthropic SDK + OpenAI SDK |
| **协议** | MCP（Model Context Protocol） |
| **前端** | React 18 / Ink 5（终端 UI） |
| **包管理** | uv（推荐）|
| **会话存储** | JSON 文件快照 |
| **记忆存储** | YAML + 关键词搜索（无向量数据库） |
| **Node.js** | 18+（React TUI 需要） |

---

## ⚡ 快速开始

### 环境要求

- Python 3.10+
- Node.js 18+
- [uv](https://github.com/astral-sh/uv)

### 安装

```bash
git clone <this-repo>
cd OpenHarness
uv sync --extra dev

# 初始化前端
cd frontend/terminal && npm ci && cd ../..
```

### 配置 Provider

```bash
# 示例：使用 DeepSeek
export OPENAI_API_KEY="your-key"
oh provider add deepseek \
  --label "DeepSeek" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model deepseek-v4-flash \
  --base-url https://api.deepseek.com
oh provider use deepseek

# 示例：使用 Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
oh provider add claude \
  --label "Claude" \
  --provider anthropic \
  --api-format anthropic \
  --auth-source anthropic_api_key \
  --model claude-sonnet-4-6 \
  --base-url https://api.anthropic.com
oh provider use claude

# 或使用交互式向导
oh setup
```

### 启动

```bash
# 交互式 TUI
oh

# 单次提问
oh -p "Explain this codebase"

# JSON 输出
oh -p "List all functions" --output-format json

# 安全预览（不调用模型）
oh --dry-run -p "Review this change"

# 恢复上次会话
oh --continue

# Windows PowerShell（避免与 Out-Host 别名冲突）
openh
```

---

## 📁 项目结构

```
src/openharness/
├── cli.py                    # CLI 入口（Typer）
├── engine/
│   ├── query.py              # Agent Loop 核心：run_query() + _execute_tool_call()
│   ├── query_engine.py       # QueryEngine 外观类
│   ├── messages.py           # ConversationMessage 数据模型
│   └── stream_events.py      # 流式事件类型定义
├── tools/                    # 43+ 工具（File/Shell/Search/Web/Agent/MCP）
├── services/
│   ├── compact/              # 三层 Auto-Compact 系统（~1700 行）
│   └── session_storage.py    # 会话 JSON 持久化
├── permissions/              # 权限治理（三级模式 + 路径规则）
├── hooks/                    # 生命周期 Hook（Pre/PostToolUse/Stop/Compact）
├── memory/                   # 持久记忆系统（YAML + 关键词搜索）
├── skills/                   # Skills 注册与加载
└── providers/                # LLM Provider 适配层（Anthropic/OpenAI Compatible）
```

---

## 🔒 安全说明

- API 密钥通过 `keyring` / 环境变量 / 配置文件分层存储，优先使用系统密钥链
- 敏感路径（`/etc/`、`~/.ssh/`）内置权限保护，不可绕过
- `web_fetch` URL 含注入校验，拒绝 `file://` / `localhost` 内部服务请求
- 命令 Hook 中 `$ARGUMENTS` 进行 shell 转义防注入
- Docker sandbox 提供网络隔离和资源限制（可选）

---

## 📄 License

MIT © 2024

Built on top of [OpenHarness](https://github.com/HKUDS/OpenHarness) by novix-science, licensed under MIT.

---

<div align="center">
<sub>如果这个项目对你有帮助，欢迎点个 ⭐</sub>
</div>
