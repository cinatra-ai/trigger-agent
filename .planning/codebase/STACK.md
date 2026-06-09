# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- TypeScript — target ES2023, JSX via react-jsx (config: `tsconfig.json`); no `src/` directory exists yet — the repo currently ships only configuration and a `cinatra/oas.json` agent spec
- JavaScript (ESM) — `extension-kind-gate.mjs` (self-contained CI gate, Node built-ins only, zero external deps)

**Secondary:**
- JSON — agent flow specification `cinatra/oas.json`, agent manifest embedded in `package.json`

## Runtime

**Environment:**
- Node.js 24 (pinned in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (managed via corepack — `corepack enable` in CI)
- Lockfile: not committed (CI runs `pnpm install --no-frozen-lockfile` for standalone repos)
- `.npmrc` file present — note existence only, contents not read

## Frameworks

**Core:**
- Cinatra Agent Framework — proprietary Cinatra platform (agentspec_version `26.1.0`); this repo is a `Flow`-type agent registered as `@cinatra-ai/trigger-agent`
- The agent flow uses `InputMessageNode` (HITL gate), `ApiNode` (passthrough API call), `StartNode`, `EndNode` node types defined in `cinatra/oas.json`

**Testing:**
- Not applicable — no test files present; tests run in the parent cinatra monorepo (source mirror pattern)

**Build/Dev:**
- TypeScript compiler (`tsc`) — standalone `tsconfig.json` targeting `outDir: dist`, `rootDir: src`
- No bundler configured; module resolution mode is `bundler`

## Key Dependencies

**Critical:**
- `@cinatra-ai/trigger-agent` package itself (`package.json` `name`) — published to the Cinatra Marketplace via `registry.cinatra.ai`
- No runtime npm dependencies declared in `package.json`; `dependencies` field is absent

**Infrastructure:**
- All `@cinatra-ai/*` host-internal packages are provided by the cinatra monorepo at workspace install time — they MUST NOT appear in `dependencies`/`devDependencies`; they are permitted only as optional `peerDependencies` (enforced by CI gate in `ci.yml`)

## Configuration

**Environment:**
- `CINATRA_BASE_URL` — templated into the `persist` node URL (`{{CINATRA_BASE_URL}}/api/agents/passthrough`) inside `cinatra/oas.json`; injected at runtime by the Cinatra platform dispatcher

**Build:**
- `tsconfig.json` — standalone strict TypeScript config (no monorepo extends); strict mode on, `noImplicitAny: false`, `isolatedModules: true`, `verbatimModuleSyntax: true`

## Platform Requirements

**Development:**
- Node 24, corepack/pnpm
- TypeScript compilation targets `src/` → `dist/` (no `src/` files committed yet)

**Production:**
- Deployed as a Cinatra Marketplace extension; published via GitHub Release triggering the reusable workflow `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml`
- Publish secret `CINATRA_MARKETPLACE_VENDOR_TOKEN` must be set as a GitHub org secret

---

*Stack analysis: 2026-06-09*
