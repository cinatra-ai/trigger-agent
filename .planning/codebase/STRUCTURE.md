# Codebase Structure

**Analysis Date:** 2026-06-09

## Directory Layout

```
trigger-agent/
├── cinatra/
│   └── oas.json          # Agent flow definition (nodes, edges, HITL metadata, I/O schema)
├── skills/
│   └── trigger/
│       └── SKILL.md      # LLM system prompt: step-by-step agent instructions
├── .github/
│   └── workflows/
│       ├── ci.yml        # Standalone CI: classify, install, typecheck, test, pack, kind-gates
│       └── release.yml   # Release workflow (publish to registry)
├── extension-kind-gate.mjs  # Zero-dependency CI gate: OAS primitive scan + BPMN shape check
├── package.json          # Cinatra agent manifest (name, version, cinatra.kind: "agent")
├── tsconfig.json         # TypeScript config (targets src/ — currently no src/ files)
├── .npmrc                # npm registry config
└── LICENSE               # Apache-2.0
```

## Directory Purposes

**`cinatra/`:**
- Purpose: Cinatra platform sidecar artifacts
- Contains: `oas.json` — the agent's OAS (OpenAgent Spec) flow definition, consumed directly by the Cinatra dispatcher at runtime
- Key files: `cinatra/oas.json`

**`skills/trigger/`:**
- Purpose: LLM instruction skill for the trigger agent
- Contains: `SKILL.md` — the system prompt injected into the LLM orchestrator; describes inputs, 4-step sequence, field formatting rules, and constraints
- Key files: `skills/trigger/SKILL.md`

**`.github/workflows/`:**
- Purpose: GitHub Actions CI/CD
- Contains: `ci.yml` (build, typecheck, test, pack dry-run, kind-gates), `release.yml` (publish)
- Key files: `.github/workflows/ci.yml`, `.github/workflows/release.yml`

## Key File Locations

**Entry Points:**
- `cinatra/oas.json`: Flow entry point — `start_node.$component_ref: "start"`. The Cinatra dispatcher mounts this file to execute the agent.
- `extension-kind-gate.mjs`: CI entry point — `main()` function, invoked via `node extension-kind-gate.mjs --package-root .` in the `kind-gates` CI job.

**Configuration:**
- `package.json`: Agent manifest. Declares `cinatra.apiVersion`, `cinatra.kind: "agent"`, package name `@cinatra-ai/trigger-agent`, version, and empty `dependencies`.
- `tsconfig.json`: TypeScript compiler config targeting `src/` with strict mode (no `src/` currently exists; present for future expansion or monorepo integration).
- `.npmrc`: npm/pnpm registry configuration. (Contents not read — may contain auth tokens.)

**Core Logic:**
- `cinatra/oas.json`: All agent behaviour — node graph, HITL gate renderer bindings, data wiring, I/O schemas, approval policies.
- `skills/trigger/SKILL.md`: All LLM instructions — step sequence, field rules, constraints.

**Testing / CI:**
- `extension-kind-gate.mjs`: Exports pure validation functions (`validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflowPackageShape`) testable in isolation.
- `.github/workflows/ci.yml`: Runs kind-gates job with `node extension-kind-gate.mjs --package-root .`

## Naming Conventions

**Files:**
- Cinatra sidecar artifacts: `cinatra/<artifact>` — e.g., `cinatra/oas.json`, (for workflow extensions) `cinatra/workflow.bpmn`
- Skills: `skills/<skill-name>/SKILL.md` — uppercase SKILL.md, kebab-case skill directory
- CI gate script: `extension-kind-gate.mjs` — kebab-case, `.mjs` ES module extension
- GitHub Actions: `.github/workflows/<name>.yml`

**Directories:**
- `cinatra/`: Always lowercase, holds platform sidecar files
- `skills/`: Lowercase; one subdirectory per skill named after the skill

## Where to Add New Code

**New agent step (additional OAS node):**
- Add node definition to `cinatra/oas.json` under `$referenced_components`
- Add control-flow and data-flow edges in the `control_flow_connections` and `data_flow_connections` arrays

**Updated LLM instructions:**
- Edit `skills/trigger/SKILL.md` only

**New skill variant:**
- Create `skills/<skill-name>/SKILL.md`

**TypeScript source (future):**
- Place under `src/` — `tsconfig.json` already targets `src/**/*.ts` and `src/**/*.tsx`
- Tests would live alongside source or in a `tests/` directory (no test framework currently configured)

**Extended CI gate rules:**
- Add banned primitives or typehints to `BANNED_PRIMITIVES` / `BANNED_TYPEHINTS` arrays in `extension-kind-gate.mjs`
- Export new pure validator functions; add dispatch in `runGate`

## Special Directories

**`cinatra/`:**
- Purpose: Holds platform-consumed sidecar artifacts (`oas.json` for agents, `workflow.bpmn` for workflows)
- Generated: Partially — `oas.json` is generated from the monorepo's agent compiler, then committed as a source mirror
- Committed: Yes

**`.planning/`:**
- Purpose: GSD planning documents (codebase maps, phase plans)
- Generated: Yes (by GSD tooling)
- Committed: Optional / project-dependent

---

*Structure analysis: 2026-06-09*
