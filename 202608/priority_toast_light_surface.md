---
tier: tale
title: Give the priority toast its own light-mode-native surface
goal:
  The Ctrl+Shift+P priority toast renders as a themed, elevated card that is legible and attractive in Obsidian's light
  theme (and still in dark), instead of inheriting Obsidian's hardcoded near-black notice bubble and painting
  dark-on-dark text over it.
proposed_by: bbugyi200.athena.sj.f0.f0
create_time: 2026-08-03 07:32:49
status: done
---

- **PROMPT:**
  [prompts/202608/priority_toast_light_surface.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/priority_toast_light_surface.md)
- **AGENTS:**
  - [bbugyi200.athena.sj.f0.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.sj.f0.f0.md)
- **COMMITS:**
  - [816b6fb](https://github.com/bobs-org/bob-plugins/commit/816b6fb6371148e6bffbb3558d5fa4850d4ecaa0) —
    fix(navigation-hotkeys): resurface priority toast

# Plan: Give the priority toast its own light-mode-native surface

## Goal

Make the priority write toast look like it belongs in Obsidian's light theme. Today the card's text, pills, and chips
are drawn with tokens tuned for a light background but painted onto Obsidian's near-black notice bubble, so the toast
reads as a muddy dark slab against a white vault. After this change the toast is an elevated card on
`--background-primary` with a level-hued rail, correct contrast in both themes, and no effect whatsoever on any other
Obsidian or plugin notice.

## Tier and repositories

This is a **tale**: one follow-up coding agent can implement, test, and deploy it atomically. It is a stylesheet-only
change plus one regression test and a version bump — no JavaScript behavior, no model change, no write-path change.

All work lives in the linked `bob-plugins` repository. Open it through the required workflow and use the printed path
for every read, write, and command:

```text
sase repo open bob-plugins -r "Fix priority toast light-mode surface and contrast"
```

Files:

- `plugins/bob-navigation-hotkeys/styles.css` — the entire visual change (the `/* --- Priority notice --- */` section at
  the end of the file, currently lines 871–1047).
- `scripts/test-navigation-hotkeys.cjs` — one new stylesheet-invariant test.
- `plugins/bob-navigation-hotkeys/manifest.json` and the repo-root `README.md` — patch version bump.

The primary `bob-cli` repository needs **no** change: `docs/projects.md` describes what the toast _reports_ (level,
field, ISO date, weekday, relative distance, chips), not how it is painted, and none of that vocabulary changes.

## Diagnosis: why it looks dark

This is not a subjective preference miss — the current CSS is authored against the wrong background. Verified against
the installed Obsidian's own stylesheet (`/snap/obsidian/65/app/resources/obsidian.asar`):

```css
body {
  --background-modifier-message: color-mix(in oklch, black 90%, transparent);
}
.notice {
  background-color: var(--background-modifier-message);
  color: #fafafa;
  padding: 0.75em 1em;
  max-width: 300px;
  border-radius: var(--radius-m);
  box-shadow: 0 2px 8px var(--background-modifier-box-shadow);
  white-space: pre-wrap;
}
```

Two facts drive everything below:

1. **`--background-modifier-message` is 90% black in _both_ desktop themes.** The only override in the whole app is
   `.is-mobile.theme-dark`. So an Obsidian notice is a near-black bubble with hardcoded `#FAFAFA` text even when the
   user is in light mode. That is fine for a one-line plain-text notice; it is hostile to a structured card.
2. **Our card overrides that text color with app-background tokens.** `.bob-nh-notice` sets `color: var(--text-normal)`,
   which in the light theme is `#222222`. Every other token we chose — `--text-muted`, `--text-accent`,
   `--background-modifier-border`, `--color-orange/yellow/blue` — is likewise defined by Obsidian for use on
   `--background-primary`.

The result in light mode, measured against the actual near-black bubble and the light-theme token values
(`--text-normal: #222222`, `--text-muted: #5c5c5c`, `--background-modifier-border: #e4e4e4`, `--color-yellow: #e0ac00`,
`--color-orange: #ec7500`, `--color-green: #08b94e`, `--color-blue: #086ddd`):

| Element                                 | Light-mode paint            | Problem                    |
| --------------------------------------- | --------------------------- | -------------------------- |
| card body text                          | `#222222` on ~`#1a1a1a`     | effectively invisible      |
| muted label / weekday / receipt         | `#5c5c5c` on ~`#1a1a1a`     | ~1.5:1, unreadable         |
| ISO date chip                           | `#e4e4e4` @45% over black   | murky gray smear           |
| level pill (`color-mix(hue 80%, #222)`) | `#B88F05` (yellow) on black | dark gold on black, ~2.5:1 |
| `is-ok` chip                            | `#08b94e` on black          | saturated green on black   |

There is no per-declaration patch that fixes this. The card was designed for a light surface and needs to be _put on
one_.

There is also a second-order symptom worth fixing while here: `.notice { max-width: 300px }` squeezes a three-row card
with a right-aligned `[priority:: high]` receipt, so the header wraps almost always and the deliberate right-alignment
never renders.

## Design

### Principle: own the surface, then let the tokens be right

The single decision that fixes the whole card is to stop painting on Obsidian's message bubble and give the toast the
same surface Obsidian gives its modals: `--background-primary`, a hairline `--background-modifier-border`, and
`--shadow-s`. Every token already in the stylesheet was chosen for exactly that surface, so most of the file becomes
correct the moment the background flips. What remains is a contrast re-tune of the saturated hues, which were tuned for
a dark bubble.

Chosen surface is `--background-primary` rather than `--background-secondary` because the card's pills, chips, and ISO
receipt are all `color-mix(..., transparent)` fills that composite against it: over `#ffffff` they read as crisp pastel
tints; over `#f6f6f6` they lose their edge and the ISO chip in particular stops looking like a chip. The 1px border plus
`--shadow-s` is what separates a white card from a white note — the border is load-bearing, not decorative.

### Mechanism: re-surface only our own notices

The card is a child of `.notice`, so the surface must be set on the parent. Use `:has()`:

```css
.notice:has(.bob-nh-notice) { … }
```

This is safe and precise here:

- **Support is guaranteed.** `manifest.json` declares `minAppVersion: 1.8.7`; that Obsidian ships Chromium far past
  `:has()`'s Chromium 105 baseline.
- **It cannot leak.** The selector only matches a notice that contains our card. Plain `scheduled`, `dependsOn`, scalar,
  delete, and every other plugin's notice keep Obsidian's stock bubble untouched. A stylesheet test below pins this
  invariant.
- **No JavaScript is involved**, so the `showPriorityNotice()` fallback seam, its `try`/`catch`, the single-notice
  guarantee, `aria-label`, and the user's configured notice duration are all untouched, and the 273-test suite keeps
  passing without edits.
- Obsidian injects no wrapper between `.notice` and an appended fragment (there is no `notice-content` element in the
  app stylesheet), so `padding: 0` on `.notice` is sufficient for the card to fill the bubble edge to edge.

**Degradation.** `.bob-nh-notice` additionally paints its own `background-color: var(--background-primary)`,
`border-radius: var(--radius-m)`, and `color: var(--text-normal)`. If the `:has()` rule ever fails to apply — a future
Obsidian change, or a theme forcing `.notice` padding — the worst case is a legible light card sitting inside a dark
frame, never dark-on-dark text again.

**Do not comma-join the `:has()` selector with any non-`:has()` selector.** An unsupported selector invalidates an
entire selector list, which would silently take the other rules down with it. Keep it in its own block.

### The card

```text
┌──────────────────────────────────────────────┐
│▍ ⌁ P2                       [priority:: high]│   4px level rail, flush to the card edge
│                                              │
│  ⚄ scheduled                      in 3 days  │   "in 3 days" is the largest text on the card
│    2026-08-06 · Thu                          │   mono ISO receipt chip
│                                              │
│  ( Blocked )                                 │
└──────────────────────────────────────────────┘
```

Layout and structure are unchanged from the approved current design — the DOM that `renderPriorityNoticeFragment()`
builds is not touched. What changes is the surface, the contrast, and three hierarchy adjustments:

1. **The rail runs the full card edge.** With `.notice { padding: 0 }` and `overflow: hidden` on the notice, the card's
   `border-left` sits flush against the bubble's rounded edge and is clipped by its radius — the classic toast rail,
   instead of today's floating 3px stub inset by the notice's own padding. Widen it 3px → 4px and keep it the one place
   the pure, unmixed level hue lives.
2. **Hierarchy by size, not by hue.** `in N days` moves from `--font-ui-smaller` to `--font-ui-small`, staying semibold.
   It becomes the largest text on the card, so it reads first regardless of the user's accent color or theme. This is
   more reliable than emphasis-by-color, since `--text-accent` is user-configurable.
3. **The dice icon goes muted.** It is currently level-hued, which makes it compete with the level pill and the rail.
   Muting it leaves exactly two hue-carrying elements in the header — rail and pill — and lets the date row read as one
   quiet label.

Notice width goes `300px` → `min(380px, calc(100vw - 40px))`. At 380px the header's `[priority:: high]` receipt finally
right-aligns on one line as designed, and the `calc()` keeps a narrow desktop window or mobile from clipping.

### Contrast re-tune

Obsidian's saturated light-theme hues fail as small text on white: `--color-yellow` `#e0ac00` is 1.9:1, `--color-green`
`#08b94e` is 2.2:1, `--color-orange` `#ec7500` is 3.0:1. The existing
`color-mix(in srgb, <hue> 80%, var(--text-normal))` idiom is the right tool — it darkens toward text in light mode and
lightens toward text in dark mode — but 80% was tuned against the dark bubble and is too weak on white. At **55%** the
worst case clears WCAG AA:

| Tone                       | Light hue | Raw on `#ffffff` | Mixed 55% with `#222222` | Contrast |
| -------------------------- | --------- | ---------------- | ------------------------ | -------- |
| `--color-yellow` (level 1) | `#e0ac00` | 1.9:1            | `#8b6e0f`                | ~4.9:1   |
| `--color-orange` (level 0) | `#ec7500` | 3.0:1            | `#915013`                | ~6.2:1   |
| `--color-blue` (level 2)   | `#086ddd` | 4.9:1            | `#134b89`                | ~8.7:1   |
| `--color-green` (`is-ok`)  | `#08b94e` | 2.2:1            | `#13753a`                | ~5.8:1   |

In dark mode the card lands on `#1C1C1C`, where the raw hues are already excellent, and 55% needlessly desaturates them
(dark `--color-orange` `#e9973f` mixes to a flat tan). So override the text mix to **85%** under `.theme-dark`. This
introduces the repo's first `.theme-dark` selector; that is the standard Obsidian mechanism and is preferable to
guessing one ratio that is mediocre in both themes.

Do **not** parameterize the mix percentage through a custom property (`color-mix(… var(--p), …)`); keep percentages
literal and vary the whole declaration per theme, so the value parses unambiguously.

### Stylesheet structure cleanup

The four `.bob-nh-notice-chip.is-*` blocks plus `.bob-nh-notice-level` and `.bob-nh-notice-count` are six near-identical
color triples today, and the re-tune would otherwise duplicate the mix ratio six times and again per theme. Collapse
them onto a single tone variable so the contrast lever is one knob:

```css
.bob-nh-notice .bob-nh-notice-level,
.bob-nh-notice .bob-nh-notice-count,
.bob-nh-notice .bob-nh-notice-chip {
  --bob-nh-tone: var(--text-muted);
  /* …existing pill geometry… */
  color: color-mix(in srgb, var(--bob-nh-tone) 55%, var(--text-normal));
  border: 1px solid color-mix(in srgb, var(--bob-nh-tone) 45%, transparent);
  background-color: color-mix(in srgb, var(--bob-nh-tone) 12%, transparent);
}

.bob-nh-notice .bob-nh-notice-level {
  --bob-nh-tone: var(--bob-nh-level-color);
  font-weight: var(--font-semibold);
}
.bob-nh-notice .bob-nh-notice-chip.is-warn {
  --bob-nh-tone: var(--color-orange);
}
.bob-nh-notice .bob-nh-notice-chip.is-ok {
  --bob-nh-tone: var(--color-green);
}
.bob-nh-notice .bob-nh-notice-chip.is-info {
  --bob-nh-tone: var(--text-accent);
}

.theme-dark .bob-nh-notice .bob-nh-notice-level,
.theme-dark .bob-nh-notice .bob-nh-notice-count,
.theme-dark .bob-nh-notice .bob-nh-notice-chip {
  color: color-mix(in srgb, var(--bob-nh-tone) 85%, var(--text-normal));
}
```

`--bob-nh-tone` defaults to `--text-muted`, which is exactly what the count pill and `is-muted` chip want, so the
`.bob-nh-notice-count` and `.is-muted` blocks disappear entirely. The tone variable resolves per element, so the single
`.theme-dark` rule correctly re-mixes all six.

### Rule-by-rule specification

Everything below stays inside the existing `/* --- Priority notice --- */` section, stays scoped under `.bob-nh-notice`
(or `.notice:has(.bob-nh-notice)`), uses Obsidian variables and the repo's `color-mix(...)` idiom only, and adds no
literal colors, no animation, and no fixed heights.

**New — the surface:**

```css
.notice:has(.bob-nh-notice) {
  padding: 0;
  max-width: min(380px, calc(100vw - 40px));
  border: 1px solid var(--background-modifier-border);
  border-radius: var(--radius-m);
  background-color: var(--background-primary);
  box-shadow: var(--shadow-s);
  color: var(--text-normal);
  overflow: hidden;
  white-space: normal;
}
```

**Changed — `.bob-nh-notice`:** replace `padding-left: 10px` with `padding: 10px 12px 10px 11px`; `border-left` 3px →
4px; add `border-radius: var(--radius-m)` and `background-color: var(--background-primary)` (degradation); `gap: 6px` →
`7px`. Keep `--bob-nh-level-color`, the `is-level-*` hue map, `box-sizing`, `max-width`, `color`, and `white-space` as
they are.

**Changed — pills and chips:** as in the block above.

**Changed — icons:** `.bob-nh-notice-icon` keeps a level hue but mixed for legibility
(`color-mix(in srgb, var(--bob-nh-level-color) 70%, var(--text-normal))`); split `.bob-nh-notice-date-icon` out to
`color: var(--text-muted)`.

**Changed — `.bob-nh-notice-relative`:** `font-size: var(--font-ui-small)`; keep `var(--font-semibold)`; color becomes
`color-mix(in srgb, var(--text-accent) 75%, var(--text-normal))`, with a `.theme-dark` override back to raw
`var(--text-accent)`. The mix guards against a user-chosen light accent hue disappearing on white.

**Changed — `.bob-nh-notice-date-iso`:** border becomes solid `var(--background-modifier-border)` and background
`color-mix(in srgb, var(--background-modifier-border) 55%, transparent)`, so it reads as a chip on white instead of a
faint wash. Keep the monospace font, `tabular-nums`, `overflow-wrap: anywhere`, and `var(--text-normal)`.

**Unchanged:** `.bob-nh-notice-header`, `.bob-nh-notice-chips`, `.bob-nh-notice-date`, `-date-heading`, `-date-receipt`,
`-receipt`, `-date-label`, `-date-arrow`, `-date-separator`, `-date-weekday`, and the `@media (max-width: 420px)`
receipt-indent rule. These are layout or `--text-muted`, both of which are already correct once the surface flips.

## Implementation order

1. Add the `.notice:has(.bob-nh-notice)` surface rule and the `.bob-nh-notice` self-paint/padding/rail changes. Deploy
   and eyeball once — everything after this is refinement on a now-correct background.
2. Collapse the pill/chip color rules onto `--bob-nh-tone` at 55%, delete the now-redundant `.bob-nh-notice-count` and
   `.is-muted` color blocks, and add the single `.theme-dark` override.
3. Apply the hierarchy adjustments: relative-text size, icon colors, ISO chip surface.
4. Add the stylesheet-invariant test.
5. Bump the version and README row.

## Automated verification

No JavaScript changes, so the entire existing suite must pass **untouched** — that is itself the primary regression
signal for the notice pipeline, the model, the fallback, and the three write paths.

Add one new test to `scripts/test-navigation-hotkeys.cjs` (which already requires `node:fs` and `node:path`). It reads
`plugins/bob-navigation-hotkeys/styles.css` and pins the two invariants a stylesheet cannot otherwise defend:

- **Scoping.** Every selector in the file that references Obsidian's `.notice` class — matching `.notice` on a class
  boundary, so `.bob-nh-notice*` does not count — must also contain `.bob-nh-notice`. This is the guard against a future
  edit restyling every notice in the user's Obsidian.
- **No literal colors.** The priority-notice section (from the `/* --- Priority notice` marker to end of file) must
  contain no `#rrggbb`, `rgb(`, or `hsl(` literal, enforcing the repo's Obsidian-variables-only convention that makes
  the toast theme-portable in the first place.

Run from the `bob-plugins` checkout:

```text
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

The suite is 273 tests today and must be 274 after this, with `npm run validate` at 6/6.

## Release and deploy

- `plugins/bob-navigation-hotkeys/manifest.json`: `1.15.1` → `1.15.2`. This is a presentation fix, not a new capability,
  so patch is correct.
- Repo-root `README.md`: update the version cell in the Bob Navigation Hotkeys row. The row's description already covers
  what the toast reports and needs no wording change.
- Commit the scoped diff (four files) through the normal SASE commit workflow.
- Deploy from the `bob-plugins` checkout:

```text
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD"
```

The explicit `-r "$PWD"` is required in a SASE workspace. If sync reports the vault plugin dirty, compare it against the
committed source before considering any force option. Report what was copied and tell the user to reload Bob Navigation
Hotkeys in Obsidian.

## Manual verification (for the user, after reload)

Pixels cannot be checked from a headless agent; the automated tests cover scoping and conventions, not appearance.

1. **Light mode, ordinary task:** `Ctrl+Shift+P` → `priority` → `P2`. The toast is a light card, not a dark slab: white
   surface, hairline border, soft shadow, a colored rail down the full left edge, and every string legible.
2. **Each level:** P2/P3/P4 in turn — confirm the orange, yellow, and blue pills are all comfortably readable on white
   (yellow is the one that was failing).
3. **Hierarchy:** `in N days` is visibly the largest, strongest element; the `YYYY-MM-DD` receipt is clearly secondary
   but crisp; the `[priority:: …]` receipt is right-aligned on one line.
4. **Counted and project:** `3<Ctrl+Shift+P>` → `P4` shows both endpoint dates and a span; a `^prj` task shows
   `scheduled (project)`.
5. **Scoping — the important one:** `Ctrl+Shift+P` → `scheduled` (a plain, non-priority write). Its toast must still be
   Obsidian's ordinary dark bubble. If it turned light, the `:has()` scoping leaked.
6. **Dark mode:** switch themes and repeat step 1 — the card should read as an elevated panel and the hues should stay
   vivid, not washed out.
7. **Narrow window:** shrink Obsidian until the notice is near its width cap — nothing clipped, ISO digits wrap cleanly,
   the rail stays flush.

## Out of scope

- **Converting the other notices to cards.** Every non-priority notice keeps Obsidian's stock bubble. Mixing one light
  card among dark toasts is a deliberate call — the priority toast is a structured card and the others are one-line
  sentences — and extending the treatment is a separable follow-up.
- **Any JavaScript change.** No edits to the model, `formatPriorityNoticeText()`, `renderPriorityNoticeFragment()`'s
  DOM, `showPriorityNotice()`'s fallback, `aria-label`, notice count, or notice duration.
- **What gets written to notes.** No change to rolled dates, the `[priority:: …]` mapping, or Blocked/recovery behavior.
- **Re-picking the level hues.** The orange/yellow/blue map matches the sibling picker surfaces this epic already
  shipped; only its rendered contrast changes here.
- **Primary-repo docs.** `docs/projects.md` describes the toast's content, which is unchanged.
