# Specification — Manager Library

**What is built and why.** A system for a small library to keep its book catalog, its members and its loans, and to see which books are
borrowed most.

Governed by [`constitution.md`](constitution.md). The data model lives only in [`data-model.md`](data-model.md)
and the HTTP contract only in [`api-contract.md`](api-contract.md).

---

## 1. Actors

| Actor | What they do | How they authenticate |
|---|---|---|
| **Administrator** (`admin`) | Maintains the book catalog and the members; creates librarians; sees all reports | Username and password → token |
| **Librarian** (`librarian`) | Registers loans and returns; browses catalog, members and overdue books | Username and password → token |
| **Member** | Person who borrows books. **Does not log in**: is data, not a user | — |

The `admin` role is provisioned by the deployment from environment variables. **Nobody grants it at runtime.**

## 2. Scope

**In:** book catalog with optional cover, five fixed categories, members, loans with several lines, return per line, overdue books,
most borrowed report, login with roles.

**Out, and not reopened:** fines and payments, reservations, renewals, email or SMS, bulk import, loans between branches, book
attributes beyond those in §3 (no publisher, no year, no language).

## 3. User stories and acceptance criteria

### HU-01 · Log in
As a user I want to authenticate so that I operate according to my role.
- **CA-01.1** With valid credentials a token valid for 8 hours is received.
- **CA-01.2** With invalid credentials the response is 401 with a message that does **not** say whether the username or the password failed.
- **CA-01.3** Every endpoint except login and `/health` requires a token. No token: 401. Insufficient role: 403.

### HU-02 · Create librarians (admin)
- **CA-02.1** The admin creates a user with username, password and role `librarian`.
- **CA-02.2** The username is stored lowercase and trimmed; it is unique. Repeated: 409.
- **CA-02.3** Requesting the `admin` role through the API responds 422. The password is never returned nor written to logs.

### HU-03 · Manage books (admin)
- **CA-03.1** Create a book with title, author, ISBN, category and total copies. Available copies start equal to the total.
- **CA-03.2** The ISBN has 13 digits and is unique among active books. Repeated: 409.
- **CA-03.3** Total copies ≥ 1. Changing the total adjusts the available copies by the same difference and **never** leaves available < 0 (422).
- **CA-03.4** Uploading a cover (JPG/PNG, max 2 MB) stores an **opaque key**; without a cover the field is `null`, never an empty string.
- **CA-03.5** Deleting a book is a **soft delete**: it disappears from the catalog but stays in past loans. A book with open loans is not deleted (409).

### HU-04 · Search books
- **CA-04.1** Partial-text search on title or author, category filter, active books only, ordered by title, paginated (`page`, `size`, max 50).
- **CA-04.2** The response includes the total number of results.
- **CA-04.3** Fetching a book by a non-existent or deleted id: 404.

### HU-05 · Manage members
- **CA-05.1** Admin and librarian create members with name and document. The document is unique: repeated 409.
- **CA-05.2** Members are listed and searched by name or document, paginated.
- **CA-05.3** Members are not deleted.

### HU-06 · Register a loan
- **CA-06.1** A loan belongs to **one** member, is registered by the authenticated user and has 1 to 3 lines.
- **CA-06.2** The same book cannot repeat within a loan (422).
- **CA-06.3** Each line subtracts **1** available copy in the same operation. No copies: 409 and **nothing is registered**.
- **CA-06.4** A member cannot have more than **3 unreturned books** in total (422 with the current count).
- **CA-06.5** A member with any **overdue** book cannot borrow another (409).
- **CA-06.6** The due date is `loaned_at + 14 days`. The line freezes title, ISBN and category name **at that instant**.
- **CA-06.7** A registered loan is neither edited nor deleted.

### HU-07 · Register a return
- **CA-07.1** One **line** is returned at a time. It sets `returned_at` and adds 1 to the available copies.
- **CA-07.2** Returning an already returned line: 409. It **never** adds twice.
- **CA-07.3** Available copies never exceed the total (the database guarantees it).
- **CA-07.4** When all lines are returned, the loan is shown as **closed**.

### HU-08 · View a loan and list loans
- **CA-08.1** Fetching a loan returns its lines and its computed status (`open`, `overdue`, `closed`).
- **CA-08.2** List by date range, status and member, ordered by date descending, paginated.
- **CA-08.3** The status is **computed**, not stored.

### HU-09 · Overdue books
- **CA-09.1** Lists the unreturned lines whose `due_at` is earlier than now: book, member, days late.
- **CA-09.2** Ordered by most days late first. Only `admin` and `librarian`.

### HU-10 · Most borrowed books report
- **CA-10.1** Given a date range, one row per **book and frozen category name**, with the number of times borrowed, ordered high to low.
- **CA-10.2** End before start: 422. A range with no loans: empty list, not an error.
- **CA-10.3** The report is **not** broken down by librarian or by member.
- **CA-10.4** A report for a closed period **does not change** even if the book or its category is later renamed.
- **CA-10.5** `admin` only.

## 4. Business rules

| # | Rule |
|---|---|
| RN-1 | Available copies between 0 and total, always. |
| RN-2 | Loan period: 14 days. Maximum 3 unreturned books per member. |
| RN-3 | A member with overdue books cannot borrow more until they are returned. |
| RN-4 | What is frozen in the line (title, ISBN, category) does not follow the catalog. |
| RN-5 | Loans and returns are never deleted. Books only have soft delete. |
| RN-6 | Dates are UTC instants. Messages to the user in Spanish; code and columns in English. |

## 5. Non-functional requirements

| # | Requirement | How it is measured |
|---|---|---|
| RNF-1 | Book search responds in under 300 ms with 5,000 books | Load test with the `tool` seeder |
| RNF-2 | Two librarians lending the **last copy** at the same time: one wins, the other gets 409 | Concurrent test |
| RNF-3 | The password never travels nor is stored in clear text; it never appears in logs | Review of logs and responses |
| RNF-4 | Every HTTP error uses the same body (see contract) | Contract test |
| RNF-5 | The system starts with a single command and the database starts empty | `infra` `README` |
