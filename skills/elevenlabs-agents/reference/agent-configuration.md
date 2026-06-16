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

## Naming an agent: the resource name ≠ the persona

Two distinct names, on two distinct planes — decide both when creating an agent:

- The **agent name on the platform** (dashboard + `name` in the config + the `agent_configs/` filename) is **functional/descriptive**: `<business> <function>` (e.g. `acme order taker`, `<business> assistant`, `insurance advisor`).
- The **persona** — how the bot introduces itself in chat (a first-name character) — lives ONLY in the system prompt / `first_message`, NEVER as the platform agent name.

Why: one dashboard lists agents for many businesses; a functional name tells you whose it is and what it does without opening it. The persona is user-facing marketing and can change without renaming the resource. (The agent's repo may take the persona's name — no conflict, different planes.)

## `force_pre_tool_speech` is silently reset to false on a `text_only` agent ⚠️

Setting `force_pre_tool_speech=true` has **no effect** on a `text_only` (chat) agent — the server resets it to false on push. If you copy a voice agent's config (or follow voice-agent guidance) onto a chat agent and rely on this flag to make the agent speak before a tool call, it silently does nothing; don't debug your prompt for the missing utterance, the flag was never honored.

For a text-only agent the setting is unnecessary anyway: deliver any pre-tool / closing utterance via the `end_call` system tool's `system__message_to_speak` field (or the working node's confirmation message). (Observed: after a push, the server had reset `force_pre_tool_speech` from true to false on a `text_only` agent.)

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

#### The exact endpoint that triggers indexing

Uploading the file and triggering the RAG index are **two separate calls** — the second is the non-obvious one:

```bash
# 1. upload the doc → returns its id
curl -X POST -H "xi-api-key: $KEY" \
  -F "file=@knowledge-base/doc.md;type=text/markdown" -F "name=<name>" \
  https://api.elevenlabs.io/v1/convai/knowledge-base/file
# → {"id":"<doc_id>", ...}

# 2. TRIGGER the RAG index (this is the step that's easy to miss)
curl -X POST -H "xi-api-key: $KEY" -H 'Content-Type: application/json' \
  -d '{"model":"multilingual_e5_large_instruct"}' \
  https://api.elevenlabs.io/v1/convai/knowledge-base/<doc_id>/rag-index
# → {"id":..., "status":"new", "progress_percentage":0, "document_model_index_usage":{"used_bytes":N}}

# 3. poll until succeeded
curl -H "xi-api-key: $KEY" .../knowledge-base/<doc_id>/rag-index
# → {"indexes":[{"status":"succeeded","progress_percentage":100.0,...}]}
```

Hard check: `status: succeeded` AND `used_bytes > 0`. A ~5 KB doc indexes in seconds. The index `model` MUST match the consuming agent's `rag.embedding_model`.

#### Text docs (no file), and the index/poll/delete calls

Beyond the file-upload path, you can create a KB doc straight from text and manage its lifecycle (base `https://api.elevenlabs.io/v1/convai`, auth `xi-api-key`):

```bash
POST /knowledge-base/text   {text, name}                       # → {id}   (create a TEXT doc, no file)
POST /knowledge-base/{id}/rag-index  {model:"multilingual_e5_large_instruct"}   # index — model MUST match the agent's rag.embedding_model
GET  /knowledge-base/{id}/rag-index                            # poll: success = status "succeeded" AND document_model_index_usage.used_bytes > 0
GET  /knowledge-base/rag-index                                 # workspace-wide: total_used_bytes 0 = the "RAG Storage 0 B" symptom (nothing embedded anywhere)
DELETE /knowledge-base/{id}                                    # → 204
```

#### RAG retrieves single distinctive words

With `usage_mode: auto` (RAG-only retrieval) the KB **does** retrieve on a single-word query when the term is distinctive — correcting the common assumption that RAG is useless for short jargon. Caveat (not measured): low-signal common jargon, very short common words, or a much larger document may retrieve less reliably.

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
