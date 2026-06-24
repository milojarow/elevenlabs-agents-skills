---
name: elevenlabs-agents
description: Use when building, configuring, or debugging an ElevenLabs Conversational AI agent (Agents Platform) — editing agent config, workflows, tools, RAG, or guardrails; pushing with the elevenlabs CLI or convai REST API; or diagnosing why an agent change "didn't take", a tool returns 401, RAG isn't used, or a workflow route/tool misbehaves.
---

# ElevenLabs Agents Platform

Operating manual for the ElevenLabs Conversational AI **Agents Platform** — agents, workflows, tools, RAG, guardrails — driven as code via the `elevenlabs` CLI and the `convai` REST API.

> **⏸️ ACTIVE-SKILL MARKER:** While `elevenlabs-agents` is active — any turn touching ElevenLabs agent config, workflows, tools, RAG, guardrails, the `elevenlabs` CLI, or the `convai` API — begin every reply with ⏸️ so the operator sees the skill is engaged. Omit it on turns that don't touch the domain.

## Overview

An ElevenLabs agent is configured data: a system prompt, an LLM, a voice/text setting, a knowledge base, optional guardrails, and an optional **workflow** (a state machine of nodes + edges). Manage it **as code** — keep the agent JSON in a repo, transform it with a build script, and `agents push`.

Two principles save most lost hours:

1. **The dashboard shows a browser *draft*; the API returns the *saved* version; the embedded widget caches config at page load.** "My change didn't take" is almost always one of these — not a bug. → [reference/operating-cli-api.md](reference/operating-cli-api.md)
2. **Verify behavior against the conversation transcript, never the agent's self-report — and `simulate-conversation` does NOT run the workflow.** → [reference/verification-and-gotchas.md](reference/verification-and-gotchas.md)

## When to use

- Creating/editing agent config: model, `reasoning_effort`, language, voice, `text_only`, first message, timezone.
- Wiring a **knowledge base + RAG** — especially when the agent ignores the docs or the KB shows "RAG Storage: 0 B".
- Building a **workflow** (hub + branches), routing, or a node that must return to a previous node.
- Adding **tools** (webhook / client / system / integration such as Cal.com) — or a tool returns 401, or its connection points at the wrong account.
- Setting **guardrails** and deciding `retry` vs `end_call`.
- Pushing with the CLI when the change "doesn't take", or verifying what the LLM actually did in a conversation.

**Not for:** ElevenLabs TTS / voice cloning / dubbing / sound-effects / speech-to-text — those are different products. This skill is the **Agents / Conversational-AI** platform only.

## The map

| Concern | Reference |
|---|---|
| Agent config: model strings, `reasoning_effort`, voice/text, **KB + RAG indexing** | [reference/agent-configuration.md](reference/agent-configuration.md) |
| Tools (4 types), **header secrets `{secret_id}`**, integration **connection lifecycle** | [reference/tools-and-integrations.md](reference/tools-and-integrations.md) |
| Workflows: nodes, edges, **`backward_condition`**, per-node tool scoping | [reference/workflows.md](reference/workflows.md) |
| Guardrails: focus/content/custom, **retry vs end_call**, client transparency | [reference/guardrails.md](reference/guardrails.md) |
| Operating: **CLI push/pull, convai API, draft≠saved + cache** | [reference/operating-cli-api.md](reference/operating-cli-api.md) |
| **Objective verification**, simulate's blind spot, cross-account integration trap | [reference/verification-and-gotchas.md](reference/verification-and-gotchas.md) |
| Embedding the chat/voice **widget** on a site | [reference/widget-embedding.md](reference/widget-embedding.md) |
| **WhatsApp channel**: import flow is broken → bridge your own Cloud API transport | [reference/whatsapp-channel.md](reference/whatsapp-channel.md) |
| **Procedures** (alpha): no headless authoring path, prefer the prompt | [reference/procedures.md](reference/procedures.md) |

## Quick reference

- **REST auth:** header `xi-api-key: <key>`. Base `https://api.elevenlabs.io/v1/convai/...`.
- **CLI (agents as code):** `elevenlabs agents pull --agent <id>` → edit the agent JSON → `elevenlabs agents push`. Push is a **full override** (drafts don't survive). Interactive prompt → `printf 'y\n' | elevenlabs agents push`. Auth via `ELEVENLABS_API_KEY` env.

| Need | Endpoint |
|---|---|
| List / read / update agent | `GET`/`PATCH` `/convai/agents[/{id}]` |
| CRUD a tool | `/convai/tools[/{id}]` (`POST` create, `PATCH` update, `DELETE`) |
| Create an integration connection | `POST /convai/api-integrations/{id}/connections` |
| List conversations / read a transcript | `GET /convai/conversations[?agent_id=…][/{id}]` |
| Simulate (base prompt only — NOT the workflow) | `POST /convai/agents/{id}/simulate-conversation` |
| Model catalog + deprecations | `GET /convai/llm/list` |

**Hosted LLM model strings** rotate — confirm live with `/convai/llm/list`. Seen: `qwen35-397b-a17b` ("great for agentic"), `gpt-oss-120b`; older `glm-*` get deprecated. Claude/Gemini/OpenAI models also selectable. `reasoning_effort`: `none|minimal|low|medium|high|xhigh`.

## Common mistakes

- **"Pushed, but the dashboard/agent still shows the old config."** Draft≠saved; the widget caches at page load. Hard-reload (F5) the dashboard/widget; the API `GET` is the source of truth. → operating-cli-api.md
- **Webhook tool 401 with `X-Site-Key: {{my_secret}}`.** Header secrets are a structured `{"secret_id":"…"}`, not a `{{name}}` string (which is sent literally). Confirm via `secret.used_by.tools`. → tools-and-integrations.md
- **"RAG enabled but the agent ignores the docs / RAG Storage 0 B."** Uploading ≠ indexing, and docs must be attached in the **agent's** KB with RAG on. → agent-configuration.md
- **422 "Duplicate edge" when a node should return to the hub.** Don't add a second edge between the same pair; put a `backward_condition` on the existing forward edge. → workflows.md
- **`simulate-conversation` shows the agent refusing / not using tools.** It runs only the base prompt, not the workflow (so no per-node tools). Test a workflow with a real conversation, then read the transcript. → verification-and-gotchas.md
- **Integration tool: creating works but find/list/cancel returns empty.** The connection authenticates as one account; a public resource is bookable cross-account but listing is account-scoped. Verify the connected account. → tools-and-integrations.md
- **A small hosted LLM guesses a parameter it shouldn't** (e.g. an id). Pin the value in the node prompt — the schema `constant_value` doesn't persist on integration tools. → workflows.md
- **WhatsApp number-import never shows your self-registered Cloud API number.** ElevenLabs' embedded-signup import doesn't ingest a number you registered yourself to Meta's Cloud API (and has no headless path). Run your own Cloud API transport and bridge to the agent over its WebSocket. → whatsapp-channel.md
