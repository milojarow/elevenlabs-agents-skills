# Workflows (the state machine)

A workflow turns an agent into a **state machine**: `nodes` (states, each with its own prompt + tool scope) and `edges` (transitions the LLM takes when a condition matches). It lives in the agent config as `workflow: { nodes, edges, prevent_subagent_loops }`.

## Nodes

Each node is keyed by an id. Types seen: `start` (entry — the start node id must be `start_node`), `override_agent` (a working state: its own `additional_prompt` + `additional_knowledge_base` + `additional_tool_ids`), `end`, plus routing/transfer types (`dispatch_tool`, `agent_transfer`, `transfer_to_number`).

```jsonc
hub: {
  type: "override_agent",
  label: "Triage",
  additional_prompt: "<the node's instructions — the per-state behavior>",
  additional_knowledge_base: [ … ],
  additional_tool_ids: [],               // TOOLS ARE SCOPED PER NODE — see below
  conversation_config: { agent: { prompt: { built_in_tools: {} } } },
  position: { x: 0, y: 200 },            // REQUIRED (x,y) or the push is rejected
  edge_order: ["e_hub_a", "e_hub_b"],
  auto_advance_after_first_response: false
}
```

- **`position {x,y}` is required** on every node.
- **Per-node tool scoping:** put a node's allowed tool ids in `additional_tool_ids`. A routing/conversation node that shouldn't call tools gets `[]`. The flow's tools are the **union across nodes** — every tool a node references must exist (a deleted/stale id fails the conversation on node entry; see [operating-cli-api.md](operating-cli-api.md)).
- The **node prompt is where the per-state logic lives** — including pinning a value the LLM must not guess (see [tools-and-integrations.md](tools-and-integrations.md)).

## Edges + the `backward_condition` round-trip ⚠️

An edge has a `forward_condition` and an optional `backward_condition`:

```jsonc
e_hub_appointment: {
  source: "hub", target: "appointment",
  forward_condition: { type: "llm", label: "Book", condition: "The user wants to book a NEW appointment." },
  backward_condition: { type: "llm", label: "Changed mind", condition: "The user changed their mind or wants to ask something else." }
}
```

- `forward_condition.type`: `unconditional` (always) | `llm` (judged by the model from `condition`) | `expression`.
- **The round trip (go AND return) is ONE edge with a `backward_condition`, NOT two edges.** Adding a second edge between the same ordered pair (e.g. `appointment → hub`) returns **422 "Duplicate edge."** The `backward_condition` on the existing `hub → appointment` edge IS the return path.

So a hub-and-spokes flow: each spoke is reached by a forward edge carrying its own `backward_condition` for "user backed out." Don't model the return as its own edge.

## When a greeting node owns the opening, keep `first_message` EMPTY ⚠️

If a workflow's greeting/entry node is responsible for the opening, putting content in `conversation_config.agent.first_message` backfires two ways:

1. The downstream greeting node treats that `first_message` content as **already said** and dedupes it away, so it is never presented to the user.
2. The embedded chat widget does **not render turn 0** (the `first_message`) at all.

Operators naturally stuff the opening (a menu, a list of options) into `first_message`, then are baffled when the bot never shows it. The fix is counter-intuitive: **empty out `first_message`** and let the workflow node own the opening — it then presents the content reliably on every conversation.

Rule of thumb: use `first_message` only when **no** workflow node owns the opening; if a greeting node owns it, keep `first_message` empty. (Measured: a non-empty `first_message` listing options was deduped by the greeting node and never appeared, and the widget did not paint turn 0; emptying it made the node present the options every time.)

## Routing is internal `transfer_to_agent`

When a workflow takes an edge, the transcript shows a system `transfer_to_agent` tool call (same agent id, `to_node: <target>`). That's normal — it's how the engine moves between nodes. You'll see `notify_condition_*_met` entries with the `edge_id` and `target_node_id` in the tool results.

## Make the node assert its capability (so it doesn't refuse)

A node that owns a capability must SAY so in its prompt, or the model may refuse ("I don't have access to do that") instead of using its tools. If a "manage X" node has the tools but the base/hub prompt implies the agent can't, the model hedges. State plainly in the relevant node prompt: "doing X is your job, with your tools — never tell the user you can't or send them elsewhere."

## A terminal node can short-circuit — put final messages on the WORKING node ⚠️

A terminal node (e.g. a "close"/goodbye node whose only job is to say a closing line before `end`) gets **skipped** when its outgoing edge to `end` is ALREADY satisfied on entry. If the user says "that's all / nothing else" while still in the working node, the flow transitions to "close" — but because the `close → end` condition ("the user has no more questions") is already true, the node fires the transition to `end` (via the routing tool) **without generating its own response**. The `end_call` happens with empty text and the goodbye (or any message that lived in that terminal node) is **never spoken** — even though the node prompt orders it.

Transcript symptom: `agent_metadata.workflow_node_id` shows it DID enter "close", but that node's turns have an empty `message` and only `tool_calls=[notify_condition_*_met]` (the transition), with `termination_reason = "end_call tool was called"`.

**Fix:** don't rely on a later terminal node for critical final messages. Put the message (goodbye, close confirmation) in the **working node's confirmation message** — the node that performed the action (book/manage/etc.), which the model always generates. When the working node confirms the action ("your appointment is booked…"), have THAT message carry the goodbye too. Reinforcing the instruction in the close node doesn't help, because the node is skipped without speaking.

> `end_call` is a global built-in tool; the model calls it as a turn action with no accompanying text when it judges "the user is done."

> Reminder: you can't truly test routing with `simulate-conversation` — it doesn't run the workflow at all. Validate with a real conversation + the transcript's `agent_metadata.workflow_node_id`. See [verification-and-gotchas.md](verification-and-gotchas.md).
