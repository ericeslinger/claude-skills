# Frontend

## Angular, current-idiom

**Principle.** Use the framework the way its current version wants to be
used. Carrying an older idiom forward costs more than learning the new
one, and mixed idioms in one app are the worst of both.

**Practice (2026-08).** Angular 22+:

- **Standalone components.** No `NgModule`.
- **Signals** for component state. `input()` / `output()` functions, not
  the decorators.
- **Zoneless.** `zone.js` is not a dependency. Do not add it, and do not
  add `provideZoneChangeDetection`.
- **Control flow blocks** (`@if`, `@for`, `@switch`) — not
  `*ngIf` / `*ngFor`.
- **`@for` must `track` a stable identity** — `track item.id`, `track
  row.key`, `track $index` only for genuinely id-less primitives.
  Tracking a whole object re-creates the DOM on every change-detection
  pass for any collection that is re-emitted as a new array. This was a
  34-edit sweep across one repo; do not re-earn it.
- Routing with `withComponentInputBinding()`; route params arrive as
  component inputs.
- Angular Material where the project already uses it; otherwise plain
  components. Do not introduce a second component library.
- `packages/frontend/angular.json` — the Angular workspace lives in the
  package, not at the repo root.

## Accessibility is a build gate, not a polish pass

**Principle.** Accessibility defects are cheap to prevent and expensive
to retrofit, and the retrofit never gets scheduled. The suite enforces
it so it cannot be deferred.

**Practice.** The axe journey (see testing.md) fails the build on
serious and critical violations. Beyond what axe can see:

- **Every interactive control carries a descriptive accessible name
  that names the object, not just the verb.** `aria-label="Remove
  Kiln & Wheel from audience"`, not `aria-label="Remove"`. This is the
  single most common finding in these repos.
- A clickable icon is a `<button>` or an `<a>`, never a bare `<span>` or
  `<mat-icon>` with a click binding. Icons inside are `aria-hidden`; the
  label lives on the control.
- **No `outline: none` without a `:focus-visible` replacement.**
- Dropdowns, popovers, and comboboxes are keyboard-navigable: arrows,
  Escape, Enter.
- Because journeys query by role and accessible name, a control without
  one is hard to test — treat a test that needs a test id as a signal
  that the control is wrong.

## Guarded platform access

**Principle.** Browser APIs that can throw or be unavailable
(localStorage in private modes, clipboard without permission) must fail
soft in one place, not at twenty call sites.

**Practice.** Storage and clipboard access goes through guarded helper
functions, never raw `localStorage.setItem` / `navigator.clipboard` in a
component.

## Talking to the backend

**Principle.** The frontend has exactly one door to the backend, and it
is typed by the shared contract.

**Practice.** A single `backend.service.ts` wraps the callable surface,
calling each endpoint **by its frozen export name**, with argument and
return types imported from the schema package. Components inject that
service; components never construct callables themselves.

Rendering user-authored HTML goes through sanitization (DOMPurify) at
one seam, not ad-hoc per template.
