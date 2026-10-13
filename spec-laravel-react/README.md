# Manager Library — spec for Laravel + React

This folder is the version of the specification the challenge is built from: **PHP (Laravel)** backend and **React** frontend. It is derived
from [`../spec-.net/`](../spec-.net/) (the business content of `spec-python/` is identical). What changed and why is in
[`translation-and-decisions.md`](translation-and-decisions.md).

## Reading order

1. [`constitution.md`](constitution.md) — non-negotiable principles.
2. [`spec.md`](spec.md) — actors, user stories, acceptance criteria, business rules.
3. [`architecture-overview.md`](architecture-overview.md), [`architecture.md`](architecture.md) and [`plan.md`](plan.md) — how it is built.
4. [`data-model.md`](data-model.md) and [`api-contract.md`](api-contract.md) — the data and the HTTP contract.
5. [`frontend.md`](frontend.md) — the React side.
6. [`tasks.md`](tasks.md) — what to do and the evidence that proves it.
7. [`adr/`](adr/) — architecture decisions with alternatives and consequences.

## Language

Documentation is in English. User-facing API messages, the five category names and the UI copy remain in Spanish because they are observable
behavior and seed data, not documentation (see TD-02).

## Scope

Unchanged from the reference: ten user stories (HU-01..HU-10), six business rules, five non-functional requirements and the same
out-of-scope list. Items tagged **[C-xx]** in the documents are clarifications of silence in the reference, not new features.
