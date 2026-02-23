# PROJECT_CHANGELOG.md

## Project Goal

Build high-quality Go templates for LLMs targeted at Ollama, with full support for thinking/reasoning and tool calling where applicable. Templates are stored per-model and documented with design rationale.

---

## 2026-02-23

**User Request:** Investigate tool call parsing failures on Nemotron-3-Nano when used with nanobot ("I've completed processing but have no response to give."). Research root cause, evaluate proposed debug report fixes, and implement a reliable remediation.

**Changes Applied:**
- Conducted deep analysis of Ollama’s tool call parser architecture (PRs #10415, #11030, #13764). Identified two separate, incompatible parsing pipelines (template-based vs architecture-based) and documented them in `TEMPLATE-NOTES.md` §*Ollama Parser Architecture*.
- Created `temp-debug-report.md` with full analysis evaluation and remediation plan (Path A).
- Implemented Path A (Hybrid Template) across `Nemotron-3-Nano-thinking.gotmpl` and `Modelfile`:
  - **XML tool definitions**: replaced JSON definitions with `<tool><name>...</name><description>...</description><parameters>...</parameters></tool>` structure
  - **Strengthened output instructions**: added explicit negative example prohibiting XML parameter format reversion
  - **Fixed `else if .ToolCalls`**: changed to independent `if` blocks so both content and tool calls render in assistant history
  - **`$thinking` guard**: `<think></think>` injection on history turns now suppressed when thinking is disabled
  - **Modelfile sampling comment**: added guidance for tool-calling workloads (temperature=0.6, top_p=0.95)
- Updated `TEMPLATE-NOTES.md` changelog and Known Limitations.
- Updated `DOCLINKS.md` and `SCRATCHPAD.md` with new sources and architectural notes.

---

## 2025-07-19

**User Request:** Create a Go template for NVIDIA Nemotron 3 Nano (30B-A3B) for Ollama. Thinking/reasoning enabled by default. Tool calling must be implemented in a way that Ollama can parse correctly.

**Changes Applied:**
- Researched Nemotron 3 Nano token format (ChatML), token IDs, canonical Jinja template, and NVIDIA model card.
- Investigated Ollama's tool call parsing by reviewing source code (template.go, runner.go). Confirmed JSON-format tool calls are required for Ollama parser compatibility.
- Created `Templates-Per-Model/Nemotron-3-Nano/Nemotron-3-Nano-thinking.gotmpl` — full Go template with:
  - Thinking ON by default, controllable via `--think`/`--nothink`
  - Multi-turn thinking truncation (two-pass approach with `$lastUserIdx`)
  - JSON-format tool calling (Qwen3 pattern) for Ollama compatibility
  - Dual-path support (Messages API + Legacy `/api/generate`)
- Created `Templates-Per-Model/Nemotron-3-Nano/Modelfile` — complete Ollama Modelfile with embedded template and NVIDIA-recommended sampling parameters.
- Created `Templates-Per-Model/Nemotron-3-Nano/Nemotron-3-Nano-TEMPLATE-NOTES.md` — detailed design notes, token reference, known limitations, and changelog.
- Initialized project documentation files (PROJECT_CHANGELOG.md, SCRATCHPAD.md, DOCLINKS.md).
