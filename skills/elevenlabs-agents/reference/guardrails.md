# Guardrails

Guardrails live under `platform_settings.guardrails`. They constrain what the agent may say/do. The two decisions that matter: **which kind** of guardrail, and **what it does when it fires** (`retry` vs `end_call`).

## Kinds

| Kind | What it catches |
|---|---|
| `focus` | drift off the agent's purpose |
| `prompt_injection` | user attempts to override instructions |
| `content` | built-in categories (each with an `is_enabled` + threshold) — e.g. `medical_and_legal_information` |
| `custom` | your own LLM-judged rules, in `custom.config.configs[]` |

### Custom rule shape

```json
{
  "is_enabled": true,
  "name": "Stay on the product",
  "prompt": "Block ONLY if the agent does X. It is always ALLOWED to do Y and Z (the agent's job).",
  "execution_mode": "blocking",
  "trigger_action": { "type": "retry", "feedback": "Steer back to Y/Z; for X, suggest a professional." }
}
```

Write the rule **inclusively**: spell out what is explicitly ALLOWED (the agent's legitimate job) and block only the narrow bad case — otherwise a broad built-in category will muzzle normal, on-topic answers.

## `retry` vs `end_call` — the decision

| `trigger_action.type` | Effect | Requires | Client sees |
|---|---|---|---|
| **`retry`** | Silently regenerates the response with `feedback` steering it | `execution_mode: "blocking"` | nothing — a transparent re-generation |
| **`end_call`** | Terminates the session | — | the chat simply ends |

**The client never sees a "GUARDRAIL TRIGGERED" message either way.** With `retry` the user just gets a better answer; with `end_call` the conversation abruptly stops (and the user has no idea why).

**For a customer-facing bot, prefer `retry` + `blocking`.** `end_call` on a content rule will cut off real customers mid-conversation — use it only for genuine abuse, not for "stay on topic." A built-in content category set to `end_call` is a common cause of "the bot just hangs up on people."

## Common tuning

- A built-in category (e.g. `medical_and_legal_information`) over-triggers and blocks legitimate domain talk (an insurance agent explaining tax-deductibility, a clinic bot describing a service). **Disable the built-in category** and replace it with a tuned **custom** rule that allows the legitimate cases and blocks only the real risk (a definitive diagnosis, legal advice on a specific case).
- Keep rules few and inclusive. Each rule is an LLM judgment call on every turn; vague rules cause false positives the user can't see.
