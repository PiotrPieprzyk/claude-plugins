---
name: handoff
description: Package a finished design as a handoff folder (design-handoff/) that Claude Code can build from. HANDOFF.md holds only what all deliverables share (contents list, layout, design tokens, global rules). Each deliverable (page, email or component) gets its own folder with a DESCRIPTION.md and .dc.html mockups per viewport width, split into sections when long. Use when the user asks to hand off, export or package a design for development, prepare a design for Claude Code, or write a design spec/handoff for a page, email or component.
---

# Writing a design handoff

A handoff is a folder that tells a developer (usually Claude Code) **what the design
looks like and how it behaves for the user**. It is the designer's source of truth:
tokens, layout, copy, states and interactions. It is **not** a technical spec. The
developer decides how to build it (markup, ARIA, SEO, image pipeline, backend).

The test for every line you write:

> **Could a designer verify it by looking at the screen or using the prototype?**
> Yes → it belongs in the handoff. No (needs devtools, code, a server or a crawler) → leave it out.

## 1. Folder layout

```
design-handoff/
├── HANDOFF.md                         ← common only: contents list, layout, tokens, global rules
└── <deliverable>/                     ← one per page / email / component, kebab-case
    ├── DESCRIPTION.md                 ← everything about THIS deliverable
    │
    │   SPLIT into sections (long pages: landing page, multi-section screen)
    ├── <section>-<width>.dc.html      ← e.g. hero-375.dc.html, hero-1440.dc.html
    │
    │   or NOT SPLIT (one artboard: email, component, short screen, dialog)
    └── <deliverable>-<width>.dc.html  ← e.g. welcome-email-600.dc.html
```

- `<deliverable>` names the thing as the business does: `landing`, `checkout`,
  `order-confirmation-email`, `price-card`.
- **Shared components are deliverables too.** If buttons, form fields or cards are
  reused across pages, give each its own folder (`button/`, `form-field/`) with its own
  DESCRIPTION.md and mockup. Pages then refer to them by name ("orange call button,
  see `button/`") instead of describing them again.
- `<width>` is the artboard width in px, never a word: `375`, `768`, `1440`, `600`.
  Produce every width you designed. With mobile-first, the smallest width is the reference.
- `<section>` is a short kebab-case name (`hero`, `services`, `pricing`, `footer`,
  `sticky-bar`). Section order lives in DESCRIPTION.md, not in file names.
- A section that exists at only one width (e.g. a mobile-only sticky bar) has only that file.
- States: put all states of a section or component into **one** file, switchable through a
  `data-props` enum (`status: empty | loading | ready | error`). If a side-by-side states
  sheet helps, add `<section>-states-<width>.dc.html` and mark it "reference only".

**Split or not?** Split when the deliverable has several stacked sections that can be
built and reviewed independently (rule of thumb: 3+ sections or taller than about 3 screens).
Don't split emails, single components, dialogs or short screens.

## 2. HANDOFF.md: common things only

One per handoff folder, shared by every deliverable. Use the template in
[templates.md](templates.md#handoffmd).

**Rule:** if something applies to only one page, section or component, it goes in that
deliverable's DESCRIPTION.md, not here. HANDOFF.md may **list** deliverables but never
describes them. No component specs, no section layouts, no per-slot image ratios.

Sections, in order:

1. **Intent**: what it is, who it's for, the feel in one sentence ("solid local
   craftsmen, not startup"), the language of the copy, chosen variants and dropped ones
   ("Hero A chosen, don't build Hero B").
2. **Contents**: a table of deliverables (type, split or not, widths) linking to each
   DESCRIPTION.md. Nothing more.
3. **How to read `.dc.html` files**: copy the block from the template verbatim, then
   list this design's props (e.g. `{{accent}}` = `#FF6B1A`).
4. **Layout**: breakpoints, max content width, side padding per breakpoint, grid,
   and the instruction that artboards are fixed-width but the build is fluid.
5. **Design tokens**: tables, each token with a name, a value and *where it's used*:
   - Colour (incl. text-on-colour pairs and state colours: error, success, disabled)
   - Typography (family + fallback, weights, a role × breakpoint scale, letter-spacing)
   - Spacing, radius scale, shadows, minimum tap target, the focus look
   - Motion (durations, easing), if anything moves
   - Iconography (style, stroke, set, "no emoji" if so)
6. **Global rules**: constraints that hold everywhere, e.g. "accent only on call/submit
   buttons; text on it is always dark".
7. **Imagery**: general art direction only (real vs. stock, subject matter, mood, how
   placeholders are labelled).
8. **Placeholders**: every `[bracketed]` value in the copy, so they can be found and filled.

## 3. DESCRIPTION.md: one per deliverable

Use the template in [templates.md](templates.md#descriptionmd). It covers what the
deliverable looks like **and** how it behaves.

- **Split deliverable:** start with a section table in page order
  (`# | Section | Files | Anchor | Background`), then one `##` block per section in the
  same order. End with `## Page behaviour` for anything that spans sections (sticky
  header or bar, scroll-to-anchor, show/hide on scroll).
- **Non-split deliverable:** a single `##` block.

Each block has, where relevant:

1. **Anatomy**: the parts top to bottom, with exact copy.
2. **Layout per width**: what stacks, reorders or resizes between artboards.
3. **Variants**: for components, e.g. type × look × size.
4. **States**: a table: `State | What the user sees | Entered when | Left when`.
5. **Interactions**: triggers and visible responses, timings, motion.
6. **Rules**: constraints for this part only ("phone number visible at every width").
7. **Imagery**: per slot: ratio per width, focal point, what the shot shows.
8. **Edge content**: long text, empty lists, many items, missing image.

For emails, add the inbox preview text, what must stay readable with images off, and
the dark-mode look if designed. Leave rendering-engine hacks to the developer.


## 4. The `.dc.html` mockups

- One artboard per file, at its exact width. Real copy, real tokens, inline styles.
- Keep behaviour in the `<script type="text/x-dc">` block driven by `data-props`
  (enums for states, colours for themeable tokens), so the developer can flip states.
- Use real interactive elements in the mockup where the design has them (`<button>`,
  `<input>`, `<a>`), so intent is unambiguous.
- Placeholders for images: striped/neutral box with a label describing the shot needed.

## 5. What does NOT go in the handoff

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
contrast pairs (as part of the colour tokens), minimum tap-target sizes (as a token),
what the focus state *looks like*, input text size (e.g. 16px so mobile
doesn't zoom), reduced-motion *look* (e.g. "no slide, just appear"), error message copy and
when it appears.

Rewrite, don't delete, when a technical line hides a design decision:

| Instead of | Write |
|---|---|
| "Use IntersectionObserver on the hero CTA" | "The sticky bar appears once the hero call button has scrolled off screen." |
| "`aria-invalid` + `aria-describedby` on the phone field" | "Invalid phone: red 2px border, `#FDF3F2` background, message below: '…'. Shown after leaving the field and on submit, not while typing." |
| "AVIF/WebP with JPG fallback, hero preloaded" | "Hero photo 16:10 desktop / 4:3 mobile, focal point on the crew." |

## 6. Workflow

1. Confirm what's being handed off: deliverables, chosen variants, widths designed.
2. Collect tokens from the mockups — every colour, size and radius used must map to a
   token in HANDOFF.md; unify near-duplicates (`#1F3D2A` vs `#1F3D2B`) with the user.
3. Decide the deliverables (including shared components), split or not, and name the files.
4. Write HANDOFF.md, then a DESCRIPTION.md per deliverable.
5. Re-read HANDOFF.md. Move anything that describes a single page, section or
   component into that deliverable's DESCRIPTION.md.
6. Run the section 5 check over every file: delete or rewrite any line that fails the
   "could a designer verify it by looking?" test.
7. Check every file referenced in the docs exists and every `.dc.html` is referenced.
