# Data contracts, validation, and authorization

## One schema, three consumers

**Principle.** Every boundary a value crosses must be validated against
the *same* definition. A schema that is authoritative in one place and
transcribed by hand in another is not a contract, it is a coincidence
waiting to end.

**Practice.** Zod schemas in `packages/schema`. The schema value and its
inferred type share a name, so `import { CreatePost }` gives you both.
Three consumers:

1. **Backend** — `safeParse` every request at the boundary.
2. **Frontend** — the inferred types; no hand-written request
   interfaces.
3. **Firestore rules** — shape validators *generated* from the schemas,
   **for those collections the client writes directly** (below). Where
   writes go through a function, the callable boundary already enforces
   the schema and the rules say `allow write: if false` instead.

## The boundary wrapper

**Principle.** Cross-cutting boundary concerns — authentication, request
validation, ownership — must be structurally impossible to forget. If
they are a checklist, some endpoint will eventually skip one.

**Practice.** A single `typedCall(schema, handler)` wrapper is the only
way an endpoint is defined. Inside it, in this order:

1. **Auth first.** Reject unauthenticated callers before parsing.
   Ordering is deliberate: an anonymous caller must not learn the
   expected argument shape from a validation error.
2. **Then zod.** `safeParse`; on failure throw `invalid-argument` with
   the prettified error.
3. **Then the handler**, which receives already-typed data and a caller
   record.

On top of that, every callable also asserts the caller owns the subject
it acts as (`assertOwnsProfile`) or performs an explicit capability
check. "The client only sends its own id" is not a check.

Errors are typed platform errors with meaningful codes
(`unauthenticated`, `permission-denied`, `invalid-argument`,
`failed-precondition`), and specs assert the *code*, not the message.

## Choosing the write path (decide this per collection)

**Principle.** For each collection, the security boundary belongs in
exactly one place: the place where the invariant it protects is
*expressible* and *testable*. That is a per-collection decision, and it
is decided by the invariant, never by how many endpoints it saves.

**The argument that does not count.** "Fewer create/update functions to
write" is not a reason. Code volume is cheap now; the controls are what
is scarce. When both paths could work, the tie goes to the one with the
better knobs — which is the function.

**What each path actually buys.**

Direct client writes, enforced by rules, buy three things a callable
cannot give back:

- **Offline and optimistic behavior.** The Firestore SDK applies the
  write locally, echoes it to listeners immediately, and syncs when the
  network returns. A callable simply fails offline. This is a product
  property, not a code-volume one, which is why it survives the
  argument above.
- **No cold start** on the write path.
- **No per-write invocation cost.**

Function-mediated writes buy control that rules structurally cannot
express. Rules are a deliberately limited DSL:

- **A hard cap on document lookups**: 10 `get`/`exists` calls per
  single-document request or query, 20 across a multi-document read,
  transaction, or batched write. Exceeding it is a permission error, not
  a slow query.
- **Rules cannot write.** So a rule can never maintain an invariant that
  requires *changing* another document — no derived rows, no
  rematerialized reports, no audit entry, no counter.
- **No loops, no external calls, no secrets, no rate limiting.**
- **Untyped, and testable only through an emulator suite** — slower and
  coarser than the unit layer a callable gets.

A callable, by contrast, gets the same zod schema *plus* refinements and
cross-field checks, arbitrary capability logic, several documents
changed atomically, an audit entry in the same transaction, and a unit
spec with a scripted fake client.

**Practice — the decision rule.**

Route the write through a **function** if *any* of these hold:

- the write must maintain an invariant involving data the writer does
  not own or cannot see (derived records, rosters, counters, membership
  closure);
- its legality depends on more lookups than the rules budget allows, or
  on data outside the database;
- several documents must change atomically for the result to be
  correct;
- it must be audited, rate-limited, or produce a decided-by artifact;
- the validation is expressible in zod but not in rules — refinements,
  cross-field constraints, dynamic-key record values.

Allow **direct client writes** only when *all* of these hold:

- the document is owned by the writer, and its legality is decidable
  from the document itself plus the auth token plus a lookup or two;
- nothing derived depends on it;
- it is high-frequency, low-stakes, or genuinely wants offline and
  optimistic behavior — drafts, local-first editing, per-user
  preferences, presence.

Trusted-editor content editing is the honest middle case: goblin lets
signed-in editors write module content directly under a `canEdit` flag,
because the invariant really is "this person may edit this document" and
the editing UX wants immediacy. Grading cells and materialized reports
in gradebook are the opposite case and are function-only. Overstory
routes essentially everything through callables because its visibility
model is a graph closure — no rule could compute it.

Whichever path a collection takes, **write the choice and its reason
down** with the schema. The next person's instinct will be to "simplify"
it in one direction or the other.

## Security rules

**Principle.** The database's own rules are the last line that a
compromised or hand-rolled client cannot get past. Their first job is to
say what the client may not touch *at all*; their second, only for the
collections clients do write, is to enforce *shape* as well as access —
otherwise an authenticated user can write arbitrary fields and types
into your collection and every reader downstream inherits the problem.
And where rules do validate shape, they must never be a hand-maintained
restatement of a schema, because the two diverge in the first week.

**Practice — generate the shape validators, where they are needed.**

Generate `isValid<X>` for a collection **only if the client writes it
directly.** For a function-only collection the rule is
`allow write: if false`, which is the strongest statement available and
leaves nothing for a validator to do; generating one there produces dead
rules that imply the client has a write path it does not have. A project
in which every write goes through a callable needs no generator at all —
the callable boundary is where the schema is enforced, and adding the
codegen would be ceremony.

Where it does apply, a codegen script walks the zod schemas and emits
per-collection `isValid<X>(data)` functions into a marked
`AUTOGENERATED` block in the `.rules` file:

```
data.keys().hasAll([...required])
  && data.keys().hasOnly([...all])
  && <per-field type check>          // string / number / bool / list / map / timestamp
  && <enum membership>
  && (!('opt' in data) || data.opt is string)   // optional-field guards
```

Discriminated unions emit one OR-branch per variant. Authorization stays
**hand-written around** the generated block — the generator owns shape
only, and never touches who may write.

The reference implementation is
`gradebook/packages/migrations/generate-rules.ts`, driven by a
collection registry in the schema package. Copy it rather than writing a
new one.

Two things make this real rather than decorative:

- `npm run rules:gen -- --check` fails if the committed rules are stale,
  and it runs inside `npm run gate`. Codegen without a drift check is
  just a suggestion.
- The generator warns loudly on a zod type it has no mapping for instead
  of silently emitting `true`.

Known non-goals of the generator, documented so nobody re-discovers them
as bugs: dynamic-key record *values* (the map type is checked, not the
values), cross-document checks, and refinements. A collection needing any
of those is telling you it wants the function path.

**Whatever the write path, the read path is a separate decision.**
Function-only writes do not imply function-only reads — direct
rules-guarded reads with realtime listeners are usually still the right
answer, and are why the data is in Firestore at all.

## Rules invariants that hold regardless of write path

- Data that is nobody's business is not client-readable at all — PII in
  a `profiles` collection is read through a backend endpoint, not by
  the client under a rule.
- **Permission-granting fields are never client-writable** — an `admin`
  flag, a `canEdit` flag. They are set by admin SDK scripts or by hand
  in the console. Nothing in the client can escalate itself. This holds
  even in a collection the client otherwise writes freely.
- Storage rules stay DENY-ALL when bytes move through backend
  endpoints. Storage paths are opaque ids, never derived from uid or
  profile. Uploads are re-encoded (which strips EXIF) and the original
  discarded.
- Collection group queries need their own rules — a nested `match` only
  covers document paths, so a `match /{path=**}/sections/{id}` rule is
  required for the group query to be allowed at all.

## Transactions

**Principle.** A transaction holds locks. Anything inside it that can
block for an unbounded time turns a correctness mechanism into an
outage.

**Practice.** Database work stays inside the transaction wrapper; **no
external HTTP calls inside a transaction**. Any privileged mutation of
another person's standing — ban, time-out, policy change — writes an
audit entry *in the same transaction*. Workflow actions that already
produce a decided-by artifact (a proposal record, a review record) are
exempt because the artifact is the audit trail.

## Rules get their own test suite

**Principle.** Security rules are the highest-consequence code in the
repo and the least exercised by ordinary tests, because the app is
written to stay inside them. They need adversarial tests that
deliberately step outside.

**Practice.** A `rules-tests` package runs against the Firestore
emulator under a *dedicated project id* (so it never touches dev data on
a shared emulator), with a persona per role — unauthenticated, each user
type, each relationship to a resource — and asserts both `assertSucceeds`
and `assertFails` for every collection. It runs in CI on every PR.
