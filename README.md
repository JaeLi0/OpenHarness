# OpenHarness

Extended version of [OpenHarness](https://github.com/HKUDS/OpenHarness) — an open-source Python port of Claude Code, providing a full Agent Harness infrastructure (tools, memory, permissions, multi-agent coordination) for LLMs. Original project by novix-science.

![OpenHarness in action]
<img width="760" height="242" alt="image" src="https://github.com/user-attachments/assets/2ee0a32a-5b26-4180-bb47-d1ce33e84ee9" />

<!-- Replace docs/screenshot.png with your actual screenshot path -->

## What I Built On Top

The original OpenHarness had several known stability issues in its tool execution pipeline, context compression system, and memory search — and lacked support for non-Anthropic LLM backends. I made five targeted improvements to address these:

---

## Improvements

### 1. Tool Execution: Fixing Four Hang and Deadlock Bugs

**Problem:** Three tools had process-level hang bugs that could permanently block the Agent Loop:
- `grep`/`glob` tools spawned `rg` subprocesses with `stderr=PIPE`. When `rg` produced enough stderr output to fill the OS pipe buffer (~64KB), the subprocess's `write()` call blocked indefinitely — freezing the entire agent.
- `bash` tool left `stdout` streams open after timeout, causing resource leaks.
- `session_runner` background task never read its own `stderr` pipe, causing a deadlock when the subprocess wrote enough output to fill the buffer.

**Approach:** Redirected `stderr` to `DEVNULL` in `grep`/`glob` to discard unneeded diagnostic output; added `asyncio.wait_for(2s)` drain on `bash` tool's `stdout` after timeout; merged `stderr` into `stdout` via `stderr=STDOUT` in `session_runner` to prevent buffer starvation.

**Result:** All four hang/deadlock root causes eliminated. `grep`/`glob` no longer freeze on large repositories with verbose `rg` output; `bash` timeouts clean up correctly; `session_runner` background tasks remain stable under load.

---

### 2. Context Compression: Boundary Handling and Vision Token Estimation

**Problem:** The three-layer compression system (Microcompact → Session Memory → Full LLM Compact) had two edge case gaps:
- Large tool results (>16K chars) were not consistently flagged as compressible, causing token overflow at compression boundaries.
- Image blocks in multi-modal conversations had no token estimate, causing severe under-counting and triggering unexpected `context_length_exceeded` API errors.

**Approach:** Ensured oversized tool outputs are written to `tool_artifacts/` files and marked as micro-compactable; added old MCP results to the `context_collapse` pruning chain. Added per-image token estimation (default 3072 tokens/image, configurable via `OPENHARNESS_IMAGE_TOKEN_ESTIMATE`) so the auto-compact threshold calculation accounts for visual context.

**Result:** Multi-modal long sessions no longer trigger API rejection errors due to token under-estimation; compression boundaries handle large tool outputs correctly.

---

### 3. Permission System: Serializing Concurrent Confirmation Dialogs

**Problem:** When the LLM returned multiple `tool_use` blocks in one turn, all tools were dispatched concurrently via `asyncio.gather`. If multiple tools required user permission, `_ask_permission` was called in parallel — causing confirmation dialogs to overlap and user responses to be silently dropped.

**Approach:** Introduced an `asyncio.Lock` in `_ask_permission` to serialize permission prompts. Only one dialog is active at any time; subsequent requests queue behind the lock and are presented individually after the previous one resolves.

**Result:** Zero dropped permission responses in concurrent tool execution scenarios. Each tool use gets an independent, unambiguous confirmation interaction.

---

### 4. Memory System: YAML Parsing and Chinese Search

**Problem:** The memory system had two limitations:
- YAML frontmatter in memory files was parsed by manually splitting on `---` delimiters and iterating line-by-line. This broke on standard YAML features like block scalars (`>`, `|`) and quoted values.
- Memory search only matched against metadata fields (name, description, type). The actual body content of memory files was invisible to search, significantly limiting recall. Chinese queries returned no results because the tokenizer only handled ASCII words.

**Approach:** Replaced the manual parser with `yaml.safe_load()` for full YAML spec compliance. Extended the search scorer to include body content (metadata match weight 2×, body match weight 1×). Added Han character recognition (`一-鿿`) in `_tokenize()` so each CJK character is treated as an independent search token.

**Result:** Memory files with complex YAML structures are parsed correctly. Chinese queries now produce meaningful results; body-text search expands recall coverage beyond metadata-only matching.

---

### 5. Provider Layer: OpenAI-Compatible Backend Support

**Problem:** The original implementation only supported Anthropic and a few fixed providers. Developers using DeepSeek, Qwen, Groq, Ollama, or any other OpenAI-compatible endpoint had no path to integration.

**Approach:** Implemented a full `OpenAICompatibleClient` that maps OpenAI streaming events to OpenHarness's internal `StreamEvent` types. Added handling for provider-specific `reasoning_content` fields (used by DeepSeek R1 and similar models) to emit `ThinkingDeltaEvent`, keeping the frontend provider-agnostic. Added built-in provider profiles for Qwen (DashScope), Gemini, and MiniMax.

**Result:** 10+ OpenAI-compatible backends now supported out of the box (DeepSeek, DashScope, GitHub Models, Groq, Ollama, and others). Switching providers requires only `oh provider use <name>` — no code changes.

---

## Tech Stack

- **Runtime:** Python 3.10+ / asyncio
- **Type safety:** Pydantic v2
- **LLM clients:** Anthropic SDK + OpenAI SDK
- **Protocol:** MCP (Model Context Protocol)
- **Frontend:** React 18 / Ink 5 (terminal UI)
- **Session storage:** JSON file snapshots
- **Memory:** YAML + keyword search (no vector DB)

## How to Run

**Requirements:** Python 3.10+, Node.js 18+, [uv](https://github.com/astral-sh/uv)

```bash
git clone <this-repo>
cd OpenHarness
uv sync --extra dev

cd frontend/terminal && npm ci && cd ../..

# Configure a provider (example: DeepSeek)
export OPENAI_API_KEY="your-key"
oh provider add deepseek \
  --label "DeepSeek" \
  --provider openai \
  --api-format openai \
  --auth-source openai_api_key \
  --model deepseek-v4-flash \
  --base-url https://api.deepseek.com
oh provider use deepseek

# Launch
oh
```

## Original Project

This project is built on top of [OpenHarness](https://github.com/HKUDS/OpenHarness) by novix-science, licensed under MIT.

## License

MIT
