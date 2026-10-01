# Moves

The remedies a finding proposes. Each move gives **When** (the smell), **Move** (the
restructuring), **After** (the shape it leaves), and **Watch out** (when it's the wrong
call). Sketches are pseudo-code; render them in the project's language and idiom (L14)
when writing a finding.

## The ambition ladder

Try remedies from the top. Settle lower only when the higher rungs don't apply, and say
why in the finding.

1. **Delete.** The complexity was never needed (M17, M6).
2. **Reframe the model.** A better concept makes the conditionals disappear (M1, M3, M5,
   M11).
3. **Move ownership.** The feature becomes a natural extension of an existing owner (M10, M9).
4. **Restructure.** Split, merge or dispatch so each part holds one idea (M16, M7, M2).
5. **Extract.** Name a piece and reuse it (M19, M13).

## Contents

M1 [Introduce the missing concept](#m1-introduce-the-missing-concept) ·
M2 [Replace the repeated switch with dispatch](#m2-replace-the-repeated-switch-with-dispatch) ·
M3 [Make illegal states unrepresentable](#m3-make-illegal-states-unrepresentable) ·
M4 [Parse, don't validate](#m4-parse-dont-validate) ·
M5 [Derive, don't store](#m5-derive-dont-store) ·
M6 [Collapse the pass-through layer](#m6-collapse-the-pass-through-layer) ·
M7 [Deepen: merge shallow modules](#m7-deepen-merge-shallow-modules) ·
M8 [Functional core, imperative shell](#m8-functional-core-imperative-shell) ·
M9 [Invert the dependency](#m9-invert-the-dependency) ·
M10 [Move logic to its owner](#m10-move-logic-to-its-owner) ·
M11 [Turn the special case into the default flow](#m11-turn-the-special-case-into-the-default-flow) ·
M12 [Replace the flag argument](#m12-replace-the-flag-argument) ·
M13 [Data over code](#m13-data-over-code) ·
M14 [Make the update atomic](#m14-make-the-update-atomic) ·
M15 [Parallelise independent work](#m15-parallelise-independent-work) ·
M16 [Split by reason to change](#m16-split-by-reason-to-change) ·
M17 [Delete it](#m17-delete-it) ·
M18 [Centralise the wiring](#m18-centralise-the-wiring) ·
M19 [Unify the near-duplicates](#m19-unify-the-near-duplicates) ·
M20 [Strangle, don't rewrite](#m20-strangle-dont-rewrite) ·
[Anti-moves](#anti-moves)

---

### M1 Introduce the missing concept

**When.** The same parameter group travels together. The same computation is repeated near
the same data. Conditionals orbit an idea nobody has named.
**Move.** Name the idea as a type or module and move the behaviour onto it.
**After.** `price(amount, currency, taxRate, region)` becomes `price(Money, TaxPolicy)`.
The re-validation and the conversion helpers scattered around the code disappear.
**Watch out.** Name it in the domain's language (CONTEXT.md), not as `Info`/`Data`/`Context`.

### M2 Replace the repeated switch with dispatch

**When.** The same discriminator is branched on in 3+ places, so adding a variant means
editing all of them.
**Move.** Give each variant its behaviour once: polymorphism, a strategy per variant, or a
registry keyed by the discriminator.

```
before:  label(k)  { switch k: case A: …  case B: … }      // in 5 files
         fee(k)    { switch k: case A: …  case B: … }
after:   VARIANTS = { A: {label, fee, …}, B: {label, fee, …} }
         VARIANTS[k].fee(order)                            // adding C = one entry
```

**Watch out.** Two call sites are not yet a pattern. For a small closed set, one
exhaustive `match`/`switch` in **one** place (compiler-checked) often beats a class
hierarchy.

### M3 Make illegal states unrepresentable

**When.** Booleans or nullables combine into states that must never occur
(`loading && error`), and code keeps guarding against them.
**Move.** Model the states as a tagged union, sealed hierarchy or enum with data.

```
before:  { loading: bool, error?: E, data?: D }       // 8 combinations, 3 legal
after:   Idle | Loading | Failed(E) | Loaded(D)       // 4 states, all legal
```

**After.** The guards disappear, and the compiler enforces exhaustive handling.
**Watch out.** Use the language's native construct (TS unions, Rust/Swift enums, Kotlin/
Java/Dart sealed types, Python unions with `match`). Don't hand-roll a visitor pattern
where the language has pattern matching.

### M4 Parse, don't validate

**When.** Null checks, format checks and range checks are repeated deep inside the code,
or silent fallbacks (`?? ""`) hide what the data actually guarantees.
**Move.** Parse untrusted input once, at the edge, into a type that carries the guarantee
(`EmailAddress`, `NonEmptyList`, a validated `Command`). Inner code accepts only the
parsed type.
**After.** Defensive checks inside the core disappear, and invalid data can't reach it.
**Watch out.** The parser must reject loudly. A parser that defaults bad input to
something plausible is a silent fallback with extra steps.

### M5 Derive, don't store

**When.** A stored value is a function of other stored values. Sync code keeps two
representations aligned. Parallel lists or enums must be edited together.
**Move.** Keep one source of truth and compute the rest (memoise only if measured).
Generate the parallel list from the canonical one, or co-locate them in one table.
**After.** The sync code and its bugs disappear.
**Watch out.** Sometimes storing is right: audit trails, expensive aggregates, snapshots
of external state. Then make the derivation explicit and the update atomic (M14).

### M6 Collapse the pass-through layer

**When.** A layer forwards calls unchanged, and the deletion test says complexity would
vanish with it.
**Move.** Delete the layer; callers use the deep module directly.
**Watch out.** A layer that translates between two models, enforces a policy, or is a real
seam (two adapters) is earning its keep. Thin is not the same as useless.

### M7 Deepen: merge shallow modules

**When.** Understanding one concept means reading five tiny modules whose interfaces are
as complex as their bodies.
**Move.** Merge them behind one small interface that hides the sequencing and the
intermediate types.
**After.** Callers learn one interface, and tests target it.
**Watch out.** Don't merge things that change for different reasons. That is M16 in
reverse.

### M8 Functional core, imperative shell

**When.** Decisions are interleaved with I/O, so the logic can only be tested with mocks or
real infrastructure.
**Move.** Gather inputs → call a pure function that decides and returns a plan or result →
perform the effects in a thin shell.

```
before:  handle(req) { u = db.get(); if … { mail.send(); db.save(…) } else … }
after:   decide(user, req) -> Decision          // pure, exhaustively testable
         handle(req) { d = decide(db.get(), req); apply(d) }   // thin shell
```

**Watch out.** Don't contort naturally sequential, I/O-dependent flows into "plans" of
plans. Apply the move where decisions are rich and I/O is incidental.

### M9 Invert the dependency

**When.** Core code imports infrastructure (DB, HTTP, filesystem, framework).
**Move.** The core declares the port it needs, in its own terms. Infrastructure implements
it, and the composition root (M18) connects them.
**Watch out.** One adapter is a hypothetical seam. Justify the port with a second adapter
(an in-memory test adapter counts). Otherwise you have added indirection, not
architecture.

### M10 Move logic to its owner

**When.** Code computes mostly on another module's data (feature envy), or
feature-specific checks live in shared code.
**Move.** Move the logic into the module that owns the data or concept, and expose the
result, not the internals.
**After.** Shared paths stop knowing about features, and the owner's invariants stay in
one place.

### M11 Turn the special case into the default flow

**When.** Edge cases are handled by branches bolted onto the main path (empty list, first
item, missing optional, "legacy" mode).
**Move.** Generalise the main path so the edge case is an ordinary instance: the empty
collection that just iterates zero times, a null object, an identity element, a default
policy.
**After.** The branch is gone, not moved.
**Watch out.** If the special case really is different behaviour, model it as a variant
(M2/M3) rather than hiding it.

### M12 Replace the flag argument

**When.** A boolean or mode parameter forks a function's behaviour.
**Move.** Split it into two functions with honest names, or pass the variant object that
carries the behaviour.
**After.** `render(doc, true, false)` becomes `renderPreview(doc)`.

### M13 Data over code

**When.** Long `if`/`switch` chains map inputs to outputs, or rules differ only in
constants.
**Move.** Express the mapping as a data table with a single small interpreter.
**Watch out.** Once the table needs conditionals of its own, you are building a
configuration language. Stop and use code (M2).

### M14 Make the update atomic

**When.** A flow writes in several steps, and a failure between them leaves state
inconsistent.
**Move.** Use, in order of preference: one write of a composed value; a database
transaction; build-then-swap (write new, switch the pointer); idempotent steps plus an
outbox or retry; compensation as the last resort.
**After.** Every failure leaves the old state or the new state, never a mix of both.

### M15 Parallelise independent work

**When.** Operations with no data dependency run one after another.
**Move.** Run them concurrently with the language's structured primitive (`Promise.all`,
`asyncio.gather`/`TaskGroup`, `errgroup`, `coroutineScope`, `Task.WhenAll`, `join!`).
**Watch out.** Make sure they really are independent, including rate limits, ordering
assumptions and shared transactions. The goal is a simpler flow, not micro-optimisation.

### M16 Split by reason to change

**When.** A file or class holds several responsibilities, or its methods use disjoint
subsets of its fields.
**Move.** Split along the responsibilities, and name each part for what it owns.
**After.** `OrderService` (1,400 lines) becomes `OrderPricing`, `OrderFulfilment` and
`OrderNotifications`, each readable in one sitting.
**Watch out.** Don't split by syntactic kind (`types.ts`, `helpers.ts`, `constants.ts`).
That scatters one concept across files: an anti-move.

### M17 Delete it

**When.** The code is dead, a flag is permanently on or off, a shim has no callers, a
generic mechanism has one user, or something hand-builds what the language or framework
provides.
**Move.** Delete it, or replace it with the built-in.
**After.** Fewer concepts. Deletion is the strongest code-judo move there is.
**Watch out.** Check for reflection, string-based dispatch, public API consumers and
external callers before calling something dead. Name what you checked in the finding.

### M18 Centralise the wiring

**When.** Environment reads, singletons and import-time side effects are scattered
everywhere.
**Move.** Build one composition root at the entry point. It parses and validates config
into a typed object, constructs the object graph, and passes dependencies explicitly.
**After.** Modules become pure to import and trivial to construct in tests.

### M19 Unify the near-duplicates

**When.** Two implementations of the same idea are drifting apart, or a bespoke helper sits
beside the canonical one.
**Move.** Pick the canonical one (or write the right one), migrate the callers, then delete
the others.
**Watch out.** Unify only things that change for the same reason. Coincidental likeness is
not duplication.

### M20 Strangle, don't rewrite

**When.** The target design is far from the current one.
**Move.** Put a seam in front of the old code. Route new and migrated behaviour through
the new shape, move callers incrementally, and delete the old path when it has no callers.
Each step is landable and behaviour-preserving.
**Watch out.** A suggestion that needs a big-bang rewrite needs an extraordinary case. Say
what makes incremental migration impossible.

---

## Anti-moves

Refactors that look like improvements but aren't. Never propose them, and flag them when
the codebase already did them:

- **Spreading, not reducing.** Extracting a method that carries the same branch somewhere
  else. The reader still holds every concept.
- **Premature abstraction.** An interface with one implementation and no test double, or a
  generic engine for two cases.
- **DRYing coincidences.** Merging code that looks alike but changes for different reasons.
  The next change forces flags into the shared version.
- **Splitting by kind.** `types/`, `utils/`, `constants/`, `helpers/` holding fragments of
  every feature.
- **Ceremonial layering.** Controller → Service → Repository where the middle layer only
  forwards.
- **Wrapping the library "in case we swap it".** A port with no second adapter, which
  usually also hides the library's best features.
- **Configuration as code-in-disguise.** YAML/JSON rule tables with conditionals and
  expressions, which is a worse programming language.
- **Pattern cosplay.** Factories, visitors, builders or observers where the language has a
  direct construct (functions, pattern matching, named arguments, closures).
