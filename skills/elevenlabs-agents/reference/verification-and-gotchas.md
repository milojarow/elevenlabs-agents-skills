# Objective verification + the traps that waste hours

The agent's chat text is a *claim*. To know what actually happened, read the **conversation transcript** from the API. And know the one tool that looks like a test but isn't.

## Verify against the conversation, not the chat

```bash
# most recent conversation for an agent
GET /convai/conversations?agent_id=<id>&page_size=1
#   → .conversations[0].conversation_id, .status (done|failed|in-progress), .call_successful

# the transcript (the ground truth)
GET /convai/conversations/{conversation_id}
```

In the transcript, per turn:

- `transcript[].role` — `user` | `agent`
- `transcript[].message` — the text
- `transcript[].tool_calls[]` — each has `.tool_name` and **`.params_as_json`** = the EXACT arguments the LLM emitted. This is how you prove what the model really sent (e.g. which id it passed), not what it said in chat.
- `transcript[].tool_results[]` — `.tool_name` + the response your endpoint returned (status, body). This is where you see a `200 {ok:true}` vs a `4xx`.
- `transcript[].agent_metadata.workflow_node_id` — which workflow node produced the turn (`null` means no workflow ran — see below).
- top-level `.metadata.termination_reason` — on a `failed` conversation, the cause (e.g. a referenced tool id that no longer exists).

**`in-progress` conversations have an empty/partial transcript** — it finalizes when the session ends. If you need to inspect it, the human must end the chat first.

## simulate-conversation does NOT run the workflow ⚠️

`POST /convai/agents/{id}/simulate-conversation` (body `{simulation_specification:{simulated_user_config:{first_message, language, prompt:{prompt}}}}`, plus `new_turns_limit`) returns `{analysis, guardrails_result, simulated_conversation}`.

**It runs ONLY the base agent prompt — not the workflow state machine.** Proof: in a simulated turn `agent_metadata.workflow_node_id` is `null`; in a REAL conversation it is the node name. Because tools are scoped per workflow node and the base agent's `tool_ids` is often empty, the simulated agent has **no tools** and will improvise — refuse, claim "no access", or never call a tool.

This is NOT "the simulator mocks/skips tool execution" — the entire state machine is bypassed. So:

- **simulate is useful only** for testing the base system prompt / persona / a tool-less agent.
- **To test a workflow** (routing, per-node tools, an OTP/verify/find/cancel chain), run a **real conversation** (widget or client) and read the transcript. There is no shortcut.

## Headless real conversation via WebSocket (how to test without a browser)

Since `simulate-conversation` bypasses the workflow, a *real* conversation is the only way to exercise routing, per-node tools and RAG — and you can drive one headless over a WebSocket, no browser:

```
GET /convai/conversation/get-signed-url?agent_id=<id>   (header xi-api-key)
→ {"signed_url":"wss://..."}  → open the WebSocket
```

Protocol observed (text-only agent):

- On open, send `{"type":"conversation_initiation_client_data"}`.
- Server sends `conversation_initiation_metadata` → carries `conversation_id` (note it to read the transcript afterward).
- User turn: `{"type":"user_message","text":"..."}`.
- **A text-only agent's reply arrives as `agent_chat_response_part`** (start/delta/stop streaming), NOT only as `agent_response` — a client that listens only for `agent_response` misses the text. Accumulate the deltas.
- Tool calls surface as `agent_tool_response` with `{tool_name, is_called, is_error, is_blocked}` — this is the proof the webhook actually fired.
- Keepalive: reply to `{"type":"ping"}` with `{"type":"pong","event_id":...}`.

A ~60-line runner with no deps suffices: Bun has a native WebSocket, so `bun run` (or `podman run --rm oven/bun:<tag>` if the host lacks bun) can script the turns, capture events and assert the tool fired. Close on a hard timeout (~90s).

**Caveat:** a real conversation runs the REAL webhooks. If a tool sends email or writes a production DB, redirect the destination before the test and clean up after.

## The cross-account integration trap

An integration tool (Cal.com and similar) authenticates as **one connected account**. A *public* resource can be acted on across accounts, but **listing/managing is account-scoped**. Classic symptom: **"creating works, but finding/listing/cancelling returns empty."**

What's really happening: the create call succeeds because the target resource is public (the created item lands in the resource owner's account); the list call returns nothing because the connection's account isn't the owner. Diagnosis — compare what each credential sees:

```bash
# for each API key, who is it and what does it see?
GET <provider>/me                      # account identity (username / email / id)
GET <provider>/<list-endpoint>?...     # does THIS account return the item?
```

If two keys you believed were "the same account" return different identities, that's the bug — the integration connection is wired to the wrong account. Fix: re-create the connection on the correct account (see [tools-and-integrations.md](tools-and-integrations.md) — you cannot change a tool's connection in place; you clone the tools onto a new connection and repoint).

## Pin parameters a small model will otherwise guess

A small hosted LLM will invent a value for a tool parameter even when the parameter's description says "do not guess" — e.g. it sent `eventTypeId: 1` (a public demo id) instead of the real one, silently booking against the wrong resource. On `api_integration_webhook` tools the schema's `constant_value` does **not** persist (the schema is owned by the integration). The reliable fix for a 397B-class model: **state the exact value in the workflow node's prompt** ("the X id is ALWAYS 12345 — use it, never guess"). Verify it took by reading the tool call's `params_as_json` in the next conversation. For weaker models, the bulletproof fallback is to rebuild the tool as a plain `webhook` tool with the value hardcoded in the URL/body so the LLM can't supply it at all.

### It's the model SIZE, not the tool and not `reasoning_effort`

Observed in an A/B: a large model (e.g. `qwen35-397b-a17b`, even with `reasoning_effort: low`) does NOT invent a `required` param value even on a "permissive" tool (param left to the LLM, no `constant_value`/`dynamic_variable`). The SAME tool, with a small/"mini" model, DOES produce invented placeholders. The integration tool doesn't validate the value — a `required` param without a `constant_value`/`dynamic_variable` is the LLM's responsibility either way (confirm which params are the model's via `GET /convai/tools/{id}` → see [tools-and-integrations.md](tools-and-integrations.md)).

So if your agent hallucinates params:

- **It's not the tool — it's the model.** Either (a) move to a more capable model, or (b) replace the integration tool with your own **`webhook` tool whose backend DERIVES/validates the param** (from conversation state or an already-captured datum) instead of trusting the LLM's arg. Backend derivation makes even a small model reliable, and is sturdier than hoping the model "behaves." (Full wrap procedure in [tools-and-integrations.md](tools-and-integrations.md).)
- **`reasoning_effort` is NOT the lever.** The large model that didn't invent ran at `low`; the mini that did invent ran at the provider default `medium`. Tool-calling reliability tracks model size/quality (and backend validation), not the reasoning level — don't "turn up the reasoning" expecting it to stop inventing arguments.

## IDN / punycode for SMTP-backed webhook tools

If a webhook tool (or its backing service) authenticates SMTP for a mailbox on an **IDN domain** (ñ / non-ASCII), the SMTP username must be the **punycode** form (`user@xn--…`), not the Unicode form, or auth fails (`535`). The display name can stay Unicode.
