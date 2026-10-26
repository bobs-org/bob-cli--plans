---
tier: epic
title: Day strip typical day — a quiet reference line under each numeric fact
goal: 'Under Worked, Reviewed, Done, and Created, every Day strip shows one quiet
  line with that metric''s typical day: the median of the seven days before D, needing
  at least four days with data. On finished days a neutral ▲, ▼, or = mark comes first,
  so the line reads `▼ typical 4h 10m`. Today shows `typical 4h 10m` with no mark.
  bob computes every number and word in a new additive `typical` object on day-grouped
  JSON points, and the plugin only paints it. The line has no colour, no percentages,
  and no bars. The strip never changes height when data arrives. The decision is recorded
  so no future agent "fixes" it back out.

  '
phases:
- id: typical-field
  title: bob stats — additive `typical` object on day-grouped count and minutes points
  depends_on: []
  size: medium
  description: 'typical-field: in bob-cli, add the optional `typical` object (window
    D−7…D−1, median, ≥4 usable days, relation, words, detail) to current day-grouped
    `points` of count and minutes metrics. Share the usable-value rule with rolling
    medians and extend the Day lead-in from 6 to 7 days. Cover it with hand-derived
    unit and golden tests, document the schema-1 field in docs/stats.md, and check
    parity read-only on the live vault.'
- id: strip-line
  title: Day strip — paint bob's typical line under the four numeric facts
  depends_on:
  - typical-field
  size: medium
  description: 'strip-line: in bob-plugins bob-ledger-tools, build a pure `ref` model
    from `point.typical` and paint an aria-hidden reference line with SVG ▲/▼/= marks
    on every fact. Keep the line''s space on every fact. Put bob''s words in the fact''s
    aria-label and tooltip. Require day-grouped reports for cache reuse, and repaint
    when only the reference changes. Extend the unit tests and the preview harness
    (new states, alignment, clipping, contrast, measured budgets). Update the plugin
    docs and the bob-cli Day strip contract, bump the plugin version, then sync.'
- id: record
  title: Decision record, Day Strip glossary strand, bead bob-cli-bs, three-week check
  depends_on:
  - strip-line
  size: small
  description: 'record: in bob-cli, add the decisions strand for the typical-day comparison
    and update the Day Strip glossary strand. Regenerate memory and docs, then close
    memory bead bob-cli-bs as superseded by the new record. Propose the three-week
    exit check as a follow-up for the land agent to file as a snoozed task bead.'
decisions:
  decision_record:
    ask: Add a decisions strand recording that the Day strip shows each numeric fact's
      typical day, never a target?
    default: true
    memory:
    - decisions:day-strip-shows-typical-day
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
  glossary_day_strip:
    ask: Update the Day Strip glossary strand so its contents list includes the typical
      line under each numeric fact?
    default: true
    memory:
    - glossary:day-strip
    corrects: 'Its contents: five LABEL-value facts, a [[root-parent]] legend with
      `so far · HH:MM` over the root-parent bar, and a [[pomodoro-theme]] legend over
      the full-width theme bar that separates it from the badges.'
    answer: true
proposed_by: bbugyi200.athena.research.5d.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 13:12:27
status: done
bead_id: bob-cli-cn
---

- **PROMPT:** [prompts/202610/day_strip_typical_day.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/day_strip_typical_day.md)
- **BEAD:** [bob-cli-cn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-cn/README.md)

# Plan: The Day strip's typical day

## Context

The Day strip is the read-only Stats receipt at the top of every canonical daily note's
`bob-plan` block. It has five LABEL-value facts, then a root-parent legend over its bar,
then a theme legend over the full-width theme bar. Bryan wants every numeric fact to
show how it compares with the previous seven days.

The research report
`research:202610/day_strip_vs_previous_week/day_strip_vs_previous_week.md` (read it with
`sase artifact read`) merged five independent designs and checked them against the live
vault. Bryan accepted **all** of its requirements, R1 to R9:

- **R1, median.** Use the median, labelled "typical", not the mean. One 86-task day made
  a normal week read "below average", and against the mean Done flips direction on 19%
  of days. The median also matches the Stats headlines and the chart's 7-day line.
- **R2, window.** The window is D−7…D−1. D is excluded, real zeros count, missing,
  not-tracked, and provisional days are skipped, partial days are included, and at least
  4 usable days are needed. This is the engine's existing rolling-median rule, so
  `typical(D) == rolling_median(D−1)`.
- **R3, no verdict on today.** Today shows the reference with no ▲/▼, because only about
  53% of a day's Worked minutes are logged by noon.
- **R4, neutral text.** One neutral text line with a ▲/▼/= mark. No colour, percentages,
  deltas, bars, or sparklines.
- **R5, four metrics.** Worked, Reviewed, Done, and Created get the line. Review closed
  keeps its status mark and gets no average.
- **R6, bob computes.** bob computes the value, the relation, and the words. The plugin
  paints them and calculates nothing.
- **R7, height.** Raise the height budget rather than shrink the values.
- **R8, record it.** Record the decision and re-scope memory bead `bob-cli-bs`, which
  proposed "never targets (no deltas…)".
- **R9, exit check.** Reassess after about three weeks.

Figure 1 of the report is `day_strip_vs_previous_week__final_mockup.png`, next to the
report in the research sidecar (open that sidecar with `/sase_repo` to view it). It
shows the target look, rendered with the production painter.

**What changed since the research (bob-cli `38a02d7b`, bob-plugins `7d005fb`):**

- **The theme lane landed** (bob-plugins `457db03`, bob-cli `5e8c6ecd`). The strip is
  now two lanes, and its budgets are **116 px on one line and 167 px folded**, enforced
  by `scripts/preview-day-strip.cjs` and documented in `docs/stats.md`.
- **The research's 98/162 px budgets are therefore stale.** This plan sets new budgets
  from measurement instead. The research measured +14.2 px per row of facts, so expect
  about 132/198 px.
- **The painted-key guard would hide the new field.** The registry's guard in
  `239-plugin-day-strip-registry.js` (`finish`) keys only on `fact.display`. A report
  whose only change is the new `typical` field would never repaint unless that key
  grows.

## Design contract (every phase implements exactly this)

### What the reader sees

A finished day (the facts row only; both lanes below are unchanged):

```text
WORKED              REVIEWED          DONE              CREATED           REVIEW CLOSED
3h 25m              75                17                9                 ✓
▼ typical 4h 10m    ▲ typical 54      ▼ typical 19      ▼ typical 23
```

Today, at 12:16:

```text
WORKED              REVIEWED          DONE              CREATED           REVIEW CLOSED
2h 10m              43                7                 10                ◐
typical 4h 10m      typical 54        typical 17        typical 22
```

The strip now has three tiers. The small uppercase **label**, then the large tabular
**value**, then the quiet sentence-case **reference**. The value always dominates. The
reference lines share one baseline across the row and sit under the existing hairline
column separators. The root-parent and theme bars remain the strip's only graphics.

### States (one row per situation; the line's space is always kept)

| Situation                                 | Reference line                                   | Fact aria-label (plugin)                             |
| ----------------------------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| Finished day, above                       | `▲ typical 4h 20m`                               | `Worked 5h 00m, 40m above the typical day of 4h 20m` |
| Finished day, below                       | `▼ typical 4h 10m`                               | `Worked 3h 25m, 45m below the typical day of 4h 10m` |
| Finished day, equal                       | `= typical 25`                                   | `Done 25, the same as the typical day of 25`         |
| Today (provisional)                       | `typical 4h 10m` (no mark, flush left)           | `Worked 2h 10m so far, typical full day 4h 10m`      |
| Fewer than 4 usable days                  | `typical –`                                      | `Reviewed 60, no typical day yet`                    |
| D has no value (`missing`, `not_tracked`) | blank                                            | unchanged (`Reviewed –`)                             |
| Review closed (bool)                      | blank                                            | unchanged                                            |
| Skeleton (first load)                     | blank                                            | unchanged (`Worked, loading`)                        |
| Stale cached frame                        | same text; the number drops to the line's weight | as for the underlying state                          |
| Older bob (no `typical` on the point)     | blank                                            | unchanged                                            |

- **The mark comes first,** so a finished day reads "below a typical 4h 10m". Today has
  no mark slot, so its text starts flush with the value's left edge. A note only turns
  from today into a past day at midnight, so the one-glyph shift never happens in front
  of the reader.
- **The tooltip** (`title`) gains bob's `detail` sentence directly after the existing
  date line (`Sun Oct 25: 3h 25m`), for the ready and short states only.
- **The strip group's `aria-label` is unchanged.** Each fact carries its own clause.
  There is no live-region announcement, no nested button, and no extra tab stop. The
  fact stays the single link to Stats, with D pinned.

### Words (bob-owned; the plugin never builds them)

Notation for the table below:

- `T` is the typical display, `format_value(typical, unit)`.
- `A` is the absolute difference in the metric's unit: `format_minutes` for minutes, the
  integer for counts.
- `S` is `format_date_span(start..end)`, for example `Oct 18 – Oct 24`, or
  `Dec 26, 2026 – Jan 1, 2027` when the span crosses a year.
- `n` is the number of usable days.

| Case           | `relation` | `words`                              | `detail`                                                                                                                     |
| -------------- | ---------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Above          | `above`    | `{A} above the typical day of {T}`   | `Typical day {T}: median of {S} ({n} of 7 days with data). This day is {A} above.`                                           |
| Below          | `below`    | `{A} below the typical day of {T}`   | `Typical day {T}: median of {S} ({n} of 7 days with data). This day is {A} below.`                                           |
| Equal          | `equal`    | `the same as the typical day of {T}` | `Typical day {T}: median of {S} ({n} of 7 days with data). This day matches it.`                                             |
| D provisional  | `null`     | `typical full day {T}`               | `Typical full day {T}: median of {S} ({n} of 7 days with data). Today is still in progress, so this is context, not a pace.` |
| D has no value | `null`     | `typical day {T}`                    | `Typical day {T}: median of {S} ({n} of 7 days with data).`                                                                  |
| n < 4          | `null`     | `no typical day yet`                 | `No typical day yet: {n} of the previous 7 days {has\|have} data (needs 4).`                                                 |

In the n < 4 row, `has` is for n = 1 and `have` is for everything else, including 0.
Examples: `1h 05m above the typical day of 3h 20m`, `3 above the typical day of 0`, and
`Typical day 4h 10m: median of Oct 18 – Oct 24 (7 of 7 days with data). This day is 45m below.`

### Data contract (bob-cli, schema 1, additive)

Each element of a metric's `points` array gains an optional `typical` object, but only
when all of these hold:

- the report's group is `day`;
- the metric's unit is `count` or `minutes`;
- the point belongs to the current window's `points` array.

```json
{
  "date": "2026-10-25",
  "end": "2026-10-25",
  "value": 205,
  "display": "3h 25m",
  "quality": "ok",
  "provisional": false,
  "rolling_median": 240,
  "typical": {
    "start": "2026-10-18",
    "end": "2026-10-24",
    "days": 7,
    "value": 250,
    "display": "4h 10m",
    "relation": "below",
    "words": "45m below the typical day of 4h 10m",
    "detail": "Typical day 4h 10m: median of Oct 18 – Oct 24 (7 of 7 days with data). This day is 45m below."
  }
}
```

- **Short history.** With fewer than 4 usable days, `value` and `relation` are `null`
  and `display` is `"–"` (the same null display that `format_value` gives points).
  `days` says how many days were usable.
- **`relation` is `null`** when D is provisional, when D has no value, or when history
  is short.
- **Comparison at display precision.** `relation` compares `D.value` with
  `typical.value` as integers. Minutes and counts already display as exact integers, so
  this comparison is the display precision.
- **Never present** on `prior_points`, on week or month `points`, on `daily_points`, or
  on bool, ratio, or signed metrics. Human and CSV output do not change. No report-level
  definition object is added; the `detail` sentence carries the definition.

### Visual spec (bob-ledger-tools `styles.css`)

```css
/* third tier: label / value / reference; reserved on every fact */
.bob-day-strip-ref {
  display: flex;
  align-items: baseline;
  gap: 0.3em;
  height: 1.25em;
  margin-top: -0.05em;
  color: var(--text-muted); /* never --text-faint: it fails 4.5:1 */
  font-size: 0.72em;
  font-weight: 450;
  line-height: 1.25;
  letter-spacing: 0;
  font-variant-numeric: tabular-nums lining-nums;
  white-space: nowrap;
}
.bob-day-strip-ref-mark {
  flex: none;
  width: 0.62em;
  height: 0.62em;
  fill: currentColor;
}
.bob-day-strip-ref-num {
  font-weight: 600;
} /* the number carries the line */
.bob-day-strip-ref[data-ref-state="short"] .bob-day-strip-ref-num {
  font-weight: 450;
}
.bob-day-strip-fact:hover .bob-day-strip-ref {
  color: var(--text-normal);
} /* mirrors the head */
.bob-day-strip-stale .bob-day-strip-ref-num {
  font-weight: 450;
} /* steps back with the numerals */
@media (forced-colors: active) {
  .bob-day-strip-ref {
    color: CanvasText;
  }
}
```

- **Marks are inline SVG, never text glyphs.** U+25B2, U+25BC, and `=` fall back to
  different fonts with different weights and baselines across macOS, Linux, and the
  Obsidian themes. Small SVGs in `currentColor` render identically everywhere, the same
  way the boolean status marks do.
- **Mark geometry.** Use `viewBox="0 0 10 10"` and `aria-hidden`/`focusable="false"`:
  - above: `M5 1.6 L8.8 8 H1.2 Z`;
  - below: `M1.2 2 H8.8 L5 8.4 Z`;
  - equal: two rounded bars `M1.5 2.9h7v1.5h-7z M1.5 5.6h7v1.5h-7z`.

  With `align-items: baseline`, each mark rests on the text baseline just under cap
  height. It is quiet but unmistakable.

- **No hue anywhere.** Hue already means area, theme, and status in this strip. A 9-hour
  Worked day is information, not praise, and Created has no good direction.
- **The line keeps its space on every fact,** bool and skeleton facts included, so the
  skeleton equals the loaded height and data arriving never moves anything.
- **The reference never clips.** It is wider than some values (`9` over `▼ typical 23`),
  so it must contribute its full width to its grid track. Do not give it `min-width: 0`
  or ellipsis.

## Phase `typical-field` (bob-cli)

Work in bob-cli on `src/native/stats/`.

1. **`report.rs`.** Add
   `TypicalDay { start, end, days, value, display, relation, words, detail }`, with
   `iso_date` serializers on the dates. Add a `TypicalRelation` enum with
   `serde(rename_all = "snake_case")` (`above | below | equal`). Add
   `SeriesPoint.typical: Option<TypicalDay>` with
   `#[serde(skip_serializing_if = "Option::is_none")]`, and set `typical: None` in every
   existing constructor (`series.rs`, `render.rs`, `report.rs` tests).
2. **New `typical.rs`** (wired in `stats/mod.rs`). It holds the pure computation and the
   word builders. Reuse `series.rs`'s `median`, `format_value`, `format_minutes`, and
   `format_date_span`; widen them to `pub(super)` as needed rather than copying them.
3. **One usable-value rule.** Factor one helper and the constants `DAY_WINDOW = 7` and
   `DAY_WINDOW_MIN = 4` (`7.div_ceil(2)`) out of `add_rolling_medians`. The helper keeps
   points with `!provisional && value.is_some()`. Both rolling medians and the typical
   day call it, so they cannot drift apart.
   - `missing` and `not_tracked` points always have `value: None`. Verified in
     `metrics.rs` and `formula.rs`.
4. **Where to compute.** In `make_points`, for `GroupName::Day` only, after
   `add_rolling_medians` and before `drop_before`. Gate it with a parameter that is true
   only for the current `points`; `prior_points` never get it. Only `Count` and
   `Minutes` units qualify.
   - **Select the window by calendar date** (`D−7 ≤ date ≤ D−1`), not by index, so a gap
     can never shift it.
5. **Lead-in.** Change `lead_in_start` for `Day` from 6 to 7 days, so the first
   displayed point has a full window, and update its doc comment ("rolling median and
   typical day"). Rolling medians are unaffected, because they look back only 6 days.
   `evaluation_dates` follows automatically. Verify that the golden warning counts are
   unchanged.
6. **Words.** Build `words` and `detail` exactly as in the Words table.
7. **Tests.** Derive every expected value by hand; never snapshot the implementation.
   - **Window:** D is excluded; real zeros count; missing and not-tracked days are
     skipped; partial days are included.
   - **History:** 3 usable days gives short (`value` null, `display` `"–"`, `relation`
     null), and 4 gives ready. Cover the n = 0, 1, and 3 grammar.
   - **Today:** a provisional D has `relation: null` and today's words.
   - **Live-vault vectors from the research.**
     - Prior week `86, 7, 21, 49, 1, 7, 6`: D 25 gives typical 7, `above`.
     - Prior week `18, 37, 17, 31, 25, 86, 7`: D 25 gives typical 25, `equal`.
     - Prior week `25, 18, 37, 17, 31, 25, 86`: D 27 gives typical 25, `above`.
     - The spike must not drag the next week's typical day.
   - **Even-count medians** round half away from zero, and `relation` compares against
     that rounded value.
   - **Parity:** in a long day series, every `points[i].typical.value` equals
     `points[i-1].rolling_median`.
   - **Lead-in:** an explicit one-day range `D..D` still gets `days: 7` when the vault
     has data.
   - **Unchanged and omitted:** the existing rolling-median tests pass untouched. There
     is no `typical` on bool, ratio, or signed metrics, on `prior_points` (with
     `compare`), on week or month points, or on `daily_points`, and the JSON omits the
     key there.
   - **Calendar:** month boundaries (Nov 3 → Oct 27–Nov 2), the year boundary (Jan 2,
     2027 → `Dec 26, 2026 – Jan 1, 2027`), and the leap day (Mar 1, 2028 includes Feb
     29).
   - **Words:** amounts of 60 minutes or more (`1h 05m`), counts, equal, and a zero
     typical day.
   - **Golden fixture** (`compute.rs`): assert hand-derived `typical` objects for
     `worked` and `done` on every displayed day. For example, `worked` on Oct 10 has a
     window of Oct 3–9 with one usable day, so it is short with
     `1 of the previous 7 days has data (needs 4).`; today, Oct 11, has 2 usable days.
     Also assert at JSON level that bool `review_closed` and ratio `habits` points carry
     no `typical` key.
8. **Docs** (`docs/stats.md`; follow the `docs.md` memory routing).
   - Add a short "Typical day" paragraph to "Ranges, grouping, summaries, display" that
     gives the definition, the window, and the threshold.
   - In "Schema-1 contract", add `typical` to the point fields and to the
     optional-fields list, with the JSON example above.
   - Update the "Golden fixture" prose for the new assertions.
   - Re-measure the default 4-week JSON bytes and the compute time for the "Performance"
     section. Use the locally built binary against the live vault (read-only). Expect
     about +8% bytes, and compute time within 10% of before.
9. **Live parity, read-only.** Run the local build:
   `cargo run --release -- stats show worked reviewed done created -f json -g day -r 2026-10-18..2026-10-26`,
   plus a range around Oct 6. Compare with the research's values:
   - Oct 25: Worked 3h 25m ▼ typical 4h 10m; Reviewed 75 ▲ typical 54; Done 17 ▼ typical
     19; Created 9 ▼ typical 23.
   - Oct 6: Done 86 ▲ typical 7.

   Record the results, and explain any drift from vault edits, in a note on this phase's
   bead.

10. **Gate:** `just check` (pre-existing unrelated failures may exist; list them, don't
    fix them) and `just docs-check`. Commit as
    `feat(stats): add typical day to daily points`. Don't add a `!`, and don't edit the
    Cargo.toml version or CHANGELOG.md
    (`decisions:bob-cli-releases-ride-a-daily-train`).

## Phase `strip-line` (bob-plugins bob-ledger-tools, plus bob-cli docs)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Edit the `src/` fragments,
run `npm run build`, and never hand-edit `main.js`. Every fragment must stay at or below
1000 lines; `239-plugin-day-strip-registry.js` is already at 906.

1. **Model** (`239-plugin-day-strip-model.js`).
   - **Pure helper.** Add `dayStripRef(unit, point)`, which returns
     `{state: "ready" | "short" | "blank", mark, display, words, detail}`. It computes
     nothing:
     - **blank** when the unit is not `minutes` or `count`, the point or a well-formed
       `point.typical` is absent, `typical.display` is not a string, or D has no finite
       value;
     - **short** when `typical.value` is null;
     - **ready** otherwise.
   - **The mark** is bob's `relation` when it is one of `above`, `below`, or `equal`;
     otherwise it is null. As a guard, the mark is also null whenever
     `point.provisional` is true.
   - **`dayStripBuildFact`** adds `ref` and builds `ariaLabel`, using the shape in the
     States table: `<label> <display>[ so far], <words>` for ready and short facts.
     Insert `detail` into `tooltip` right after the date line. Skeleton facts get
     `ref: {state: "blank"}`. Export the helper through `350-exports.js` for tests.
2. **Painter** (`239-plugin-day-strip-render.js`).
   - **Every fact** in the ready and skeleton states gets one
     `span.bob-day-strip-ref[aria-hidden="true"]` after its value, with `data-ref-state`
     and `data-ref-mark`.
   - **Ready:** the SVG mark (if any), then `span.bob-day-strip-ref-word` containing
     `typical` (fixed strip copy, like `so far`), then `span.bob-day-strip-ref-num`
     containing bob's display.
   - **Short:** the same, without a mark.
   - **Blank:** an empty line.
   - **The fact's `aria-label`** comes from `fact.ariaLabel`.
3. **Styles:** the CSS in the Visual spec.
4. **Hosts** (`239-plugin-day-strip-hosts.js`).
   - `dayStripDefaultReportCovers` returns false for a report whose `range.group` is
     `week` or `month`. Its `daily_points` never carry `typical`; that happens when
     `stats.range` is 92 days or longer.
   - When a cold-cache default fetch for today settles with such a report, re-run
     selection for that strip at once, so it fetches the explicit `-g day` range. It
     must not wait for the next vault change or the ten-minute refresh.
5. **Repaint key** (`239-plugin-day-strip-registry.js` `finish`). Add each fact's
   `ref.state`, `ref.mark`, and `ref.display` to the painted key. Then a report that
   only adds or changes `typical` repaints.
6. **Unit tests**:
   - **`test-ledger-tools-day-strip.cjs`** covers every row of the States table,
     including the mark guard on provisional points. It checks that the aria-label ends
     with bob's words, that the line is aria-hidden and is never a button or a tab stop,
     that lines are reserved on bool and skeleton facts, that a stale frame keeps the
     line, and that an older-bob report renders blank lines.
   - **`test-ledger-tools-day-strip-hosts.cjs`** checks that a week-grouped default
     never covers D, and that the cold-cache case re-selects once.
   - **The registry tests** check that a report differing only in `typical` repaints.
7. **Preview harness** (`scripts/preview-day-strip.cjs`, the template, and
   `plugins/bob-ledger-tools/scripts/fixtures/stats/day-strip-fixtures.cjs`).
   - **Fixtures.** Add `typical` to the day points of the fixtures, then add these
     states:
     - past-day: Worked below, Reviewed above, Done equal, Created below;
     - today with no marks;
     - short history;
     - D missing (blank);
     - old-bob (no `typical`);
     - wide-values, with `▲ typical 10h 05m`.
   - **New checks:**
     - every fact has exactly one aria-hidden reference line;
     - reference lines share a baseline within 0.5 px on each wrapped row;
     - no reference line is clipped or spills past its fact (`scrollWidth` ≤
       `clientWidth` + 0.5);
     - marks are square within ±0.5 px, lie inside their line, and have a vertical
       centre within 1 px of the number box's centre;
     - the line's computed colour has at least 4.5:1 contrast with the strip background
       in light and dark;
     - old-bob, ready, and skeleton heights are equal within 0.5 px.
   - **Updated check (o):** the aria-label starts with `<label> <display>` and, when a
     reference is ready or short, ends with bob's words.
   - **Budgets.** Set `DAY_STRIP_HEIGHT_ONE_LINE` and `DAY_STRIP_HEIGHT_WRAPPED` to the
     new measured maxima plus 2 px; expect about 132 and 198 px.
     - **Stop condition:** if the increase is more than 16 px per row of facts, tighten
       the line's margins first. Never shrink the values or labels to pay for it.
   - **Visual review.** Look at the 410, 540, and 700 px screenshots in light and dark
     against Figure 1 before calling the phase done.
8. **Live parity.** Feed the strip model the JSON from phase `typical-field`'s local
   build for Oct 6, Oct 22, Oct 25, and today. Confirm three things, then record them on
   this phase's bead:
   - each painted reference equals bob's `typical.display`;
   - each mark matches `relation`;
   - the typical value equals the Stats chart's rolling median at D−1.
9. **Docs.**
   - **bob-plugins `docs/bob-ledger-tools.md`:** add one sentence on the reference line
     to the Day strip paragraph, and a short clause to the manifest description. Then
     run `npm run docs:gen`.
   - **bob-cli `docs/stats.md` § Day strip:**
     - describe the third tier, the States table, the words and tooltip source, and the
       new height budgets (replacing 116/167);
     - add the day-group requirement to "Data";
     - change "The strip derives its colours…" and the fixed-copy list to include
       `typical`.
10. **Release.**
    - Bump the bob-ledger-tools minor version, following the convention of `457db03`.
    - Run `npm test`, `npm run validate`, `npm run docs:check`, and the preview harness,
      then `bob plugins sync`.
    - Don't install a locally built bob. Until the next bob release lands through the
      release train, the deployed strip shows blank reserved lines. That state is
      expected and correct.
    - Commit bob-plugins as `feat(day-strip): show each fact's typical day`. Commit the
      bob-cli docs as `docs(stats): describe the Day strip typical line`.

## Phase `record` (bob-cli memory, docs, beads)

Use `/sase_memory_write` before any memory edit, and stay inside the accepted memory
decisions.

> [!decision] decision_record Add the new strand `decisions:day-strip-shows-typical-day`
> (slug `day-strip-shows-typical-day` in the decisions web), shaped like the other
> decisions strands. Use `metadata.status: accepted` and `decided:` set to the landing
> date.
>
> - **keyword:** `The Day Strip Shows Each Fact's Typical Day, Never A Target`
> - **aliases:** `typical day`, `day strip comparison`, `vs previous week`
> - **summary:** "Each numeric Day strip fact shows bob's typical day (median of the 7
>   days before D, at least 4 with data) as neutral text; finished days add ▲/▼/=, today
>   none; no colour, percentages, goals, streaks, or bars."
> - **Applies to:** bob-cli day-grouped `bob stats` points (`typical`), the
>   bob-ledger-tools Day strip, and any later surface comparing one day with recent
>   days.
> - **Claim:** the design contract above, in one paragraph.
> - **Why**, with the rejected alternatives:
>   - mean / `7d avg`: the spike example, the 19% flip rate, and Stats medians;
>   - delta captions: they hide what normal is;
>   - bullet rail or dashed progress fill: it must be learned, and on today it becomes a
>     progress bar toward a target;
>   - 8-bar sparkline: it brings back the removed history marks;
>   - green/orange tint: hue already means area, theme, and status;
>   - a verdict on today: only about 53% of Worked is logged by noon;
>   - plugin-side computation: it breaks "bob owns every value";
>   - facts only, never a comparison: the stance proposed in bead `bob-cli-bs` and
>     `plan:202610/day_strip_under_the_badges.md`, reversed by Bryan's request.
> - **Evidence:** the research ref above, this epic's plan ref (shown by
>   `sase bead read` on the epic), `docs/stats.md` § Day strip, and the phase commits.
> - **Cost:** about 14 px per row of facts; marks flip about every other day; on today's
>   note the feature is only a number; about +8% default-report bytes.
> - **Reopens when:** the three-week check finds one of these:
>   - Bryan never looks at the line: remove it (plugin-only, no engine revert);
>   - he chases it: switch to recap mode, with lines on finished days only;
>   - the marks distract: drop the marks and keep the reference.
>
> Then close `bob-cli-bs`:
> `sase bead close bob-cli-bs -R superseded --reason "Superseded by decisions:day-strip-shows-typical-day, which keeps its never-targets rule (no goals, percentages, streaks, or status colours) and adds the neutral typical-day reference"`.

> [!decision] decision_record = no Write no strand. Add a `sase bead note` to
> `bob-cli-bs` saying the strip now shows a neutral typical-day reference, so its "no
> deltas" wording needs Bryan's re-scope before anyone records it. Then record the
> skipped strand as a `PROPOSED FOLLOW-UP:` note on this phase's bead.

> [!decision] glossary_day_strip Rewrite only the contents sentence of the
> `glossary:day-strip` strand to read: "Its contents: five LABEL-value facts, each
> numeric one over a quiet `typical` line (the median of the seven days before, with a
> ▲/▼/= mark once the day is over), a [[root-parent]] legend with `so far · HH:MM` over
> the root-parent bar, and a [[pomodoro-theme]] legend over the full-width theme bar
> that separates it from the badges." Link the new decision,
> `[[decisions/day-strip-shows-typical-day]]`, when that strand exists.

> [!decision] glossary_day_strip = no Leave the strand. Record the proposed sentence as
> a `PROPOSED FOLLOW-UP:` note on this phase's bead.

Then:

1. Run `sase memory init`, `just docs-gen` (which regenerates
   `docs/reference/glossary.md`), and `just docs-check`. Commit as
   `docs(memory): record the Day strip typical-day decision`.
2. Append `PROPOSED FOLLOW-UP: Day strip typical-day exit check` to this phase's bead,
   for the land agent to file with `/sase_new_task` as a task bead snoozed to about
   three weeks after the strip deploy (around 2026-11-16). The task asks Bryan two
   questions:
   - did the line change a decision or usefully reassure you?
   - did you catch yourself working to beat the typical number?

   It then applies the decision's Reopens-when actions.

## Out of scope (follow-ups, not v1)

- Repeating the line on the Stats page's pinned-day view, or adding a "vs typical"
  column to the Stats table.
- Weekly trend tiles ("is my rhythm changing?"), which are a Stats-page question.
- A pace mark for Worked only.
- A strip-wide hover/focus disclosure to replace `title` tooltips (WCAG 1.4.13).
- Human or CSV `bob stats` output for the typical day.
- A machine-readable split of `partial` into estimated versus incomplete. The trigger is
  if partial days become common.

## Done means

- `bob stats … -f json -g day` carries hand-verified `typical` objects with the exact
  contract above, and `rolling_median` is unchanged.
- The strip paints the States table pixel-for-pixel in the harness at 410, 540, and 700
  px, in light and dark.
- Heights stay within the new measured budgets, the skeleton equals the loaded height,
  and nothing clips.
- Live parity holds for Oct 6, Oct 22, Oct 25, and today.
- The plugin is synced to the vault.
- The decision and the glossary are recorded per the accepted memory decisions.
- `bob-cli-bs` is resolved, and the three-week check is proposed.
