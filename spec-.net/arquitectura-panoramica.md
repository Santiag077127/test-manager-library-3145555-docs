# Architecture at a glance — Manager Library

Read this page in five minutes before opening the rest. Details are in
[\`architecture.md\`](architecture.md).

## The system in one diagram

\`\`\`mermaid
flowchart TB
    U["Librarian / Administrator"] --> APP["app (React)<br/>frontend"]
    V["Visitor"] --> PAGE["page<br/>static public site"]
    APP -->|"HTTP /api/v1"| API["api<br/> .NET backend (Laravel in this challenge)"]
    API --> PG[("PostgreSQL 16<br/>library schema")]
    API --> COV[("Covers<br/>disk / S3")]
    TOOL["tool<br/>demonstration seeder"] -->|"uses the API"| API
    INFRA["infra<br/>containers and empty engine"] -.-> PG
    INFRA -.-> API
\`\`\`

## The six repositories and their role

| Repository | Role | Tested with |
|---|---|---|
| \`docs\` | Specification | This reading must be sufficient to build the system |
| \`api\` | Business rules and HTTP contract | Domain, application, contract, and concurrency tests |
| \`app\` | Operational screens | An end-to-end walkthrough: login → loan → return → report |
| \`page\` | Public presentation, without API calls | Looks good on mobile and desktop |
| \`infra\` | \`docker compose up\` and empty database | Starts with one command |
| \`tool\` | Demonstration data (books, members, loans) | The presentation fair has real data |

## The five ideas that support the design

1. **A loan is an immutable fact**; only an item's \`returned_at\` changes.
2. **Copies are withdrawn and returned in the same operation** that creates or closes the item.
3. **Status is calculated**: \`open\`, \`overdue\`, and \`closed\` are not stored in the database.
4. **Frozen values do not follow the catalog**: an October report does not change because a book is
   renamed in November.
5. **The database is the final barrier**: if the copy \`CHECK\` fails, something wrote outside the
   system.
