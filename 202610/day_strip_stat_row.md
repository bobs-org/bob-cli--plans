---
tier: tale
title: Day strip stat row with stacked stats where the values lead
goal: The Day strip's facts line becomes a polished row of stacked stats whose large
  tabular values clearly dominate their labels, with identical data, links, and accessibility.
size: medium
decisions:
  stale_values:
    ask: Should a stale cached frame (refresh failed) show its values in muted text?
    choices:
      muted: Values switch to muted text while `refresh failed` shows, so cached numbers
        never look live
      full: Values stay full strength; only the faint meta text at the far right says
        `refresh failed`
    default: muted
    why: The faint far-right meta is easy to miss; dimming puts the signal on the
      numbers themselves
    answer: muted
proposed_by: bbugyi200.athena.145.w6
proposed_with: claude/opus
decided_by: auto
status: done
---

# Day strip stat row: stacked stats where the values lead

## Goal

Redesign line 1 of the Day strip (the five facts above the root-parent legend and the
full-width working-time bar) so that it looks polished and **the values are the first
thing the eye sees**. Today each fact is an inline `LABEL value` run: the labels and
values are nearly the same size and weight, and nothing separates one fact's value from
the next fact's label (`WORKED 3h 05m REVIEWED 10 DONE 8 …`). As a result, the line
reads as one run-on sentence.

The new design is a quiet row of stacked stats. Each stat is a small uppercase label
over a large tabular value. The stats sit in content-aware columns separated by inset
hairlines, and the unit letters are smaller than the numerals:

```
WORKED      │ REVIEWED   │ DONE    │ CREATED   │ REVIEW CLOSED
3h 05m      │ 10         │ 8       │ 11        │ not yet        ← values ≈21px, 650 weight
• gtd 102m  • dev 83m                              so far · 14:50
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  (bar unchanged)
```

(In `3h 05m`, the `h` and `m` are about 60% size and muted. The numerals carry the
weight.)

All of this is presentation only. No data, model semantics, interaction targets, fetch
or refresh policy, or CLI JSON changes. bob still owns every displayed string. The
joined text of each value is exactly bob's `display`.

This design was prototyped against bob-plugins `origin/master` (a61d907) and rendered
through the production preview harness (`scripts/preview-day-strip.cjs`). The prototype
passed every existing geometry and interaction check except the old height budgets,
which it was expected to exceed. The values below come from that prototype. They are not
guesses. The existing 63 day-strip unit tests also passed against it unchanged.

## Design principles (keep these in the code comments)

1. **The values lead.** The value is 1.5em of the strip font (≈21px), weight 650, with
   `tabular-nums lining-nums` and −0.02em tracking. The label above it is 0.7em
   uppercase, weight 600, +0.06em tracking, in `--text-muted`. The value is about 2.1×
   the label size.
2. **Numerals carry the weight, and units step back.** In bob's numeric displays
   (`3h 05m`, `45m`, `12`, `+3`, `−5m`, `75%`), the unit letters are 0.6em of the value
   and `--text-muted`. The space between `3h` and `05m` is kept at full value size.
3. **Words are statements, not quantities.** `met`, `missed`, and `not yet` are drawn at
   body size (0.66em of the value, weight 600) on the same baseline as the numerals. A
   missing or not-tracked `–` stays at full value size in `--text-faint`, a quiet
   placeholder.
4. **Rhythm comes from structure, not decoration.** Columns share free width equally,
   and inset 1px hairlines in `--background-modifier-border` separate them. There is no
   card, border, fill, shadow, or metric colour. **Hue stays reserved for root parents
   in the legend and bar**, so colour keeps one meaning in the strip. Status colours
   stay out because the strip shows facts, never targets (see open memory bead
   `bob-cli-bs`).
5. **Quiet states never move a value.** Warnings move into the fixed-height label line,
   and the skeleton keeps the loaded geometry exactly. A stale frame mutes its values
   (`stale_values`).
6. **The whole stat is the link.** The hover wash fills the stat cell, and the label
   brightens to `--text-normal`. The focus ring is inset, so the code-block container
   can never clip it.
7. **It folds gracefully.** When the strip container is at most 34em wide, the facts
   fold into three columns (3 + 2), and stats that start a row draw no hairline.
8. **It aligns.** The legend row gets the same 0.7em inline padding as the stat cells,
   so the first label, the first legend dot, and the meta line up. The bar stays
   full-bleed and still ends the strip.

## Implementation

All plugin work is in the **bob-plugins** linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and work only in the path it prints. First
read its `AGENTS.md`. Edit `src/` fragments, never `main.js`, and run `npm run build`.
Each hand-edited fragment must stay at or below 1000 lines. Pull/rebase onto the latest
`origin/master` before starting.

### 1. Painter: `plugins/bob-ledger-tools/src/239-plugin-day-strip-render.js`

Add a pure helper above `dayStripRenderFact`, and export it through `helpers` in
`src/350-exports.js` next to the other `dayStrip*` helpers:

```js
// Bob's numeric displays ("3h 05m", "45m", "12", "+3", "−5m", "75%")
// split into numeral and unit runs so the numerals carry the weight;
// any other display (words, "–") stays one word run. The runs only
// style bob's string: their joined text is always the display itself.
const DAY_STRIP_NUMERIC_DISPLAY = /^[+−-]?\d+(?:[.,]\d+)?[a-z%]*(?: \d+(?:[.,]\d+)?[a-z%]*)*$/;

function dayStripValueRuns(display) {
  const text = typeof display === "string" ? display : "";
  if (!text || !DAY_STRIP_NUMERIC_DISPLAY.test(text)) return [{ kind: "word", text }];
  const runs = [];
  const pattern = /([+−-]?\d+(?:[.,]\d+)?)|([a-z%]+)|( )/g;
  let match;
  while ((match = pattern.exec(text))) {
    if (match[1]) runs.push({ kind: "num", text: match[1] });
    else if (match[2]) runs.push({ kind: "unit", text: match[2] });
    else runs.push({ kind: "space", text: match[3] });
  }
  return runs;
}
```

These shapes match bob-cli `src/native/stats/series.rs` `format_value` /
`format_minutes`: counts, signed counts with `+`/`−`, `Nm` / `Nh MMm`, ratios `N%`, and
bool `met`/`missed`, plus `not yet` for provisional today and `–` for null.

Rewrite `dayStripRenderFact` so that each fact is:

```
span.bob-day-strip-fact[role=link][tabindex=0][aria-label][title][data-day-strip-metric]
  span.bob-day-strip-head
    span.bob-day-strip-label            ← fact.label (CSS uppercases)
    span.bob-day-strip-warning[aria-hidden][title]  ← only when fact.warning && !skeleton
  span.bob-day-strip-value[ + -word | -empty ][ + -skeleton ][title when not_tracked]
    span.bob-day-strip-{num|unit|space|word}   ← one per run, text = run.text
```

- `-empty` applies when `fact.display === "–"` and the fact is not a skeleton, and it
  takes precedence over `-word`. `-word` applies when the runs are a single word run.
- Keep `aria-label` (`<label> <display>`, or `<label>, loading`), `title` (the tooltip),
  `data-day-strip-metric`, and the existing `dayStripActivate` wiring byte for byte. The
  link's `aria-label` overrides the content, so screen readers still hear
  `Worked 3h 05m`.
- The skeleton keeps its per-fact placeholder displays from the model (`0h 00m`, `00`,
  `not yet`). Its runs then match a loaded today strip's DOM shape, so the existing
  "skeleton … same tag and class sequence" test keeps passing.
- Do not touch the mix, legend, bar, meta, unavailable line, or the model. (The model
  file needs no change.)

### 2. Styles: `plugins/bob-ledger-tools/styles.css` (the Day strip section)

Replace the fact rules (`.bob-day-strip-facts` through `.bob-day-strip-warning`) with
the following. Update the section's header comment to say "a row of stacked stats". Its
"no card … Area hues are the only colour" sentence stays true.

```css
.bob-day-strip-facts {
  display: grid;
  grid-auto-flow: column;
  grid-auto-columns: auto; /* content-aware tracks that share free width equally */
  min-width: 0;
  row-gap: 0.35em;
}
@container (max-width: 34em) {
  .bob-day-strip-facts {
    grid-auto-flow: row;
    grid-template-columns: repeat(3, auto);
  }
  .bob-day-strip-fact:nth-child(3n + 1)::before {
    display: none;
  }
}
.bob-day-strip-fact {
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  min-width: 0;
  gap: 0.15em;
  padding: 0.25em 0.7em 0.3em;
  border-radius: var(--radius-s);
  cursor: pointer;
}
.bob-day-strip-fact + .bob-day-strip-fact::before {
  content: "";
  position: absolute;
  top: 0.45em;
  bottom: 0.45em;
  left: 0;
  width: 1px;
  background: var(--background-modifier-border);
  pointer-events: none;
}
.bob-day-strip-fact:hover {
  background: var(--background-modifier-hover);
}
.bob-day-strip-fact:hover .bob-day-strip-head {
  color: var(--text-normal);
}
.bob-day-strip-fact:focus-visible {
  outline: 2px solid var(--interactive-accent);
  outline-offset: -2px;
}
.bob-day-strip-head {
  /* fixed height: a ⚠ never moves the value */
  display: flex;
  align-items: center;
  min-width: 0;
  height: 1.3em;
  gap: 0.35em;
  color: var(--text-muted);
  font-size: 0.7em;
  font-weight: 600;
  letter-spacing: 0.06em;
  line-height: 1.3;
  text-transform: uppercase;
}
.bob-day-strip-label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.bob-day-strip-value {
  /* one 1.5em line box for every value → one shared baseline */
  display: block;
  width: fit-content;
  max-width: 100%;
  overflow: hidden;
  color: var(--text-normal);
  font-size: 1.5em;
  font-weight: 650;
  font-variant-numeric: tabular-nums lining-nums;
  letter-spacing: -0.02em;
  line-height: 1.1;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.bob-day-strip-unit {
  margin-left: 0.04em;
  color: var(--text-muted);
  font-size: 0.6em;
  font-weight: 600;
  letter-spacing: 0;
}
.bob-day-strip-value-word .bob-day-strip-word {
  font-size: 0.66em;
  font-weight: 600;
  letter-spacing: 0;
}
.bob-day-strip-value-empty {
  color: var(--text-faint);
  font-weight: 500;
}
.bob-day-strip-value-skeleton {
  border-radius: 4px;
  background: var(--background-modifier-hover);
  color: transparent;
}
.bob-day-strip-value-skeleton .bob-day-strip-unit {
  color: transparent;
}
.bob-day-strip-warning {
  flex: none;
  color: var(--text-muted);
  font-size: 1.1em;
  letter-spacing: 0;
  line-height: 1;
}
```

> [!decision] stale_values = muted Add
> `.bob-day-strip-stale .bob-day-strip-value:not(.bob-day-strip-value-empty) { color: var(--text-muted); }`
> with a one-line comment ("a cached frame after a failed refresh steps back a shade so
> it never reads as live"). With `full`, add no stale rule.

Other style edits in the same section:

- `.bob-day-strip-mix-row`: `margin-top: 0.35em` (was 0.2em) and add `padding: 0 0.7em`,
  so the legend and meta align with the stat text.
- **Empty-track fix:** the docs promise that a zero-worked `quality: ok` day "shows an
  empty track and `no Pomodoros`". The track never shows today, because the bar always
  holds eight hidden slot spans, so `.bob-day-strip-bar:empty` never matches. Add
  `.bob-day-strip-bar:not(:has(.bob-day-strip-bar-segment:not([hidden])))` to the
  existing track selector list. (Obsidian's Chromium supports `:has`.)
- **Retry spacing:** add `margin-left: 0.4em` to `.bob-day-strip-retry`. Today it reads
  `…timed out.Retry`.
- **Forced colours:** add a small `@media (forced-colors: active)` block for the strip:
  hairlines get `background: GrayText`, and the focus outline gets
  `outline-color: Highlight`. This follows the existing reading-board forced-colors
  block.
- Delete the now-unused rules: the old inline-pill margins and the old
  `.bob-day-strip-label` font rules, which move to `.bob-day-strip-head`. Leave the
  legacy `.bob-day-strip-mix-bar…{display:none}` line alone.

### 3. Unit tests: `scripts/test-ledger-tools-day-strip.cjs`

Add:

- `dayStripValueRuns`:
  - `"3h 05m"` → num `3`, unit `h`, space, num `05`, unit `m`.
  - `"45m"` → num, unit. `"0m"` → num, unit. `"12"` → num. `"+3"` → num `+3`. `"−5m"` →
    num `−5`, unit `m`. `"75%"` → num `75`, unit `%`.
  - `"met"`, `"missed"`, `"not yet"`, `"–"`, `""`, and `"3 h"` each → one word run.
  - For every case, the joined run text equals the input.
- Painter, for every fixture in `day-strip-fixtures.cjs` with a report:
  - Each fact's `.bob-day-strip-value` `textContent` equals that fact's model `display`.
  - Each fact's `aria-label` is unchanged (`<label> <display>`).
  - `.bob-day-strip-head` is the fact's first child, and the value is its second.
  - A `⚠` appears only inside `.bob-day-strip-head`, never inside a value.
  - Word displays carry `bob-day-strip-value-word`, and `–` carries `-empty` (not
    `-word`).
- The existing skeleton DOM-shape test, button-free tests, activation tests, and order
  tests must keep passing unmodified.

### 4. Preview harness: `scripts/preview-day-strip.cjs` and `scripts/preview-day-strip.template.html`

- **Height budgets:** use ≤84px when the facts fit on one row and ≤136px when they fold
  to two rows. The prototype measured 81.1px and 131.6px at a 16px base. Update
  `checkMeasurement` and the matrix-report sentence. The rule that `today` and
  `wide-values` must fit on one row at 540 and 700 stays.
- **Fact-overlap insets:** stats no longer use the hover pill's negative margins. Set
  the overlap-check insets to 0 and rewrite the comment above them. Adjacent cells may
  touch but must not overlap.
- **New checks.** Add each to the "Geometry checks" sentence and to `checkMeasurement`:
  - (n) **The values lead:** for ready and stale cells, every fact's computed value
    `font-size` is at least 2× its `.bob-day-strip-head` `font-size`.
  - (o) **Faithful text:** every fact's `aria-label` ends with its value's
    `textContent`, and no value contains `⚠`.
  - (p) **Clean folds:** at 410px, facts fold to two rows, and every stat that starts a
    row has a computed `::before` `display` of `none` or `content` of `none`. At 540px
    and 700px, every fact after the first draws its hairline.
  - (q) **Empty track:** in the `no-pomodoros` cell, the bar's computed
    `background-color` is not transparent.
- **New case:** add "Closed-out past day · met". In `loadFixtures`, deep-clone `past`
  and set the `review_closed` point for its day to `value: 1, display: "met"`. This
  covers the third word value.
- Update the "Synthetic cases" sentence. Also update the case list in
  `plugins/bob-ledger-tools/scripts/fixtures/stats/README.md` if it names preview cases.
  (It currently lists fixtures, so add the derived `met` case only if you add it to the
  fixture module instead of `loadFixtures`.)
- Run `node scripts/preview-day-strip.cjs --out /tmp/bob-day-strip-preview`. The run
  must report `failures: []` and `overflow: []`. Look at the 700/540/410 light and dark
  screenshots yourself before calling the work done.

### 5. Docs

**bob-cli (this repo)** `docs/stats.md` → `### Day strip`:

- Replace "Each fact reads `LABEL value`: the report's metric label, uppercased, then
  the D point `display`." with prose that matches the new rendering. Each fact stacks
  the report's metric label (small, uppercased, muted) over the D point `display`
  (large, tabular). Facts share the line's width in columns split by hairlines and fold
  into three columns below about 34em. In numeric displays, the unit letters are set
  smaller. The displayed text is always bob's `display` exactly.
- Change "plus a small `⚠`" to "plus a small `⚠` beside the label".
- `stale_values` = `muted`: change "A failed refresh keeps the values with
  `refresh failed`" so that it says the cached values stay, in muted text, with
  `refresh failed`.
- Leave the "Interaction" paragraph alone. Open bug bead `bob-cli-c6` rewrites it.
- Run `just docs-check`.

**bob-plugins** `docs/bob-ledger-tools.md`: change "three calm lines — LABEL-value
facts, …" to describe a row of stacked stats (a small uppercase label over a large
value), followed by the legend and the bar.

**Memory:** no change. The glossary strand `glossary:day-strip` (and its generated
`docs/reference/glossary.md` row) says "five LABEL-value facts", which is still accurate
for the facts' content. The pending decision-record bead `bob-cli-bs` is unaffected.

### 6. Version, build, and sync (bob-plugins)

- Bump `plugins/bob-ledger-tools/manifest.json` to the next minor version (1.71.0 →
  1.72.0 if still current). Then run `npm run docs:gen` so the version in
  `docs/index.md` follows.
- Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`. All
  must pass.
- Run `bob plugins sync` to deploy to the vault.

## Verification checklist

- `npm test` (includes the new day-strip tests) and `npm run validate` pass in
  bob-plugins. `just docs-check` passes in bob-cli.
- The preview matrix passes with the new budgets and checks (n)–(q) at 410/540/700 ×
  light/dark, including skeleton = today height (d), hover never shifts a fact (h), and
  the focus ring shows (h).
- Visual review of the screenshots:
  - Values clearly dominate their labels.
  - `3h 05m` shows small muted `h`/`m`.
  - `not yet` / `missed` / `met` sit on the numerals' baseline.
  - `–` is faint.
  - Hairlines are inset and never start a row.
  - The legend's first dot lines up with the first label.
  - The no-Pomodoros cell shows the empty track.
  - The unavailable line reads `… timed out. Retry`.
- Nothing about what the strip _says_ changes. aria-labels, tooltips, click/keyboard
  targets, and the openStats payloads are identical to before.

## Out of scope

- Legend items overlapping each other when root labels are long. This bug already exists
  and is filed as `bob-cli-c8`. Do not fix it here, and do not make it worse: checks (k)
  and (l) must still pass.
- `openStatsForDay` ignoring `newLeaf` (`bob-cli-c6`).
- Any new data on the strip: no deltas, goals, sparklines, history marks, or per-metric
  colours. The strip shows facts, never targets.
- The Stats page (`bob-stats` block) tiles.
