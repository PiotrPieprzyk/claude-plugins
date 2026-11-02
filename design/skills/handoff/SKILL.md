---
name: handoff
description: Package a finished design as a handoff folder (design-handoff/) that Claude Code can build from — HANDOFF.md with design tokens and visual rules, plus one folder per deliverable (page, email or component) holding .dc.html mockups per viewport width and a STRUCTURE.md (split into sections) or BEHAVIOUR.md (single file). Use when the user asks to hand off, export or package a design for development, prepare a design for Claude Code, or write a design spec/handoff for a page, email or component.
---

# Writing a design handoff

A handoff is a folder that tells a developer (usually Claude Code) **what the design
looks like and how it behaves for the user**. It is the designer's source of truth:
tokens, layout, copy, states and interactions. It is **not** a technical spec — the
developer decides how to build it (markup, ARIA, SEO, image pipeline, backend).

The test for every line you write:

> **Could a designer verify it by looking at the screen or using the prototype?**
> Yes → it belongs in the handoff. No (needs devtools, code, a server or a crawler) → leave it out.

## 1. Folder layout

```
design-handoff/
├── HANDOFF.md                    ← shared: intent, tokens, components, rules, placeholders
└── <deliverable>/                ← one per page / email / component, kebab-case
    │
    │   SPLIT into sections (long pages: landing page, multi-section screen)
    ├── STRUCTURE.md              ← section order + per-section layout and behaviour
    ├── <section>-<width>.dc.html ← e.g. hero-375.dc.html, hero-1440.dc.html
    └── ...
    │
    │   or NOT SPLIT (one artboard: email, component, short screen, dialog)
    ├── BEHAVIOUR.md              ← states and interactions
    └── <deliverable>-<width>.dc.html   ← e.g. welcome-email-600.dc.html
```

- `<deliverable>` names the thing as the business does: `landing`, `checkout`,
  `order-confirmation-email`, `price-card`.
- `<width>` is the artboard width in px, never a word: `375`, `768`, `1440`, `600`.
  Produce every width you designed; mobile-first means the smallest width is the reference.
- `<section>` is a short kebab-case name (`hero`, `services`, `pricing`, `footer`,
  `sticky-bar`). Order lives in STRUCTURE.md, not in file names.
- A section that exists at only one width (e.g. a mobile-only sticky bar) has only that file.
- States: put all states of a section/component into **one** file, switchable through a
  `data-props` enum (`status: empty | loading | ready | error`). If a side-by-side states
  sheet helps, add `<section>-states-<width>.dc.html` and mark it "reference only".

**Split or not?** Split when the deliverable has several stacked sections that can be
built and reviewed independently (rule of thumb: 3+ sections or taller than ~3 screens).
Don't split emails, single components, dialogs or short screens.

## 2. HANDOFF.md (one per handoff folder)

Shared by every deliverable in the folder. Use the template in
[templates.md](templates.md#handoffmd). Sections, in order:

1. **Intent** — what it is, who it's for, the feel in one sentence ("solid local
   craftsmen, not startup"), language of the copy, chosen variants and dropped ones
   ("Hero A chosen, don't build Hero B").
2. **What's in this folder** — the tree, with one line per file.
3. **How to read `.dc.html` files** — copy the block from the template verbatim, then
   list this design's props (e.g. `{{accent}}` = `#FF6B1A`).
4. **Layout** — breakpoints, max content width, side padding per breakpoint, grid,
   and the instruction that artboards are fixed-width but the build is fluid.
5. **Design tokens** — tables, each token with a name, value and *where it's used*:
   - Colour (incl. text-on-colour pairs and state colours: error, success, disabled)
   - Typography (family + fallback, weights, a role × breakpoint scale table, letter-spacing)
   - Spacing (section padding, gaps), radius, shadows/elevation, borders
   - Motion (durations, easing, what animates) — if anything moves
6. **Components** — reusable pieces with variants: buttons (type × look × height),
   form fields (default / focus / error / disabled), chips, cards, icons (style, stroke,
   set, list of icons used, "no emoji" if so).
7. **Rules** — the non-obvious constraints a developer would otherwise break:
   "accent only on call/submit buttons", "text on accent is always dark",
   "phone number visible in the hero at every width".
8. **Imagery** — art direction only: what photos/illustrations show, real vs. stock,
   aspect ratios per slot, crop/focal point, how placeholders are labelled (they double
   as a shot list).
9. **Placeholders** — every `[bracketed]` value in the copy, so they can be found and filled.

## 3. STRUCTURE.md (split deliverable)

Use the template in [templates.md](templates.md#structuremd).

1. A **section table** in page order: `# | Section | Files | Anchor | Background`.
2. One `###` heading per section, in the same order, with:
   - **Layout per width** — columns, sizes that matter, what stacks/reorders on mobile.
   - **Content** — elements top to bottom with their exact copy (or a pointer to the mockup).
   - **Behaviour** — what the user sees happen: hover/press, open/close, sticky,
     sliders, carousels, validation messages and *when* they appear, state table if any.
3. Page-level behaviour that spans sections (sticky header/bar, scroll-to-anchor,
   show/hide on scroll) goes in a final `## Page behaviour` section.

## 4. BEHAVIOUR.md (non-split deliverable)

Use the template in [templates.md](templates.md#behaviourmd).

1. **Anatomy** — the parts top to bottom, with copy.
2. **Layout per width** — what changes between artboards.
3. **States** — a table: `State | What the user sees | How they get there | How they leave`.
4. **Interactions** — triggers and visible responses, timings, motion.
5. **Edge content** — long names, empty lists, many items, missing image: how it should look.

For emails: note the inbox preview text, which parts are images vs. live text as the user
sees them (e.g. "the headline must stay readable with images off"), and the dark-mode look
if designed. Leave rendering-engine hacks to the developer.

## 5. The `.dc.html` mockups

- One artboard per file, at its exact width. Real copy, real tokens, inline styles.
- Keep behaviour in the `<script type="text/x-dc">` block driven by `data-props`
  (enums for states, colours for themeable tokens), so the developer can flip states.
- Use real interactive elements in the mockup where the design has them (`<button>`,
  `<input>`, `<a>`), so intent is unambiguous.
- Placeholders for images: striped/neutral box with a label describing the shot needed.

## 6. What does NOT go in the handoff

These are implementation decisions. They belong in a tech spec, a UI story
(`/ui:story`) or the developer's judgment — not here.

| Leave out | Why / where it goes |
|---|---|
| Image formats, `srcset`, lazy loading, preloading, compression, HEIC conversion | Image pipeline → dev. Keep only aspect ratios and art direction. |
| SEO: `<title>`, meta description, JSON-LD, Open Graph, sitemap | Marketing/tech spec. |
| ARIA attributes, landmarks, semantic tags, `<label for>`, `aria-live`, screen-reader text | Implementation → dev / `/ui:story`. |
| A11y test checklists ("no horizontal scroll at 320/375/768…", "check with screen reader") | QA → `/ui:story` / test plan. |
| Browser APIs and techniques (IntersectionObserver, `clip-path`, `env(safe-area-inset)`, `<input type="range">` as the control) | Describe the *visible* behaviour; dev picks the technique. |
| Backend, endpoints, email delivery, file upload mechanics, client-side validation code | API story (`/api:story`) / tech spec. |
| Frameworks, libraries, CSS methodology, file structure of the code | Dev. |
| Analytics, tracking, cookie banners (unless they're designed screens) | Tech/legal spec. |
| Business claims to confirm, legal sign-off, project to-dos | Project notes, not design. |

**What stays even though it sounds technical** — because it's visible and the designer owns it:
contrast pairs (as part of the colour tokens), minimum tap-target sizes (as part of
component sizes), what the focus state *looks like*, input text size (e.g. 16px so mobile
doesn't zoom), reduced-motion *look* (e.g. "no slide, just appear"), error message copy and
when it appears.

Rewrite, don't delete, when a technical line hides a design decision:

| Instead of | Write |
|---|---|
| "Use IntersectionObserver on the hero CTA" | "The sticky bar appears once the hero call button has scrolled off screen." |
| "`aria-invalid` + `aria-describedby` on the phone field" | "Invalid phone: red 2px border, `#FDF3F2` background, message below: '…'. Shown after leaving the field and on submit, not while typing." |
| "AVIF/WebP with JPG fallback, hero preloaded" | "Hero photo 16:10 desktop / 4:3 mobile, focal point on the crew." |

## 7. Workflow

1. Confirm what's being handed off: deliverables, chosen variants, widths designed.
2. Collect tokens from the mockups — every colour, size and radius used must map to a
   token in HANDOFF.md; unify near-duplicates (`#1F3D2A` vs `#1F3D2B`) with the user.
3. Decide split / not split per deliverable, then name the files.
4. Write HANDOFF.md, then STRUCTURE.md or BEHAVIOUR.md per deliverable.
5. Run the section 6 check over every file: delete or rewrite any line that fails the
   "could a designer verify it by looking?" test.
6. Check every file referenced in the docs exists and every `.dc.html` is referenced.
