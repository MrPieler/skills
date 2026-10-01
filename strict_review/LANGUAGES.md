# Language and framework standards

The standards lens L14 holds code to. Read the section for every language and framework
in scope during step 1. Each section has four parts:

- **Toolchain**: what an expert project runs, at what strictness.
- **Expect**: the idioms an expert would reach for.
- **Flag**: high-impact departures from best practice.
- **Shape**: the ecosystem's architectural conventions.

**Version first.** Note each language's version before judging. Suggesting syntax the
project's version lacks is a false positive. Missing a newer construct that would delete
real complexity is a finding. Version notes are marked *(vX+)*.

**Canonical references** to cite in findings: Python — PEP 8, PEP 20, PEP 484+, the
typing docs. TypeScript — the TS Handbook, typescript-eslint strict configs.
Go — Effective Go, Go Code Review Comments, Go Proverbs. Rust — the Rust API Guidelines,
Clippy lints. Java — Effective Java. Kotlin — Kotlin coding conventions. C# — .NET
Framework Design Guidelines. Swift — the Swift API Design Guidelines. Dart — Effective
Dart. Ruby — the Ruby Style Guide. PHP — PSR-1/4/12. C++ — the C++ Core Guidelines.
For languages not listed here, use the official style guide plus the community-standard
references, and name them.

## Contents

Languages: [Python](#python) · [TypeScript / JavaScript](#typescript--javascript) ·
[Go](#go) · [Rust](#rust) · [Java](#java) · [Kotlin](#kotlin) · [C# / .NET](#c--net) ·
[Swift](#swift) · [Dart](#dart) · [Ruby](#ruby) · [PHP](#php) · [C++](#c) · [C](#c-1)

Frameworks: [React](#react) · [Next.js](#nextjs) · [Node backends](#node-backends) ·
[Django](#django) · [FastAPI](#fastapi) · [Flutter](#flutter) · [Spring](#spring) ·
[ASP.NET Core](#aspnet-core) · [Rails](#rails)

---

## Python

**Toolchain.** A type checker (mypy or pyright) in strict or near-strict mode on core
packages. A linter and formatter (ruff, or black + flake8/pylint). Configuration in
`pyproject.toml`. Locked dependencies.

**Expect.**
- Type hints on every public function; built-in generics `list[str]` *(3.9+)*; `X | None` *(3.10+)*.
- `dataclasses` (`frozen=True`, `slots=True` *(3.10+)*) or pydantic/attrs for records,
  instead of dicts with known keys or tuples indexed by position.
- `enum.Enum`/`StrEnum` *(3.11+)* instead of string constants; `match` *(3.10+)* for
  structural dispatch on variants; `typing.Protocol` for structural interfaces.
- Context managers (`with`) for every resource; `pathlib` over `os.path` string-building.
- Comprehensions and generators over index loops; `enumerate`/`zip` over `range(len())`.
- EAFP where it reads better; specific exception types, `raise … from err` to keep cause.
- Module-level functions where a class would only hold `self` and one method.

**Flag.**
- Mutable default arguments; bare `except:` / `except Exception: pass`.
- Java-isms: getters and setters instead of attributes or `@property`; single-method classes;
  deep inheritance where composition or a function would do; `Manager`/`Helper` classes.
- `dict[str, Any]` or `**kwargs` threading through layers where a typed model belongs.
- `import *`; import-time side effects; circular imports papered over by local imports.
- `isinstance` ladders standing in for polymorphism or `match`.
- Global mutable state; `print` used where logging belongs in library code.

**Shape.** `src/` layout for packages. Thin I/O edges (CLI/HTTP) over a pure core. Explicit
`__all__` or a deliberate public surface. Tests with pytest fixtures, not `unittest`
boilerplate, unless the project standardises otherwise.

## TypeScript / JavaScript

**Toolchain.** `strict: true` at minimum. A strong codebase adds
`noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`. typescript-eslint with
type-aware rules (`no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`).
A formatter. For plain JS, flag the absence of TS or JSDoc `// @ts-check` on non-trivial
code as a finding.

**Expect.**
- Discriminated unions with exhaustive `switch` and a `never` check; `satisfies` *(4.9+)*
  to check without widening; `as const` for literal tables.
- Runtime validation (zod/valibot or equivalent) at trust boundaries, with types inferred from
  the schema: one source of truth.
- `unknown` plus narrowing instead of `any`; type guards instead of `as` casts.
- `async/await` with `Promise.all`/`allSettled` for independent work; `AbortController`
  for cancellation; `using` *(5.2+)* where resource disposal matters.
- ES modules; named exports for discoverability; `readonly` and immutable updates.

**Flag.**
- `any`, `as` casts at boundaries, non-null `!` on plausibly-null values, `@ts-ignore`.
- Floating promises; `async` callbacks passed to `forEach`/`map` and never awaited.
- Enums where a union of literals is simpler (TS `enum` emits runtime code and has
  quirks); `namespace` in modern module code.
- Classes used as bags of static functions; barrel files that create cycles or defeat tree-shaking.
- Hand-written types duplicating a schema, ORM model or API contract that could generate
  them.
- `==` instead of `===`; mutation of function arguments; `var`.

**Shape.** Feature-oriented folders over type-oriented ones. Shared types derive from one
canonical schema. Server-only code kept out of client bundles.

## Go

**Toolchain.** `gofmt`/`goimports`, `go vet`, staticcheck or golangci-lint. Race detector
in CI tests.

**Expect.**
- Errors as values: `if err != nil` returned with context via `fmt.Errorf("…: %w", err)`;
  `errors.Is/As` for inspection; sentinel or typed errors only where callers branch on them;
  `errors.Join` *(1.20+)*.
- Small interfaces defined **by the consumer** ("accept interfaces, return structs").
- `context.Context` as the first parameter of anything blocking or I/O-bound, honoured for
  cancellation.
- `defer` for cleanup; zero values that are useful; table-driven tests.
- Generics *(1.18+)* only where they remove real duplication. `log/slog` *(1.21+)* for
  structured logs.

**Flag.**
- `panic` for ordinary errors; errors ignored with `_`; errors both logged and returned.
- Interfaces declared next to their only implementation "for mocking"; Java-style
  `IService`/`ServiceImpl` pairs; getters named `GetX`.
- `interface{}`/`any` where a concrete type or generic fits.
- Goroutines without an owner, cancellation or error path; shared state without
  synchronisation; channels where a mutex is simpler (and the reverse).
- `util`/`common`/`helpers` packages; package names that stutter (`user.UserService`).
- `init()` doing real work; package-level mutable state.

**Shape.** Packages organised by domain responsibility, not by layer. `internal/` for
non-public code. `cmd/<app>/main.go` as a thin composition root. Avoid a `models` package
that every other package imports.

## Rust

**Toolchain.** `rustfmt`, `clippy` (with `-D warnings` in CI; `clippy::pedantic` selectively).
`cargo deny`/`audit` for dependencies.

**Expect.**
- `Result` with `?`; library errors as enums (`thiserror`), application errors with context
  (`anyhow`/`eyre`); `Option` combinators over match pyramids.
- Newtypes and enums to make illegal states unrepresentable; the type-state pattern where
  protocol order matters.
- Borrowing over cloning; `&str`/`&[T]` parameters; iterators over index loops;
  `impl Trait` in argument position for flexibility.
- Conversion traits (`From`, `TryFrom`, `AsRef`) per API Guidelines C-CONV; `Default`,
  `Debug`, `Clone` derived where sensible (C-COMMON-TRAITS).
- `let … else` *(1.65+)* for early returns.

**Flag.**
- `unwrap`/`expect` in library or request-path code; `unsafe` without a `// SAFETY:`
  justification; `.clone()` sprinkled to silence the borrow checker.
- `Rc<RefCell<…>>`/`Arc<Mutex<…>>` webs that signal an ownership model nobody designed.
- Stringly-typed errors (`Result<T, String>`); `Box<dyn Error>` in library APIs.
- Blocking calls inside async code; holding a `std::sync::Mutex` guard across `.await`.
- Trait objects or generics where an enum over a closed set is simpler.

**Shape.** Small crates with clear public APIs (`pub(crate)` by default). A workspace for
multi-crate systems. The binary crate kept thin over a library crate.

## Java

**Toolchain.** A static analyser (Error Prone, SpotBugs, PMD or Sonar), a formatter, and
null annotations (JSpecify) or a checker (NullAway).

**Expect.**
- Records *(16+)* for data carriers; sealed interfaces *(17+)* plus pattern-matching
  `switch` *(21+)* for closed variants.
- `Optional` as a return type only (not for fields or parameters); immutable collections
  (`List.of`).
- Composition over inheritance; constructor injection; small interfaces; `final` by default.
- try-with-resources; `java.time`; streams where they read better than loops (not
  everywhere).
- Virtual threads *(21+)* over hand-rolled async pyramids for blocking I/O.

**Flag.**
- Anaemic `XxxServiceImpl` layers mirroring every entity; interfaces with one
  implementation and no test double.
- Checked exceptions wrapped and rethrown at every layer, or swallowed; `catch (Exception e)`.
- Null returns for collections; raw types; unchecked casts.
- Field injection; static mutable state; getter/setter boilerplate where a record fits.
- Deep inheritance trees, abstract base classes holding unrelated helpers.

**Shape.** Package-by-feature over package-by-layer. Domain free of framework annotations
where feasible. One composition root (the DI container config).

## Kotlin

**Toolchain.** ktlint or detekt; `-Werror` for compiler warnings where feasible.

**Expect.**
- `val` and immutable collections by default; `data class` for values; `sealed` classes or
  interfaces with exhaustive `when` for variants; `value class` for typed ids.
- Null safety used to model reality: nullable only where absence is meaningful; `?.`, `?:`,
  `let` over manual null checks.
- Coroutines with structured concurrency (scoped launches, `coroutineScope`, `supervisorScope`);
  `Flow` for streams.
- Extension functions for readability, used in moderation; default and named arguments
  instead of overload ladders and builders.

**Flag.**
- `!!`; `lateinit` used to dodge constructor design; Java-style getters and setters.
- `GlobalScope`; coroutines launched without an owning scope; blocking calls in coroutines
  without `Dispatchers.IO`.
- Platform types from Java leaking unchecked nullability into Kotlin APIs.
- `companion object` used as a dumping ground for statics.

**Shape.** As Java. On Android: unidirectional data flow with ViewModel + StateFlow, and
the UI layer free of business logic.

## C# / .NET

**Toolchain.** `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>` (at least for
nullable warnings), .NET analyzers at a recommended or stronger level, `.editorconfig`.

**Expect.**
- Records *(C# 9+)* for values; `required` members *(11+)*; primary constructors *(12+)*
  for DI-heavy classes; pattern matching (`switch` expressions, property patterns).
- `async`/`await` all the way down; `CancellationToken` threaded through I/O;
  `ConfigureAwait(false)` in libraries.
- `IOptions<T>` with validation for configuration; built-in DI with explicit lifetimes.
- `using`/`await using` for disposal; LINQ where it reads better (not on hot paths with
  hidden allocations).

**Flag.**
- `.Result`/`.Wait()` on tasks (deadlocks, thread-pool starvation); `async void` outside
  event handlers.
- The null-forgiving `!` papering over real nullability; `dynamic`; `object` parameters.
- Captive dependencies (singleton holding a scoped service); service locator via
  `IServiceProvider` in business code.
- Exceptions as control flow; `catch (Exception)` without rethrow; generic repositories
  wrapping EF Core with no added value (EF's `DbContext` already is a unit of work).

**Shape.** Vertical slices or clean-architecture projects with dependencies pointing
inward. `Program.cs` as the composition root. Domain projects free of ASP.NET/EF references
where the design claims so.

## Swift

**Toolchain.** SwiftLint, swift-format. Strict concurrency checking (complete in Swift 6
language mode).

**Expect.**
- Value types (`struct`, `enum` with associated values) by default; `final class` when
  reference semantics are needed; protocol-oriented design with small protocols.
- `async/await` and actors *(5.5+)* for concurrency; `@MainActor` for UI state; `Sendable`
  correctness.
- `guard let` for early exit; `Result` or `throws` (typed throws *(6+)*) instead of
  sentinel values; Codable for serialisation.
- API naming per the Swift API Design Guidelines (clarity at the point of use, argument
  labels that read as phrases).

**Flag.**
- Force unwraps (`!`), `try!`, `as!` in non-test code; implicitly unwrapped optionals beyond
  IBOutlets.
- Massive view controllers or views mixing networking, persistence and layout.
- Completion-handler pyramids where async/await is available; GCD juggling replacing actors.
- Singletons reached from everywhere; retain cycles from strong `self` captures in escaping
  closures.

**Shape.** Feature modules (Swift packages) with a thin app target. SwiftUI with
observable models and unidirectional data flow.

## Dart

**Toolchain.** `analysis_options.yaml` including `package:flutter_lints` or `package:lints`
recommended (or stricter: `very_good_analysis`), with `strict-casts`, `strict-inference`
and `strict-raw-types` enabled. `dart format`.

**Expect.**
- Sound null safety used honestly; `final` fields and variables by default; `const`
  constructors where possible.
- Dart 3 *(3.0+)* features: records for small multi-value returns, patterns and
  `switch` expressions, `sealed` classes for closed variants with exhaustive switching,
  class modifiers (`final`, `interface`, `base`).
- Immutable models with `copyWith` (or `freezed`) and value equality.
- `async`/`await` with explicit error handling; `Stream`s closed and subscriptions cancelled.
- Effective Dart naming: `lowerCamelCase` members, `UpperCamelCase` types,
  `lowercase_with_underscores` file names.

**Flag.**
- `dynamic` and unchecked `as` casts; `!` on values that can be null; `late` used to dodge
  initialisation design.
- JSON handled as `Map<String, dynamic>` deep inside the app instead of parsed into typed
  models at the edge.
- Mixed relative and `package:` imports of the same library; files importing `main.dart`.
- Unawaited futures in non-fire-and-forget code (`unawaited()` makes intent explicit).

**Shape.** `lib/src/` for private implementation with a small public surface. For apps,
feature-first folders (`lib/features/<feature>/{data,domain,presentation}` or similar).

---

## React

- Components are pure functions of props and state. Derive values during render; don't
  mirror props or computed values into state.
- Follow the React docs' "You Might Not Need an Effect": no effects for data
  transformation, event handling or state sync. Effects only synchronise with external systems,
  and they clean up.
- Data fetching through a cache layer (framework loaders, server components, TanStack
  Query/SWR), not ad-hoc `useEffect` + `useState` with race conditions.
- State lives at the lowest common owner. Context is for stable, widely-read values, not as a
  global store for frequently changing data. Use a store library only when the state is
  truly global.
- Stable, unique `key`s; hooks never conditional; custom hooks to name and reuse stateful
  logic.
- Flag: god components mixing fetching, business rules and markup; prop drilling through
  5+ layers; reflexive `useMemo`/`useCallback` everywhere *or* missing where referential
  identity actually matters; business logic in event handlers instead of plain modules.

## Next.js

- App Router: Server Components by default. `'use client'` only at the leaves that need
  interactivity. Server-only modules guarded (`server-only`) so secrets and DB clients
  cannot reach the client bundle.
- Data fetched on the server close to where it's used; caching and revalidation intent
  explicit; mutations through Server Actions or route handlers that validate input.
- Flag: whole trees marked `'use client'`; client-side fetching of data the server already
  has; route handlers that trust input; business logic inside `page.tsx`/`route.ts` instead
  of a domain module they call.

## Node backends

- Validation at the edge (zod or equivalent) into typed commands; a domain layer
  independent of Express/Fastify/Nest.
- Centralised error handling middleware; async errors propagated, never swallowed.
- Configuration parsed and validated once at boot; graceful shutdown; no blocking sync I/O
  on the request path.
- Flag: fat route handlers holding business rules; ORM entities returned straight to clients;
  `process.env` read deep in the code.

## Django

- Fat models or a service layer by project convention, applied consistently. Views stay
  thin. Forms or serializers validate input.
- `select_related`/`prefetch_related` to avoid N+1; `transaction.atomic` around multi-write
  operations; settings split by environment, secrets from env.
- Flag: business logic in views or templates; signals used for core domain flow (hidden
  control flow); raw SQL without reason; querysets evaluated in loops.

## FastAPI

- Pydantic models at the boundary (request/response models); dependencies via `Depends`
  for DB sessions, auth and settings (`pydantic-settings`).
- Async endpoints only with async-capable drivers. Blocking calls in `async def` stall the
  event loop.
- Routers per feature; domain logic in plain modules testable without the app.
- Flag: blocking I/O inside `async def`; ORM models returned directly; global DB sessions;
  business logic inside route functions.

## Flutter

- Composition of small widgets. Split `build` methods by **extracting widgets** rather than
  helper methods returning widgets, so rebuilds stay scoped and `const` works.
- `const` constructors wherever possible; keys for list items that move.
- One deliberate state-management approach (Provider/Riverpod/Bloc/etc.), applied
  consistently. Business logic lives outside widgets, in notifiers, blocs, controllers or
  plain Dart.
- Controllers, streams and animation tickers disposed; `mounted`/`context.mounted` checked
  after `await` before using `BuildContext`.
- Data layer (repositories) separate from UI; JSON parsed to typed models; navigation
  through a single routing approach.
- Flag: `setState` in large stateful widgets holding business logic; widgets importing
  `main.dart`; network or storage calls inside `build`; `BuildContext` used across async
  gaps; singletons such as `SharedPreferences` accessed directly from widgets.

## Spring

- Constructor injection; `@Transactional` on service boundaries (and awareness that it
  doesn't apply to self-invocation); `@ConfigurationProperties` with validation over
  scattered `@Value`.
- Flag: business logic in controllers; JPA entities exposed as API DTOs; lazy-loading
  exceptions dodged with eager fetching everywhere; `@Autowired` fields.

## ASP.NET Core

- Minimal APIs or controllers kept thin; validation at the edge; `ProblemDetails` for
  errors; options pattern for configuration; correct DI lifetimes.
- Flag: business logic in controllers; `DbContext` used across threads or captured in
  singletons; sync-over-async; middleware doing domain work.

## Rails

- Follow Rails conventions unless there is a reason not to. Models own domain rules;
  controllers stay thin; service objects or POROs when a model grows several unrelated
  responsibilities.
- Flag: callbacks driving core business flow; N+1 queries (missing `includes`); fat
  controllers; concerns used as dumping grounds; logic in views or helpers.

## PHP

**Toolchain.** `declare(strict_types=1)`; PHPStan or Psalm at a high level; PSR-12 via a
formatter; Composer autoloading (PSR-4).
- Expect: typed properties, parameters and returns; enums *(8.1+)*; readonly properties
  *(8.1+)* and classes *(8.2+)*; constructor promotion *(8.0+)*; `match` *(8.0+)*.
- Flag: arrays as untyped records across layers; static facades/global state in domain
  code; suppressed errors (`@`); mixed HTML and logic.

## Ruby

**Toolchain.** RuboCop (with project config); optionally Sorbet or RBS for types.
- Expect: small objects with clear messages; blocks and Enumerable over index loops;
  keyword arguments; pattern matching *(3.0+)* where it clarifies; frozen string literals.
- Flag: monkey-patching core classes; `method_missing` magic without a strong reason;
  deep inheritance; rescuing `Exception`.

## C++

**Toolchain.** Warnings at `-Wall -Wextra` (or `/W4`) treated as errors; clang-tidy with
Core Guidelines checks; sanitizers (ASan, UBSan) in tests.
- Expect: RAII for every resource; `std::unique_ptr` over owning raw pointers; value
  semantics; `const` correctness; `std::span`/`std::string_view` *(C++17/20)* for views;
  `enum class`; `std::optional`/`std::variant`/`std::expected` *(C++23)* over sentinels.
- Flag: manual `new`/`delete`; C-style casts; macros where `constexpr` or templates fit;
  ownership unclear in interfaces; header-heavy designs that rebuild the world.

## C

**Toolchain.** `-Wall -Wextra -Werror` (or the equivalent); static analysis (clang-tidy,
cppcheck); sanitizers in tests.
- Expect: clear ownership conventions documented at each interface; consistent error
  return conventions; bounds-checked buffer handling; `static` for file-local symbols;
  opaque types for encapsulation.
- Flag: unchecked return values; unbounded string functions; global mutable state; macros
  with side effects; inconsistent cleanup paths (prefer a single `goto cleanup` pattern
  over duplicated frees).
