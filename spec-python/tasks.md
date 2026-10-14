# Tasks — Manager Library (Python)

A task is complete only when its **evidence** exists and can be executed (Article XII). The initial
state of every task is **pending**.

| # | Task | Depends on | Evidence |
|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | — | `GET /health` returns `{"status":"ok"}`; the container starts with one command |
| T-02 | Initial migration: six tables, FKs, unique indexes, and five seeded categories | T-01 | The `data-model.md` §10 queries return the documented model; no column has a `DEFAULT` |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` (value objects) | T-01 | Unit tests: 13-digit ISBN, quantity > 0, end ≥ start |
| T-04 | Domain: `Book` (`Withdraw`, `Restock`, logical deletion) | T-03 | Tests: never < 0, never > total |
| T-05 | Domain: `Loan` (`AddItem`, `ReturnItem`, calculated status, 14 days, max. 3) | T-04 | Tests with a fake `Clock`: `open`, `overdue`, `closed` |
| T-06 | Domain: `Member` and `User` (username normalization, valid role) | T-03 | `"Ana "` becomes `ana`; an `admin` role through the API is rejected |
| T-07 | **Authentication** ports and use cases (`LoginService`, `CreateLibrarianService`) with fakes | T-06 | HU-01 and HU-02 pass with fakes |
| T-08 | **Book** ports and use cases (create, search, edit, logical deletion, cover) with fakes | T-04 | HU-03 and HU-04 pass with fakes |
| T-09 | **Loan** use case (`LendBooksService`) with fakes | T-05, T-08 | Complete HU-06: limits, overdue items, unavailable copies |
| T-10 | **Return** and **overdue** use cases | T-09 | HU-07 and HU-09; returning twice → `ITEM_ALREADY_RETURNED` |
| T-11 | **Most-borrowed report** use case and its read port | T-09 | HU-10; old title remains after renaming (ADR-004) |
| T-12 | Persistence adapter: book, member, and user repositories | T-02, T-08 | Integration tests against real PostgreSQL |
| T-13 | Persistence adapter: loan repository and atomic conditional decrement | T-12, T-09 | Two loans for the last copy: one 201, one 409 (RNF-2) |
| T-14 | Security adapter: password hashing and JWT token | T-07 | The password never appears in logs or responses (RNF-3) |
| T-15 | Cover adapter (disk) | T-08 | Uploading a 2 MB PNG succeeds; 3 MB → 422; no cover → `cover_key` is null |
| T-16 | HTTP adapter: [`api-contract.md`](api-contract.md) routes, single error body, role authorization | T-10, T-11, T-14, T-15 | A contract test covers every route and every §2 code |
| T-17 | Composition and startup (initial administrator from environment) | T-16 | The system starts with an empty database and can log in as administrator |
| T-18 | Move **domain-only** invariants to the database: `ck_book_total_positive`, `ck_user_role` | T-12 | A violating `psql` `INSERT` fails; `data-model.md` §4 is updated |
| T-19 | Architecture tests (Articles I, II, and III) | T-17 | an `import-linter` contract and a test in `tests/architecture/` pass and fail when a forbidden import is introduced |
| T-20 | `infra`: `docker compose up` starts the empty engine, API, and frontend | T-17 | One command; the database starts without tables until migrations run |
| T-21 | `tool`: demonstration seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue items) | T-17 | After running it, `/loans/overdue` returns 3 items and the report has rows |
| T-22 | `app`: login, catalog, members, loan, return, overdue, report | T-16 | End-to-end manual walkthrough at the fair |
| T-23 | `page`: static public presentation site | — | Opens without the API; correct on mobile and desktop |

## Assessment definition of done

Reaching T-17 with evidence means the **core** is complete. T-18 to T-23 distinguish good work
from outstanding work. SDD performance—not the number of tasks—is evaluated: one evidenced task is
worth more than three partial tasks.
