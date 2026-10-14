# ADR-003 · Books are logically deleted, not physically deleted

- **Status:** accepted
- **Context:** a loan item points to a book. Physically deleting the book breaks history and reports.
- **Decision:** a book has `deleted_at`. Logical deletion sets it; the catalog filters
  `deleted_at IS NULL` in one place (a global filter). A book with open loans cannot be deleted
  (`409 BOOK_HAS_OPEN_LOANS`). ISBN uniqueness uses a **partial** index on active books, so an ISBN
  from a logically deleted book can be registered again.
- **Rejected alternatives:** physical deletion with cascade (destroys history); an `active = false`
  flag (does not state when the book was deleted).
- **Consequences:** `loan_item.book_id` has a `RESTRICT` FK as a **last-resort barrier**: a manual
  `DELETE` or future code change fails loudly instead of corrupting history. The cover binary is
  the only data that is physically deleted.
- **Verification:** requesting a logically deleted book returns 404; a `DELETE` through `psql` on
  a book with items fails because of the FK.
