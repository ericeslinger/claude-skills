# claude-skills

Eric Eslinger's house style, packaged as a Claude Code plugin so every
session — laptop, cloud, or scheduled — starts already knowing how these
projects are built.

It replaces the sentence "look at repo X for an example of how I like to
work."

## What's in it

One plugin, `house-style`, with three skills:

| Skill | What it does |
| --- | --- |
| `/house-style:conventions` | The conventions themselves. Claude loads this automatically when working in one of these repos; a short index page plus reference files it reads on demand. |
| `/house-style:new-project` | Scaffolds a new Angular + Firebase monorepo to the standard, in the order that keeps each step runnable. |
| `/house-style:revise-conventions` | Opens a PR against this repo when Eric's opinions change. |

The conventions cover: the Angular + Firebase stack and the Firestore vs
Postgres decision rule; monorepo layout with a shared zod contract
package; choosing per collection between function-mediated and direct
client writes, with rules shape-validators generated from the schemas
where clients do write; the unit-spec/e2e gate ladder (`gate` to commit,
`e2e` to push); frontend and accessibility rules; the documentation
practice; and shipping.

Reference implementations: [hubbub-club](https://github.com/ericeslinger/hubbub-club)
(Overstory Social), [gradebook](https://github.com/ericeslinger/gradebook)
(stubgrub), [goblin](https://github.com/ericeslinger/goblin) (Armature).

## Install

### For a project (works in cloud sessions too — recommended)

Commit this to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "ericeslinger": {
      "source": { "source": "github", "repo": "ericeslinger/claude-skills" }
    }
  },
  "enabledPlugins": { "house-style@ericeslinger": true }
}
```

Local sessions pick it up once the folder is trusted; cloud sessions
install repo-declared plugins at session start. This is the only route
that covers both — a skill living in `~/.claude/skills` on the laptop
does **not** reach cloud sessions or routines.

### For every project on one machine

```
/plugin marketplace add ericeslinger/claude-skills
/plugin install house-style@ericeslinger
```

Then `/plugin marketplace update` to pull changes.

## Editing it

**Claude has standing permission to open pull requests against this
repo** when Eric states a new general preference, reverses an old one,
or when the docs are contradicted by how the code actually works. See
[`revise-conventions`](plugins/house-style/skills/revise-conventions/SKILL.md).
Claude does not merge them.

Two rules keep this repo worth loading:

1. **Principle vs Practice.** Every entry separates the durable *why*
   from the current technique. When a better tool arrives, the Practice
   is replaced and the Principle survives.

2. **The pressure test.** An assertion earns its place only if a
   competent Claude session would plausibly do otherwise by default —
   it must be counterfactual, falsifiable, and load-bearing. Everything
   else is a tax on every session that loads this repo.
   [`docs/pressure-test.md`](docs/pressure-test.md) records what was
   cut and why, so the platitudes stay cut.

## Layout

```
.claude-plugin/marketplace.json     the marketplace catalog
plugins/house-style/
  .claude-plugin/plugin.json
  skills/conventions/SKILL.md       index — always short
          references/*.md           loaded on demand
  skills/new-project/SKILL.md
  skills/revise-conventions/SKILL.md
docs/pressure-test.md               the standard, and the rejection ledger
CLAUDE.md                           for Claude working IN this repo
```
