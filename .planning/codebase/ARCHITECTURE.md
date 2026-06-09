<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│              Cinatra Platform (host / dispatcher)            │
│   injects cinatra_run_id, mounts renderer, calls agent      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Trigger Agent Flow                         │
│               `cinatra/oas.json` (OAS definition)           │
├──────────────┬──────────────────┬───────────────────────────┤
│  StartNode   │ InputMessageNode │       ApiNode             │
│  `start`     │ `configure_gate` │      `persist`            │
│  (input:     │ (HITL renderer   │  POST /api/agents/        │
│  run_id)     │  configure)      │  passthrough              │
└──────┬───────┴────────┬─────────┴──────────┬────────────────┘
       │                │                    │
       │ cinatra_run_id │ userResponse        │ outputs
       └────────────────┴────────────────────▼
                                    ┌────────────────┐
                                    │   EndNode      │
                                    │  (`end`)       │
                                    │  triggerType   │
                                    │  scheduledAt   │
                                    │  cronExpression│
                                    │  timezone      │
                                    │  enabled       │
                                    └────────────────┘
                                            │
                                            ▼
                              ┌─────────────────────────────┐
                              │  agent_run_triggers (DB)    │
                              │  written via trigger_config_ │
                              │  set (actor-aware upsert)   │
                              └─────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| OAS Flow definition | Declares all nodes, edges, data wiring, HITL metadata | `cinatra/oas.json` |
| StartNode (`start`) | Receives `cinatra_run_id` input from dispatcher; hides it from user UI | `cinatra/oas.json` § `$referenced_components.start` |
| InputMessageNode (`configure_gate`) | Emits INTERRUPT to frontend with renderer `@cinatra-ai/trigger-agent:configure`; collects `userResponse` JSON | `cinatra/oas.json` § `$referenced_components.configure_gate` |
| ApiNode (`persist`) | POSTs `trigger_config_set` call to `{{CINATRA_BASE_URL}}/api/agents/passthrough`; upserts `agent_run_triggers` | `cinatra/oas.json` § `$referenced_components.persist` |
| EndNode (`end`) | Collects and surfaces final outputs | `cinatra/oas.json` § `$referenced_components.end` |
| Agent SKILL.md | LLM system prompt: step-by-step instructions for the orchestrating LLM | `skills/trigger/SKILL.md` |
| Extension-kind gate | Zero-dependency CI validator for OAS retired-primitive scan and BPMN shape | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Cinatra Agent Flow (declarative node graph, HITL-first)

**Key Characteristics:**
- All logic is declared in `cinatra/oas.json` as a directed node graph; no imperative runtime code in the repo itself
- Two HITL gates gate progression: `configure_gate` (user fills trigger form) and an LLM confirm step (described in SKILL.md but not a separate OAS node — the LLM re-reads the collected `userResponse` before calling `trigger_config_set`)
- The `persist` node is an ApiNode that POSTs to the host platform's passthrough endpoint rather than calling an external API directly; write authority is enforced inside `setRunTriggerForActor` on the host
- The flow is a **source mirror** of the cinatra monorepo; it declares no direct `@cinatra-ai/*` dependencies, only optional peerDependencies resolved by the monorepo

## Layers

**Flow definition layer:**
- Purpose: Declares the agent's node graph, data wiring, HITL renderers, and I/O schema
- Location: `cinatra/oas.json`
- Contains: StartNode, InputMessageNode (HITL gate), ApiNode (persist), EndNode, ControlFlowEdges, DataFlowEdges
- Depends on: Cinatra platform runtime (resolves `{{CINATRA_BASE_URL}}` and `$component_ref`)
- Used by: Cinatra dispatcher (mounts and executes the flow)

**LLM instruction layer:**
- Purpose: Step-by-step instructions that guide the LLM orchestrator through the confirm → persist sequence
- Location: `skills/trigger/SKILL.md`
- Contains: Input spec, step descriptions for Configure/Confirm/Persist/Return, field formatting rules, constraints
- Depends on: OAS-defined tool `trigger_config_set`
- Used by: Cinatra LLM orchestrator (injected as system context)

**CI gate layer:**
- Purpose: Validates the OAS for banned retired CRM primitives and validates BPMN shape for workflow extensions
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate` — all pure functions, zero external dependencies
- Depends on: Node.js built-ins only (`fs`, `path`)
- Used by: `.github/workflows/ci.yml` (kind-gates job)

## Data Flow

### Primary Request Path

1. Dispatcher injects `cinatra_run_id` into `start` node (`cinatra/oas.json` StartNode)
2. Control flows `start → configure_gate` (ControlFlowEdge `start_to_configure_gate`)
3. `configure_gate` (InputMessageNode) emits INTERRUPT to frontend; user fills trigger form via renderer `@cinatra-ai/trigger-agent:configure`
4. Gate releases; `userResponse` (JSON string) flows via DataFlowEdge `configure_gate_to_persist_userResponse` to `persist` node
5. `cinatra_run_id` flows via DataFlowEdge `start_to_persist_cinatra_run_id` (as `agent_run_id`) to `persist` node
6. `persist` ApiNode POSTs `{ tool: "trigger_config_set", input: { runId, userResponse }, agent_run_id }` to `{{CINATRA_BASE_URL}}/api/agents/passthrough`
7. Platform's `setRunTriggerForActor` upserts `agent_run_triggers` row
8. Response fields (`triggerType`, `scheduledAt`, `cronExpression`, `timezone`, `enabled`) flow via DataFlowEdges to `end` node and are returned as flow outputs

### LLM Confirm Step (SKILL.md overlay)

The SKILL.md instructs the LLM to interpose a second INTERRUPT (confirm gate) between configure and persist, presenting a read-only summary to the user before calling `trigger_config_set`. This confirm step is described behaviorally (in the LLM prompt) rather than as a separate OAS node.

**State Management:**
- No client-side state; all trigger configuration is persisted in `agent_run_triggers` via the `trigger_config_set` passthrough call
- `userResponse` is a JSON string passed as a single opaque value through the data flow

## Key Abstractions

**InputMessageNode (HITL gate):**
- Purpose: Suspends flow execution and emits an INTERRUPT to the frontend with a named renderer
- Examples: `configure_gate` in `cinatra/oas.json`
- Pattern: `renderer` metadata key selects the frontend React component; `formSchema` describes the expected JSON output

**ApiNode (passthrough persist):**
- Purpose: Calls a platform tool via the passthrough API rather than a direct DB write; enforces actor-aware authorization
- Examples: `persist` in `cinatra/oas.json`
- Pattern: `tool` field in POST body identifies the server-side handler; `riskClass: "write"` + `requiresApproval: true` + `approvalPolicy: "always"` enforce the confirm gate

**extension-kind-gate:**
- Purpose: Self-contained, zero-dependency pre-publish CI sanity checker
- Examples: `extension-kind-gate.mjs`
- Pattern: Pure functions exported for unit testing; `runGate` dispatches by `cinatra.kind`; only Node built-ins used

## Entry Points

**Flow execution:**
- Location: `cinatra/oas.json` — `start_node.$component_ref: "start"`
- Triggers: Cinatra dispatcher spawns the agent run and supplies `cinatra_run_id`
- Responsibilities: Accept run ID, gate on user configure, persist trigger config, return outputs

**CI gate:**
- Location: `extension-kind-gate.mjs` — `main()` (invoked when `process.argv[1]` resolves to the file)
- Triggers: `node extension-kind-gate.mjs --package-root .` in GitHub Actions `kind-gates` job
- Responsibilities: Parse args, dispatch by kind, scan OAS for banned primitives, exit 0/1

## Architectural Constraints

- **No runtime code:** The agent ships no `src/` or compiled JavaScript. All behaviour is declared in `cinatra/oas.json` and interpreted by the Cinatra platform at runtime.
- **Global state:** None. `extension-kind-gate.mjs` is entirely pure (no module-level mutable state).
- **Circular imports:** Not applicable (no module graph beyond Node built-in imports in the gate file).
- **Single persist call:** `trigger_config_set` must be called exactly once per run. The handler is idempotent (upsert), but duplicate calls waste a BullMQ scheduler slot.
- **First-party deps:** No `@cinatra-ai/*` packages in `dependencies`/`devDependencies`; they are optional `peerDependencies` resolved only inside the cinatra monorepo workspace.
- **Registry independence:** CI must pass unauthenticated (before `@cinatra-ai` registry is reachable), so `extension-kind-gate.mjs` uses zero external dependencies.

## Anti-Patterns

### Calling trigger_config_set before confirm gate releases

**What happens:** LLM skips the confirm HITL gate and calls `trigger_config_set` immediately after `configure_gate` releases.
**Why it's wrong:** User never explicitly approves the configuration; the `agent_run_triggers` row is written without consent.
**Do this instead:** Follow SKILL.md step sequence — emit confirm INTERRUPT, wait for user click, then call `trigger_config_set` (`skills/trigger/SKILL.md` lines 19-22).

### Calling trigger_config_set more than once per run

**What happens:** LLM calls `trigger_config_set` a second time (e.g., after a retry).
**Why it's wrong:** Wastes a BullMQ scheduler slot even though the upsert is idempotent.
**Do this instead:** Treat a successful first call as terminal; return the persisted config JSON immediately (`skills/trigger/SKILL.md` lines 41-42, 47).

### Writing directly to agent_run_triggers

**What happens:** Any code path bypasses `trigger_config_set` and writes the DB row directly.
**Why it's wrong:** Skips `setRunTriggerForActor` actor-aware authorization on the host.
**Do this instead:** Always route through the `persist` ApiNode → passthrough endpoint → `trigger_config_set` (`cinatra/oas.json` § `$referenced_components.persist`).

## Error Handling

**Strategy:** Platform-owned. The `persist` ApiNode relies on the Cinatra passthrough endpoint to surface errors. The extension-kind gate returns string arrays of violations and exits with code 1 on any error.

**Patterns:**
- `extension-kind-gate.mjs`: pure functions return `string[]` errors; `runGate` aggregates and `main()` prints + exits
- OAS flow: no explicit error nodes; error handling delegated to the Cinatra runtime

## Cross-Cutting Concerns

**Logging:** Not applicable in the OAS flow (platform handles). CI gate uses `console.log`/`console.error`.
**Validation:** `extension-kind-gate.mjs` validates OAS at CI time (retired-primitive scan). Field-level validation of `triggerType`, `scheduledAt`, `cronExpression`, `timezone` is enforced by SKILL.md formatting rules and by the `formSchema` in the `configure_gate` HITL definition.
**Authentication:** Delegated entirely to the Cinatra platform. The `persist` ApiNode uses `{{CINATRA_BASE_URL}}` with platform-injected credentials; actor-aware authorization is in `setRunTriggerForActor` on the host.

---

*Architecture analysis: 2026-06-09*
