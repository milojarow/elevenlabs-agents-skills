# elevenlabs-agents-skills

**Operating manual for the ElevenLabs Conversational AI Agents Platform — build, configure, and debug agents as code.**

## What is this?

A Claude Code skill for the ElevenLabs **Agents Platform** (Conversational AI): agents, workflows (state machines), tools (webhook / client / system / api_integration), knowledge base + RAG, and guardrails — operated through the `elevenlabs` CLI and the `convai` REST API. It encodes the field-tested way to manage an agent as code (a build script + push), plus the walls that are easy to hit and hard to diagnose.

### Why this skill exists

- **The dashboard lies to your eyes.** Edits live in a browser *draft*; the API returns the *saved* version; the widget caches the config at page load. Most "my change didn't take" reports are this, not a bug.
- **`simulate-conversation` does NOT run the workflow** — it runs only the base prompt, so it can't test routing, per-node tools, or anything in your state machine. The objective truth lives in the *conversations* API.
- **Secrets in a tool header are a structured `{"secret_id": "..."}`, not a `{{name}}` string** — the string form is sent literally and 401s.
- **Integration tools (Cal.com, etc.) authenticate as a specific connected account** — public resources can be created cross-account while listing/managing is account-scoped, which produces "creating works, finding doesn't" puzzles.
- **A small hosted LLM will guess a parameter it shouldn't** unless you pin the value in the node prompt — the schema's `constant_value` doesn't persist on integration tools.

## The skill

| Skill | Description |
|-------|-------------|
| **elevenlabs-agents** | Configure agents (models, RAG, voice/text), build workflows with `backward_condition` routing, wire tools and integration connections, set guardrails, operate via CLI + convai API, and verify behavior objectively against the conversations transcript. |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/elevenlabs-agents-skills
```

Then install:

```
/plugin → Discover → elevenlabs-agents-skills → Install
```

## Requirements

- An ElevenLabs account with Agents Platform access and an API key (header `xi-api-key`).
- The `@elevenlabs/cli` (`elevenlabs`) for the agents-as-code flow (`agents pull` / `agents push`).

## License

MIT
