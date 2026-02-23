# PROJECT_CHANGELOG.md

## Project Goal

Build high-quality Go templates for LLMs targeted at Ollama, with full support for thinking/reasoning and tool calling where applicable. Templates are stored per-model and documented with design rationale.

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
