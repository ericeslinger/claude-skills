# Committing, pushing, deploying

## The gate ladder

**Principle.** Each irreversible step gets a check proportional to how
hard it is to undo.

**Practice.**

| Before you | Run |
| --- | --- |
| commit | `npm run gate` |
| push / open a PR | `npm run e2e` |
| deploy functions | `scripts/deploy-functions.sh --dry-run` |

Do not commit on a red gate and do not describe an unrun gate as
passing. If a suite is failing for a reason unrelated to your change,
say which suite and why, and let Eric decide.

## Git

**Principle.** Publishing is Eric's call, not the assistant's. Branch
and push are cheap and reversible; a PR is a social act.

**Practice.**

- Work on a feature branch, push to `origin`.
- **Do not open pull requests unless explicitly asked.** The default
  workflow is: branch → push to origin → Eric opens the PR.
- Where a repo is a fork with an `upstream`, **never** push to
  `upstream`, any ref, any form. Gradebook states this explicitly; treat
  it as the pattern wherever an upstream exists.
- A repo's own `CLAUDE.md` may harden these rules. It never loosens
  them.
- Commit messages: what changed and why, referencing backlog ids where
  they exist. No model identifiers or tooling banners in commits, PR
  bodies, or code comments.

## Firebase CLI

**Principle.** The toolchain version is part of the build. A globally
installed CLI is an unpinned dependency that differs per machine and
per CI runner.

**Practice.** **Always `npx firebase`**, resolving the `firebase-tools`
pinned in the root devDependencies. Never a standalone or globally
installed `firebase` binary. This is stated in more than one repo's
CLAUDE.md because it has bitten more than once.

## Deploying functions

**Principle.** The thing that breaks a functions deploy is almost never
the code — it is the *packaging*: a workspace dependency that resolved
locally and does not exist in the cloud build. Prove the bundle loads
under production conditions before shipping it.

**Practice.** `scripts/deploy-functions.sh --dry-run` reproduces what
the cloud builder does with the uploaded source: stage the built output
plus its `package.json` into a temp dir, `npm install --omit=dev`, then
`import()` the bundle and assert the expected function exports are
present. That proves the workspace packages really were inlined by the
bundler and that the export set has not drifted.

The dry run is part of `npm run gate`, so packaging breakage is caught
at commit time, not at deploy time.

## Test-only code must be structurally absent from production

**Principle.** A development affordance that ships is a vulnerability.
"It checks a flag at runtime" is weaker than "it does not exist in the
artifact," and only the second is provable.

**Practice.** Emulator-only triggers (a forced tick, a seed hook) are
`undefined` outside the emulator environment, and the deploy dry-run
asserts they are not in the production bundle. Seed scripts carry hard
production guards. Test helpers live in a directory production code
never imports, so the bundler never reaches them.

## Migrations and operational scripts

**Principle.** Anything that touches production data is a reviewable
artifact, runs read-only first, and is kept after it has run.

**Practice.** A `migrations` package with numbered, individually
runnable migrations and a runner that **defaults to a dry run** and
requires `--apply` to write. Operational one-offs (provision users,
export auth, sanitize records, purge strays) live there too as named
scripts rather than as pasted snippets — they get run again.
