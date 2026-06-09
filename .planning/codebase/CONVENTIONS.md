# Coding Conventions

**Analysis Date:** 2026-06-09

## Overview

This is a content-only Cinatra agent extension. It ships no TypeScript source files of its own — the agent logic lives in `skills/trigger/SKILL.md` (prompt-driven instructions) and `cinatra/oas.json` (the agent surface/flow spec). The only JavaScript in the repo is the self-contained CI gate script `extension-kind-gate.mjs`.

## Naming Patterns

**Files:**
- Kebab-case for all filenames: `extension-kind-gate.mjs`, `oas.json`, `workflow.bpmn`
- Config/spec sidecars live under `cinatra/`: `cinatra/oas.json`
- Skill instructions live under `skills/<skill-name>/`: `skills/trigger/SKILL.md`

**Functions (in `extension-kind-gate.mjs`):**
- camelCase for all exported and private functions: `parseArgs`, `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `walkLlmStrings`, `scanOasString`, `wordBoundary`
- Verbs prefix function names describing their action: `validate*`, `find*`, `scan*`, `walk*`, `run*`, `parse*`

**Variables:**
- camelCase for local variables and parameters
- UPPER_SNAKE_CASE for module-level constants: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `OBJECTS_LIST_CRM_RE`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`

**Types:**
- Not applicable — the gate script (`extension-kind-gate.mjs`) is plain `.mjs`, not TypeScript. The `tsconfig.json` is present for future TypeScript source under `src/` but no `src/` directory exists yet.

## Code Style

**Formatting:**
- No Prettier or ESLint config files detected. The gate script uses consistent 2-space indentation throughout.
- Double-quoted strings used for all string literals.
- Trailing commas used in multi-line arrays and objects.

**Linting:**
- No `.eslintrc*` or `biome.json` detected. No lint tooling configured in `package.json` scripts.

**Module system:**
- ESM throughout. `package.json` declares `"type": "module"`. Imports use `node:` prefix for all builtins: `import { readFileSync, existsSync, readdirSync } from "node:fs"`.
- No CommonJS `require()` except in the inline CI shell script in `.github/workflows/ci.yml` (which uses `node -e 'const p = require(...)'` for a one-liner — acceptable because it runs in a non-module context).

## TypeScript Configuration (for future `src/`)

Defined in `tsconfig.json`:
- `"strict": true` with `"noImplicitAny": false` (strict mode on, but `any` is explicitly allowed)
- `"target": "ES2023"`, `"module": "ESNext"`, `"moduleResolution": "bundler"`
- `"verbatimModuleSyntax": true` — import type assertions are required for type-only imports
- `"isolatedModules": true` — each file must be independently compilable
- Output to `dist/`, with declaration maps and source maps enabled

## Import Organization

**In `extension-kind-gate.mjs`:**
- All imports are at the top of the file, grouped by Node builtin modules only (no third-party imports — zero-dependency constraint).
- Named imports used: `import { readFileSync, existsSync, readdirSync } from "node:fs"`.

## Error Handling

**Patterns in `extension-kind-gate.mjs`:**
- Pure functions return `string[]` error arrays — no thrown errors from validation logic. Callers accumulate errors, then decide.
- File I/O is wrapped in `try/catch`; errors push a human-readable message into the errors array and return early.
- `err instanceof Error ? err.message : String(err)` is the standard pattern for stringifying caught errors.
- The `main()` entrypoint wraps its call in `try/catch` and prints `"extension-kind-gate: unexpected error"` before `process.exit(1)`.
- Exit codes are explicit: `0` = pass, `1` = violations.

**Example pattern:**
```js
try {
  parsed = JSON.parse(readFileSync(oasPath, "utf8"));
} catch (err) {
  errors.push(`cinatra/oas.json failed to parse: ${err instanceof Error ? err.message : String(err)}`);
  return errors;
}
```

## Comments

**When to Comment:**
- File-level block comments explain the file's scope, what it does NOT do, and why (e.g., zero-dependency rationale in `extension-kind-gate.mjs` lines 1–34).
- Section dividers use `// ---...---` lines with a heading comment to visually separate logical sections.
- Inline comments explain non-obvious intent, tradeoffs, or cross-references to monorepo counterparts.

**Comment style:**
- `//` line comments throughout, no `/* */` block comments in code.
- Comments reference the authoritative source in the monorepo (e.g., "Ported verbatim from scripts/audit/oas-banned-primitives-gate.mjs").

## Function Design

**Size:** Functions are focused and single-purpose. Validation functions are pure (no side effects, return `string[]`).

**Parameters:** Functions take typed primitives or plain strings/objects. No options-bag pattern detected.

**Return Values:** Validation functions return `string[]` (empty = pass, non-empty = failures). `runGate` returns `{ kind, errors }`. `parseArgs` returns `{ packageRoot }`.

## Module Design

**Exports:** Named exports only — no default exports. All logic is exported for testability: `parseArgs`, `validateAgent`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `validateWorkflow`, `runGate`.

**Entry Point Guard:** The `main()` function is invoked only when the script is run directly, using an `invokedDirectly` check comparing `process.argv[1]` to `import.meta.url`. This allows the module to be `import`-ed in tests without executing side effects.

## Agent Skill Conventions (SKILL.md)

**Format:** SKILL.md uses a YAML front matter block (`name`, `description`) followed by Markdown with numbered steps.

**Step structure:**
- Each step is labeled `STEP N — <Name> (<type>):` where type is one of: `HITL`, `LLM via orchestration`.
- Field formats are specified precisely: ISO 8601 UTC for dates, 5-field cron strings for schedules.
- Constraints section enumerates hard rules (never call X before Y, never call X more than once, etc.).

## OAS / cinatra/oas.json Conventions

- Agent flow defined as `"component_type": "Flow"` in `cinatra/oas.json`.
- Inputs and outputs use `title` + `type` fields (no `description` at the field level in this agent).
- Nodes reference components via `{ "$component_ref": "<name>" }`.
- `hitlScreens` registered in `metadata.cinatra.hitlScreens` as `"@<package-name>:<screen-name>"` strings.

---

*Convention analysis: 2026-06-09*
