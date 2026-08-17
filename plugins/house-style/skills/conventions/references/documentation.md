# Documentation practice

Eric's repos carry unusually heavy prose, on purpose. It is not
ceremony: it is how decisions stop being re-argued. Read before writing.

## The document set

**Principle.** Separate *what we are trying to build and why* from *how
it is built* from *what is not built yet*. Collapsing them produces a
document that is simultaneously stale, aspirational, and unusable as a
work queue.

**Practice.** Up to four kinds of top-level document, by role:

| Document | Role |
| --- | --- |
| `README.md` | Orientation: layout, prerequisites, commands. Skimmable. |
| `CLAUDE.md` | Working conventions and invariants for this repo. |
| `DESIGN.md` | The architecture — the design of record. |
| `PRINCIPLES.md` | The social/intellectual underpinnings — the *why*. Explicitly co-edited and explicitly not frozen. |
| a backlog (`launch_blockers.md`, `TECH_DEBT.md`) | What is designed but unshipped, with stable ids. |

Not every project needs all of them. A content pipeline needs a README
and a CLAUDE.md; a product with a social thesis needs PRINCIPLES.

**Orientation lives in CLAUDE.md.** Its first section names every other
document and says what each is for and which to read first. That
paragraph is what makes a fresh session productive in one read.

## The backlog carries stable ids

**Principle.** A work item needs a name that survives reordering, so
that commits, conversations, and design docs can refer to it without
ambiguity — and so that "we decided this already" is checkable.

**Practice.**

- Every entry gets a stable id (`LB-A1`, `LB-B4`). **Ids are never
  renumbered.** Retire an item by moving it to a shipped section with a
  `SHIPPED <date>` note, never by deleting the line.
- Entries carry **status describing what the code actually does today**,
  not what the plan says: `NOT BUILT` / `PARTIAL` (with the gap named) /
  `DECIDED / UNBUILT` / `UNDECIDED`.
- `DECIDED / UNBUILT` means the design conversation is finished and the
  item is safe to pick up without asking. `UNDECIDED` means stop and
  ask.
- Every entry **refs** the design section it comes from. The backlog is
  a priority index; the design doc is the authority. If they disagree,
  the design doc wins and the backlog gets fixed.
- Tiers describe the moment an item gates (first non-Eric user / first
  stranger / open registration), not vague priority.

## Decisions are dated and not re-litigated

**Principle.** The expensive failure mode of an AI collaborator is
re-opening settled questions with a plausible-sounding alternative every
time context is lost. The counter is a written decision with a date.

**Practice.** Settled decisions are marked in place: `DECIDED
2026-08-14, §19.8; do not relitigate`. When you encounter one:

- **Do not propose the alternative again**, even a good one, unless new
  information genuinely invalidates the premise — and if you think it
  does, say which premise and why, once.
- Retiring or reversing a decision is an explicit edit with a new date
  and a note, not a silent change.
- Closed items keep their reasoning. Struck-through entries with a
  `FIXED`/`CLOSED <date>` note and the evidence are more useful than a
  deleted line, because the next person asks the same question.

## Writing style for these docs

- Hard-wrapped at ~75 columns.
- Say what is true today and mark what is aspirational. `PARTIAL` with
  the gap named beats a checkbox.
- Cross-reference by section number and keep them accurate.
- Note when a doc was compiled and from what, so a reader knows its
  vintage.
- When a document is co-edited, say so in its header — including the
  invitation to disagree in place.

## Before proposing architecture

Read the design of record and the backlog first. Most "should we do X?"
questions in these repos already have an answer with a date on it, and
proposing X again costs Eric the time to re-explain. If you cannot find
an answer, say what you looked at before asking.
