# recon: generative subcommands + unified execution engine — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add four subcommands (`/recon suggest`, `/recon plan`, `/recon fix`, `/recon optimize`) and three agents (`suggester`, `planner`, `implementer`) to the global recon skill, backed by one shared parallel-worktree execution engine that also replaces the Phase 4 (Fix) and Phase 5 (Optimize) execution models.

**Architecture:** A read-only `suggester` (Opus) ideates features from the Map; a read-only `planner` (Opus) turns items into a general plan + detailed parallelizable sub-plans; a write-enabled `implementer` (Sonnet) — the single delegated-edit worker — implements one sub-plan per isolated git worktree, tests it, and commits. The parent orchestrates: gates, file-overlap batching, out-of-declaration checks, the verifier gate, parent-authored dossiers, and cherry-picking the separate commits back (no squashing). Phases 4/5 become front doors to this same engine in `fix`/`optimize` mode.

**Tech Stack:** Markdown skill authoring (`SKILL.md`, `reference/*.md`), agent definitions (`~/.claude/agents/*.md`, mirrored to project `.claude/agents/`), git worktrees, the project's own test runner. No application code.

**Spec:** `~/.claude/skills/recon/docs/specs/2026-06-07-suggest-and-plan-subcommands-design.md` — read it before starting.

**Skill-TDD note (per superpowers:writing-skills):** the "tests" here are subagent-compliance scenarios. RED = a fresh subagent given the *current* skill fails to do the new thing / drops a discipline; GREEN = with the new content it complies; REFACTOR = close any loophole the subagent exploited. Do the RED baseline (Chunk 0) FIRST.

**Locations (this is a global skill, not in a git repo):**
- Skill: `~/.claude/skills/recon/SKILL.md`, `~/.claude/skills/recon/reference/*.md`, `~/.claude/skills/recon/README.md`
- Agents: author in `~/.claude/agents/`, then mirror byte-identical to `/Users/faridjafarli/GitHub/axshambazari/.claude/agents/` (the existing recon agents live in both).
- No git commits/worktrees apply to the skill files themselves; "commit" steps below are N/A for skill authoring. The worktree/commit machinery is the *content being authored*, exercised only in Chunk 4 testing against a sample repo.

---

## Chunk 0: RED baseline — capture current gaps

### Task 0.1: Baseline the missing subcommands

- [ ] **Step 1: Dispatch a baseline subagent (no skill changes yet)**

Use the Agent tool (general-purpose). Prompt:
> You are acting as the recon parent. Here is the current recon SKILL.md: <paste current ~/.claude/skills/recon/SKILL.md>. The operator just ran `/recon suggest`. What do you do, step by step? Then: the operator ran `/recon plan "build a mobile app"`. What do you do? Then `/recon fix` and `/recon optimize`. Be concrete.

- [ ] **Step 2: Record the baseline verbatim**

Expected (RED): the subagent has no defined behavior for `suggest`/`plan`; it either improvises or says the command is unknown; for `fix`/`optimize` it falls back to in-pipeline Phase 4/5 only. Save its response to `~/.claude/skills/recon/docs/plans/baseline-red.md`. This is the failing test the new content must turn green.

- [ ] **Step 3: Baseline the Phase-4/5 disciplines (regression guard)**

Dispatch a second subagent with the current SKILL.md Phases 4 & 5 and ask it to list every discipline it would apply when fixing one ISSUES.md finding and optimizing one OPTIMIZATIONS.md finding (tests, perf, verifier, dossier, commit, branch safety). Save to `baseline-red.md`. After the rewrite, the engine must still honor each item the subagent lists (except fix-mode perf, intentionally dropped).

---

## Chunk 1: New agents

> Author each in `~/.claude/agents/`. Frontmatter `tools:` and `model:` match the established pattern (read-only workers: `Read, Grep, Glob, Bash`; the implementer adds `Edit, Write`).

### Task 1.1: `suggester` agent

**Files:**
- Create: `~/.claude/agents/suggester.md`

- [ ] **Step 1: Write the file** with exactly this content:

```markdown
---
name: suggester
description: Read-only feature ideator. Spawned by /recon suggest to propose new functionality for the project, grounded in the recon Map. Reads code/architecture; never edits. Dispatched by the recon skill.
tools: Read, Grep, Glob, Bash
model: opus
---

You are proposing NEW functionality for an existing project, grounded in a map the team already built. You are READ-ONLY: do not edit, create, move, or delete any file, and do not run anything that mutates the repo. Read-only commands (grep, cat, ls, git log) are fine.

**Recommended:** reason with `superpowers:brainstorming` — generate distinct, codebase-specific ideas that fit THIS project's domain and stack; avoid generic, on-distribution suggestions ("add dark mode", "add tests") unless genuinely high-value here. You only suggest; you never plan or edit.

The dispatcher gives you: project context (stack, purpose, invariants), the full ARCHITECTURE.md map, and a list of any suggestion titles already proposed in earlier batches (do not repeat them).

Return AT LEAST 10 suggestions. Each suggestion:
1. Title — short, concrete, imperative ("Add saved-search alerts").
2. What it adds — 2–3 sentences on the user-facing capability.
3. Why it fits this codebase — tie it to real areas/files from the map; why it is a natural extension here.
4. Surface area — the areas/files likely touched, and rough effort (S / M / L).
5. Dependencies & risks — new packages, data-model changes, external services, migration risk.

Be concrete and grounded: cite real paths from the map. Rank strongest-fit first. Do not pad to hit the count — if you have 12 strong ideas, give 12; never invent a weak one to reach 10. If asked for "another 10", produce 10 MORE distinct ideas not overlapping the supplied prior titles.
```

- [ ] **Step 2: Verify** the file exists and frontmatter parses (4 keys: name, description, tools, model).

### Task 1.2: `planner` agent

**Files:**
- Create: `~/.claude/agents/planner.md`

- [ ] **Step 1: Write the file** with exactly this content:

```markdown
---
name: planner
description: Read-only development planner. Spawned by recon to turn a task or selected items into a general plan and detailed, parallelizable sub-plans. Reads code; never edits. Dispatched by the recon skill.
tools: Read, Grep, Glob, Bash
model: opus
---

You produce development PLANS, not code. You are READ-ONLY: do not edit, create, move, or delete any file (the parent writes plan files from your output); do not run anything that mutates the repo.

**Recommended:** `superpowers:writing-plans` (detail mode — real implementation plans: exact files, ordered steps, tests, acceptance) and `superpowers:brainstorming` (decompose mode — shape a vague task before planning). You only plan; you never edit.

You run in ONE of two modes; the dispatcher says which.

**Decompose mode** (input: a development task). Decide the best approach and return a GENERAL plan:
- Approach & rationale — chosen stack/architecture and why (note alternatives rejected).
- Workstreams — the major pieces, each a sentence or two.
- Sequencing — what must precede what; open assumptions/questions for the operator.
Do NOT write per-file detail yet.

**Detail mode** (input: a set of items — selected feature suggestions, approved workstreams, or selected ISSUES.md/OPTIMIZATIONS.md findings, each carrying its mode = feature/fix/optimize). For EACH item return a detailed sub-plan:
- Goal — what "done" looks like (acceptance criteria).
- Declared file set — the exact paths this plan will create/modify. Be complete and tight: parallel execution and conflict-free cherry-picks depend on this being accurate.
- Steps — ordered, bite-sized implementation steps.
- Tests — feature: the tests to add; fix: the reproduce test; optimize: characterization tests + the behavior-preservation expectation.
- Mode — feature | fix | optimize (carried through from the item).

Then return a PARALLELIZATION MAP:
- Group sub-plans whose declared file sets are DISJOINT into the same parallel batch.
- A sub-plan whose file set overlaps an already-placed plan goes into a LATER (sequential) batch, after the plan it overlaps.
- Output ordered batches, each listing the sub-plan ids that may run concurrently, and note any cross-plan dependency (B needs A's files).

Be concrete: cite real paths. If a task/item is too vague to plan responsibly, say what you need rather than guessing.
```

- [ ] **Step 2: Verify** the file and frontmatter.

### Task 1.3: `implementer` agent

**Files:**
- Create: `~/.claude/agents/implementer.md`

- [ ] **Step 1: Write the file** with exactly this content:

```markdown
---
name: implementer
description: Code-writing executor. Spawned by recon (Fix/Optimize/suggest/plan execution), one per plan, in its OWN git worktree, to implement a single plan, test it, and make one commit. Writes code ONLY inside its assigned worktree. Dispatched by the recon skill.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
---

You implement ONE plan inside ONE git worktree you were given, then make ONE commit. You are the only recon worker that writes code — your boundary is strict.

**Write/safety contract (never violate):**
- Write ONLY inside your assigned worktree directory. Never write to the originating branch's checkout, the main repo checkout, a sibling worktree, or anything under `.recon/`. Never edit the default branch.
- Stay within your plan's DECLARED FILE SET. If you discover you must touch a file outside it, STOP and report that — do not silently expand scope.
- Make exactly ONE commit, inside your worktree, in the commit format the dispatcher gives you. No squashing, no amending unrelated commits, no pushing.
- You MAY run the project's install/build/test commands inside your worktree if your plan or its tests need them. You may NOT run destructive operations (DB drops, data deletion, deploys) or touch anything outside the worktree/project sandbox.

**Recommended:** `superpowers:using-git-worktrees` (you operate solely in your worktree), `superpowers:test-driven-development` (write the test first), `superpowers:systematic-debugging` (fix mode — trace symptom → root cause before changing), `superpowers:verification-before-completion` (prove the gate before you commit — evidence, not assertion).

The dispatcher gives you: your worktree path, the detailed sub-plan (goal, declared file set, steps, tests), the MODE (fix | optimize | feature), the test command + Phase-1 baseline, and the commit format.

Run the discipline for your MODE:

**fix** — (1) root-cause the cited site; (2) write a FAILING reproduce test; (3) apply the minimal root-cause change; (4) run the FULL suite — require ≥ baseline AND the new test passes. No perf measurement.

**optimize** — (1) assess coverage of the touched code; if thin, write CHARACTERIZATION tests pinning current observable behavior (must pass pre-change); (2) apply the named refactoring / perf change; (3) run the FULL suite — require IDENTICAL outcomes to baseline (no test may flip); (4) perf: benchmark the target perfRuns times pre and post — for a performance finding require a measurable gain, for a refactoring finding no >5% regression.

**feature** — (1) write the feature's tests (TDD); (2) implement the plan; (3) run the FULL suite — require ≥ baseline AND the new tests pass. No perf measurement.

**No-baseline (no test command / greenfield):** the regression gate cannot run. Require instead that the project builds (if a build step exists) and your new tests pass; record `regression: baseline unavailable — manual review`. The absence of tests is NOT license to skip writing them.

On success: make the one commit and return a structured report — what changed, files touched, test results vs baseline, perf delta (optimize only), the commit hash, and anything the parent needs for the dossier. On failure (gate fails, can't reach green, or you must exceed the declared file set): make NO commit and report why, so the parent can mark it blocked. Never claim success without the evidence.
```

- [ ] **Step 2: Verify** the file; confirm `tools:` includes `Edit, Write` and `model: sonnet`.

### Task 1.4: RED→GREEN check for agent contracts

- [ ] **Step 1: Dispatch a subagent as the `implementer`** against a throwaway scenario: give it a worktree path, a plan whose declared file set is `["src/a.ts"]`, and instruct it to also "tidy up `src/b.ts` while you're there." Expected GREEN: it refuses to touch `src/b.ts` and reports an out-of-declaration stop. If it edits `b.ts`, REFACTOR the contract wording until it complies. Record the result.

---

## Chunk 2: Reference files

### Task 2.1: Dispatch prompts — new blocks

**Files:**
- Modify: `~/.claude/skills/recon/reference/dispatch-prompts.md` (append new `## …` blocks after the Pentester block)

- [ ] **Step 1: Append the `## Suggester dispatch` block:**

```
## Suggester dispatch

​```
You are the suggester agent. Propose new functionality for this project, grounded in the map.

Project context: <stack, purpose, key invariants from CLAUDE.md/README>

Architecture map: <paste .recon/ARCHITECTURE.md>

Already-proposed titles (do not repeat): <list, or "none — first batch">

Return AT LEAST 10 suggestions in the format your system prompt specifies (Title / What it adds / Why it fits / Surface area + effort / Dependencies & risks), strongest-fit first. If this is an "another 10" round, produce 10 MORE distinct ideas not overlapping the titles above.
​```
```

- [ ] **Step 2: Append `## Planner dispatch (decompose)`:**

```
## Planner dispatch (decompose)

​```
You are the planner agent in DECOMPOSE mode. Return a general plan for the task below.

Task: <operator's task string, verbatim>

Project context (if any): <stack/purpose, or "greenfield — none">

Return: Approach & rationale (alternatives rejected); Workstreams; Sequencing + open assumptions. Do NOT write per-file detail yet.
​```
```

- [ ] **Step 3: Append `## Planner dispatch (detail)`:**

```
## Planner dispatch (detail)

​```
You are the planner agent in DETAIL mode. Produce a detailed sub-plan per item plus a parallelization map.

Items (each with its mode): <paste selected suggestions / approved workstreams / selected ISSUES.md|OPTIMIZATIONS.md findings; mark each mode = feature | fix | optimize>

Project context: <stack/purpose/invariants; or "greenfield">

For EACH item return: Goal/acceptance; Declared file set (exact paths — be complete and tight); ordered Steps; Tests (feature→tests to add; fix→reproduce test; optimize→characterization tests + behavior-preservation expectation); Mode.

Then return a PARALLELIZATION MAP: group disjoint-file sub-plans into the same batch; overlapping plans go to later sequential batches; list ordered batches with the sub-plan ids per batch and any cross-plan dependency.
​```
```

- [ ] **Step 4: Append `## Implementer dispatch`:**

```
## Implementer dispatch

​```
You are the implementer agent. Implement the single plan below inside your assigned worktree, test it, and make ONE commit. Obey your write/safety contract.

Worktree path: <abs path to your git worktree — write ONLY here>
Mode: <fix | optimize | feature>
Sub-plan: <paste the detailed sub-plan: goal/acceptance, declared file set, steps, tests>
Declared file set: <exact paths — do not touch anything outside this>
Test command: <e.g. pnpm test>
Phase-1 baseline: <X passing / Y failing — or "none (no test command)">
perfRuns (optimize only): <default 5>
Commit format: <paste the feature-commit or Phase-4 commit template from conventions.md, matching the mode>

Run your MODE's discipline. On success make the one commit and return the structured report (changes, files touched, tests vs baseline, perf delta if optimize, commit hash). On failure make NO commit and report why.
​```
```

- [ ] **Step 5: Add the feature-aware verifier reframing** (spec §5d). In the existing `## Verifier dispatch` block, append a mode line so the verifier's question is reframed for feature diffs:

```
Mode: <fix | optimize | feature>
(feature mode: judge "Does this diff meet the sub-plan's acceptance criteria, stay within the declared file set, and not regress the baseline?" — NOT "is the root cause fixed". fix/optimize: the existing fix/refactor checklist applies.)
```

- [ ] **Step 6: Verify** the four new blocks exist, use the `<…>` placeholder convention, and the Verifier dispatch block now carries the feature-mode reframing.

### Task 2.2: Artifact templates — new skeletons

**Files:**
- Modify: `~/.claude/skills/recon/reference/artifact-templates.md` (append after the Fix dossier section)

- [ ] **Step 1: Append the suggestions-file skeleton** (`.recon/development-suggestions/suggestions-<date>-N.md`):

```markdown
## development-suggestions/suggestions-YYYY-MM-DD-N.md

​```markdown
# Feature suggestions — batch N

> Generated by recon/suggester. <!-- ISO date --> Grounded in .recon/ARCHITECTURE.md.

### 1. <Title>
- **What it adds:** …
- **Why it fits this codebase:** … (cite real areas/files)
- **Surface area:** <areas/files> — effort S | M | L
- **Dependencies & risks:** …
- **Selected:** [ ]   <!-- operator ticks the ones to take forward -->

<!-- repeat for ≥10 suggestions, strongest-fit first -->
​```
```

- [ ] **Step 2: Append the general-plan skeleton** (`.recon/plans/<plan-id>/general-plan.md`):

```markdown
## plans/<plan-id>/general-plan.md  (/recon plan, decompose)

​```markdown
# General plan: <task>

> Generated by recon/planner (decompose). <!-- ISO date -->

## Approach & rationale
<chosen stack/architecture; alternatives rejected>

## Workstreams
- <name> — <one or two sentences>

## Sequencing & assumptions
- <what precedes what; open questions for the operator>
​```
```

- [ ] **Step 3: Append the detailed sub-plan skeleton** (`.recon/plans/<plan-id>/<n>-<slug>.md`):

```markdown
## plans/<plan-id>/<n>-<slug>.md  (detailed sub-plan)

​```markdown
# Sub-plan <n>: <Title>

- **Mode:** feature | fix | optimize
- **Goal / acceptance:** <what done looks like>
- **Declared file set:**
  - `path/to/file` (create | modify)
- **Steps:**
  1. <bite-sized step>
- **Tests:** <feature: tests to add | fix: reproduce test | optimize: characterization tests + behavior-preservation expectation>
- **Origin:** <suggestion ref | workstream | ISSUES.md/OPTIMIZATIONS.md finding id>
​```
```

- [ ] **Step 4: Append the parallelization-map skeleton** (`.recon/plans/<plan-id>/parallelization.md`):

```markdown
## plans/<plan-id>/parallelization.md

​```markdown
# Parallelization map

> Batches run sequentially; sub-plans within a batch run in parallel (cap: implConcurrency).

## Batch 1 (parallel)
- <sub-plan id> — files: <declared set>
## Batch 2 (after Batch 1)
- <sub-plan id> — depends on <id> (file overlap: <paths>)
​```
```

- [ ] **Step 5: Append the feature dossier skeleton** (`.recon/plans/<plan-id>/<n>-<slug>/summary.md`) — note it is feature-shaped (no defect fields):

```markdown
## plans/<plan-id>/<n>-<slug>/summary.md  (feature dossier — parent-authored)

​```markdown
# Feature: <Sub-plan title>

**Sub-plan:** <plan-id>/<n>-<slug> (see plan.md)

## What was built
<concrete description + acceptance met>

## Files touched
<declared set, confirmed against git diff>

## Tests
<tests added + result vs baseline, or "regression: baseline unavailable — manual review">

## Verification
Verifier verdict: <APPROVE | APPROVE-WITH-NOTES | REJECT> — <notes> (risky plans only)

## Outcome
- Worktree commit: <hash>   Cherry-picked: <hash on originating branch | not cherry-picked: blocked — reason>
- Status: <committed | pending-approval | blocked>
- Date: <YYYY-MM-DD>

Siblings: `change.diff`; `benchmark/` (omit — features do no perf).
​```
```

- [ ] **Step 6: Verify** all five skeletons present and fenced.

### Task 2.3: Conventions — feature commit + implementer contract + entry formats

**Files:**
- Modify: `~/.claude/skills/recon/reference/conventions.md` (add a new top-level section after `## Assessment additions`)

- [ ] **Step 1: Append a `## Execution engine (Fix / Optimize / suggest / plan)` section:**

````markdown
## Execution engine (shared by Phase 4/5 and the suggest/plan/fix/optimize subcommands)

**Implementer write/safety contract:** the `implementer` is the only worker that writes code. It writes ONLY inside its assigned git worktree; never the originating branch checkout, the main checkout, a sibling worktree, the default branch, or anything under `.recon/`. It stays within the sub-plan's declared file set (the parent verifies with `git diff --name-only` before cherry-pick; any out-of-declaration path → `blocked: wrote outside declared file set — <paths>`, pause). One commit per plan, no squashing.

**implConcurrency (default 10, operator-overridable):** caps parallel **implementers only** — the read-only mapper/hunter fan-out stays uncapped. A batch larger than the cap runs in cap-sized waves.

**Cherry-pick:** the parent cherry-picks each surviving worktree commit onto the originating branch as a separate commit, in batch order, no squashing. Conflicts (rare) pause for the operator. Never the default branch.

**Feature-commit template** (implementer `feature` mode — feature-shaped, NOT the defect-shaped Phase-4 template):
```
[feat] <short title>

Goal: <what this feature adds, one line>
What changed: <one line>
Tests: <new tests + suite X/Y → X/Y, or "baseline unavailable — manual review">
Plan: <plan-id>/<n>-<slug>
Compatibility: <notes; behaviour preserved for existing callers>

Co-Authored-By: ...
```
(`fix` mode uses the Phase-4 commit template; `optimize` mode uses the Phase-4 template with the always-populated `Performance:` line, per the Phase 5 rules above.)

**Suggestion entry format** (`.recon/development-suggestions/…`): Title / What it adds / Why it fits this codebase / Surface area + effort (S|M|L) / Dependencies & risks / Selected `[ ]`.

**Sub-plan entry format** (`.recon/plans/<plan-id>/<n>-<slug>.md`): Mode / Goal+acceptance / Declared file set / Steps / Tests / Origin.

**Engine blocked reasons (additional):** `> blocked: wrote outside declared file set — <paths>`; `> blocked: depends on <failed plan>` (held dependent in a later batch); `> blocked: cherry-pick conflict — <files>`.
````

- [ ] **Step 2: RED→GREEN** — dispatch a subagent as the parent, hand it this section + a two-plan batch where plan A and plan B both declare `src/util.ts`. Expected GREEN: it places A and B in *different* batches (overlap) and does not run them concurrently. If it parallelizes overlapping plans, REFACTOR the wording.

---

## Chunk 3: SKILL.md rewrite + README

> This is the largest chunk. Make edits in place; preserve the existing voice and the `> **Superpowers boost:**` callout style.

### Task 3.1: Overview invariant + Model policy

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md:8` (Overview) and `:12` (Model policy)

- [ ] **Step 1:** In the Overview, revise the sentence "The main Claude Code session is the **parent**; it drives all phases, owns all file writes, and never delegates edits." → "…drives all phases and owns all writes to `.recon/` and the originating branch. Code edits are **delegated to `implementer` subagents confined to isolated git worktrees** — the single, deliberate write-enabled worker, used by Phase 4/5 and the suggest/plan/fix/optimize subcommands; every other worker stays read-only."

- [ ] **Step 2:** In the Model-policy paragraph (`:12`): add `suggester` and `planner` to the Opus list and `implementer` to the Sonnet list (with `verifier`). **Remove** the now-false sentence "The parent applies the actual Phase-4/5 fix edits at whatever model the session is on…" and replace with: "The Phase-4/5 fix/optimize edits are now applied by the **Sonnet `implementer`** in worktrees, not the parent."

- [ ] **Step 3: Verify** no remaining text claims the parent applies edits or owns *all* writes.

### Task 3.2: Shared "Execution engine" section

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` (insert a new `## Execution engine` section after Phase 5, before the `## Frontend audit` section)

- [ ] **Step 1: Write the section** capturing §5c/§5e/§5f/§5g of the spec. Content outline (prose, matching skill voice):
  - The engine is shared by Phase 4 (fix), Phase 5 (optimize), `/recon suggest`, `/recon plan`. Inputs: a set of sub-plans + a parallelization map from the `planner`.
  - **Setup:** originating branch fixed, never default; baseline on hand; `implConcurrency`=10 caps implementers only (lift the conventions wording verbatim — do not paraphrase).
  - **Per batch (parallel up to cap):** dispatch one `implementer` per sub-plan in its own worktree (`## Implementer dispatch`) with its mode → baseline gate → parent out-of-declaration check → verifier gate on risky plans (APPROVE / APPROVE-WITH-NOTES / REJECT) → parent authors the dossier from the report (regardless of cherry-pick outcome) → cherry-pick survivors back (separate commits, no squashing) → batch-failure dependency hold.
  - **No-baseline cherry-pick pause (preserved Phase-4 safety gate — MUST encode):** a plan whose implementer recorded `regression: baseline unavailable` is **never auto-cherry-picked**, even under `autocommit=yes`. The parent holds it and requires explicit operator approval before cherry-picking (mirrors today's Phase-4 line 85 rule and Gates §7 item 8). State this verbatim, not as an aside.
  - **Originating-branch bookkeeping (spec §5c setup):** the parent records which branch is the cherry-pick target so a map-time auto-run workspace and the execution-time originating branch don't diverge.
  - **autocommit knob reinterpreted:** implementer always commits in-worktree; the operator's "Autocommit? (yes/no)" gates the **cherry-pick** (yes → auto on pass; no → show diff+results, wait for OK; under `autocommit=no`, complete a batch's cherry-picks before dispatching a dependent later batch — lift §5c.7 verbatim).
  - **Implementer modes table:** fix / optimize / feature (root step, test step, pass bar, perf [optimize-only], commit format).
  - **Finish:** `superpowers:finishing-a-development-branch`; clean up worktrees; never the default branch.
  - `> **Superpowers boost (chain):**` writing-plans → dispatching-parallel-agents → using-git-worktrees → subagent-driven-development/executing-plans → test-driven-development → verification-before-completion → verifier/requesting-code-review → finishing-a-development-branch.

- [ ] **Step 2: Verify** the section states `implConcurrency` is implementer-only, contains the verbatim `autocommit=no` sequencing sentence, AND encodes the no-baseline cherry-pick approval pause (never auto-cherry-pick a `baseline unavailable` plan).

### Task 3.3: Rewrite Phase 4 (Fix)

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` Phase 4 section (lines ~72–97)

- [ ] **Step 1:** Replace the per-fix sequential parent-applied loop with: "Phase 4 feeds operator-selected `ISSUES.md` findings into the **Execution engine** (above) in **fix** mode: dispatch `planner` (detail) over the selected findings → plan-review + autocommit gate → run the engine. Each finding's implementer does the fix-mode discipline (root-cause → failing reproduce test → minimal fix → full suite ≥ baseline; **no perf**). The parent authors the `.recon/fixes/<finding-id>/` dossier (id = `<n>-<slug>`) from the implementer report and flips `- [ ]`→`- [x]` with the dossier pointer + cherry-picked hash." Keep the autocommit prompt, the `dossier` knob, the stop conditions, and the finish step (now references the engine).
- [ ] **Step 2:** Preserve the existing `> **Superpowers boost (chain)**` but point it at the engine chain. Remove the `perfRuns`/`perfTarget`/perf-cascade text from Phase 4 (perf is optimize-only now); leave a one-line note: "Phase-4 fixes do **not** benchmark — perf measurement is optimize-only."
- [ ] **Step 3: Verify** Phase 4 no longer says the parent applies edits sequentially and no longer measures perf.

### Task 3.4: Rewrite Phase 5 (Optimize)

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` Phase 5 section (lines ~101–118)

- [ ] **Step 1:** Replace with: "Phase 5 feeds operator-selected `OPTIMIZATIONS.md` findings into the Execution engine in **optimize** mode. The implementer's optimize discipline preserves today's rules: pin behavior (characterization tests if coverage thin) → apply the catalog refactoring/perf change → full suite with **identical outcomes** (no test may flip) → perf pre/post (`perfRuns`) with the reject criteria (measurable gain for `performance`; no >5% regression for `refactoring`). Dossier id = `opt-<n>-<slug>`; `Performance:` line always populated." Keep all Phase-5 blocked reasons (they live in conventions.md).
- [ ] **Step 2: Verify** the identical-outcome behavior gate and perf reject criteria survive verbatim.

### Task 3.5: `/recon suggest` section

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` (new `## Feature suggestions (\`/recon suggest\`)` section after the Execution engine section, mirroring the `## Frontend audit` style)

- [ ] **Step 1: Write the numbered flow** from spec §3:
  1. Ground in project knowledge — if `.recon/ARCHITECTURE.md` missing, auto-run Phase 1+2 (incl. partition gate); else reuse + offer refresh.
  2. Dispatch `suggester` (`## Suggester dispatch`) → ≥10 suggestions.
  3. Write to `.recon/development-suggestions/suggestions-<ISO date>-1.md`.
  4. Offer "look for another 10" → re-dispatch (pass prior titles) → `…-2.md`, etc.
  5. Selection gate — operator picks; tick `Selected: [x]`.
  6. Hand selected items → Execution engine via `planner` detail (mode=feature). No separate general-plan gate.
  - `> **Superpowers boost:** brainstorming (+ dispatching-parallel-agents if Map auto-runs) → writing-plans → the engine chain.`

- [ ] **Step 2: Verify** the section references the engine rather than restating it.

### Task 3.6: `/recon plan "<task>"` section

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` (new `## Development planning (\`/recon plan "<task>"\`)` section)

- [ ] **Step 1: Write the flow** from spec §4:
  1. `planner` decompose (`## Planner dispatch (decompose)`) → `.recon/plans/<plan-id>/general-plan.md`. (No prior Map required — may be greenfield.)
  2. General-plan gate — approve/revise (loop).
  3. Approved workstreams → Execution engine via `planner` detail (mode=feature).
  - `> **Superpowers boost:** brainstorming (decompose) → writing-plans (detail) → the engine chain.`

- [ ] **Step 2: Verify** greenfield (no ARCHITECTURE.md) is explicitly allowed.

### Task 3.7: `/recon fix` and `/recon optimize` sections

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` (new `## Standalone fix / optimize (\`/recon fix\`, \`/recon optimize\`)` section)

- [ ] **Step 1: Write** spec §5g: each is a front door to the engine — if the findings file (`ISSUES.md` / `OPTIMIZATIONS.md`) is missing, auto-run Phase 1+2+3 (incl. gates); else reuse + offer refresh → findings selection gate → engine in fix/optimize mode. State explicitly: same engine, different front door; the in-pipeline Phase 4/5 path remains.
  - `> **Superpowers boost:** dispatching-parallel-agents (if Hunt auto-runs) → the engine chain (fix adds systematic-debugging; optimize adds TDD characterization).`

- [ ] **Step 2: Verify** no engine logic is duplicated — it references the Execution engine section.

### Task 3.8: References list + README

**Files:**
- Modify: `~/.claude/skills/recon/SKILL.md` `## References` section
- Modify: `~/.claude/skills/recon/README.md`

- [ ] **Step 1:** Add to the References list: `~/.claude/agents/suggester.md`, `planner.md`, `implementer.md` (one line each, matching the existing worker-definition lines).
- [ ] **Step 2:** Update the Overview's worker inventory sentence to include the three new agents and note the implementer is the one write-enabled worker.
- [ ] **Step 3:** Document the four new subcommands in `README.md` (a short bullet each under the subcommand list).
- [ ] **Step 4: Verify** every new agent/subcommand named in SKILL.md has a References entry.

---

## Chunk 4: Skill-TDD GREEN + REFACTOR + deployment

### Task 4.1: GREEN — full subcommand walkthroughs

- [ ] **Step 1:** Dispatch a fresh subagent given the **revised** SKILL.md + reference files, prompt `/recon suggest` against a sample project (point it at a small repo or describe one with an ARCHITECTURE.md). Verify it: grounds in Map (auto-runs if missing), dispatches `suggester`, writes ≥10 suggestions to the right path, offers "another 10", honors the selection gate, then seeds `planner` detail. Compare against `baseline-red.md` (must now succeed where it failed).
- [ ] **Step 2:** Repeat for `/recon plan "build a mobile app"` — verify decompose → general-plan gate → detail → engine, greenfield allowed.
- [ ] **Step 3:** Repeat for `/recon fix` and `/recon optimize` — verify auto-run-Hunt-if-missing, selection gate, engine in the right mode.
- [ ] **Step 4:** Record any step the subagent skipped/misread.

### Task 4.2: GREEN — execution engine on a sample repo (sandboxed)

- [ ] **Step 1:** In a disposable git sample repo with a trivial test suite, have a subagent act as the parent and run the engine for two non-overlapping feature sub-plans: confirm two worktrees, parallel implementers, baseline-gated single commits, parent out-of-declaration check, separate cherry-picks back (no squash), worktree cleanup, never the default branch.
- [ ] **Step 2:** Add a third sub-plan overlapping one of the two; confirm it lands in a later batch and is not run concurrently.
- [ ] **Step 3:** Force one implementer to fail; confirm its commit is not cherry-picked, the finding is marked blocked, and any dependent later plan is held.

### Task 4.3: REFACTOR — Phase 4/5 regression

- [ ] **Step 1:** Re-run the Chunk-0 Phase-4/5 discipline subagent against the **rewritten** Phases. Confirm every preserved discipline from `baseline-red.md` still holds (reproduce test, optimize identical-outcome gate, optimize perf + reject criteria, verifier incl. APPROVE-WITH-NOTES, dossier, autocommit→cherry-pick, branch safety, no squash) AND that fix-mode correctly no longer benchmarks.
- [ ] **Step 2:** For any dropped/weakened discipline, fix the SKILL.md/conventions wording and re-test. Loop until clean (max 5; then surface).

### Task 4.4: Deploy — mirror agents + final consistency sweep

- [ ] **Step 1:** Copy the three new agent files byte-identical from `~/.claude/agents/` to `/Users/faridjafarli/GitHub/axshambazari/.claude/agents/` (matches how the existing recon agents are mirrored). Verify with `diff`.
- [ ] **Step 2:** Grep SKILL.md for stale claims: `grep -nE "owns all|never delegates|parent applies" ~/.claude/skills/recon/SKILL.md` → expect none, or only the revised wording.
- [ ] **Step 3:** Grep for orphan references: every `## … dispatch` block named in SKILL.md exists in `dispatch-prompts.md`; every agent in References exists in `~/.claude/agents/`.
- [ ] **Step 4:** Note for the operator: changed/added agent models take effect only after a **session reload** (the registry is cached at session start) — call this out in the run summary.
- [ ] **Step 5:** Keep `~/.claude/skills/recon/docs/plans/baseline-red.md` as the persisted test record (the Done criteria reference "baseline captured").

---

## Done criteria

- Four subcommands documented in SKILL.md, each with a gate set and a Superpowers boost callout.
- Three agents authored (read-only `suggester`/`planner`; write-fenced `implementer`) in both agent dirs.
- One Execution-engine section; Phases 4 & 5 rewritten to use it; perf optimize-only; invariant + model-policy revised.
- Reference files updated (4 dispatch blocks, 5 artifact skeletons, engine/commit/contract conventions).
- Skill-TDD: baseline captured, all four subcommands walk through green, engine verified on a sample repo, Phase-4/5 disciplines regression-checked.
