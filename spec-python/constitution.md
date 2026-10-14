# Constitution — Manager Library (Python)

Non-negotiable project principles. They are neither style nor preference: every article states **how
it is verified**, because a principle that nobody can verify is merely an intention. A change that
violates an article is rejected without considering the rest.

---

## Article I — The domain depends on nothing

`src/library/domain/` imports no other project module or third-party library: only the standard
library (`dataclass(frozen=True)`, `datetime`, `typing`); no SQLAlchemy, FastAPI, or pydantic.

**Why:** this is the only way to test invariants without a database, server, or container and to
change the ORM without modifying rules.
**Verification:** an `import-linter` contract and a test in `tests/architecture/` prevent the domain
from depending on `application`, `adapters`, or `bootstrap`.

## Article II — The core declares its ports; the outside world implements them

The application layer knows only the domain. Everything it needs from the outside (repositories,
clock, password hashing, cover storage, token generation) is expressed as a **port that the
application itself defines**. No framework or ORM type appears in a port signature.

**Verification:** an `import-linter` contract and a test in `tests/architecture/` prevent
`application` from importing `adapters`. Application tests run with fakes and without referencing
`adapters`.

## Article III — A single composition point

A single module (`src/library/bootstrap/composition/port_bindings.py`) is where a port meets its
adapter. Infrastructure is not instantiated outside it.
**Verification:** no `adapters` class is instantiated outside that module; an architecture test
enforces this rule.

## Article IV — SOLID is not decorative

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One application service equals one use case. `LendBooksService` lends books and does nothing else | A service with two reasons to change is split |
| **OCP** | Changing cover storage (disk → S3) does not modify the domain or application | The new adapter adds only one line to composition |
| **LSP** | Test fakes replace the real adapter without changing expectations | The same test passes against the fake and the real implementation |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementer leaves methods unused |
| **DIP** | The consumer defines the port | Ports live in the application package |

## Article V — Migrations own the schema

The `infra` repository starts an **empty** engine. All DDL (tables, indexes, constraints,
extensions, and seed data) lives in application migrations (Alembic revisions). See
[ADR-001](adr/adr-001-propiedad-del-esquema.md).
**Verification:** `infra` contains no `CREATE TABLE`.

## Article VI — The database enforces every invariant it can enforce

A rule that exists only in code protects the application, not the data. Every invariant expressible
by the engine (`CHECK`, `UNIQUE`, FK) is declared there. Anything not yet enforced is marked
**pending** in [`data-model.md`](data-model.md); it is not promised.
**Verification:** the queries in `data-model.md` §10 match the document's declarations.

## Article VII — Derived values are calculated, not stored

Loan status (`open`, `overdue`, `closed`) and days late are calculated. They have no columns.
**Verification:** `data-model.md` lists neither a `status` nor a `days_late` column.

## Article VIII — Frozen values do not follow the catalog

A loan item stores the title, ISBN, and category as they were when the loan was created. No report
reads them from the live catalog.
**Verification:** a test renames a book after lending it and confirms that the report retains the
old title ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)).

## Article IX — No secret enters the repository or a log

No passwords, precomputed hashes, or token keys. Every secret comes from the environment. The real
`.env` is in `.gitignore`; only `.env.example` is versioned.
**Verification:** `gitleaks` or a pattern `grep` in CI; no HTTP response includes `password` or
`password_hash`.

## Article X — The database engine wins

If the data document and the database disagree, the document is wrong and must be corrected.

## Article XI — Language

Code, columns, routes, and documentation are in English. User-facing messages remain in Spanish.

## Article XII — Nothing is complete without evidence

A task in [`tasks.md`](tasks.md) is complete when its **evidence** exists and can be executed, not
when it is “already implemented.”
