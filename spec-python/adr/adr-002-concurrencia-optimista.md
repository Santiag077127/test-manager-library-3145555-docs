# ADR-002 · Atomic conditional updates for copies

- **Status:** accepted
- **Context:** two librarians can loan the **last copy** of the same book at the same time (RNF-2).
  If both read \`available_copies = 1\` and both write, the system loans a book it does not have.
- **Decision:** every decrement uses an atomic conditional update that affects an active book only if
  \`available_copies > 0\`. The adapter verifies that exactly one row was affected. Otherwise, it
  rolls back the complete transaction and the server returns **409 \`BOOK_NOT_AVAILABLE\`**. Creating
  the loan, its items, and all decrements occurs in the same transaction. The **final barrier** is
  the \`CHECK\` \`ck_book_available_range\` (\`0 <= available_copies <= total_copies\`).
- **Rejected alternatives:** a version column, because it is absent from the documented physical
  model; pessimistic row locking (\`SELECT … FOR UPDATE\`) for every loan, because it serializes the
  whole library for a rare case; an in-memory counter, because it does not survive more than one
  process.
- **Consequences:** no \`version\`, \`updated_at\`, or other column is added to the model. If the
  \`CHECK\` fails, **something wrote outside the adapter**: that is a defect, not a business case.
  Returning the same item twice does not add two copies because \`returned_at\` is already populated
  (CA-07.2).
- **Verification:** a concurrency test sends two simultaneous requests for a book with one copy:
  exactly one receives 201.
