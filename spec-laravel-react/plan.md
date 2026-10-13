# Technical plan — Manager Library (Laravel + React)

How to build what [`spec.md`](spec.md) and [`constitution.md`](constitution.md) describe. The schema is **not** described here: it lives in
[`data-model.md`](data-model.md). The shape of the system is in [`architecture.md`](architecture.md); the UI in [`frontend.md`](frontend.md).

## 1. Technical decisions

| # | Decision | Choice (Laravel + React) | Why |
|---|---|---|---|
| D-01 | Backend language and runtime | PHP 8.3 | Challenge stack |
| D-02 | HTTP framework | Laravel (current stable) in API-only mode | Contract in [`api-contract.md`](api-contract.md) |
| D-03 | Persistence | Plain SQL through PDO (Laravel's connection) as an **adapter only** + one Laravel migration that applies `database/sql/library.sql`, on PostgreSQL 16 | The schema belongs to the migrations (ADR-001) |
| D-04 | Concurrency | Atomic conditional `UPDATE` on the counter + `CHECK` range ([ADR-002](adr/adr-002-optimistic-concurrency.md)) | Last copy (RNF-2); needs no extra column |
| D-05 | Dates | UTC instants (`timestamptz`); the clock is a port (`Clock`) | Test due dates without waiting 14 days |
| D-06 | Report | One aggregated query through a read port ([ADR-004](adr/adr-004-aggregated-frozen-report.md)) | Stable and cheap |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: live inside their owner's row, no table | They have no identity |
| D-08 | Covers | Binary in external storage (local disk in dev, S3 in production); the database stores only `cover_key` | Opaque key, never a path |
| D-09 | Passwords | bcrypt (cost 12) behind the `PasswordHasher` port; the domain never sees the plain password | Article IX |
| D-10 | Bootstrap | An Artisan command / boot step creates the initial admin from `ADMIN_USERNAME` / `ADMIN_PASSWORD` | The database cannot produce the hash |
| D-11 | Authentication | JWT HS256, 8 hours, key in `JWT_SECRET`, behind the `TokenIssuer` port (library: `firebase/php-jwt`). **Not** Sanctum/Passport | Stateless, and avoids extra framework tables (see [`data-model.md`](data-model.md) §0.1) |
| D-12 | Backend tests | Pest, with fakes for ports and a real database only in integration tests | Test pyramid |
| D-13 | Frontend | React 18 + Vite + TypeScript, React Router, `fetch`-based API client | Challenge stack; `CORS_ORIGIN` default is the Vite dev port |
| D-14 | Demo clock | `X-Demo-Now` request header honored **only** when `APP_ENV=demo` | The seeder must create past loans using only the API (T-21) |

## 2. Configuration per environment

| Variable | Required | Example |
|---|---|---|
| `DB_CONNECTION` / `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | yes | `pgsql` / `db` / `5432` / `library` / `library` / (from `.env`) |
| `JWT_SECRET` | yes | long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | yes | defined in `.env`, never in the repo |
| `COVERS_DIR` | no | `storage/app/covers` |
| `CORS_ORIGIN` | yes | `http://localhost:5173` |
| `APP_ENV` | yes | `local`, `demo` or `production`; only `demo` enables `X-Demo-Now` |

Frontend: `VITE_API_BASE_URL` (default `http://localhost:8000/api/v1`).

> The reference used a single `DATABASE_URL`. Laravel reads separate `DB_*` variables (a `DB_URL` is also supported). Same information,
> framework-native form.

## 3. Test strategy

| Level | What it tests | Against what |
|---|---|---|
| Domain | Invariants: max 3, due date at 14 days, available within range | Nothing: pure objects |
| Application | Use cases: lend, return, report | Fakes of the ports |
| Integration | Repositories and the report query | Real PostgreSQL |
| Contract | The single error body and status codes | The running API |
| Architecture | Articles I, II and III | Pest `arch()` |
| Concurrency | Two loans of the last copy | Real PostgreSQL |
| Frontend | Role guards and the lend flow | Vitest + React Testing Library (smoke level) |

## 4. Build order

Skeleton and health → initial migration with seed → domain → use cases with fakes → adapters → API → contract and concurrency tests → infra →
seeder → frontend → public page. The detail, with evidence, is in [`tasks.md`](tasks.md).
