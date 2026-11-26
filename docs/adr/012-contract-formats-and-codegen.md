# ADR-012: Contract formats and codegen

- Status: Accepted · Date: 2026-11-26

## Context

`ADR-001` split the `interline` project into three repositories (`interline-app` in TypeScript, `interline-engine` in Go, and `interline-contracts`) and postponed contract formats and codegen until REST integration in Phase 1. Now `core` (NestJS) and `engine` (Go) are starting to integrate over REST (`GET /v1/flights/{flightKey}/availability`), so both sides need an agreed API contract format, shared `flightKey` validation rules, and a way to deliver schemas and generated types across separate repositories.

## Decision

A decision was made to have the following contract format and codegen rules:

- **REST-contracts:** OpenAPI v3.1 with file per service in `interline-contracts/openapi/`. First file is `engine.yaml`.
- **JSON-fields:** in `camelCase` in all contracts, including Kafka events.
- **Format flight key:** pattern in spec with anchors (`^` and `$`) so `core` and `engine` validate keys identically.
- **Errors:** at the moment only statuses `400`, `404` and `5XX`; error bodies (Problem Details) will be added in P2.8.
- **Security:** service JWT will be added in P2.7; until then `security-defined` is off in `redocly.yaml`.
- **Specs check:** Redocly CLI 2.75.0 with the `recommended` ruleset and acceptance tests.
- **Go without codegen:** at the start. Handler and JSON are handwritten in P1.2, tests validate actual responses against the spec via `libopenapi-validator` v0.16.2.
- **Deliver spec to the repo:** for `interline-app`, TS types are generated from a pinned commit in `interline-contracts`, committed into `packages/contracts`, and checked in CI; for `interline-engine` tests, the spec path is passed via `CONTRACTS_DIR`.
- **Types Generator:** `openapi-typescript` v7.13.0, because this is a stable major that works cleanly with `tsc` v6.0.3 and generates only types, so the client (fetch, errors, timeouts, JWT) stays under our control, without surprises.

## Alternatives considered

- **`oapi-codegen` for Go immediately.** Pros: generates server stubs and DTOs from the spec. Cons: extra generated boilerplate and build setup for a single endpoint; rejected in favor of a manual handler with `libopenapi-validator` at the start. Not tried yet.
- **Code-first spec generation** (from server annotations/decorators). Pros: spec lives next to handler code. Cons: ties the contract to one framework and breaks the contract-first workflow between TS and Go.
- **OpenAPI v3.0.** Pros: wide legacy tooling support. Cons: uses its own dialect (`nullable: true`) instead of JSON Schema 2020-12, which OpenAPI 3.1 shares with F3 events.
- **`@hey-api/openapi-ts` v0.99.0.** Pros: generates both types and an HTTP client, supports TS 6 in peers. Cons: pre-1.0 (`0.x`) package with potential breaking changes between minor versions.
- **`git submodule` or manual copies in both repos.** Pros: spec files stay locally in each repo without fetching a pinned commit. Cons: submodules require manual pointer bumps and `--recursive` setup, while manual copies silently drift out of sync.

## Consequences

**Pros:** a single source of truth between `core` and `engine`, one schema model (JSON Schema 2020-12) for REST and future Kafka events, strict TS types checked in CI and no "magic" generators in Go or the TS client.

**Cons:** manual HTTP handler and JSON tags in Go (bugs are caught only by tests with `libopenapi-validator`), manual TS client (plus a `typescript ^5` peer warning on install), updating the pinned commit in `interline-app` and passing `CONTRACTS_DIR` for `engine` tests; `security-defined` and error bodies remain deferred until P2.7 and P2.8.

**When to revisit:** re-evaluate Go codegen (`oapi-codegen`) when the second REST endpoint (Phase 2 search) is added to `engine`.
