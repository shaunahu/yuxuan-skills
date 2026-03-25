# DescriptionFormattingAgent

## What it does

Converts raw metadata abstracts into clean, consistently structured Markdown — turning long, unformatted text blocks into readable documents with proper lists, links, and paragraph spacing. It is designed to improve the discoverability and readability of dataset descriptions within the AODN data discovery platform.

## Inputs

| Name | Type | Description |
|------|------|-------------|
| `request` | `Dict[str, str]` | A dictionary containing the metadata to process |
| `request["title"]` | `str` | The title of the metadata record |
| `request["abstract"]` | `str` | The raw description/abstract text to be formatted |

## Outputs / Returns

A dictionary keyed by `model_config.response_key` (configured externally) whose value is a Markdown-formatted string of the abstract. If the abstract is absent, the value is an empty string. If formatting is skipped, the original text is returned with URLs and emails wrapped in Markdown link syntax.

## Dependencies

- `ollama` — local LLM client used in development (llama3 model)
- `openai` (async client) — used in production/staging via the supervisor's `llm_client`
- `structlog` — structured logging
- `data_discovery_ai.agents.baseAgent.BaseAgent` — base class providing `is_valid_request` and `set_required_fields`
- `data_discovery_ai.config.config.ConfigUtil` — loads model config (model name, temperature, max tokens, response key)
- `data_discovery_ai.enum.agent_enums.LlmModels, AgentType` — enums for model and agent type selection
- `asyncio`, `re`, `json` — standard library

## Limitations

- Abstracts of 200 words or fewer, or single-paragraph abstracts, are **not sent to the LLM** — only URL/email wrapping is applied.
- The agent has no memory between calls; all state is per-request.
- The Ollama (dev) path processes the full abstract in a single prompt, which may degrade quality for very long texts compared to the chunked GPT path.
- JSON parsing from Ollama responses uses a multi-step fallback strategy because llama3 output formatting is inconsistent.
- Chunk boundary context is limited to the last sentence of the previous chunk; formatting continuity across large structural boundaries (e.g., a list split across chunks) may degrade.
- The supervisor reference (`self.supervisor`) must be injected via `set_supervisor()` before calling `execute()`; behaviour without it is not explicitly guarded. [TODO: verify]

---

## When it runs

The agent is invoked when a metadata record's abstract needs to be displayed to end users. It is called via `execute()`, which internally decides whether LLM formatting is warranted. A supervisor object must be injected via `set_supervisor()` prior to execution.

## Decision logic

1. **Validate the request** — checks that all required fields are present using `is_valid_request()` (inherited from `BaseAgent`).
2. **Check if formatting is needed** (`needs_formatting`) — returns `True` only if the abstract exceeds 200 words **and** contains at least one newline character (i.e., has multiple paragraphs). Short or single-paragraph abstracts skip the LLM entirely.
3. **Apply manual URL/email wrapping** — regardless of whether the LLM is invoked, `manual_wrapper_description()` first converts raw URLs and email addresses in the text to Markdown link syntax (`[text](url)` / `[email](mailto:email)`).
4. **Route to the correct LLM backend**:
   - **GPT (production/staging):** Calls `take_action_async()`, which splits the abstract into ≤1000-character chunks at paragraph/sentence boundaries, then formats each chunk sequentially — passing the last sentence of the previous chunk as context to maintain continuity.
   - **Ollama (development):** Calls the local llama3 model synchronously with the full abstract in a single prompt.
5. **Parse and return** — extracts the `formatted_abstract` value from the JSON response; falls back to the raw LLM output string if JSON parsing fails at all levels.

---

Draft generated from code only. Please verify:
- The exact config path and value of `model_config.response_key` — the output dictionary key depends on this.
- Whether a missing `self.supervisor` is handled gracefully or raises an unhandled `AttributeError`.
- Whether `set_required_fields()` is always called before `execute()` in all usage paths (base class validation depends on this).
- Confirm whether the Ollama path applies chunking for very long abstracts or always uses a single prompt (current code suggests single prompt only).
