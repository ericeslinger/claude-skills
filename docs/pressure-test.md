# Pressure test

_Seeded 2026-08-17 when the repo was created, from a devil's-advocate
pass over every candidate assertion. Its purpose is to keep rejected
platitudes from being re-added by a future session that thinks they were
simply overlooked._

## The test

An assertion earns a place in this repo only if a competent Claude
session would plausibly do **otherwise** by default. Four questions,
all four must pass:

1. **Counterfactual** — delete the line; does any session behave
   differently?
2. **Falsifiable** — can a reviewer point at a file and say "this
   violates it"?
3. **Load-bearing** — does something break, or a known bug recur, if it
   is ignored?
4. **Right altitude** — durable *why* in Principle, current technique in
   Practice.

Anything that fails is not neutral. It is a tax on every session that
loads this repo.

## Rejected — already the default, or unfalsifiable

These were considered and cut. Do not re-add them.

| Candidate | Why it was cut |
| --- | --- |
| "Use TypeScript with strict mode" | Default in every project template Claude would produce. |
| "Write clear commit messages" | Default behavior; the *specific* rules (no tool banners, reference backlog ids) survived and are in `shipping.md`. |
| "Don't commit secrets" | Default, and enforced by tooling, not by a convention doc. |
| "Handle errors properly" | Unfalsifiable. The narrow version — typed platform error codes, specs assert the code not the message — survived. |
| "Prefer async/await over callbacks" | Default. |
| "Follow DRY / SOLID" / "keep code clean" | Unfalsifiable; nobody has ever been shown a file and agreed it violates SOLID. |
| "Write tests for edge cases" | Vacuous. The falsifiable version — a colocated spec per non-trivial file, no transitive credit — survived. |
| "Use meaningful variable names" | Default. |
| "Add JSDoc to public functions" | Not observably true in the reference repos; would have been invented. |
| "Set up ESLint with recommended rules" | **None of the three repos has ESLint.** Asserting it would have been fabrication. `repo-layout.md` says so explicitly, because the absence is the surprising part. |
| "Use environment variables for configuration" | Default, and the actual practice (committed emulator placeholders, real keys pasted at deploy) is repo-specific. |
| "Keep functions small" | Unfalsifiable. The falsifiable neighbor — one handler per file, `index.ts` is wiring only — survived. |
| "Use semantic HTML" | Subsumed by the a11y rules, which are falsifiable and enforced by axe in CI. |
| "Document your architecture decisions" | Too generic to change behavior. Replaced by the concrete practice: stable backlog ids, `DECIDED <date>; do not relitigate` markers, status describing what the code does today. |

## Survived, but narrowly — and why

- **"Use zod."** On its own this is close to a default for TypeScript
  projects and would have been near-worthless. It survives only in its
  non-default form: zod as the *single* source of truth shared by three
  consumers, with the Firestore rules **generated** from it and a
  `--check` drift gate in CI. The headline in `data-contracts.md` is the
  codegen, not the library.

- **"Validate input at the boundary."** Generic on its own. What earns
  the space is the *ordering* — auth before parse, so anonymous callers
  learn nothing about the argument shape — and the structural claim that
  a single wrapper is the only way an endpoint gets defined. Both are
  non-obvious and both are checkable.

- **"Write end-to-end tests."** Claude will write Playwright tests if
  asked. What is counter-default: one spec per *journey* rather than per
  page, the shell script owning the whole stack instead of Playwright's
  `webServer`, port and database isolation from the dev stack, and the
  axe journey failing the build. Those specifics are the content.

- **Markdown hard-wrapped at ~75 columns.** Weakest entry in the repo:
  purely cosmetic, and one reference repo's README violates it in
  tables. It survives on counterfactual grounds only — Claude writes
  unwrapped paragraphs by default, and the three repos are visibly
  wrapped — and it is labeled cosmetic where it appears. If it ever
  needs to be cut to make room, cut it first.

- **"Ambient inputs enter as parameters."** Standard testability advice,
  which is a strike against it. It survives because Claude demonstrably
  does sprinkle `new Date()` through handler bodies, and because in
  these projects it is load-bearing for a specific reason: the domain
  clock is a database value, so a naked `new Date()` is not merely hard
  to test, it is *wrong*.

## Entries whose value is scar tissue

The strongest content in the repo. Each records a real failure:

- `typecheck:spec` as a separate gate step — vitest strips types, so
  type errors inside specs are invisible to the suite.
- `npx firebase`, never a global binary — stated in two repos' CLAUDE.md
  independently.
- The deploy dry-run that stages a production-only install and
  `import()`s the bundle — catches the workspace-dependency-not-inlined
  failure that only appears in the cloud build.
- `@for` must track a stable identity — a 34-edit sweep across 15
  templates after NG0956.
- Identity is the auth uid, never email — gradebook migrated a
  production database to undo the alternative.
- Import platform statics via subpaths — loader interop drops inherited
  statics, and it fails at runtime rather than at build.
- Emulator-only triggers must be structurally absent from the production
  bundle, with the dry-run asserting it.

When adding to this repo, aim for this category.
