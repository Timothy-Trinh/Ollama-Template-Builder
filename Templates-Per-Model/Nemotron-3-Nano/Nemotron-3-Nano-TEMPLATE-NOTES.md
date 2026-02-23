# Nemotron 3 Nano (30B-A3B) — Template Notes

## Model Overview

| Property | Value |
|---|---|
| **Full Name** | NVIDIA Nemotron-3-Nano-30B-A3B-v1 |
| **Architecture** | Mixture-of-Experts (30B total, 3B active) |
| **Context Length** | 262,144 tokens |
| **Chat Format** | ChatML (`<\|im_start\|>` / `<\|im_end\|>`) |
| **Thinking/Reasoning** | Yes — enabled by default |
| **Tool Calling** | Yes |
| **BOS Token** | Not used (`add_bos_token: false`) |
| **EOS Token** | `<\|im_end\|>` (ID 11) |
| **HuggingFace** | [nvidia/Nemotron-3-Nano-30B-A3B-v1](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B-v1) |

## Key Token IDs

| Token | ID | Special |
|---|---|---|
| `<\|im_start\|>` | 10 | Yes |
| `<\|im_end\|>` | 11 | Yes |
| `<think>` | 12 | No |
| `</think>` | 13 | No |
| `<tool_call>` | 14 | No |
| `</tool_call>` | 15 | No |
| `<tool_response>` | 16 | No |
| `</tool_response>` | 17 | No |

## Template Design Decisions

### Thinking Mode (Default ON)

- Thinking is **enabled by default** when the user does not explicitly set it, matching NVIDIA's canonical Jinja template behavior.
- Controlled via Ollama's `.IsThinkSet` / `.Think` variables.
- When ON: generation prompt ends with `<|im_start|>assistant\n<think>\n` — the model begins reasoning.
- When OFF: generation prompt ends with `<|im_start|>assistant\n<think></think>` — signals the model to skip reasoning and respond directly.

### Multi-Turn Thinking Truncation

- Uses a two-pass approach: first pass finds the index of the last user message (`$lastUserIdx`), second pass renders messages.
- Assistant messages **before** the last user message have their thinking content replaced with empty `<think></think>` tags. This saves context window space in long conversations.
- Assistant messages **at or after** the last user message retain full `<think>...</think>` content.
- All assistant messages that lack thinking content always receive a `<think></think>` prefix — the model expects this tag structure regardless.
- Requires Go 1.23+ template engine (variable reassignment with `=`).

### Tool Calling — JSON Format (Ollama Compatibility)

**Important trade-off:** This template uses **JSON-format tool calls** (Qwen3/ChatML style) instead of NVIDIA's native XML format.

**Why:**
- Ollama's tool call parser extracts tool calls by recognizing JSON objects with `"name"` and `"arguments"` fields in the model's output.
- NVIDIA's canonical format uses XML (`<function=name><parameter=key>value</parameter></function>`), which Ollama cannot parse into structured tool call objects.
- The Qwen3 template (also ChatML) is a proven reference — it uses JSON tool calls with Ollama successfully.

**How it works:**
1. Tool definitions are serialized as JSON inside `<tools></tools>` XML tags (via Ollama's `{{ .Function }}` helper, which calls `json.Marshal`).
2. The system prompt instructs the model to return tool calls as JSON inside `<tool_call></tool_call>` tags.
3. Tool call history is rendered as `{"name": "...", "arguments": {...}}` (using `.Function.Name` and `.Function.Arguments`).
4. Tool responses are wrapped in `<tool_response></tool_response>` inside `<|im_start|>user` turns.

**Potential limitation:** Since the model was fine-tuned on XML-format tool calls, it *may* occasionally produce XML instead of JSON, especially with ambiguous prompts. The system instruction and history should guide it toward JSON in most cases.

### Ollama Parser Architecture — Two Paths, Two Format Requirements

> **This section documents a critical architectural distinction discovered February 2026. It determines which tool call output format the template must produce.**

**As of January 17, 2026 (PR #13764, merged into main)**, Ollama maintains two separate and incompatible parsing pipelines for Nemotron-3-Nano:

| Pipeline | Location | Activation | Expected Output Format |
|---|---|---|---|
| **Architecture-based** | `model/parsers/Nemotron3NanoParser` → `Qwen3CoderParser` | Official library model (`ollama pull nemotron-3-nano`) | Native XML: `<function=name>\n<parameter=key>value\n</parameter>\n</function>` |
| **Template-based** | `tools/tools.go` → `tools.NewParser()` | Custom Modelfile (our case) | JSON: `{"name": ..., "arguments": {...}}` inside `<tool_call>` tags |

**Why they differ:**

PR #13764 refactored `Nemotron3NanoParser` to delegate tool call parsing to `Qwen3CoderParser`. That parser collects raw content between `<tool_call>` and `</tool_call>` tags, then passes it through `transformToXML()` — a function that normalizes Nemotron's shorthand XML (`<function=name>`) into proper XML and unmarshals it. This means **the architecture-based parser natively handles the model's trained output format**.

The template-based parser (`tools/tools.go`) has no XML support. Its `findArguments()` function scans for a JSON `{...}` object after the tool name substring. No JSON object = no parsed tool call.

**What this means in practice:**

- **For our custom Modelfile**: The template-based parser is almost certainly active. The Go template's `.ToolCalls` branch defines the prefix tag, and `findArguments()` then requires JSON. **JSON output format is required.**
- **For the official library model**: The architecture-based parser handles XML natively. Switching to native XML *could* work there, but cannot be assumed to work for a custom Modelfile.
- **These two format expectations are mutually exclusive.** There is no output string that satisfies both parsers.

**Verification test**: After creating a new Modelfile version, call it with a tool request and inspect the raw API response. If `tool_calls` is populated → template-based parser is running correctly on JSON output. If `tool_calls` is null but the XML appears in `content` → the architecture-based parser is running and expects XML.

**References:**
- [PR #13764 — Nemotron parser refactor to Qwen3Coder](https://github.com/ollama/ollama/pull/13764) (merged Jan 17, 2026)
- [tools/tools.go — `findArguments()` function](https://github.com/ollama/ollama/blob/main/tools/tools.go)
- [model/parsers/qwen3coder.go — `transformToXML()` and `parseToolCall()`](https://raw.githubusercontent.com/ollama/ollama/main/model/parsers/qwen3coder.go)
- [Issue #8287 — Empirical validation: hybrid format achieves 100/100 tool call compliance](https://github.com/ollama/ollama/issues/8287)

### Tool Response Per-Message Wrapping

- Each tool response gets its own `<|im_start|>user\n<tool_response>...\n</tool_response><|im_end|>` turn.
- NVIDIA's canonical Jinja groups consecutive tool responses under a single `<|im_start|>user` turn. Implementing grouping in Go templates requires tracking previous message role, adding complexity.
- The per-message approach matches Qwen3's Ollama template and works correctly with Ollama's message collation.

### `arguments` String-vs-Dict Bug

- NVIDIA's canonical Jinja template has a known bug ([HF Discussion #51](https://huggingface.co/nvidia/Nemotron-3-Nano-30B-A3B-v1/discussions/51)): it uses the `|items` filter on `tool_call.arguments`, which crashes when arguments is a JSON string instead of a dict. NVIDIA stated they will not fix it.
- This template **avoids the bug entirely**: In Ollama, `.Function.Arguments` is always `map[string]any` (a Go map), and `{{ .Function.Arguments }}` serializes it as JSON via the `.String()` method. No `|items` filter needed.

## Recommended Sampling Parameters

From NVIDIA's model card:

| Use Case | Temperature | Top-P |
|---|---|---|
| Reasoning/Thinking | 1.0 | 1.0 |
| Tool Calling | 0.6 | 0.95 |

The Modelfile uses reasoning defaults (temperature=1.0, top_p=1.0). Adjust for tool-heavy workloads if needed.

## Files

| File | Purpose |
|---|---|
| `Nemotron-3-Nano-thinking.gotmpl` | Standalone Go template (source of truth) |
| `Modelfile` | Complete Ollama Modelfile with embedded template + parameters |
| `Nemotron-3-Nano-TEMPLATE-NOTES.md` | This file |

## Changelog

**2026-02-23 — Path A Remediation (Tool Call Reliability)**
- Investigated tool call parsing failures causing the "I've completed processing but have no response to give." error in nanobot.
- Researched Ollama parser architecture (PR #13764, #11030, #10415); documented two-parser system in §*Ollama Parser Architecture*.
- Changed tool definitions from JSON to hybrid XML format (`<tool>`, `<name>`, `<description>`, `<parameters>`) to align with NVIDIA’s canonical training format.
- Strengthened output instructions with explicit negative example prohibiting XML parameter format.
- Fixed `else if .ToolCalls` → independent `if` conditions in assistant history rendering (content and tool calls now both render when present).
- Added `$thinking` guard: empty `<think></think>` tags no longer injected on history turns when thinking is disabled.
- Added sampling parameter guidance comment to Modelfile.

**2025-07-19 — Initial Creation**
- Created Go template for Nemotron 3 Nano with thinking ON by default.
- Implemented multi-turn thinking truncation via two-pass approach.
- Used JSON-format tool calling (Qwen3 pattern) for Ollama parser compatibility.
- Created Modelfile with NVIDIA-recommended sampling parameters.
- Dual-path template: Messages path (chat API) + Legacy path (/api/generate).

## Known Limitations

1. **Tool call format — hybrid approach**: The model was trained on XML-format tool calls. The template uses XML for tool *definitions* (activates training-time recognition) and JSON for tool call *output* (required by Ollama’s template-based parser). Negative prompting is included to suppress output reversion to XML. See §*Ollama Parser Architecture* for the two-parser background.
2. **Variable reassignment**: Requires Ollama built with Go 1.23+ (standard in recent Ollama releases).
3. **Tool response grouping**: Consecutive tool responses are wrapped individually rather than grouped, which slightly differs from the canonical Jinja template.
4. **No `continue` keyword**: System messages in the message loop are ignored by not matching any role condition, rather than using `continue` (which requires Go 1.23+). This works correctly.
