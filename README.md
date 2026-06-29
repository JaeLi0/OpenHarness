<div align="center">

# 🔧 OpenHarness

**Lightweight Agent Harness Infrastructure — giving LLMs hands, eyes, memory, and safety boundaries**

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-MIT-green)](./LICENSE)
[![Version](https://img.shields.io/badge/Version-v0.1.7-orange)](./CHANGELOG.md)
[![Pydantic](https://img.shields.io/badge/Pydantic-v2-red?logo=pydantic)](https://docs.pydantic.dev/)
[![MCP](https://img.shields.io/badge/Protocol-MCP-purple)](https://modelcontextprotocol.io/)

*"The model is the Agent. The code is the Harness."*

[中文](README.zh-CN.md) · [Quick Start](#-quick-start) · [Architecture](#-architecture) · [What I Built On Top](#-what-i-built-on-top)

</div>

---

## 📖 Overview

OpenHarness is an open-source Python port of [Claude Code](https://github.com/anthropics/claude-code), providing a complete **Agent Harness infrastructure** so any LLM can have tool-calling, persistent memory, permission governance, and multi-agent coordination — just like Claude Code.

This repository is an improved fork of [novix-science/OpenHarness](https://github.com/HKUDS/OpenHarness), fixing 4 process hang bugs, 2 compression boundary issues, and a permission concurrency race, while adding support for 10+ OpenAI-compatible providers.

---

## ✨ Features

| Module | Description |
|--------|-------------|
| 🔄 **Agent Loop** | Streaming tool-call loop with parallel execution, exponential backoff retry, token counting and cost tracking |
| 🛠️ **43+ Tools** | Full coverage of File I/O, Shell, Search, Web, and MCP protocol — every tool has Pydantic input validation |
| 🧠 **Memory System** | Persistent memory via MEMORY.md, YAML frontmatter structured storage, Chinese keyword search support |
| 📦 **Skills System** | On-demand Markdown knowledge loading, compatible with the anthropics/skills ecosystem |
| 🔌 **Plugin System** | Command + Hook + Agent three-in-one plugin architecture, compatible with claude-code plugins |
| 🔒 **Permission Governance** | Three-level modes (default/auto/plan), path rules, command deny lists, Pre/PostToolUse Hooks |
| 🤖 **Swarm Coordination** | Sub-agent spawning, background task lifecycle management, notification system |
| 🖥️ **React TUI** | React/Ink-based terminal UI with Markdown rendering, permission dialogs, and command selector |
| 🌐 **Multi-Provider** | Anthropic / DeepSeek / Qwen / Gemini / Groq / Ollama and 10+ more backends |
| 💾 **Auto-Compact** | Three-layer context compression (no-LLM → structured summary → full LLM), enabling multi-day uninterrupted sessions |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Entry Layer                        │
│   CLI (oh / openh)    React TUI    ohmo Personal Agent  │
│   Feishu / Slack / Telegram / Discord Channels          │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                    Runtime Bundle                        │
│     Config parsing → Auth → Provider → Session mgmt     │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  QueryEngine (Agent Loop)                │
│  ┌──────────────────────────────────────────────────┐   │
│  │  while True:                                     │   │
│  │    ① Auto-Compact check (3-layer compression)   │   │
│  │    ② Streaming API call → text delta events     │   │
│  │    ③ stop_reason == "tool_use"?                  │   │
│  │       ├─ Yes → Permission check → PreHook →     │   │
│  │       │         Execute → PostHook → Inject →   │   │
│  │       │         continue loop                   │   │
│  │       └─ No  → Stop Hook → exit                 │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────┬──────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Tool      │  │ Permission  │  │    Hook     │
│  Registry   │  │  Checker    │  │  Executor   │
│  43+ Tools  │  │ mode/paths  │  │  Pre/Post   │
└─────────────┘  └─────────────┘  └─────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                    Tool Execution Layer                  │
│  File I/O  │  Shell  │  Search  │  Web  │  Agent  │ MCP │
└─────────────────────────────────────────────────────────┘
```

### Auto-Compact: Three-Layer Compression Strategy

```
Token count exceeds threshold (default 167K / 200K window)
  │
  ├─ Layer 1: Microcompact (zero LLM calls)
  │   Clears old tool results → [Old tool result content cleared]
  │   Keeps the 5 most recent tool results
  │
  ├─ Layer 2: Session Memory (zero LLM calls)
  │   Compresses old messages into structured text summaries
  │   Keeps the 12 most recent messages
  │
  └─ Layer 3: Full LLM Compact (calls LLM)
      Generates a 9-dimension <summary>
      (intent / concepts / files / errors / pending tasks ...)
      Keeps 6 most recent messages + structured attachments
```

---

## 🚀 What I Built On Top

This repo makes **5 targeted fixes and extensions** on top of the original OpenHarness:

### 1. 🔧 Tool Execution: Fixing Four Hang and Deadlock Bugs

**Problem:**
- `grep`/`glob` spawned `rg` subprocesses with `stderr=PIPE`. On large repos, `rg`'s stderr output filled the OS pipe buffer (~64KB), causing the subprocess `write()` to block indefinitely — freezing the Agent Loop
- `bash` tool left `stdout` streams open after timeout, causing resource leaks
- `session_runner` background tasks never read their own `stderr` pipe, causing deadlock when output filled the buffer

**Fix:** Redirected `grep`/`glob` `stderr` to `DEVNULL`; added `asyncio.wait_for(2s)` drain on `bash` tool `stdout` after timeout; merged `stderr` into `stdout` via `stderr=STDOUT` in `session_runner`.

**Result:** All four hang/deadlock root causes eliminated. `grep`/`glob` run stably on large repos; `bash` timeouts clean up correctly; `session_runner` background tasks remain stable under load.

---

### 2. 📉 Context Compression: Boundary Handling and Vision Token Estimation

**Problem:** Large tool results (>16K chars) were not consistently flagged as compressible, causing token overflow at compression boundaries. Image blocks in multi-modal conversations had no token estimate, causing severe under-counting and unexpected `context_length_exceeded` API errors.

**Fix:** Ensured oversized tool outputs are written to `tool_artifacts/` and marked as micro-compactable; added old MCP results to the `context_collapse` pruning chain; added per-image token estimation (default 3072 tokens/image, configurable via `OPENHARNESS_IMAGE_TOKEN_ESTIMATE`).

**Result:** Multi-modal long sessions no longer trigger API rejection due to token under-estimation; compression boundaries handle large tool outputs correctly.

---

### 3. 🔒 Permission System: Serializing Concurrent Confirmation Dialogs

**Problem:** When the LLM returned multiple `tool_use` blocks in one turn, all tools were dispatched concurrently via `asyncio.gather`. If multiple tools required user confirmation, `_ask_permission` was called in parallel — dialogs overlapped and user responses were silently dropped.

**Fix:** Introduced an `asyncio.Lock` in `_ask_permission` to serialize permission prompts. Only one dialog is active at a time; subsequent requests queue behind the lock and are presented individually.

**Result:** Zero dropped permission responses in concurrent tool execution scenarios. Each tool use gets an independent, unambiguous confirmation interaction.

---

### 4. 🧠 Memory System: YAML Parsing and Chinese Search

**Problem:** YAML frontmatter in memory files was parsed by manually splitting on `---` delimiters, breaking on block scalars (`>`, `|`) and quoted values. Memory search only matched metadata fields (name/description/type) — body content was invisible to search. Chinese queries returned no results because the tokenizer only handled ASCII.

**Fix:** Replaced manual parser with `yaml.safe_load()` for full YAML spec compliance. Extended the search scorer to include body content (metadata match weight 2×, body match weight 1×). Added Han character recognition (`一-鿿`) in `_tokenize()` so each CJK character is treated as an independent search token.

**Result:** Memory files with complex YAML structures parse correctly. Chinese queries now produce meaningful results; body-text search significantly expands recall coverage.

---

### 5. 🌐 Provider Layer: OpenAI-Compatible Backend Support

**Problem:** The original implementation only supported Anthropic and a few fixed providers. Developers using DeepSeek, Qwen, Groq, Ollama, or any other OpenAI-compatible endpoint had no integration path.

**Fix:** Implemented a full `OpenAICompatibleClient` mapping OpenAI streaming events to OpenHarness's internal `StreamEvent` types. Added handling for provider-specific `reasoning_content` fields (DeepSeek R1 and similar) to emit `ThinkingDeltaEvent`, keeping the frontend provider-agnostic. Added built-in profiles for Qwen (DashScope), Gemini, and MiniMax.

**Result:** 10+ OpenAI-compatible backends supported out of the box (DeepSeek, DashScope, GitHub Models, Groq, Ollama, and others). Switching providers requires only `oh provider use <name>` — no code changes.

---

## 🛠️ Tech Stack

| Category | Technology |
|----------|------------|
| **Runtime** | Python 3.10+ / asyncio |
| **Type Safety** | Pydantic v2 |
| **LLM Clients** | Anthropic SDK + OpenAI SDK |
| **Protocol** | MCP (Model Context Protocol) |
| **Frontend** | React 18 / Ink 5 (terminal UI) |
| **Package Manager** | uv (recommended) |
| **Session Storage** | JSON file snapshots |
| **Memory Storage** | YAML + keyword search (no vector DB) |
| **Node.js** | 18+ (required for React TUI) |

---

## ⚡ Quick Start

### Requirements

- Python 3.10+
- Node.js 18+
- [uv](https://github.com/astral-sh/uv)

### Install

```bash
git clone <this-repo>
cd OpenHarness
uv sync --extra dev

# Initialize the frontend
cd frontend/terminal && npm ci && cd ../..
```

### Configure a Provider

```bash
# Example: DeepSeek
export OPENAI_API_KEY="your-key"
oh provider add deepseek \
  --label "DeepSeek" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model deepseek-v4-flash \
  --base-url https://api.deepseek.com
oh provider use deepseek

# Example: Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
oh provider add claude \
  --label "Claude" \
  --provider anthropic \
  --api-format anthropic \
  --auth-source anthropic_api_key \
  --model claude-sonnet-4-6 \
  --base-url https://api.anthropic.com
oh provider use claude

# Or use the interactive wizard
oh setup
```

### Launch

```bash
# Interactive TUI
oh

# Single prompt
oh -p "Explain this codebase"

# JSON output
oh -p "List all functions" --output-format json

# Dry run (no model calls, no tool execution)
oh --dry-run -p "Review this change"

# Resume last session
oh --continue

# Windows PowerShell (avoids conflict with Out-Host alias)
openh
```

---

## 📁 Project Structure

```
src/openharness/
├── cli.py                    # CLI entry point (Typer)
├── engine/
│   ├── query.py              # Agent Loop core: run_query() + _execute_tool_call()
│   ├── query_engine.py       # QueryEngine facade
│   ├── messages.py           # ConversationMessage data model
│   └── stream_events.py      # Streaming event type definitions
├── tools/                    # 43+ tools (File/Shell/Search/Web/Agent/MCP)
├── services/
│   ├── compact/              # Three-layer Auto-Compact system (~1700 lines)
│   └── session_storage.py    # Session JSON persistence
├── permissions/              # Permission governance (three-level modes + path rules)
├── hooks/                    # Lifecycle hooks (Pre/PostToolUse/Stop/Compact)
├── memory/                   # Persistent memory system (YAML + keyword search)
├── skills/                   # Skills registry and loader
└── providers/                # LLM provider adapter layer (Anthropic/OpenAI-compatible)
```

---

## 🔒 Security

- API keys stored via `keyring` / env vars / config files with system keychain priority
- Sensitive paths (`/etc/`, `~/.ssh/`) are hard-blocked and cannot be bypassed
- `web_fetch` validates URLs and rejects `file://` / `localhost` requests
- `$ARGUMENTS` in command hooks is shell-escaped to prevent injection
- Optional Docker sandbox provides network isolation and resource limits

---

## 📄 License

MIT © 2024

Built on top of [OpenHarness](https://github.com/HKUDS/OpenHarness) by novix-science, licensed under MIT.

---

<div align="center">
<sub>If this project helped you, consider giving it a ⭐</sub>
</div>
