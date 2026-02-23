# LLM Chat Template Schemes: A Comprehensive Reference

## Overview

Every instruction-tuned LLM is trained with a specific serialization format that converts a list of `{role, content}` message objects into a single token sequence the model can process. These formats differ in their special tokens, role delimiters, and how they handle features like thinking (chain-of-thought) and tool calling. Understanding each scheme is essential for writing correct Jinja2 or Go templates.[^1][^2]

***

## Template Scheme Table

| Scheme Name | Special Tokens | Providers Using It | Documentation |
|---|---|---|---|
| **ChatML** | `<\|im_start\|>`, `<\|im_end\|>` | Qwen (all versions), 01.AI / Yi, Liquid AI (LFM2/LFM2.5), Arcee AI (default), many community fine-tunes | [OpenAI ChatML spec (GitHub)](https://github.com/openai/openai-python/blob/main/chatml.md)[^3]; [HuggingFace Qwen3 deep dive](https://huggingface.co/blog/qwen-3-chat-template-deep-dive)[^4] |
| **Llama Instruct** | `<\|begin_of_text\|>`, `<\|start_header_id\|>`, `<\|end_header_id\|>`, `<\|eot_id\|>`, `<\|eom_id\|>`, `<\|python_tag\|>` | Meta (Llama 3, 3.1, 3.2, 3.3) | [Meta Llama 3.1 prompt format](https://www.llama.com/docs/model-cards-and-prompt-formats/llama3_1/)[^5]; [Meta Llama 3 prompt format](https://www.llama.com/docs/model-cards-and-prompt-formats/meta-llama-3/)[^6] |
| **Mistral Instruct** | `<s>`, `</s>`, `[INST]`, `[/INST]` (v1-v3 with whitespace variations) | Mistral AI (Mistral 7B, Mixtral, Mistral Large, Codestral, etc.) | [Mistral tokenization deep dive](https://docs.mistral.ai/cookbooks/concept-deep-dive-tokenization-chat_templates)[^7] |
| **Gemma** | `<start_of_turn>`, `<end_of_turn>`, `<bos>`, `<eos>` | Google (Gemma, Gemma 2, Gemma 3) | [Google Gemma prompt structure](https://ai.google.dev/gemma/docs/core/prompt-structure)[^8]; [FunctionGemma formatting](https://ai.google.dev/gemma/docs/functiongemma/formatting-and-best-practices)[^9] |
| **DeepSeek** | `<｜begin▁of▁sentence｜>`, `<｜User｜>`, `<｜Assistant｜>`, `<｜end▁of▁sentence｜>`, `<think>`, `</think>` | DeepSeek (V2, V3, V3.1, R1, R1-0528) | [DeepSeek V3.1 HuggingFace card](https://huggingface.co/deepseek-ai/DeepSeek-V3.1)[^10]; [DeepSeek thinking mode docs](https://api-docs.deepseek.com/guides/thinking_mode)[^11] |
| **Granite** | `<\|start_of_role\|>`, `<\|end_of_role\|>`, `<\|end_of_text\|>` | IBM (Granite 3.x, Granite 4.0 family) | [IBM Granite prompt engineering](https://www.ibm.com/granite/docs/use-cases/prompt-engineering)[^12]; [Unsloth Granite 4.0 guide](https://unsloth.ai/docs/models/tutorials/ibm-granite-4.0)[^13] |
| **Harmony** | `<\|start\|>`, `<\|end\|>`, `<\|message\|>`, `<\|channel\|>`, `<\|constrain\|>`, `<\|return\|>`, `<\|call\|>` | OpenAI (gpt-oss-120b, gpt-oss-20b) | [OpenAI Harmony format guide](https://developers.openai.com/cookbook/articles/openai-harmony/)[^14]; [gpt-oss model card (PDF)](https://cdn.openai.com/pdf/419b6906-9da6-406c-a19d-1bb078ac7637/oai_gpt-oss_model_card.pdf)[^15] |
| **Kimi** | `<\|im_system\|>`, `<\|im_user\|>`, `<\|im_assistant\|>`, `<\|im_middle\|>`, `<\|im_end\|>` | Moonshot AI (Kimi K2, Kimi K2-Thinking) | [Kimi K2-Instruct HuggingFace card](https://huggingface.co/moonshotai/Kimi-K2-Instruct)[^16]; [Unsloth Kimi K2 guide](https://unsloth.ai/docs/models/tutorials/kimi-k2-thinking-how-to-run-locally)[^17] |

### Notes on Providers Not Listed Above

- **StepFun (Step-2, Step-3.5)**: Uses a proprietary API format. Their open weights for Step-Audio models use custom audio tokens but the text chat interface follows OpenAI-compatible API conventions. No public standalone chat template specification is documented for their text LLMs.
- **Arcee AI**: Their models (e.g., Arcee-Blitz, Arcee-Nova) primarily use **ChatML** by default, with the option to swap in any template from the model's `tokenizer_config.json`.[^18]
- **01.AI / Yi**: Yi-1.5 and Yi-34B-Chat use **ChatML** (`<|im_start|>`, `<|im_end|>`) as confirmed in their tokenizer configs, with `<|startoftext|>` as BOS[^19].
- **Liquid AI (LFM2/LFM2.5)**: Uses a **ChatML-like** format with an additional `<|startoftext|>` BOS token. Tool calls use `<|tool_call_start|>`/`<|tool_call_end|>` and `<|tool_list_start|>`/`<|tool_list_end|>` for definitions[^20][^21].

***

## Detailed Scheme Breakdowns

### 1. ChatML

The most widely adopted format. Originally designed by OpenAI, now the de facto standard for Qwen, Yi, and many community models.[^3][^22]

```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Hello!<|im_end|>
<|im_start|>assistant
Hi there!<|im_end|>
```

**Thinking mode** (Qwen3): When `enable_thinking=True`, the assistant response is prefilled with `<think>\n` and the model generates its chain-of-thought inside `<think>...</think>` before the actual answer. When disabled, the template injects an empty `<think>\n\n</think>\n\n` block to skip reasoning.[^4][^23]

**Tool calling** (Qwen3): Tool definitions are serialized as JSON objects inside `<tools>...</tools>` XML tags in the system message. The model returns calls inside `<tool_call>...</tool_call>` tags. Tool responses come back wrapped in `<tool_response>...</tool_response>` tags within a user message.[^23][^24]

### 2. Llama Instruct

Meta's format uses header-based role tokens rather than start/end markers.[^5][^6]

```
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

You are a helpful assistant<|eot_id|><|start_header_id|>user<|end_header_id|>

Hello!<|eot_id|><|start_header_id|>assistant<|end_header_id|>

```

**Key tokens**:
- `<|begin_of_text|>` — BOS, appears once at the very start
- `<|eot_id|>` — End of turn, terminates each message
- `<|eom_id|>` — End of message (used when the model calls a built-in tool and expects to continue)
- `<|python_tag|>` — Precedes tool call code in the model's response
- Role `ipython` — Used for tool/function response messages[^5]

**Tool calling**: Llama 3.1+ defines tools in a system message with JSON schemas. The model emits tool calls after `<|python_tag|>`, and results return under the `ipython` role[^5][^25].

### 3. Mistral Instruct

The simplest format, wrapping user messages in `[INST]...[/INST]` blocks. System messages are concatenated into the user message (prepended to first or last user message depending on tokenizer version).[^7]

```
<s>[INST] user message [/INST] assistant message</s>[INST] new user message [/INST]
```

**Tokenizer versions** matter for whitespace handling:
- **V1**: Spaces around `[INST]` content
- **V2**: No space before content, space after control tokens
- **V3 (Tekken/Nemo)**: No extra whitespace at all[^7]

**Tool calling**: Mistral uses the standard OpenAI-compatible `tools` parameter with JSON schema definitions. The model returns `tool_calls` objects in its response. Tool call IDs are required.[^26][^27]

### 4. Gemma

Google's format uses `<start_of_turn>` / `<end_of_turn>` with roles `user` and `model` (not "assistant").[^8][^28]

```
<bos><start_of_turn>user
What is Cramer's Rule?<end_of_turn>
<start_of_turn>model
```

**Important quirk**: `<end_of_turn>` is the turn delimiter, but the actual EOS token is `<eos>`. The template does not always append `<eos>` after the final turn, which can affect training.[^29]

**Tool calling (FunctionGemma)**: Uses six dedicated tokens — `<start_function_declaration>`/`<end_function_declaration>`, `<start_function_call>`/`<end_function_call>`, `<start_function_response>`/`<end_function_response>` — plus an `<escape>` token for string delimiters within structured data.[^9]

### 5. DeepSeek

Uses fullwidth Unicode-like tokens with underscored separators. No explicit role-close token — the system prompt text appears directly after BOS, and roles like `<｜User｜>` and `<｜Assistant｜>` act as turn markers.[^30][^10]

```
<｜begin▁of▁sentence｜>You are a helpful assistant<｜User｜>Hello!<｜Assistant｜>
```

**Thinking mode**: In thinking mode, `<think>` is appended after `<｜Assistant｜>` to trigger chain-of-thought. The model generates reasoning inside `<think>...</think>`, then produces the final answer. In multi-turn, previous thinking content is dropped from context but `</think>` is retained.[^10][^30]

**Tool calling**: Supported in **non-thinking mode** only. Uses a function-calling format similar to OpenAI's API conventions.[^11]

### 6. Granite (IBM)

Uses role-boundary tokens without nesting or XML-style tags.[^12][^31]

```
<|start_of_role|>system<|end_of_role|>You are a helpful assistant.<|end_of_text|>
<|start_of_role|>user<|end_of_role|>Hello!<|end_of_text|>
<|start_of_role|>assistant<|end_of_role|>Hi there!<|end_of_text|>
```

**Tool calling**: Granite 4.0 follows OpenAI's function definition JSON schema. When tools are supplied, the chat template automatically formats them as a system prompt. The model supports multiple system turns within a single conversation.[^32][^12]

### 7. Harmony (OpenAI gpt-oss)

The most complex format, designed for multi-channel reasoning models. Uses a header/message structure with explicit channels.[^14]

```
<|start|>system<|message|>You are ChatGPT...
Reasoning: high
# Valid channels: analysis, commentary, final.<|end|>
<|start|>developer<|message|># Instructions
Be helpful.<|end|>
<|start|>user<|message|>What is 2+2?<|end|>
<|start|>assistant<|channel|>analysis<|message|>Simple arithmetic.<|end|>
<|start|>assistant<|channel|>final<|message|>2 + 2 = 4.<|return|>
```

**Unique features**:
- **Multi-channel output**: `analysis` (hidden CoT), `commentary` (tool preambles, shown to user), `final` (the answer)
- **Role hierarchy**: system > developer > user > assistant > tool
- **Stop tokens**: `<|return|>` (done generating) and `<|call|>` (wants to call a tool)
- **Tool definitions**: Use TypeScript-like syntax inside a `namespace functions { ... }` block in the developer message
- **Tool calls**: Use `to=functions.function_name` in the header and `<|constrain|> json` for typed output[^14]

### 8. Kimi (Moonshot)

A ChatML variant with distinct per-role tokens instead of a generic `<|im_start|>role` pattern[^17][^16].

```
<|im_system|><|im_middle|>You are Kimi.<|im_end|>
<|im_user|><|im_middle|>Hello!<|im_end|>
<|im_assistant|><|im_middle|>Hi!<|im_end|>
```

Each role gets its own start token (`<|im_system|>`, `<|im_user|>`, `<|im_assistant|>`), and `<|im_middle|>` bridges between the role token and the content. The EOS/stop token is `<|im_end|>`[^17].

**Tool calling**: Uses OpenAI-compatible `tools` parameter with JSON schema. The model returns `tool_calls` with function name and arguments. Tool results come back under the `tool` role with a `tool_call_id`.[^16]

***

## Anatomy of a Chat Template

Regardless of which scheme you're implementing, every chat template must handle the same fundamental components. Here is a platform/language-agnostic breakdown of what every template needs.

### 1. Sequence Boundaries

Every template starts with a **BOS (beginning-of-sequence)** token and understanding how/when the **EOS (end-of-sequence)** token terminates generation.

| Concept | Purpose | Examples |
|---|---|---|
| BOS token | Marks the absolute start of the input sequence | `<\|begin_of_text\|>` (Llama), `<\|startoftext\|>` (LFM2), `<bos>` (Gemma), `<｜begin▁of▁sentence｜>` (DeepSeek) |
| EOS / stop token | Signals the model to stop generating | `<\|im_end\|>` (ChatML), `<\|eot_id\|>` (Llama), `</s>` (Mistral), `<\|end_of_text\|>` (Granite), `<\|return\|>` or `<\|call\|>` (Harmony) |

**Template rule**: BOS appears exactly once at the beginning. EOS appears at the end of each completed message (training) or is generated by the model to signal completion (inference).[^2][^1]

### 2. Role Delimiters

Each message has a role (`system`, `user`, `assistant`, `tool`). The template must mark where each role's content begins and ends.

**Pattern A — Wrapper tokens** (ChatML, Granite, Kimi):
```
<role_start>{role}\n{content}<role_end>
```

**Pattern B — Header tokens** (Llama):
```
<header_start>{role}<header_end>\n\n{content}<turn_end>
```

**Pattern C — Instruction tags** (Mistral):
```
[INST] {content} [/INST] {response}
```

**Pattern D — Turn markers** (Gemma, DeepSeek):
```
<turn_start>{role}\n{content}<turn_end>
```

**Pattern E — Structured messages** (Harmony):
```
<start>{role}<message>{content}<end>
```

### 3. System Message Handling

The system message sets the model's behavior and is handled differently across schemes:

- **Dedicated system role** (ChatML, Llama, Granite, Harmony, Kimi): System gets its own delimited message block
- **Prepended to user message** (Mistral): System prompt is concatenated into the first or last user message because the format has no explicit system role
- **Optional** (Gemma): Can use a `developer` turn or simply omit, depending on model version[^8][^7]

### 4. Generation Prompt

When serving inference, the template must end with an **open assistant turn** — the role header for the assistant with no content and no closing token. This signals the model to begin generating.[^33][^2]

```
# ChatML
<|im_start|>assistant\n

# Llama
<|start_header_id|>assistant<|end_header_id|>\n\n

# Gemma
<start_of_turn>model\n

# Granite
<|start_of_role|>assistant<|end_of_role|>

# Harmony
<|start|>assistant
```

**Important**: During training, the generation prompt is NOT added — the full assistant response including its closing token is included instead.[^33]

### 5. Thinking / Reasoning Blocks

For reasoning models, the template must handle a "thinking" or "chain-of-thought" phase before the final answer.

| Scheme | Thinking Mechanism | Enable/Disable |
|---|---|---|
| ChatML (Qwen3) | `<think>...</think>` inside assistant content | `enable_thinking=True/False`; disabled = inject empty `<think>\n\n</think>\n\n`[^4] |
| DeepSeek | `<think>...</think>` after `<｜Assistant｜>` | `thinking=True/False` param; prior turn thinking is dropped from context[^10] |
| Harmony | `<\|channel\|>analysis` messages (separate message blocks) | `Reasoning: high/medium/low` in system message; CoT goes to `analysis` channel, answer to `final`[^14] |
| Kimi K2-Thinking | `<think>...</think>` inside assistant content | Uses `<think>` tag similar to Qwen3/DeepSeek[^17] |

**Template rule for thinking**: 
1. If thinking is enabled and it's a generation prompt, append the thinking-start token/tag after the assistant role header
2. If thinking is disabled, either inject an empty think block (Qwen3) or omit the think token and use a `</think>` prefix (DeepSeek)
3. In multi-turn context, strip previous thinking content from history but retain the closing marker

### 6. Tool / Function Calling

Tool calling requires the template to handle four phases:

#### Phase 1: Tool Definitions
Where and how available tools are described to the model:

| Scheme | Definition Location | Format |
|---|---|---|
| ChatML (Qwen3) | System message | JSON schemas inside `<tools>...</tools>` XML tags[^23] |
| Llama 3.1+ | System message | JSON schemas with `Environment: ipython` directive[^5] |
| Mistral | API `tools` parameter | OpenAI-compatible JSON schema[^26] |
| Gemma | Turn content | `<start_function_declaration>...<end_function_declaration>` blocks[^9] |
| Granite | System message (auto-formatted by template) | OpenAI function definition JSON schema[^12] |
| Harmony | Developer message `# Tools` section | TypeScript-like namespace syntax[^14] |
| Liquid AI | System message | JSON objects between `<\|tool_list_start\|>...<\|tool_list_end\|>`[^21] |

#### Phase 2: Tool Calls (Model Output)
How the model signals it wants to call a tool:

| Scheme | Call Format |
|---|---|
| ChatML (Qwen3) | `<tool_call>\n{"name": "...", "arguments": {...}}\n</tool_call>` inside assistant message[^23] |
| Llama | `<\|python_tag\|>` followed by function call; stop token is `<\|eom_id\|>` (expects to continue)[^5] |
| Gemma | `<start_function_call>...<end_function_call>` with structured data using `<escape>` delimiters[^9] |
| Harmony | Header includes `to=functions.name`, channel is `commentary`, body is JSON with `<\|constrain\|> json`; stop token is `<\|call\|>`[^14] |
| Liquid AI | `<\|tool_call_start\|>...<\|tool_call_end\|>` with Pythonic function call syntax[^21] |

#### Phase 3: Tool Results
How execution results are fed back:

| Scheme | Result Format |
|---|---|
| ChatML (Qwen3) | `<tool_response>\n{result}\n</tool_response>` wrapped inside a `user` role message[^23] |
| Llama | Message with role `ipython` containing the result[^5] |
| Gemma | `<start_function_response>...<end_function_response>` in user turn[^9] |
| Harmony | `<\|start\|>functions.name to=assistant<\|channel\|>commentary<\|message\|>{result}<\|end\|>`[^14] |
| Liquid AI | Content between `<\|tool_response_start\|>...<\|tool_response_end\|>` in a `tool` role message[^21] |

#### Phase 4: Final Response
After receiving tool results, the model generates a new assistant turn with the final answer, using the standard response format.

### 7. Multi-Turn History

When building multi-turn conversations, the template must:

1. **Alternate roles correctly** — user/assistant turns must alternate (with tool turns inserted between as needed)
2. **Close every completed message** — each non-final message needs its proper end token
3. **Strip or retain thinking** — most schemes drop previous thinking content from context in subsequent turns (DeepSeek, Harmony) while keeping the response[^14][^10]
4. **Handle tool round-trips** — tool call → tool result → assistant continuation must be represented as sequential messages

### 8. Template Variable Checklist

Every Jinja or Go template implementation should accept and handle these variables:

| Variable | Type | Purpose |
|---|---|---|
| `messages` | `[]Message` | The conversation history |
| `add_generation_prompt` | `bool` | Whether to append the open assistant header for inference |
| `tools` | `[]Tool` (optional) | Available tool/function definitions |
| `enable_thinking` | `bool` (optional) | Toggle reasoning mode on/off |
| `bos_token` | `string` | The model's BOS token |
| `eos_token` | `string` | The model's EOS/stop token |

***

## Jinja ↔ Go Template Conversion Notes

When converting between Jinja2 and Go's `text/template`:

- **Jinja `{% for %}` loops** → Go `{{ range }}` blocks. Note: Jinja has `loop.first`, `loop.last`, `loop.index0`; Go requires manual tracking via custom functions or pipeline tricks.
- **Jinja filters** (e.g., `| tojson`, `| trim`, `| length`) → Go needs custom template functions registered via `FuncMap`.
- **Jinja `{% set %}` for namespace variables** → Go templates have no mutable state; you'll need to pass state through custom functions or restructure logic.
- **Jinja conditional `{% if tools is defined %}`** → Go uses `{{ if .Tools }}` (zero-value check).
- **Jinja string slicing** (`content[:10]`) → Requires a custom Go `slice` function.
- **Special token output** — Both Jinja and Go templates output special tokens as literal strings. The tokenizer layer (not the template) converts these to their token IDs. Ensure the Go template outputs the exact same byte-for-byte string as the Jinja template.[^34][^2]

The key challenge is that Jinja2 templates used by HuggingFace models are often quite complex (especially Qwen3's, which handles thinking state, multi-step tool detection, and reasoning content splitting). A Go equivalent will likely need helper functions for string manipulation that Jinja handles natively.[^4][^23]

***

## Authoritative Documentation Links

| Resource | URL |
|---|---|
| HuggingFace Chat Templating Guide | https://huggingface.co/docs/transformers/en/chat_templating |
| HuggingFace Template Writing Guide | https://huggingface.co/docs/transformers/en/chat_templating_writing |
| OpenAI ChatML Original Spec | https://github.com/openai/openai-python/blob/main/chatml.md |
| OpenAI Harmony Format Guide | https://developers.openai.com/cookbook/articles/openai-harmony/ |
| Meta Llama 3.1 Prompt Format | https://www.llama.com/docs/model-cards-and-prompt-formats/llama3_1/ |
| Mistral Tokenization & Templates | https://docs.mistral.ai/cookbooks/concept-deep-dive-tokenization-chat_templates |
| Google Gemma Prompt Structure | https://ai.google.dev/gemma/docs/core/prompt-structure |
| Google FunctionGemma Format | https://ai.google.dev/gemma/docs/functiongemma/formatting-and-best-practices |
| DeepSeek V3.1 Template | https://huggingface.co/deepseek-ai/DeepSeek-V3.1 |
| IBM Granite Prompt Engineering | https://www.ibm.com/granite/docs/use-cases/prompt-engineering |
| Liquid AI Chat Template Docs | https://docs.liquid.ai/docs/key-concepts/chat-template |
| Kimi K2 Instruct Card | https://huggingface.co/moonshotai/Kimi-K2-Instruct |
| Qwen3 Chat Template Deep Dive | https://huggingface.co/blog/qwen-3-chat-template-deep-dive |

---

## References

1. [Messages and Special Tokens - Hugging Face Agents Course](https://huggingface.co/learn/agents-course/en/unit1/messages-and-special-tokens) - They act as the bridge between conversational messages (user and assistant turns) and the specific f...

2. [Chat templates - Hugging Face](https://huggingface.co/docs/transformers/en/chat_templating) - Chat templates should already include all the necessary special tokens, and adding additional specia...

3. [ChatML and the ChatGPT API - Matt Rickard](https://blog.matt-rickard.com/p/chatml-chatgpt-api) - OpenAI released a ChatGPT API today that's 1/10th the price of the leading model, text-davinci-003.

4. [The 4 Things Qwen-3's Chat Template Teaches Us - Hugging Face](https://huggingface.co/blog/qwen-3-chat-template-deep-dive) - Qwen-3 shows us that through the chat_template we can provide better flexibility, smarter context ha...

5. [Llama 3.1 | Model Cards and Prompt formats](https://www.llama.com/docs/model-cards-and-prompt-formats/llama3_1/) - This token is used for padding text sequences to the same length in a batch. <|start_header_id|>. <|...

6. [Llama 3 | Model Cards and Prompt formats](https://www.llama.com/docs/model-cards-and-prompt-formats/meta-llama-3/) - <|eot_id|>: This signifies the end of the message in a turn. <|start_header_id|>{role}<|end_header_i...

7. [Demystifying Mistral's Instruct Tokenization & Chat Templates](https://docs.mistral.ai/cookbooks/concept-deep-dive-tokenization-chat_templates) - Here, the only special strings were [INST] to start the user message and [/INST] to end the user mes...

8. [Gemma formatting and system instructions | Google AI for Developers](https://ai.google.dev/gemma/docs/core/prompt-structure) - Token to indicate the end of dialogue turn: <end_of_turn>. Here's an example dialogue: <start_of_tur...

9. [FunctionGemma formatting and best practices](https://ai.google.dev/gemma/docs/functiongemma/formatting-and-best-practices) - FunctionGemma builds on the Gemma prompt structure, using <start_of_turn>role and <end_of_turn> to d...

10. [deepseek-ai/DeepSeek-V3.1 - Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V3.1) - The multi-turn template is the same with non-thinking multi-turn chat template. It means the thinkin...

11. [Thinking Mode | DeepSeek API Docs](https://api-docs.deepseek.com/guides/thinking_mode) - Please refer to the sample code below for the correct way. Sample Code​. Below is a simple sample co...

12. [Prompt Engineering Guide - IBM Granite](https://www.ibm.com/granite/docs/use-cases/prompt-engineering) - Granite 4.0 models chat template also supports multiple system turns within a single conversation. T...

13. [IBM Granite 4.0 | Unsloth Documentation](https://unsloth.ai/docs/models/tutorials/ibm-granite-4.0) - Chat template: Copy <|start_of_role|>system<|end_of_role|>You are a helpful assistant. Please ensure...

14. [OpenAI Harmony Response Format](https://developers.openai.com/cookbook/articles/openai-harmony/) - If you are using tiktoken these tokens are encoded in the o200k_harmony encoding. All special tokens...

15. [[PDF] gpt-oss-120b & gpt-oss-20b Model Card](https://cdn.openai.com/pdf/419b6906-9da6-406c-a19d-1bb078ac7637/oai_gpt-oss_model_card.pdf) - ... format known as the harmony chat format. This format provides special tokens to delineate messag...

16. [moonshotai/Kimi-K2-Instruct - Hugging Face](https://huggingface.co/moonshotai/Kimi-K2-Instruct) - We have updated our tokenizer implementation. Now special tokens like [EOS] can be encoded to their ...

17. [Kimi K2 Thinking: Run Locally Guide | Unsloth Documentation](https://unsloth.ai/docs/models/tutorials/kimi-k2-thinking-how-to-run-locally) - Chat template and prompt format. Kimi Chat does use a BOS (beginning of sentence token). The system,...

18. [Open Source Catalog](https://www.arcee.ai/open-source-catalog) - Chat Template. If you want to use a chat template other than chatml, copy it from the model's tokeni...

19. [Does Yi-1.5-Chat model use the standard CHATML template? #23](https://github.com/01-ai/Yi-1.5/issues/23) - Using standard chatml templates, bos_token and eos_token mainly depend on the tokenizer_config.json ...

20. [Chat Template - LFM Docs! - Liquid AI](https://docs.liquid.ai/docs/key-concepts/chat-template) - Conversations are formatted using special tokens: <|startoftext|> — Start of the conversation. <|im_...

21. [LiquidAI/LFM2.5-1.2B-Thinking - Hugging Face](https://huggingface.co/LiquidAI/LFM2.5-1.2B-Thinking) - Chat Template. LFM2.5 uses a ChatML-like format. See the Chat Template documentation for details. Ex...

22. [ChatML vs Harmony: Understanding the new Format from ...](https://huggingface.co/blog/kuotient/chatml-vs-harmony) - Key characteristics: Uses special tokens <|im_start|> and <|im_end|> to mark message boundaries; Sim...

23. [unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF · New Chat Template ...](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF/discussions/10) - This one from old qwen3 discussions does work for a few calls with RooCode and llamacpp: qwen3-chat-...

24. [qwen3/template - Ollama](https://ollama.com/library/qwen3/blobs/eb4402837c78) - Qwen3 is the latest generation of large language models in Qwen series, offering a comprehensive sui...

25. [Llama 3.1 changed its chat template, again... : r/LocalLLaMA - Reddit](https://www.reddit.com/r/LocalLLaMA/comments/1eg5wgb/llama_31_changed_its_chat_template_again/) - This new chat template adds proper support for tool calling, and also fixes issues with missing supp...

26. [Function Calling | Mistral Docs](https://docs.mistral.ai/capabilities/function_calling) - Function calling, under the Tool Calling umbrela, allows Mistral models to connect to external local...

27. [mistralai/Mistral-7B-Instruct-v0.3 - Hugging Face](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3) - For a full tool calling example, please see the function calling guide, and note that Mistral does u...

28. [google/gemma-2b-it - Hugging Face](https://huggingface.co/google/gemma-2b-it) - Turns finish with the <end_of_turn> token. You can follow this format to build the prompt manually, ...

29. [Gemma template won't end with eos_token · Issue #32110 - GitHub](https://github.com/huggingface/transformers/issues/32110) - This behavior is quite inconsistent with the templates of other models. gemma always end with <end_o...

30. [deepseek-v3.1 Model by Deepseek-ai - NVIDIA NIM APIs](https://build.nvidia.com/deepseek-ai/deepseek-v3_1/modelcard) - DeepSeek-V3.1 Overview. Description. DeepSeek-V3.1 is a hybrid model that supports both thinking and...

31. [adding default system prompt · ibm-granite/granite-4.0-micro at ...](https://huggingface.co/ibm-granite/granite-4.0-micro/commit/111f8049e9fce173f9e0db6de78b726cdfdd74d1) - Granite 4.0 instruct models feature improved *instruction following (IF)* and *tool-calling* capabil...

32. [Granite 4.0 - IBM](https://www.ibm.com/granite/docs/models/granite) - When a list of tools is supplied, the chat template automatically formats this list as a system prom...

33. [Templates - Hugging Face](https://huggingface.co/docs/transformers/v4.51.1/chat_templating) - Underlying this high-level pipeline is the apply_chat_template method. A chat template is a part of ...

34. [ChatML chat history formatting : r/LocalLLaMA - Reddit](https://www.reddit.com/r/LocalLLaMA/comments/17jcohn/chatml_chat_history_formatting/) - Tokenizer normally does not translate that to actual tokens im_start and im_end . Make sure the mode...

