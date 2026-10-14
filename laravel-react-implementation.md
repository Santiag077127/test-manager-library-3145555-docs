# Implementation Profile — Laravel, React, and PostgreSQL

This document adapts the \`spec-python/\` and \`spec-.net/\` references to the stack required by the
challenge: **PHP/Laravel**, **React**, and **PostgreSQL 16**. It does not redefine the product. If
it conflicts with \`spec.md\`, \`data-model.md\`, or \`api-contract.md\`, those documents prevail.

## 1. Responsibility boundaries

| Repository | Responsibility |
|---|---|
| \`api\` | Laravel API, migrations, business rules, and cover storage |
| \`app\` | React application that consumes \`/api/v1\` |
| \`page\` | Static public site; it does not call the API |
| \`infra\` | Containers, network, volume, and empty PostgreSQL engine; it contains no DDL |
| \`tool\` | Demonstration data through HTTP against the API; never direct SQL |

## 2. \`api\` architecture

The implementation preserves the hexagonal architecture in
[\`spec-.net/architecture.md\`](spec-.net/architecture.md): dependencies point toward the domain,
and only composition may know both ports and adapters.

A Laravel-compatible organization is:

\`\`\`text
src/Library/
├── Domain/                 # Pure PHP: entities, values, and invariants
├── Application/
│   ├── Ports/              # interfaces consumed by use cases
│   └── UseCases/           # one use case per service
├── Adapters/
│   ├── Inbound/Http/       # routes, controllers, validation, and errors
│   └── Outbound/           # Eloquent/SQL, JWT, hashing, clock, and covers
└── Bootstrap/              # port and adapter composition
\`\`\`

Laravel and Eloquent may appear only in adapters and composition. Repository, clock, password
hashing, JWT, and cover-storage interfaces belong to \`Application\`. The Laravel service provider
that registers those interfaces and implementations is the single composition point.

This structure is an implementation guide, not a seventh layer or a product requirement. Code,
routes, and columns remain in English; documentation is in English, while user-facing messages
remain in Spanish.

## 3. Migrations and an empty database lifecycle

PostgreSQL starts empty because \`infra\` creates only the engine, its user, network, and volume. It
must not contain \`CREATE TABLE\`, schema initialization scripts, or an ORM mechanism that creates
tables at startup in that repository.

Laravel migrations in \`api\` are the sole DDL source and must create the \`library\` schema, the six
business tables, constraints, indexes, and the five fixed categories. The deployment operation runs
Laravel migrations explicitly before declaring the API ready. Therefore:

1. \`docker compose up\` in \`infra\` can start PostgreSQL without business tables.
2. Running \`api\` migrations materializes the \`library\` schema.
3. API startup may provision the initial administrator only after migrations complete successfully;
   it does not replace migrations or create DDL.

The six business tables are exactly \`category\`, \`book\`, \`member\`, \`loan\`, \`loan_item\`, and
\`user\`, all in \`library\`. Laravel's technical migration-tracking table is metadata and must remain
outside the \`library\` schema (for example, in \`public\`). It is not a business table, is not queried
by the domain, and does not expand the six-entity model described in
[\`spec-.net/data-model.md\`](spec-.net/data-model.md).

## 4. Copy concurrency

Do not add a version column, \`updated_at\`, or any other attribute to the physical model. When a
loan is recorded, the persistence adapter performs an atomic conditional update for each active
book:

- It decrements one copy only if \`available_copies > 0\`.
- It verifies that exactly one row was affected.
- If any item cannot be decremented, it rolls back the complete transaction and the API returns
  \`409 BOOK_NOT_AVAILABLE\`.
- It creates the loan, its items, and all decrements in the same transaction.
- It retains \`ck_book_available_range\` as the database engine's final barrier.

This strategy meets RNF-2 without introducing a column absent from \`data-model.md\`. The concurrency
test must demonstrate that two simultaneous requests for the last copy result in exactly one \`201\`
and one \`409\`.

## 5. Covers and logical deletion

The database stores only \`cover_key\`, an opaque key or \`null\`; never a local path or binary. The
storage adapter is responsible for writing and reading the binary. It must validate JPEG/PNG by
content and enforce the 2 MB limit defined by the contract.

When a cover is deleted, first clear \`cover_key\` and commit the database transaction. Only then ask
the adapter to delete the binary. If physical deletion fails, do not restore the database reference:
the orphaned binary remains a recoverable operational incident.

Deleting a book means setting \`deleted_at\`; never physically deleting the row. Deletion is not
allowed when the book has open loans. Catalog and detail queries filter on \`deleted_at\` being null.

## 6. Derived values and history

\`loan.status\` and \`days_late\` are not columns. They are calculated at read time using a UTC clock:

- \`closed\`: every item has \`returned_at\`.
- \`overdue\`: at least one unreturned item has \`due_at\` earlier than the current instant.
- \`open\`: unreturned items exist and none is overdue.

The contract does not define the exact numeric representation of \`days_late\` beyond calculating it
from \`due_at\` and the current UTC instant. Do not persist it or assume a rounding rule without a
product decision.

When every \`loan_item\` is created, the application copies \`book_title\`, \`isbn\`, and
\`category_name\` from the catalog at that instant. The report groups by \`book_id\`, \`book_title\`,
and \`category_name\` from \`loan_item\`, not by the current values in \`book\` or \`category\`.
Therefore, later catalog changes do not alter historical reports.

## 7. Contract, configuration, and verification

The Laravel API preserves [\`spec-.net/api-contract.md\`](spec-.net/api-contract.md) unchanged: base
\`/api/v1\`, \`snake_case\` JSON, UUID v4, UTC dates, and the single error body. Every endpoint except
\`POST /auth/login\` and \`GET /health\` requires JWT. The CORS policy allows only the origin
configured through the environment.

Required environment variables are \`DATABASE_URL\`, \`JWT_SECRET\`, \`ADMIN_USERNAME\`,
\`ADMIN_PASSWORD\`, and \`CORS_ORIGIN\`; \`COVERS_DIR\` is optional. Only \`.env.example\` is
versioned; no secrets, passwords, or precomputed hashes are included.

Architecture tests must verify dependency rules, but the specification does not prescribe a PHP
tool. Domain, application, integration, contract, and concurrency tests follow the strategy
described in \`plan.md\` and \`tasks.md\`.

## 8. Areas without sufficient definition

No product decisions are added for the following topics because the contract does not define them:

- The rounding algorithm used to present \`days_late\` when the delay is less than a full day.
- The exact successful-response headers and representation for \`GET /books/{id}/cover\`.
- The detailed shape of some responses without examples, such as deletions and the category list.
- The precise temporal-boundary semantics of \`from\` and \`to\` in the loan list.

Implementation must request a decision before fixing behavior that affects the public contract.
