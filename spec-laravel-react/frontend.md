# Frontend — Manager Library (React)

How the `app` repository delivers the user stories of [`spec.md`](spec.md) through the contract in [`api-contract.md`](api-contract.md).
The reference spec stops at "T-22: manual walkthrough"; this document gives that task a specification of its own. It adds **no** feature:
every screen below maps to an existing user story.

## 1. Stack and conventions

- React 18 + Vite + TypeScript, React Router, plain `fetch` wrapped in one API client module.
- Code, identifiers and routes in English. **UI copy shown to the end user is Spanish** (RN-6); the API already returns Spanish messages.
- Base URL from `VITE_API_BASE_URL`. No secret is ever bundled.

## 2. Screens ↔ user stories

| Route | Screen | Roles | Stories |
|---|---|---|---|
| `/login` | Login form | public | HU-01 |
| `/books` | Catalog: search box (`q`), category filter, pagination, total count, cover thumbnails | admin, librarian | HU-04 |
| `/books/new`, `/books/:id/edit` | Create / edit book, upload or remove cover, soft delete | admin | HU-03 |
| `/members` | List and search members, create member | admin, librarian | HU-05 |
| `/loans/new` | Lend: pick a member, add 1–3 distinct books, submit | admin, librarian | HU-06 |
| `/loans` | Loan list with date range, status and member filters | admin, librarian | HU-08 |
| `/loans/:id` | Loan detail with its lines and a per-line **Return** button | admin, librarian | HU-07, HU-08 |
| `/overdue` | Overdue lines sorted by days late | admin, librarian | HU-09 |
| `/reports/most-borrowed` | Date-range form and result table | admin | HU-10 |
| `/users/new` | Create a librarian | admin | HU-02 |

## 3. Authentication and guards

- On login the token and `role` are kept in `sessionStorage` (cleared when the tab closes, smaller exposure than `localStorage`).
- A `RequireAuth` wrapper redirects to `/login` when there is no token. A `RequireRole` wrapper hides routes and menu entries the role cannot
  use. **This is convenience only**: the API is the real authority and answers 403.
- Any response with status 401 clears the session and returns to `/login`. A 403 shows the API message and keeps the session.
- The navigation shows only what the role may use (a librarian never sees book create/edit/delete actions, Users or Report).

## 4. API client

One module (`src/api/client.ts`) is the only place that calls `fetch`. It adds `Authorization: Bearer`, parses JSON and converts every
non-2xx response into a typed `ApiError { status, code, message, details }` using the single error body. Screens never read raw responses.

## 5. Error handling

| Case | Behavior |
|---|---|
| 422 with `details[].field` | Show each `issue` next to its field (e.g. `items[0].book_id`) |
| 409 / 422 business errors | Show `message` in a visible alert; for `MEMBER_LIMIT_EXCEEDED` and `MEMBER_HAS_OVERDUE` explain why the lend cannot continue |
| 401 | See §3 |
| Network failure / 500 | Generic Spanish message; never print traces |

The Lend form disables submit while a request is in flight (prevents double submits) and caps the book selector at 3 distinct books, but
always trusts the server's verdict.

## 6. Covers

`GET /books/{id}/cover` requires the Bearer token, so `<img src>` cannot call it. A `Cover` component fetches the image through the API
client, turns it into a `Blob`, uses `URL.createObjectURL`, and revokes the URL on unmount. A 404 (no cover) renders a neutral placeholder,
not an error. Upload uses `multipart/form-data` field `file`, with a client-side pre-check of type (JPG/PNG) and size (≤ 2 MB) — the server
re-validates.

## 7. Dates

The API sends UTC instants with `Z`. The UI formats them in the user's local time zone for display only and sends date filters as
`YYYY-MM-DD` (inclusive, UTC semantics per the contract). Never compute due dates or status on the client: show `due_at`, `status` and
`days_late` exactly as received.

## 8. Quality bar

- Responsive from phone to desktop; keyboard-reachable forms; visible focus; every input has a label.
- Loading, empty and error states exist for every list.
- Smoke tests (Vitest + React Testing Library): role guards, the lend flow happy path, and the mapping of one 422 to a field message.
- Evidence for T-22 is the end-to-end walkthrough: login → search → lend → return → overdue → report, as admin and as librarian.
