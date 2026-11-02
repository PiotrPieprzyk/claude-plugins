# Handoff templates

Fill the `<…>` parts and delete sections that don't apply. Keep the headings so every
handoff reads the same.

## HANDOFF.md

Only what is **common to every deliverable**. Don't describe any page, section or
component here: list them, and link to their `DESCRIPTION.md`.

````markdown
# Handoff: <what it is, for whom>

<One paragraph: purpose, audience, the feel in one sentence, language of the copy.>

**Chosen variants:** <e.g. Hero A "Ekipa w akcji">. **Dropped, don't build:** <e.g. Hero B>.

---

## 1. Contents

| Deliverable | Type | Split | Widths | Description |
|---|---|---|---|---|
| `landing/` | page | yes | 375, 1440 | [DESCRIPTION.md](landing/DESCRIPTION.md) |
| `button/` | component | no | 375 | [DESCRIPTION.md](button/DESCRIPTION.md) |
| `welcome-email/` | email | no | 600 | [DESCRIPTION.md](welcome-email/DESCRIPTION.md) |

## 2. How to read the `.dc.html` files

Each file is a design mockup, not production code.
- The real markup sits inside `<x-dc>…</x-dc>`. Styles are inline, so read exact sizes,
  colours and spacing from them.
- `{{name}}` is a design prop. Props used here: <`{{accent}}` = `#FF6B1A`, …>.
- `<sc-for>` is a loop and `<sc-if>` is a condition.
- The `<script type="text/x-dc">` block shows the intended behaviour. `data-props`
  lists the states and variants you can switch (e.g. `status: empty | uploading | ready`).
  `$preview` gives the artboard size.
- Ignore `support.js`, `<helmet>` wrappers and `hint-*` attributes. They belong to the design tool.
- Artboards are fixed-width. **Build fluid layouts** (see Layout).

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

Text-on-colour pairs (all ≥ 4.5:1, don't lighten them): <`--ink` on `--white`, …>

### Typography
Font: **<family>** (<source>, weights …). Fallback: `<stack>`.

| Role | Mobile | Desktop | Weight / tracking |
|---|---|---|---|
| H1 | | | |
| H2 | | | |
| Body | | | |
| Small | | | |
| Eyebrow | | | |

### Spacing, shape, elevation
- Section padding: <mobile> / <desktop>
- Gaps: <…>
- Radius: <scale, e.g. sm 8 · md 12 · lg 16 · pill 999>
- Shadows: <token → value>
- Minimum tap target: <48px>
- Focus look: <e.g. 3px `--accent` outline, 2px offset>

### Motion
| Token | Duration / easing | Reduced motion |
|---|---|---|

### Iconography
<Style, stroke, caps, set (e.g. Lucide). "No emoji" if so.>

## 5. Global rules
- <A constraint that applies everywhere, e.g. "accent only on call/submit buttons; text on it is always dark".>

## 6. Imagery
- <Real vs. stock, subject matter, mood.>
- Placeholders are labelled with the shot needed, so together they work as a shot list.

## 7. Placeholders to fill in
`[…]` · `[…]`: every bracketed value used in the copy.
````

## DESCRIPTION.md

One per deliverable. Describes **this** page, email or component, and nothing shared.
Split deliverables use part A for each section. A non-split deliverable uses part A once, for the whole thing.

````markdown
# <Deliverable>

<One line: what it is and where it's used.> Tokens and global rules: [../HANDOFF.md](../HANDOFF.md).

<Split only:>
| # | Section | Files | Anchor | Background |
|---|---|---|---|---|
| 1 | Header | `header-375`, `header-1440` | | `--green-900` |
| 2 | Hero | `hero-375`, `hero-1440` | | photo + `--green-800` |

---

## <n>. <Section / deliverable name>
Files: `<name>-375.dc.html`, `<name>-1440.dc.html`

### Anatomy
1. <part>: "<exact copy>"

### Layout per width
- 375: <stacking, sizes that matter>
- 1440: <columns, sizes that matter>

### Variants
| Variant | Look | Size | Used for |
|---|---|---|---|

### States
| State | What the user sees | Entered when | Left when |
|---|---|---|---|
| default | | | |
| hover / pressed | | | |
| loading | | | |
| error | | | |
| empty | | | |

### Interactions
- <trigger> → <visible response, timing, motion>

### Rules
- <A constraint for this part only, e.g. "phone number visible at every width".>

### Imagery
- <slot>: <ratio per width, focal point, what the shot shows>

### Edge content
- Long text: <wrap / truncate after 2 lines with …>
- Empty / many items / missing image: <…>

---

<Split only:>
## Page behaviour
- <Behaviour that spans sections: sticky header or bar (when it appears and disappears, at which widths, how it moves), anchor links.>

<Email only:>
## Email specifics
- Inbox preview text: "<…>"
- With images off: <what must stay readable>
- Dark mode: <look, if designed>
````
