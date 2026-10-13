# ADR-003 · Books are soft-deleted, not removed

- **Status:** accepted
- **Context:** a loan line points to a book. If the book is physically deleted, the history and the report break.
- **Decision:** the book carries `deleted_at`. Deleting fills it; the catalog filters by `deleted_at IS NULL` in a single place
  (global filter). A book with open loans is **not** deleted (409 `BOOK_HAS_OPEN_LOANS`). ISBN uniqueness is a **partial**
  index over active books, so a deleted ISBN can be registered again.
- **Rejected alternatives:** physical delete with cascade (destroys the history); an `active = false` flag (does not say when).
- **Consequences:** `loan_item.book_id` carries a `RESTRICT` FK as a **last-resort barrier**: a manual `DELETE` or a future code change
  fails loudly instead of corrupting the history. The cover binary is the only datum that is physically deleted.
- **How it is verified:** fetching a deleted book returns 404; a `DELETE` through `psql` over a book with lines fails on the FK.
