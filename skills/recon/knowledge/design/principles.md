# Design principles (the positive system)

What "good" looks like, so the reviewer proposes improvements toward a coherent system and design.md
encodes real rules. Pair with `anti-patterns.md` (what to avoid).

---

## Color — use OKLCH

Define the color system in **OKLCH** (Lightness, Chroma, Hue), not RGB/HSL/hex.
- OKLCH is **perceptually uniform** — it represents color the way the eye actually perceives it, so
  lightness and balance behave predictably.
- It produces **smoother gradients** (hex/RGB interpolation can band or go muddy).
- It makes **contrast** easier to reason about and tune.

Define a real **scale** (background → surfaces → text → accents) and the **contrast relationships**
between tiers, not just a few one-off colors.

## Contrast → hierarchy

Contrast is the primary tool for **visual hierarchy** — it guides the eye to what matters first.
State contrast explicitly. Without it, every element competes at the same weight and the page reads
as undifferentiated. Prefer a **dominant color with sharp, sparing accents** over a timid, even
palette. Respect accessibility contrast ratios (WCAG AA for text) as a floor, not the goal.

## Typography

Control type from the system, not the model's default.
- **Ban the slop fonts** (Inter, Geist — see `anti-patterns.md`); choose a distinctive pairing.
- Define **roles** — display/heading, body, mono — and where each is used, with a clear type scale
  (sizes, weights, line-heights) so hierarchy is consistent across pages.

## Layout, rhythm & negative space

- **Symmetry vs asymmetry:** symmetric, grid-balanced layouts read professional and orderly;
  **asymmetry** gives an artistic, distinctive feel and room to experiment. The **product decides** —
  don't apply either reflexively.
- **Negative space** is a feature: it lets the design breathe and creates focus. Asymmetric layouts
  especially benefit from generous negative space.
- Define a **spacing rhythm** (a consistent scale) and apply it everywhere — inconsistent spacing is a
  top source of the "different page, different app" incoherence.

## Responsive & materials

Specify how the layout adapts across breakpoints and the materials/surfaces (elevation, borders,
shadows, motion) — consistently, as part of the system.

## Accessibility

Semantics before ARIA; visible focus states; sufficient contrast; keyboard support. Good design and
accessible design reinforce each other.

---

## Sources (pointers, kept current externally)

- **`web-design-guidelines`** (Vercel Labs, `vercel-labs/agent-skills`) — a UI design-principles audit
  that points at an **externally maintained** source, so the principles stay current. Install:
  `npx skills add vercel-labs/agent-skills`; then invoke the `web-design-guidelines` skill (Vercel
  documents the agent command as `/web-interface-guidelines`). Prefer it over any frozen, hardcoded
  checklist.
- **Vizzly CLI** (`vizzly-testing/cli`) — visual-regression TDD for Level 3 (see `design-levels.md`).
