# Embedding the widget

Drop the conversation widget on any site with a web component + a script tag:

```html
<elevenlabs-convai agent-id="<agent-id>"></elevenlabs-convai>
<script src="https://unpkg.com/@elevenlabs/convai-widget-embed" async type="text/javascript"></script>
```

This renders a floating launcher. For a **text** agent, the same widget works as a chat box (the agent's `conversation.text_only` drives chat-vs-voice).

> Public-agent caveat: the embeddable widget needs a public agent with auth disabled. For authenticated/in-app flows, use the SDKs (`signed-url` attribute instead of `agent-id`).

## The attributes you'll actually set

| Attribute | Use |
|---|---|
| `agent-id` / `signed-url` | which agent (signed-url for auth'd flows) |
| `variant` | `compact` (floating button) or `expanded` |
| `action-text` / `start-call-text` / `end-call-text` | button + tooltip copy |
| `avatar-image-url`, `avatar-orb-color-1/2` | branding |
| `dismissible`, `disable-banner` | let users minimize / hide "Powered by ElevenLabs" |
| `server-location` | `us` \| `eu-residency` \| `in-residency` \| `global` |

Size/position via the exposed CSS custom properties on the `elevenlabs-convai` element (`--elevenlabs-convai-widget-width/height`) + normal `position`/`z-index`. The full attribute list is in the ElevenLabs widget docs — don't memorize it, look it up when styling.

## Passing context in: dynamic variables

The embedding page can pass **custom dynamic variables** to the agent (referenced as `{{var}}` in the prompt) — e.g. a logged-in user's id/email so the agent can skip a verification step. Declare custom vars in `conversation_config.agent.dynamic_variables.dynamic_variable_placeholders`; system vars (`{{system__*}}`) are automatic (see [agent-configuration.md](agent-configuration.md)). A `client` tool is the other half of this — it lets the agent ask the page a question at runtime ("is this user logged in?").

## File uploads

Embedded chat accepts image/PDF uploads when the agent uses a multimodal LLM and `conversation_config.conversation.file_input.enabled` is on; cap with `max_files_per_conversation`.

## Caveat that ties back to operating

The widget loads the agent config **at page load** and caches it for that session. After you push a config change, an open widget keeps the old config — **reload the page** before re-testing. This is the same cache trap described in [operating-cli-api.md](operating-cli-api.md).
