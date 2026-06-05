# Agent configuration

The agent config nests under `conversation_config`. The fields you actually touch:

```jsonc
conversation_config: {
  agent: {
    first_message: "Hi, I'm …",        // the opening line
    language: "es",                      // ISO code; set it (don't leave the template's "en")
    dynamic_variables: { dynamic_variable_placeholders: {} },  // declare CUSTOM vars here (system vars don't need it)
    prompt: {
      prompt: "<system prompt>",         // read this from its own .txt in the build script
      llm: "qwen35-397b-a17b",           // model string — see below
      reasoning_effort: "low",           // none|minimal|low|medium|high|xhigh
      temperature: 0.5, max_tokens: -1,
      timezone: "America/Matamoros",     // IANA; powers relative dates + {{system__time}}
      knowledge_base: [ … ],             // see RAG below
      rag: { enabled: true, embedding_model: "multilingual_e5_large_instruct" },
      tool_ids: [],                      // GLOBAL tool list; in a workflow, tools are scoped per node instead
      built_in_tools: {}                 // end_call, transfer_to_agent, …
    }
  },
  tts: { voice_id: "<library voice id>" },  // required even for text; use a LIBRARY voice, not a personal clone
  conversation: { text_only: true }         // true = chat bot (no voice)
}
```

**Template fields that bite if you forget to overwrite them** after pulling someone else's agent: `first_message`, `language`, `tts.voice_id`, `conversation.text_only`, `prompt.timezone`. They silently keep the source agent's values.

## Model strings

The hosted models rotate — **confirm live with `GET /convai/llm/list`** (it also flags deprecation, context/token limits, image-input). Seen in the wild: `qwen35-397b-a17b` ("great for agentic"), `gpt-oss-120b`; the older `glm-*` family gets deprecated (the UI will recommend a replacement). Claude / Gemini / OpenAI model ids are also selectable. Don't hardcode a model you didn't see in `/llm/list`.

`reasoning_effort`: `none | minimal | low | medium | high | xhigh`.

## Knowledge base + RAG

`knowledge_base` is an array of references:

```json
[ { "type": "folder", "id": "<folder_id>", "name": "manuals", "usage_mode": "auto" } ]
```

- `type`: `"file"` (one doc) or `"folder"` (bundles many docs as ONE reference — cleaner than listing each file).
- `rag`: `{ "enabled": true, "embedding_model": "<model>" }`.
- Embedding models: `e5_mistral_7b_instruct` (English), **`multilingual_e5_large_instruct`** (multilingual — use for Spanish), `qwen3_embedding_4b`.

### "RAG enabled but the agent ignores the docs / RAG Storage: 0 B" ⚠️

**Uploading a document ≠ indexing it for RAG.** The KB page showing **"RAG Storage: 0 B"** means nothing was embedded — there is nothing to retrieve. Two things must both be true:

1. The docs are attached in the **agent's** knowledge base (not only the global/workspace KB), **with RAG enabled**.
2. They've been **indexed** — the RAG-storage counter must rise off 0 B. Re-attaching the docs in the agent's KB with RAG on triggers indexing; the rising counter is the semaphore that it worked.

Fix the "0 B" by re-adding the docs in the agent's own KB with RAG on, then confirm the storage counter is non-zero before expecting retrieval.

## System dynamic variables

Reference runtime values in any prompt with `{{system__<name>}}` — **system variables are auto-available** (no need to declare them; only *custom* variables go in `dynamic_variable_placeholders`). Verified ones include:

- **`{{system__time}}`** — current time **in the agent's `timezone`**, formatted like `Friday, 01:07 05 June 2026` (24-h). Use this for time-of-day logic (greetings, "today").
- `{{system__time_utc}}` — ISO-8601 UTC. `{{system__timezone}}` — the IANA tz.
- `{{system__conversation_id}}`, `{{system__caller_id}}`, `{{system__agent_id}}`, `{{system__call_duration_secs}}`, `{{system__is_text_only}}`, `{{system__conversation_history}}`.

**Interpolation is reliable in the base system prompt.** Put time-dependent instructions there (e.g. "the local time is {{system__time}}; greet accordingly") rather than relying on a workflow node prompt to resolve them.

### Showing calendar slots in local time (UTC → local) ⚠️

Calendar integration tools — Cal.com `calcom_get_available_slots`, and calendar integrations generally — return available times in **UTC**. If the prompt doesn't handle it, the agent (1) shows the raw UTC times and (2) asks the client for their timezone — bad UX that makes clients abandon (e.g. the client asks for "9 AM", the bot lists "15:30 UTC, 16:15 UTC…" and asks "what city are you in?").

Reliable fix = a global rule in the **base** system prompt:

- **Never** ask the client for their timezone — assume the **business's** timezone.
- **Always** convert slot times from UTC to the business's local time BEFORE showing them; present only local 12-h time (AM/PM), never with a "UTC" label.
- Convert **DST-safe** (don't hardcode an offset that breaks in winter): derive the live offset as `{{system__time}}` (current local time) minus `{{system__time_utc}}` (current UTC), then apply that same offset to every slot. Both system vars are always available, so the model derives the offset live.

State this explicitly — the model does NOT solve it on its own, and it only surfaces when you **offer a list** of times: in booking the client usually gives a time directly so it goes unnoticed; in rescheduling the bot offers the list and it breaks.
