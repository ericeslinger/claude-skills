---
name: conventions
description: Eric's house style for Angular + Firebase TypeScript monorepos — repo layout, the shared zod contract, Firestore rules generated from schemas, the unit-test/e2e gate ladder, and the documentation practice. Load before writing code, tests, schemas, security rules, or project docs in any of Eric's repos, and before scaffolding a new project.
when_to_use: Starting work in a repo owned by ericeslinger; adding a callable/endpoint, component, schema, or Firestore rule; deciding Firestore vs Postgres; writing or skipping tests; about to commit or push; writing a DESIGN/backlog/decision doc; asked "how do I like to work" or "look at repo X for how I do this".
---

# House style

The durable version of "look at repo X for an example of how I like to
work." Reference implementations: `ericeslinger/hubbub-club` (Overstory
Social), `ericeslinger/gradebook` (stubgrub), `ericeslinger/goblin`
(Armature).

Read this page, then load only the reference file the task needs. Each
reference separates **Principle** (durable — survives a stack change)
from **Practice** (how it is done as of 2026-08 — replaceable). When a
better technique arrives, change the Practice and keep the Principle.

| Load when | File |
| --- | --- |
| Deciding stack, storage, or "can Firestore model this?" | [references/stack.md](references/stack.md) |
| Adding packages, handlers, or moving code between layers | [references/repo-layout.md](references/repo-layout.md) |
| Writing schemas, validation, security rules, authz | [references/data-contracts.md](references/data-contracts.md) |
| Writing tests, or deciding whether something needs one | [references/testing.md](references/testing.md) |
| Building UI | [references/frontend.md](references/frontend.md) |
| Writing DESIGN/backlog/decision docs, or reading in | [references/documentation.md](references/documentation.md) |
| Committing, pushing, deploying | [references/shipping.md](references/shipping.md) |

The project's own `CLAUDE.md` outranks this file. This is the default;
the repo is the specific.

## The short version

1. **Angular + Firebase, TypeScript everywhere.** Angular (standalone,
   signals, no zone.js), Firebase Hosting / Auth / Cloud Functions.
   Firestore is the default store; Postgres only when the data model
   is genuinely relational. Not React, not Next, not Express-on-a-VM.
2. **npm-workspaces monorepo**, frontend and backend as separate
   packages, plus a third `schema` package that both depend on.
3. **The schema package is the contract.** Zod schemas are the single
   source of truth for every boundary: the backend validates requests
   against them and the frontend uses the inferred types. **Decide the
   write path per collection** — a function-mediated write is enforced
   at the callable boundary and the rules say `allow write: if false`;
   a collection the client writes directly gets shape validators
   *generated* from the same schemas into `.rules`, so malformed data
   cannot reach the database even from a rogue client. Code volume is
   not a reason to prefer either; the controls are.
4. **Two test layers, two gates.** Colocated unit specs for every
   non-trivial file — `npm run gate` must be green **to commit**.
   Playwright journeys over the real emulated stack — `npm run e2e`
   must be green **to push or open a PR**.
5. **Documents are the design of record.** Decisions get written down
   with a date and are not re-litigated; the backlog carries stable
   ids. Read the docs before proposing architecture.

## Working rules that apply everywhere

- **Don't invent a third way.** If a repo already has a pattern for the
  thing you are adding (a handler, a spec, a journey, a script), copy
  the nearest existing example rather than introducing a new shape.
- **A test that does not run is not a test.** Wire every new suite into
  `npm run gate` (or `e2e`) in the same change that creates it.
- **Ambient inputs enter as parameters.** Clock, randomness, and
  network reach domain logic through arguments at named seams, never a
  naked `new Date()` or `Math.random()` mid-handler. This is what makes
  the unit layer possible without a database.
- **Say what is actually true.** If a check was skipped, the suite is
  red, or a claim is unverified, say so plainly. Never report a gate as
  passing that you did not run.
- **Prose in repo docs is hard-wrapped at ~75 columns.** Match the file
  you are editing. Cosmetic, but the diffs stay readable.

## When Eric's opinions change

This repo is editable. If Eric states a new general preference, or a
practice in these files is contradicted by how the code actually works,
update it — see the `revise-conventions` skill (`/house-style:revise-conventions`).
Do not silently work around a stale convention; fix it here so the next
session inherits the correction.
