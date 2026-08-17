# Working in this repo

This repo is not a project — it is the **conventions** other projects
load. Everything in it is paid for out of the context budget of every
session that loads it, so the bar for adding is high and the bar for
removing is low.

Orientation: `README.md` (what it is, how to install), `docs/pressure-test.md`
(the standard for content, and the ledger of what was rejected),
`plugins/house-style/skills/` (the content itself).

## The two structural rules

1. **Principle vs Practice.** Every entry states the durable *why*
   (Principle) separately from the current technique (Practice, dated).
   Replacing a tool means rewriting the Practice and leaving the
   Principle alone. If you cannot state the Principle, the entry is
   probably a preference, not a convention.

2. **The pressure test.** An assertion earns its place only if a
   competent Claude session would plausibly do otherwise by default:
   counterfactual, falsifiable, load-bearing, at the right altitude.
   Full statement in `docs/pressure-test.md`. When you reject something,
   add it to the ledger there — the rejection is as valuable as the
   entry, because it stops a future session from "helpfully" adding it
   back.

## Editing

- **Standing permission: open pull requests.** Not merges, not
  force-pushes, not direct commits to `main`. The procedure is in
  `skills/revise-conventions/SKILL.md` — follow it rather than
  improvising, including the PR body format.
- **Prefer editing an existing entry to adding one.** Length is the
  failure mode of this repo.
- **Repo-specific rules do not belong here.** They go in that repo's
  `CLAUDE.md`. This repo holds what generalizes across projects.
- **Do not assert a practice you have not verified.** Every claim here
  was checked against hubbub-club, gradebook, or goblin. If you are
  generalizing from one repo, say so in the entry.
- Keep `skills/conventions/SKILL.md` short — it is an index, and it
  loads whenever the skill triggers. Detail belongs in `references/`,
  which loads only when needed.

## Mechanics

- `SKILL.md` frontmatter: `name` and `description` are what matter;
  `description` decides when Claude auto-loads the skill, so write it as
  triggers, not as a summary. `when_to_use` appends to it. The combined
  text is truncated at 1,536 characters in the listing.
- Skills are namespaced by plugin: `/house-style:conventions`.
- Bump `version` in **both** `.claude-plugin/plugin.json` and the
  matching entry in `.claude-plugin/marketplace.json` on every
  substantive change — installed copies only update when the version
  string changes.
- Prose hard-wrapped at ~75 columns.
- After editing, validate the JSON manifests parse and that every
  reference file linked from a `SKILL.md` exists.
