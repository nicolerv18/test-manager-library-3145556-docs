# Translation and decisions log

Traceability for the Laravel + React version of the spec. It records **what was changed relative to the reference (`spec-.net/`), why,
and what it affects**. The scope of the product (actors, user stories HU-01..HU-10, acceptance criteria, business rules RN-1..RN-6,
non-functional requirements RNF-1..RNF-5, out-of-scope list) is **unchanged**.

Reference used: `spec-.net/` (business content is identical in `spec-python/`; verified with `diff -r`, which shows differences only in
framework, ORM, hash algorithm and test-tool wording).

## 1. Translation decisions (TD)

| ID | Decision | Reason | Affects |
|---|---|---|---|
| TD-01 | New self-contained folder `spec-laravel-react/`; `spec-.net/` and `spec-python/` are left untouched | Keeps the original as a read-only reference and makes this diff reviewable | Whole folder |
| TD-02 | **Documentation is English.** User-facing API messages, category names (`Novela`, `Ciencia`, …) and UI copy stay **Spanish** | The messages are observable behavior (CA-01.2, RNF-4, contract §1) and category names are seed *data*; translating them would change behavior, not documentation | Constitution Art. XI |
| TD-03 | Hexagonal architecture is kept; persistence is an **adapter** only (PDO/SQL, see TD-10), domain is pure PHP | Constitution Arts. I–III are about dependency direction, not about the language | architecture.md |
| TD-04 | Tooling mapping (below) | Technology is translated, business is not | plan.md, constitution.md, tasks.md |
| TD-05 | Auth is **JWT** behind the `TokenIssuer` port, not Sanctum/Passport | Plan D-11 already says JWT HS256; Sanctum would add a `personal_access_tokens` table, contradicting "six tables" | plan.md, data-model.md §0.1 |
| TD-06 | Concurrency uses an atomic conditional `UPDATE` instead of a version token | The reference's version token has no column in `data-model.md`; the update gives the same observable guarantee with no schema change | ADR-002, architecture.md §4 |
| TD-07 | Files renamed to English: `arquitectura-panoramica.md` → `architecture-overview.md`; ADR filenames translated | Consistency with the English documentation | Links inside the folder |
| TD-08 | New `frontend.md` | The reference leaves the React side as a single "manual walkthrough" task (T-22); a spec-driven build needs it specified | frontend.md |
| TD-09 | `DATABASE_URL` → Laravel `DB_*` variables; added `APP_ENV`, `VITE_API_BASE_URL` | Framework-native configuration for the same information | plan.md §2 |
| TD-10 | Persistence uses plain SQL through PDO (Laravel's connection) instead of Eloquent | The critical rules are SQL (atomic conditional `UPDATE`, `CHECK`, partial indexes, trigram search); writing them directly keeps them reviewable, testable against a real PostgreSQL without the framework, and avoids Eloquent's plural tables, timestamps and implicit behavior (C-10) | plan D-03, architecture.md |

### Tooling mapping

| Concern | .NET reference | Python reference | This version |
|---|---|---|---|
| Language | C# 12 / .NET 8 | Python 3.12 | PHP 8.3 |
| HTTP | ASP.NET Core Minimal API | FastAPI | Laravel (API-only) |
| ORM / migrations | EF Core 8 + EF migrations | SQLAlchemy 2 + Alembic | PDO/SQL (adapter) + one Laravel migration running `database/sql/library.sql` |
| Password hash | BCrypt (12) | Argon2id | bcrypt (cost 12) |
| Token | JWT HS256, 8 h | JWT HS256, 8 h | JWT HS256, 8 h (`firebase/php-jwt`) |
| Unit / app tests | xUnit | pytest | Pest |
| Architecture tests | NetArchTest | import-linter | Pest `arch()` |
| Composition point | `PortBindings.cs` | `port_bindings.py` | `PortBindingsServiceProvider.php` |
| Frontend | — (out of reference) | — | React 18 + Vite + TypeScript |

## 2. Clarifications (C) — the reference was silent or ambiguous

None of these adds, removes or widens scope. Entries marked **confirm** are the author's proposed reading and should be confirmed by the
product owner (the instructor) if they ever answer a question; until then they are the working rule.

| ID | Gap in the reference | Resolution | Where | Status |
|---|---|---|---|---|
| C-01 | Report/loan filters take dates, but `loaned_at` is an instant; "both inclusive" unspecified for instants | Dates (`YYYY-MM-DD`) or UTC instants accepted; both ends inclusive; date-only `to` = end of that day, UTC | contract §1, HU-08, HU-10 | confirm |
| C-02 | Order of validation on `POST /loans` undefined; CA-06.4 says "with the current count" but not where | Fixed 8-step order; count shown in `details[0].issue`; compared value is *unreturned + requested* | contract §3.6, HU-06 | confirm |
| C-03 | Unknown / deleted `book_id` or unknown `member_id` in a loan: no code defined | 404 `NOT_FOUND`, with the offending field in `details` | contract §3.6, HU-06 | confirm |
| C-04 | `days_late` rounding undefined | Whole 24-hour periods since `due_at`, rounded down (a line 3 hours late shows 0) | contract §3.6, HU-09 | confirm |
| C-05 | `PUT /books/{id}`: ISBN editable? new total below copies out? | ISBN immutable (ignored if sent); total that would make available < 0 → 422 `VALIDATION_ERROR` | contract §3.4, HU-03, data-model | confirm |
| C-06 | RNF-1 (300 ms / 5,000 books) cannot be met by the listed indexes for partial text; case sensitivity unspecified; behavior for `size` > 50 unspecified | Case-insensitive match + partial trigram GIN indexes (`pg_trgm`) in the initial migration; `size` > 50 → 422 | data-model §4, contract §3.4, HU-04 | confirm (`size` > 50 rule) |
| C-07 | Cover route needs a Bearer token but `<img>` cannot send one; oversize/wrong type response; cover replacement | Frontend fetches as blob; oversize/wrong type → 422 `VALIDATION_ERROR`; old binary removed after commit | contract §3.4, §4, frontend §6, HU-03 | proposed |
| C-08 | `/loans/overdue` vs `/loans/{id}` route collision (real in Laravel) | UUID-constrained params and literal routes declared first | contract §1, §3.6 | proposed |
| C-09 | T-21 requires past/overdue loans created "only through the API", but the loan instant comes from the server clock | Optional `X-Demo-Now` header, honored only when `APP_ENV=demo`, ignored elsewhere | contract §5, plan D-14, T-21 | decided by the student |
| C-10 | Laravel's skeleton and framework create tables beyond the "six tables"; Eloquent adds timestamps and plural names | Remove unused skeleton tables; JWT instead of Sanctum; `$timestamps=false`; no Eloquent models (TD-10), so no `$timestamps`/`$table` needed; `migrations` table stays in `public` as framework bookkeeping | data-model §0.1 | accepted |
| C-11 | No error code for missing/expired token or for an unexpected failure | Added `UNAUTHENTICATED` (401) and `INTERNAL_ERROR` (500) to the single error body | contract §2 | confirm |
| C-12 | Double return must "never add twice" but the mechanism is unspecified | Decided atomically with `UPDATE … WHERE returned_at IS NULL` | HU-07, ADR-002, contract §3.6 | proposed |

## 3. Risks and open points

| ID | Risk | Mitigation / next step |
|---|---|---|
| R-1 | Laravel's `migrations` table and a non-default schema `library` can conflict | **Resolved:** the connection keeps `search_path=public`, so `migrations` lives in `public` (framework bookkeeping, not one of the six tables); all application SQL is schema-qualified (`library.*`). Chosen because it needs no framework override and was proven by running the schema against PostgreSQL 16 | resolved |
| R-2 | Laravel's default exception renderers can leak a non-contract body (HTML 404/405/500, validation JSON) | One exception handler maps every `Throwable`; contract test walks error cases (T-16) |
| R-3 | PHP's own upload limits can reject a 2 MB cover before the app sees it | Set `upload_max_filesize` and `post_max_size` ≥ 2 MB in the container; covered by T-15 evidence |
| R-4 | `X-Demo-Now` is a time-forging feature | Honored only for `APP_ENV=demo`; a test proves it is ignored otherwise (T-21); never enabled in `infra` defaults |
| R-5 | Time budget (hard deadline 17:00 on 2026-10-13, no extension) | Reach T-17 with evidence first; commit in small steps |

## 4. Coverage check

Every user story, acceptance criterion, business rule and non-functional requirement of the reference is present in `spec.md` with the
same identifier. Verified mechanically (identifier sets compared against `spec-.net/spec.md`).
