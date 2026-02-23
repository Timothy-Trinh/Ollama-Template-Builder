# Ollama Template Building Insights

> Generalizable lessons for building Go templates for any LLM on Ollama.
> Derived from deep source code analysis and empirical testing (February 2026).

---

## 1. Custom Modelfiles Use the Template-Based Parser — Always JSON Arguments

Any custom Modelfile routes through `tools/tools.go` → `findArguments()`, which scans for a JSON `{...}` object. No matter what XML/tag wrapper you use, the tool call **arguments** inside must be JSON. This is hardcoded and applies to every custom template.

The architecture-based parsers (`model/parsers/`) only activate for official library models pulled via `ollama pull`. If you are building a custom Modelfile, you are on the template-based path.

**Source:** [`tools/tools.go` — `findArguments()`](https://github.com/ollama/ollama/blob/main/tools/tools.go)

---

## 2. `parseTag()` Derives the Prefix From Your Template AST

Ollama inspects the Go template's parse tree at model load time. It finds the first `TextNode` after `.ToolCalls` and uses that string as the prefix to scan model output for tool calls. If your template has no `.ToolCalls` branch, it falls back to `{` (bare JSON detection).

**The tag you put immediately after `range .ToolCalls` determines what the parser looks for in the model's output.** This is invisible at runtime but critical for correct tool call extraction.

For example, if your template has:
```
{{ range .ToolCalls }}<tool_call>
{"name": "{{ .Function.Name }}", ...}
</tool_call>{{ end }}
```
Then `<tool_call>\n` becomes the prefix token the parser scans for.

**Source:** [`tools/template.go` — `parseTag()`](https://github.com/ollama/ollama/blob/main/tools/template.go)

---

## 3. `else if .ToolCalls` Is a Common Anti-Pattern

Many templates (copied from Qwen3 or similar references) use:
```
{{- if .Content }}{{ .Content }}
{{- else if .ToolCalls }}...
```

This silently drops tool calls from conversation history whenever a message has **both** content and tool calls. The model then has no record of prior tool calls, degrading multi-turn accuracy.

**Fix:** Always use two independent `if` blocks:
```
{{- if .Content }}{{ .Content }}
{{- end }}
{{- if .ToolCalls }}...
{{- end }}
```

This applies to any model capable of producing content alongside tool calls.

---

## 4. Negative Prompting Measurably Improves Format Compliance

When a model was trained on format X but Ollama requires format Y, adding an explicit `"Do NOT use [format X]"` instruction alongside the positive example measurably improves compliance. This was demonstrated empirically — Ollama issue #8287 showed Nemotron-mini going from ~0% to 100/100 tool call compliance when explicit format instructions with a negative example were added.

**Applicable to any model where the trained output format differs from the required output format.** Always include both:
1. A clear positive example of the required format
2. An explicit prohibition of the format the model was trained on

**Source:** [Issue #8287 — Nemotron-mini tool calling empirical validation (100-run test)](https://github.com/ollama/ollama/issues/8287)

---

## 5. Tool Definitions and Tool Output Are Independent Format Concerns

The **definition format** (how tools are presented to the model in the system prompt) and the **output format** (what the model emits when calling a tool) don't need to match. You can use XML definitions to align with training data while requiring JSON output for parser compatibility.

The parser only inspects the model's **output**. The system prompt definitions are consumed by the model, not the parser. This means you can optimize each side independently:
- **Input side:** Match the model's training format for best tool recognition
- **Output side:** Match the parser's requirements (JSON for custom Modelfiles)

---

## 6. `<think></think>` Injection Should Be Conditional

For any thinking-capable model: if the user disables thinking, injecting empty `<think></think>` tags on history turns adds noise tokens and may prime the model to include thinking structure anyway.

**Fix:** Guard think tag injection with a `$thinking` check on history turns:
```
{{- if and .Thinking (ge $i $lastUserIdx) }}
<think>{{ .Thinking }}</think>
{{ else if $thinking }}
<think></think>{{ end }}
```

The generation prompt handles the on/off toggle separately (emitting `<think>\n` vs `<think></think>` depending on mode). The history guard applies only to the message rendering loop.

---

## 7. Official Library Model ≠ Custom Modelfile Behavior

The same model pulled from `ollama.com/library` may parse tool calls through a completely different code path than a custom Modelfile for the same weights:

| Path | Invoked For | Parser Location |
|---|---|---|
| Architecture-based | Official library models (`ollama pull modelname`) | `model/parsers/` |
| Template-based | Custom Modelfiles (`ollama create -f Modelfile`) | `tools/tools.go` |

These two paths may have **different format expectations** (e.g., native XML vs JSON arguments). Never assume behavior observed with the library model transfers to a custom template, or vice versa. Always test your specific deployment.

**Verification test:** Call the model with a tool request. If `tool_calls` is populated → parser is working correctly. If `tool_calls` is null but tool call text appears in `content` → wrong parser path or format mismatch.

**Source:** [PR #13764 — Nemotron parser refactored to Qwen3Coder (architecture-based)](https://github.com/ollama/ollama/pull/13764)

---

## Quick Reference: Template-Based Parser Pipeline

```
Model output stream
    ↓
tools.NewParser(template, registeredTools)
    ↓
parseTag(template)  →  extracts prefix from .ToolCalls TextNode
    ↓
findTag(buffer)     →  scans output for prefix (e.g., "<tool_call>\n")
    ↓
findTool(tools, buffer)  →  substring match for registered tool names
    ↓
findArguments(tool, buffer)  →  scans for JSON {...} object ← MUST BE JSON
    ↓
api.ToolCall{Name, Arguments}  →  structured response to client
```

**Sources:**
- [`tools/tools.go`](https://github.com/ollama/ollama/blob/main/tools/tools.go) — Parser, findTag, findTool, findArguments
- [`tools/template.go`](https://github.com/ollama/ollama/blob/main/tools/template.go) — parseTag, findToolCallNode
- [PR #10415 — Streaming tool call parser](https://github.com/ollama/ollama/pull/10415)
- [PR #11030 — Loosened tool parsing](https://github.com/ollama/ollama/pull/11030)
