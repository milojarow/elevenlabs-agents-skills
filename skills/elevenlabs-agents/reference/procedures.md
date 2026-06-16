# Procedures (alpha) — no headless authoring path yet

**Procedures** are a per-task behavior feature (an "employee handbook"): instructions that load when a user's input matches the procedure's `trigger`. The feature is **alpha** ("expect breaking changes").

## There is no headless REST authoring path ⚠️

- **No REST list/CRUD endpoint:** `GET /v1/convai/procedures` returns **404**. There is no documented REST or CLI path to create/read/update procedures programmatically.
- **Documented authoring is dashboard-only:** the **Procedures tab**, or **SOP import** (PDF/DOCX/TXT/MD, max 20 MB, up to 10 procedures per import).
- The agent config **does** carry a top-level `procedures: {}` field that stores them, but its schema is **undocumented and explicitly alpha** — PATCHing it blind is risky on a production agent.
- A procedure's components: `name` (a dashboard label, **not** passed to the LLM), `trigger` (when it applies), `content` (markdown instructions), and inline refs to tools / KB / other procedures.
- Built-in tools `procedure_create` / `procedure_update` / `procedure_delete` (null by default in the config) let the agent author its **own** procedures at runtime (a self-improving paradigm) — distinct from authoring them by hand.

## Prefer the prompt for now

If you manage agents as code, Procedures cannot (yet) be part of that pipeline — no headless authoring path, and the config field is an alpha schema you should not PATCH on a revenue-taking agent.

For a reactive, intent-scoped behavior that conceptually fits a Procedure, **encode it directly in the agent prompt** instead. A prompt implementation passed the full eval battery (7/7, including the two hardest scope scenarios), while Procedures offered no headless path and an undocumented alpha schema. Trade-off: don't risk a production agent on PATCHes to an undocumented alpha schema when a prompt already nails the behavior.

> Verified against the live API + docs: `GET /v1/convai/procedures` returned 404; documented authoring was dashboard/SOP-import only; the `procedures` config field was undocumented/alpha; the `procedure_*` built-in tools were observed null in the config.
