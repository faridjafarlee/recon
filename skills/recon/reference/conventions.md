# Recon conventions

Single source of truth for scoring and recording findings. Workers receive the
relevant parts inline at dispatch; the parent applies them when recording.

## Severity rubric
Severity is a qualitative judgement that weighs impact, blast-radius, and likelihood together — there is no numeric formula.
- **critical** — exploitable security hole, data loss, or outage path; or a change-preventer smell forcing edits across many modules. Fix before shipping.
- **high** — real defect or strong refactoring need with broad reach; should be scheduled now.
- **medium** — genuine issue, contained blast radius; fix opportunistically.
- **low** — cosmetic or stylistic (e.g. Comments smell, minor naming); batch or defer.
Workers suggest a severity; the parent sets the final value.

## Finding entry format
- **Title:** short imperative phrase
- **Category:** security | performance | reliability | tooling | test | refactoring
- **Severity:** critical | high | medium | low
- **Location:** `path/file:line` (+ related sites)
- **What it is:** one or two concrete sentences
- **Why it matters:** the real impact
- **Trigger / exploit:** the concrete path that hits it
- **Proposed fix:** the minimal root-cause change, described (not applied)
- **Confidence:** high | medium | low — and what would confirm it
- **Source:** [scan] | [targeted] | [scan+targeted] | [frontend] | [scan+frontend] | [pentest] | [scan+pentest] | [targeted+pentest] | [frontend+pentest] — set by the parent when recording, based on which worker returned the finding
- **Status:** `- [ ]`

## Dismissed leads
Rejected candidates get one line: `- <Title> — <reason>` so later passes
don't re-raise them.

## Phase 4 — fix status & commit

Finding status in ISSUES.md:
- `- [ ]` — open (not yet fixed).
- `- [x]` — fixed. Append `→ .recon/fixes/<finding-id>/fix.md` and the commit hash.
- A blocked finding keeps `- [ ]` and gains a note line: `> blocked: <reason>`.

Phase-4 commit message (extends the project commit template with a Performance line):
This is a standard structured commit template plus a `Performance:` line; it applies to Phase-4 fix commits only (use `n/a` when no benchmark applies). (If the host project defines its own commit convention — e.g. a `CLAUDE.md` commit template — follow that and keep the `Performance:` line.)
```
[<category>] <short title>

Problem: <one line>
Root cause: <one line>
Severity: <critical|high|medium|low> — <one-line why>
Trigger/Exploit: <one line>
Fix: <one line>
Performance: <target> <before mean> → <after mean> (<±%>), or n/a
Compatibility: full suite <X/Y> → <X/Y>; <notes>

Co-Authored-By: ...
```

### Phase 5 — Optimize (additional)
Phase 5 applies `OPTIMIZATIONS.md` findings with behavior preservation. Same fixed/blocked grammar, plus these blocked reasons:
- `> blocked: behavior changed — <which test flipped>`
- `> blocked: no measurable gain` (performance finding)
- `> blocked: perf regression (>5%)` (refactoring finding)
- `> blocked: cannot pin behavior baseline`

Phase-5 commits use the Phase-4 commit format; the `Performance:` line is **always populated** — a measured gain for `performance` findings, before/after guard numbers for `refactoring` findings (never `n/a`).

## Assessment additions

- **Source tag** (a field the parent sets on every finding when recording): `Source: [scan] | [targeted] | [scan+targeted] | [frontend] | [scan+frontend] | [pentest] | [scan+pentest] | [targeted+pentest] | [frontend+pentest]`. `[scan]` = per-area hunter; `[targeted]` = threat-scout; `[frontend]` = frontend-auditor; `[pentest]` = pentester (active red-team). Combined tags mark a finding seen by more than one worker — when an active exploit confirms a prior static finding, the parent upgrades its Source (e.g. `[targeted]` → `[targeted+pentest]`) and re-labels it "proven exploitable". Combos join the contributing worker tags with `+`; three-way combos like `[scan+targeted+pentest]` are allowed when three workers confirm the same finding. Dedup key is `file:line` (primary); for a pentest finding with no traceable `file:line`, fall back to `endpoint+method+vuln-class`. The primary key governs any conflict.
- **Frontend findings section:** browser-found issues go under a `## Frontend findings (Playwright)` heading in ISSUES.md, placed AFTER the static `## Findings` and `## Dismissed leads`. Browser evidence (screenshots/logs) lives under `.recon/frontend/`. A browser finding matching a static one by `file:line` is cross-referenced (the static entry's `Source:` gains `+frontend`), not duplicated.
- **Diagnostic-notes deliverable:** ISSUES.md and OPTIMIZATIONS.md ARE the running diagnostic record. Every confirmed finding is written up in full (Problem/root-cause/severity/trigger/proposed-fix) whether or not it is later fixed — un-fixed findings keep `- [ ]`. Never silently drop a finding; if time/scope runs out, the written diagnosis stands alone.
- **Exploited vulnerabilities section (`/recon pentest`):** active-pentest results (`Source: [pentest]`) go under an `## Exploited vulnerabilities (active pentest)` heading in ISSUES.md, placed AFTER `## Frontend findings (Playwright)`. Each confirmed vuln uses the finding entry format plus a **PoC** (exact request + redacted impact-proving response) and an evidence path under `.recon/pentest/`. A failed exploit against a high-value hypothesis is a one-line entry under a `### Negative results` subsection: `- <OWASP ID> — <probe> → <outcome> (defense confirmed)` — not a full entry.
- **Pentest safety model (`/recon pentest`):** active attacks run ONLY against an operator-attested, in-scope target (loopback by default; a non-loopback target needs a second explicit confirmation), recorded in `.recon/pentest/authorization.md` (operator handle + target + timestamp). Strictly non-destructive: no DoS, no data destruction/mass mutation; state-changing PoCs only on throwaway accounts created up-front and cleaned up after. All gating is **pre-dispatch** — the parent classifies each probe `read-only`/`state-changing`/`install` and gets operator approval for every `state-changing`/`install` probe (and the test-data batch) before dispatching the one-shot `pentester`. Recon stops only the app it started.
- **Pentest findings in Phase 4:** a selected `[pentest]` finding's reproduce test IS its PoC exploit — it must succeed against the vulnerable code and fail after the fix (security red→green). The verifier gate is mandatory for high-severity `[pentest]` fixes.

## Execution engine (shared by Phase 4/5 and the suggest/plan/fix/optimize subcommands)

One execution model powers Phase 4 (fix), Phase 5 (optimize), `/recon suggest`, and `/recon plan`. The `planner` (Opus) produces detailed sub-plans + a parallelization map; the parent runs the engine below; the `implementer` (Sonnet) is the single delegated-edit worker.

**Implementer write/safety contract:** the `implementer` is the only worker that writes code. It writes ONLY inside its assigned git worktree; never the originating-branch checkout, the main checkout, a sibling worktree, the default branch, or anything under `.recon/`. It stays within the sub-plan's declared file set — the parent verifies with `git diff --name-only` before cherry-pick; any out-of-declaration path → `blocked: wrote outside declared file set — <paths>`, pause. One commit per plan, no squashing, no pushing.

**implConcurrency (default 10, operator-overridable):** caps parallel **implementers only** — the read-only mapper/hunter fan-out stays uncapped. A batch larger than the cap runs in cap-sized waves.

**Batching:** the parallelization map groups sub-plans with DISJOINT declared file sets into the same parallel batch; overlapping plans go to later sequential batches. Run batches in order.

**No-baseline cherry-pick pause:** a plan whose implementer recorded `regression: baseline unavailable` is NEVER auto-cherry-picked, even under `autocommit=yes`; the parent holds it for explicit operator approval (mirrors the Phase-4 baseline-unavailable rule).

**Cherry-pick:** the parent cherry-picks each surviving worktree commit onto the originating branch as a separate commit, in batch order, no squashing. Conflicts (rare) pause for the operator. Never the default branch. Under `autocommit=no`, complete a batch's cherry-picks (after operator OK) before dispatching a later batch that depends on it.

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
(`fix` mode uses the Phase-4 commit template above; `optimize` mode uses the Phase-4 template with the always-populated `Performance:` line, per the Phase 5 rules.)

**Suggestion entry format** (`.recon/development-suggestions/…`): Title / What it adds / Why it fits this codebase / Surface area + effort (S|M|L) / Dependencies & risks / Selected `[ ]`.

**Sub-plan entry format** (`.recon/plans/<plan-id>/<n>-<slug>.md`): Mode / Goal+acceptance / Declared file set / Steps / Tests / Origin.

**Engine blocked reasons (additional):** `> blocked: wrote outside declared file set — <paths>`; `> blocked: depends on <failed plan>` (held dependent in a later batch); `> blocked: cherry-pick conflict — <files>`.

### `design` mode (the `/recon redesign` track)

The engine's fourth mode. Builder = **`designer`** (Sonnet), **not** `implementer`. Gate = visual-regression TDD (static anti-pattern checks + Vizzly/Playwright visual diff + suite ≥ baseline) — see the SKILL "Design improvement" section.

- **`designConcurrency` (default 3, operator-overridable)** — separate from `implConcurrency` (10) because design builds are heavier: each `designer` runs a **dev server + headless browser** in its worktree. Larger batches run in waves; each designer uses a **distinct port** = `BASE_PORT + in-wave slot index`.
- **Shared visual baseline** captured against the originating branch before the first wave (`.recon/design/baseline/` or the seeded Vizzly baseline), re-captured between waves; parent-owned.
- **No-baseline carve-out:** the §5.2 operator **visual approve/reject IS the human gate**, so design mode does **not** also apply the generic "never auto-cherry-pick a `baseline unavailable` plan" pause (no redundant double gate).
- **Design dossier** lives at `.recon/design/<n>-<slug>/` (NOT under `.recon/fixes/`).

**Design-commit template** (`designer`, feature-shaped, `[design]`):
```
[design] <short title>

Goal: <the visual improvement, one line>
Design.md pins: <which rules/anti-patterns this satisfies>
What changed: <one line>
Visual: <before→after summary; screenshots in the dossier>
Tests: <static checks + suite X/Y → X/Y, or "baseline unavailable — manual review">
Plan: <plan-id>/<n>-<slug>
Compatibility: <behaviour preserved>

Co-Authored-By: ...
```

**Design blocked reasons (additional):** `> blocked: visual regression rejected`; `> blocked: introduces anti-pattern — <which>`.
