# Testing Patterns

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra agent extension (source mirror). It ships no `src/` TypeScript files and declares no test framework in `package.json`. There are no test files in the repository. Testing of the agent's TypeScript sources (if added) is deferred to the cinatra monorepo, which owns the build/typecheck/test lifecycle for host-internal `@cinatra-ai/*` extensions.

The sole testable artifact in this repo is `extension-kind-gate.mjs`, which exports pure functions designed for unit testing but has no test suite here.

## Test Framework

**Runner:**
- Not configured. No `jest.config.*`, `vitest.config.*`, or test runner in `package.json`.

**Assertion Library:**
- Not detected.

**Run Commands:**
```bash
# No test script defined in package.json.
# CI step runs: corepack pnpm test --if-present
# Since no test script exists, this is a no-op for standalone CI.
```

## CI Test Behavior

Defined in `.github/workflows/ci.yml`:

- The `build` job classifies the repo as a **source mirror** (because `package.json` declares host-internal `@cinatra-ai/*` optional peers).
- When `first_party=1`, the CI skips install, typecheck, and test steps entirely with the message: *"Skipping standalone tests (host-internal @cinatra-ai/* peers — the cinatra monorepo runs these)."*
- Test coverage for agent behavior runs inside the cinatra monorepo, not in this extracted repo.

## Kind Gate Validation (CI-equivalent of tests)

The `kind-gates` job in `.github/workflows/ci.yml` runs `extension-kind-gate.mjs --package-root .` as the functional validation step for this agent.

This gate:
- Reads and parses `cinatra/oas.json`
- Scans all LLM-visible strings (`system`, `user`, `description` fields) for retired CRM primitive usage
- Returns exit code `0` (pass) or `1` (violations found)

The gate functions in `extension-kind-gate.mjs` are all exported pure functions, making them directly importable in a test suite if one is added:

| Exported Function | What it validates |
|---|---|
| `parseArgs(argv)` | CLI argument parsing |
| `validateAgent(packageRoot)` | OAS JSON parse + banned-primitive scan |
| `validateWorkflowPackageShape(pkg)` | package.json shape for workflow kind |
| `validateBpmnSanity(xml)` | BPMN XML well-formedness + BPMN 2.0 MODEL namespace |
| `findWorkflowSidecars(packageRoot)` | Locates all `cinatra/workflow.bpmn` files |
| `validateWorkflow(packageRoot)` | Full workflow extension validation |
| `runGate(packageRoot)` | Dispatches to agent or workflow gate |

## Test File Organization

**Location:**
- No test files exist. If added, the tsconfig targets `src/**/*.ts` — tests would conventionally live alongside source or in a `src/__tests__/` directory.

**Naming:**
- Not applicable (no tests). Convention for `.mjs` gate tests would be `extension-kind-gate.test.mjs`.

## Mocking

**Framework:** Not detected.

**What to Mock (if tests were added):**
- File system calls (`readFileSync`, `existsSync`, `readdirSync`) from `node:fs` — the gate functions are pure above the I/O layer, so mocking fs is the primary test boundary.
- `process.argv` for `parseArgs` tests — pass synthetic `argv` arrays directly since the function accepts `argv` as a parameter.
- `process.exit` for `main()` integration tests.

**What NOT to Mock:**
- The pure validation functions (`validateBpmnSanity`, `validateWorkflowPackageShape`, `walkLlmStrings`, `scanOasString`) — these are pure string-in / string-out functions; pass fixture strings directly.

## Fixtures and Factories

**Test Data:**
- Not applicable (no tests). If added, fixture `.json` and `.xml` strings for OAS and BPMN validation would be the primary fixture type, passed inline or loaded from a `fixtures/` directory.

## Coverage

**Requirements:** None enforced. No coverage threshold configured.

**View Coverage:**
```bash
# Not configured.
```

## Test Types

**Unit Tests:**
- Not present. The exported pure functions in `extension-kind-gate.mjs` are architecturally ready for unit testing.

**Integration Tests:**
- Not present. CI's `node extension-kind-gate.mjs --package-root .` serves as the integration gate.

**E2E Tests:**
- Not applicable.

## Agent Behavior Testing

The agent's prompt-driven behavior (SKILL.md steps, HITL gate flows, `trigger_config_set` call) is not unit-testable in isolation. It is tested:
1. Via the cinatra monorepo's agent test harness (not in this repo).
2. Via the OAS-level kind gate (`validateAgent`) which scans for retired primitives in `cinatra/oas.json`.

The `cinatra/oas.json` flow spec defines inputs, outputs, nodes, and edges that the Cinatra runtime validates at marketplace publish/install time.

---

*Testing analysis: 2026-06-09*
