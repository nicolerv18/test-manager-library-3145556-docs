# Constitution — Manager Library (Laravel + React)

Non-negotiable project principles. They are not style or preference: every article states **how it is verified**, because a principle
nobody can check is only an intention. A change that violates an article is rejected without discussing the rest.

> Adapted from the .NET reference constitution. The twelve articles and their intent are unchanged; only the
> technology-specific wording and the verification tooling were translated to PHP/Laravel. See
> [`translation-and-decisions.md`](translation-and-decisions.md).

---

## Article I — The domain depends on nothing

`app/Domain/` imports no other project module and no third-party library: only core PHP (`readonly` classes, `DateTimeImmutable`,
enums). No Eloquent, no `Illuminate\*`, no Laravel helpers (`now()`, `collect()`, `config()`).

**Why:** it is the only way to test invariants without a database, a server or a container, and to swap the ORM without touching rules.
**Verification:** a Pest architecture test (`arch()`) forbids `App\Domain` from using `App\Application`, `App\Adapters`, `App\Providers`
or `Illuminate`.

## Article II — The core declares its ports; the world implements them

The application layer only knows the domain. Everything it needs from the outside (repositories, clock, password hashing, cover storage,
token issuing) is expressed as a **port (PHP interface) that the application layer defines itself**. No framework or ORM type appears in
a port signature.

**Verification:** a Pest architecture test forbids `App\Application` from using `App\Adapters` and `Illuminate`. Application tests run
with fakes and never reference `App\Adapters`.

## Article III — A single composition point

One class (`app/Providers/PortBindingsServiceProvider.php`) is where a port meets its adapter. Outside it, no infrastructure is
instantiated and no adapter class is referenced.
**Verification:** an architecture test asserts that classes under `App\Adapters\Outbound` are referenced only by that provider (and by
integration tests).

## Article IV — SOLID is not decorative

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One application service = one use case. `LendBooksService` lends and does nothing else | A service with two reasons to change is split |
| **OCP** | Changing cover storage (disk → S3) touches neither domain nor application | The new adapter only adds one line in the provider |
| **LSP** | Test fakes replace the real adapter without changing expectations | The same test passes against the fake and the real one |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementor leaves methods unused |
| **DIP** | The port is defined by its consumer | Ports live in the application namespace |

## Article V — The schema belongs to the migrations

The `infra` repository starts an **empty** engine. All DDL (tables, indexes, constraints, extensions and seed data) lives in the
application's Laravel migrations. See [ADR-001](adr/adr-001-schema-ownership.md).
**Verification:** `infra` contains no `CREATE TABLE`.

## Article VI — Invariants the database can enforce, the database enforces

A rule that lives only in code protects the application, not the data. Every invariant expressible in the engine (`CHECK`, `UNIQUE`, FK)
is declared there. Whatever is not enforced is marked **pending** in [`data-model.md`](data-model.md); it is never promised.
**Verification:** the queries in `data-model.md` §10 match what the document declares.

## Article VII — Derived data is computed, not stored

A loan's status (`open`, `overdue`, `closed`) and the days late are computed. There is no column for them.
**Verification:** `data-model.md` lists no `status` or `days_late` column.

## Article VIII — Frozen values do not follow the catalog

A loan line stores the title, ISBN and category of the moment of the loan. No report reads them from the live catalog.
**Verification:** a test that renames a book after lending it and checks that the report keeps the old title
([ADR-004](adr/adr-004-aggregated-frozen-report.md)).

## Article IX — No secret enters the repository or a log

No passwords, no precomputed hashes, no token signing key. Every secret comes from the environment. The real `.env` is in `.gitignore`;
only `.env.example` is versioned.
**Verification:** `gitleaks` or a pattern `grep` in CI; no HTTP response includes `password` or `password_hash`.

## Article X — The engine wins

If the data document and the database disagree, the document is broken and the document is fixed.

## Article XI — Language

Code, columns and routes in English. Documentation in English. **User-facing messages returned by the API stay in Spanish** — that is part
of the observable contract ([`api-contract.md`](api-contract.md) §1), not documentation.

> Changed from the reference: the original said "documentation in Spanish". Documentation is now English; the end-user message language
> is unchanged. Rationale in [`translation-and-decisions.md`](translation-and-decisions.md), entry TD-02.

## Article XII — Nothing is done without evidence

A task in [`tasks.md`](tasks.md) is finished when its **evidence** exists and can be executed, not when "it is already coded".
