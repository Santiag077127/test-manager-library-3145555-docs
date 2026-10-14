# test-manager-library-3145555-docs

> **Technical assessment · ADSO 3145555 cohort**
> Schedule: **Wednesday, October 14, 2026, from 6:50 a.m. to 11:00 a.m.** (Colombia time). Submission is due at **11:00 a.m.**

This repository contains the *Manager Library* **specification**. It is the only repository with
content: the other five intentionally start empty.

## Instructions

Each trainee must **fork** the six project repositories and solve the project according to the
specification.

1. Fork each repository in the final table to your GitHub account.
2. Read the specification in [\`test-manager-library-3145555-docs\`](https://github.com/code-sena/test-manager-library-3145555-docs).
   It is provided in two versions: \`spec-python/\` and \`spec-.net/\`.
3. Develop in your forks.

## The challenge uses React and PHP (Laravel)

The specification is written for Python and .NET, but the challenge is **not** implemented in those
languages:

| Layer | Challenge technology |
|---|---|
| Frontend | React |
| Backend | PHP with Laravel |

Business requirements—stories, acceptance criteria, rules, HTTP contract, and data model—must be
preserved. Technology decisions are translated to React and Laravel.

## This assessment is not about writing code

Its primary purpose is to assess **Spec-Driven Development (SDD)**: how a specification is read,
interpreted, and applied to a different stack. Code is the means, not the end.

## Submission

Only work **pushed to the forks before 11:00 a.m.** counts. Later work is not reviewed. There is no
extension: push partial progress through small commits during the assessment.

## Repository contents

| Directory | Content |
|---|---|
| [\`spec-python/\`](spec-python/) | Manager Library specification written for a Python backend |
| [\`spec-.net/\`](spec-.net/) | The same system specified for a .NET backend |

Both versions describe the **same product** (the same stories, business rules, and endpoints); only
technology decisions differ. Neither version is the challenge stack.

## How to read the specification

In either directory, use this order:

1. \`constitution.md\` — non-negotiable principles.
2. \`spec.md\` — system behavior: actors, stories, acceptance criteria, and business rules.
3. \`plan.md\`, \`architecture.md\`, and \`arquitectura-panoramica.md\` — how it is built.
4. \`data-model.md\` and \`api-contract.md\` — data model and API contract.
5. \`tasks.md\` — required work and executable evidence.
6. \`adr/\` — architecture decisions, alternatives, and consequences.

## Challenge implementation profile

The \`spec-python/\` and \`spec-.net/\` directories are references for the same product, not the
challenge stack. To translate their technical decisions without changing business rules, the HTTP
contract, or the data model, follow
[\`laravel-react-implementation.md\`](laravel-react-implementation.md). It defines the Laravel,
React, and PostgreSQL 16 implementation profile and resolves infrastructure decisions that do not
belong to the product itself.

## The six repositories

| Repository | Contents |
|---|---|
| [\`test-manager-library-3145555-docs\`](https://github.com/code-sena/test-manager-library-3145555-docs) | Specification: \`spec-python/\` and \`spec-.net/\` |
| [\`test-manager-library-3145555-api\`](https://github.com/code-sena/test-manager-library-3145555-api) | PHP (Laravel) backend |
| [\`test-manager-library-3145555-app\`](https://github.com/code-sena/test-manager-library-3145555-app) | React frontend |
| [\`test-manager-library-3145555-page\`](https://github.com/code-sena/test-manager-library-3145555-page) | Static public presentation site |
| [\`test-manager-library-3145555-infra\`](https://github.com/code-sena/test-manager-library-3145555-infra) | Containers, network, volumes, and empty database engine |
| [\`test-manager-library-3145555-tool\`](https://github.com/code-sena/test-manager-library-3145555-tool) | Utilities: demonstration data seeder |
