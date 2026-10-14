# Technical Plan — Manager Library (.NET)

How to build what [`spec.md`](spec.md) and [`constitution.md`](constitution.md) define. The schema
is **not** described here; it lives in [`data-model.md`](data-model.md). The system structure lives
in [`architecture.md`](architecture.md).

## 1. Technical decisions

| # | Decision | Selection (.NET) | Rationale |
|---|---|---|---|
| D-01 | Language and runtime | .NET 8 / C# 12 | Reference specification stack |
| D-02 | HTTP framework | ASP.NET Core (Minimal API) | Contract in [`api-contract.md`](api-contract.md) |
| D-03 | Persistence | EF Core 8 + EF migrations on PostgreSQL 16 | Migrations own the schema (ADR-001) |
| D-04 | Concurrency | Atomic conditional update per book + range `CHECK` ([ADR-002](adr/adr-002-concurrencia-optimista.md)) | Last copy (RNF-2) |
| D-05 | Dates | UTC instants (`timestamptz`); the clock is a `Clock` port | Test overdue behavior without waiting 14 days |
| D-06 | Report | One aggregate query through a read port ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)) | Stable and inexpensive |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: stored within their owner's row, without a table | They have no identity |
| D-08 | Covers | Binary in external storage (disk locally, S3 in production); the database stores only `cover_key` | Opaque key, never a path |
| D-09 | Passwords | BCrypt (cost factor 12) behind the `PasswordHasher` port; the domain never sees the plain-text password | Article IX |
| D-10 | Startup | The application creates the initial administrator from `ADMIN_USERNAME` / `ADMIN_PASSWORD` | The database cannot produce the hash |
| D-11 | Authentication | HS256 JWT, eight hours, key in `JWT_SECRET` | Simple and stateless |
| D-12 | Tests | xUnit, with fakes for ports and a real database only for integration tests | Test pyramid |

## 2. Environment configuration

| Variable | Required | Example |
|---|---|---|
| `DATABASE_URL` | yes | `postgresql://library:library@db:5432/library` |
| `JWT_SECRET` | yes | long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | yes | defined in `.env`, never in the repository |
| `COVERS_DIR` | no | `./var/covers` |
| `CORS_ORIGIN` | yes | `http://localhost:5173` |

## 3. Testing strategy

| Level | What it tests | Against |
|---|---|---|
| Domain | Invariants: maximum three, 14-day due date, available copies in range | Nothing: pure objects |
| Application | Use cases: loan, return, report | Port fakes |
| Integration | Repositories and report query | Real PostgreSQL |
| Contract | Single error body and status codes | Running API |
| Architecture | Articles I, II, and III | NetArchTest |
| Concurrency | Two loans for the last copy | Real PostgreSQL |

## 4. Build order

Skeleton and health → initial migration with seed data → domain → use cases with fakes → adapters →
API → contract and concurrency tests. Details and required evidence are in
[`tasks.md`](tasks.md).
