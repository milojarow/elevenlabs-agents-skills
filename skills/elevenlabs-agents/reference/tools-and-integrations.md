# Tools & integrations

## The four tool types

| Type | What it is | Auth / wiring |
|---|---|---|
| **`webhook`** | Your own HTTP endpoint the agent calls | you define `api_schema` (url, method, headers, request/response body schema); secrets via header (see below) |
| **`client`** | Runs in the client app (browser/mobile), not server-side | the embedding app registers a handler; good for "is the user logged in?" style calls |
| **`system`** | Built-in (`end_call`, `transfer_to_agent`, `transfer_to_number`, …) | toggled in `built_in_tools` |
| **`api_integration_webhook`** | A pre-built connector (Cal.com, etc.) | references `api_integration_id` + `api_integration_connection_id`; the request schema is owned by the integration (`base_api_schema`) |

A tool is a workspace object (`/convai/tools/{id}`); agents reference tool ids. Scope which tools a given workflow node may use via the node's `additional_tool_ids` (see [workflows.md](workflows.md)).

## webhook tool — schema shape

```json
{ "type": "webhook", "name": "send_otp", "description": "…",
  "response_timeout_secs": 20,
  "api_schema": {
    "url": "https://api.example.com/v1/otp/send", "method": "POST",
    "request_headers": { "X-Site-Key": { "secret_id": "<secret_id>" } },
    "request_body_schema": { "properties": { "email": { "type": "string", "description": "…" } }, "required": ["email"] }
  } }
```

### Header secrets: `{"secret_id": "…"}`, NOT `{{name}}` ⚠️

The single nastiest tool wall. To put a workspace secret in a header, the value must be a **structured object** `{"secret_id": "<id>"}`. A **string** `"{{my_secret}}"` is **not interpolated — it is sent literally**, so the endpoint sees the literal text `{{my_secret}}` and returns **401**.

```bash
# 1. the secret must exist and hold the RIGHT value
GET  /convai/secrets                              # lists {secret_id, name, type, used_by} — NOT the value
PATCH /convai/secrets/{secret_id}  {"name":"…","value":"<value>","type":"update"}   # update value (can't read it back)

# 2. set the header as a structured secret ref, then confirm it LINKED
#    GET /convai/tools/{id} → .tool_config.api_schema.request_headers
#    "X-Site-Key": { "secret_id": "<secret_id>" }   ✅ resolves
#    "X-Site-Key": "{{my_site_key}}"                 ❌ sent literally → 401
GET /convai/secrets   # the secret's used_by.tools[] must now list your tool. Empty used_by = it never linked.
```

`used_by.tools == []` is the tell that the reference didn't bind. Two failure modes stack: a *wrong value* in the secret AND a *string-instead-of-secret_id* reference — fix both.

## Integration tools & the connection lifecycle

`api_integration_webhook` tools authenticate through a **connection** (`api_integration_connection_id`, e.g. `icxn_…`) created from a credential the integration defines.

```bash
# discover the credential schema + the integration's tool templates
GET /convai/api-integrations/{integration_id}      # → .credentials[] (e.g. credential_id "calcom_api_key", field "api_key"), .tool_provider.tools[]

# create a connection (note: user_config_values, NOT user_config)
POST /convai/api-integrations/{integration_id}/connections
     {"name":"…", "credential_id":"<credential_id>", "user_config_values":{"<field>":"<value>"}}
     # → {connection_id}.   GET on /connections is 405 (not listable) — track the id you get back.
```

### You CANNOT change a tool's connection in place

`PATCH`-ing `api_integration_connection_id` on an existing tool returns **400 "Create a new tool to use a different connection."** To migrate an integration to a different account/key:

1. Create the new connection (above).
2. For each tool: `GET` it, set `.tool_config.api_integration_connection_id = <new>`, then **`POST /convai/tools`** (clone — yields a NEW tool id; duplicate names are fine).
3. Repoint the agent to the new tool ids (in each workflow node's `additional_tool_ids`) and **push**. → then verify the saved agent uses the new ids.
4. `DELETE /convai/tools/{old}` for each old tool (204).
5. `DELETE /convai/api-integrations/{integration_id}/connections/{conn_id}` (200) — **fails 400 if ANY tool still uses it** (it names the offender, including orphan tools not attached to any agent). Delete those first.

> After repointing, the agent's referenced tools must ALL exist. If you delete an old tool while a *cached* conversation still references it, that conversation fails with `Documents with ids {…} not found` — see the cache trap in [operating-cli-api.md](operating-cli-api.md).

### Inspect a tool's REAL schema to diagnose param hallucination

When an agent "invents" a parameter value, the first step is to confirm whether that param is even the LLM's to fill. `GET /convai/tools/{id}` returns the real parameter schema: `required[]`, and per property `dynamic_variable`, `constant_value`, `is_system_provided`, `is_omitted`. If all of those are empty/false for a property, the value comes **from the LLM** (it's the model's responsibility); if `constant_value`/`dynamic_variable`/`is_system_provided` is set, the platform supplies it. Read this before blaming the prompt — it tells you exactly which params the model is choosing.

### A parameter the LLM shouldn't fill

On integration tools the request schema (and its `constant_value`) is owned by the integration and won't persist your edits. If a small model keeps guessing a parameter (an id, a filter), pin the value in the workflow **node prompt**, or rebuild as a plain `webhook` tool with the value baked in. Also watch enum-valued filters: passing an invalid status filter (e.g. a record's own status string used as a list filter) silently returns zero rows — use the documented filter values only. See [verification-and-gotchas.md](verification-and-gotchas.md).

### When the integration schema BLOCKS you → wrap it with your own webhook tool

Worse than a non-persisting `constant_value`: **a parameter you need isn't in the integration's base schema at all**, so the LLM can't pass it and pinning it in the prompt does nothing. The base schema comes from the integration template and is **not editable by API** — a `PATCH` to `base_api_schema` returns 200 but is silently discarded; overrides go in `api_schema_overrides.schema_overrides` with a union shape discriminated by `source` (422 `Unable to extract tag using discriminator 'source'` if you don't match it) — in practice editable only via the dashboard.

**Example (timezone):** a Cal.com `get_available_slots` integration tool that sends no `timeZone` → Cal.com returns slots in **UTC** → the LLM has to convert UTC→local and does it **wrong** (off by the offset). "Fixing" it with a fixed subtraction in the prompt (e.g. "−5 hours") then breaks on every **DST** change.

**Robust fix — replace the integration tool with your own `webhook` tool to a backend you control:**

1. Build an endpoint that calls the upstream API with the right params, does the transform **in code** (real tz library = DST-correct), and returns clean fields. The schema is 100% yours. Bake fixed values (e.g. the `eventTypeId`) in server-side so the LLM never touches them.
2. Create the `type: "webhook"` tool via `POST /convai/tools`. **Model it on a webhook tool that ALREADY works in that agent** (GET its config, change name/url/request_body_schema, POST). Header secret = `{"secret_id":"…"}` (reuse the same secret).
3. **Repoint the nodes:** GET the full agent, in the relevant nodes swap old→new id in `additional_tool_ids` and `gsub` the tool name in `additional_prompt`, then `PATCH` the **whole workflow** back (full replacement, not partial). Then `PATCH` the agent prompt to drop the by-LLM conversion instruction.
4. **Verify with a real conversation** (offered times vs real times), not the self-report or `simulate`.

**Slot/time pattern:** always return TWO fields per slot — a **display value** (local time, converted in code) and a **machine value** (original ISO/UTC). The agent shows one and books with the other. **Never let the LLM do timezone arithmetic.**

**Keep the repo as source of truth:** the new tool lives in ElevenLabs (created by API) and the workflow references it by id — update your build script (the tool-id const + the name in node goals) and re-generate the agent JSON, so a future `agents push` doesn't revert the fix. The old integration tool is left orphaned (unused) — keep or delete it.
