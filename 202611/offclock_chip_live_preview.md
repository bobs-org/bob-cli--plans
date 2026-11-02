---
tier: tale
title: Render
goal:
  Each off-clock Event Task tag renders as a single dashed timer-off chip in Live
  Preview instead of two split pills.
size: small
decisions:
  reveal:
    ask:
      Should the raw `#offclock` text come back whenever the cursor is anywhere on the
      Event Task line?
    choices:
      line:
        Raw tag returns while the cursor is on the line, as documented today and like
        `[buffer::]`
      tag:
        Raw tag returns only when the selection touches the tag, like the `#task` hash
        mark
    default: line
    why: Keeps today's documented behavior and matches the buffer field on the same row
    answer: line
proposed_by: bbugyi200.athena.16z
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 07:55:55
status: wip
---

# Fix the doubled `#offclock` chip in Live Preview

All changes are in the **`bob-plugins`** linked repo (open it with `/sase_repo`; read
its `AGENTS.md` first). bob-cli itself does not change.

## Problem

In Live Preview, every off-clock Event Task in a daily note's Events section (for
example `- [*] #task **0800-0815** Swings! [📅](…) #offclock [root:: love] …`) renders
its tag as **two** dashed pills, each with its own `timer-off` icon: `⦸ #` and
`⦸ offclock`. Both sit inside Obsidian's grey tag pill. On inactive lines the `#` is
hidden, but its pill and icon stay, so the line shows an empty `⦸` pill next to
`⦸ offclock`.

`docs/bob-ledger-tools.md` (the "An Event Task carrying a standalone `#offclock` tag"
paragraph) describes the intended result: **one** quiet mark (a muted `timer-off` icon,
a dashed outline, tooltip "Off-clock · on the calendar, off the clock"), with the `#`
folded away unless the cursor is on the line. Reading view already renders it that way
because `calendarEventPaintOffClockTag` restyles a single `a.tag`.

## Root cause

`plugins/bob-ledger-tools/src/291-plugin-gcal-events.js` →
`buildCalendarEventDecorations` paints the tag with
`Decoration.mark({ class: "bob-calendar-event-off-clock" })` across `#offclock`, plus a
second mark (`bob-calendar-event-off-clock-hash`) on the `#`. The extension is
registered with `Prec.highest(ViewPlugin…)`.

CodeMirror 6 draws a higher-precedence mark as the **inner** DOM node and splits it at
the edges of lower-precedence marks. Obsidian's highlighter tokenizes a tag as two
spans, `cm-hashtag-begin` (`#`) and `cm-hashtag-end` (`offclock`). Because our mark has
the highest precedence, it is split into two `span.bob-calendar-event-off-clock`
elements, one inside each Obsidian span. In `styles.css`, each fragment then gets its
own `::before` icon and dashed outline. Obsidian's grey `.cm-hashtag` background stays
on the outer spans: the `.bob-calendar-event-off-clock.cm-hashtag` /
`.bob-calendar-event-off-clock .cm-hashtag` selectors assume the opposite nesting, so
they never match.

Lowering precedence so the mark wraps Obsidian's spans would depend on how Obsidian
orders its highlighter internally, and that can't be tested here. Repo precedent already
avoids this: `264-plugin-task-tag-marks.js` swaps `#task` for a widget
(`Decoration.replace`) and reveals the raw text when the selection touches it. Use the
same pattern.

## Changes

### 1. Off-clock chip widget (new code in `291b-plugin-gcal-event-buffers.js`)

`291-plugin-gcal-events.js` has 896 lines, and hand-edited fragments must stay ≤ 1000.
Put the new code in `291b` with the other Event Task Live Preview helpers. If `291b`
would pass 1000 lines, add a new fragment such as `291c-plugin-gcal-off-clock-chip.js`
and list it in `src/fragments.json` right after `291b`.

- Add a pure DOM builder `buildCalendarEventOffClockChip(doc, text)` that returns
  `<span class="bob-calendar-event-off-clock" data-off-clock="true" title="Off-clock · on the calendar, off the clock">offclock</span>`.
  Use `GCAL_EVENTS_OFF_CLOCK_TITLE`. The text is the tag's own spelling without the `#`,
  so `#Untracked` shows `Untracked` and `#off-clock` shows `off-clock`, the same as
  Reading view.
- Add a `WidgetType` subclass (guarded like `TaskTagMarkWidget` in
  `138-task-tag-marks.js`: defined only when `WidgetType` exists, otherwise `null`):
  - `eq` compares the tag text, so unchanged chips never flicker.
  - `toDOM` builds the chip and adds the same reveal-on-mousedown behavior as
    `addTaskTagMarkReveal`: `preventDefault`, then dispatch a selection at
    `view.posAtDOM(dom)`, then `focus`. Reuse that helper if it is in scope (fragments
    share one script scope); do not copy it.

### 2. Decoration builder (`buildCalendarEventDecorations` in `291-…`)

- Delete both `Decoration.mark` pushes for `row.offClockTag` (the off-clock mark and the
  `-off-clock-hash` mark).
- Replace them with one `Decoration.replace({ widget })` over
  `[base + start, base + end)` when the tag is not revealed. When it is revealed, emit
  nothing so Obsidian's native tag renders and stays editable. When the tag counts as
  revealed:

> [!decision] reveal = line The tag is revealed when any selection range touches the
> tag's line (`from ≤ lineEnd && to ≥ lineStart`).

> [!decision] reveal = tag The tag is revealed when any selection range touches
> `[absFrom, absTo]`, the same check as `buildTaskTagMarkDecorations`.

- Keep the existing guard: if `WidgetType` / the widget class is unavailable, emit no
  off-clock decoration at all. That is safer than a mark that splits.
- The builder already sorts decorations by `from` then `to`. The replace range must not
  overlap any other decoration on the row. Today none do (`row.tags` only covers
  `*maybe*` / `*free*` / `*day N of M*`, and badges are emoji), but a regression test
  should pin this (see Tests).

### 3. Rebuild on selection changes (`createCalendarEventExtension`)

The view plugin rebuilds only on `docChanged`, `viewportChanged`, or the refresh effect.
Now it must also rebuild on `update.selectionSet`. Don't rescan the whole note on every
cursor move: cache the last `gcalEventsScanMarkdown` result together with the inputs it
was built from (`view.state.doc` identity, `isToday`, `nowMinutes`), and reuse it when
only the selection changed. Two acceptable approaches: split
`buildCalendarEventDecorations` into scan + emit steps, or pass an optional cached scan
into it. Either way, keep the function's `Decoration.none` fallbacks and its
never-throws contract.

### 4. CSS (`plugins/bob-ledger-tools/styles.css`)

- Remove the `.bob-calendar-event-off-clock.cm-hashtag` and
  `.bob-calendar-event-off-clock .cm-hashtag` selectors from the chip rule. Only the
  widget span (Live Preview) and `a.tag.bob-calendar-event-off-clock` (Reading view)
  carry the class now.
- Remove the `.cm-line:not(.cm-active) .bob-calendar-event-off-clock-hash` selector. No
  Live Preview hash span remains. Keep the
  `a.bob-calendar-event-off-clock .bob-calendar-event-off-clock-hash { display: none; }`
  rule for Reading view.
- Make sure the Reading-view `a.tag` chip still beats the theme's `.tag` background,
  padding, and border. If specificity is too low, add an
  `a.tag.bob-calendar-event-off-clock` selector to the chip rule. Check the second
  `.bob-calendar-event-off-clock` rule near the end of the file (inside an `@media`
  block) for the same stale selectors.
- Update the comment above the `[buffer::]` rule. It says "like the `#` of `#offclock`",
  and in Live Preview the `#` now goes away with the chip widget.

### 5. Docs (`docs/bob-ledger-tools.md`)

Update the off-clock paragraph to describe the new Live Preview behavior: the tag
becomes a single chip that shows the tag name without `#`, and the raw `#offclock`
returns while the cursor is on the line (or on the tag, if `reveal = tag`). Run
`npm run docs:check`. If the manifest description or generated inventory text changes,
regenerate it with the repo's docs tooling.

## Tests

Add to `scripts/test-ledger-tools-meet-event-run.cjs`, or to
`scripts/test-ledger-tools-gcal-events.cjs` if its harness already stubs `Decoration`.
Reuse the stub-`Decoration` / `makeView({ lines, selection })` pattern from
`scripts/test-ledger-tools-task-tag-marks.cjs`:

1. An off-clock Event Task line in a canonical daily note's Events section, with no
   selection on that line, yields exactly **one** replace decoration over the
   `#offclock` range whose widget renders the text `offclock`. No
   `bob-calendar-event-off-clock` or `-off-clock-hash` mark decorations appear.
2. Alias and case are preserved: `#Untracked` → widget text `Untracked`; `#off-clock` →
   `off-clock`.
3. Reveal: with the cursor on the line (per the `reveal` decision), no off-clock
   decoration is emitted. The time-pill mark is still emitted with
   `data-off-clock="true"`.
4. Line with both a buffer and `#offclock` (e.g.
   `… Lunch #offclock [buffer:: -30m +15m] …`): decorations are added in sorted order
   without throwing, and the replace range does not overlap the time-pill or buffer
   decorations.
5. A selection-only update reuses the cached scan: spy or count `gcalEventsScanMarkdown`
   calls, or assert that the cache object is identity-equal.
6. The widget DOM builder (pure, with a fake `doc`) sets the class, `data-off-clock`,
   and the title.
7. Keep the existing Reading-view `calendarEventPaintOffClockTag` test unchanged. It
   must still pass.

## Build, verify, deploy

1. `npm run build`. Never hand-edit the generated `plugins/bob-ledger-tools/main.js`.
2. `npm test` (it includes `build:check`) and `npm run docs:check`. Both must pass.
3. `bob plugins sync` to deploy to the vault (required by `bob-plugins` `AGENTS.md`).
4. Manual check in Obsidian on today's daily note. Each `#offclock` Event Task should
   show one muted dashed chip (`⦸ offclock`) with no grey Obsidian tag pill around it.
   With the cursor on the line, it should show the plain `#offclock` tag. Reading view
   should be unchanged.
