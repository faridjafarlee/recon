# recon

A multi-phase **codebase recon → improve** system for [Claude Code](https://code.claude.com), packaged as a plugin.

The main Claude Code session is the **parent**: it drives the phases and fans out **read-only worker subagents** (one per area). The only worker that writes code is the `implementer`, which runs in an **isolated git worktree**, is **test-gated**, and whose commits the parent **cherry-picks back** as separate commits. Nothing touches your default branch.

> **Heads-up:** as a plugin, the skill is namespaced. The trigger is **`/recon:recon`** (not `/recon`), and subcommands follow it: `/recon:recon pentest`, `/recon:recon suggest`, etc.

---

## Install

```text
/plugin marketplace add faridjafarlee/recon
/plugin install recon@recon
```

Then **reload** (or restart the session) so the skill + agents register:

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
- `perfRuns` (default **5**), `perfTarget` = `auto|suite|bench|off` — **optimize-mode only**; fixes don't benchmark.
- `dossier` = `commit` (default) `|gitignore`.

## Models

Discovery/planning workers (`mapper`, `hunter`, `refactoring-auditor`, `threat-scout`, `frontend-auditor`, `pentester`, `suggester`, `planner`) run on **Opus**; `verifier` and `implementer` run on **Sonnet** — set per-agent via `model:` frontmatter. Model changes take effect after a session reload.

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
├── agents/                  # 10 workers (read-only on source; implementer writes only in its worktree)
└── docs/                    # design spec + implementation plan (provenance)
```

Bundled files are referenced internally via `${CLAUDE_PLUGIN_ROOT}` so they resolve wherever the plugin is installed.

## Notes

- After installing or updating, **reload** so the new agents become dispatchable (the agent registry is cached at session start).
- The refactoring knowledge base is original prose (Fowler methodology), not copied from third-party sites.

## License

[MIT](LICENSE) © Farid K. Jafarli
