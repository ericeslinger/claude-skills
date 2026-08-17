# Repository layout

## Monorepo, with the contract in the middle

**Principle.** Frontend and backend are separate deployables with
separate build and test cycles, but they share exactly one thing — the
contract between them — and that thing must be impossible to fork. A
type duplicated on both sides drifts silently; a type *imported* by both
sides cannot.

**Practice.** npm workspaces, one `npm install` for the repo:

```
packages/
  schema/      the zod contract — depends on nothing in the repo
  frontend/    Angular app (angular.json lives HERE, not at the root)
  backend/     Cloud Functions
  e2e/         Playwright journeys
  <extra>/     migrations, rules-tests, generated SDKs — as needed
```

Both `frontend` and `backend` depend on `schema`; nothing depends on
`frontend` or `e2e`. The root `package.json` holds only workspace-wide
scripts (`build`, `test`, `gate`, `dev`, `e2e`) and the pinned
`firebase-tools`.

Extra packages are cheap and preferable to a junk drawer. Real examples:
`rules-tests` (emulator allow/deny suites), `migrations` (the migration
runner plus operational scripts), `dataconnect-generated` (a committed
generated SDK).

## One handler per file; `index.ts` is wiring

**Principle.** The unit of a backend is one endpoint, and it should be
findable by name in the file tree. An `index.ts` that contains logic
becomes the file every change touches and every merge conflicts in.

**Practice.**

- `backend/src/index.ts` does `initializeApp()` and re-exports. Nothing
  else. It is explicitly exempt from the unit-test policy because it
  contains nothing to test.
- Handlers live one per file, grouped in concept directories
  (`posts/`, `clubs/`, `moderation/`, …), each with a colocated
  `.spec.ts`.
- Shared machinery sits at `backend/src/` top level with names that say
  what they are: `guards.ts` (the auth + validation wrapper), `db.ts`,
  `authz.ts`. Each gets its own spec.
- Test-only helpers live in `backend/src/testing/` and are never
  imported from production code, so the bundler never sees them.

## Frozen export names

**Principle.** The callable surface is an API. Renaming an export is a
breaking change to every deployed client, and TypeScript will not catch
it because the frontend calls by string.

**Practice.** Callable export names are frozen. A table-driven
`boundary.spec.ts` enumerates the entire callable surface and fails if
the export set drifts — adding a callable means extending that table in
the same commit. When you add an endpoint, check that spec first: it is
the registry.

## Scripts are shell, checked in, and self-documenting

**Principle.** Any multi-step operational procedure that a human might
run twice belongs in a script in the repo, with the reasoning in a
header comment. Instructions living in a README get skipped; procedures
living in someone's shell history do not survive.

**Practice.** `scripts/` holds `dev-local.sh`, `e2e.sh`,
`deploy-functions.sh`, plus whatever the project needs. Each starts with
a comment block explaining what it orchestrates and *why it is a script
rather than config* — usually because readiness, warmup, or isolation
logic cannot be expressed declaratively. `set -euo pipefail`, and
unknown arguments are a hard error (an unrecognized `--dry-run` must
never fall through to a real deploy).

## Small stuff, stated once

- Import platform statics via subpaths (`firebase-admin/firestore`),
  never `admin.firestore.FieldPath` — loader interop drops inherited
  statics and it fails at runtime, not at build.
- Multi-row SQL `VALUES` expansions chunk at 500 rows. An array passed
  as one parameter is one parameter and needs no chunking.
- Prettier where it is configured; no ESLint in these repos — do not add
  a linter without asking.
