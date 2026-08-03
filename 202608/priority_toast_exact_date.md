---
tier: tale
title: Give the priority toast a durable exact-date receipt
goal:
  Priority write toasts visibly retain the exact YYYY-MM-DD date or counted date span alongside the relative-day summary
  in every supported scope and fallback.
proposed_by: bbugyi200.athena.sj.f0
create_time: 2026-08-03 07:07:06
status: done
---

- **PROMPT:**
  [prompts/202608/priority_toast_exact_date.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/priority_toast_exact_date.md)

# Plan: Give the priority toast a durable exact-date receipt

## Goal

Keep the redesigned `Ctrl+Shift+P` → `priority` toast's relative-day count as its visual headline while making the exact
scheduled value unmistakably visible in `YYYY-MM-DD` form. Single-task and `^prj` writes must show the one date that was
written; counted writes must show both exact endpoints of the actual rolled span. The DOM notice and its plain-text
fallback must derive from one model and expose the same scheduling facts.

## Tier and repositories

This is a **tale**: one follow-up coding agent can implement and verify the presentation refinement atomically.

All implementation lives in the linked `bob-plugins` repository. Before reading or changing it, open it through the
required repository workflow and use the printed path for every command:

```text
sase repo open bob-plugins -r "Make exact ISO dates explicit in priority notices"
```

Relevant files there are:

- `plugins/bob-navigation-hotkeys/main.js` — priority notice schedule summary, model, plain-text formatter, renderer,
  and all three priority write paths.
- `plugins/bob-navigation-hotkeys/styles.css` — fully scoped notice layout and date-receipt styling.
- `scripts/test-navigation-hotkeys.cjs` — pure model, fragment renderer, fallback, and end-to-end priority tests.
- `plugins/bob-navigation-hotkeys/manifest.json` and root `README.md` — patch version and plugin inventory text.

The primary `bob-cli` repository's `docs/projects.md` already promises “the rolled ISO date with weekday” for this
toast. Treat that sentence as an acceptance criterion; it does not need a wording-only edit unless implementation
changes the documented vocabulary.

Run tests and deployment commands from the opened `bob-plugins` checkout. Deploy with
`bob plugins sync -p bob-navigation-hotkeys -r "$PWD"`; the explicit source root is required in SASE workspaces.

## Current behavior and diagnosis

The recently-added notice pipeline is already centralized correctly:

1. `setBulletPriorityValue()` and `setCountedBulletPriorityValue()` pass the dates that were actually written into
   `buildPriorityNoticeModel()` for ordinary, project, and counted scopes.
2. `getPriorityNoticeScheduleSummary()` validates those values, sorts valid counted dates, and currently folds the ISO
   value and weekday into one composite `dateText` string.
3. `formatPriorityNoticeText()` includes that composite text in the non-DOM fallback.
4. `renderPriorityNoticeFragment()` places `model.dateText` into a generic `.bob-nh-notice-date-value` span on the same
   flexible row as the label and relative count.

That means the scheduling data has not been lost, and no rolling or write-path change is needed. The weakness is in the
visual contract: the exact ISO value is merely part of a composite display string, has no semantic model field or
distinct visual treatment, and the renderer test asserts only the relative text and status chip. A future or existing
presentation regression can therefore make the date effectively absent without a test failure.

This change promotes the exact written date from incidental text to a first-class “receipt,” just as the header already
does for `[priority:: high]`.

## Design

### Schedule block

Use a compact two-line schedule block so the relative and exact forms reinforce rather than compete with each other:

```text
  ⚄ scheduled                         in 3 days
    2026-08-06  · Thu
```

For a counted write:

```text
  ⚄ scheduled                       in 2–7 days
    2026-08-05  →  2026-08-10
```

For a project write, retain the existing target cue:

```text
  ⚄ scheduled (project)               in 2 days
    2026-08-05  · Wed
```

The first line answers “how far away?” at a glance. The second line is the durable vault receipt: monospace,
tabular-numeric, copyable text showing exactly what landed. Keeping it on its own line makes it reliable on narrow
desktop and mobile notices instead of depending on flex wrapping among three peers.

For a single scheduled day, show one exact ISO value and its weekday. If every counted task lands on the same day,
collapse to that same form. If counted tasks land on different days, show the sorted minimum and maximum ISO values with
a visible arrow; do not synthesize intermediate dates or list every roll. For an invalid/non-date value, preserve the
raw text as the receipt and omit weekday and relative text, matching today's graceful fallback.

### Semantic model

Refine `getPriorityNoticeScheduleSummary()` and `buildPriorityNoticeModel()` so rendering does not have to parse a
preformatted `dateText`. The schedule summary should expose frozen semantic fields along these lines:

```js
{
  exactDateText, // "2026-08-06" or "2026-08-05 → 2026-08-10"
  weekdayText,   // "Thu" for a collapsed/single day, otherwise ""
  relativeText,  // "in 3 days" or "in 2–7 days"
  textDateText,  // stable plain-text form, if useful to avoid duplicating punctuation
}
```

Exact property names may follow nearby conventions, but the model must contain a dedicated exact-date field rather than
requiring the renderer to recover it from a composite string. Keep the model frozen and keep all strings derived from
the validated scheduled values actually passed by the writers.

Preserve the established plain-text wording so headless/failure environments do not regress:

```text
priority → P2 (high); scheduled → 2026-08-06 · Thu · in 3 days; marked task Blocked
priority → P2 (high) on 3 tasks; scheduled → 2026-08-05 to 2026-08-10 · in 2–7 days; marked 3 tasks Blocked
```

It is acceptable for the DOM-only counted receipt to use an arrow while the linear fallback retains `to`; the semantic
endpoints must be identical. Do not change priority/schedule writes, random rolling, Blocked/recovery behavior, notice
count, accessibility text, or timeout behavior.

### DOM and accessibility

Update `renderPriorityNoticeFragment()` to render:

- a `.bob-nh-notice-date-heading` containing the existing dice icon, scope-aware label, and relative text;
- a `.bob-nh-notice-date-receipt` beneath it;
- one `.bob-nh-notice-date-iso` span for a single value, or two such spans separated by an aria-hidden visual arrow for
  a range;
- a muted weekday span only for a single/collapsed date.

Build all nodes with Obsidian's `createDiv`/`createSpan` helpers and text properties; never use `innerHTML`. The card's
existing `aria-label = model.text` remains the screen-reader linearization and must continue to contain every exact
date. Mark only a decorative range arrow as hidden; do not hide the dates themselves. If splitting the range into
separate spans would force renderer-side parsing, instead model the endpoints separately—the model, not the DOM, owns
date semantics.

Keep `showPriorityNotice()`'s single-notice, catch-and-fallback behavior unchanged. A renderer failure must still emit
exactly one plain string containing the ISO date(s).

### Styling

Modify only rules scoped beneath `.bob-nh-notice`:

- make the outer date block a small column and the heading a wrapping flex row;
- align the receipt beneath the heading content with a modest indent that can collapse on narrow widths;
- style exact ISO spans with `var(--font-monospace)`, tabular numerals, normal foreground contrast, a subtle
  `var(--background-modifier-border)`-based surface/border, `var(--radius-s)`, and compact padding;
- keep `.bob-nh-notice-relative` as the strongest accent and keep weekday/arrow muted;
- allow long or invalid raw values to wrap without widening or clipping the Obsidian notice;
- use only Obsidian variables and the repository's existing `color-mix(...)` idiom; add no fixed notice width,
  animation, or literal color.

The exact receipt should be visibly secondary to `in N days`, but should not use such low contrast that it disappears in
either theme.

## Implementation sequence

1. Refactor the priority schedule summary/model into explicit exact-date endpoint, weekday, relative, and fallback-text
   fields. Preserve its date validation, chronological sorting, same-day collapse, frozen outputs, and DST-safe offset
   math.
2. Update `formatPriorityNoticeText()` to consume the semantic model while keeping existing fallback strings stable.
3. Restructure only the date portion of `renderPriorityNoticeFragment()` into the two-line block. Leave the header,
   receipt, level classes, status chips, and notice fallback seam intact.
4. Update the scoped priority-notice CSS for the schedule block and exact-date receipt.
5. Add and strengthen tests before changing release metadata.
6. Bump Bob Navigation Hotkeys from `1.15.0` to `1.15.1` and update the root README row to say priority notices report
   both the exact rolled ISO date/span and its distance from today.

## Automated verification

Extend `scripts/test-navigation-hotkeys.cjs` in the existing notice test section.

Model tests:

- single task: exact date `2026-08-06`, weekday `Thu`, relative `in 3 days`, and unchanged fallback sentence;
- project: the same exact-date fields plus `scheduled (project)` and unchanged outcome chips;
- counted range supplied out of order: sorted exact endpoints `2026-08-05` and `2026-08-10`, relative `in 2–7 days`, and
  both endpoints in the fallback text;
- counted values all on one day: one exact value plus weekday, not a duplicated range;
- invalid value: raw receipt remains visible, relative/weekday are empty, and neither DOM nor text contains `NaN`.

Renderer tests:

- assert the dedicated `.bob-nh-notice-date-iso` node contains `2026-08-06`, not merely that the model contains it;
- assert a counted fragment contains both ISO endpoint nodes in chronological order and the range affordance;
- assert the relative node and scope-aware label are still rendered;
- assert `aria-label` contains the exact date(s), and the chips row remains omitted when empty;
- exercise a long invalid receipt to confirm the renderer preserves its text without throwing.

End-to-end regression tests:

- retain the existing single, counted, and `^prj` write assertions proving the fallback notice includes exact values;
- retain exactly-one-notice assertions and the actual file/frontmatter write assertions;
- leave plain `scheduled`, `dependsOn`, scalar, delete, and pinned priority-roll date-picker notices byte-identical.

Run:

```text
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

The full suite is currently 273 tests and manifest validation is 6/6; all must pass after the additions.

## Release, deploy, and manual verification

Review the complete linked-repo diff and commit only the scoped source, styles, tests, manifest, and README changes
through the normal SASE commit workflow. Then run:

```text
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD"
```

If sync reports a dirty vault plugin, compare it with the committed source before considering any force option. Report
what was copied and remind the user to reload Bob Navigation Hotkeys in Obsidian.

After reload, manually verify:

1. ordinary P2: the card shows emphasized `in N days` plus a separate `YYYY-MM-DD · weekday` receipt;
2. counted P4: the card shows both exact sorted endpoint dates and the relative span;
3. `^prj`: the same exact receipt appears under `scheduled (project)`;
4. light and dark themes plus a narrow/mobile notice: ISO digits remain legible, wrap cleanly, and are not clipped;
5. the toast still appears once and inherits the user's configured notice duration.

## Out of scope

- Changing date-roll ranges, randomness, property order, or any content written to notes/frontmatter.
- Adding relative/exact-date cards to non-priority property notices.
- Listing every counted task's individual rolled date; the truthful min/max span remains the compact summary.
- Changing notice duration, dismissal, or global Obsidian notice styling.
- Reworking the otherwise-approved level pill, priority receipt, accent rail, or outcome chips.
