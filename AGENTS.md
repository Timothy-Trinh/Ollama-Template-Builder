# AGENTS.md

Welcome, Agent. Your task is to build better Go templates for LLMs, using a combination of references, web search, and operator guidance.

## Project Documentation

Files in the *References* folder are not to be modified without direct, and explicit approval by an operator. Reference "Ollama-Go-Template-Addendum.md" whenever working in this repo.

## Instructions for Agents

- Before starting work, review PROJECT_CHANGELOG.md, DOCLINKS.md, and SCRATCHPAD.md for project history, reference sources, and working memory from prior sessions. Update these files as needed during your session.
- When starting a new session in this repo, you should assume that the user wants to create or modify a Go template for an LLM. These templates are stored in the 'Templates-Per-Model' folder, in subfolders per LLM. If a subfolder does not exist for the LLM you are working on, create it, and place templates for your LLM in that subfolder.
- When creating templates you MUST use documents in the References folder as a reference - if you are capable of web search, consider using it for additional context on the ask. When using web search, huggingface should generally be used as a reference for specific information on LLMs.
- If the user's ask is unclear, or does not seem to pertain to the purpose of this repo, ask clarifiying questions before proceeding - this repo is for building new templates for LLMs. Ask the user if there is an LLM they should build a template for, revise a template for, etc.
- When building a template, two things matter most:
    - The target LLM has an expected response format. Some of the formats used are described in the document LLM-Chat-Template-Schemes.md - refer to this for information. A template WILL NOT FUNCTION if it is not designed for the response format the model expects. If the operator does not clarify the response format for the LLM we are building a template for, use web search to find more information. If you cannot use web search, ask the operator for clarity.
    - When building a template using this repo, the expectation is that you fold as much functionality into the template as possible. You must assume that we are building templates for Ollama, and you must reference Ollama-Go-Template-Addendum.md. When starting, if it is unclear, ask the operator if the LLM you are working on is a thinking or non-thinking model, and if the model is a thinking model, ensure thinking is included as part of the template. Assume that any LLM you are creating a template for supports tools, and build a tool calling section of the template with the appropriate modifications for the LLM's expected response format. Unless it is otherwise known, assume the LLM should output json based tool output, and create the template accordingly. 
- After creating an initial template, give the operator a report detailing what functionality the template attempts to implement, what you know about the model for the purposes of template creation, limitations, and uncertanties. Additionally, create a 'modelname'-TEMPLATE-NOTES.md file in the template subfolder - eg 'templates-per-model/llama-3.1/', detailing in brief, what you know about the model, and how you built the template, with a changelog and a brief summary of user requests.