# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **elevenlabs-agents-skills** repository — an operating manual for the ElevenLabs Conversational AI **Agents Platform** (agents, workflows, tools, RAG, guardrails) via the `elevenlabs` CLI and `convai` REST API.

**Repository**: https://github.com/milojarow/elevenlabs-agents-skills

## Repository Structure

```
elevenlabs-agents-skills/
├── .claude-plugin/          # Claude Code plugin configuration
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT License
├── evaluations/             # Test scenarios for the skill
└── skills/
    └── elevenlabs-agents/
        ├── SKILL.md          # Entry point: when to use + the map
        └── reference/        # Depth (progressive disclosure)
```

## The skill

### elevenlabs-agents
The operating manual for the ElevenLabs Agents Platform. Covers agent configuration (hosted LLM model strings, `reasoning_effort`, language/voice/`text_only`, timezone, knowledge base + RAG indexing), workflows as state machines (`start_node`, nodes, edges, the `backward_condition` "go-and-return" mechanism, per-node tool scoping), tools and integration connections (webhook / client / system / api_integration_webhook, header secrets as `{"secret_id"}`, the create→clone→repoint→delete connection-swap procedure), guardrails (retry vs end_call, blocking, client transparency), the operating layer (CLI `agents pull/push`, convai API, draft≠saved + cache), and objective verification via the conversations API (`transcript[].tool_calls[].params_as_json`, `workflow_node_id`) — including why `simulate-conversation` cannot test a workflow.

## Skill Activation

Activates when the task is building, configuring, or debugging an ElevenLabs Conversational AI agent — editing agent config, workflows, tools, RAG, or guardrails; pushing with the `elevenlabs` CLI; calling the `convai` REST API; or diagnosing why a change "didn't take" or a tool/flow misbehaved.

## Updating this skill

After any session that discovers a new ElevenLabs Agents pattern, endpoint, or wall. Keep entries generic — patterns and examples, never client data. The git log of this repo is the diary.
