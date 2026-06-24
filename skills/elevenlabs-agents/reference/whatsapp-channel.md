# WhatsApp as a channel for an agent

ElevenLabs offers a WhatsApp **number-import / embedded-signup** flow to bring a WhatsApp Business number under a Conversational AI agent. Treat that flow as unreliable for a self-managed Cloud API number — and architect the integration so you don't depend on it.

## The number-import / embedded-signup flow does NOT ingest a self-registered Cloud API number ⚠️

If you registered a WhatsApp Business number yourself against Meta's Cloud API — i.e. you flipped it to `CONNECTED` / `CLOUD_API` in Meta via `POST /{phone_number_id}/register` — ElevenLabs' import flow never surfaces it in the import list. Observed reproducibly with multiple real numbers across separate verified Business portfolios: the number does not appear, neither passively over time nor on a re-import attempt. This is an ElevenLabs backend limitation, not a misconfiguration on your side.

There is also **no scripted/headless path** to the import: the import API requires the embedded-signup OAuth `code` (a `token_code`) that is only obtainable through the interactive embedded-signup browser flow.

## Workaround — run your own WhatsApp transport, keep the agent as the brain

Don't make ElevenLabs the WhatsApp transport. Instead:

- Run your **own WhatsApp Cloud API transport** in your backend — Meta Graph API for outbound sends plus inbound webhooks for received messages.
- Keep the ElevenLabs agent as the conversational **brain**, driven over its WebSocket (signed-URL → `wss://…`; see [verification-and-gotchas.md](verification-and-gotchas.md) for the headless WebSocket protocol).
- Your backend is the **bridge**: inbound WhatsApp message → user turn into the agent socket; agent reply → outbound WhatsApp send.

This sidesteps the broken import entirely and gives you full control of the transport. For the Cloud API transport side (registration, sending, webhooks), see the `whatsapp-cloud-api` skill.
