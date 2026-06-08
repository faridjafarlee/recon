# Design: `/recon suggest` and `/recon plan` — generative subcommands

**Date:** 2026-06-07
**Skill:** `recon` (global, `~/.claude/skills/recon/`)
**Status:** Design — pending user approval before implementation planning

---

## 1. Motivation

Today recon is a **read-only diagnostic** pipeline (Setup → Map → Hunt → Fix → Optimize)
plus two focused subcommands (`test-frontend`, `pentest`). Every worker is read-only;
even the Fix/Optimize phases have the **parent** apply edits while workers stay read-only.

This design adds two **generative, forward-looking** subcommands:

- **`/recon suggest`** — ideate new features for *this* project, grounded in the
  Phase-1 (Setup) + Phase-2 (Map) knowledge recon already builds.
- **`/recon plan "<task>"`** — take an arbitrary development task from the operator
  (e.g. "make plan for mobile app development") and produce a full development plan.

Both converge on a shared back half: **planner → plan-review gate → "implement?" gate →
parallel implementer worktrees → cherry-pick back**. The two subcommands differ only in
how the planner is *seeded*.

This introduces recon's **first parallel code-writing worker** (`implementer`). Its
read/write boundary, gating, worktree isolation, and "never the default branch" rules are
specified as explicitly as the pentester's safety contract.

**This same backbone also REPLACES the existing Phase 4 (Fix) and Phase 5 (Optimize)
execution models.** Today those phases have the **parent** apply each change **sequentially**,
lowest-risk-first. Under this design they are re-expressed on the shared engine: a planner
(Opus) writes a detailed plan per selected finding, then parallel `implementer`s (Sonnet) —
one per plan, each in its own worktree — apply the change, run the tests, and commit; the
parent cherry-picks the separate commits back (no squashing). All the Phase-4/5 discipline
(reproduce / characterization tests, perf benchmarking, behavior preservation, the verifier
gate, the dossier, autocommit) is **preserved** — it moves *into* the implementer loop, with
the parent authoring the dossier from the implementer's report. So there is **one execution
model** across fixes, optimizations, feature suggestions, and ad-hoc plans.

**Invariant change:** the current SKILL.md overview says *"the parent owns all file writes
and never delegates edits."* That is revised: the parent owns all writes to `.recon/` and the
originating branch, but **delegates code edits to `implementer`s confined to isolated
worktrees** — the single, deliberate write-enabled worker, used uniformly by Fix, Optimize,
suggest, and plan execution.

---

## 2. New components

### Agents (new)

| Agent | Model | Role | Writes code? |
|-------|-------|------|--------------|
| `suggester` | Opus | Reads ARCHITECTURE.md + setup context; emits ≥10 well-described feature suggestions. Used by `/recon suggest` only. | No (read-only) |
| `planner` | Opus | Two modes. **Decompose:** task → general plan. **Detail:** items → per-item detailed sub-plans + a file-overlap-aware parallelization map. Items may be selected suggestions, approved workstreams, **or selected ISSUES.md / OPTIMIZATIONS.md findings** (Phase 4/5). Shared by all four entry points. | No (read-only; produces plan markdown returned to parent) |
| `implementer` | Sonnet | One per detailed sub-plan, in its own git worktree. **Multi-mode** — runs the discipline matching the plan's origin: **fix** (reproduce test → fix → suite ≥ baseline → perf), **optimize** (characterization tests → refactor/perf → identical-outcome behavior gate → perf reject criteria), or **feature** (build → tests → suite ≥ baseline). Makes ONE mode-appropriate commit. Shared execution worker. | **Yes** — the single delegated-edit worker |

Existing `verifier` (Sonnet) is reused as the risky-plan gate during execution (see §5d
for its feature-aware role).

#### `implementer` write/safety contract (load-bearing — recon's first write-enabled worker)

recon's standing invariant is that **the parent owns all writes and workers are read-only**.
The implementer is the sole, deliberate exception, and its boundary is fenced as tightly as
the pentester's network contract:

- **Writes are confined to its own worktree path only.** The implementer is given one
  worktree directory and may create/modify/delete files **only within it**. It may **never**
  write to the originating branch's checkout, the main repo checkout, any sibling
  implementer's worktree, or anything under `.recon/`. Editing the originating branch or the
  default branch directly is forbidden.
- **It must stay within its declared file set.** The sub-plan declares the paths the plan
  will touch (§5a). Before cherry-pick, the **parent** runs `git diff --name-only` in the
  worktree and compares against the declared set: any file touched **outside** the declared
  set is an out-of-declaration write — the parent does **not** cherry-pick that plan, marks
  it `blocked: wrote outside declared file set — <paths>`, and pauses for the operator. This
  is what keeps disjoint-batch parallelism (and therefore conflict-free cherry-picks) valid.
- **It makes exactly ONE commit**, inside its worktree, in the feature-commit format
  (§6, conventions.md). No squashing, no amending other commits.
- **Install/build policy:** the implementer **may** run the project's declared install/build
  step if its plan or the test suite needs it (e.g. `npm install` for a new dependency the
  plan adds), confined to its worktree. It may **not** run destructive operations (DB drops,
  data deletion, deploys) or touch anything outside the worktree/project sandbox.
- **It commits only when the baseline gate passes** (§5c); on any failure it reports and
  makes **no commit**.

**Model policy addition** to SKILL.md: `suggester` + `planner` = Opus; `implementer` = Sonnet;
`verifier` = Sonnet (unchanged). Parent orchestrates at the session model.

### Artifacts (new, all under `.recon/`)

```
.recon/development-suggestions/
    suggestions-YYYY-MM-DD-1.md     # batch 1 (≥10 suggestions)
    suggestions-YYYY-MM-DD-2.md     # batch 2 from "another 10", etc.
.recon/plans/<plan-id>/
    general-plan.md                 # /recon plan only: decompose-mode output
    <n>-<slug>.md                   # one detailed sub-plan per item
    parallelization.md              # batch map: which sub-plans run in parallel
```

- Dates are **ISO `YYYY-MM-DD`** (sortable, unambiguous).
- `<plan-id>` = `YYYY-MM-DD-<short-slug>` of the task or suggestion-set. If that id already
  exists (two runs same day, similar slug), append `-2`, `-3`, … so plan dirs never collide.
- Batch suffix `-1`, `-2`, … = each "look for another 10" round is its own file.

---

## 3. `/recon suggest` flow

1. **Ground in project knowledge.** If `.recon/ARCHITECTURE.md` is **missing**, auto-run
   Phase 1 (Setup) + Phase 2 (Map), including the partition gate, to build it. If it
   **exists**, reuse it and offer a quick refresh.
2. **Dispatch `suggester` (Opus)** with: stack/setup summary + the full ARCHITECTURE.md.
   Returns **≥10 suggestions**, each with: title, what it adds, why it fits *this* codebase,
   rough surface area (files/modules likely touched), dependencies/risks, rough effort.
3. **Write** batch to `.recon/development-suggestions/suggestions-<date>-1.md`.
4. **Offer "look for another 10."** If yes, re-dispatch `suggester` told to avoid all prior
   titles; write the new batch to `…-2.md` (then `-3.md`, …). Repeatable.
5. **Selection gate.** Operator picks suggestions (by number, across batches) to take forward.
   Recording is faithful: unselected suggestions remain in their files for later.
6. **Hand selected items → `planner` (detail mode).** (No separate general-plan gate here —
   the selected suggestions are already the items.)

---

## 4. `/recon plan "<task>"` flow

1. **Dispatch `planner` (decompose mode, Opus)** with the operator's task string + (if
   available) project context. It decides the best development approach and writes a
   **general plan** to `.recon/plans/<plan-id>/general-plan.md`: workstreams/milestones,
   chosen stack/approach with rationale, sequencing, and open assumptions.
   (`/recon plan` does **not** require an existing ARCHITECTURE.md — the task may be greenfield.)
2. **General-plan gate.** Operator approves or requests revisions (loop until approved).
3. **Hand approved workstreams → `planner` (detail mode).**

---

## 5. Shared back half: planner detail mode + execution

### 5a. Planner — detail mode (Opus)
For each item (selected suggestion OR approved workstream):
- Write a **detailed sub-plan** `.recon/plans/<plan-id>/<n>-<slug>.md`: goal, files to
  create/modify, ordered implementation steps, tests to add, acceptance criteria, and a
  declared **file set** (paths it expects to touch).

Then produce the **parallelization map** `.recon/plans/<plan-id>/parallelization.md`:
- Group sub-plans with **disjoint declared file sets** into the same **parallel batch**.
- Sub-plans whose file sets overlap an already-placed plan go into a **later sequential batch**.
- Output: ordered list of batches, each listing the sub-plan ids that run concurrently.

### 5b. Plan-review + "implement?" gate
Parent presents the detailed sub-plans + batch map. Operator confirms whether to implement
(and may deselect individual sub-plans). Implementation does not start without this confirmation.

### 5c. Execution — per batch, parallel

**Setup (once):**
- **Originating branch** is fixed and **never the default branch**. If the session is on the
  default branch, the parent creates/checks out a work branch first. This originating branch
  is the cherry-pick target. (If `/recon suggest` auto-ran Map inside a worktree per the
  Phase-1 boost, the originating branch = that worktree's branch; the parent records which
  branch is the target so map-time and execution-time workspaces don't diverge.)
- **Baseline** is the Phase-1 test baseline. Re-capture if missing. If the project has **no
  test command at all** (common for `/recon plan` greenfield), there is no regression baseline
  — see the no-baseline rule below.
- **Concurrency cap** (new — recon has none today): implementers are far heavier than
  read-only mappers (each runs a full suite in a worktree), so parallel implementers are
  capped at **`implConcurrency` (default 10, operator-overridable)**. A batch larger than the
  cap runs in cap-sized waves. (This is the first concurrency cap in recon; it applies only
  to the implementer execution path.)

For each batch, in order:

1. **Dispatch.** For each sub-plan in the batch, dispatch one **`implementer` (Sonnet)** in
   its **own git worktree** branched off the originating branch
   (`superpowers:using-git-worktrees`). Implementers run **in parallel** up to `implConcurrency`.
2. **Implement + baseline gate.** Each implementer implements its sub-plan, then:
   - **If a baseline test suite exists:** run the **full suite**; require it is **≥ the
     Phase-1 baseline** ("nothing broken"). Plus any tests the sub-plan adds must pass.
   - **No-baseline (greenfield / no test command):** the regression gate cannot run. The
     implementer instead requires (a) the project **builds** (if a build step exists) and
     (b) the **new tests the sub-plan adds pass**. It records `regression: baseline
     unavailable — manual review`, and such a plan is **never auto-cherry-picked**: the
     parent pauses for explicit operator approval before cherry-picking it (mirrors Phase-4's
     baseline-unavailable rule).
   - On success → make **ONE commit** (feature-commit format, §6) inside the worktree.
     On failure (gate fails, or plan can't be completed) → report + make **no commit**.
3. **Out-of-declaration check (parent).** Before considering a plan's commit, the parent runs
   `git diff --name-only` in the worktree vs the sub-plan's declared file set. Files touched
   outside the declared set → `blocked: wrote outside declared file set — <paths>`, not
   cherry-picked, pause for operator (see §2 contract).
4. **Verifier gate (risky plans).** For high-risk/large sub-plans, the parent dispatches the
   **`verifier`** (feature-aware role, §5d) on the diff + results. Three verdicts (preserved
   from Phase 4): **APPROVE** → proceed; **APPROVE-WITH-NOTES** → proceed, but record the
   notes in the dossier's Verification section and surface them to the operator;
   **REJECT** → not cherry-picked, `blocked: verifier rejected — <reason>`, pause + surface.
5. **Cherry-pick back.** The parent cherry-picks each surviving worktree commit onto the
   originating branch — **separate commits, no squashing**, in batch order. A cherry-pick
   **conflict** (rare, given disjoint batching + the out-of-declaration check) **pauses for
   the operator**.
6. **Batch-failure → dependency check.** If a sub-plan in this batch failed/blocked, before
   running the next batch the parent checks whether any later sub-plan's declared file set
   **overlaps the failed plan's** (i.e. was sequenced *after* it because it depended on its
   changes). Such dependent sub-plans are **held**, marked `blocked: depends on <failed
   plan>`, and surfaced — they are not dispatched until the operator decides (fix the failed
   plan, re-plan, or skip the dependents). Independent later batches proceed normally.
7. Proceed to the next batch only after the current batch's commits are cherry-picked and the
   dependency check is done. The §5c.6 dependency check keys on **cherry-pick completion**,
   not mere commit existence — so under **`autocommit=no`** the parent must obtain the
   operator's OK and complete the current batch's cherry-picks **before** dispatching a
   later batch that depends on this one's files. (Independent later batches need not wait;
   this is sequencing, not a deadlock.)

**Finish:** report summary (implemented / blocked / held / skipped + commit hashes per
sub-plan), then `superpowers:finishing-a-development-branch` (merge / PR / keep / discard).
Worktrees are cleaned up. Never commit to the default branch.

### 5d. Verifier — feature-aware role
The existing `verifier` checklist is fix/refactor-oriented (root-cause, behavior
preservation). For a **new-feature** diff there is no prior behavior to preserve and no bug
root cause, so the verifier would partly misfire. The implementer dispatch reframes the
verifier's question for features: **"Does this diff meet the sub-plan's stated acceptance
criteria, stay within the declared file set, and not regress the baseline?"** — not "is the
root cause fixed." This is a dispatch-prompt reframing (in `dispatch-prompts.md`), not a new
agent; if testing shows the reframing is insufficient, a feature-aware verifier variant is a
follow-up (out of scope for v1).

### 5e. Implementer modes (one agent, three disciplines)
The implementer is dispatched with a **mode** set by the plan's origin. The mode selects the
discipline inside the worktree; everything else (worktree isolation, declared-file-set,
single commit, cherry-pick) is identical.

| | **fix** (ISSUES.md) | **optimize** (OPTIMIZATIONS.md) | **feature** (suggest / plan) |
|---|---|---|---|
| Root step | root-cause (`superpowers:systematic-debugging`) | assess coverage; pin behavior | (none) |
| Test step | write **failing reproduce test** (TDD) | **characterization tests** if coverage thin | write the feature's tests (TDD) |
| Apply | minimal root-cause change | the catalog refactoring / perf optimization | implement the sub-plan |
| Pass bar | suite **≥ baseline** + new test passes | **identical outcomes** to baseline (stricter; no test may flip) | suite **≥ baseline** + new tests pass |
| Perf | **none** — perf check is optimize-only | `perfRuns` pre/post + **reject criteria** (measurable gain / no >5% regression) | **none** — perf check is optimize-only |
| Commit format | defect-shaped (Phase-4 template) | refactor/perf-shaped | feature-shaped |
| No-baseline rule | as §5c | as §5c (cannot pin behavior → blocked) | as §5c (build + new tests) |

**Perf measurement runs in optimize mode only.** The `perfRuns`, `perfTarget` knobs and the
perf-target cascade apply solely to optimize-mode plans; fix and feature plans do **no**
pre/post benchmarking (their pass bar is purely the test gate). This is a deliberate change
from today's Phase 4, which benchmarks fixes too. The `dossier` knob applies to all modes.

### 5f. Phase 4 (Fix) & Phase 5 (Optimize) re-expressed on the engine
Phases 4 and 5 are **rewritten** to feed their operator-selected findings into the shared
engine instead of the parent applying changes sequentially:

1. **Select** (existing Phase-3 gate): operator picks ISSUES.md findings (→ Phase 4) and/or
   OPTIMIZATIONS.md findings (→ Phase 5).
2. **Plan:** dispatch `planner` (detail mode, Opus) over the selected findings → one detailed
   sub-plan per finding (carrying its mode = fix/optimize, its `file:line`, declared file set,
   and the test/perf requirements from the matrix) + the file-overlap parallelization map.
3. **Plan-review + "implement?" gate** (existing autocommit prompt, reinterpreted below).
4. **Execute** via §5c: parallel `implementer`s (mode=fix or mode=optimize), one per finding,
   in worktrees, capped at `implConcurrency`, in disjoint-file batches.
5. **Dossier (parent-authored):** after each implementer returns, the **parent** writes the
   dossier from the implementer's structured report + the worktree diff — implementers stay
   barred from writing `.recon/`. The dossier is authored **regardless of cherry-pick
   outcome**: a blocked/rejected plan still gets a dossier (recording why it was blocked),
   matching today's inline-dossier-on-block behavior. Fix/optimize dossiers:
   `.recon/fixes/<finding-id>/` (`fix.md` + `change.diff` + `benchmark/`), id schemes
   preserved (Phase-4 = `<n>-<slug>`, Phase-5 = `opt-<n>-<slug>`). Feature (suggest/plan)
   dossiers: `.recon/plans/<plan-id>/<n>-<slug>/` with `summary.md` (not `fix.md` — no defect)
   + `change.diff` + optional `benchmark/`.
6. **Verifier gate + cherry-pick** via §5c (verifier checklist is fix/refactor-oriented for
   these modes; feature-aware only for feature mode). Flip the ISSUES.md/OPTIMIZATIONS.md
   checkbox `- [ ]`→`- [x]` with the dossier pointer + cherry-picked commit hash on success.

**Autocommit knob, reinterpreted for parallel worktrees:** the implementer **always** commits
inside its own worktree (that commit is what gets cherry-picked). The operator's
"Autocommit each fix? (yes/no)" prompt now gates the **cherry-pick back**: *yes* → a plan that
passes its gate + verifier is cherry-picked automatically; *no* → the parent shows the diff +
test/perf results and waits for the operator's OK before cherry-picking. Stop conditions
(verifier REJECT, can't reach green, scope sprawl, behavior changed in optimize mode) →
the worktree commit is **not** cherry-picked, finding marked `blocked: <reason>`, pause.

**What is preserved verbatim from today's Phase 4/5:** the autocommit prompt + `dossier`
knob, the reproduce-test (Fix) / characterization-test + identical-outcome behavior gate
(Optimize), and — **for Optimize only** — the `perfRuns`/`perfTarget` knobs, the perf-target
cascade, and the perf reject criteria; plus the dossier contents, the verifier gate (incl.
APPROVE-WITH-NOTES), the structured commits, the stop conditions, and finish via
`superpowers:finishing-a-development-branch`. **What changes:** (a) the parent no longer
edits — `implementer`s do, in parallel worktrees; (b) ordering shifts from
lowest-risk-sequential to disjoint-file batches (risk can be a within-batch tiebreaker);
(c) the dossier is parent-authored from the implementer report rather than written inline;
(d) **perf benchmarking is now optimize-only** — fix-mode plans no longer run pre/post perf
(a deliberate departure from today's Phase-4 step 2/6 perf measurement).

### 5g. Standalone `/recon fix` and `/recon optimize`
Both phases also get **standalone subcommand entry points** (alongside `test-frontend`,
`pentest`, `suggest`, `plan`), so the operator can jump straight to applying findings without
re-running the full pipeline:

- **`/recon fix`** → seeds the engine with operator-selected **`.recon/ISSUES.md`** findings
  (implementer mode = **fix**).
- **`/recon optimize`** → seeds the engine with operator-selected **`.recon/OPTIMIZATIONS.md`**
  findings (implementer mode = **optimize**).

Each subcommand:
1. **Prerequisite (auto-run Hunt if missing).** If the needed findings file doesn't exist,
   auto-run Phase 1 (Setup) + 2 (Map) + 3 (Hunt), including their gates, to produce it; if it
   exists, reuse it (offer a refresh). (Consistent with `/recon suggest` auto-running Map.)
2. **Selection gate** — operator picks which findings to apply (same gate as the in-pipeline
   Phase-3 hand-off).
3. **Run the engine** exactly as §5f (plan → gates → parallel implementers → dossier →
   verifier → cherry-pick), in the corresponding mode.

These subcommands and the in-pipeline Phases 4/5 are **the same engine run with a different
front door** — no duplicated logic. The in-pipeline path (Hunt → Phase-3 gate → Phase 4/5)
remains; the subcommands are direct entry points for when a findings file already exists or
the operator wants only the apply step.

---

## 6. Where it plugs into the skill

- **`SKILL.md`:**
  - **Four** new subcommand sections (mirroring the `test-frontend` / `pentest` style):
    `suggest`, `plan`, `fix`, `optimize`.
  - A **shared "Execution engine" section** (the §5c/§5e backbone) that Phase 4, Phase 5, and
    all four subcommands reference, so the worktree/batch/cherry-pick logic is stated once.
  - **Rewrite Phase 4 & Phase 5** to feed selected findings into the engine (§5f) instead of
    the parent applying changes sequentially — preserving their disciplines, knobs, and dossier.
    Note that `/recon fix` and `/recon optimize` (§5g) are standalone front doors to the same
    Phase 4/5 engine run.
  - **Revise the overview invariant** line: parent owns `.recon/` + originating-branch writes
    but delegates code edits to worktree-confined `implementer`s.
  - Extend **and revise** the **Model policy** paragraph: add `suggester`/`planner` = Opus and
    `implementer` = Sonnet (the first non-`verifier` Sonnet worker), and **remove the now-false
    sentence** "The parent applies the actual Phase-4/5 fix edits at whatever model the session
    is on" — edits now happen in the Sonnet `implementer`, not the parent.
  - In the shared Execution-engine section, state that `implConcurrency` caps **implementers
    only** — the read-only mapper/hunter fan-out stays uncapped — so a future editor doesn't
    apply the cap to mappers.
- **`~/.claude/agents/`** (mirrored to project `.claude/agents/`): `suggester.md`,
  `planner.md`, `implementer.md`. The `implementer` definition spells out its write boundary,
  worktree isolation, baseline gate, single-commit rule, and default-branch prohibition.
- **`reference/dispatch-prompts.md`:** new dispatch blocks — `## Suggester dispatch`,
  `## Planner dispatch (decompose)`, `## Planner dispatch (detail)`, `## Implementer dispatch`.
- **`reference/artifact-templates.md`:** skeletons for suggestions file, general-plan,
  detailed sub-plan, and parallelization map.
- **`reference/conventions.md`:** a **new feature-commit template** for the implementer
  (feature-shaped — goal / what changed / tests / plan-ref — NOT the defect-shaped Phase-4
  template with Severity/Trigger fields, which don't apply to a new feature); the
  implementer's worktree/declared-file-set/cherry-pick contract; the suggestion entry format;
  the sub-plan entry format (incl. the declared file set).
- **`README.md`:** document the two new subcommands.

---

## 7. Gates (operator checkpoints)

1. Partition gate (only if Map is auto-run by `/recon suggest`/`fix`/`optimize`) — existing.
2. ARCHITECTURE review gate (only if Map is auto-run) — existing.
3. Hunt findings gate (only if Hunt is auto-run by `/recon fix`/`optimize`) — existing.
4. **Suggestion selection gate** (`suggest`) — new.
5. **Findings selection gate** (`fix` / `optimize`, when run standalone) — reuses the
   Phase-3 hand-off gate.
6. **General-plan approval gate** (`plan`) — new.
7. **Plan-review + "implement?" gate** (all engine runs) — new.
8. Execution pauses (all new): cherry-pick conflict; verifier REJECT; out-of-declaration
   write; baseline-unavailable cherry-pick approval; batch-failure dependency hold.
9. Finish gate (`finishing-a-development-branch`) — existing.

---

## 8. Non-goals (YAGNI)

- No auto-execution without the explicit "implement?" gate.
- No squashing / no rewriting history — commits stay separate per sub-plan.
- No cross-batch rolling rebase (rejected in favor of disjoint-file batching).
- `/recon plan` does not require a prior full recon map (may be greenfield).
- No new severity/scoring system — suggestions carry rough effort, not severities.

---

## 9. Superpowers wiring (match existing convention)

recon threads `superpowers:*` skills throughout: every phase ends with a **"Superpowers
boost:"** callout, and most agents carry a **"Recommended:"** line naming the skill that
worker reasons with. The new components MUST follow the same convention.

### New SKILL.md sections — "Superpowers boost" callouts
- **`/recon suggest`:** `superpowers:brainstorming` (the suggester is ideating — frame
  suggestions as a brainstorm grounded in the Map, not a generic feature dump);
  `superpowers:dispatching-parallel-agents` (if Setup+Map is auto-run);
  then hand off to the planner via `superpowers:writing-plans`.
- **`/recon plan`:** `superpowers:brainstorming` (decompose mode — turn the vague task into a
  shaped general plan); `superpowers:writing-plans` (detail mode — the sub-plans are
  implementation plans).
- **`/recon fix` / `/recon optimize`:** `superpowers:dispatching-parallel-agents` (if the Hunt
  is auto-run), then the shared execution chain below — fix adds
  `superpowers:systematic-debugging` (root-cause per finding), optimize adds
  `superpowers:test-driven-development` for characterization tests.
- **Shared execution (the boost chain, mirroring Phase 4):**
  `superpowers:writing-plans` (the detailed sub-plans) →
  `superpowers:dispatching-parallel-agents` (per-batch implementer fan-out) →
  `superpowers:using-git-worktrees` (one isolated worktree per implementer) →
  `superpowers:subagent-driven-development` (the parallel-implementer model IS
  subagent-driven plan execution) / `superpowers:executing-plans` →
  `superpowers:test-driven-development` (implementer writes the feature's tests) →
  `superpowers:verification-before-completion` (the baseline / "nothing broken" gate) →
  `verifier` and/or `superpowers:requesting-code-review` for risky plans →
  `superpowers:finishing-a-development-branch` (merge / PR / keep / discard).

### New agent "Recommended:" lines
- **`suggester.md`:** `superpowers:brainstorming` — generate distinct, codebase-specific
  ideas; avoid generic, on-distribution feature suggestions. You only suggest; you never plan
  or edit.
- **`planner.md`:** `superpowers:writing-plans` (detail mode — produce real implementation
  plans with clear steps/acceptance) and `superpowers:brainstorming` (decompose mode — shape
  the task before planning). You only plan; you never edit.
- **`implementer.md`:** `superpowers:using-git-worktrees` (you operate solely inside your
  assigned worktree), `superpowers:test-driven-development` (write the test first — reproduce
  test in fix mode, characterization tests in optimize mode, feature tests in feature mode),
  `superpowers:systematic-debugging` (fix mode — trace symptom → root cause before changing),
  and `superpowers:verification-before-completion` (prove the gate holds before you commit —
  evidence before claims).

### Dispatch prompts
The new dispatch blocks in `dispatch-prompts.md` name these skills inline where the worker
should act on them (e.g. the implementer block instructs TDD + verification-before-completion;
the planner detail block instructs writing-plans), consistent with how the existing blocks
embed rubric/format.

## 10. Testing (skill-TDD, per `superpowers:writing-skills`)

Because this is a skill edit, the implementation plan MUST include skill-testing, not just
authoring:
- **Baseline (RED):** dispatch a subagent to act as the recon parent given only the *current*
  SKILL.md and the prompt "/recon suggest" / "/recon plan X" — confirm it has no defined
  behavior (the gap this design fills).
- **GREEN:** with the new sections present, dispatch a subagent to follow `/recon suggest`
  and `/recon plan "<task>"` against a sample project and verify it: grounds in Map, produces
  ≥10 suggestions to the right path, honors each gate, seeds the planner correctly, produces a
  disjoint-batch parallelization map, and (in a sandbox) runs implementers in worktrees with
  baseline-gated single commits cherry-picked back.
- **REFACTOR:** close any step the test subagent skipped or misread (e.g. committing to the
  default branch, squashing, skipping the implement gate).
- **Phase 4/5 regression (RED→GREEN):** because this *rewrites* Phase 4/5, baseline a subagent
  on the **current** Phase-4 flow (sequential parent-applied) then verify the rewritten flow
  still honors every preserved discipline — reproduce test (fix), optimize-mode perf pre/post +
  reject criteria, optimize-mode identical-outcome behavior gate, verifier gate (incl.
  APPROVE-WITH-NOTES), dossier authored, autocommit→cherry-pick reinterpretation, never the
  default branch, no squashing — AND that fix-mode no longer benchmarks (the intended change). A discipline dropped in the
  rewrite is a failing test to fix before ship.
