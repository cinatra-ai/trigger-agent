# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**Cinatra Platform API:**
- Service: Cinatra passthrough API — receives the `trigger_config_set` tool call from the agent's persist node
  - Endpoint: `{{CINATRA_BASE_URL}}/api/agents/passthrough` (POST)
  - Auth: env var `CINATRA_BASE_URL` injected by the Cinatra dispatcher at runtime
  - Defined in: `cinatra/oas.json` (`persist` ApiNode)

**Cinatra Marketplace:**
- Service: `registry.cinatra.ai` — target registry for publishing this extension
  - Triggered by: GitHub Release (tag matching `v<package.json.version>`)
  - Submit flow: extension-submit-for-review → approve → promotion saga
  - Defined in: `.github/workflows/release.yml`

## Data Storage

**Databases:**
- `agent_run_triggers` — Cinatra platform database table; the `trigger_config_set` tool upserts a row keyed by `agent_run_id`
  - The agent NEVER writes directly to this table; all writes go through `trigger_config_set` to enforce actor-aware authorization via `setRunTriggerForActor` (documented in `skills/trigger/SKILL.md`)
  - Connection: managed entirely by the Cinatra platform; not configured in this repo

**File Storage:**
- Not applicable

**Caching:**
- Not applicable; note that calling `trigger_config_set` more than once per run wastes a BullMQ scheduler slot (documented constraint in `skills/trigger/SKILL.md`)

## Authentication & Identity

**Auth Provider:**
- Cinatra platform dispatcher — injects `cinatra_run_id` and `CINATRA_BASE_URL` at run dispatch time; actor-level authorization is enforced server-side by `setRunTriggerForActor`
- No OAuth, JWT, or API-key handling occurs inside this repo

## Monitoring & Observability

**Error Tracking:**
- Not detected within this repo; handled by the Cinatra platform

**Logs:**
- Not detected; runtime observability is the platform's responsibility

## CI/CD & Deployment

**Hosting:**
- Cinatra Marketplace (`registry.cinatra.ai`); installs into the cinatra monorepo workspace as an optional peer

**CI Pipeline:**
- GitHub Actions — two workflows:
  - `ci.yml` (`.github/workflows/ci.yml`): runs on push/PR to `main`; classifies repo as source mirror or standalone; runs install, typecheck, `npm pack --dry-run`, and `extension-kind-gate.mjs` OAS validation
  - `release.yml` (`.github/workflows/release.yml`): triggered on GitHub Release published or manual `workflow_dispatch` from a tag; delegates to `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`

## Environment Configuration

**Required env vars:**
- `CINATRA_BASE_URL` — Cinatra platform base URL; injected by the dispatcher at agent run time; templated in `cinatra/oas.json`
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — GitHub org secret required for the release workflow to publish to the marketplace

**Secrets location:**
- GitHub org secrets (not stored in this repo)
- `.npmrc` file present in repo root — contents not read

## Webhooks & Callbacks

**Incoming:**
- HITL gates emit INTERRUPT events to the frontend with renderer `@cinatra-ai/trigger-agent:configure` (configure gate) — the frontend renderer collects user input and submits `userResponse` back to the gate
- Defined in: `cinatra/oas.json` (`configure_gate` InputMessageNode, `a2uiSurfaceId: "trigger-agent:configure-gate:input"`)

**Outgoing:**
- POST to `{{CINATRA_BASE_URL}}/api/agents/passthrough` with `tool: "trigger_config_set"` — this is the only outgoing call this agent makes (`cinatra/oas.json`, `persist` ApiNode)

## BullMQ Scheduler

- The Cinatra platform uses BullMQ to schedule recurring and one-time triggers persisted via `trigger_config_set`
- Duplicate calls to `trigger_config_set` waste a BullMQ scheduler slot; the tool is idempotent (upsert) but only one call per run should occur (constraint documented in `skills/trigger/SKILL.md`)

---

*Integration audit: 2026-06-09*
