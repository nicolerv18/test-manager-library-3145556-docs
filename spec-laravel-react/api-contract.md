# API contract — Manager Library

The HTTP contract is **language-independent**: whoever implements it in React + Laravel must respect it exactly as someone implementing it
in Python or .NET. What each operation does is in [`spec.md`](spec.md); what is stored, in [`data-model.md`](data-model.md).

> Clarifications added in this version are tagged **[C-xx]** (see [`translation-and-decisions.md`](translation-and-decisions.md)).

## 1. Conventions

| Topic | Rule |
|---|---|
| Base | `/api/v1` |
| Format | JSON, UTF-8. Field names in `snake_case` |
| Identifiers | UUID v4 as text |
| Dates | ISO 8601 in UTC, with `Z` (`2026-10-14T14:15:00Z`) |
| Date filters **[C-01]** | `from` / `to` accept a date (`YYYY-MM-DD`) or a UTC instant. Both ends are **inclusive**. A date-only `from` means `00:00:00Z` of that day; a date-only `to` means the end of that day (`< next day 00:00:00Z`). Filters apply to `loan.loaned_at` |
| Money | None. The system handles no amounts |
| Authentication | `Authorization: Bearer <token>` on everything except `POST /auth/login` and `GET /health` |
| Pagination | `?page=1&size=20`. `size` maximum 50. The response is `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Language | Error **messages in Spanish**; fields and codes in English |
| Route parameters **[C-08]** | `{id}`-style parameters are constrained to UUID format, and literal routes (`/loans/overdue`) are declared before parameterized ones, so `overdue` is never captured as an id |

### Single error body (for all errors)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` may be empty. It **never** includes traces, SQL, server paths or the password. The framework's own error formats (Laravel's
default validation and exception JSON, HTML error pages) must never reach the client: every error, including 401, 403, 404, 405 and 500,
is rendered in this body. A generic server error uses `code: "INTERNAL_ERROR"`, status 500.

### Status codes

| Status | When |
|---|---|
| 200 / 201 / 204 | Success / created / no content |
| 401 | No token, invalid or expired token, invalid credentials |
| 403 | Insufficient role |
| 404 | Does not exist (or is soft-deleted) |
| 409 | State conflict: duplicate, no copies, already returned, member with overdue, book with open loans |
| 422 | Validation: invalid field, broken input rule, inverted date range |

## 2. Business error codes

| `code` | Status | Origin |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Any resource |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Any invalid field |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |
| `UNAUTHENTICATED` | 401 | Missing, invalid or expired token **[C-11]** |
| `INTERNAL_ERROR` | 500 | Unexpected failure; no internal detail exposed **[C-11]** |

> **[C-11]** The reference defines 401/500 behavior but names no code for a missing/expired token or for an unexpected failure. These two
> codes complete the single error body (RNF-4); they add no feature.

## 3. Endpoints

### 3.1 Authentication and health

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Returns a token |
| GET | `/health` | public | `{ "status": "ok" }` |

`POST /auth/login`

```json
// request
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Users

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/users` | admin | Creates a librarian |

```json
// request  (role only accepts "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (no password nor password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categories

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/categories` | authenticated | The five categories, ordered by name |

### 3.4 Books

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/books` | authenticated | Search: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | authenticated | Detail |
| POST | `/books` | admin | Creates |
| PUT | `/books/{id}` | admin | Replaces title, author, category and total |
| DELETE | `/books/{id}` | admin | Soft delete |
| PUT | `/books/{id}/cover` | admin | Uploads a cover (`multipart/form-data`, field `file`) |
| DELETE | `/books/{id}/cover` | admin | Removes the cover |

Book (response):

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` (request): `title`, `author`, `isbn`, `category_id`, `total_copies`. The server computes the available copies.

`PUT /books/{id}` **[C-05]**: body `title`, `author`, `category_id`, `total_copies`. The ISBN is immutable and is not part of the body (an
`isbn` field, if sent, is ignored). If the new total would leave `available_copies < 0` (that is, the new total is lower than the copies
currently out), the response is 422 `VALIDATION_ERROR` and nothing changes.

`cover_key` is an opaque key. To paint the cover use `GET /books/{id}/cover`, which returns the binary (`Content-Type: image/jpeg` or
`image/png`) or 404. **[C-07]** This route requires the Bearer token like every other, so a browser `<img src>` cannot call it directly:
the frontend fetches it with the token and renders a blob URL (see [`frontend.md`](frontend.md)).

`GET /books` **[C-06]**: `q` matches title or author, partial and case-insensitive; `size` above 50 is a 422 `VALIDATION_ERROR` (it is not
silently clamped).

### 3.5 Members

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/members` | admin, librarian | List and search: `q` (name or document), `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detail with `active_loans` (count of unreturned) |
| POST | `/members` | admin, librarian | Creates: `full_name`, `document_number` |

### 3.6 Loans

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Registers a loan |
| GET | `/loans` | admin, librarian | List: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/overdue` | admin, librarian | Overdue lines (declared before `/loans/{id}`) |
| GET | `/loans/{id}` | admin, librarian | Detail with lines and status |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Returns one line |

`POST /loans`

```json
// request  (1 to 3 items, no repeated books)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

**Validation order for `POST /loans` [C-02].** The first failing check decides the response; later checks are not evaluated:

| Step | Check | Result |
|---|---|---|
| 1 | Token present and valid; role admin/librarian | 401 / 403 |
| 2 | Body shape: `member_id` is a UUID, `items` has 1 to 3 entries, each with a UUID `book_id` | 422 `VALIDATION_ERROR` |
| 3 | No repeated `book_id` | 422 `DUPLICATE_BOOK_IN_LOAN` |
| 4 | Member exists **[C-03]** | 404 `NOT_FOUND` |
| 5 | Every book exists and is not soft-deleted **[C-03]** | 404 `NOT_FOUND` (`details[].field = items[i].book_id`) |
| 6 | Member has no overdue line | 409 `MEMBER_HAS_OVERDUE` |
| 7 | `unreturned + requested ≤ 3` | 422 `MEMBER_LIMIT_EXCEEDED` (`details[0].issue` states the current count, e.g. `"tiene 2 sin devolver; máximo 3"`) |
| 8 | Stock, decided atomically when writing | 409 `BOOK_NOT_AVAILABLE`; the whole loan rolls back |

`status` is computed: `open` (there are unreturned lines and none overdue), `overdue` (some unreturned and overdue), `closed` (all returned).
"Overdue" means `due_at < now` (strictly earlier).

`GET /loans` returns the paginated envelope with the same loan shape as above; `status` filters by the computed status.

`POST /loans/{loan_id}/items/{item_id}/return` → 200 with the line (`returned_at` filled). An `item_id` that does not belong to `loan_id`
responds 404. A second return of the same line responds 409 `ITEM_ALREADY_RETURNED` and does not touch the stock **[C-12]**.

`GET /loans/overdue` → `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at", "days_late" }`. `days_late` is the whole
number of 24-hour periods since `due_at`, rounded down **[C-04]**. Here the member's name **is** shown, because it is an operational
listing; **the report of §3.7 does not**.

### 3.7 Report

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` and `to` required (date, both inclusive, UTC) |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

One row per frozen `book_id + book_title + category_name`. Order: `times_borrowed` descending, then title. No members nor librarians. Missing
`from`/`to` or `to` before `from`: 422 (`VALIDATION_ERROR` for a missing/invalid value, `INVALID_DATE_RANGE` for an inverted range).

## 4. Security

| Aspect | Rule |
|---|---|
| Token | Signed JWT (HS256), 8 hours, with `sub` and `role`. No personal data inside |
| Passwords | Only the hash is stored; the hash never leaves in any response |
| Authorization | Evaluated on each endpoint by role; a 403 does not reveal whether the resource exists |
| CORS | Only the frontend origin configured through the environment |
| Cover upload | Only `image/jpeg` and `image/png`, max 2 MB; the type is validated by content, not by extension. The PHP upload limits (`upload_max_filesize`, `post_max_size`) must be set ≥ 2 MB so the contract error (422), not a PHP failure, is what the client sees |

## 5. Demo mode (non-production only) **[C-09]**

To let the `tool` seeder create past and overdue loans **using only the API** (T-21), the API honors an optional request header:

```
X-Demo-Now: 2026-09-20T10:00:00Z
```

- It is read **only when `APP_ENV=demo`**. In any other environment the header is ignored (not an error), so it cannot be abused in
  production.
- When honored, it replaces the `Clock` port's "now" for that request (affecting `loaned_at`, `due_at` and `returned_at`).
- Must be a valid UTC instant; otherwise 422 `VALIDATION_ERROR` in demo mode.
- It adds no endpoint and no field to any response. A test must prove it is ignored when `APP_ENV` is not `demo`.
