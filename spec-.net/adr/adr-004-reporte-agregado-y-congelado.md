# ADR-004 · The report is aggregated by the engine and uses frozen values

- **Status:** accepted
- **Context:** the most-borrowed report (HU-10) joins `loan` and `loan_item` over a range. If it
  read the title or category from the live catalog, renaming a book or category would change a report
  for a closed period.
- **Decision:** (1) the report is **not persisted**: it is calculated through a single aggregate
  query behind a read port. (2) It groups by `book_id`, `book_title`, and `category_name` as
  frozen in the item. (3) `loan_item.category_name` intentionally has **no FK**. (4) If a book
  changed category within the range, the report shows **two rows**; it does not choose a winner.
- **Rejected alternatives:** joining the live catalog (the report is no longer stable); taking “the
  latest value in the range” (a new loan changes what was already read); materializing a report table
  (a second source of truth).
- **Consequences:** CA-10.1 says “one row per book and frozen category name,” not “one row per
  book.” The report includes neither members nor librarians (privacy).
- **Verification:** loan a book, rename it, and request the same-range report: it retains the old
  title.
