---
name: trigger-agent
description: A single HITL trigger-configuration gate (immediate, scheduled, or recurring) that deterministically persists the result via trigger_config_set. No LLM step; no separate confirm gate.
---

# Trigger Agent

`cinatra/oas.json` compiles to a deterministic flow with no LLM node:
`start → configure_gate → persist → end`. There is no llm-bridge node, so this
file is never consumed as a runtime system prompt — it documents the shipped
flow for maintainers and for anything that reads this package's skill card.

## Inputs

- `cinatra_run_id` — the trigger-agent's run id, injected by the dispatcher.
  Wired `start.cinatra_run_id → persist.agent_run_id`; the persist step writes
  the `agent_run_triggers` row keyed by this id.

## Flow

STEP 1 — Configure (HITL gate, renderer `@cinatra-ai/trigger-agent:configure`):
The gate emits an INTERRUPT to the frontend. The user picks a trigger type and
fills in the schedule via the existing `TriggerScreenClient` form (radio
choices, schedule pickers, prompt-driven AI suggestions). The gate's single
output, `userResponse`, is a JSON-encoded string of the form values
(`triggerType`, `timezone`, and `scheduledAt` or `cronExpression` depending on
`triggerType`).

STEP 2 — Persist (deterministic API call, requires approval):
`userResponse` and `agent_run_id` are forwarded as-is to
`POST {{CINATRA_BASE_URL}}/api/agents/passthrough` with
`tool: "trigger_config_set"` and `input: { runId: agent_run_id, userResponse }`.
This node is `riskClass: "write"` with `approvalPolicy: "always"`, so the
platform's write-approval gate must release before the call executes — nothing
is written to `agent_run_triggers` before that approval. `trigger_config_set`
parses `userResponse` server-side and upserts the row (idempotent).

STEP 3 — Return:
The persist node's outputs (`triggerType`, `scheduledAt`, `cronExpression`,
`timezone`, `enabled`) are wired straight to the flow's `end` node as typed
outputs for downstream nodes.

## Constraints

- There is exactly one HITL gate (`configure`). There is no `confirm` gate and
  no LLM-bridge step — do not reintroduce either without also wiring the
  matching nodes into `cinatra/oas.json`.
- All writes go through `trigger_config_set` (server-side, actor-aware
  authorization via `setRunTriggerForActor`); the deterministic persist node
  never writes to `agent_run_triggers` directly.
- The flow contains a single `persist` node, so it does not intentionally
  schedule duplicate `trigger_config_set` calls; a retried call is still safe
  because the handler is idempotent (upsert).
