# Addendum: Ollama Go Template System — Complete Reference

## Why Templates Break on New Models

Ollama uses Go's `text/template` engine to render the prompt that gets sent to the model. When a new model is released, it typically ships with a Jinja2 template in its HuggingFace `tokenizer_config.json`. Ollama must convert this to a Go template. The conversion often lags behind or is incomplete — missing tool-calling logic, thinking blocks, or multimodal image handling — which is why new models frequently "don't work" properly in Ollama until someone writes a correct Go template for the `TEMPLATE` field in the Modelfile.[^1][^2]

***

## The Complete Variable Reference

These are the variables available inside an Ollama Go template. They come from two code paths in Ollama's source — a **legacy path** (single-turn) and a **messages path** (multi-turn chat).[^1]

### Top-Level Variables

| Variable | Type | Description | Code Path |
|---|---|---|---|
| `.System` | `string` | All system messages concatenated (collated from `Messages` automatically) | Both |
| `.Prompt` | `string` | User prompt (single-turn mode) | Legacy only |
| `.Response` | `string` | Assistant response text | Both |
| `.Suffix` | `string` | Fill-in-middle suffix text | Legacy only |
| `.Messages` | `[]*templateMessage` | Full conversation history | Messages path |
| `.Tools` | `templateTools` | Available tool definitions (JSON-serializable) | Messages path |
| `.Think` | `bool` | Whether thinking mode is enabled | Both |
| `.ThinkLevel` | `string` | Thinking level string (`low`, `medium`, `high` for gpt-oss) | Both |
| `.IsThinkSet` | `bool` | Whether the user explicitly set the think flag (vs. default false) | Both |

**Important**: Ollama detects which path to use by scanning the template for variable references. If your template references `.Messages`, it uses the messages path. If it only references `.Prompt`/`.Response`, it uses the legacy path. Templates that reference `.Messages` get the richer data structure.[^1]

### Message Object Fields

When iterating over `.Messages` with `range`, each message exposes:[^1]

| Field | Type | Description |
|---|---|---|
| `.Role` | `string` | One of: `system`, `user`, `assistant`, `tool` |
| `.Content` | `string` | The text content of the message |
| `.Thinking` | `string` | The reasoning trace (when thinking is enabled) |
| `.Images` | `[]ImageData` | List of image byte arrays (for multimodal models) |
| `.ToolCalls` | `[]templateToolCall` | Tool calls the assistant wants to make |
| `.ToolName` | `string` | Name of the tool (for `tool` role messages — the result return) |
| `.ToolCallID` | `string` | ID linking a tool result back to a specific call |

### ToolCall Object Fields

Each item in `.ToolCalls` has:[^1]

| Field | Type | Description |
|---|---|---|
| `.ID` | `string` | Unique identifier for this call |
| `.Function.Index` | `int` | Numeric index of this call |
| `.Function.Name` | `string` | Function name to invoke |
| `.Function.Arguments` | `templateArgs (map[string]any)` | Argument key-value pairs; has `.String()` → JSON |

### Tool Definition Fields

Each item in `.Tools` (when tools are passed via the API) exposes:[^1]

| Field | Type | Description |
|---|---|---|
| `.Type` | `string` | Always `"function"` |
| `.Function.Name` | `string` | Function name |
| `.Function.Description` | `string` | Human-readable description |
| `.Function.Parameters.Type` | `string` | Always `"object"` |
| `.Function.Parameters.Required` | `[]string` | Required property names |
| `.Function.Parameters.Properties` | `map[string]ToolProperty` | Map of property name → definition |

Each property in `.Properties` has `.Type`, `.Description`, and `.Enum`.[^1]

***

## Built-In Template Functions

Ollama registers these custom functions on the Go template engine (found in `template.go`):[^3]

| Function | Signature | Description |
|---|---|---|
| `json` | `json .Value` → `string` | Marshals any value to JSON string. Critical for serializing tool definitions and arguments. |
| `currentDate` | `currentDate` → `string` | Returns today's date as `YYYY-MM-DD` |
| `yesterdayDate` | `yesterdayDate` → `string` | Returns yesterday's date as `YYYY-MM-DD` |
| `toTypeScriptType` | `toTypeScriptType .Property` → `string` | Converts a ToolProperty to its TypeScript type string (for Harmony-style templates) |

In addition, all standard Go `text/template` built-in functions are available:

| Function | Description |
|---|---|
| `eq`, `ne`, `lt`, `le`, `gt`, `ge` | Comparison operators |
| `and`, `or`, `not` | Boolean logic |
| `len` | Length of arrays/maps/strings |
| `slice` | Sub-slice of an array (e.g., `slice $.Messages $i`) |
| `index` | Index into array or map |
| `printf` | Formatted string output |
| `call` | Call a function value |

***

## How Ollama Handles Thinking / Reasoning

Ollama has first-class support for thinking models. The `think` parameter in the API request flows into the template as `.Think` (bool), `.ThinkLevel` (string), and `.IsThinkSet` (bool).[^4][^5]

### How it works at the template level

The template itself does **not** typically need to handle `<think>` tags directly. Ollama's runtime does the heavy lifting:

1. When `think: true` is passed, Ollama appends `<think>` after the generation prompt
2. The model generates reasoning tokens inside `<think>...</think>`
3. Ollama's **parser** splits the output: everything inside `<think>` goes to `message.thinking`, everything after `</think>` goes to `message.content`
4. In multi-turn, previous `.Thinking` content is available on message objects

For templates that **do** need to reference thinking state (e.g., to conditionally format previous reasoning in context), use:[^5]

```
{{- if and .Think .Thinking }}
<think>
{{ .Thinking }}
</think>
{{- end }}
{{ .Content }}
```

### Supported models and their behavior

| Model | `think` Value | Behavior |
|---|---|---|
| Qwen 3 | `true`/`false` | Standard boolean toggle[^5] |
| DeepSeek R1 | `true`/`false` | Standard boolean toggle[^5] |
| DeepSeek V3.1 | `true`/`false` | Standard boolean toggle[^5] |
| GPT-OSS | `"low"`, `"medium"`, `"high"` | String levels only; `true`/`false` is ignored[^5] |

### CLI shortcuts

```bash
ollama run deepseek-r1 --think "Solve this step by step"
ollama run deepseek-r1 --think=false "Quick answer"
ollama run gpt-oss --think=high "Complex reasoning task"
# Interactive toggle:
/set think
/set nothink
```


***

## How Ollama Handles Tool Calling

Tool calling in Ollama works through a combination of the template (which formats tool definitions and calls into the prompt) and Ollama's built-in parser (which extracts structured tool calls from the model output).[^6]

### The tool calling lifecycle

1. **API receives tools**: User passes `tools` array in the chat request
2. **Template renders definitions**: The `.Tools` variable becomes available; the template injects them into the system message
3. **Model generates a tool call**: The model outputs structured text (e.g., `<tool_call>{"name": "...", "arguments": {...}}</tool_call>`)
4. **Ollama parses the output**: The parser recognizes the tool call format and populates `message.tool_calls` instead of `message.content`
5. **User executes the tool**: Client-side code runs the actual function
6. **Results are sent back**: A `tool` role message with the result is added to the conversation
7. **Model generates final answer**: With the tool result in context, the model produces the final response

### Template pattern for tool definitions

Here's the standard pattern used by ChatML models (Qwen3) in Ollama:[^7]

```go
{{- if .System }}{{- if .Tools }}
<|im_start|>system
{{ .System }}

# Tools

You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>
{{- range .Tools }}
{"type": "function", "function": {{ .Function }}}
{{- end }}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
{{- end }}<|im_end|>
{{ end }}
```

### Template pattern for tool calls in assistant messages

```go
{{- if eq .Role "assistant" }}
<|im_start|>assistant
{{ if .Content }}{{ .Content }}
{{- else if .ToolCalls }}<tool_call>
{{ range .ToolCalls }}{"name": "{{ .Function.Name }}", "arguments": {{ .Function.Arguments }}}
{{ end }}</tool_call>
{{- end }}
{{- end }}
```

### Template pattern for tool results

```go
{{- if eq .Role "tool" }}
<|im_start|>user
<tool_response>
{{ .Content }}
</tool_response><|im_end|>
{{- end }}
```

Note: In the Qwen3 Ollama template, tool results are wrapped as a `user` role message with `<tool_response>` tags, even though the API-level role is `tool`. This is a common pattern because many models were trained to expect tool results as special user messages.[^7]

***

## How Ollama Handles Multimodal / Images

Images in Ollama are passed as base64-encoded byte arrays in the `images` field of a message. Inside the template, they're available as `.Images` on each message object.[^8][^9]

However, most Ollama templates do **not** reference `.Images` directly. Instead, Ollama's `collate` function in the source code automatically inserts `[img-N]` placeholder tags into the message `.Content` when images are present. The model's underlying engine (llama.cpp / Ollama's new engine) then processes these placeholders and injects the actual image embeddings at the correct positions.[^3]

For template authors, the key insight is: **you generally don't need to handle images in the template**. They're processed at a layer below the template. However, if you need to explicitly reference images (rare), you can:

```go
{{- range $i, $img := .Images }}
[image {{ $i }}]
{{- end }}
```

The Llama 3.2 Vision template, for example, does not have special image handling in its Go template — images are injected by the runtime.[^10]

***

## Jinja2 → Go Template Conversion Guide

This section provides a systematic mapping from Jinja2 constructs (as found in HuggingFace model configs) to Go template equivalents for Ollama.

### Core Syntax Mapping

| Jinja2 | Go Template | Notes |
|---|---|---|
| `{{ variable }}` | `{{ .Variable }}` | Go uses dot-prefix for struct fields |
| `{% if condition %}` | `{{ if condition }}` | |
| `{% elif condition %}` | `{{ else if condition }}` | |
| `{% else %}` | `{{ else }}` | |
| `{% endif %}` | `{{ end }}` | |
| `{% for item in list %}` | `{{ range .List }}` | Inside `range`, `.` becomes the current item |
| `{% endfor %}` | `{{ end }}` | |
| `{% set var = value %}` | N/A | Go templates have no mutable variables — restructure logic |
| `{{ loop.index0 }}` | `{{ $i }}` (with `range $i, $_ :=`) | Declare index variable in range |
| `{{ loop.last }}` | `{{ eq (len (slice $.List $i)) 1 }}` | Common Ollama pattern for detecting last item |
| `{{ loop.first }}` | `{{ eq $i 0 }}` | |
| `{{ value \| tojson }}` | `{{ json .Value }}` | Ollama's custom `json` function |
| `{{ value \| length }}` | `{{ len .Value }}` | |
| `{{ value \| trim }}` | N/A (handle in logic) | No built-in trim in Go templates |
| `{% if tools is defined %}` | `{{ if .Tools }}` | Zero-value check (nil/empty = false) |
| `{{ namespace(...) }}` | N/A | Go has no namespace; use `$` for root scope |

### Whitespace Control

| Jinja2 | Go Template | Effect |
|---|---|---|
| `{%- ... %}` | `{{- ... }}` | Trim leading whitespace |
| `{% ... -%}` | `{{ ... -}}` | Trim trailing whitespace |
| `{%- ... -%}` | `{{- ... -}}` | Trim both |

This is one of the most important aspects of template conversion. Incorrect whitespace can add extra newlines or spaces that corrupt the special token sequence, causing the model to malfunction.[^1]

### Accessing Parent Scope Inside `range`

In Jinja2, variables from the outer scope are always accessible. In Go templates, when you enter a `range` block, `.` is rebound to the current item. To access the outer (root) context, use `$`:

```
Jinja:  {{ tools }}          (inside a for loop)
Go:     {{ $.Tools }}        (inside a range block)
```

This is heavily used in Ollama templates — for example, to check if tools exist while iterating over messages:[^7]

```go
{{- range $i, $_ := .Messages }}
{{- if and (eq .Role "user") $.Tools }}
  {{/* tools are available from parent scope */}}
{{- end }}
{{- end }}
```

### The `$last` Pattern

A very common Ollama pattern is detecting whether the current message is the last one in the list, which determines whether to add the generation prompt. The standard idiom:[^11][^7]

```go
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 }}
  ...
{{- if and $last (ne .Role "assistant") }}<|im_start|>assistant
{{ end }}
{{- end }}
```

This works because `slice $.Messages $i` returns a sub-slice starting at index `$i`. If its length is 1, we're on the last element.

### Converting a Complete Template: Worked Example

Here's a side-by-side conversion of a simplified ChatML template:

**Jinja2 (HuggingFace)**:
```jinja
{% for message in messages %}
{%- if message['role'] == 'system' %}
<|im_start|>system
{{ message['content'] }}<|im_end|>
{%- elif message['role'] == 'user' %}
<|im_start|>user
{{ message['content'] }}<|im_end|>
{%- elif message['role'] == 'assistant' %}
<|im_start|>assistant
{{ message['content'] }}<|im_end|>
{%- endif %}
{%- endfor %}
{% if add_generation_prompt %}
<|im_start|>assistant
{% endif %}
```

**Go Template (Ollama)**:
```go
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 }}
{{- if eq .Role "system" }}
<|im_start|>system
{{ .Content }}<|im_end|>
{{- else if eq .Role "user" }}
<|im_start|>user
{{ .Content }}<|im_end|>
{{- else if eq .Role "assistant" }}
<|im_start|>assistant
{{ .Content }}{{ if not $last }}<|im_end|>
{{ end }}
{{- end }}
{{- if and (ne .Role "assistant") $last }}
<|im_start|>assistant
{{ end }}
{{- end }}
```

Key differences in the Go version:
- `add_generation_prompt` doesn't exist as a variable — instead, Ollama convention is to check if the last message is not an assistant message and then open the assistant header
- The assistant's closing `<|im_end|>` is omitted on the last message (the one being generated)
- `message['content']` becomes `.Content` (struct field access)

***

## Real Ollama Templates: Annotated Examples

### DeepSeek R1 (Thinking Model)

This is the actual template from `ollama.com/library/deepseek-r1`:[^11]

```go
{{- if .System }}{{ .System }}{{ end }}
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1}}
{{- if eq .Role "user" }}<｜User｜>{{ .Content }}
{{- else if eq .Role "assistant" }}<｜Assistant｜>{{ .Content }}
  {{- if not $last }}<｜end▁of▁sentence｜>{{- end }}
{{- end }}
{{- if and $last (ne .Role "assistant") }}<｜Assistant｜>{{- end }}
{{- end }}
```

Notable points:
- System message has no special tokens — it's raw text at the beginning
- The fullwidth Unicode tokens (`<｜User｜>`, etc.) must be byte-exact
- No explicit thinking handling in the template — Ollama's runtime handles `<think>` parsing
- `<｜end▁of▁sentence｜>` only appears between turns, not on the last (generating) turn

### Llama 4 (Tool Calling + Vision)

From `ollama.com/library/llama4`:[^12]

```go
{{- if .System }}<|header_start|>system<|header_end|>
{{ if .Tools }}Environment: ipython
You have access to the following functions...
{{ range .Tools }}{{ . }}
{{ end }}
{{- end }}{{ .System }}<|eot|>
{{- end }}
{{- range $i, $_ := .Messages }}
{{- if eq .Role "system" }}{{- continue }}
{{- end }}
{{- if and (ne .Role "tool") (not .ToolCalls) }}<|header_start|>{{ .Role }}<|header_end|>
{{ .Content }}
{{- else if .ToolCalls }}<|header_start|><|python_start|>
{{- range .ToolCalls }}{"name": "{{ .Function.Name }}", "parameters": {{ .Function.Arguments }}}
{{- end }}<|python_end|><|eom|>
{{- continue }}
{{- else if or (eq .Role "tool") (eq .Role "ipython") }}<|header_start|>ipython<|header_end|>
{{ .Content }}
{{- end }}
{{- if eq (len (slice $.Messages $i)) 1 }}
{{- if ne .Role "assistant" }}<|eot|><|header_start|>assistant<|header_end|>
{{ end }}
{{- else }}<|eot|>
{{- end }}
{{- end }}
```

Notable points:
- System message skipped in the `range` loop (handled separately above)
- Tool definitions are rendered by `{{ . }}` on each Tool (which triggers its `.String()` → JSON)
- Tool calls use `<|python_start|>` / `<|python_end|>` with `<|eom|>` (end-of-message, not end-of-turn)
- Tool results use the `ipython` role header
- `{{- continue }}` skips to the next iteration (requires Go 1.23+ template engine)

### Qwen3 (ChatML + Tools + Thinking)

The full Qwen3 template from Ollama:[^7]

```go
{{- if .Messages }}
{{- if or .System .Tools }}<|im_start|>system
{{- if .System }}
{{ .System }}
{{- end }}
{{- if .Tools }}

# Tools

You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>
{{- range .Tools }}
{"type": "function", "function": {{ .Function }}}
{{- end }}
</tools>

For each function call, return a json object with function name and arguments
within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
{{- end }}<|im_end|>
{{ end }}
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 -}}
{{- if eq .Role "user" }}<|im_start|>user
{{ .Content }}<|im_end|>
{{ else if eq .Role "assistant" }}<|im_start|>assistant
{{ if .Content }}{{ .Content }}
{{- else if .ToolCalls }}<tool_call>
{{ range .ToolCalls }}{"name": "{{ .Function.Name }}", "arguments": {{ .Function.Arguments }}}
{{ end }}</tool_call>
{{- end }}{{ if not $last }}<|im_end|>
{{ end }}
{{- else if eq .Role "tool" }}<|im_start|>user
<tool_response>
{{ .Content }}
</tool_response><|im_end|>
{{ end }}
{{- if and (ne .Role "assistant") $last }}<|im_start|>assistant
{{ end }}
{{- end }}
{{- else }}
{{- if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}{{ if .Prompt }}<|im_start|>user
{{ .Prompt }}<|im_end|>
{{ end }}<|im_start|>assistant
{{ end }}{{ .Response }}{{ if .Response }}<|im_end|>{{ end }}
```

This template demonstrates the dual-path pattern:
- **Messages path** (top): Full multi-turn with tools and proper role handling
- **Legacy path** (bottom, after `{{ else }}`): Simple `.System`/`.Prompt`/`.Response` for basic `/api/generate` calls

***

## Common Pitfalls and Debugging

### Pitfall 1: Whitespace corruption
Extra newlines between special tokens will tokenize differently and confuse the model. Always use `{{-` and `-}}` aggressively.[^1]

### Pitfall 2: Forgetting `$.` inside `range`
Inside a `range` block, `.Tools`, `.System`, etc. won't work — you need `$.Tools`, `$.System`. This is the #1 cause of broken tool templates.

### Pitfall 3: Missing stop tokens
The template alone isn't enough. You must also set `PARAMETER stop` in the Modelfile for each special token that should halt generation:[^13]
```
PARAMETER stop "<|im_start|>"
PARAMETER stop "<|im_end|>"
PARAMETER stop "<|eot_id|>"
```

### Pitfall 4: `ToolCalls` appears empty
There's a known issue where `Messages[].ToolCalls` may not be correctly populated if the template receives tool calls in an unexpected format. Ensure the API request matches Ollama's expected structure exactly.[^14]

### Pitfall 5: No `continue` in older Ollama versions
The `{{- continue }}` action in Go templates is relatively new. If targeting older Ollama versions, use nested `if/else` blocks instead.

### Debugging templates
Use `ollama show --modelfile <model>` to inspect the current template. For deeper debugging, set `OLLAMA_DEBUG=1` to see the rendered prompt.[^13]

***

## Documentation Links

| Resource | URL |
|---|---|
| Ollama Template Documentation | https://github.com/ollama/ollama/blob/main/docs/template.md |
| Ollama Modelfile Reference | https://docs.ollama.com/modelfile |
| Ollama Tool Calling Guide | https://docs.ollama.com/capabilities/tool-calling |
| Ollama Thinking Documentation | https://docs.ollama.com/capabilities/thinking |
| Ollama template.go Source | https://github.com/ollama/ollama/blob/main/template/template.go |
| Ollama API Reference (Go types) | https://pkg.go.dev/github.com/ollama/ollama/api |
| Go text/template Documentation | https://pkg.go.dev/text/template |
| Qwen3 Ollama Template (live) | https://ollama.com/library/qwen3/blobs/eb4402837c78 |
| DeepSeek R1 Ollama Template (live) | https://ollama.com/library/deepseek-r1/blobs/369ca498f347 |
| Llama 4 Ollama Template (live) | https://ollama.com/library/llama4/blobs/8a13cf51fd9e |

---

## References

1. [template - GitHub](https://github.com/ollama/ollama/blob/main/docs/template.md) - Get up and running with OpenAI gpt-oss, DeepSeek-R1, Gemma 3 and other models. - ollama/ollama

2. [How exactly do the templates in a model file work? : r/ollama - Reddit](https://www.reddit.com/r/ollama/comments/1bk9gue/how_exactly_do_the_templates_in_a_model_file_work/) - I can't really find a solid, in-depth description of the TEMPLATE syntax (the Ollama docs just refer...

3. [ollama/template/template.go at main - GitHub](https://github.com/ollama/ollama/blob/main/template/template.go) - Get up and running with OpenAI gpt-oss, DeepSeek-R1, Gemma 3 and other models. - ollama/template/tem...

4. [Thinking · Ollama Blog](https://ollama.com/blog/thinking) - Ollama now has the ability to enable or disable thinking. This gives users the flexibility to choose...

5. [Thinking - Ollama's documentation](https://docs.ollama.com/capabilities/thinking) - Thinking streams interleave reasoning tokens before answer tokens. Detect the first thinking chunk t...

6. [Tool calling - Ollama's documentation](https://docs.ollama.com/capabilities/tool-calling) - Invoke a single tool and include its response in a follow-up request. Also known as “single-shot” to...

7. [qwen3/template - Ollama](https://ollama.com/library/qwen3/blobs/eb4402837c78) - Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive sui...

8. [Ollama's new engine for multimodal models](https://ollama.com/blog/multimodal-models) - This demonstrates how a user can input multiple images at once, or do so via follow up prompts and a...

9. [ollama.generate() with multimodal llama3.2-vision does not pass ...](https://github.com/ollama/ollama-python/issues/319) - I would like to be able to pass a prompt like the one above and an image into ollama.generate() with...

10. [llama3.2-vision](https://ollama.com/library/llama3.2-vision) - The Llama 3.2-Vision instruction-tuned models are optimized for visual recognition, image reasoning,...

11. [deepseek-r1/template - Ollama](https://ollama.com/library/deepseek-r1/blobs/369ca498f347) - DeepSeek-R1 is a family of open reasoning models with performance approaching that of leading models...

12. [llama4/template - Ollama](https://ollama.com/library/llama4/blobs/8a13cf51fd9e) - Meta's latest collection of multimodal models. ... You have access to the following functions. To ca...

13. [Modelfile Reference - Ollama's documentation](https://docs.ollama.com/modelfile) - A Modelfile is the blueprint to create and share customized models using Ollama. ​. Table of Content...

14. [Messages[].ToolCalls not passed correctly to the template #9802](https://github.com/ollama/ollama/issues/9802) - Ollama's template system is not correctly processing the tool_calls field in assistant messages. Whe...

