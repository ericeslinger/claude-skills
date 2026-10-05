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

**Principle.** Shipping is gated by checks, not by who clicks. A code
change that passes every gate and an independent review is cheap to
revert, so the assistant carries it all the way to `main`. A change to
the shape of stored data is not cheap to revert, so that one waits for
Eric.

**Practice** (2026-10-05, reverses "do not open pull requests unless
explicitly asked; Eric opens the PR"):

- Work on a feature branch, push to `origin`, and **open the PR
  yourself**. The body states which gates ran, their results, and the
  head sha they ran on.
- Spawn the repo's PR review agent (a subagent in `.claude/agents/`, as
  in goblin and overstory) on the PR. Answer every finding: fix and
  push, or reply why not. After any push, re-run both gates and ask the
  same agent to re-review.
- **Merge** (merge commit) when all hold: the latest review approves;
  `npm run gate` and `npm run e2e` are green on the exact head; `main`
  has not moved since; and the change contains **no schema change or
  data migration**. A schema change is any change to the shape of
  persisted data: the zod schemas for stored documents, the database
  rules, SQL migrations. Those PRs stay open for Eric, with a comment
  saying why.
- **After merging, delete the merged branch**, on `origin` and locally.
  Only branches whose PR has merged; never a branch with unmerged
  commits, and never someone else's.
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
