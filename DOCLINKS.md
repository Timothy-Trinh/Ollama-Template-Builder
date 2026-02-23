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
