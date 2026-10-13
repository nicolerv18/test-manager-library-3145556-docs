# test-manager-library-3145556-docs

> **Technical test · ADSO group 3145556**
> Due date: **Tuesday, October 13, 2026, at 5:00 p.m.** (17:00, Colombia time).

This repository contains the **spec** of *Manager Library*. It is the only one with content: the other five start empty.

## Instructions

Each apprentice must **fork** the six repositories of the project and **solve the project
with the proposed spec**.

1. Fork, into your GitHub account, each repository in the table at the end.
2. Read the spec in [`test-manager-library-3145556-docs`](https://github.com/code-sena/test-manager-library-3145556-docs).
   It is delivered in two versions: `spec-python/` and `spec-.net/`.
3. Develop in the forks.

## The challenge is developed with React and PHP (Laravel)

The spec is written for Python and for .NET, but the challenge is **not** done in those languages:

| Layer | Challenge technology |
|---|---|
| Frontend | React |
| Backend | PHP with Laravel |

What the spec defines about the business —stories, acceptance criteria, rules, API contract,
data model— is respected. What it defines about the technology is translated to React and Laravel.

## The test is not about writing the code

The main purpose is to see the **performance with SDD** (*Spec-Driven Development*): how a specification is read, interpreted and
applied to carry it over to a different stack. The code is the means, not the end.

## Delivery

What counts is what is **uploaded to the forks before 5:00 p.m.** (17:00). Anything that arrives later is not reviewed.
There is no extension: push partial progress with small commits during the test.

## What this repository contains

| Folder | Content |
|---|---|
| [`spec-python/`](spec-python/) | The Manager Library spec written for a Python backend |
| [`spec-.net/`](spec-.net/) | The same system specified for a .NET backend |

Both versions describe the **same product** (the same stories, business rules and
endpoints); the technology decisions change. Neither of them is the challenge stack.

## How to read the spec

In either of the two folders, in this order:

1. `constitution.md` — non-negotiable principles.
2. `spec.md` — what the system must do: actors, stories, acceptance criteria, business rules.
3. `plan.md`, `architecture.md` and `architecture-overview.md` — how it is built.
4. `data-model.md` and `api-contract.md` — the data model and the API contract.
5. `tasks.md` — what has to be done and with what evidence it is considered done.
6. `adr/` — the architecture decisions, with their alternatives and consequences.

## The six repositories

| Repository | What goes there |
|---|---|
| [`test-manager-library-3145556-docs`](https://github.com/code-sena/test-manager-library-3145556-docs) | The spec: `spec-python/`, `spec-.net/` and `spec-laravel-react/` |
| [`test-manager-library-3145556-api`](https://github.com/code-sena/test-manager-library-3145556-api) | Backend in PHP (Laravel) |
| [`test-manager-library-3145556-app`](https://github.com/code-sena/test-manager-library-3145556-app) | Frontend in React |
| [`test-manager-library-3145556-page`](https://github.com/code-sena/test-manager-library-3145556-page) | Static public presentation site |
| [`test-manager-library-3145556-infra`](https://github.com/code-sena/test-manager-library-3145556-infra) | Containers, network, volumes and empty database engine |
| [`test-manager-library-3145556-tool`](https://github.com/code-sena/test-manager-library-3145556-tool) | Utilities: demo data seeder |
