# SCRATCHPAD.md

## Working Memory

### Ollama Tool Call Parsing

- Ollama's tool call parser operates at the server level (not in the runner). The runner handles token generation and stop sequences.
- The parser recognizes JSON objects with `"name"` and `"arguments"` fields. The template must instruct the model to output this format.
- The Qwen3 template (ChatML) is the proven reference for JSON tool calls. It uses `<tool_call>` XML tags around JSON objects.
- `{{ .Function }}` calls `.String()` which does `json.Marshal` — outputs full function definition as JSON.
- `{{ .Function.Arguments }}` is `map[string]any` in Go, serialized as JSON. Avoids the Jinja string-vs-dict bug entirely.
- Ollama's `collate()` function merges consecutive same-role messages EXCEPT tool messages. Each tool message remains separate.

### Nemotron-Specific Notes

- `add_bos_token: false` — no BOS token needed.
- Think tags (`<think>`/`</think>`) are **not** special tokens (IDs 12/13) — they're regular text tokens. This means the model can generate them freely.
- Tool tags (`<tool_call>`, etc.) are also non-special (IDs 14-17).
- The model always expects `<think></think>` before assistant content, even when thinking is disabled. This is a Nemotron-specific requirement.
- NVIDIA recommends temperature=1.0, top_p=1.0 for reasoning; temperature=0.6, top_p=0.95 for tool calling.

### Variable Reassignment in Go Templates

- `$var = value` (assignment) modifies an existing variable declared with `$var := value` at an outer scope.
- Works in Go 1.23+ (modern Ollama). Inside `range` or `if` blocks, `=` modifies the outer scope variable.
- Used for `$thinking = false` inside `if` block and `$lastUserIdx = $i` inside `range` block.

### Ollama Parser Architecture — Two Separate Systems

- **OLD path (`tools/tools.go`)**: Template-based parser. `parseTag()` extracts first TextNode after `.ToolCalls` from Go template as the prefix. Then `findArguments()` scans for a JSON object `{...}`. Arguments MUST be JSON. Used when the Go template has a `.ToolCalls` branch. Called from `server/routes.go` via `tools.NewParser(m.Template.Template, req.Tools)`.
- **NEW path (`model/parsers/`)**: Architecture-based parsers. `Nemotron3NanoParser` handles thinking state, delegates to `Qwen3CoderParser` for all tool call and content parsing. `Qwen3CoderParser` extracts raw content between `<tool_call>` and `</tool_call>`, then calls `transformToXML()` which parses the native Nemotron XML format (`<function=name><parameter=key>value</parameter>`). This **CAN parse native XML** — no JSON required.

### PR #13764 — Critical Finding (Merged Jan 17, 2026)

- Ollama removed the old Nemotron-specific XML regex parser and replaced it with Qwen3CoderParser delegation.
- PR description: "Nemotron parser now only handles the thinking state machine and transitions to Qwen3CoderParser for content and tool call parsing."
- Qwen3CoderParser's `transformToXML()` function normalizes Nemotron's `<tag=value>` format to proper XML and parses it.
- This means: **on Ollama ≥ post-Jan-17-2026 builds, native XML tool call output from Nemotron-3-Nano can be parsed** without any JSON requirement, IF the architecture-based parser is being used.
- Key uncertainty: custom Modelfiles with Go templates may still trigger `tools/tools.go` (JSON-requiring) path. Architecture parser may only activate for library models or when no custom template overrides the parsing path.

### Issue #8287 — Empirical Validation (Nemotron-Mini Predecessor)

- Nemotron-mini, the predecessor model, with default template: 0% reliable tool call parsing.
- With explicit hybrid instructions in template (`<toolcall>\n{"name": ..., "arguments": ...}\n</toolcall>`): 100/100 compliance.
- This empirically proves the hybrid format works for Nemotron family models. Source: https://github.com/ollama/ollama/issues/8287

### Native XML vs Hybrid JSON — Which Parser Path Is Active?

- These two formats are **mutually exclusive**. There is no format compatible with BOTH `tools/tools.go` AND `Qwen3CoderParser`.
- `tools/tools.go` needs JSON `{...}` arguments after the tag.
- `Qwen3CoderParser` needs `<function=name><parameter=key>value</parameter>` XML inside `<tool_call>` tags.
- For custom Modelfiles: likely `tools/tools.go` path is used (template-driven).
- For official library model (`ollama pull nemotron-3-nano`): likely architecture-based `Nemotron3NanoParser` → `Qwen3CoderParser`.

---

## Musing (Current Session)

Session complete. Path A remediation implemented across `.gotmpl` and `Modelfile`. All five changes applied and verified. Documentation updated. No open questions.

Next recommended step: build the model with `ollama create nemotron3-nano-thinking -f Modelfile` and run the verification test — call with a tool-capable request and confirm `tool_calls` is populated in the API response (not in `content`).
