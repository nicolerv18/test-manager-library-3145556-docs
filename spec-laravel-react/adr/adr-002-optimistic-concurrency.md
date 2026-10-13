# ADR-002 · Optimistic concurrency on copies (atomic conditional update)

- **Status:** accepted — **adapted for Laravel** (see "Change from the reference")
- **Context:** two librarians may lend the **last copy** of the same book at the same time (RNF-2). If both read `available_copies = 1` and
  both write, the system lends a book it does not have.
- **Decision:** **optimistic** concurrency with no lock held. The decrement is a single conditional statement executed inside the loan's
  transaction:

  ```sql
  UPDATE library.book
     SET available_copies = available_copies - 1
   WHERE id = :id AND deleted_at IS NULL AND available_copies > 0;
  ```

  If it affects **0 rows**, someone else took the last copy (or the book vanished): the transaction rolls back and the server responds
  **409 `BOOK_NOT_AVAILABLE`**, registering nothing (CA-06.3). The statement is the arbiter; the in-memory `Book::withdraw` check is only an
  early, friendlier rejection. When a loan has several books, rows are updated in **ascending id order** so two concurrent loans cannot
  deadlock. The **last barrier** is the `CHECK` `ck_book_available_range` (`0 <= available_copies <= total_copies`).
  Returns mirror it: the line is closed with `UPDATE loan_item SET returned_at = :now WHERE id = :id AND returned_at IS NULL`
  (0 rows → 409 `ITEM_ALREADY_RETURNED`, stock untouched), then `available_copies = available_copies + 1 WHERE available_copies < total_copies`.
- **Change from the reference:** the .NET/Python reference used a per-book **version token**. That needs an extra column (not in
  `data-model.md`) or an engine-specific system column that Laravel/Eloquent does not expose. The conditional update gives the same
  observable guarantee ("one wins, the other gets 409") with no schema change.
- **Rejected alternatives:** pessimistic row lock (`SELECT … FOR UPDATE`) on every loan: serializes the whole library for a rare case;
  in-memory counter: does not survive more than one process; a `version` column: adds schema not required by any criterion.
- **Consequences:** if the `CHECK` fires, **something wrote outside the adapter**: that is a defect, not a business case. Returning the
  same line twice does not add twice because `returned_at` is already filled (CA-07.2).
- **How it is verified:** a concurrent test with two simultaneous requests over a book with one copy: exactly one receives 201; and a
  test that returns the same line twice leaves the stock unchanged after the first return.
