# AI-slop anti-patterns

The recognizable tells of agent-generated UI. Each entry: the tell, why it reads as AI-generated, the
better move, and whether it's **[greppable]** (a reliable static check for the L3 gate) or
**[visual]** (judged from the rendered screenshot, not code).

These are the catalog the `design-reviewer` flags and the source of the `designer`'s static pins.

---

- **Default fonts: Inter, Geist.** **[greppable]** — every agent reaches for them, so they signal
  "generated." Check `font-family` declarations, `@font/next/font` imports, Google-font links.
  **Better:** choose a distinctive, context-fit pairing (a characterful display face + a clean text
  face); state it in design.md and use roles deliberately.

- **Centered-only hero / single centered CTA.** **[visual]** (with a **[greppable]** heuristic: a hero
  whose only layout is `text-center` + a lone centered button). Reads as the default template.
  **Better:** intentional composition — often asymmetric, with a clear focal point and negative space.

- **Lucide icons (the default icon set).** **[greppable]** — detect `lucide-react`/`lucide` imports.
  Ubiquity makes them a tell. **Better:** an icon set that fits the product's character (or a small
  custom set); at minimum, don't lean on the default everywhere.

- **Glassmorphism (frosted glass + gradients).** **[greppable]** — detect `backdrop-blur` /
  `backdrop-filter` (and gradient-over-glass card patterns). An overused "modern" default.
  **Better:** solid, intentional surfaces with real depth (considered shadows/borders/layering).

- **Purple-on-white default theme.** **[visual]** (heuristic **[greppable]**: dominant purple accents
  on a white ground). The classic "first thing the model reaches for." **Better:** commit to a
  cohesive palette derived from the product's context, not the default.

- **Timid, evenly-distributed palettes.** **[visual]** — every color at similar weight → no focus.
  **Better:** a dominant color with sharp, sparing accents (see `principles.md` → contrast/hierarchy).

- **Symmetry by default.** **[visual]** — everything centered/grid-balanced regardless of content.
  Fine for some professional contexts, but applied reflexively it's flat. **Better:** use asymmetry
  and negative space where an artistic, distinctive feel is wanted; let the product decide.

- **Generic, predictable layout with no context-specific character.** **[visual]** — the page could be
  any SaaS. **Better:** details that are specific to *this* product's domain and brand.

---

**For the L3 static gate:** only the **[greppable]** items become hard, mechanically-checked pins
(font names, Lucide imports, `backdrop-blur`/`backdrop-filter`, the centered-only-hero heuristic).
The **[visual]** items are judged from the before/after screenshot diff + operator approval — do not
fake them as greps (e.g. "is this palette timid?" is not a reliable static check).
