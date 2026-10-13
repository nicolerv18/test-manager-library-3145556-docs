# Tasks — Manager Library (Laravel + React)

Each task is finished when its **evidence** exists and can be executed (Article XII). Initial state of all of them: **pending**.

| # | Task | Depends on | Evidence |
|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | — | `GET /api/v1/health` responds `{"status":"ok"}`; the container starts with one command; unused skeleton migrations/tables removed (data-model §0.1) |
| T-02 | Initial migration: six tables, FKs, unique indexes, trigram indexes and the five seeded categories | T-01 | The queries in `data-model.md` §10 return what is documented; no column has a `DEFAULT`; the location of Laravel's `migrations` table is verified (risk R-1) |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` (value objects) | T-01 | Unit tests: 13-digit ISBN, quantity > 0, end ≥ start |
| T-04 | Domain: `Book` (`withdraw`, `restock`, soft delete) | T-03 | Tests: never < 0, never > total |
| T-05 | Domain: `Loan` (`addItem`, `returnItem`, computed status, 14 days, max 3) | T-04 | Tests with a fake `Clock`: `open`, `overdue`, `closed` |
| T-06 | Domain: `Member` and `User` (username normalization, valid role) | T-03 | `"Ana "` becomes `ana`; role `admin` through the API rejected |
| T-07 | Ports and use cases for **authentication** (`LoginService`, `CreateLibrarianService`) with fakes | T-06 | HU-01 and HU-02 pass against fakes |
| T-08 | Ports and use cases for **books** (create, search, edit, delete, cover) with fakes | T-04 | HU-03 and HU-04 pass against fakes |
| T-09 | **Lend** use case (`LendBooksService`) with fakes | T-05, T-08 | HU-06 complete: limits, overdue, no copies, and the validation order of `api-contract.md` §3.6 |
| T-10 | **Return** and **overdue** use cases | T-09 | HU-07 and HU-09; returning twice → `ITEM_ALREADY_RETURNED`; `days_late` rounds down |
| T-11 | **Most borrowed report** use case and its read port | T-09 | HU-10; the old title is kept after renaming (ADR-004); both range ends inclusive |
| T-12 | Persistence adapter: book, member, user repositories | T-02, T-08 | Integration tests against real PostgreSQL |
| T-13 | Persistence adapter: loan repository + atomic conditional stock update (ADR-002) | T-12, T-09 | Two loans of the last copy: one 201, the other 409 (RNF-2); a failed loan leaves no rows |
| T-14 | Security adapter: password hash (bcrypt, cost 12) and JWT token | T-07 | The password never appears in logs nor responses (RNF-3) |
| T-15 | Cover adapter (disk) | T-08 | Uploading a 2 MB PNG works; 3 MB → 422; no cover → null `cover_key`; PHP upload limits ≥ 2 MB |
| T-16 | HTTP adapter: routes of [`api-contract.md`](api-contract.md), single error body for **every** error (incl. 401/403/404/405/500), role authorization, UUID-constrained routes, `/loans/overdue` before `/loans/{id}` | T-10, T-11, T-14, T-15 | A contract test walks every route and every code in §2 |
| T-17 | Composition and boot (initial admin from the environment) | T-16 | The system starts with an empty database and you can log in as admin |
| T-18 | Push the **domain-only** invariants down to the database: `ck_book_total_positive`, `ck_user_role` | T-12 | A `psql` `INSERT` that violates them fails; `data-model.md` §4 is updated |
| T-19 | Architecture tests (articles I, II and III) | T-17 | Pest `arch()` passes and fails if a forbidden dependency is introduced |
| T-20 | `infra`: `docker compose up` starts an empty engine, the API and the frontend | T-17 | One command; the database starts with no tables until migrations run; no `CREATE TABLE` in `infra` |
| T-21 | `tool`: demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) using `X-Demo-Now` in `APP_ENV=demo` | T-17 | After running it, `/loans/overdue` returns 3 lines and the report has rows; the header is ignored outside demo mode (test) |
| T-22 | `app`: login, catalog, members, lend, return, overdue, report ([`frontend.md`](frontend.md)) | T-16 | Manual end-to-end walkthrough at the fair; role guards verified for admin and librarian |
| T-23 | `page`: static public presentation site | — | Opens without the API; correct on mobile and desktop |

## Definition of done for the test

Whoever reaches T-17 with evidence has the **core** done. T-18 to T-23 separate good work from outstanding work. What is evaluated is the
performance with SDD, not the number of tasks: one task with evidence is worth more than three half-done.
