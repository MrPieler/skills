---
name: strict_review
description: Runs an extremely strict, read-only audit of a codebase as it stands today — architecture, dependency direction, module depth, domain and state modelling, spaghetti control flow, oversized files, duplication, contracts, failure handling, testability, and adherence to the best practices of each language and framework in use — and writes a prioritised report of ambitious, behaviour-preserving improvement suggestions ready for follow-up. Never edits code and ignores git history and PRs. Use for a strict review, a harsh or relentless codebase audit, a whole-codebase or architecture quality review, a deep maintainability audit of existing code, or when the user invokes /strict_review.
disable-model-invocation: true
---

# Strict Review

A whole-codebase review. It doesn't judge a diff or a PR. It judges the code **as it
stands**, both its code and its architecture, and asks what it would take to make it
dramatically better. The output is a report of suggestions. Nothing in the repository
changes.

The temperament is strict, serious and ambitious. The question is never "is this
acceptable?" but "what is the simplest design that could do this job, written the way
an expert in this language would write it, and how far is the code from it?"

## Hard rules

1. **Read-only.** Never edit, create, move or delete files in the repository. Never run
   formatters, fixers, codemods, installers, migrations, builds or test suites, since they
   write caches and artifacts into the tree. Read-only commands (listing, `grep`, line
   counts, the inventory script) are fine. The only file you write is the report, outside
   the repository. Applying a suggestion is a separate task the user starts afterwards.
2. **Present tense only.** Ignore history: no `git log`, `blame`, `diff`, churn, PRs or
   branches. Judge what the code *is*. `git ls-files` is allowed purely to list files.
3. **Evidence or silence.** Every finding cites `file:line` and is verified by reading
   the code. A suspicion you could not verify is labelled `Probable` with what would
   confirm it, or dropped.
4. **Behaviour-preserving by default.** Suggestions keep observable behaviour identical.
   A suggestion that changes behaviour, bug fixes included, says so explicitly.
5. **Structure and idiom over nits.** Never list individual violations a configured
   linter, formatter or compiler already catches. If the project lacks its language's
   standard tooling, or runs it with weak settings, that is **one** finding.

## Be ambitious

Don't stop at "this could be a bit cleaner." For every hotspot, look for the **code-judo
move**: a reframing that deletes whole branches, layers, modes or helpers rather than
rearranging them, and makes the design feel inevitable in hindsight. Prefer the remedy
highest on this ladder: **delete → reframe the model → move ownership → restructure →
extract**. A refactor that leaves the reader holding the same number of concepts is not
an improvement. A cleaner version of the same messy idea is not enough when a simpler
idea is plausible. Catalogue of moves and anti-moves: [MOVES.md](MOVES.md).

## Workflow

Copy this checklist and tick it off as you go:

```
- [ ] 1 Ground     docs, rules, stack, language versions, toolchain
- [ ] 2 Inventory  run scripts/inventory.py
- [ ] 3 Map        reconstruct the actual architecture; trace core flows
- [ ] 4 Sweep      every lens in LENSES.md, including L14 language idiom
- [ ] 5 Ambition   code-judo pass over every hotspot
- [ ] 6 Verify     evidence, steelman, deletion test, path check
- [ ] 7 Report     write the report; summarise in chat; offer follow-ups
```

### 1. Ground: learn what the code is meant to be

- **Scope**: the whole repository unless the user names a path, package or app.
- **Intent**: read, where present, root and nested `CLAUDE.md`/`AGENTS.md`, `README`,
  `CONTEXT.md` or a glossary, architecture docs, and `docs/adr/`. Judge the code against
  its own stated rules as well as against general principle. ADRs are decisions: raise a
  finding against one only when the friction is concrete, and say "reopens ADR-NNNN
  because…".
- **Stack**: read the manifests (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`,
  `pubspec.yaml`, `*.csproj`, `pom.xml`/`build.gradle`, `Gemfile`, `composer.json`…) for
  languages, frameworks, entry points.
- **Language baseline**: record each language's **version** (it decides which idioms are
  available) and its **toolchain configuration**: compiler strictness, linter and
  formatter configs, type checker. Then open [LANGUAGES.md](LANGUAGES.md) and read the
  sections for every language and framework in scope. These are the standards L14 holds
  the code to. For a language LANGUAGES.md doesn't cover, use its official style guide and
  the community-standard references, and name them in findings.

### 2. Inventory

Run `python "<this skill's base directory>/scripts/inventory.py" "<root>"`. It is
read-only and prints language mix, size distribution, largest files, mass per unit,
debt markers, suppressions, type-system escape hatches, a nesting proxy, and a heuristic
import graph (Python, JS/TS, Go, Dart) with cycles and fan-in/fan-out. Options:
`--depth N` (rerun deeper when it hints that one unit dominates), `--exclude GLOB` (for
generated platform scaffolding such as Flutter's `android/*`/`ios/*`/`windows/*`), and
`--top N`. Everything it prints is a lead to read, never a finding by itself. Markup-heavy
files (JSX, templates) inflate the nesting proxy.

### 3. Map: reconstruct the actual architecture

Before judging anything, write down (it goes into the report):

- the **modules** by real responsibility, not folder name, each with the one sentence it
  owns. If you can't write that sentence, that is a finding;
- the **dependency direction** between them, and every edge that violates it;
- where **state** lives and who may mutate it;
- **2–4 core flows** traced end to end, from entry point to side effect;
- **intended vs actual**: where the docs and the code disagree.

Most of the highest-value findings come from this step.

### 4. Sweep: apply every lens

Work through **all** lenses in [LENSES.md](LENSES.md), L1–L14. Each has a standard,
signals, and a concrete way to hunt for them. Depth by codebase size:

- **Under ~5k lines**: read every source file.
- **Up to ~15k lines**: read the core and every file over 300 lines in full; sample the
  periphery.
- **Larger**: fan out to read-only subagents (general-purpose type), one per area or lens
  group, using the brief in [REPORT.md](REPORT.md#subagent-brief). Keep cross-cutting
  judgments (layering, domain model, code-judo moves) in your own context, because they
  need the whole map.

### 5. Ambition pass: find the code-judo

For each hotspot from steps 3–4, ask:

- If this were written today by an expert in this language who knew everything the code
  now knows, what would it look like? How far is the code from that, and what
  incremental path gets there?
- What would have to be true for this branch, layer, mode or flag to disappear entirely?
- Which **missing concept** forces these conditionals or duplicates into existence?
- Does the language or framework already provide what this code builds by hand?

### 6. Verify: hold every finding to the bar

For every candidate:

- **Re-read** the cited code. Does the evidence say what the finding claims?
- **Steelman**: state the strongest reason the code is the way it is (framework
  constraint, performance, external contract, ADR, language version). If the suggestion
  doesn't beat it, drop or downgrade it.
- **Deletion test** for any abstraction you call shallow: if it were deleted, would
  complexity vanish or reappear across its callers?
- **Path check**: can it land in behaviour-preserving steps? Big-bang rewrites need a far
  stronger case and a staged path (MOVES.md, M20).
- **Rate** severity, conviction and effort per [REPORT.md](REPORT.md).
- **Merge** findings that share a root cause. Five symptoms of one missing concept make
  one finding with five pieces of evidence.

Before reporting, confirm that every lens has a scorecard rating, every core flow was
traced, and every file over 500 lines was opened. If any of these is missing, the audit
is not done.

### 7. Report

Write the report to the session scratchpad if one is provided, otherwise to the OS temp
directory (`$TMPDIR`, `%TEMP%` on Windows, else `/tmp`), as
`strict-review-<repo>-<YYYYMMDD-HHMM>.md`, following the template in
[REPORT.md](REPORT.md). Give the absolute path. In chat, give the verdict, the scorecard,
the top moves, and the finding counts by severity. Then **offer** follow-ups and don't start them:

- turn findings into issues (`/to-issues`) or hand them to another agent (`/handoff`);
- stress-test a proposed move before committing to it (`/grill-me`,
  `/improve-codebase-architecture`);
- implement a named finding, as a separate, explicit request.

## Tone

Direct, serious, demanding. Not rude. Don't soften a structural problem into a mild
suggestion, and don't inflate a nit into drama. Phrasing guidance is in
[REPORT.md](REPORT.md#tone).
