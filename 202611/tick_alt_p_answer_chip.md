---
tier: tale
title: TICK review rows show the Alt+P answer as a chip
goal:
  On TICK review rows, the footer's Alt+P priority preview becomes a tinted keycap chip
  that stands out at a glance.
size: medium
decisions:
  landing_glow:
    ask:
      Should the Alt+P chip glow once each time the review walk lands on a new TICK row?
    choices:
      glow:
        The chip's tint fades from 32% to 13% over 900 ms once per row; reduced motion
        stays static
      still: The chip is always static and stands out by shape and color alone
    default: glow
    why: One short settle draws the eye from the editor to the answer without looping
    answer: glow
  glossary_review_footer:
    ask:
      Should the review-footer glossary strand describe the TICK Alt+P chip and its
      narrow-bar shedding order?
    memory:
      - glossary:morning-gtd-review-footer
    default: true
    corrects:
      and as space runs out it drops groups from the end of the walk (marking the cut
      with `…`), then detail, hint, and meter.
    answer: true
proposed_by: bbugyi200.athena.1ay
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 10:02:26
status: wip
---

# TICK review rows: show the Alt+P answer as a chip

## Goal

When the cursor is on a task in the **TICK** (TICKLER) review group, the review footer's
Alt+P preview (today plain muted text like `Alt+P decay P2→P3`, easy to miss between
`scheduled Nov 10` and `NEW 1`) becomes the most visible element in the footer. It
renders as a tinted **answer chip** with a keycap, so the eye finds it at once and reads
what Alt+P will do.

Why TICK only: for a tickler, "not now" is a priority roll, and Alt+P _is_ that answer
(`docs/freshness.md`, step 3). TICK rows show no other action hint. On every other tier,
the Alt+P preview comes second to that tier's own answers (Ctrl+Alt+F keep, Alt+N
release, Ctrl+Enter done), so a loud chip there would compete with them. A one-line tier
list (`FRESHNESS_FOOTER_ANSWER_TIERS = ["tickler"]`) keeps it easy to extend later.
Every other tier, the off-walk hint, and the hint-only `⟳ Nothing due` footer stay as
they are.

## Design

The look below was prototyped against the production footer painter and stylesheet in
headless Chrome, using the default Obsidian light and dark palettes at 1100, 700, and
460 px. Measured contrast is in the "Color" section.

```
before  ⟳ Review 3/20 · TICKS 2/17 · scheduled Nov 10 · Alt+P decay P2→P3 · NEW 1 | ROTTEN 1 POST 1 · ✓ 12 today
after   ⟳ Review 3/20 · TICKS 2/17 · scheduled Nov 10 · (█Alt+P█ decay P2 → P3) · NEW 1 | ROTTEN 1 POST 1 · ✓ 12 today
```

### Anatomy (left to right, inside one pill)

1. **Keycap** `Alt+P`: a solid pill filled with the chip's ink, with the text in the
   status-bar background color (an inverted key). This is what makes the chip pop, and
   it reads as "a key you press" without any other words.
2. **Verb**: `roll`, `decay`, `set`, `cancel?`, or `cancel` (weight 600).
3. **Levels** (where the kind has them):
   - decay: `P2 → P3`. The source level is at weight 500 and the destination at 750, so
     the eye lands on where the task ends up. The arrow gets 0.22em of margin on each
     side, so it reads `P2 → P3`, not `P2→P3`.
   - roll: `P2 2/3`. The level stays at weight 750; the step count is at weight 500.
     Without a roll limit it shows just `P2`.
   - set (assign): `P1` at weight 750.
   - cancel: no levels (`Alt+P cancel?`).

The text content stays exactly the current compact string (`Alt+P decay P2→P3`). Screen
readers, the button's aria-label, copy/paste, and every existing text check see the same
words as today.

### Color

- The chip uses the **current tier's hue** (TICK blue, the same `--bob-review-hue` as
  the `TICKS 2/17` context), which visually ties the answer to the group. The wash is
  13% of the hue mixed into the status-bar background, the hairline edge is 40%, and
  both are **opaque** `color-mix` with the bar background, so contrast can be measured.
- A **cancel** recommendation switches the chip to `--color-red`. Cancel ends the task
  (after a second press on the Task Card), and that is the one case where color should
  warn.
- Inside the chip the ink is darker in light theme (OKLCH L 0.44) and brighter in dark
  theme (L 0.84) than the footer's 0.52 / 0.80, because the text sits on a tinted wash.
  Measured with the production painter (WCAG):

  | Surface                                 | Light | Dark |
  | --------------------------------------- | ----: | ---: |
  | chip text on wash                       |  6.13 | 7.63 |
  | keycap text on keycap                   |  7.32 | 8.70 |
  | cancel chip text                        |  6.52 | 6.59 |
  | chip text at the glow's peak (32% wash) |  4.67 | 6.00 |

- Browsers without relative color syntax keep the footer's existing `color-mix` ink
  fallback, which also clears 4.5:1 on the wash.

### Motion

> [!decision] landing_glow = glow When a TICK row's chip is first painted (a new row,
> not a cursor move within the same row), its wash animates from 32% to the resting 13%
> over 900 ms (`cubic-bezier(0.2, 0.7, 0.2, 1)`), once. Only `background` animates, so
> nothing shifts and nothing escapes the status bar's clipping.
> `prefers-reduced-motion: reduce` disables it. With `still`, omit the keyframes and the
> `animation` declaration and keep everything else.

### Narrow status bars

Today the fit sheds tail groups, then compacts the detail, then drops the detail, the
hint, and the meter, in that order. When the chip is shown, the **meter drops before the
hint**: groups → compact detail → detail → meter → chip. The chip is the last item to
go, after the review position and the TICK context. In the prototype, the chip still
fits at 460 px with groups and detail shed. Without a chip the order is unchanged.

### Tooltip and hover

The chip gets its own `title`: the preview's full detail sentence
(`Alt+P: decay P2 → P3, 9–31 days out`). Hovering the chip says exactly what the key
does, and hovering anywhere else still shows the footer's full tooltip. The chip is not
a button and does not take focus. Clicking it falls through to the footer's existing
behavior (the chip sits outside the main button, so it does nothing, as the hint does
today).

## Data contract

### 1. bob-navigation-hotkeys `api.quickRoll` v1: additive `parts`

`src/396-quick-roll-preview.js`, `planQuickRollPreview`: every `{ ok: true }` preview
gains a frozen `parts` object. The api `version` stays 1, and the root api version
stays 3.

```js
parts: Object.freeze({
  keyLabel, // "Alt+P"
  verb, // "roll" | "decay" | "set" | "cancel?" | "cancel"
  fromLabel, // decay: source level; null otherwise
  toLabel, // decay: target; roll: the level; assign: the level; cancel: null
  step, // roll with a limit: integer; else null
  limit, // roll with a limit: integer; else null
});
```

Invariant (tested for every kind, including `cancelConfirms: false`):
`keyLabel + " " + verb` plus `" " + toLabel` (roll, assign) or
`" " + fromLabel + "→" + toLabel` (decay), plus `" " + step + "/" + limit` (roll with a
limit), equals `compact` exactly. The `line`, `compact`, and `detail` strings do not
change, so the `]s` landing toast is byte-identical.

### 2. bob-ledger-tools footer model: `view.answer`

`src/120-freshness-footer.js`:

- Add `const FRESHNESS_FOOTER_ANSWER_TIERS = ["tickler"];`.
- Add the pure `freshnessFooterAnswerChip(preview, current)`. It never throws, and it
  returns `null` unless `current.tier` is in `FRESHNESS_FOOTER_ANSWER_TIERS`,
  `preview.ok === true`, `preview.compact` is a non-blank string, and `preview.kind` is
  one of `roll | decay | assign | cancel`. Otherwise it returns a frozen:

  ```js
  {
    kind,                       // preview.kind → data-answer-kind
    tier,                       // current.tier → data-review-tier
    text,                       // === preview.compact.trim() === view.hintText
    title,                      // preview.detail (trimmed) || text
    key,                        // JSON signature of [current.key, current.rank, kind, text, title]
    segments: [{ role, text }], // concatenated texts === text
  }
  ```

  Segment roles are `key`, `space`, `verb`, `from`, `arrow`, `to`, `count`, and
  `action`.
  - **Structured path:** when `preview.parts` is present and valid, build
    `key, space, verb[, space, from, arrow, to | space, to[, space, count]]` by kind
    (for roll, `to` holds the level). Use it only if the joined text equals `text`.
  - **Fallback path** (older nav without `parts`, or parts that do not rebuild
    `compact`): split `text` at its first space into `key`, `space`, and `action` (the
    rest). With no space, use one `action` segment and no key. The chip still renders,
    and the words never diverge from the hint text.

  Including the row identity in `key` is what makes the landing glow replay on a new
  row, even when two rows share the same compact text, and never replay on a cursor move
  within one row.

- `freshnessFooterView`: when `current` and `quickRollReady`, set
  `answer = freshnessFooterAnswerChip(quickRollOption, current)`. Every returned view
  carries `answer` (`null` by default): `freshnessFooterHiddenView`,
  `freshnessFooterHintOnlyView`, and the main return. That way the existing
  "byte-identical" `deepEqual` tests keep holding. `hintText`, `tooltip`, and
  `ariaLabel` are unchanged.
- Export `freshnessFooterAnswerChip` and `FRESHNESS_FOOTER_ANSWER_TIERS` from
  `src/350-exports.js` helpers.

## Implementation

All paths in this section are in the **bob-plugins** linked repo (open it with
`/sase_repo`: `sase repo open bob-plugins`) unless marked **bob-cli**. Edit `src/`
fragments only, then run `npm run build`. Never hand-edit a generated `main.js`. Every
hand-edited fragment must stay at or below 1000 lines.

1. **nav preview `parts`**:
   `plugins/bob-navigation-hotkeys/src/396-quick-roll-preview.js` as specified above.
   Update the frozen-object `deepEqual` expectations in
   `scripts/test-navigation-quick-roll-preview.cjs`, and the preview shapes asserted in
   `scripts/test-navigation-quick-roll.cjs` if any break. Add one test that checks the
   rebuild invariant for roll (with and without a limit), decay, assign, cancel
   (`cancelConfirms` true and false), and the `keyLabel` override.
2. **footer model**: `plugins/bob-ledger-tools/src/120-freshness-footer.js` as specified
   above.
3. **painter**: `plugins/bob-ledger-tools/src/121-freshness-footer-paint.js`.
   - Add `freshnessFooterPaintHint(node, view)`, used by `freshnessFooterPaint` in place
     of `freshnessFooterSetText(parts.hint, view.hintText)`, and also on the hidden
     path.
   - With `view.answer`: if `node._bobAnswerKey !== answer.key`, rebuild. Call
     `empty()`, then create
     `span.bob-freshness-answer[data-review-tier=<tier>][data-answer-kind=<kind>][title=<title>]`.
     Inside it, render the `key` segment as `kbd.bob-freshness-answer-key`, the space
     after it as a `span.bob-freshness-answer-space` (it collapses inside the flex chip;
     `gap` spaces it), and every remaining segment inside one
     `span.bob-freshness-answer-action` as `span.bob-freshness-answer-<role>`. Use spans
     even for spaces (no bare text nodes; the tests' fake DOM has none). Add the class
     `bob-freshness-hint-answer` to the hint node and store `_bobAnswerKey`.
   - Without `view.answer`: if the node holds a chip (`_bobAnswerKey` set), `empty()`
     it, drop the class, and clear the key. Then set the text exactly as today.
   - Build with `freshnessFooterCreateChild`, `freshnessFooterSetText`, and
     `freshnessFooterSetAttr` (`createEl` with a DOM fallback), so the painter never
     throws. A chip build failure falls back to plain `hintText`.
   - `freshnessFooterFit`: when `view.answer` is set, shed the meter before the hint.
     The return shape is unchanged.
4. **status bar**: `plugins/bob-ledger-tools/src/240-plugin-freshness-status.js`. Add
   `view.answer ? view.answer.key : ""` to the `paintKey` join, so a new TICK row with
   the same text still repaints and glows.
5. **styles**: `plugins/bob-ledger-tools/styles.css`. Add an "Alt+P answer chip" block
   **after** the tier-hue and ink rules of the review footer section, with a comment in
   the section's voice. Use this CSS (verified in the prototype):

   ```css
   .bob-freshness-hint.bob-freshness-hint-answer {
     display: inline-flex;
     align-items: center;
     color: inherit;
   }
   .bob-freshness .bob-freshness-answer {
     --bob-answer-ground: var(--status-bar-background, var(--background-secondary));
     --bob-answer-wash: color-mix(
       in srgb,
       var(--bob-review-hue) 13%,
       var(--bob-answer-ground)
     );
     --bob-answer-edge: color-mix(
       in srgb,
       var(--bob-review-hue) 40%,
       var(--bob-answer-ground)
     );
     --bob-review-ink-l: 0.44;
     display: inline-flex;
     align-items: center;
     gap: 0.42em;
     padding: 0.16em 0.62em 0.16em 0.18em;
     border-radius: 999px;
     background: var(--bob-answer-wash);
     box-shadow: inset 0 0 0 1px var(--bob-answer-edge);
     color: var(--bob-review-ink);
     font-weight: 600;
     line-height: 1;
     cursor: default;
     animation: bob-freshness-answer-land 900ms cubic-bezier(0.2, 0.7, 0.2, 1) 1;
   }
   .theme-dark .bob-freshness .bob-freshness-answer {
     --bob-review-ink-l: 0.84;
   }
   .bob-freshness .bob-freshness-answer[data-answer-kind="cancel"] {
     --bob-review-hue: var(--color-red, #e93147);
   }
   .bob-freshness-answer-key {
     display: inline-flex;
     align-items: center;
     padding: 0.2em 0.45em 0.18em;
     margin: 0;
     border: none;
     border-radius: 999px;
     background: var(--bob-review-ink);
     color: var(--bob-answer-ground);
     font: inherit;
     font-size: 0.9em;
     font-weight: 700;
     letter-spacing: 0.02em;
     line-height: 1;
     box-shadow: none;
   }
   .bob-freshness-answer-action {
     display: inline;
     white-space: nowrap;
   }
   .bob-freshness-answer-verb {
     font-weight: 600;
   }
   .bob-freshness-answer-from,
   .bob-freshness-answer-count {
     font-weight: 500;
   }
   .bob-freshness-answer-arrow {
     margin: 0 0.22em;
   }
   .bob-freshness-answer-to {
     font-weight: 750;
   }
   @keyframes bob-freshness-answer-land {
     from {
       background: color-mix(
         in srgb,
         var(--bob-review-hue) 32%,
         var(--bob-answer-ground)
       );
     }
     to {
       background: var(--bob-answer-wash);
     }
   }
   @media (prefers-reduced-motion: reduce) {
     .bob-freshness .bob-freshness-answer {
       animation: none;
     }
   }
   ```

   The chip carries `data-review-tier`, so the existing
   `.bob-freshness [data-review-tier]` rules supply `--bob-review-hue` and compute
   `--bob-review-ink` from the chip's own `--bob-review-ink-l`. The cancel override has
   higher specificity than the tier-hue rule on purpose. `.bob-freshness-omit`
   (`!important`) still hides the inline-flex hint. Keep the existing `·` separator
   before the chip; it sits outside the pill and keeps the footer's rhythm (a
   pill-as-separator variant read worse in the prototype).

6. **build**: `npm run build`.

## Tests (node, in `npm test`)

In `scripts/test-ledger-tools-freshness-footer.cjs` (add a small `deepText(node)` helper
that joins descendant `text` values, because the fake DOM's `textContent` is not
derived):

- `freshnessFooterAnswerChip`: structured segments for decay, roll (with and without a
  limit), assign, cancel `?`, and immediate cancel. Cover the fallback for missing
  parts, for parts that do not rebuild `compact`, and for a compact with no space.
  `null` for non-TICK tiers, `ok: false`, a blank compact, an unknown kind, and garbage
  input (never throws). Check that the joined segment text equals `text` in every case,
  and that `key` differs across rows with identical compact text.
- `freshnessFooterView`: a TICK row with a preview has `answer` set and an unchanged
  `hintText`, tooltip, and aria. NEXT, ROTTEN, NEW, and RECURRING rows, off-walk rows,
  hint-only views, and hidden views have `answer: null`. The existing "byte-identical"
  test still passes.
- Painter: a TICK view paints `.bob-freshness-answer` with `data-review-tier="tickler"`,
  `data-answer-kind`, `title`, a `kbd` key, and `deepText(hint) === view.hintText`. A
  repaint with the same `answer.key` keeps the same chip node (no rebuild), while a new
  key rebuilds. Switching to a non-TICK view clears the chip and paints plain text, and
  a hidden view clears it too.
- Fit: copy the progressive-fit test for a TICK view, and assert the order groups →
  detail → **meter → hint**. The existing non-TICK fit test stays as it is.
- Status-bar wiring: with a TICK-tier cursor match and a nav `quickRoll.preview` that
  returns `parts`, the hint holds the chip. A same-text preview on a different TICK row
  changes `freshnessFooterPaintKey`. Existing assertions on non-TICK rows stay
  byte-identical (if a fixture task turns out to be TICK, assert with `deepText`
  instead).
- Stylesheet: the CSS defines `.bob-freshness-answer`, the cancel override, the keycap,
  and a `prefers-reduced-motion` guard (the guard only under `glow`).

## Visual verification (preview harness)

Extend `scripts/preview-review-footer.template.html` and
`scripts/preview-review-footer.cjs`:

- Add fixtures `tick-decay` (TICKLER current row, decay preview with `parts`),
  `tick-cancel` (cancel `?` with `parts`), and `tick-legacy` (decay preview without
  `parts`). `next-roll` stays chip-free.
- Before measuring, call `document.getAnimations().forEach((a) => a.finish())`. Resolve
  chip colors through a 1×1 canvas (`fillStyle` → `getImageData`), because computed
  `color-mix` backgrounds come back as `color(srgb …)`, which the current `rgb()` parser
  misreads.
- New checks, run at every width and theme: a chip exists on every `tick-*` fixture and
  not on `next-roll`. Hint text equals `view.hintText`. Chip text on the chip background
  is at least 4.5:1, and so is the keycap text on the keycap. The chip stays visible at
  1100, 700, and 460 px. There is no host overflow. Update the summary's fixture and
  check lines.
- Run `node scripts/preview-review-footer.cjs --out /tmp/bob-review-footer-preview`. It
  must report 0 failed checks. Look at `light-1100.png`, `dark-1100.png`, and
  `light-460.png`, and attach the paths in the final report.

## Docs, manifests, memory

- Bump the minor version of each plugin you change, from whatever is current when you
  start (bob-ledger-tools is 1.101.1 and bob-navigation-hotkeys is 2.76.1 as of this
  plan). Add one sentence to each manifest `description`:
  - ledger-tools: on TICKLER rows the Alt+P preview renders as a tier-tinted answer chip
    with a keycap (red when it would cancel), and it outlasts the meter as the bar
    narrows.
  - nav: `api.quickRoll` previews carry additive `parts`.

  Then run `npm run docs:gen` so the generated inventory and Description rows match.

- `docs/bob-ledger-tools.md`, in the footer paragraph ("On review rows, nav
  `api.quickRoll` v1 supplies Alt+P's hint and tooltip…"): describe the TICK chip, the
  cancel red, the chip's own hover sentence, and that the chip drops after the meter.
- `docs/develop/plugin-apis.md`: document the `parts` field and its rebuild invariant
  next to the `quickRoll` preview shape. The api version is unchanged.
- **bob-cli** `docs/freshness.md`, footer paragraph ("The Obsidian footer keeps a
  condensed version…"): add one sentence saying that on a TICKLER row the Alt+P preview
  shows as a highlighted `Alt+P` answer chip (for example `Alt+P decay P2→P3`), red when
  the recommendation is cancel. Leave the guide steps as they are.

> [!decision] glossary_review_footer **bob-cli**
> `sase/memory/glossary/morning-gtd-review-footer.md`: in the queue-row sentence, say
> that on TICK rows the `Alt+P` preview renders as a tinted answer chip. Rewrite the
> shedding clause to say that it drops groups from the end of the walk, then detail,
> hint, and meter, except on TICK rows, where the meter goes before the Alt+P chip. Keep
> the strand terse. Then run `sase memory init`. When declined, leave the strand alone.

## Verification

In bob-plugins:

- `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`, all green.
- The preview harness reports 0 failures, and the screenshots are reviewed.
- `bob plugins sync` deploys the plugins to the vault. Follow the repo's `AGENTS.md`
  rules if sync refuses.

In bob-cli: `just docs-check`, and run `sase memory init` (under the memory decision).

## Non-goals

- No change on non-TICK tiers, in the off-walk hint, or in the `⟳ Nothing due` hint-only
  footer.
- No change to the `]s` landing toast, to Alt+P's behavior, or to the recommendation
  itself.
- The chip is display-only: no click-to-apply, and no new hotkey or command.
- No macOS `⌥P` glyph. The keycap keeps the `Alt+P` label used by every toast, hint, and
  doc.
