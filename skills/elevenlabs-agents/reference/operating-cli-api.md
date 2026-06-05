# Operating: CLI, convai API, and the draft≠saved trap

Two ways to drive an agent: the **`elevenlabs` CLI** (agents as code — preferred) and the **`convai` REST API** (everything, including what the CLI doesn't expose). Plus the single most common confusion: **what you see ≠ what is saved ≠ what a live conversation uses.**

## REST API basics

- Base: `https://api.elevenlabs.io/v1/convai/...`
- Auth: header `xi-api-key: <key>`. (Not `Authorization: Bearer`.)
- Errors: HTTP 4xx/5xx with `{"detail": ...}` (sometimes a string, sometimes a FastAPI-style validation array `[{"type","loc","msg","input"}]` — read `.detail` first).

Useful endpoints:

| Need | Method + path |
|---|---|
| List / read / update agent | `GET`/`PATCH` `/agents[/{agent_id}]` |
| Tool CRUD | `GET`/`POST`/`PATCH`/`DELETE` `/tools[/{tool_id}]` |
| Secrets (list / create / update / delete) | `/secrets[/{secret_id}]` |
| Integration metadata (credential schema + tool templates) | `GET /api-integrations/{integration_id}` |
| Create an integration connection | `POST /api-integrations/{integration_id}/connections` |
| Conversations (list / transcript) | `GET /conversations[?agent_id=…][/{conversation_id}]` |
| Simulate (base prompt only) | `POST /agents/{id}/simulate-conversation` |
| Model catalog | `GET /llm/list` |

## The CLI — agents as code

```bash
npm i -g @elevenlabs/cli          # provides `elevenlabs`
export ELEVENLABS_API_KEY=<key>   # CLI auth
elevenlabs agents pull --agent <agent_id>   # writes agent_configs/<Name>.json + agents.json (id/version/branch map)
# …edit the JSON (ideally via a build/transform script, see below)…
elevenlabs agents push            # uploads the local JSON
```

- **Push is a FULL OVERRIDE.** It uploads the whole local config and overwrites the saved agent. Anything that existed only as a UI draft is **lost**. There is no merge.
- **Interactive prompt:** push asks to confirm ("Will push (force override)"). In a non-interactive/headless context, feed it: `printf 'y\ny\n' | elevenlabs agents push`.
- **`agents.json`** maps each `agent_configs/*.json` to its `id` + `version_id` + `branch_id`. Keep it in the repo.

### Build-script-as-source-of-truth pattern

For any non-trivial agent, don't hand-edit the pulled JSON. Keep a **transform script** that reads the pulled config, applies your config (prompt from a `.txt`, model, KB, workflow, guardrails) and writes it back. Then `push`. Benefits: the agent's full intended state is reproducible and in git; a push always reflects the complete intent (so the "drafts lost on push" risk is moot — there are no drifting UI drafts to protect). The system prompt lives in its own file the script reads in.

## draft ≠ saved ≠ what a conversation uses — THE trap

Three distinct states. Confusing them is the #1 "my change didn't take":

| State | What it is | How to read it |
|---|---|---|
| **UI draft** | Unsaved edits in the dashboard browser session | only in that browser tab until published / discarded |
| **Saved version** | The persisted agent config | `GET /agents/{id}` — **the source of truth** |
| **What a live conversation uses** | The config the widget/test-panel loaded **at page load** | re-established only on a fresh page load |

Consequences and fixes:

- **You `push` via API/CLI, but the dashboard still shows the old config.** The dashboard is showing its stale draft/cached view. The API `GET` already has your change. **Hard-reload (F5 / Ctrl-Shift-R)** the dashboard; "Discard changes" if it offers.
- **A conversation runs against an old config even after you pushed.** The widget/test-panel cached the config when the page loaded. Starting a "new chat" in the *same loaded page* reuses the cached config. **Reload the page**, then start the conversation. (Symptom seen in the wild: a conversation fails with `Documents with ids {…} not found` because the cached config referenced tools you have since deleted.)
- **The operator edits in the UI while you manage via the repo.** Pick ONE source of truth. If the repo is canonical, tell the operator not to UI-edit, and to F5 after each push. A push will clobber their UI draft silently.

**Rule of thumb when verifying a change landed:** trust `GET /agents/{id}` (saved), not the dashboard, and have the human hard-reload before re-testing a conversation.
