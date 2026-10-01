# Lenses

The bar first, then fourteen lenses. Each lens gives a **Standard** (what good looks
like), **Signals** (what to flag), **Hunt** (how to find them), and **Remedies**
(moves in [MOVES.md](MOVES.md)). Apply every lens. A lens rated `N/A` in the scorecard
needs a reason.

## Contents

- [The bar](#the-bar)
- L1 [Architecture and dependency direction](#l1-architecture-and-dependency-direction)
- L2 [Module depth and cohesion](#l2-module-depth-and-cohesion)
- L3 [Domain and state model](#l3-domain-and-state-model)
- L4 [Control flow and spaghetti](#l4-control-flow-and-spaghetti)
- L5 [Contracts, types and interfaces](#l5-contracts-types-and-interfaces)
- L6 [Failure handling](#l6-failure-handling)
- L7 [Orchestration, concurrency and atomicity](#l7-orchestration-concurrency-and-atomicity)
- L8 [Duplication and canonical reuse](#l8-duplication-and-canonical-reuse)
- L9 [Size and decomposition](#l9-size-and-decomposition)
- L10 [Testability and test architecture](#l10-testability-and-test-architecture)
- L11 [Naming and domain language](#l11-naming-and-domain-language)
- L12 [Dead weight and speculative generality](#l12-dead-weight-and-speculative-generality)
- L13 [Wiring, configuration and side effects](#l13-wiring-configuration-and-side-effects)
- L14 [Language and framework idiom](#l14-language-and-framework-idiom)

## The bar

**Presumptive blockers.** The codebase is not healthy while any of these stand, unless
the code or an ADR gives a reason that survives the steelman:

1. A dependency cycle between modules, or domain/core code importing infrastructure,
   framework or UI code.
2. Two sources of truth for one fact, kept in step by hand.
3. A multi-step update that a failure midway leaves half-applied.
4. A file over ~1000 lines that mixes responsibilities.
5. The same discriminator (type, kind, status, mode, flag) branched on in 3+ places.
6. Feature-specific checks scattered through shared code paths.
7. A near-duplicate of an existing canonical helper or module.
8. A type-system escape hatch at a seam, hiding the invariant callers rely on.
9. Errors swallowed or flattened at a seam where the caller needs to know.
10. Core logic that can't be exercised without real I/O, a network, or a framework boot.
11. Code fighting its language or framework: hand-building what the platform provides,
    or using a pattern the ecosystem explicitly warns against, in a way that costs
    correctness or maintainability.
12. A reframing that would delete a whole category of complexity, left unproposed. This
    one binds the reviewer: finding it is the job.

**Not findings.** Don't report:

- individual violations a configured linter, formatter, type checker or compiler catches;
- style preferences with no structural or idiomatic cost, or conventions the project chose
  on purpose (stated in CLAUDE.md or an ADR, or applied consistently);
- generated, vendored or third-party code, unless it has been hand-edited;
- micro-optimisations. Report performance only when the structure causes it (N+1 by
  design, serial awaits of independent work, quadratic loops over growing data);
- "I would have written it differently" without a named cost;
- coincidental duplication: two snippets that look alike but change for different reasons;
- verbose tests, unless the verbosity hides what is tested or couples tests to
  implementation.

**Thresholds are defaults, not laws.** Adjust to the language's density and norms. Size
is evidence of mixed responsibility, never the finding by itself: the finding names the
responsibilities to split along.

## L1 Architecture and dependency direction

**Standard.** A recognisable shape: a few modules with one-sentence responsibilities,
dependencies pointing one way (from delivery and infrastructure toward the domain, never
back), and no cycles. The shape on disk matches the shape in the docs.

**Signals.**
- Cycles between modules (the inventory lists them; judge whether each is real coupling
  or a benign parent↔child directory artefact).
- Domain/core files importing HTTP, ORM, UI, framework, SDK or filesystem modules.
- "shared"/"common"/"core" packages importing feature packages.
- A feature reaching into another feature's internals instead of its public entry point.
- Organisation by technical kind (`controllers/`, `services/`, `models/`) where every
  feature change touches every folder. This is a finding only when change-locality
  measurably suffers.
- Docs describing layers the code doesn't respect.

**Hunt.** Start from the inventory's unit graph, cycles and fan-in/fan-out; rerun with a
deeper `--depth` when one unit dominates. Tabulate who-imports-whom between top-level
modules. Open the highest fan-out files first, because layering usually breaks there. Grep core
directories for infrastructure imports (DB clients, `fetch`/`requests`/`http`, `fs`/`os`,
framework names).

**Remedies.** M9 invert the dependency · M10 move logic to its owner · M16 split by
reason to change · M18 centralise the wiring.

## L2 Module depth and cohesion

**Standard.** Each module hides a lot behind a small interface (deep), changes for one
reason (cohesive), and is named for what it owns. Seams exist where something actually
varies.

**Signals.**
- Pass-through layers: functions or classes that forward to one callee with the same
  arguments (controller → service → repository where the service adds nothing).
- Interfaces, abstract classes or ports with exactly one implementation and no test double.
- Understanding one concept means bouncing across many tiny files.
- Dumping grounds: `utils`, `helpers`, `common`, `misc`, `manager`, `base` with unrelated
  contents.
- God modules: high fan-in **and** high fan-out, holding unrelated responsibilities.
- Interfaces leaking implementation: callers must know an ordering ("call `init()`
  first"), internal state, or storage details.

**Hunt.** Apply the deletion test to every suspected wrapper. Grep for interface/protocol/
abstract declarations and count implementors. Open each `utils`-style file and group its
contents by concept; each group is a candidate home elsewhere. For god modules, list what
the top-level symbols are about; three or more unrelated topics means the module should be split.

**Remedies.** M6 collapse the pass-through · M7 deepen · M16 split · M19 unify
near-duplicates.

## L3 Domain and state model

**Standard.** Domain concepts exist as named types with their rules attached. Illegal
states can't be represented, every fact has one source of truth, and mutable state has
one owner.

**Signals.**
- Primitive obsession: ids, money, statuses, units, timestamps as bare strings or numbers,
  re-validated wherever they land.
- Parameter groups that always travel together, meaning a concept with no name.
- Boolean/nullable combinations encoding a state machine (`isLoading && !error && data`),
  or status strings compared ad hoc.
- Stored values derivable from other stored values; caches without an invalidation story;
  parallel lists, enums or maps that must be edited together.
- Mutable shared or global state written from several places.
- Business rules living in UI handlers, controllers, SQL strings or serialisers.

**Hunt.** For each core entity, find every place it is constructed and mutated. List its
fields and ask which combinations are legal. Grep status/kind literals and count the
files comparing them. Look for sync code, meaning functions whose only job is to keep two
things consistent.

**Remedies.** M1 introduce the missing concept · M3 illegal states unrepresentable ·
M4 parse, don't validate · M5 derive, don't store.

## L4 Control flow and spaghetti

**Standard.** Functions read top to bottom at one level of abstraction. Special cases
live with the concept they concern. Adding a variant means adding code, not editing
branches across files.

**Signals.**
- The same discriminator switched on in multiple places. This is the defining spaghetti smell.
- Flag arguments (`render(x, true, false)`) that fork behaviour inside.
- One-off special cases (`if user.id == …`, `if feature == "legacy"`) in shared paths.
- Deep nesting; long `if/else if` chains mapping inputs to outputs.
- Functions that orchestrate and compute at once, or run past ~60–80 lines.
- Exceptions or sentinel values (`-1`, `null`, `""`) used as control flow.

**Hunt.** Grep each enum/kind/status literal and count the distinct files that branch on
it; 3+ is a finding. Use the inventory's nesting proxy. Grep for boolean literal
arguments (`, true)`, `=False`) and read the functions they configure. Read the longest
function in each large file.

**Remedies.** M2 replace the switch with dispatch · M11 special case into default flow ·
M12 replace the flag argument · M13 data over code · M8 functional core.

## L5 Contracts, types and interfaces

**Standard.** Data is parsed into precise types at the edges and trusted inside. Public
interfaces are small, consistent and hard to misuse. Types state the real invariants;
escape hatches are rare, local and justified.

**Signals.**
- Escape hatches (inventory list): TS `any`/`as`/`!`; Python `Any`/`cast`/`type: ignore`;
  Go `interface{}`; C# `dynamic`/`!`; Rust `unwrap`/`unsafe`; Kotlin `!!`; Dart `dynamic`/`!`;
  Swift `as!`/`try!`; Java raw types and unchecked casts.
- Optional/nullable fields that are always or never present in a given state: the
  optionality papers over a missing variant.
- Ad-hoc object/dict shapes crossing modules instead of named types; stringly-typed keys
  and dispatch.
- Validation repeated deep inside; silent fallbacks (`?? default`, `or {}`,
  `.get(k, "")`) hiding an unclear invariant.
- Sibling modules with inconsistent interfaces for the same kind of operation.
- Internals leaking through public interfaces (ORM entities returned from handlers,
  framework types in domain signatures).

**Hunt.** Open the top files of the escape-hatch and suppression lists. At each external
boundary (HTTP, CLI args, files, queues, DB rows, third-party responses) find where data
becomes a trusted type. If it never does, that's a finding. Grep fallback idioms and name
the invariant each one hides.

**Remedies.** M4 parse, don't validate · M3 illegal states · M1 missing concept.

## L6 Failure handling

**Standard.** One deliberate error strategy per layer. Errors are raised where detected,
handled where something can be done about them, and translated at seams. Nothing fails
silently, and partial failure has a defined outcome.

**Signals.**
- Swallowed errors: empty `catch`/`except`, `except Exception: pass`, `_ = err`,
  `catch { return null }`, log-and-continue where the caller needed to know.
- Mixed strategies for the same kind of failure (exceptions here, result objects there,
  `null` elsewhere).
- Broad catches that flatten every failure into one generic message and drop the cause.
- Retries without idempotency; external calls without timeouts.
- The same handling duplicated at every call site instead of at one seam.

**Hunt.** Grep `catch\s*(\([^)]*\))?\s*\{\s*\}`, `except\s*:`, `except Exception`,
`_ = err`, `\.catch\(\(\)\s*=>`, `rescue\s*=>`. For each core flow from the Map step,
trace one failure: where is it detected, translated, surfaced to the user or operator?

**Remedies.** M14 make the update atomic · M8 functional core (errors as values) · M10
move handling to the one seam that owns it.

## L7 Orchestration, concurrency and atomicity

**Standard.** Orchestration is separate from business logic and only as sequential as the
data dependencies require. Related updates commit together or not at all. Concurrent access
to shared state is designed, not hoped for.

**Signals.**
- Sequential awaits/calls of independent operations.
- Multi-write flows with no transaction, compensation or idempotency key.
- Check-then-act races (read, decide, write, with no lock, CAS or unique constraint).
- Fire-and-forget async work with no error path; unawaited promises, tasks or futures.
- Orchestration entangled with computation so neither can be tested alone.
- Event or callback webs where the order of effects is implicit.

**Hunt.** List each core flow's steps with their data dependencies; any independent pair
running serially is a lead. For every flow writing twice or more, ask what happens if the
process dies between the writes. Grep `await` inside loops, `.then(` chains, thread/
goroutine/isolate spawning, global mutable state.

**Remedies.** M14 make the update atomic · M15 parallelise · M8 functional core,
imperative shell.

## L8 Duplication and canonical reuse

**Standard.** Each idea is implemented once, in the module that owns it, and everyone uses
that implementation. A change to one concept touches one place.

**Signals.**
- Near-duplicate functions, components or queries drifting apart.
- A bespoke helper beside a canonical one doing the same job (two HTTP clients, two date
  formatters, two retry loops, two config loaders).
- Shotgun surgery: adding one variant of a concept means editing N files in lockstep
  (enum + switch + map + label + validator).
- Parallel hierarchies or registries that must stay in step.

**Hunt.** Grep distinctive tokens from a suspect body to find siblings. List the canonical
utilities the project already has and grep for re-implementations. Pick one domain variant
(a status, a provider, a type) and grep every file mentioning it: that list is the cost of
adding the next one. Coincidental duplication isn't duplication. Things that change for
different reasons stay apart.

**Remedies.** M19 unify near-duplicates · M2/M13 for shotgun-surgery maps · M5 one
source of truth.

## L9 Size and decomposition

**Standard.** Files and functions are small enough to reread cheaply because each holds
one responsibility. A file's name tells you everything in it.

**Signals.**
- Files over 1000 lines are a presumptive finding. Files of 500–1000 lines must earn their size through cohesion.
- Functions over ~60–80 lines, or that mix levels of abstraction.
- Classes whose methods use disjoint subsets of fields (two classes in one).
- Flat directories with dozens of files and no sub-structure.

**Hunt.** Use the inventory's largest files and size bands. For each large file, outline its
top-level symbols and cluster them by responsibility. The clusters are the proposed split,
named for what each owns. Weak calls between clusters confirm the split.

**Remedies.** M16 split by reason to change · M8 separate orchestration from logic.

## L10 Testability and test architecture

**Standard.** The core can be exercised through its interfaces without real I/O. Tests
assert behaviour at seams, survive refactors, and read as specifications. The judgement is
about whether the design allows good tests, not about coverage percentages.

**Signals.**
- Core logic reachable only through HTTP, the DB or the UI, so tests must boot the world or mock
  everything.
- Heavy mocking of internal collaborators: tests coupled to implementation.
- Tests of private helpers instead of the module's interface.
- Hidden inputs in core code: clock, randomness, environment, globals read directly.
- Critical modules with no tests; fixture setup duplicated across many files.

**Hunt.** For each core module, find its tests and read what they mock. Grep core code for
`Date.now`, `datetime.now`, `time.Now`, `DateTime.Now`, `Math.random`, `random.`, env
reads. List core modules with no corresponding tests.

**Remedies.** M8 functional core · M9 invert the dependency (with a real test adapter) ·
M18 centralise the wiring.

## L11 Naming and domain language

**Standard.** Code uses the domain's words, one name per concept, consistently from UI to
storage. A newcomer learns the domain by reading the names.

**Signals.**
- One concept, several names (`customer`/`client`/`account`), or one name, several concepts.
- Names describing mechanism, not meaning (`DataManager`, `processItems`, `data2`).
- Names that lie: `get*` that writes, `validate*` that mutates, `is*` with side effects.
- Glossary terms the code doesn't use; code terms the glossary lacks.

**Hunt.** Collect the nouns in core type and table names and cluster synonyms. Compare with
`CONTEXT.md`/README vocabulary. Check `get*`/`is*` definitions for writes. Report naming
only when it causes real confusion or masks a modelling problem; never rename-only nits.

**Remedies.** M1 introduce the missing concept (a naming finding is often a modelling one).

## L12 Dead weight and speculative generality

**Standard.** Every line earns its keep today. Flexibility exists only where something
actually varies.

**Signals.**
- Unused exports, functions, files, routes, config keys, manifest dependencies.
- Feature flags for features fully shipped or abandoned; compatibility shims nothing
  calls; commented-out code.
- Plugin systems, generic engines or config-driven frameworks with one plugin or one config.
- Parameters always passed the same value; abstractions for futures that never came.
- TODO/FIXME/HACK clusters, where the codebase confesses its own debt.

**Hunt.** For candidate dead symbols, grep for references outside their definition. Keep the
steelman in mind: dynamic dispatch, reflection, string routing and public library APIs can
make a symbol look dead when it isn't. Check manifest dependencies against actual imports.
Read every TODO/FIXME/HACK cluster from the inventory.

**Remedies.** M17 delete it.

## L13 Wiring, configuration and side effects

**Standard.** Object graphs are assembled in one composition root. Configuration is read,
parsed and validated once at startup into a typed object. Modules do no work at import time.

**Signals.**
- Environment/config reads scattered through the code (`process.env`, `os.environ`,
  `os.Getenv`, `Environment.GetEnvironmentVariable`, `dotenv` calls).
- Singletons, service locators or global registries reached from anywhere.
- Import-time side effects: connections opened, files read, registrations performed.
- Hidden temporal coupling between initialisation steps.

**Hunt.** Grep the env/config idioms and count distinct files. Grep module-level calls
(`connect(`, `open(`, `register(`, `initializeApp(`). Find the entry point and trace how
dependencies reach the core.

**Remedies.** M18 centralise the wiring · M9 invert the dependency.

## L14 Language and framework idiom

**Standard.** The code reads as if written by an expert in **this** language and framework,
at **the version in use**. It uses the language's idioms and the platform's built-ins
instead of hand-built substitutes, follows the ecosystem's architectural conventions, and
runs the standard toolchain at strict settings. Per-language standards are in
[LANGUAGES.md](LANGUAGES.md).

**Signals.**
- Code fighting the language: patterns imported from another ecosystem (Java-style class
  hierarchies in Python or Go, getter/setter boilerplate, manual loops where the language
  has a direct construct, exceptions where the language uses error values, or the reverse).
- Hand-built versions of what the standard library or framework provides (bespoke date
  maths, home-made state management beside the framework's, manual resource cleanup where
  the language has `with`/`using`/`defer`/RAII).
- Framework misuse that costs correctness or structure (the framework sections of
  LANGUAGES.md list the high-impact ones).
- Idioms behind the version in use (missing newer constructs that would simplify the code),
  or ahead of it (suggesting syntax the toolchain doesn't support is a false positive).
- Missing or weak standard tooling: no type checker, a lax compiler mode, no linter, or
  suppression comments used as policy.
- Project layout, packaging or module conventions that break the ecosystem's norms in ways
  that confuse tools or newcomers.

**Hunt.** From step 1 you know each language's version and toolchain config. Read the
matching LANGUAGES.md sections and walk their checklists against the core modules and the
largest files. For every suspected idiom finding, confirm it applies at this version, and
check that the project hasn't deliberately chosen otherwise.

**Reporting.** Every L14 finding names the guideline it rests on (for example *Effective
Go: errors are values*, *PEP 8*, *Rust API Guidelines C-CONV*, *React docs: "You Might Not
Need an Effect"*, *Effective Dart: prefer final fields*). Group idiom findings by root cause:
"this codebase writes Python as if it were Java" is one finding with many pieces of
evidence, not twenty.

**Remedies.** Usually M17 delete it (replace the hand-built thing with the built-in), M3
or M4 using the language's own type tools, or M18 for framework-sanctioned wiring.
