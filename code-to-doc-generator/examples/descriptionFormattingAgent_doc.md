# DescriptionFormattingAgent

https://github.com/aodn/data-discovery-ai/blob/main/data_discovery_ai/agents/descriptionFormattingAgent.py

## What it does

Converts raw, unstructured dataset abstract text into clean Markdown format, making it more readable in the data discovery UI. For short or simple descriptions it applies lightweight URL/email link conversion only; for long, multi-paragraph abstracts it calls an LLM (GPT-4o-mini in production, Llama 3 via Ollama in development) to reformat the full text.

## Inputs

| Name | Type | Description |
|------|------|-------------|
| `title` | `str` | Dataset title — passed to the LLM as context when full formatting is triggered |
| `abstract` | `str` | Raw description text to be formatted. If absent, agent returns an empty string |

> Supplied as a single `request: Dict[str, str]`. Only abstracts with **more than 200 words AND at least one newline** (`\n`) are sent to the LLM.

## Outputs / Returns

A dict with a single key defined by `model_config.response_key`, whose value is a Markdown-formatted string. Three possible outcomes:

| Scenario | Output |
|----------|--------|
| Request invalid, `abstract` present | URL/email-linked version of the original abstract (no LLM call) |
| Request invalid, no `abstract` key | `""` (empty string) |
| Valid request + long abstract | LLM-reformatted Markdown abstract |

## Dependencies

- `BaseAgent` — parent class providing `is_valid_request` and `set_required_fields`
- `ConfigUtil` — loads description formatting config (`model`, `temperature`, `max_tokens`, `response_key`)
- `LlmModels` enum — distinguishes between `GPT` (OpenAI) and `OLLAMA` (local Llama 3) backends
- `AgentType` enum — identifies this agent as `DESCRIPTION_FORMATTING`
- `ollama.chat` — local LLM client used in development
- `asyncio` — used to run the async GPT chunked-processing path synchronously via `asyncio.run()`
- `structlog` — structured logging

---

## When it runs

Invoked by a supervisor that holds the `llm_client` (OpenAI async client). Runs during dataset indexing or re-indexing when abstract text needs to be normalised for display.

## Decision logic

1. **Gate check (`make_decision`)** — two conditions must both be true to trigger the LLM:
   - `is_valid_request(request)` passes (required fields present), AND
   - `needs_formatting(abstract)` returns `True` (word count > 200 **and** abstract contains `\n`)
2. **Short / invalid path** — if either condition fails, apply `manual_wrapper_description()` only: detect bare URLs and email addresses using regex and convert them to Markdown link syntax. No LLM is called.
3. **Long / valid path** — apply `manual_wrapper_description()` first (link wrapping), then call `take_action()`:
   - **GPT (production):** runs `take_action_async()`, which splits the abstract into chunks of ≤1000 characters at paragraph/sentence boundaries and calls the OpenAI API sequentially per chunk, passing the last sentence of the previous chunk as context to maintain continuity.
   - **Ollama (development):** sends the full abstract in a single call to the local Llama 3 model.
4. **Response parsing (`retrieve_json`)** — extracts the `formatted_abstract` value from the LLM's JSON response. Handles three formats: clean JSON object, triple-quoted JSON (common with Llama), and plain text fallback.
5. **Error fallback** — any exception in the LLM call returns the (link-wrapped) original abstract unchanged, so downstream consumers always receive something usable.
6. **Log result** — logs the final response at DEBUG level.

---

## Helper functions quick reference

| Function | Purpose |
|----------|---------|
| `needs_formatting(abstract)` | Returns `True` if abstract > 200 words and contains a newline |
| `manual_wrapper_description(abstract)` | Regex-wraps bare URLs and emails in Markdown link syntax |
| `chunk_text(text, max_length=1000)` | Splits long text at paragraph/sentence boundaries for chunked LLM calls |
| `format_chunk_async(...)` | Async call to OpenAI for a single chunk; passes previous tail for continuity |
| `extract_last_sentence(text)` | Extracts the last sentence of a formatted chunk to use as context for the next |
| `build_system_prompt()` | Constructs the LLM system prompt with formatting rules |
| `build_user_prompt(chunk_text, previous_tail)` | Constructs the user prompt with chunk markers and optional previous-tail context |
| `retrieve_json(model, output)` | Parses `formatted_abstract` from LLM output, with multi-format fallback |
| `_wrap_url(m)` / `_wrap_email(m)` | Regex match handlers that produce `[text](url)` / `[email](mailto:email)` |
| `_strip_trailing_punct(s)` | Removes trailing punctuation from URLs/emails before wrapping |

---

## Edge cases and gotchas

- **Chunking is sequential, not parallel** — despite the async signature on `take_action_async`, chunks are processed one-by-one in a `for` loop (not `asyncio.gather`). This is intentional to maintain previous-tail context between chunks. Don't assume parallel speed gains.
- **Ollama ignores chunking** — the local dev path sends the full abstract in one shot, regardless of length. This may hit token limits for very long abstracts.
- **`needs_formatting` requires both conditions** — an abstract can be 500 words with no newlines and still skip the LLM. If you're debugging why a long abstract wasn't reformatted, check for the presence of `\n`.
- **`manual_wrapper_description` always runs before the LLM** in the full-formatting path — so the LLM receives already link-wrapped text. Ensure downstream rendering handles nested Markdown links correctly.
- **`response_key` is config-driven** — the output dict key name is not hardcoded; check `ConfigUtil` to know what key to read from the response.

---

Draft generated from code only. Please verify:
- `[TODO: verify]` What exact fields does `BaseAgent.is_valid_request()` require? (not defined in this file)
- `[TODO: verify]` What is the string value of `model_config.response_key`?
- `[TODO: verify]` What OpenAI model string does `LlmModels.GPT.value` resolve to? (referenced as GPT-4o-mini in comments but not confirmed in this file)
- `[TODO: verify]` Confirm whether Ollama path is dev-only or if it is ever used in staging/production environments
