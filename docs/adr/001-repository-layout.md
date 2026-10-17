# ADR-001: Repository Layout

- Status: Accepted · Date: 2026-10-17

## Context

The `interline` project consists of two tracks: TypeScript / Node.js 26 (`web` on Next.js and `core` on NestJS) and Go 1.27 (the `engine` and test simulator `airline-sim`).
They have shared artifacts, such as: initial schedule data (`seed/` folder), architecture decisions (`docs/adr/` folder), future system document (`docs/system.md` file), that describes all the system sides and shared contracts (REST, gRPC, events).

## Decision

A decision was made to divide the project into three repositories:

- **`interline-app:`** the pnpm monorepo for TypeScript part that includes `web` on Next.js, `core` on NestJS and internal packages.
- **`interline-engine:`** one Go-module (`github.com/hellprise/interline-engine`) for two services (`engine` and `airline-sim`) and one generator `cmd/seedgen`.
- **`interline-contracts:`** shared repository for documents, ready-to-use CSV files and contract schemas.
  Highlighting the `interline-contracts` repository, this repository can be changed by both Go and TypeScript tracks, and ready-to-use CSV files in `seed/` are stored in the repository, so the TypeScript side does not need to install Go to run schedule imports.
  Contract formats and codegen were deliberately postponed until the stages where they are needed (REST in Phase 1, gRPC and event schemas in Phase 3).

## Alternatives considered

A single shared repository for all services was considered - a polyglot monorepo for Go + TS.

- **`Pros:`** shared access to all services, possibility to modify necessary services during one session and merge it in one PR. Also, shared things such as documents, schemas, contracts would be reused.
- **`Cons:`** Mixing two different toolchains (`pnpm 12` + `Node 26` **vs** `go.mod` + `Go 1.27`), CI becomes more complex, complex handling of update-bots (`Renovate`).

## Consequences

Each repository has a single toolchain and a fast, simple CI pipeline. However, each change to seeds, schemas or contracts requires making commits and creating PRs across multiple repositories.
