# Stack and storage

## The default stack

**Principle.** One language across the whole system, one vendor for the
serving infrastructure, and no server to operate. The cost of context
switching between runtimes and hosting models is higher than whatever a
best-of-breed component buys.

**Practice (2026-08).**

| Layer | Choice |
| --- | --- |
| Frontend | Angular (v22+), standalone components, signals, no zone.js |
| Hosting | Firebase Hosting |
| Auth | Firebase Auth — the uid is the identity everywhere |
| Backend | Cloud Functions for Firebase, TypeScript, ESM, Node 22 |
| Store | Firestore by default; Postgres when relational (below) |
| Contract | zod, in a shared workspace package |
| Unit tests | vitest (jest only where it is already there — legacy) |
| E2E | Playwright against the real emulator stack |
| Package manager | npm workspaces |

Do not propose React, Next.js, Svelte, Vue, Supabase, Prisma, tRPC,
or a container-hosted Express service in a new project here. If one of
them is genuinely the right answer for a specific problem, make the
argument explicitly and get a decision — do not slide it in.

## Identity

**Principle.** One identifier for a person, chosen by the auth system,
used unmodified everywhere. Identity derived from mutable
user-controlled data (email, username) leaks into every key and every
rule, and every change of it becomes a migration.

**Practice.** `request.auth.uid` is the identity in rules, documents,
SQL, and logs. Never key on email, never normalize with `.lower()`.
Personally identifying fields live in one profile record, read through a
backend endpoint, not client-readable in bulk.

Gradebook migrated an entire production database off email keys to get
here (`docs/archive/`, the `v2` cutover). The cost is known; do not
re-incur it.

## Firestore vs Postgres

**Principle.** Pick the store from the *shape of the queries the product
needs*, not from familiarity. Document stores are excellent at "fetch
this aggregate by id" and bad at "compute a set by traversing
relationships." Choosing wrong is a rewrite, so choose it explicitly and
write down the reason.

**Practice — Firestore is the default.** It is the right answer when:

- reads are mostly by known id, or by a small number of indexable
  fields;
- the aggregates are self-contained (a course, a module, a document
  with its subcollections);
- realtime listeners on documents are a feature, not a workaround;
- the client can read data directly under rules, which removes a whole
  backend tier.

**Reach for Postgres when the model is genuinely relational:**

- **graph traversal or closure** — "everything reachable from X through
  edges," transitive membership, stratified composition. Overstory's
  club algebra is the worked example: it needs recursive closure over
  club→club edges to answer a single visibility question, which
  Firestore cannot express without denormalizing the entire graph into
  every document;
- **set operations across collections** — joins, `IN` against another
  query's result, aggregate integrity across rows;
- **multi-row invariants** inside one transaction, where correctness
  depends on rows the write does not name;
- **ad-hoc analytical queries** the product will need and nobody can
  enumerate in advance as indexes.

Mixed is fine and normal: Firestore for aggregates, Postgres for the
relational core. What is not fine is denormalizing a graph into
Firestore documents to avoid admitting the model is relational — that
trade buys write amplification and consistency bugs forever.

**Practice — when Postgres wins.** Prefer Firebase Data Connect (Cloud
SQL Postgres) so hosting, auth, and deploy stay in one vendor, and keep
the escape hatch honest: it is real Postgres, so leaving is a data
export, not a rewrite. Local dev runs Postgres in Docker Compose, shared
by the emulators and the functions, so there is exactly one database
under the local stack.

Raw SQL against Data Connect tables tracks the snake_case tables that
generation produces from the GraphQL schema; inserts supply `id` and
`created_at` explicitly, because the schema-level `@default` is applied
by the service layer, not the database.

## Domain clocks

**Principle.** If the product has a domain notion of time — a period, a
tick, a term, an edition — that clock is a value in the database, and
domain logic reads it. Wall-clock time is an input at named seams, not
an ambient fact. Otherwise tests can only be written by lying to the
system clock, which is a thing you will regret on a shared machine.

**Practice.** A table/collection is the authoritative clock; domain
logic keys off the latest recorded tick. Wall time enters only through
explicitly named modules (edition math, TTLs, `created_at`) and always
as a `now` parameter. Dev and test advance the *logical* clock, never
the system clock. A naked `new Date()` inside handler logic is a review
smell.
