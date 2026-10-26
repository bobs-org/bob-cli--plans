---
tier: tale
title: Day strip REVIEW CLOSED shows check, X, and in-progress marks
goal:
  The Day strip's REVIEW CLOSED fact shows a green check, red X, or yellow In Progress
  half-ring instead of words.
size: medium
decisions:
  celebrate:
    ask:
      When today's review flips from not yet to met live, should the check draw itself
      once?
    choices:
      draw:
        Seal pops in and the check strokes on once (~0.4 s); never on load or for missed
      none: Marks always paint static; no per-host memo, keyframes, or arrive tests
    default: draw
    why:
      Closing the review is the strip's daily win; a one-shot draw never fires on note
      open
    answer: draw
proposed_by: bbugyi200.athena.14j
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 11:00:36
status: wip
---

# Day strip: REVIEW CLOSED shows a mark, not a word

## Goal

Today the Day strip's REVIEW CLOSED fact prints bob's words `met`, `missed`, and
`not yet` in small bold text. Replace those words with three status marks that sit on
the same line as the strip's big numerals:

- **met**: a green check.
- **missed**: a red ✕.
- **not yet** (today's review still open): Bob's existing yellow **In Progress
  half-ring**, the glyph Bob already uses for In Progress Task Links.

All three must look polished in light and dark themes. Bob still owns the data: the
plugin chooses a mark from bob's point value, and bob's display word stays the
accessible text and tooltip text.

Nearly all of the work is in the **bob-plugins** linked repo (plugin
`bob-ledger-tools`). Open it with your `/sase_repo` skill and read its `AGENTS.md`.
bob-cli itself changes only in `docs/stats.md`. bob-cli's `stats --json` output does not
change.

## The design (validated in a Chromium prototype)

Five styles were compared side by side at 700 px in light and dark themes:

- solid disc with a knocked-out glyph;
- bare check / ✕ strokes;
- ring plus glyph;
- tinted disc plus glyph;
- tinted disc with a hairline rim, plus glyph.

**Tinted seal with a hairline rim** won. The check and ✕ themselves carry the colour, as
asked. A 14% tint disc gives the mark enough body to stand beside 650-weight numerals.
The rim crisps the edge, especially in light themes. Solid discs looked loud in a strip
whose only other colour is the area bar. Bare strokes looked thin.

All marks share Bob's established 16-unit glyph box and 1.9 stroke, from the freshness
check mark and the progress half-ring. The three marks share one silhouette, so the
column never jumps.

### Geometry (`viewBox="0 0 16 16"`)

| Part                                                                      | met                                 | missed                                 | not yet (pending)                                         |
| ------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------- | --------------------------------------------------------- |
| Tint disc `circle cx=8 cy=8 r=7.1`                                        | ink 14%                             | ink 14%                                | ink 14%                                                   |
| Rim `circle r=6.6`, stroke 1, ink 45%, no fill                            | yes                                 | yes                                    | —                                                         |
| Ring `circle r=6.05`, stroke 1.9, full ink, no fill (house progress ring) | —                                   | —                                      | yes                                                       |
| Glyph                                                                     | check `M4.75 8.35l2.1 2.1 4.4-4.65` | ✕ `M5.6 5.6l4.8 4.8M10.4 5.6l-4.8 4.8` | half disc `M8 4.1A3.9 3.9 0 0 1 8 11.9Z`, filled with ink |

- **Glyph strokes:**
  - The check and ✕ use stroke 1.9 in full ink, with round caps and joins and no fill.
  - Give the glyph path `pathLength="1"` so the optional draw-in can use a unit dash.
- **Size:**
  - The mark is `1.04em` square, measured in the value's font. The value font is 1.5×
    the strip font.
  - The value line keeps its `1.1` line box. Lay it out as
    `display:flex; align-items:center; height:1.1em`, so the mark centres on the
    numerals' line and every value bottom still aligns (matrix check (c)).
- **Inks:** set them through one custom property on the value element, e.g.
  `--bob-day-strip-mark-ink`.
  - met: `var(--color-green, #3f8f5f)`.
  - missed: `var(--color-red, #c65367)`.
  - pending: `var(--task-status-in-progress, var(--color-yellow, #c69026))`. This is the
    same In Progress ink the progress marks and the vault's `task-statuses` snippet use.
  - Tints use `color-mix(in srgb, var(--bob-day-strip-mark-ink) 14%, transparent)`,
    which matches the vault's `--task-status-*-bg` 13–14% tints. The rim uses 45%.
- **Stale frame** (`.bob-day-strip-stale`): the ink becomes `var(--text-muted)`, so a
  cached verdict steps back a shade like the stale numerals do and never reads as live.
- **Forced colors:** the ink becomes `CanvasText`, the tint disc gets `fill: none`, and
  the rim gets full ink. The shapes alone (check, ✕, half-ring) still carry the meaning.
- **Colour-blind safety:** the three states differ by shape, never by hue alone.

### States around the marks

- **Skeleton:**
  - A boolean fact's skeleton is a round placeholder exactly the mark's size: a
    `--background-modifier-hover` circle, with no transparent "not yet" text box.
  - When data arrives, the mark replaces it in place, at the same height and with no
    width jump.
  - Matrix check (d), skeleton height equals today's height, must still hold.
- **No mark:**
  - `missing` / `not_tracked` points and null values keep today's `–` (or word)
    rendering.
  - Non-bool units never get a mark.
- **Warning:** the `⚠` stays in the head as today, and the mark still paints.
- **Tooltip:**
  - The fact `title` still reads `<Day>: <display>`.
  - For the pending mark only, drop the trailing ` so far`. The tooltip is now the only
    visible home of "not yet", and "not yet so far" reads badly.
  - Other provisional facts keep ` so far`.
- **Accessibility:**
  - The SVG is `aria-hidden="true"` and `focusable="false"`.
  - The value also holds a visually hidden span (class e.g. `bob-day-strip-mark-text`,
    clip-path inset 50% pattern) whose text is bob's `display`. So `value.textContent`
    still equals `fact.display`.
  - The fact's `aria-label` stays `<label> <display>` (matrix check (o)), and the strip
    group sentence is unchanged.

> [!decision] celebrate = draw When a repaint of the same host element, for the same
> day, turns a `pending` mark into `met` in a `ready`, non-stale frame, add an arrive
> class to that value:
>
> - The seal scales from 0.6 to 1 and fades in over about 280 ms, using
>   `cubic-bezier(.34,1.56,.64,1)`.
> - The check strokes on over about 320 ms after a 90 ms delay.
>
> Remember the last painted marks per host in a module-level
> `WeakMap<hostElement, {day, marks}>`, updated only on `ready` paints. First load,
> skeleton→met, another day, met→met, and anything→missed never animate.
>
> The resting state must be the drawn state. Put `stroke-dashoffset: 1` only inside the
> `@keyframes` `from` frame, and use `animation-fill-mode: backwards` for the delay.
> Then the strip's existing `prefers-reduced-motion` rule (`animation: none` on
> `.bob-day-strip *`) leaves a fully drawn check, never an invisible one.

> [!decision] celebrate = none Skip the memo, the arrive class, the keyframes, and their
> tests. Every mark paints static.

## Data rule: which mark

Choose the mark from the bob point, never from the English word. Mirror bob's own
terminal marks exactly: `bool_mark` in bob-cli `src/native/stats/render.rs` (● met, ○
missed, ◌ not yet), plus the `not yet` rule in `daily_point`
(`src/native/stats/series.rs`).

1. The metric `unit !== "bool"` → no mark.
2. `point.provisional === true && point.value === 0` → `pending`.
3. `quality` is `missing` or `not_tracked`, or `value` is not a finite number → no mark.
4. `value > 0` → `met`; otherwise → `missed`.

Do not use `checklist_state`: bob reports `missed` for today's still-open provisional
point, so it cannot tell "not yet" from "missed".

## Implementation (bob-plugins, plugin `bob-ledger-tools`)

Edit only `src/` fragments and `styles.css`, then run `npm run build`. Never hand-edit
`main.js`. Each hand-edited fragment must stay ≤ 1000 lines.

1. **`src/239-plugin-day-strip-model.js`**
   - Add a pure `dayStripMarkState(unit, point)` that implements the data rule.
   - Have `dayStripBuildFact` return `mark: "met" | "missed" | "pending" | null`.
   - Skeleton facts get `mark: null`. Their `unit` is already `"bool"` for
     `review_closed`, so the painter can still draw the round placeholder.
   - Apply the pending tooltip tweak (no ` so far`) here.
   - Export `dayStripMarkState` from `src/350-exports.js` next to `dayStripValueRuns`.
2. **`src/239-plugin-day-strip-render.js`**
   - Add `dayStripRenderMark(doc, state)`. It builds the SVG with `statsSvgEl`, defined
     in `239-plugin-stats-chart-render.js`, which loads earlier in `fragments.json`. Use
     the geometry table above.
   - Child classes: `bob-day-strip-mark` on the `svg`, plus `-disc`, `-rim`, `-ring`,
     `-glyph`, and `-half` on its children.
   - In `dayStripRenderFact`, when `fact.mark` is set and `doc.createElementNS` exists,
     paint:
     - `span.bob-day-strip-value.bob-day-strip-value-mark[data-mark=<state>]`;
     - the SVG inside it;
     - the hidden display-text span. The value never gets the `-value-word` class.
   - Without `createElementNS`, fall back to today's word runs unchanged.
   - For a skeleton fact with `unit === "bool"`, paint `.bob-day-strip-value-mark` with
     the skeleton class and one `span.bob-day-strip-mark-skeleton` circle instead of
     transparent text.
   - If the `celebrate` decision is `draw`, add the WeakMap memo in `paintDayStrip`.
     Read it before `replaceChildren`, write it after a `ready` paint, and pass an
     `arrive` flag into the fact render.
3. **`styles.css`** (the Day strip block, around `.bob-day-strip-value-word`)
   - Add the mark rules:
     - value-mark layout;
     - the `1.04em` SVG size with `overflow: visible`;
     - per-state ink custom property;
     - disc, rim, ring, glyph, and half paint;
     - the visually hidden text;
     - the round skeleton placeholder;
     - stale muted ink.
   - Add a `forced-colors` override inside the existing `@media (forced-colors: active)`
     block.
   - If the decision is `draw`, add the arrive keyframes.
   - Update the strip's header comment "Area hues are the only colour." to say that
     status marks also use the status inks.
4. Run `npm run build` to regenerate `plugins/bob-ledger-tools/main.js`.

## Tests

In `scripts/test-ledger-tools-day-strip.cjs`:

- Give the fake `doc` a `createElementNS` that returns `FakeNode`s, so the mark path is
  the default path exercised.
- **Mapping table for `dayStripMarkState`:**
  - met: value 1;
  - missed: value 0 on a complete day;
  - pending: provisional with value 0;
  - met: provisional with value 1;
  - no mark: value null, `missing`, `not_tracked`, a `count` unit, and an absent point.
- **Painter:**
  - A met, missed, or pending fact paints `.bob-day-strip-value-mark[data-mark]` with
    exactly one aria-hidden `svg.bob-day-strip-mark` holding that state's parts:
    - met and missed: disc, rim, glyph;
    - pending: disc, ring, half.
  - `value.textContent === fact.display`.
  - The fact's `aria-label` is unchanged, and there is no `-value-word` class.
  - Keep the existing all-fixtures "facts stack a label head over a faithful value" test
    green. Its word-class assertion must expect no word class on mark values.
- **Fallback:** a doc without `createElementNS` paints the word runs exactly as today.
- **Skeleton:** the bool fact paints `bob-day-strip-mark-skeleton` and no `svg`.
- **Tooltip:** pending reads `<short day>: not yet` with no ` so far`. A provisional
  numeric fact still ends ` so far`.
- **Reduced-motion guard (if `draw`):**
  - Paint pending then met on the same host and day → arrive class present.
  - Repaint met again → absent.
  - Absent for: pending→met on a different day, skeleton→met, a stale met frame, and
    pending→missed.
  - A stylesheet assertion: in `styles.css`, `stroke-dashoffset` appears only inside an
    `@keyframes` block.

## Chromium preview harness

The harness is `scripts/preview-day-strip.cjs` plus
`scripts/preview-day-strip.template.html`.

- Add Obsidian's default palette to both theme blocks in the template, so the matrix
  shows real inks instead of fallbacks:
  - light:
    `--color-red:#e93147; --color-orange:#ec7500; --color-yellow:#e0ac00; --color-green:#08b94e`;
  - dark:
    `--color-red:#fb464c; --color-orange:#e9973f; --color-yellow:#e0de71; --color-green:#44cf6e`.
- The existing cells already cover pending (today in progress), missed (complete past
  day), and met (`metPast`). Add a stale today cell if none exists.
- Add matrix checks, and list them in `matrixMarkdown`'s check sentence:
  - **(r) Geometry:** every `.bob-day-strip-mark` is square within ±0.5 px, measures
    0.95–1.12× its value's computed font size, lies inside its value box, and has its
    vertical centre within 1 px of the value box centre.
  - **(s) Inks:** in ready cells, the met, missed, and pending marks compute three
    distinct non-gray rim/ring colours. In stale cells, the mark ink computes to the
    muted text colour.
- Run `node scripts/preview-day-strip.cjs --out <fresh tmp dir>` and look at the
  700/540/410 px light and dark screenshots yourself. The report must list no failures.
  Headless Chrome may need `TMPDIR=/tmp` if its singleton socket path is too long.

## Docs

- **bob-cli `docs/stats.md`, `### Day strip` section** (this is the behavior contract):
  - "The displayed text is always bob's `display` exactly" now holds for numeric and
    word values. Boolean facts paint a mark chosen by the data rule, with bob's
    `display` kept as accessible text and tooltip.
  - In the display-states paragraph, replace "an unmet closeout reads `not yet`" and
    "`missed` is allowed" with the three marks (green check, red ✕, yellow In Progress
    half-ring) and the stale muted ink.
  - Note the round skeleton placeholder.
  - Say the strip's colours are area hues plus status inks for boolean marks.
  - Mention the one-shot check draw if the decision is `draw`.
  - Run `just docs-check`.
- **bob-plugins `docs/bob-ledger-tools.md`, Day strip paragraph:** add one clause saying
  Review closed shows a green check, a red ✕, or the yellow In Progress half-ring while
  today's review is still open. bob-cli's `docs/reference/plugins/` copy is assembled
  from this file and gitignored, so do not edit it.

## Verification and deploy

- **bob-plugins:**
  - Run `npm run build`, then `npm test`. The test script begins with `build:check` and
    the docs inventory check.
  - Run `npm run docs:check`.
  - Run the preview harness above.
  - Then run `bob plugins sync` so the vault gets the new plugin build.
- **bob-cli:** run `just docs-check`.
- **Commits:**
  - bob-plugins: one `feat(day-strip): …` commit.
  - bob-cli: one `docs(stats): …` commit.
  - Do not touch bob-cli's `Cargo.toml` version or `CHANGELOG.md`.

## Out of scope

- bob-cli stats computation, JSON, terminal marks, and display strings: all unchanged.
- Stats dash tiles. They show multi-day headlines like `2 of 2 days`, not per-day words.
- The checklist calendar cells in the Stats panels.
