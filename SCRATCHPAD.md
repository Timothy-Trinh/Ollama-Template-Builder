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

---

## Musing (Current Session)

Session complete. All deliverables for Nemotron 3 Nano template created. No open questions at this time.
