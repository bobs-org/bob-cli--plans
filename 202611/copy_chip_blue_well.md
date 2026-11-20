---
tier: tale
title: Copy chips you can spot at a glance
goal:
  Give every copy chip and bob-copy card a solid blue copy well, one look in every view,
  and a hover tooltip.
size: medium
decisions:
  hue:
    ask: Which hue should mark every copy chip and bob-copy card?
    choices:
      blue:
        Dedicated copy hue var(--color-blue), distinct from links, run chips, code, and
        task rows
      accent:
        Theme accent var(--interactive-accent); copy chips then share a hue with links
        and run chips
    default: blue
    why:
      In the vault the accent already means links and run chips; blue is free on a task
      row
    answer: blue
  glossary_copy_chip:
    ask: Should the glossary Copy Chip strand stop calling the inline chip a quiet pill?
    memory:
      - glossary:copy_chip
    default: true
    corrects: plugin renders the span as a quiet pill and the fence as a titled card.
    answer: true
proposed_by: bbugyi200.apollo.fa
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 12:59:10
status: wip
---

# Plan: Copy chips you can spot at a glance — the blue copy well

## Goal

Make every copy chip (`copy:` / `copy[LABEL]:` spans and `bob-copy` fences, rendered by
the `bob-chips` plugin in the `bob-plugins` linked repo) unmistakable at a glance in
Obsidian. It must also be intuitive (hovering tells you what you'll copy), reliable (one
look in every view; no reflow; no stranded glyph), and beautiful in light and dark
themes. Behavior, gestures, grammar, payloads, and the exactness contract do not change.
This is a look and discoverability change.

Design mockups, light and dark (before, after, states, card, accent alternative):
`file:explicit:cf07ac247aed970f3efaa84f` and `file:explicit:848b252dda32714e1c868bbf`.
View them with `sase artifact open <ref>`.

## Why chips are hard to spot today (diagnosis)

All of this was measured against the current `plugins/bob-chips/styles.css` (0.13.0) and
Obsidian 2.1.2's app CSS:

1. **Inverted hierarchy.** The vault's `inline-code` snippet gives plain code a cyan
   wash, a 42% cyan border, and cyan-tinted ink. A copy chip gets
   `--background-secondary` (#f6f6f6 on white), a hairline
   `--background-modifier-border`, and a `--text-faint` glyph. Inert code is louder than
   the one clickable thing, and the only copy cue is the faintest ink on the line.
2. **Lost on tinted rows.** On the Blocked, Next, and WIP row washes, the chip reads as
   a pale gray box (see the gtd_daily screenshot rows).
3. **Two looks for one thing.** In Reading view, embeds, hover previews, and
   Tasks/Dataview results, a chip is a `<button>`. Obsidian's
   `button:not(.clickable-icon) { background-color: var(--interactive-normal); box-shadow: var(--input-shadow) }`
   has specificity (0,1,1), which beats `.bob-copy-chip` at (0,1,0). So those chips
   render as white, shadowed form buttons, while Live Preview spans are flat gray pills.
4. **The documented tooltip never shows.** `docs/bob-chips.md` promises "Copy:
   <preview>" on hover. But `.bob-copy-chip-tip { display: none }` has no reveal rule,
   so hovering a labeled chip like **summary prompt** never says what it copies.
5. **Stranded glyph.** Chromium may break the line between the glyph and the face. A
   width sweep split them in 652 of 3,636 layouts. Once the glyph is a solid colored
   well, a lone well at a line end would look broken.

## The design: a blue copy well

Every inline copy chip becomes a two-part capsule:

- a solid **well** in the copy hue, holding the white Lucide `copy` glyph: the
  pre-attentive beacon;
- a lightly tinted **body**, holding the face in hue-tinted ink: calm and readable.

`copy` gets one identity hue (decision **hue**, default `var(--color-blue)`). On a
gtd_daily row that hue is free: links and command chips use the accent (purple), inline
code is cyan, task lanes are red/green/yellow, and success is green. The `bob-copy` card
and the authoring card's icon echo the same well, so "blue well = copy" holds
everywhere.

> [!decision] hue = accent Use `var(--interactive-accent)` wherever this plan says
> `var(--color-blue)`. Every other rule is unchanged. Docs then say "the theme accent"
> instead of "blue".

### Tokens (inline chip)

Express every color through one custom property, so states change only the ink:

| Part              | Rule                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ink               | `--bob-copy-chip-ink: var(--bob-copy-chip-hue, var(--color-blue))` set on the chip. `--bob-copy-chip-hue` is the one user knob (a snippet may set it on `body`); styles.css never sets it.                                                                                                                                                                                                                                                                                                                  |
| Body fill         | `color-mix(in srgb, var(--bob-copy-chip-ink) 10%, var(--background-primary))`. Opaque, so row washes never muddy it.                                                                                                                                                                                                                                                                                                                                                                                        |
| Border            | `1px solid color-mix(in srgb, var(--bob-copy-chip-ink) 50%, var(--background-primary))`, radius `var(--radius-s)`                                                                                                                                                                                                                                                                                                                                                                                           |
| Face ink          | `color-mix(in srgb, var(--bob-copy-chip-ink) 30%, var(--text-normal))`. Labeled faces use `font-weight: 500` in the text font; literal faces stay monospace at normal weight.                                                                                                                                                                                                                                                                                                                               |
| Geometry          | `padding: 0.1em 0.5em 0.12em`, `font-size: 0.92em`, `line-height: inherit`, `vertical-align: baseline`, `box-decoration-break: clone` (+ `-webkit-`). Vertical padding never grows the line box (inline element).                                                                                                                                                                                                                                                                                           |
| Well              | `.bob-copy-chip > .bob-copy-chip-glyph`: `display: inline`; `margin-inline: -0.5em 0.42em` (bleeds into the chip's start padding up to the border); `padding: 0.1em 0.3em 0.12em` (the same vertical padding as the chip, so its box matches the body's exactly); start-side radii `calc(var(--radius-s) - 1px)`; `background-color: var(--bob-copy-chip-ink)`; `color: var(--text-on-accent)`. It also resets the shared base rule's `inline-flex` / `width` / `min-width` / `margin-right` / faint color. |
| Glyph             | `.bob-copy-chip > .bob-copy-chip-glyph svg`: `0.95em` square, `vertical-align: -0.13em`; set `--icon-stroke: 2.25px` on the well so Obsidian's `.svg-icon` stroke is bolder at this size. The well's width is padding plus a fixed-size svg, so a copy → check → alert swap never reflows.                                                                                                                                                                                                                  |
| No stranded glyph | Host `white-space: nowrap`; `.bob-copy-chip-face { white-space: normal; overflow-wrap: anywhere }`. Prototyped in Chromium: 0 splits in 3,636 layouts, and long faces still wrap inside the face. A U+2060 word joiner does **not** fix it.                                                                                                                                                                                                                                                                 |
| Cursor            | `cursor: copy`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

### States

No state changes border width, padding, font weight, or well size. Only colors and the
glyph change.

| State                                | Look                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rest                                 | As above                                                                                                                                                                                                                                                                                                                    |
| Hover                                | Fill at 17%; border at full ink. A 120 ms `background-color` / `border-color` transition, dropped under reduced motion.                                                                                                                                                                                                     |
| Active (pressed)                     | Fill at 24%                                                                                                                                                                                                                                                                                                                 |
| Focus-visible                        | `outline: 2px solid var(--bob-copy-chip-ink); outline-offset: 2px; box-shadow: none` (this neutralizes Obsidian's `button:focus-visible` ring)                                                                                                                                                                              |
| `is-copied` (~1.2 s, existing timer) | `--bob-copy-chip-ink: var(--color-green)`: green well with the white `check`, green-tinted body and border. Plus the existing pulse, with its keyframes changed to **from-only** (`from { background-color: var(--bob-yank-flash-bg, rgba(80, 200, 120, 0.35)) }`), so it eases back to whatever the element's own fill is. |
| `is-failed` (~2 s, existing timer)   | `--bob-copy-chip-ink: var(--text-error)`: red well with the white `alert-triangle`. The Notice behavior is unchanged.                                                                                                                                                                                                       |
| Reduced motion                       | No pulse and no transitions; the glyph swap and state colors stay                                                                                                                                                                                                                                                           |
| Print / PDF                          | Plain inline code exactly as today: no border, background, padding, well, tooltip, or weight; monospace face. Use a `[class]` selector (as the run-chip print block does) so no state rule can leak into print.                                                                                                             |

### Tooltip (intuitive)

Reveal the existing `.bob-copy-chip-tip` on `:hover` and `:focus-visible`, with the same
recipe as `.bob-run-chip-tip`:

- absolutely positioned above the chip (`bottom: calc(100% + 6px)`, `left: 0`);
- interface font at `--font-ui-smaller`, `font-weight: normal`, `white-space: normal`;
- max width `min(36em, 80vw)`, `--shadow-s`, `pointer-events: none`,
  `z-index: var(--layer-tooltip)`;
- shown after the same 0.4 s visibility delay. Give it its own keyframes name or reuse
  the run chip's; never edit the run-chip section to do it.

The chip gets `position: relative`. Keep the tooltip **text** exactly as built today:

- labeled: `Copy: ` plus the 120-char `⏎` preview;
- unlabeled: `Click to copy`;
- Live Preview: plus ` · Alt+Click to edit`.

Existing tests pin that text. In `src/040-chip-dom.js`, give the tip an id
(`bob-copy-chip-tip-<seq>`) and `role="tooltip"`, and point the host's
`aria-describedby` at it (as command chips already do). Ids are not payloads, so the
no-payload-in-attributes contract holds. A tooltip may clip at the top edge of the
editor scroller on the very first visible line; command chips share that accepted limit.

### One look in every view (reliable)

Live Preview spans and Reading/embed/hover/Tasks/Dataview buttons must compute identical
fill, border, shadow, ink, padding, font, and outline at rest, hover, active,
focus-visible, copied, and failed. Every copy-chip rule must therefore beat these
Obsidian rules (extracted verbatim from Obsidian 2.1.2):

```css
button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: var(--text-color);
  font-size: var(--font-ui-small);
  border-radius: var(--button-radius);
  border: 0;
  padding: var(--size-4-1) var(--size-4-3);
  height: var(--input-height);
  font-weight: var(--input-font-weight);
  cursor: var(--cursor);
  font-family: inherit;
  outline: none;
  user-select: none;
  white-space: nowrap;
  transition: var(--button-transition);
}
button:not(.clickable-icon) {
  color: var(--text-color);
  background-color: var(--interactive-normal);
  box-shadow: var(--input-shadow);
}
@media (hover: hover) {
  button:hover {
    background-color: var(--interactive-hover);
    box-shadow: var(--input-shadow-hover);
  }
}
button:focus-visible {
  box-shadow: 0 0 0 3px var(--background-modifier-border-focus);
}
button:not(.clickable-icon).mobile-tap {
  background-color: var(--interactive-hover);
  box-shadow: var(--input-shadow-hover);
}
```

A workable approach (the implementer may choose another that the checks below prove):

- Base rules on `:is(span, button).bob-copy-chip` (0,1,1). It ties with Obsidian's rule
  and wins on source order, because plugin CSS loads after app CSS.
- State rules such as `:is(span, button).bob-copy-chip:hover` reach (0,2,1).
- An explicit `button.bob-copy-chip.mobile-tap` rule covers the tap state.
- Set `height: auto`, `box-shadow: none`, `font: inherit`, and `text-align: inherit`
  explicitly.

### The `bob-copy` card and the authoring card

The card keeps its structure, behavior, fold, and text, and takes the same identity:

- **Frame:** border `color-mix(in srgb, ink 35%, var(--background-modifier-border))`,
  background `var(--background-primary)`, so the body reads like paper.
- **Header:** fill at 8% ink; bottom border at 25% ink.
- **Header glyph:** `.bob-copy-fence-header .bob-copy-chip-glyph` becomes a solid
  rounded-square well (`1.5em`, `var(--radius-s)`, ink background, `--text-on-accent`
  glyph).
- **Title:** face ink, weight 600.
- **Copy button** (`.bob-copy-fence-copy`) takes the chip body recipe: 12% fill, 50%
  border, face ink, weight 500, `cursor: copy`. Hover is 17% fill with a full-ink
  border; focus-visible is an ink outline. It must also beat Obsidian's button rules.
  Keep its fixed `min-width` so "Copied ✓" / "Copy failed" never reflow the header.
- **`is-copied`** swaps the card ink to green; **`is-failed`** swaps it to
  `--text-error`. The header pulse uses the from-only keyframes.
- **Show-all toggle:** stays muted; its hover uses the ink.
- **Print:** keep today's print rules (full body, no chrome), and also clear the header
  fill.

In the authoring card (`Mod+Shift+Y`), `.bob-copy-author-icon` becomes the same solid
well: ink background and a `--text-on-accent` glyph, instead of the 12% accent wash. Its
live preview already uses `copyChipBuildInline` / `copyChipBuildFence`, so it inherits
the new look. Keep the inert preview's button and toggle `pointer-events: none`.

### Explicitly unchanged

- **Command chips and `bob-run` cards.** Their glyphs also carry the
  `bob-copy-chip-glyph` class, so every new glyph rule must be scoped under
  `.bob-copy-chip >` or `.bob-copy-fence-header`. Leave the shared base
  `.bob-copy-chip-glyph` rule as it is. The CSS text from `.bob-run-chip {` up to
  `.bob-copy-fence {` must stay byte-identical;
  `scripts/test-command-chips-live-preview.cjs` slices exactly that range. Run chips are
  a separate family and are unused in the vault today.
- Source mode, revealed source when the cursor enters a span, the `Mod+Y` Copy card UI
  (it renders no chip faces), the grammar, gestures, cursor rule, clipboard path,
  feedback timers, the live region, and the Notices.
- No new plugin setting. The CSS custom property is the only knob.

## Steps

Work in the `bob-plugins` linked repo (open it with `/sase_repo`; read its `AGENTS.md`),
except step 7. `bob-chips` is a source-fragment plugin: edit `src/` and `styles.css`,
run `npm run build`, and never hand-edit `main.js`. Keep every hand-edited fragment at
or under 1000 lines.

1. **Styles.** In `plugins/bob-chips/styles.css`, rewrite the inline copy-chip section
   (before `.bob-run-chip {`) per the tokens, states, tooltip, and parity rules above.
   Then restyle the `.bob-copy-fence*` section and `.bob-copy-author-icon`.
   - Use theme variables only: no hex, and no `rgb[a](` except the existing
     `--bob-yank-flash-bg` fallback.
   - Every new glyph rule must be scoped (see Explicitly unchanged).
2. **DOM.** In `src/040-chip-dom.js`, add the tip id, `role="tooltip"`, and the host's
   `aria-describedby`, keeping every best-effort `try` guard. Change nothing else in the
   builders.
3. **Static style test.** Add `scripts/test-copy-chips-look.cjs`, registered in
   `package.json` `test:unit`. In the copy sections of `styles.css` it asserts:
   - no fixed colors;
   - the `var(--bob-copy-chip-hue, var(--color-blue))` default;
   - nowrap host with a normal face;
   - a well scoped under `.bob-copy-chip >`;
   - a tooltip reveal on `:hover` / `:focus-visible`;
   - from-only pulse keyframes;
   - no `border-width` or `border: [2-9]px` in state rules;
   - reduced-motion and print blocks covering the chip, the well, and the tooltip.

   Also assert that the run-chip slice still matches its existing test's expectations.
   Extend `scripts/test-copy-chips-feedback-dom.cjs` (or the closest existing DOM test)
   with these checks:
   - the tip has an id and `role="tooltip"`, and `aria-describedby` points at it;
   - the payload still appears in no attribute;
   - the existing tooltip texts are unchanged.

4. **Preview harness.** Add `scripts/preview-copy-chips.cjs` and
   `scripts/preview-copy-chips.template.html`, modeled on
   `scripts/preview-command-chips.cjs`: headless Chromium over CDP (`CHROMIUM_PATH` when
   none is on `PATH`; Playwright's Chromium works), the built `main.js` and
   `styles.css`, the production `copyChipBuildInline` / `copyChipBuildFence`, and a
   Lucide icon shim.

   The page baseline includes the five Obsidian button rules above verbatim, plus
   context scaffolding for judging the look:
   - Blocked, Next, and WIP row washes;
   - a cyan inline-code pill like the vault's `inline-code` snippet;
   - an accent link.

   Each chip renders twice: a Live Preview span (`interactive: false`) and a Reading
   button (`interactive: true`). The run exits nonzero on any failed check:
   1. **Parity.** Span and button compute identical fill, border colors, box-shadow,
      ink, padding, font, and outline. Check labeled and literal faces in each state
      (rest; forced `:hover`, `:active`, `:focus-visible` via `CSS.forcePseudoState`;
      `is-copied`; `is-failed`), plus the well's fill.
   2. **Geometry.** Chip and well client rects are identical across all states (glyph
      swapped to `check` / `alert-triangle` for copied/failed).
   3. **No stranded glyph.** Sweep container widths 120–420 px in 3 px steps with 0–11
      lead words, using a short label, a one-letter label, and a long literal. The glyph
      svg and the face's first character always share a line, and long faces still wrap.
   4. **Contrast.** At rest in light and dark (Obsidian default palette, accent
      `hsl(258 88% 66%)`, blue `#086ddd` / `#027aff`): face ink vs. body fill ≥ 4.5:1,
      and glyph vs. well ≥ 3:1. Parse the computed `rgb()` and `color(srgb …)` forms.
      Copied and failed are transient and are also announced, so they are exempt.
   5. **Tooltip.** Hidden at rest. After the delay settles under forced hover and
      focus-visible, it is visible above the chip with nonzero size. Its text equals the
      builder's text.
   6. **Print.** Every state prints as plain inline code: no border, fill, padding,
      well, tooltip, or weight. Fence cards print their full body without chrome.
   7. **Reduced motion.** No running animation or transition on chips or cards.
   8. **Attributes.** No rendered attribute contains a payload. Accessible names
      (`aria-labelledby` = Copy prefix + face) are unchanged; `aria-describedby` names
      the tip.
   9. **Card button.** The card's Copy button computes the chip body recipe, never
      Obsidian's button fill or shadow.

   Screenshots go to `--out`:
   - `copy-chips-light.png`, `copy-chips-dark.png`;
   - `copy-chips-narrow-light.png` (390 px);
   - `copy-chips-reduced-motion-dark.png`, `copy-chips-print.png`;
   - `copy-fences-light.png`, `copy-fences-dark.png`.

   Add an offline `--self-test` (page assembly and matrix, no browser) and a wrapper,
   `scripts/test-copy-chips-preview.cjs`, registered in `test:unit`. Run the real
   preview, inspect every screenshot yourself, and tune within the token table only when
   a capture shows a real defect. Record any tuned value in the docs.

5. **Real-Obsidian e2e.** Extend `scripts/obsidian-copy-chips-e2e.cjs`. In its isolated
   scratch vault, for one labeled chip rendered in Live Preview and in Reading view:
   - computed-style parity between the LP span and the Reading button, at rest and after
     a trusted CDP mouse hover;
   - the tooltip becomes visible about 0.5 s after hovering;
   - the copy click still flips the well green and copies byte-exact (existing cases).

   Save screenshots to `--out`. Run it where Obsidian is installed (athena: copy the
   working tree to a fresh `/tmp` directory, never into `~/bob`, and run
   `xvfb-run -a node --experimental-websocket scripts/obsidian-copy-chips-e2e.cjs --out /tmp/copy-chips-e2e`).
   If it cannot run, say so plainly in the final report; never claim it passed.

6. **Docs, version, build, deploy.**
   - `docs/bob-chips.md`:
     - change the intro's "a quiet chip" to name the blue copy well;
     - rewrite "Look and feedback" from this plan: the well and body, the
       `--bob-copy-chip-hue` knob and its default, one look in every view, the
       no-stranded-glyph rule, the hover/focus tooltip, the states table, the
       reduced-motion and print rules;
     - update the fence-card paragraph (well, tinted header, Copy button);
     - fix "Command chips share the copy chip's geometry, with a faint accent wash" so
       it states the command chip's own quiet geometry (padding `0 .45em`, hairline
       accent border, 7% accent wash) instead.
   - `docs/develop/validation.md`: add a "Copy chip preview and e2e" section mirroring
     the command-chip one.
   - Bump `plugins/bob-chips/manifest.json` to `0.14.0`.
   - Run `npm run build`, `npm run docs:gen`, `npm test`, and `npm run validate`, plus
     `node scripts/preview-command-chips.cjs --out /tmp/command-chips-preview`, which
     must still pass untouched.
   - Commit, then run `bob plugins sync` (follow the repo's refusal rules; never `-F` on
     your own).
7. **Glossary (bob-cli; decision `glossary_copy_chip`).**
   - In `sase/memory/glossary/copy-chip.md`, replace "renders the span as a quiet pill"
     with the new look, e.g. "renders the span as a pill with a solid blue copy well".
     Under `hue = accent`, say "accent-colored copy well". Change nothing else in the
     strand.
   - Run `sase memory init`, then `uv run --project tools/docs bob-docs gen` to
     regenerate `docs/reference/glossary.md`, then `just docs-check`.

## Acceptance

- In the vault, `gtd_daily`'s **summary prompt** / **last-week prompt** chips show a
  solid blue well and a tinted blue body. They read the same in Live Preview and in
  Tasks/Reading renders, and stand apart from cyan inline code and purple links on
  Blocked, Next, and WIP rows, in light and dark.
- Hovering or keyboard-focusing a chip shows its "Copy: …" tooltip after about 0.4 s.
- No layout shift on hover, focus, copy, or failure. The glyph never sits alone at a
  line end, and long faces still wrap.
- Print/PDF output is unchanged plain inline code; with the plugin off, chips are
  unchanged inline code.
- Command chips and `bob-run` cards are visually unchanged, and their preview passes.
- `npm test`, `npm run validate`, the copy-chip preview, and (where runnable) the
  real-Obsidian copy-chip e2e all pass. Bundle and docs are fresh; the plugin is
  deployed with `bob plugins sync`.
