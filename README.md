# recon

**An orchestrator of subagents for spec-driven development** — packaged as a plugin for [Claude Code](https://code.claude.com).

recon decomposes work into specs and plans, then fans it out across purpose-built subagents — read-only auditors plus write-gated, worktree-isolated implementers. It maps and audits a codebase **and** drives spec-driven feature work (brainstorm → spec → plan → parallel build) end to end, with a human gate between every phase.

The main Claude Code session is the **parent**: it drives the phases and fans out **read-only worker subagents** (one per area). The only worker that writes code is the `implementer`, which runs in an **isolated git worktree**, is **test-gated**, and whose commits the parent **cherry-picks back** as separate commits. Nothing touches your default branch.

> **Heads-up:** as a plugin, the skill is namespaced. The trigger is **`/recon:recon`** (not `/recon`), and subcommands follow it: `/recon:recon pentest`, `/recon:recon suggest`, etc.

---

## Install

**Recommended companions (install first):**

- **[Superpowers](https://github.com/obra/superpowers-marketplace)** — recon's workflows lean on `superpowers:*` skills throughout (brainstorming, writing-plans, test-driven-development, using-git-worktrees, finishing-a-development-branch). Install it so those chains work:

  ```text
  /plugin marketplace add obra/superpowers-marketplace
  /plugin install superpowers@superpowers-marketplace
  ```

- **[Web Design Guidelines](https://github.com/vercel-labs/agent-skills)** (Vercel Labs) — a live-source UI design-principles audit (`/web-interface-guidelines`) that `/recon:recon redesign` leans on. Install via the `skills` CLI:

  ```text
  npx skills add vercel-labs/agent-skills
  ```

- **[Vizzly CLI](https://github.com/vizzly-testing/cli)** (optional) — visual-regression TDD for `/recon:recon redesign`. With it, the `designer` reviews pixel diffs with metadata and approve/reject; without it, recon falls back to plain Playwright before/after screenshots. Install (Node 22+): `npm i -g @vizzly-testing/cli`, then `vizzly init` and `vizzly tdd start`.

**Then recon:**

```text
/plugin marketplace add faridjafarlee/recon
/plugin install recon@recon
```

Then **reload** (or restart the session) so the skills + agents register:

```text
/reload-plugins
```

## Update

When new changes are pushed, pull them with:

```text
/plugin marketplace update recon
```

(The plugin is unversioned on purpose — every push is an update. Run the command above to fetch the latest commit.)

---

## What it does

| # | Phase | What it does | Output |
|---|-------|--------------|--------|
| 1 | **Setup** | detect stack, capture test baseline, partition the repo into areas (seeded from `.understand-anything/domain-graph.json` if present) | area list |
| 2 | **Map** | one `recon:mapper` per area (parallel, read-only) → architecture map | `.recon/ARCHITECTURE.md` |
| 3 | **Hunt** | one `recon:hunter` per area + `recon:refactoring-auditor` + `recon:threat-scout` (read-only) → evidenced candidates; the parent re-reads each cited line before recording | `.recon/ISSUES.md`, `.recon/OPTIMIZATIONS.md` |
| 4 | **Fix** | applies selected **`ISSUES.md` defects** via parallel `recon:implementer`s in worktrees: root-cause → TDD reproduce → fix → full-suite regression → verifier → dossier → cherry-pick | commits + `.recon/fixes/<id>/` |
| 5 | **Optimize** | applies selected **`OPTIMIZATIONS.md` refactorings/perf**, **behavior-preserving**: characterization tests where coverage is thin → refactor → *identical* test outcomes → perf gain check → verifier → dossier → cherry-pick | commits + `.recon/fixes/opt-<n>-<slug>/` |

Drive the full pipeline with **`/recon:recon`** — the parent stops at a gate between each phase for your review/approval. You can stop after any phase (e.g. map + hunt only).

### Subcommands

| Command | What it does |
|---------|--------------|
| `/recon:recon test-frontend` | Browser audit of the running app via the read-only `recon:frontend-auditor` (Playwright). Records `[frontend]` findings; offers a scoped Fix pass. |
| `/recon:recon pentest` | **Authorized, non-destructive** active API red-team via `recon:pentester` (loopback default; pre-dispatch gating of every state-changing/install probe). Confirmed vulns get a request/response PoC. |
| `/recon:recon suggest` | The `recon:suggester` (Opus) proposes ≥10 codebase-specific features grounded in the Map → you select → `recon:planner` writes parallelizable sub-plans → optional build. |
| `/recon:recon plan "<task>"` | The `recon:planner` decomposes an arbitrary task (greenfield OK) into a general plan → you approve → detailed parallel sub-plans → optional build. |
| `/recon:recon fix` | Front door to the Fix engine on selected `ISSUES.md` findings (auto-runs the Hunt first if missing). |
| `/recon:recon optimize` | Front door to the Optimize engine on selected `OPTIMIZATIONS.md` findings (auto-runs the Hunt first if missing). |
| `/recon:recon redesign` | Improve the app's **UI/visual design**: `recon:design-reviewer` audits the running app (source + Playwright) → writes `.recon/design.md` + an improvement plan → you select → parallel `recon:designer`s implement each in a worktree, gated by visual-regression TDD → cherry-pick back. |

## Subagents & orchestration

recon is a **parent + workers** system. The **parent** is your Claude Code session: it never edits source itself — it **drives the phases, dispatches subagents, independently re-verifies their findings, owns every write to `.recon/` and the work branch, and stops at a human gate between phases.** Each worker is a single-purpose subagent with a tight tool allowlist. All workers are **read-only on your source** except the two write-enabled builders (`implementer` and `designer`), which write *only* inside their own isolated git worktrees.

Workers are spawned through the Agent tool by their **plugin-namespaced type** (`recon:mapper`, `recon:hunter`, …). The fill-in prompt for each lives in `skills/recon/reference/dispatch-prompts.md`; the severity rubric + finding format in `reference/conventions.md`; the output skeletons in `reference/artifact-templates.md`. Workers return **structured notes only** — they never write the deliverables; the parent does, after re-reading each cited `file:line`.

### The roster

| Subagent | Model | Source access | Dispatched in | Role |
|----------|-------|---------------|---------------|------|
| `recon:mapper` | Opus | read-only | Phase 2 (Map) — one per area, parallel | Reads one area; returns purpose, entry points, key files, data flow, dependencies, risk leads → `ARCHITECTURE.md`. |
| `recon:hunter` | Opus | read-only | Phase 3 (Hunt) — one per area, parallel | Audits one area for defects + improvement leads → candidate findings. |
| `recon:refactoring-auditor` | Opus | read-only | Phase 3 — once, repo-wide | Scans for code smells using its bundled refactoring catalog → `OPTIMIZATIONS.md`. |
| `recon:threat-scout` | Opus | read-only | Phase 3 — once, repo-wide | Targeted threat-model hunt of the highest-risk bug classes for *this kind* of app. |
| `recon:verifier` | Sonnet | read-only | Phase 4/5 + engine — on risky changes | Independently re-checks a change before cherry-pick → `APPROVE` / `APPROVE-WITH-NOTES` / `REJECT`. |
| `recon:frontend-auditor` | Opus | read-only (drives a browser) | `/recon:recon test-frontend` | Exercises the running app's user flows via Playwright → client-side findings. |
| `recon:pentester` | Opus | read-only on source, **active on the network** | `/recon:recon pentest` | Authorized, non-destructive API red-team → PoC-proven vulnerabilities. |
| `recon:suggester` | Opus | read-only | `/recon:recon suggest` | Ideates ≥10 codebase-specific features grounded in the Map. |
| `recon:planner` | Opus | read-only | suggest / plan / fix / optimize | **Decompose** mode: a task → a general plan. **Detail** mode: items → per-item sub-plans + a file-overlap **parallelization map**. |
| `recon:implementer` | Sonnet | **writes — only inside its own git worktree** | execution engine (fix/optimize/feature) | One per sub-plan; implements, tests against the baseline, makes one commit. |
| `recon:design-reviewer` | Opus | read-only (drives a browser) | `/recon:recon redesign` | Audits the frontend's design (source + Playwright) across 3 levels → `.recon/design.md` + a prioritized improvement plan. |
| `recon:designer` | Sonnet | **writes — only inside its own git worktree** | execution engine (design) | One per design sub-plan; implements a visual change, verifies via visual-regression TDD (Vizzly/Playwright), makes one commit. Runs its own dev server + headless capture. |

### How a run flows

```text
parent (your session) ── drives phases · owns .recon/ + the work branch · gates between phases
  │
  ├─ Phase 2  Map ──▶ recon:mapper ×N            (∥, read-only) ─────────────▶ .recon/ARCHITECTURE.md
  │
  ├─ Phase 3  Hunt ─▶ recon:hunter ×N            (∥, read-only) ─┐
  │                   recon:refactoring-auditor  (∥, read-only)  ├─▶ ISSUES.md / OPTIMIZATIONS.md
  │                   recon:threat-scout         (∥, read-only) ─┘   (parent re-reads each cited
  │                                                                    file:line before recording)
  │
  └─ Phase 4/5 · suggest · plan ───────────────▶  Execution engine  ▼

     recon:planner (Opus) ──▶ detailed sub-plans + parallelization map (disjoint-file batches)
          │
          ▼  per batch — up to implConcurrency in parallel
     recon:implementer (Sonnet) ─ own git worktree ─ implement → test ≥ baseline → ONE commit
          │
          ▼
     recon:verifier (risky changes) ─▶ APPROVE / REJECT
          │
          ▼
     parent ─ cherry-picks each surviving commit back onto the work branch (separate commits, no squash)
```

### The execution engine

Phase 4 (Fix), Phase 5 (Optimize), and the build half of `suggest` / `plan` all funnel into one engine, so there is a single, consistent way changes land:

1. **Plan** — `recon:planner` (detail mode) turns the selected items into one sub-plan each, every sub-plan declaring the exact **file set** it will touch, then groups sub-plans with **disjoint** file sets into parallel **batches** (overlapping ones fall to later batches).
2. **Build, in parallel, isolated** — for each batch the parent spawns one `recon:implementer` per sub-plan, each in its **own git worktree** (so concurrent builds can't collide), capped at `implConcurrency`. Each implementer works to a mode: **fix** (reproduce test → fix), **optimize** (characterization tests → refactor, identical outcomes, perf gain), or **feature** (TDD build).
3. **Gate** — the parent checks each diff against its declared file set, runs `recon:verifier` on risky ones, and writes a per-change **dossier** under `.recon/fixes/…`.
4. **Integrate** — the parent **cherry-picks** each passing worktree commit back onto the work branch as a separate commit (no squashing), pausing on any conflict or `REJECT`. Never the default branch.

## Safety model

- **Only one worker edits** — every worker is read-only (no Edit/Write in its `tools:`) except `recon:implementer`, which writes **only inside its own git worktree** (never the originating/default branch or `.recon/`), stays within its plan's **declared file set** (parent-verified before cherry-pick), and makes one commit the parent cherry-picks back (**no squashing**).
- **Never commits to the default branch.**
- **Every fix is proven** — a green full suite vs the Phase-1 baseline + a reproduce test (Fix) or characterization tests with *identical outcomes* (Optimize). No weakening/deleting tests to pass.
- **Verifier gate** on high-severity/risky changes — a `REJECT` blocks the cherry-pick and pauses (in both autocommit modes).
- **`pentest` is authorized + non-destructive only** — attacks only an operator-attested, in-scope target (loopback default), runs no destructive/DoS techniques, and gates every state-changing/install action before dispatch.
- **`.recon/ISSUES.md` is the diagnostic deliverable** — every confirmed finding is recorded in full whether or not it gets fixed.

## Per-run knobs (Fix / Optimize)

- **Autocommit** `yes|no` — asked at the start; gates whether each passing worktree commit is auto-cherry-picked or shown for your OK first.
- `implConcurrency` (default **10**) — max parallel implementers (read-only workers are uncapped).
- `designConcurrency` (default **3**) — max parallel designers in `/recon:recon redesign` (each runs a dev server + browser, so heavier).
- `perfRuns` (default **5**), `perfTarget` = `auto|suite|bench|off` — **optimize-mode only**; fixes don't benchmark.
- `dossier` = `commit` (default) `|gitignore`.

## Models

Per-agent models are in the roster above (Opus for discovery/planning, Sonnet for `verifier` + `implementer`), set via each agent's `model:` frontmatter. A model change takes effect only after a session reload — the agent registry is cached at session start.

## Layout

```text
recon/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace manifest (this repo is its own marketplace)
├── skills/recon/
│   ├── SKILL.md             # the orchestrator playbook (5 phases + engine + subcommands)
│   ├── reference/           # conventions, dispatch prompts, artifact templates
│   └── knowledge/refactoring/   # code-smells + refactoring-techniques catalog
├── agents/                  # 12 workers (read-only on source; implementer + designer write only in their worktrees)
└── docs/                    # design spec + implementation plan (provenance)
```

Bundled files are referenced internally via `${CLAUDE_PLUGIN_ROOT}` so they resolve wherever the plugin is installed.

## Notes

- After installing or updating, **reload** so the new agents become dispatchable (the agent registry is cached at session start).
- The refactoring knowledge base is original prose (Fowler methodology), not copied from third-party sites.

## License

[MIT](LICENSE) © Farid K. Jafarli
