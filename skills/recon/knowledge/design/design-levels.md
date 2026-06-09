# The three levels of design

The reviewer's mental model for taking a frontend from "looks AI-generated" to "looks intentionally
built." Each level is a different unit of work; most teams stop at Level 1.

Core premise: a capable model **converges to one safe default style**. That default is a giveaway —
the moment you recognize it, it reads as AI slop. The job at every level is to push the design *off*
the default and toward something coherent and context-specific.

---

## Level 1 — a single page (prompt/spec craft)

Most of the quality of a single page is decided before any pixel is rendered, by how the design is
**specified**. Structure the intent in this order — skipping the later items is the #1 reason output
looks generic:

1. **Intent** — what this page/app is and the feeling it should create.
2. **Non-negotiables** — the exact elements required and how they should look/behave.
3. **Color system** — defined explicitly (see `principles.md` → OKLCH), not left to the model's
   default. This is the most-skipped step.
4. **Contrast flows** — state the contrast relationships; contrast is what creates hierarchy and
   guides the eye. Without explicit contrast the model treats every element as equally important and
   no hierarchy forms.
5. **Typography** — which fonts are **banned** (AI-slop giveaways) and which to use where. Naming the
   bans forces the model off its defaults.
6. **Layout & rhythm** — symmetry vs asymmetry, spacing rhythm, negative space (see `principles.md`).
7. **Sections, materials, responsive behavior** — the concrete structure and how it adapts.
8. **Anti-patterns** — the most important item: explicitly list the AI-slop tells to avoid (see
   `anti-patterns.md`). What you forbid matters as much as what you ask for.

A well-structured spec like this can one-shot a distinctive page; a vague prompt gets the default.

---

## Level 2 — systems, not pages

Agent-built apps usually look fine on the landing page and then fall apart: the dashboard gets
different button styles, spacing, and typography, as if the agent forgot it was the same app. That
incoherence is itself an AI-generated tell. The fix is to make the **visual system** explicit and
durable, split across two files:

- **CLAUDE.md → project information only.** It stays loaded every session; putting design content
  here distracts the agent on unrelated work. Keep it lean — but it still matters, because project
  context informs good design.
- **design.md → the entire visual system.** Layout, the OKLCH color system, typography, spacing,
  components, and the anti-pattern list (everything from Level 1). It must be the kind of file *any*
  agent can pick up and immediately know the visual system.

Two practices keep design.md good:
- **Self-refining header.** A line at the top instructing the agent to append any new design value it
  discovers, so every session starts from a more refined system than the last.
- **Cross-verify + audit.** Verify design.md against a reference template, and audit the design
  against a **live-source** design-guidelines skill (one that points at an externally maintained set
  of principles, so it stays current rather than frozen). See `principles.md` → Sources.

In recon, this file is `.recon/design.md` (recon's working visual system).

---

## Level 3 — testing designs like code (TDD for UI)

Design is subjective, but it is still testable. TDD works for code because a test **pins** the
intended behavior and the implementation must satisfy the pin. Design uses the same idea with
different kinds of pins.

- **design.md is the source of truth** for the tests: every anti-pattern, every color rule, every
  spacing/typography constraint becomes a check (a "pin").
- **Write the tests before the change.** If tests are written after, the agent writes checks that
  rationalize the existing code (it's already in context). Tests-first forces the implementation to
  fit the intended design.
- **Two kinds of checks:**
  - **Static checks** — directly assert the mechanically-detectable anti-patterns (banned fonts,
    default icon sets, glassmorphism utilities, centered-only hero). See `anti-patterns.md` for which
    are greppable.
  - **Visual regression** — screenshot the rendered UI and diff it; iterate to incrementally improve.
- **Visual-regression TDD tool — Vizzly CLI (`vizzly-testing/cli`).** It runs local TDD: a server
  watches screenshot changes; the agent pushes Playwright screenshots; you review **diffs with
  metadata** (which pixels changed, by how much) and **approve or reject**. Each rejected diff becomes
  feedback for the next pass, so the design **converges to what you actually want** rather than what
  the agent guesses. If Vizzly isn't installed, fall back to plain Playwright before/after
  screenshots reviewed by the operator.
