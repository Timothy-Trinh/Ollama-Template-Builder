# DOCLINKS.md

## References Used

| Source | Description |
|---|---|
| `References/Ollama-Go-Template-Addendum.md` | Complete reference for Ollama's Go template system — variables, functions, thinking handling, tool calling lifecycle, Jinja→Go conversion guide, annotated examples. |
| `References/LLM-Chat-Template-Schemes.md` | Survey of major LLM chat template schemes (ChatML, Llama, Mistral, etc.) with format details per model family. |
| [NVIDIA Nemotron 3 Nano HuggingFace Model Card](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B-v1) | Official model card with architecture details, recommended sampling parameters, and canonical Jinja template. |
| [HuggingFace tokenizer_config.json](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B-v1/resolve/main/tokenizer_config.json) | Token IDs, special tokens, and canonical chat template source. |
| [HuggingFace Discussion #51](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B-v1/discussions/51) | Bug report for `arguments` string-vs-dict issue in canonical Jinja. NVIDIA confirmed they will not fix. |
| [Community Fixed Jinja Template (omarkamali gist)](https://gist.github.com/omarkamali/c3d295e1ac0939f0f07e3c2a1ddbb921) | Community-contributed fixed Jinja template addressing the arguments bug. |
| [Ollama template.go source](https://github.com/ollama/ollama/blob/main/template/template.go) | Ollama template engine source — templateArgs, templateProperties types, collate function, built-in functions (json, currentDate). |
| [Ollama runner.go source](https://github.com/ollama/ollama/blob/main/runner/ollamarunner/runner.go) | Ollama runner source — stop sequence handling, token generation. Tool call parsing happens at a higher server level, not in the runner. |
| [Ollama PR #13764 — Nemotron parser refactored to Qwen3Coder](https://github.com/ollama/ollama/pull/13764) | Jan 2026 PR delegating Nemotron tool call parsing to `Qwen3CoderParser`. `Nemotron3NanoParser` now handles only thinking state; `Qwen3CoderParser` handles content and tool calls using native XML parsing. Critical for understanding which parser path is active for library vs custom Modelfile. |
| [Ollama tools/tools.go — `findArguments()` function](https://github.com/ollama/ollama/blob/main/tools/tools.go) | Template-based tool call parser used for custom Modelfiles. `findArguments()` scans for a JSON `{...}` object — no XML support. |
| [Ollama model/parsers/qwen3coder.go — `transformToXML()` / `parseToolCall()`](https://raw.githubusercontent.com/ollama/ollama/main/model/parsers/qwen3coder.go) | Architecture-based parser for the official library model. Natively handles Nemotron XML format (`<function=name><parameter=key>value</parameter>`). Incompatible with template-based parser. |
| [Ollama issue #8287 — Nemotron-mini empirical validation](https://github.com/ollama/ollama/issues/8287) | 100-run test showing hybrid format (XML definitions + JSON output instructions) achieves 100/100 tool call compliance vs 0% with default template. |
| [Ollama PR #11030 — Loosened tool parsing](https://github.com/ollama/ollama/pull/11030) | Three-step tool parsing: prefix tag → function name → JSON arguments. Step 3 is hardcoded JSON, confirming no XML argument support in template-based parser. |
| `References/TEMPLATE-BUILDING-INSIGHTS.md` | Generalizable insights for building Go templates for any LLM on Ollama — parser mechanics, common anti-patterns, hybrid format strategy, and verification guidance. Derived from source code analysis and empirical testing. |
