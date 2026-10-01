# Report

How findings are rated, written and assembled. Every finding must pass the **handoff
test**: a competent engineer or a fresh agent with no memory of the audit can act on it
from its text alone.

## Contents

- [Ratings](#ratings)
- [Finding template](#finding-template)
- [Report template](#report-template)
- [Chat summary](#chat-summary)
- [Subagent brief](#subagent-brief)
- [Tone](#tone)

## Ratings

**Severity**: how much the problem costs.

| Level | Meaning |
|---|---|
| **S1 Critical** | The structure actively produces defects or blocks change: a presumptive blocker (LENSES.md, The bar) with wide reach. Examples: cycles through the core, two sources of truth for central data, non-atomic writes to important state, an untestable core. |
| **S2 Major** | A presumptive blocker of limited reach, or a structural problem that makes a common kind of change expensive (one concept, N files). |
| **S3 Moderate** | Real friction, contained to one module or one flow. |
| **S4 Minor** | Worth knowing. Goes only in the short "Also noticed" list, never padded. |

**Conviction**: how sure you are.

- **Confirmed**: traced in the code; every link of the argument is cited.
- **Probable**: strong evidence, but one link is unverified. Name what would confirm it.
- Anything weaker is not a finding. If a human could settle it, put it under *Questions
  for the maintainers*. Otherwise drop it.

**Effort**: the scope of the change, not a time estimate.

- **S**: one module, one sitting.
- **M**: several modules, one coherent change.
- **L**: a subsystem, in several staged changes.
- **XL**: a cross-cutting migration (strangler path required, M20).

**Behaviour**: `preserving` (the default) or `changing`. A changing finding states exactly
what changes.

**Ordering.** Sort by severity, then by leverage (payoff ÷ effort) within each severity.
An S2 that is effort S and unblocks three other findings outranks an S2 that is effort L.

## Finding template

```markdown
### SR-07 · <Imperative title: what to do, not what is wrong>
**Severity** S2 · **Conviction** Confirmed · **Effort** M · **Lens** L3 Domain & state ·
**Behaviour** preserving · **Move** M3

**Where.** `orders/service.py:120-188`, `orders/api.py:40-61`, `billing/invoice.py:12-30`

**Problem.** One sentence stating the structural fact.

**Evidence.**
- `orders/service.py:131`: `if order.status == "paid" and not order.shipped_at:` … (≤3-line
  quotes, each with its location)
- the same check in `orders/api.py:52` and `billing/invoice.py:18`

**Cost today.** What this makes expensive or fragile, concretely. ("Adding a 'refunded'
status means editing 6 files; two of them already disagree on what 'shipped' means.")

**Steelman.** The strongest reason the code is this way, and why the proposal still wins.

**Proposal.** The target shape in plain language, plus a short sketch at signature or
pseudo-code level, in the project's language and idiom.

**Path.**
1. Behaviour-preserving step, independently landable.
2. …
3. Delete the old path.

**Verify.** How to prove behaviour is unchanged: the existing tests that cover it, or the
characterisation tests to write first if none exist.

**Risks.** Blast radius, migration hazards, what might break that tests won't catch.

**Relations.** Depends on SR-03 · Unblocks SR-09 · Subsumes SR-11.
```

For L14 findings, add **Guideline**: the named reference the finding rests on
(LANGUAGES.md lists them), plus the language version that makes it applicable.

## Report template

```markdown
# Strict review: <repo> (<scope>)

<YYYY-MM-DD> · <N source files, M lines> · <languages with versions> · <frameworks>
Read-only review of the code as it stands. No files were changed.

## Verdict

3–5 sentences. Overall health. The one structural problem that matters most. Whether the
architecture is fundamentally sound and needs pruning, or fundamentally strained and needs
reshaping.

## Scorecard

| Lens | Rating | Why (one line) |
|---|---|---|
| L1 Architecture & dependency direction | Healthy / Strained / Failing / N/A | … |
| … all fourteen lenses … | | |

## Top moves

The 3–5 findings with the best payoff-to-effort ratio, in the recommended order, one line each with
its ID. If you could only do three things, these are they.

## Architecture map

- **Modules**: table of module | responsibility (one sentence) | depends on | notes.
- **Dependency graph**: a Mermaid `graph LR` of modules. Violating edges drawn dashed and
  labelled.
- **Core flows**: each traced flow as a numbered chain of `file:function` steps.
- **Intended vs actual**: where the docs and the code disagree.

## Language & toolchain baseline

Per language: version, toolchain and strictness found, notable gaps. This is the context for L14.

## Findings

### S1 Critical
…findings, ordered by leverage…
### S2 Major
### S3 Moderate

## Also noticed (S4)

At most ~10 one-liners, each with `file:line`.

## Defects noticed in passing

Correctness bugs seen along the way, marked behaviour-changing. They are outside the
review's main focus, but never hide one.

## Questions for the maintainers

Open, non-leading questions whose answers would change a finding ("Who calls the v1
export endpoint today?"), each tagged with the finding it affects.

## Roadmap

Findings sequenced into dependency-ordered phases. For example: Phase 1 is foundations
(characterisation tests, wiring), Phase 2 is the model (missing concepts, illegal states),
Phase 3 is the structure (splits, inversions), Phase 4 is clean-up (deletions, idiom).
Each phase lists its finding IDs and what it unblocks.

## Method & limits

What was read in full, what was sampled, what was excluded and why. Tools run. The
inventory's heuristic limits. Lenses where the review is weaker than usual.
```

## Chat summary

After writing the report, reply with:

1. The absolute report path.
2. The verdict (the 3–5 sentences).
3. The scorecard, compressed to `Lens: rating`, failing and strained first.
4. The top moves, one line each.
5. Counts: `S1 n · S2 n · S3 n · S4 n · defects n · questions n`.
6. The follow-up offer (SKILL.md step 7). Don't start any of it.

## Subagent brief

When fanning out, give each subagent this brief, with the bracketed parts filled in:

```
You are a read-only sub-reviewer in a strict code review of the code as it stands.
Do not edit, create, move or delete any file. Do not run builds, tests, formatters,
installers or git history commands (log, blame, diff). Reading files, grep and
listing are fine.

Scope: [paths]
Lenses to apply: [e.g. L4, L5, L6, L14] (definitions in [absolute path to LENSES.md])
Language standards: [absolute path to LANGUAGES.md], sections [Python, FastAPI]
Context: [architecture map so far: modules, responsibilities, dependency direction]

Read every source file in your scope that is under ~600 lines in full; outline larger
ones, then read their densest parts. Return:
1. Candidate findings, each with: title, lens, file:line ranges, evidence (≤3-line
   quotes), the concrete cost it imposes, a proposed remedy, confidence
   (confirmed/probable), and for idiom findings the guideline it rests on.
2. The one-sentence responsibility of each module in your scope.
3. Dependencies you saw leaving your scope (from → to, with file:line).
4. Anything you suspect is cross-cutting but could not confirm from inside your scope.
Report nothing for a lens with no real findings. Don't pad.
```

You verify every candidate a subagent returns before it enters the report. Subagent
output is a lead, exactly like the inventory.

## Tone

Direct, serious, demanding. Not rude. Name structural problems as structural. Don't
soften them into suggestions or inflate nits into drama.

- `orders/service.py is 1,400 lines holding three responsibilities: pricing, fulfilment
  and notifications. Split along those seams (M16).`
- `kind is switched on in five files. That's a missing model: one variant type with its
  behaviour attached, and those switches disappear (M2).`
- `this layer forwards every call unchanged. Delete it; callers lose nothing (M6).`
- `orders/ imports billing/ and billing/ imports orders/. Neither can change alone. Move
  invoice creation into billing, or invert through a port (M10/M9).`
- `status is stored and also derivable from events, so there are two sources of truth. Derive it
  and the sync code goes (M5).`
- `this works, but every caller re-checks the same nulls. Parse once at the edge and these
  branches vanish (M4).`
- `this is Java written in Python: getter/setter classes and a ServiceImpl per entity.
  Dataclasses and module functions delete about a third of this package (L14, PEP 8/PEP 20).`
- `the widgets import main.dart to reach shared state. Lift that state into a provider
  and the cycle disappears (L14, Flutter).`
- `there's a code-judo move here: if X owned Y, these three special cases would be ordinary
  instances of the default flow (M11).`
