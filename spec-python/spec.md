# Specification — Manager Library

**What is being built and why.** A system that enables a small library to manage its book catalog,
members, and loans, and to see which books are borrowed most often.

It is governed by [`constitution.md`](constitution.md). The data model lives only in
[`data-model.md`](data-model.md), and the HTTP contract lives only in
[`api-contract.md`](api-contract.md).

---

## 1. Actors

| Actor | Responsibilities | Authentication |
|---|---|---|
| **Administrator** (`admin`) | Maintains the book catalog and members; creates librarians; views all reports | Username and password → token |
| **Librarian** (`librarian`) | Records loans and returns; consults the catalog, members, and overdue items | Username and password → token |
| **Member** | Person who borrows books. **Does not log in to the system**: this is data, not a user | — |

The `admin` role is provisioned during deployment from environment variables. **It is never
granted at runtime.**

## 2. Scope

**Included:** book catalog with optional covers, five fixed categories, members, loans with multiple
items, return by item, overdue books, most-borrowed report, and role-based login.

**Excluded, and not open for reconsideration:** fines and payments, reservations, renewals, email or
SMS, bulk import, inter-branch loans, and book attributes beyond those in §3 (no publisher, year, or
language).

## 3. User stories and acceptance criteria

### HU-01 · Log in
As a user, I want to authenticate so I can operate according to my role.

- **CA-01.1** Valid credentials return a token with an eight-hour lifetime.
- **CA-01.2** Invalid credentials return 401 with a message that **does not reveal** whether the
  username or password failed.
- **CA-01.3** Every endpoint except login and `/health` requires a token. No token: 401.
  Insufficient role: 403.

### HU-02 · Create librarians (administrator)

- **CA-02.1** The administrator creates a user with a username, password, and `librarian` role.
- **CA-02.2** The username is trimmed, converted to lowercase, and unique. A duplicate returns 409.
- **CA-02.3** Requesting the `admin` role through the API returns 422. The password is never returned
  or written to logs.

### HU-03 · Manage books (administrator)

- **CA-03.1** Create a book with title, author, ISBN, category, and total copies. Available copies
  initially equal total copies.
- **CA-03.2** The ISBN has 13 digits and is unique among active books. A duplicate returns 409.
- **CA-03.3** Total copies ≥ 1. Changing the total adjusts available copies by the same difference and
  **never** leaves available copies below 0 (422).
- **CA-03.4** Uploading a cover (JPG/PNG, maximum 2 MB) stores an **opaque key**; without a cover, the
  field is `null`, never an empty string.
- **CA-03.5** Deleting a book is a **logical deletion**: it no longer appears in the catalog but
  remains in past loans. A book with open loans cannot be deleted (409).

### HU-04 · Search books

- **CA-04.1** Partial-text search in title or author, category filter, active books only, ordered by
  title, and paginated (`page`, `size`, maximum 50).
- **CA-04.2** The response includes the total number of results.
- **CA-04.3** Requesting a nonexistent or logically deleted book by ID returns 404.

### HU-05 · Manage members

- **CA-05.1** Administrators and librarians create members with a full name and document number. The
  document number is unique; a duplicate returns 409.
- **CA-05.2** Members can be listed and searched by name or document number, with pagination.
- **CA-05.3** Members are not deleted.

### HU-06 · Record a loan

- **CA-06.1** A loan belongs to **one** member, is recorded by the authenticated user, and contains
  between one and three items.
- **CA-06.2** The same book cannot appear more than once in a loan (422).
- **CA-06.3** Each item decrements available copies by **1** in the same operation. If no copies are
  available, return 409 and **record nothing**.
- **CA-06.4** A member cannot have more than **three unreturned books** in total (422 with the current
  count).
- **CA-06.5** A member with any **overdue** book cannot borrow another book (409).
- **CA-06.6** The due date is `loaned_at + 14 days`. The item freezes the title, ISBN, and category
  name **at that instant**.
- **CA-06.7** A recorded loan cannot be edited or deleted.

### HU-07 · Record a return

- **CA-07.1** Return **one item** at a time. Set `returned_at` and increment available copies by 1.
- **CA-07.2** Returning an already returned item produces 409. It **never** increments twice.
- **CA-07.3** Available copies never exceed total copies; the database guarantees this.
- **CA-07.4** When every item has been returned, the loan appears as **closed**.

### HU-08 · View and list loans

- **CA-08.1** Requesting a loan returns its items and calculated status (`open`, `overdue`,
  `closed`).
- **CA-08.2** List by date range, status, and member, ordered by date descending, with pagination.
- **CA-08.3** Status is **calculated**, not stored.

### HU-09 · Overdue books

- **CA-09.1** List unreturned items whose `due_at` is earlier than now: book, member, and days late.
- **CA-09.2** Order by the greatest number of days late first. Available only to `admin` and
  `librarian`.

### HU-10 · Most-borrowed books report

- **CA-10.1** For a date range, return one row per **book and frozen category name**, with the number
  of times borrowed, ordered from highest to lowest.
- **CA-10.2** An end date earlier than the start date returns 422. A range with no loans returns an
  empty list, not an error.
- **CA-10.3** The report is **not** broken down by librarian or member.
- **CA-10.4** A report for a closed period **does not change** if the book or its category is renamed
  later.
- **CA-10.5** Available only to `admin`.

## 4. Business rules

| # | Rule |
|---|---|
| RN-1 | Available copies are always between 0 and total copies. |
| RN-2 | Loan period: 14 days. Maximum three unreturned books per member. |
| RN-3 | A member with overdue items cannot borrow more until they are returned. |
| RN-4 | Values frozen in the item (title, ISBN, category) do not follow the catalog. |
| RN-5 | Loans and returns are not deleted. Books use logical deletion only. |
| RN-6 | Dates are UTC instants. User-facing messages are in Spanish; code and columns are in English. |

## 5. Non-functional requirements

| # | Requirement | Measurement |
|---|---|---|
| RNF-1 | Book search responds in under 300 ms with 5,000 books | Load test using the `tool` seeder |
| RNF-2 | When two librarians simultaneously loan the **last copy**, one succeeds and the other receives 409 | Concurrency test |
| RNF-3 | A password is never transmitted or stored in plain text and never appears in logs | Review logs and responses |
| RNF-4 | Every HTTP error uses the same body (see the contract) | Contract test |
| RNF-5 | The system starts with one command and the database starts empty | `infra` README |
