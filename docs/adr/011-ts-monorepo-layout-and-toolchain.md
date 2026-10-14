# ADR-011: TS monorepo layout and toolchain

- Status: Accepted · Date: 2026-10-13

## Context

The `interline` system is a multi-service travel booking and itinerary platform. The JavaScript/TypeScript ecosystem (`interline-app`) encompasses several decoupled components:

- Shared domain contracts and schemas (`@interline/contracts`).
- Background ETL and schedule ingestion workers (`@interline/schedule-import`).
- Upcoming backend application (a single modular NestJS monolith, alongside the core flight search/booking Go engine).
- Upcoming frontend web application (Next.js client interface).

To support rapid iteration, type safety, and atomic updates between shared contracts and consuming applications, we needed to establish the foundational workspace architecture and tooling stack (Node 26, TypeScript 6.0, pnpm 12, Vitest 5, ESLint 10).

Key requirements:

1. Enable frictionless cross-package imports and refactoring without packaging overhead.
2. Maintain single-source-of-truth dependency versions across all packages.
3. Align TypeScript configuration with Node 26 native TypeScript execution.
4. Establish fast, deterministic linting, type-checking, and testing in local development and CI.

---

## Decision

### 1. Workspace Layout (pnpm workspaces)

We adopt a monorepo structure in `interline-app` managed by **pnpm workspaces** (`pnpm-workspace.yaml`). All packages reside under `packages/*`.

Inter-package dependencies are linked using `workspace:*` protocol (e.g. `@interline/contracts: "workspace:*"` inside `@interline/schedule-import`).

### 2. Centralized Dependency Catalog

We utilize pnpm 12's `catalog:` feature in root `pnpm-workspace.yaml`. Shared dependencies (`typescript`, `vitest`) are pinned in the root catalog. Workspace packages declare `"typescript": "catalog:"`, preventing version drift across packages.

### 3. Direct Source Consumption (`src` vs `dist`)

Internal packages expose their TypeScript source code directly via package exports (`"./src/index.ts"`) rather than precompiled JavaScript in `dist/`.

- Node 26 natively executes TypeScript by stripping types.
- TypeScript compiler (`tsc`) operates in `--noEmit` mode purely for static analysis.
- Vitest resolves and executes `.ts` source files natively without intermediate build steps.

### 4. TypeScript 6.0 Toolchain Alignment

All packages extend a root `tsconfig.base.json`. Compiler options strictly match Node 26 runtime semantics:

- `"moduleResolution": "NodeNext"` and `"module": "NodeNext"` for ECMAScript module resolution.
- `"erasableSyntaxOnly": true` and `"verbatimModuleSyntax": true` to ensure all TypeScript syntax can be stripped without runtime transforms (forbidding legacy TypeScript features such as `enum` or parameter properties).
- `"noEmit": true` and `"isolatedModules": true`.
- Relative imports explicitly specify `.ts` extensions (`import { ... } from './x.ts'`).

### 5. Testing and Typed Linting

- **Vitest 5:** Test discovery is orchestrated from root `vitest.config.ts` using `test.projects: ['packages/*']` rather than deprecated workspace files.
- **ESLint 10 Flat Config:** Implemented in `eslint.config.js` (native ESM in Node 26). Typed linting is enabled via `@typescript-eslint` using `projectService`, with root config files explicitly allowed via `projectService: { allowDefaultProject: ['vitest.config.ts'] }`.
- `@typescript-eslint/no-floating-promises` is strictly enforced to eliminate silent asynchronous runtime errors.

### 6. Code Formatting

Currently, no automated code formatter (such as Prettier or Biome) is configured. Introducing a dedicated formatter is deferred until frontend UI development begins.

### 7. CI Pipeline

Continuous integration in GitHub Actions runs as a single job (`CI`) executing `lint`, `typecheck`, and `test` sequentially on a single runner:

- Reuses a single `pnpm install` step.
- Utilizes GitHub Actions cache for pnpm store.
- Enforces strict branch protection on `main`.

---

## Alternatives Considered

1. **Multi-Repo Architecture (separate git repos per package):**
   - _Rejected:_ Publishing `@interline/contracts` to a package registry on every change introduces heavy latency, version management overhead, and prevents atomic commits spanning contracts and business logic.
2. **Turborepo / Nx Monorepo Orchestration:**
   - _Rejected for Day 1:_ Unnecessary complexity and caching overhead for two small packages. Sequential pnpm scripts complete in ~30 seconds. Build orchestrators can be introduced later when NestJS and Next.js applications increase compilation times.
3. **Precompiling packages to `dist/` via `tsc -b` / Bundlers:**
   - _Rejected:_ Requires build watchers and slows down local test-driven development (TDD). Consuming `src` directly leverages Node 26's native TypeScript support with zero build latency.
4. **Standalone Prettier:**
   - _Rejected:_ Flat Config ESLint does not provide adequate formatting checks.
5. **Multi-Job CI Pipeline:**
   - _Rejected:_ Running lint, typecheck, and test in separate jobs would require 3 virtual machines and 3 separate `pnpm install` runs, unnecessarily increasing CI run time and consumption.

---

## Consequences

### Positive

- **Instant feedback loop:** Edits in `@interline/contracts` take effect immediately in consuming packages without running a build command.
- **Atomic commits and PRs:** Changes across contracts and consuming services are reviewed and merged together in a single pull request.
- **No stale artifacts:** Eliminates build artifacts (`dist/`) desynchronization during local development and testing.
- **Unified dependency tree:** pnpm catalog ensures zero dependency divergence.

### Negative / Trade-offs

- Internal packages cannot be published directly to external npm registries without introducing a distribution build pipeline.
- Requires all workspace consumers and runners to support native TypeScript execution (Node 26+).
- Strict adherence to `erasableSyntaxOnly` prevents using TypeScript-only features like runtime `enum`s.
- Node.js native type stripping explicitly refuses to execute `.ts` files inside `node_modules` directories. Locally it works perfectly, because locally workspace symlinks and Vitest handle `.ts` source imports. But deploying packages into isolated production environments or Docker containers can fail at runtime. That is why a compilation step to `dist/` will be added when it is needed.
