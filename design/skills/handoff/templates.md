# Handoff templates

Fill the `<…>` parts, delete sections that don't apply. Keep the headings so every
handoff reads the same.

## HANDOFF.md

````markdown
# Handoff: <what it is, for whom>

<One paragraph: purpose, audience, the feel in one sentence, language of the copy.>

**Chosen variants:** <e.g. Hero A "Ekipa w akcji">. **Dropped — don't build:** <e.g. Hero B>.

---

## 1. What's in this folder

```
design-handoff/
├── HANDOFF.md
├── <deliverable>/          (split)
│   ├── STRUCTURE.md
│   ├── <section>-375.dc.html
│   └── <section>-1440.dc.html
└── <deliverable>/          (single)
    ├── BEHAVIOUR.md
    └── <deliverable>-600.dc.html
```

## 2. How to read the `.dc.html` files

Each file is a design mockup, not production code.
- The real markup sits inside `<x-dc>…</x-dc>`. Styles are inline: read exact sizes,
  colours and spacing from them.
- `{{name}}` is a design prop. Props used here: <`{{accent}}` = `#FF6B1A`, …>.
- `<sc-for>` is a loop, `<sc-if>` is a condition.
- The `<script type="text/x-dc">` block shows the intended behaviour; `data-props`
  lists the switchable states/variants (e.g. `status: empty | uploading | ready`).
  `$preview` gives the artboard size.
- Ignore `support.js`, `<helmet>` wrappers and `hint-*` attributes: they belong to the design tool.
- Artboards are fixed-width. **Build fluid**: see Layout.

## 3. Layout

| Breakpoint | Range | Artboard | Max content width | Side padding | Columns / gap |
|---|---|---|---|---|---|
| Mobile | < 768 | 375 | 100% | 16px | 1 / 12px |
| Desktop | ≥ 768 | 1440 | 1280px | 80px | 12 / 24px |

## 4. Design tokens

### Colour
| Token | Value | Use |
|---|---|---|
| `--<name>` | `#……` | <where> |

Text-on-colour pairs (all ≥ 4.5:1, don't lighten): <`--ink` on `--white`, `--on-green` on `--green-800`, …>

### Typography
Font: **<family>** (<source>, weights …). Fallback: `<stack>`.

| Role | Mobile | Desktop | Weight / tracking |
|---|---|---|---|
| H1 | | | |
| H2 | | | |
| Body | | | |
| Small | | | |
| Eyebrow | | | |

Form input text: <16px>.

### Spacing, shape, elevation
- Section padding: <mobile> / <desktop>
- Gaps: <…>
- Radius: buttons <…>, cards <…>, pills 999px
- Shadows: <token → value → only on …>

### Motion
| What | Duration / easing | Reduced motion |
|---|---|---|

## 5. Components

### Buttons (min height <48px>)
| Type | Look | Height | Used for |
|---|---|---|---|

Focus state: <what it looks like>.

### Form fields
| State | Border | Background | Text / message |
|---|---|---|---|
| Default | | | |
| Focus | | | |
| Error | | | |
| Disabled | | | |

### Icons
<Style, stroke, set, list of icons used. "No emoji" if so.>

## 6. Rules
- <Non-obvious constraint the developer would otherwise break.>

## 7. Imagery
- <Real vs. stock, subject matter, mood.>
- Aspect ratios: <slot → ratio per breakpoint>.
- Placeholders are labelled with the shot needed — they double as the shot list.

## 8. Placeholders to fill in
`[…]` · `[…]` — <where each appears>.
````

## STRUCTURE.md

````markdown
# <Deliverable>: structure

Sections top to bottom. Tokens and components are in [../HANDOFF.md](../HANDOFF.md).

| # | Section | Files | Anchor | Background |
|---|---|---|---|---|
| 1 | Header | `header-375`, `header-1440` | | `--green-900` |
| 2 | Hero | `hero-375`, `hero-1440` | | photo + `--green-800` |

---

## 1. <Section>
Files: `<section>-375.dc.html`, `<section>-1440.dc.html`

**Layout**
- Mobile: <stacking, sizes that matter>
- Desktop: <columns, sizes that matter>

**Content** (top to bottom)
- <element> — "<exact copy>"

**Behaviour**
- <trigger> → <what the user sees>

<If the section has states:>
| State | What the user sees | Entered when | Left when |
|---|---|---|---|

---

## Page behaviour
- <Sticky header/bar: when it appears and disappears, at which widths, how it moves.>
- <Anchor links: which buttons scroll where.>
````

## BEHAVIOUR.md

````markdown
# <Deliverable>: behaviour

File(s): `<deliverable>-<width>.dc.html`. Tokens and components are in [../HANDOFF.md](../HANDOFF.md).

## Anatomy
1. <part> — "<copy>"

## Layout per width
- <width>: <what differs>

## States
| State | What the user sees | Entered when | Left when |
|---|---|---|---|
| default | | | |
| hover / pressed | | | |
| loading | | | |
| error | | | |
| empty | | | |

## Interactions
- <trigger> → <visible response, timing, motion>

## Edge content
- Long text: <truncate after 2 lines with … / wrap>
- Empty: <…>
- Many items: <…>
- Missing image: <…>

<Email only:>
## Email specifics
- Inbox preview text: "<…>"
- With images off: <what must stay readable>
- Dark mode: <look, if designed>
````
