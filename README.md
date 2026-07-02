# Trigger Agent

Decide when a workflow runs before it starts. Add this agent at the beginning of any Cinatra workflow and your users get a small picker — run now, run once at a specific time, or run on a recurring schedule. Nothing is scheduled until the write step is approved.

## Works with

- Any Cinatra workflow that needs a configurable run schedule before its first step

## Capabilities

- Present a trigger picker (immediate, one-time scheduled, or recurring) via an interactive HITL gate
- Capture a date, time, and IANA timezone for a scheduled run; accept a plain-language prompt or a 5-field cron expression for recurring runs
- Gate the persist step on the platform's write-approval flow (`riskClass: "write"`, `approvalPolicy: "always"`) before writing anything
- Persist the trigger to the parent workflow run via `trigger_config_set`; the flow has one persist node and the call is idempotent (upsert) on retry
- Return `triggerType`, `scheduledAt`, `cronExpression`, `timezone`, and `enabled` as typed flow outputs for downstream nodes
- Install: add `@cinatra-ai/trigger-agent` to your workflow extension's `cinatra.dependencies` array in `package.json`
- Configure: no external credentials required; pass `cinatra_run_id` (injected automatically by the Cinatra dispatcher) as the only input; `timezone` defaults to `"UTC"` when omitted
- Usage example: input `cinatra_run_id` (from the dispatcher); choose `triggerType: "scheduled"`, supply `scheduledAt: "2026-07-01T09:00:00Z"` and `timezone: "America/New_York"`; outputs `triggerType`, `scheduledAt`, `timezone`, and `enabled: true`; failure mode — an empty `runId` causes `trigger_config_set` to reject
- API contract: inputs — `cinatra_run_id` (string); outputs — `triggerType` (`"immediate"` | `"scheduled"` | `"recurring"`), `scheduledAt` (ISO 8601 UTC string, when `triggerType` is `"scheduled"`), `cronExpression` (5-field cron string, when `triggerType` is `"recurring"`), `timezone` (IANA identifier), `enabled` (boolean)
- Develop: run `node extension-kind-gate.mjs --package-root .` to catch manifest and README violations before publishing; no build step is required — the agent ships its OAS and skill definition as static JSON and Markdown
- Troubleshoot: if the configure gate never appears, verify that `cinatra_run_id` is wired from the start node in the parent flow definition; if `trigger_config_set` is called with an empty `runId`, the agent is running outside a Cinatra orchestration context
