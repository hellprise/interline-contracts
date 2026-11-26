# ADR-003: Data access with pgx and hand-written SQL

- Status: Accepted · Date: 2026-11-26

## Context

The engine keeps seat inventory in PostgreSQL 18 (`seat_inventory`). The data layer needs to read flight seats by `flight_key` and reserve seats atomically in one statement, checking the result via `RowsAffected`. Under concurrent load, ten simultaneous holds for the last two seats must allow only two successes without race conditions, which is verified against a real Postgres instance.

Upcoming stages add three more requirements for the data layer:

- map PostgreSQL errors by `SQLSTATE` (`23505` unique violation, `23514` check violation) to domain errors;
- bulk-load about 4,000 schedule flights from seed using `COPY`;
- run queries tailored to specific indexes in Phase 2 and inspect their execution plans.

## Decision

We use `pgx` v5 directly (native interface), without an ORM or SQL code generator.

- **Pool ownership:** the caller (`main` or tests) creates one `*pgxpool.Pool` and passes it into repositories; a repository never creates or closes the pool.

- **Hand-written SQL and parameters:** SQL queries are written manually as Go constants, and values are passed only via `$1`, `$2`, etc parameters (separated from the SQL text to prevent injection).

- **Domain boundary and errors:** repositories map `database/pgx` errors into domain errors (`ErrFlightNotFound`, `ErrNotEnoughSeats`), while the `inventory` domain package stays decoupled and does not import `pgx` (enforced by an acceptance test).

- **`goose` compatibility:** since `goose` requires `database/sql`, tests bridge it onto the existing pool via `stdlib.OpenDBFromPool`.

## Alternatives considered

- **ORM (`GORM` or `ent`).** Pros: less boilerplate for plain CRUD and relations out of the box. Cons: SQL is generated and hidden, whereas our atomic hold, `COPY` and Phase 2 queries require exact control over SQL. Not tried here yet.

- **`sqlc`.** Generates typed Go code from SQL files. Pros: SQL stays visible, no manual `Scan`, and queries are checked against the schema at generation time. Cons: adds a codegen step to builds and CI, and dynamic queries are harder to write. Not tried here yet.

- **`database/sql` with `pgx` as a driver.** Pros: standard Go interface (the one `goose` uses). Cons: no native `COPY` or `LISTEN`/`NOTIFY` without dropping down to `pgx`; `pgx` recommends its native interface when targeting only PostgreSQL, so here only `goose` gets `database/sql` via `stdlib.OpenDBFromPool`.

## Consequences

**Pros:** full visibility and control over every SQL query (including the conditional `UPDATE` and upcoming index-tuned queries), and domain packages stay independent of database drivers.

**Cons:** more manual boilerplate (`Scan`, `RowsAffected`, error mapping per repository), and SQL typos or constraint bugs are only caught against a real PostgreSQL instance (which is why T1.4 and P1.5 run against compose Postgres).

**When to revisit:** re-evaluate `sqlc` when repositories exceed 15 hand-written queries, or if a second service introduces its own database.
