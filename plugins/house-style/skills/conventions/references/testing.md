# Testing and the gate ladder

## Two layers, two gates

**Principle.** Tests exist to make change safe, and they only do that if
failing them blocks something. A suite nobody has to pass is
documentation with a runtime cost. So each layer is tied to a specific
moment it gates.

**Practice.**

| Layer | Command | Gates |
| --- | --- | --- |
| Unit specs — no database, no emulator | `npm run gate` | **commit** |
| Playwright journeys — the real emulated stack | `npm run e2e` | **push / PR** |

`npm run gate` is the whole pre-commit bar in one command: builds, every
unit suite, spec typechecks, codegen drift checks, and the deploy
dry-run. If it is green you may commit; if you did not run it, say so
rather than implying you did.

CI re-runs both on every pull request, including the full journey suite
against a booted emulator stack. The local gates exist so CI is a
confirmation, not a discovery.

## Unit layer: colocated, per file, no free rides

**Principle.** Coverage measured over a whole codebase hides exactly the
files that need tests. The unit that matters is the *file*: if a file is
worth writing, its behavior is worth pinning next to it.

**Practice — every non-trivial file has a colocated `<name>.spec.ts`.**

- Frontend: every component, service, and pipe.
- Backend: every handler file **and** every shared module — guards, db,
  authz, merge helpers, concept-local helpers.
- **No transitive coverage credit.** Shared code being exercised through
  another file's spec does not count. It gets its own local spec.

Exempt, and only these: wiring-only `index.ts`, config files
(`app.config.ts`, `app.routes.ts`, firebase config/providers), and the
spec-support helpers themselves.

This is the rule most likely to be quietly skipped under time pressure,
and it is the one Eric actually cares about. When you add a shared
helper "just for this handler," it needs its own spec in the same
commit.

**No database in the unit layer.** Handlers are invoked directly through
a small test harness that shapes the request like the platform would,
with a scripted fake client that routes queries by substring to canned
results and records every call. That is what makes the unit suite fast
enough to be a commit gate, and it is why ambient inputs must arrive as
parameters.

**Assert error codes, not messages.** A helper that awaits a rejection
and checks the platform error code keeps specs from breaking on copy
edits.

## Spec typechecks are a separate step

**Principle.** A type error that the test runner cannot see is a type
error that ships.

**Practice.** Vitest strips types rather than checking them, so type
errors *inside spec files* are invisible to the suite. Every package
therefore has `typecheck:spec` (`tsc -p tsconfig.spec.json --noEmit`)
and `gate` runs it. This is not redundant with the build — the build
does not include specs.

## E2E layer: journeys, not page tests

**Principle.** An end-to-end test earns its cost only by exercising a
path a real person takes end to end. Assertions on individual pages
belong in the unit layer, where they are cheaper and more precise.

**Practice.**

- One spec per **journey**, named for what the person does:
  `publish-cycle`, `club-moderation`, `first-profile`, `letters`,
  `vouch-review`. Not `login.spec.ts`, not `homepage.spec.ts`.
- **Every new user-facing flow gets a journey.** This is the completion
  criterion for a feature, not a follow-up task.
- Personas and fixtures are shared modules (`personas.ts`,
  `fixtures.ts`) with a `signInAs(page, PERSONA)` helper. Journeys read
  as prose.
- Query by role and accessible name (`getByRole('combobox', { name:
  'Add to audience' })`), which makes the journeys double as an
  accessibility assertion. Do not add test ids to route around a
  control that has no accessible name — fix the control.
- **An automated a11y journey is part of the suite**: axe over every
  signed-in surface, WCAG 2.1 A/AA, failing the build on serious and
  critical violations. Exclusions (for example `color-contrast` before
  the palette is settled) are explicit and commented with what unblocks
  them.
- CI keeps traces and video on failure.

## The e2e script owns the whole stack

**Principle.** An end-to-end suite is only trustworthy if it starts from
a known state, and only usable if it does not collide with the dev
environment you were just working in. Both properties need real
orchestration — readiness probes, warmup, seeding — which declarative
test-runner config cannot express.

**Practice.** `npm run e2e` runs `scripts/e2e.sh`, which owns everything:
start the database container and wait for readiness → boot the emulators
→ build what they load → seed → serve the app → run Playwright → tear
down process groups. Do not move this into Playwright's `webServer`.

It is **isolated from the dev stack**, deliberately: its own database,
its own emulator ports, its own frontend build and port, all in a
separate `firebase.e2e.json`. `npm run dev` and `npm run e2e` run side
by side on one machine without touching each other. The script
preflights its own ports and fails with a readable message rather than
racing.

Seeding truncates every run — the suite never depends on state left by a
previous run. Seed scripts carry hard production guards.

Playwright arguments pass through: `npm run e2e -- --grep publish`.

## What does not need a test

Boilerplate: re-export barrels, config objects, generated code, pure
wiring. If you find yourself writing a spec that asserts a constant
equals itself, the file was exempt — but say so rather than deleting the
requirement.
