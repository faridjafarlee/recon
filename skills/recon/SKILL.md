---
name: recon
description: Multi-phase codebase recon — map the codebase, then hunt for evidenced issues using read-only worker subagents, with a refactoring catalog as one specialized worker. Use when asked to understand, audit, do a refactoring/tech-debt sweep, or build a mental model of a codebase. Installed as a plugin; trigger /recon:recon (subcommands: /recon:recon test-frontend | pentest | suggest | plan "<task>" | fix | optimize | redesign).
---

## Overview

The main Claude Code session is the **parent**; it drives all phases and owns all writes to `.recon/` and the originating branch. Code edits are **delegated to write-enabled subagents confined to isolated git worktrees** — `implementer` (Phase 4/5 + the suggest/plan/fix/optimize subcommands) and `designer` (the `/recon redesign` design track); every other worker stays read-only. In Phases 2 and 3, the parent dispatches read-only worker subagents — one per area — via the Agent tool; workers return structured notes only. Prompts to spawn each worker type live in `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/dispatch-prompts.md`; scoring and finding format live in `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/conventions.md`; output file skeletons live in `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/artifact-templates.md`. The twelve worker definitions are at `${CLAUDE_PLUGIN_ROOT}/agents/mapper.md`, `${CLAUDE_PLUGIN_ROOT}/agents/hunter.md`, `${CLAUDE_PLUGIN_ROOT}/agents/refactoring-auditor.md`, `${CLAUDE_PLUGIN_ROOT}/agents/verifier.md`, `${CLAUDE_PLUGIN_ROOT}/agents/threat-scout.md`, `${CLAUDE_PLUGIN_ROOT}/agents/frontend-auditor.md`, `${CLAUDE_PLUGIN_ROOT}/agents/pentester.md`, `${CLAUDE_PLUGIN_ROOT}/agents/suggester.md`, `${CLAUDE_PLUGIN_ROOT}/agents/planner.md`, `${CLAUDE_PLUGIN_ROOT}/agents/implementer.md`, `${CLAUDE_PLUGIN_ROOT}/agents/design-reviewer.md`, and `${CLAUDE_PLUGIN_ROOT}/agents/designer.md`; see the References section for all twelve. (The first four run in Phases 2–4 and `threat-scout` in Phase 3 — all read-only; `frontend-auditor` and `pentester` are subcommand workers run via `/recon test-frontend` and `/recon pentest`, and `pentester` is the one worker that sends live attack traffic — read-only on source, active on the network (gated, non-destructive). `suggester` and `planner` are read-only Opus workers for the generative subcommands; `implementer` is a write-enabled worker — Sonnet, confined to its own git worktree — used by the shared execution engine. `design-reviewer` (read-only Opus) and `designer` (Sonnet, worktree-confined — the second write-enabled worker) power the `/recon redesign` design track.)

This system pairs with the Superpowers workflow end to end; each phase below names the relevant `superpowers:*` skills to reach for. Phase 4 (Fix) applies `ISSUES.md` defects; Phase 5 (Optimize) applies `OPTIMIZATIONS.md` refactorings/performance with behavior preservation — both run through the shared Execution engine (applied by worktree-confined `implementer`s), gated, and never on the default branch. Three extra workers (all read-only on your source) support assessments: `threat-scout` (a targeted, app-type threat-model hunt in Phase 3), `frontend-auditor` (a Playwright browser audit via `/recon test-frontend`), and `pentester` (an active, gated, non-destructive API red-team via `/recon pentest`).

**Model policy.** The discovery/diagnosis/planning workers that produce documents — `mapper`, `hunter`, `refactoring-auditor`, `threat-scout`, `frontend-auditor`, `pentester`, `suggester`, `planner`, `design-reviewer` — run on **Opus** (set in each agent's `model:` frontmatter); the `verifier`, the `implementer`, and the `designer` run on **Sonnet**. The Phase-4/5 fix/optimize edits are applied by the **Sonnet `implementer`** in isolated worktrees, not the parent. A per-dispatch `model` override still wins over the frontmatter; changed agent models take effect only after a **session reload** (the registry is cached at session start).

**Plugin dispatch.** This skill ships as the `recon` plugin, so its workers register under the plugin namespace. Dispatch each by its **namespaced subagent type `recon:<name>`** — `recon:mapper`, `recon:hunter`, `recon:refactoring-auditor`, `recon:verifier`, `recon:threat-scout`, `recon:frontend-auditor`, `recon:pentester`, `recon:suggester`, `recon:planner`, `recon:implementer`, `recon:design-reviewer`, `recon:designer`. Wherever the phases below name a worker by bare name (e.g. "subagent type `mapper`"), use the namespaced form (`recon:mapper`). The agents' own `name:` frontmatter stays bare — the `recon:` prefix is added by the plugin loader.

---

## Phase 1 — Setup

1. **Read context.** Read `CLAUDE.md` and `README` (or equivalent). Note: stack, package manager, test command, and any stated invariants.

2. **Capture test baseline.** Run the test suite (e.g. `pnpm test`, `npm test`). Record: test command + passing / failing counts. This is the baseline for Phase 4 comparison. This step is **best-effort and non-blocking**: if no test command exists, the suite fails, or it times out, record that fact (e.g. `baseline: no test command found` or `baseline: suite failed — <error>`) and continue — it does not block Map or Hunt.

3. **Partition into areas.**

   **Domain-graph seeding (preferred path — use when `.understand-anything/domain-graph.json` exists):**
   - Load the JSON. The graph has nodes of type `domain`, `flow`, `step`, and `domainMeta`.
   - `filePath` and `lineRange` exist **only on `step` nodes** — `flow` and `domain` nodes carry no paths.
   - For each `domain` node: follow its `contains_flow` edges → `flow` nodes; from each `flow` node follow `flow_step` edges → `step` nodes; collect each step's `filePath`.
   - Group collected paths under their parent domain name. That group becomes one **area**.
   - `domainMeta` nodes hold entities and business rules — use them to annotate area descriptions, not to infer paths.

   **Structural fallback (use when the domain-graph file is absent):**
   - Split by top-level packages or `src/` subtrees (e.g. each NestJS module folder, each React feature folder).

4. **Partition gate.** Present the proposed area list (name + representative paths) to the operator. Let them add, remove, or rename areas before any worker spawns. Do not proceed to Phase 2 until the operator confirms.

> **Superpowers boost:** if you expect to fix findings later, create an isolated workspace now with `superpowers:using-git-worktrees` so Map/Hunt and later fixes don't disturb the main checkout.

---

## Phase 2 — Map

1. **Dispatch.** For each confirmed area, spawn one `mapper` subagent via the Agent tool in **parallel**. Call the Agent tool with the subagent type set to `mapper` (matching `${CLAUDE_PLUGIN_ROOT}/agents/mapper.md`) and pass — as the task prompt — the `## Mapper dispatch` block from `reference/dispatch-prompts.md` with all `<…>` placeholders replaced: project context (stack summary, key invariants), the area path, and the list of sibling areas being mapped concurrently.

2. **Record.** As mapper results return, write each area's notes into the matching `## <Area>` section of `.recon/ARCHITECTURE.md` (skeleton from `reference/artifact-templates.md`). Write areas **sequentially** as they arrive — never write two areas simultaneously to avoid a race. An area with no meaningful files is noted: `empty / skipped`.

3. **Gate.** Present `.recon/ARCHITECTURE.md` to the operator for review and correction before starting the hunt.

> **Superpowers boost:** apply `superpowers:dispatching-parallel-agents` discipline when fanning out one mapper per area.

---

## Phase 3 — Hunt

1. **Dispatch.** Spawn one `hunter` per area **plus** one `refactoring-auditor` targeting the whole repo (or a narrower path if the operator specifies). Call the Agent tool with the subagent type set to `hunter` (matching `${CLAUDE_PLUGIN_ROOT}/agents/hunter.md`) for each area, and with the subagent type set to `refactoring-auditor` (matching `${CLAUDE_PLUGIN_ROOT}/agents/refactoring-auditor.md`) for the repo-wide pass; pass the corresponding filled dispatch block as the task prompt. The `## Hunter dispatch` and `## Refactoring-auditor dispatch` blocks in `reference/dispatch-prompts.md` already embed the rubric and finding format — copy them verbatim and fill only the `<…>` placeholders. All dispatches may run in parallel. Also dispatch `threat-scout` **once, repo-wide**, in parallel (subagent type `threat-scout`, using the `## Threat-scout dispatch` block from `reference/dispatch-prompts.md`) — a targeted, app-type threat-model pass that complements the per-area scan.

2. **Parent verification — mandatory.** For **each candidate finding** returned by any worker, the parent **independently re-reads** the cited `file:line` before recording:
   - **Confirmed** → route by **originating worker, not category**: hunter findings always go to `.recon/ISSUES.md`; refactoring-auditor findings always go to `.recon/OPTIMIZATIONS.md` — regardless of the category label (e.g. a `performance` finding from the hunter still goes to ISSUES.md). threat-scout findings route by category — security/auth/data/reliability/performance/test/tooling defects go to `.recon/ISSUES.md`; pure refactoring smells go to `.recon/OPTIMIZATIONS.md` (in practice almost all threat-scout findings are defects → `ISSUES.md`). Use the finding entry format from `reference/conventions.md`, with `- [ ]` status.
   - **Rejected** → one line under the artifact's `## Dismissed leads` section: `- <Title> — <reason>`.
   - Worker-suggested severity and confidence are **suggestions**; the parent sets final values per the severity rubric in `reference/conventions.md`.
   - Tag each recorded finding's `Source:` — `[scan]` for a hunter finding, `[targeted]` for a threat-scout finding, `[scan+targeted]` if both found it (dedup key = `file:line`); see `reference/conventions.md`.

3. **Silence is reported.** An area, the refactoring-auditor, or threat-scout yielding zero candidates is not silently skipped: record it as a `### <Area> — no findings` sub-heading under the `## Findings` section of the relevant artifact (ISSUES.md for hunter areas; OPTIMIZATIONS.md for the refactoring-auditor; ISSUES.md for threat-scout — a clean threat-scout pass is meaningful diagnostic signal that the app's highest-risk classes were checked).

**Diagnostic notes:** `.recon/ISSUES.md` / `OPTIMIZATIONS.md` are the running diagnostic deliverable — record every confirmed finding in full whether or not it will be fixed; never silently drop one.

4. **Gate.** Present `.recon/ISSUES.md` and `.recon/OPTIMIZATIONS.md` to the operator. The operator selects what to fix. Phase 4 is not started until explicitly confirmed.

> **Superpowers boost:** use `superpowers:dispatching-parallel-agents` for the per-area hunter fan-out; when you independently confirm a candidate defect, reason with `superpowers:systematic-debugging` (trace symptom → root cause) before recording it as confirmed.

---

## Phase 4 — Fix

Phase 4 applies operator-selected **`.recon/ISSUES.md`** defects via the shared **Execution engine** (below), in `implementer` **fix** mode. Runs only after Phase 3, only on explicit operator confirmation. Scope: defects only (refactorings from OPTIMIZATIONS.md belong to Phase 5). The selection comes from the Phase-3 gate; if unclear, ask the operator which ISSUES.md findings to fix.

1. **Plan.** Dispatch `planner` (detail mode, `## Planner dispatch (detail)`) over the selected findings → one detailed sub-plan per finding (mode = fix, carrying its `file:line`, declared file set, and reproduce-test requirement) + the parallelization map.
2. **Gate.** Present the sub-plans + batch map and ask **"Autocommit each fix? (yes/no)"** (knob: `dossier` = commit (default)|gitignore — no perf knobs; fixes don't benchmark). Run only on confirmation.
3. **Execute** via the Execution engine: parallel `implementer`s (fix mode), one per finding, in isolated worktrees. Each does the fix discipline — root-cause (`superpowers:systematic-debugging`) → failing reproduce test → minimal fix → full suite ≥ Phase-1 baseline (untestable → `no automated test: <why>` + manual step). The parent authors the `.recon/fixes/<finding-id>/` dossier (id = `<n>-<slug>`) from the implementer report, runs the verifier gate on risky fixes, and cherry-picks the commit back; on success flips the finding `- [ ]` → `- [x]` with the dossier pointer + cherry-picked hash. Blocked findings keep `- [ ]` + a `> blocked: <reason>` note.

**Phase-4 fixes do NOT benchmark** — perf measurement is optimize-only (Phase 5).

> **Superpowers boost (chain):** see the Execution engine section — `superpowers:writing-plans` → `superpowers:dispatching-parallel-agents` → `superpowers:using-git-worktrees` → `superpowers:systematic-debugging` (root-cause per fix) → `superpowers:test-driven-development` → `superpowers:verification-before-completion` → `verifier` / `superpowers:requesting-code-review` for risky fixes → `superpowers:finishing-a-development-branch`.

---

## Phase 5 — Optimize

Phase 5 applies operator-selected **`.recon/OPTIMIZATIONS.md`** findings (`refactoring` + `performance`) via the shared **Execution engine** (below), in `implementer` **optimize** mode, with a **behavior-preserving** discipline. Runs after Phase 3; independent of Phase 4. Selection comes from the Phase-3 gate; if unclear, ask which OPTIMIZATIONS.md findings to optimize.

Same flow as Phase 4 (plan → autocommit gate → engine); the autocommit gate adds the optimize knobs `perfRuns` (default 5) and `perfTarget` = auto|suite|bench|off. The optimize-mode discipline each `implementer` runs:
- **Pin behavior** (instead of reproducing a bug): coverage is *adequate* if ≥1 existing test would fail were the changed function's observable behavior to change; else *thin*. If adequate, the existing suite is the behavior baseline; if thin/absent, write **characterization tests** (`superpowers:test-driven-development`) that pass on the pre-change code and join the baseline. If behavior can't be pinned → `> blocked: cannot pin behavior baseline`.
- **Apply** the catalog refactoring named in `Proposed fix` (or the perf optimization for a `performance` finding); minimal, no unrelated edits.
- **Behavior check (stricter than Phase 4):** full suite + characterization tests must show **identical outcomes** — no test may flip. Any change → `> blocked: behavior changed — <which test flipped>`.
- **Perf (optimize-only):** benchmark pre/post (≥`perfRuns`) via the perf-target cascade; reject criteria — `performance` finding must show a **measurable gain** (else `> blocked: no measurable gain`); `refactoring` finding must not regress perf **>5%** (else `> blocked: perf regression (>5%)`). `perfTarget=off` waives the perf-based criteria (`Performance: n/a — disabled by operator`) but the behavior gate still applies.
- **Dossier id** = `opt-<n>-<slug>` (never collides with Phase-4 under `.recon/fixes/`); its `Test` line records the characterization tests added or `existing coverage adequate`; the `Performance:` line is **always populated**.

> **Superpowers boost:** same engine chain as Phase 4, but `superpowers:test-driven-development` writes the **characterization tests** that pin behavior first; behavior preservation (identical test outcomes) is the success bar instead of a bug reproduction.

---

## Execution engine (Fix / Optimize / suggest / plan)

The shared engine that applies changes. Inputs: a set of detailed sub-plans + a parallelization map from the `planner`. The parent orchestrates; the **`implementer` (Sonnet)** is the only worker that writes code — one per sub-plan, each in its own git worktree. Used by Phase 4 (fix mode), Phase 5 (optimize mode), and `/recon suggest` / `/recon plan` (feature mode). Full contract: `reference/conventions.md` → "Execution engine".

**Setup (once per run):**
- Fix the **originating branch** (the cherry-pick target) and NEVER use the default branch — if the session is on it, create/checkout a work branch first. Record which branch is the target so a map-time auto-run workspace and the execution-time originating branch don't diverge.
- Ensure the **Phase-1 baseline** test state is on hand (re-capture if missing). If the project has no test command at all (e.g. greenfield `/recon plan`), see the no-baseline rule below.
- **`implConcurrency` (default 10, operator-overridable)** caps parallel **implementers only** — the read-only mapper/hunter fan-out stays uncapped. A batch larger than the cap runs in cap-sized waves.

**Per batch** (batches run in order; sub-plans within a batch run in parallel up to `implConcurrency`):
1. **Dispatch** one `implementer` per sub-plan (`## Implementer dispatch`) in its own worktree (`superpowers:using-git-worktrees`), with its mode (fix | optimize | feature), test command, baseline, and the matching commit template.
2. **Baseline gate** (inside the implementer): full suite ≥ baseline (fix/feature) or identical outcomes (optimize). **No-baseline:** if there is no test command, require the project builds + new tests pass and record `regression: baseline unavailable — manual review`.
3. **Out-of-declaration check (parent):** `git diff --name-only` in the worktree vs the sub-plan's declared file set; any outside path → `> blocked: wrote outside declared file set — <paths>`, pause.
4. **Verifier gate (risky plans):** dispatch the `verifier` (feature-aware for feature mode — see its dispatch block) on the diff + results. APPROVE → proceed; APPROVE-WITH-NOTES → proceed, record notes in the dossier + surface; REJECT → not cherry-picked, `> blocked: verifier rejected — <reason>`, pause.
5. **Dossier (parent-authored):** write the dossier from the implementer's report + worktree diff — **regardless of cherry-pick outcome** (a blocked plan still gets one). Implementers never write `.recon/`. Fix/optimize → `.recon/fixes/<id>/` (`fix.md`, ids `<n>-<slug>` / `opt-<n>-<slug>`); feature → `.recon/plans/<plan-id>/<n>-<slug>/summary.md`.
6. **No-baseline cherry-pick pause:** a plan that recorded `regression: baseline unavailable` is **never auto-cherry-picked**, even under `autocommit=yes` — hold it for explicit operator approval.
7. **Cherry-pick back:** the parent cherry-picks each surviving worktree commit onto the originating branch as a **separate commit, no squashing**, in batch order. A conflict (rare) pauses for the operator. Under `autocommit=no`, show the diff + results and get the operator's OK first; complete a batch's cherry-picks before dispatching a later batch that depends on it.
8. **Batch-failure dependency hold:** if a sub-plan failed/blocked, any later sub-plan whose declared file set overlaps it (sequenced after it) is **held** — `> blocked: depends on <failed plan>` — and surfaced; independent later batches proceed.

**autocommit knob:** the implementer ALWAYS commits inside its own worktree (that commit is what gets cherry-picked); the operator's "Autocommit? (yes/no)" gates the **cherry-pick** — *yes* = auto on gate+verifier pass; *no* = show diff + results, wait for OK.

**Builder modes:**

| | **fix** | **optimize** | **feature** | **design** |
|---|---|---|---|---|
| Builder | implementer | implementer | implementer | **designer** |
| Test | failing reproduce test | characterization tests (if thin) | feature tests (TDD) | static anti-pattern checks + visual-regression pins |
| Pass bar | suite ≥ baseline + new test | identical outcomes (no flip) | suite ≥ baseline + new tests | no new anti-pattern + visual diff approved + suite ≥ baseline |
| Perf | none (optimize-only) | `perfRuns` pre/post + reject criteria | none (optimize-only) | none |
| Commit | Phase-4 template | Phase-4 template (Performance always populated) | feature template | `[design]` template |

**`design` mode (the `/recon redesign` track).** The engine dispatches **`designer`** (`recon:designer`) instead of `implementer`, caps parallelism at **`designConcurrency`** (default 3 — design builds are heavier: each designer runs a dev server + headless browser in its worktree, on a distinct port = `BASE_PORT + in-wave slot index`, where `BASE_PORT` is chosen clear of the ground-phase app's port — e.g. app-port + 100), and captures a **shared visual baseline** against the originating branch before the first wave (`.recon/design/baseline/` or the seeded Vizzly baseline), re-captured between waves. Everything else (worktree isolation, out-of-declaration check, cherry-pick separate commits / no squash, never the default branch) is identical — **except** the no-baseline pause: in `design` mode the operator's visual approve/reject IS the human gate, so the generic "never auto-cherry-pick a `baseline unavailable` plan" pause does **not** also apply (no redundant double gate). The design test gate is detailed in the `## Design improvement` section.

**Perf-target cascade (optimize only):** (1) benchmarkable hot path → fix-specific micro-benchmark; (2) else a project bench command → run it; (3) else full-suite wall-clock; (4) else `performance: n/a — <why>`. `perfTarget`: auto=cascade (default), suite=case 3, bench=case 2, off=skip + `n/a — disabled by operator`.

**Stop conditions (pause in BOTH autocommit modes):** verifier REJECT; can't reach green / plan not found (if the reproduce test itself is wrong, discard + restart); scope sprawl (`> blocked: needs broader change`); out-of-declaration write; cherry-pick conflict; baseline-unavailable (approval required).

**Finish:** report a summary (implemented / blocked / held / skipped + commit hashes), then `superpowers:finishing-a-development-branch` (merge / PR / keep / discard). Clean up worktrees. Never commit to the default branch.

> **Superpowers boost (chain):** `superpowers:writing-plans` → `superpowers:dispatching-parallel-agents` → `superpowers:using-git-worktrees` → `superpowers:subagent-driven-development` / `superpowers:executing-plans` → `superpowers:test-driven-development` → `superpowers:verification-before-completion` → `verifier` / `superpowers:requesting-code-review` for risky plans → `superpowers:finishing-a-development-branch`.

---

## Feature suggestions (`/recon suggest`)

Invoking recon with `suggest` proposes NEW functionality for the project, grounded in the recon Map, via the read-only `suggester` (Opus). It never edits; selected suggestions flow into the Execution engine.

1. **Ground in project knowledge.** If `.recon/ARCHITECTURE.md` is missing, auto-run Phase 1 (Setup) + Phase 2 (Map) — including the partition gate — to build it; if present, reuse it (offer a refresh).
2. **Dispatch `suggester`** (`## Suggester dispatch`) with the stack/setup summary + the full ARCHITECTURE.md → **≥10 suggestions**, each with what it adds, why it fits this codebase, surface area + effort, and dependencies/risks.
3. **Write** the batch to `.recon/development-suggestions/suggestions-<ISO date>-1.md`.
4. **Offer "look for another 10."** If yes, re-dispatch `suggester` (passing all prior titles to avoid repeats) → `…-2.md`, `…-3.md`, … Repeatable.
5. **Selection gate.** The operator ticks the suggestions to take forward (`Selected: [x]`); unselected ones stay in their files for later.
6. **Plan + execute.** Hand the selected suggestions (mode = feature) to the `planner` (detail mode) → detailed sub-plans + parallelization map → plan-review + "implement?" gate → the **Execution engine**. (No separate general-plan gate — the selected suggestions are already the items.)

> **Superpowers boost:** `superpowers:brainstorming` (the suggester ideates — distinct, codebase-specific ideas, not a generic dump) → `superpowers:dispatching-parallel-agents` (if Map auto-runs) → `superpowers:writing-plans` → the engine chain.

---

## Development planning (`/recon plan "<task>"`)

Invoking recon with `plan` and a task string produces a full development plan for an arbitrary task (e.g. `/recon plan "build a mobile app"`), then optionally builds it. Does **not** require a prior recon Map — the task may be greenfield.

1. **Decompose.** Dispatch `planner` (decompose mode, `## Planner dispatch (decompose)`) with the task → a **general plan** (approach + rationale, workstreams, sequencing, open assumptions) written to `.recon/plans/<plan-id>/general-plan.md`.
2. **General-plan gate.** The operator approves or requests revisions (loop until approved).
3. **Detail + execute.** Hand the approved workstreams (mode = feature) to the `planner` (detail mode) → detailed sub-plans + parallelization map → plan-review + "implement?" gate → the **Execution engine**.

> **Superpowers boost:** `superpowers:brainstorming` (decompose — shape the vague task) → `superpowers:writing-plans` (detail — real implementation plans) → the engine chain.

---

## Standalone fix / optimize (`/recon fix`, `/recon optimize`)

`/recon fix` and `/recon optimize` are **front doors to the Execution engine** for when findings already exist or the operator wants only the apply step — the same engine as the in-pipeline Phase 4/5, no duplicated logic.

- **`/recon fix`** seeds the engine with operator-selected **`.recon/ISSUES.md`** findings (mode = fix).
- **`/recon optimize`** seeds it with operator-selected **`.recon/OPTIMIZATIONS.md`** findings (mode = optimize).

Each:
1. **Prerequisite (auto-run Hunt if missing).** If the needed findings file is absent, auto-run Phase 1 (Setup) + 2 (Map) + 3 (Hunt) — including their gates — to produce it; if present, reuse it (offer a refresh).
2. **Selection gate** — the operator picks which findings to apply (same as the Phase-3 hand-off).
3. **Run the engine** per Phase 4 (fix) or Phase 5 (optimize): `planner` detail → autocommit gate → parallel `implementer`s → dossier → verifier → cherry-pick back.

The in-pipeline path (Hunt → Phase-3 gate → Phase 4/5) is unchanged; these subcommands just enter the same engine directly.

> **Superpowers boost:** `superpowers:dispatching-parallel-agents` (if the Hunt auto-runs) → the engine chain — fix adds `superpowers:systematic-debugging` (root-cause per finding), optimize adds `superpowers:test-driven-development` (characterization tests).

---

## Design improvement (`/recon redesign`)

Invoking recon with `redesign` improves the app's **UI/visual design** — a read-only audit (`design-reviewer`) + planning, then implementation by parallel `designer`s through the shared Execution engine (`design` mode). Grounded in the 3-level design method in `knowledge/design/`.

1. **Ground.** Resolve the app URL + start command using the **same resolution + fallback as `/recon:recon test-frontend`**. **No-frontend bail:** if no frontend / dev-start command is resolvable (a pure backend/library repo), report that and **stop** — redesign needs a UI. If the frontend is down, auto-start it in the background and poll until ready (~2s × ~60s); record if recon started it. Read frontend source + any existing `.recon/design.md`.
2. **Dispatch `design-reviewer`** (subagent type `recon:design-reviewer`, `## Design-reviewer dispatch`): audit the three levels (single-page craft / system consistency / testability) from source + live Playwright screenshots. **FIRST run authors** `.recon/design.md`; **LATER run audits** the existing one and proposes refinements. If it can't reach the Playwright tools, the parent drives the browser; if neither can, do a static source-only review and say so.
3. **Record + gate.** The parent writes `.recon/design.md` + `.recon/design/review-<date>.md` (screenshots under `.recon/design/screenshots/`), presents them, and the operator **selects which improvements to pursue** and approves the design.md.
4. **Plan.** Dispatch `recon:planner` (detail mode, `## Planner dispatch (detail)`) over the selected improvements (mode = `design`) → one design sub-plan each (goal/acceptance tied to design.md, declared file set, the design.md pins) + the parallelization map.
5. **Plan-review + "implement?" gate** (+ the autocommit prompt).
6. **Execute** via the Execution engine in **`design` mode**: parallel `recon:designer`s (own worktrees + dev-server ports) → design test gate (static anti-pattern checks + Vizzly/Playwright visual diff vs the shared baseline, operator-approved + suite ≥ baseline) → out-of-declaration check → `recon:verifier` on risky (design-reframed) → parent authors the `.recon/design/<n>-<slug>/` dossier → cherry-picks each surviving commit back.
7. **Finish.** Summary + `superpowers:finishing-a-development-branch`; stop the app if recon started it; clean up worktrees. Never the default branch.

**Companions (recommended):** best with `web-design-guidelines` (`npx skills add vercel-labs/agent-skills`) for live-source principle audits and **Vizzly CLI** (`vizzly-testing/cli`) for visual-regression TDD — graceful Playwright fallback if Vizzly is absent.

> **Superpowers boost:** `superpowers:brainstorming` (the reviewer proposes distinctive, codebase-specific moves, not generic polish) → `superpowers:dispatching-parallel-agents` → `superpowers:writing-plans` → `superpowers:using-git-worktrees` → `superpowers:test-driven-development` (the designer writes the static + visual pins first) → `superpowers:verification-before-completion` → the engine chain.

---

## Frontend audit (`/recon test-frontend`)

Invoking recon with the argument `test-frontend` runs a browser audit of the running app (not the full pipeline), via the read-only `frontend-auditor` agent (Playwright). It never edits source.

1. **Resolve** the app URL + start command from `package.json` scripts (`dev`/`start`, incl. pnpm `-F <app>`) / README / a sensible default; ask the operator only if undeterminable.
2. **Probe** the URL; if down, **start the app** in the background and poll until ready (~2s × ~60s). Record that recon started it.
3. **Derive the key user flows** from the router/nav and **confirm them with the operator** (or use flows the operator supplied).
4. **Dispatch `frontend-auditor`** (subagent type `frontend-auditor`, `## Frontend-auditor dispatch` block) to drive the flows and report client-side findings with evidence under `.recon/frontend/`. If the subagent can't reach the Playwright MCP tools, the parent drives the browser directly; if neither can, do a static client-code review and say so.
5. **Record** findings under `## Frontend findings (Playwright)` in `.recon/ISSUES.md` (tagged `[frontend]`; cross-reference any that match a static finding).
6. **Stop** the app if recon started it (leave a pre-existing server alone).
7. **Offer** a scoped **Phase-4 Fix** on the operator-selected `[frontend]` findings (the same gated, test-backed loop).

---

## Active pentest (`/recon pentest`)

Invoking recon with the argument `pentest` runs an **active, gated, non-destructive** penetration test of the running app (not the full pipeline), via the `pentester` agent — the only worker that sends adversarial traffic. It is read-only on source and reuses Phase 4 for fixes. **Authorized testing only.**

1. **Authorization gate.** Confirm the operator owns / is authorized to test the target, and fix the in-scope host(s). **Loopback (`127.0.0.1`) is the default**; a non-loopback target needs a second explicit confirmation. Record the attestation (operator handle + target + timestamp) in `.recon/pentest/authorization.md`; refuse to proceed without it.
2. **Ensure target up.** Probe the target; if down, offer to start the backend (noting infra deps from `docker-compose.yml` / `package.json` / ARCHITECTURE) and poll until ready (~2s × ~60s); record if recon started it.
3. **Build + classify the attack plan.** Enumerate the surface (GraphQL introspection → OpenAPI → source grep); fold in any `threat-scout` hypotheses; assemble the OWASP API Top-10 (2023) sweep; classify each probe `read-only` / `state-changing` / `install`.
4. **Gate (pre-dispatch).** Auto-allow `read-only` probes; get operator approval for every `state-changing`/`install` probe (batch pre-grant allowed) and provision any throwaway test accounts as a single approved up-front batch.
5. **Dispatch `pentester`** (subagent type `pentester`, `## Pentester dispatch` block) with ONLY the approved probe list + test creds. The one-shot agent runs the list, proves each vuln with a request/response PoC, and reports (unapproved high-value probes come back as recommendations). If it can't reach the Playwright tools, the parent drives browser-assisted steps; pure HTTP probing needs none.
6. **Record** confirmed vulns under `## Exploited vulnerabilities (active pentest)` in `.recon/ISSUES.md` (tagged `[pentest]`, with PoC + traced `file:line` + evidence under `.recon/pentest/`); upgrade any matching static finding to "proven exploitable" (its `Source` gains `+pentest`); log one-line negative results.
7. **Clean up** throwaway test data; **stop** the app if recon started it.
8. **Offer** a scoped **Phase-4 Fix** on the operator-selected `[pentest]` findings — the **PoC is the regression test** (passes against the vuln, fails after the fix); the verifier gate is mandatory for high-severity ones.

> **Safety:** strictly non-destructive — no DoS, no data destruction, no attacks outside scope, no third-party services. The authorization + non-destructive + pre-dispatch-gating contract lives in `${CLAUDE_PLUGIN_ROOT}/agents/pentester.md` and `reference/conventions.md`.

---

## References

- `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/conventions.md` — severity rubric, finding entry format, dismissed leads format, Execution-engine contract + feature-commit template
- `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/artifact-templates.md` — ARCHITECTURE.md / ISSUES.md / OPTIMIZATIONS.md skeletons + Fix dossier + suggestions / general-plan / sub-plan / parallelization-map / feature-dossier skeletons
- `${CLAUDE_PLUGIN_ROOT}/skills/recon/reference/dispatch-prompts.md` — fill-in dispatch prompts for each worker type (incl. suggester / planner / implementer)
- `${CLAUDE_PLUGIN_ROOT}/agents/mapper.md` — mapper worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/hunter.md` — hunter worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/refactoring-auditor.md` — refactoring-auditor worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/verifier.md` — verifier worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/threat-scout.md` — threat-scout worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/frontend-auditor.md` — frontend-auditor worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/pentester.md` — pentester (active red-team) worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/suggester.md` — suggester (read-only feature ideator) worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/planner.md` — planner (read-only general + detailed planning) worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/implementer.md` — implementer (write-enabled, worktree-isolated executor) worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/design-reviewer.md` — design-reviewer (read-only UI/visual design auditor + planner) worker definition
- `${CLAUDE_PLUGIN_ROOT}/agents/designer.md` — designer (write-enabled, worktree-isolated UI design executor) worker definition
- `${CLAUDE_PLUGIN_ROOT}/skills/recon/knowledge/design/` — design knowledge base (design-levels, anti-patterns, principles) read by the design-reviewer
- Evolve this skill itself with `superpowers:writing-skills`.
