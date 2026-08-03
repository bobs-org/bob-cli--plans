---
tier: tale
title: Redesign the priority write toast around relative scheduled days
goal:
  Choosing a P2/P3/P4 level in the Ctrl+Shift+P picker shows a structured, styled Obsidian notice that states the level,
  the field that landed in the note, the rolled scheduled date with its weekday, and how many days from today that date
  is — for single-task, counted, and ^prj project writes alike — degrading to a single plain-text notice wherever a DOM
  is unavailable.
proposed_by: bbugyi200.athena.sj
create_time: 2026-08-03 06:40:56
status: wip
---

- **PROMPT:**
  [prompts/202608/priority_toast.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/priority_toast.md)

# Plan: Redesign the priority write toast around relative scheduled days

## Repositories

This tale spans the plugin monorepo plus documentation in the primary repo. Open the non-primary repository through the
`/sase_repo` skill and use the printed path for every read and write:

- `sase repo open bob-plugins -r "<reason>"` — owns `plugins/bob-navigation-hotkeys/main.js`, `styles.css`,
  `manifest.json`, `scripts/test-navigation-hotkeys.cjs`, and `README.md`. **All code in this plan lives here.**
- The primary `bob-cli` repo owns `docs/projects.md`, which documents the priority property.

Run the suite from the `bob-plugins` checkout: `npm test` (shells out to `node --test scripts/*.cjs`) and
`npm run validate` (manifest check). Deploy with `bob plugins sync -p bob-navigation-hotkeys -r "$PWD"`; the `-r "$PWD"`
is required because the default source path does not exist in a SASE workspace.

## Background: the three toasts that exist today

`Ctrl+Shift+P` → `priority` → a level routes through one of three writers, each ending in a hand-built `new Notice(…)`
string:

| Path                  | Writer                                                     | Notice today                                                                              |
| --------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| ordinary inline task  | `setBulletPriorityValue` → `setInlineBulletPropertyValues` | `priority → P2 (high); scheduled → 2026-08-06 · Thu; marked task Blocked`                 |
| `^prj` lifecycle task | `setBulletPriorityValue` → `setProjectNoteScheduledValue`  | `priority → P2 (high); scheduled → 2026-08-06; scheduled 4 tasks; marked 4 tasks Blocked` |
| counted session       | `setCountedBulletPriorityValue`                            | `priority → P2 (high) on 3 tasks; scheduled rolled per task; marked 3 tasks Blocked`      |

The inline writer takes the whole message as an `options.noticeText` string and appends its own `; marked task Blocked`
plus `scheduledRecoveryNoticeSuffix(...)`. The project writer takes an `options.noticePrefix` string (its only caller is
the priority path) and joins its own `parts` array. The counted writer template-literals five suffix fragments together.

## What is wrong with it

1. **The number the user actually wants is missing.** A rolled `2026-09-14` says nothing about urgency; "in 42 days"
   does. This is the headline request.
2. **The counted toast hides its own work.** `scheduled rolled per task` reports that dates were rolled without naming a
   single one. The user just spread N tasks across a window and cannot see the window.
3. **It is a run-on line.** Four to seven semicolon-joined clauses in one weight, one color, no hierarchy. The most
   important fact (the date) and the least important (`2 tasks unchanged`) look identical.
4. **`(high)` competes with `P2`.** The stored-value/label split is the feature's central subtlety, and the toast
   renders both as equal peers instead of showing the level as the answer and the field as the receipt.
5. **No visual identity.** Every other surface this epic built — the value stage, the priority rows, the pinned roll
   item — got `signal-high` chrome, pills, and accent borders. The toast, the one surface the user sees on _every_
   write, got a string.

## Design

### Principles

- **The relative day count is the hero.** It is the only thing on the card rendered in the accent color.
- **The exact date stays verbatim.** `2026-08-06` is the vault's language (every Dataview field is ISO) and is
  copy-pasteable. The relative count sits beside it, never replaces it.
- **Show the receipt.** `[priority:: high]` appears verbatim, muted, so the label→value mapping is never a mystery.
- **One notice per write.** Unchanged from today, and non-negotiable: the writers are one guarded transaction each.
- **Degrade, never throw.** Any environment without Obsidian's DOM sugar gets one plain-text notice with the same facts.
  This mirrors the existing `applyIcon()` guard.
- **Inherit the user's notice timeout.** Never pass an explicit duration to `new Notice(...)`; passing one silently
  overrides an Obsidian setting of "never auto-dismiss".

### The card

Single inline task (base date Mon 2026-08-03, P2 rolls +3):

```
┌────────────────────────────────────────────────┐
│ ▍ ⌁ P2                       [priority:: high] │   header: level icon + pill, muted receipt
│                                                │
│   ⚄ scheduled  2026-08-06 · Thu     in 3 days  │   date row; "in 3 days" is the accent chip
│                                                │
│   ( Blocked )                                  │   status chips, omitted when empty
└────────────────────────────────────────────────┘
```

Counted session — the span replaces the vague "rolled per task":

```
│ ▍ ⌁ P2   3 tasks                     [priority:: high] │
│   ⚄ scheduled  2026-08-05 to 2026-08-10   in 2–7 days  │
│   ( 3 Blocked )  ( 1 unchanged )                       │
```

`^prj` project task — the date row is labelled so it is obvious the date went to frontmatter:

```
│ ▍ ⌁ P2                                  [priority:: high] │
│   ⚄ scheduled (project)  2026-08-05 · Wed     in 2 days   │
│   ( scheduled 4 tasks ) ( removed #hide from 2 tasks ) ( 4 Blocked ) │
```

Elements, all built with `createDiv`/`createSpan`/`appendText` — **never `innerHTML`**, matching the convention stated
at `appendHighlighted()`:

- **Accent rail** — a 3px left border on the card, hued by level index.
- **Level icon** — `getPriorityLevelIconName(levelIndex)`: index 0 → `signal-high`, 1 → `signal-medium`, 2 →
  `signal-low`, 3+ → `signal`. Derived from position in the configured `levels` list (which is ordered most-urgent
  first), never from parsing the label, so a renamed or four-level config still works. Rendered through the existing
  `applyIcon()` so a missing `setIcon` degrades silently.
- **Level pill** — the label (`P2`), hued by level index (see CSS below).
- **Count pill** — muted, counted sessions only (`3 tasks`).
- **Receipt** — `[priority:: high]`, `var(--text-muted)`, `var(--font-ui-smaller)`, monospace, wraps under the pill row
  on narrow notices.
- **Date row** — `dices` icon (matching the pinned roll item in the date picker), muted `scheduled` label, the ISO value
  plus weekday, and the accent relative chip.
- **Status chips** — one per outcome fact, four tones: `is-warn` (Blocked), `is-ok` (recovered), `is-info` (propagated
  schedules, removed `#hide`), `is-muted` (unchanged, clamped session, ambiguous fields). The row is omitted entirely
  when there are no chips.

Accessibility: the card carries `aria-label` set to the plain-text linearization, so the toast reads as one sentence to
a screen reader instead of a pile of spans. Level hue is never the only signal — the label text always says `P2`.

### Vocabulary

Three pure helpers, next to the existing date helpers (`getLocalDateStart`, `addLocalDateDays`, `compareLocalDates`):

```js
function getLocalDayOffset(baseDate, targetDate)   // integer days, DST-safe
function formatRelativeDayOffset(offset)           // "today" | "tomorrow" | "in 3 days" | "yesterday" | "5 days ago"
function formatRelativeDayRange(minOffset, maxOffset)
```

- `getLocalDayOffset` diffs `getLocalDateStart()` of both arguments and `Math.round`s the millisecond delta over
  86_400_000. Rounding (not truncation) is what makes a DST transition inside the span still yield a whole number.
- `formatRelativeDayRange`: equal offsets → `formatRelativeDayOffset(min)`; `min >= 1` → `in ${min}–${max} days` (en
  dash, matching the existing `2–7d` window pill); otherwise →
  `${formatRelativeDayOffset(min)} to ${formatRelativeDayOffset(max)}` (so a window configured with `min_days: 0` reads
  `today to in 7 days` rather than the nonsense `in 0–7 days`).
- Past offsets cannot occur from a roll (`min_days` is validated non-negative) but are handled anyway — the same
  formatter is the natural home for any future scheduled-date notice, and an untested `NaN`/negative path is exactly the
  kind of thing that surfaces as a broken toast.

**The offset is computed from the written date, not from the roll offset.** Parse the `YYYY-MM-DD` string that actually
landed in the note back through the existing `validateProjectScheduledDate()` / `projectScheduleLocalDate()` pair and
diff it against today. That guarantees the number on screen always describes the date on screen, across all three
writers, and it keeps working if a caller ever supplies a date it did not roll. If the value fails to parse, omit the
relative chip rather than rendering `NaN`.

### Model, text, and rendering

A pure model is the single source of truth; the DOM renderer and the plain-text fallback are two views of it. This is
what makes a visual feature testable in a Node test suite with no DOM.

```js
buildPriorityNoticeModel({
  property,          // normalized priority property: { name, schedules, levels, ... }
  level, levelIndex, // the chosen level and its position in property.levels
  baseDate,          // Date the relative math is measured from
  scheduledValues,   // ["2026-08-05", …] — the dates actually written (1 for single/project, N for counted)
  taskCount,         // tasks the priority landed on (1 for the single-task path)
  scope,             // "task" | "counted" | "project"
  outcome,           // { blockedTaskCount, recoveryCounts, propagatedScheduleTaskCount,
                     //   removedHideTaskCount, ambiguousTaskCount, unchangedTaskCount, session }
}) → frozen {
  iconName, levelIndex, pill, countPill, receipt,      // receipt: "[priority:: high]"
  dateLabel,                                            // "scheduled" | "scheduled (project)"
  dateText,                                             // "2026-08-06 · Thu" | "2026-08-05 to 2026-08-10"
  relativeText,                                         // "in 3 days" | "in 2–7 days" | ""
  chips: [{ text, tone }],
  text,                                                 // the plain-text linearization
}
```

`formatPriorityNoticeText(model)` produces `text` by joining the header, date row, and chips with `; `:

- single: `priority → P2 (high); scheduled → 2026-08-06 · Thu · in 3 days; marked task Blocked`
- counted: `priority → P2 (high) on 3 tasks; scheduled → 2026-08-05 to 2026-08-10 · in 2–7 days; marked 3 tasks Blocked`
- project: `priority → P2 (high); scheduled → 2026-08-05 · Wed · in 2 days; scheduled 4 tasks; marked 4 tasks Blocked`

This deliberately keeps the existing `name → value` vocabulary and the existing Blocked/recovery phrasing so the
degraded toast still reads like every other notice in the plugin. When all counted rolls land on the same date, the date
row collapses to the single-date form.

Chip phrasing must be **derived from the existing suffix builders, not retyped**, or the two vocabularies will drift:

- Extract `scheduledRecoveryNoticeParts(counts) → string[]` and redefine `scheduledRecoveryNoticeSuffix(counts)` as
  `parts.length ? "; " + parts.join("; ") : ""`. Every existing caller keeps a byte-identical string.
- Extract `getCountedTaskNoticeParts(session, unchangedTaskCount) → string[]` from `getCountedTaskNoticeSuffix()`, and
  redefine that method in terms of it. Same byte-identical guarantee.
- The model turns each part into a chip.

Rendering seam:

```js
function renderPriorityNoticeFragment(model, root)  // appends the card to a fragment-like root
function showPriorityNotice(model, options = {})    // fragment when possible, plain text otherwise
```

`showPriorityNotice` resolves a root by calling `document.createDocumentFragment()` only when `document` exists **and**
the returned fragment has a `createDiv` function (Obsidian augments the `DocumentFragment` prototype; a bare jsdom-style
DOM does not). It wraps the render in `try`/`catch`. Any failure at any point → `new Notice(model.text)`. Under
`node --test` there is no `document`, so the existing test harness keeps receiving plain strings through its
`String(message)` stub with no harness change — while `renderPriorityNoticeFragment` is unit-tested directly against an
injected fragment-like stub. Accept `options.showNotice` for test injection, mirroring `showBulletPropertyNotice()`.

### Wiring the three writers

The generic writers must not learn what a priority is. Give each one a `buildNotice` callback seam that receives the
outcome facts it alone knows and returns a model:

1. **`setInlineBulletPropertyValues()`** — after the blocked/recovery decision, if
   `typeof options.buildNotice === "function"`, call it with `{ blocked, recoveryOutcome }` and hand the result to
   `showPriorityNotice()`. Otherwise emit today's `options.noticeText` string **byte-identical** — every non-priority
   property (`scheduled`, `dependsOn`, scalar lists) keeps its current notice, and the existing tests for them must pass
   untouched.
2. **`setProjectNoteScheduledValue()`** — same seam, called with the plan facts it already computes for `parts`
   (`plan.scheduled`, `scheduledTaskCount`, `removedHideTaskCount`, `blockedTaskCount`, `ambiguousTaskLines.length`, and
   the five recovery counts). Its `options.noticePrefix` option exists only for the priority caller, so replace it with
   `buildNotice` rather than carrying both.
3. **`setCountedBulletPriorityValue()`** — already owns its notice end to end; build the model from
   `scheduledValueByLine.values()` plus the plan counts and call `showPriorityNotice()` directly.

`setBulletPriorityValue()` supplies the `buildNotice` callbacks for paths 1 and 2, closing over `property`, `level`, the
level index, `baseDate`, and the rolled value. It must pass the same `baseDate` it rolled from, so a write that
straddles midnight still reports a consistent number.

### CSS

Add a `/* --- Priority notice ------- */` section at the end of `plugins/bob-navigation-hotkeys/styles.css`, following
the file's existing conventions exactly: every rule scoped under `.bob-nh-notice` (so nothing leaks into Obsidian's
other notices), Obsidian CSS variables only, and the `color-mix(in srgb, var(--…) N%, transparent)` idiom already used
throughout — **no literal colors**.

- `.bob-nh-notice` — flex column, `gap: 6px`, `padding-left: 10px`, 3px left border in the level hue, `max-width: 100%`,
  `white-space: normal` (some themes set `pre-wrap` on `.notice`). No fixed widths — Obsidian's notice container sizes
  itself and mobile is narrow.
- `.bob-nh-notice-header` — flex row, `align-items: center`, `gap: 6px`, `flex-wrap: wrap`; the receipt gets
  `margin-left: auto` so it right-aligns on a wide notice and wraps to its own line on a narrow one.
- `.bob-nh-notice-level` (pill), `.bob-nh-notice-count` (muted pill), `.bob-nh-notice-chip` — reuse the geometry of the
  existing `.bob-cnp-pill` rule (inline-flex, `var(--radius-s)`, `var(--font-ui-smaller)`), scaled down slightly for a
  toast.
- Level hue: set a `--bob-nh-level-color` custom property on the card via `.bob-nh-notice.is-level-0/1/2` (index clamped
  to 2; 3+ falls through to the base rule), mapping to `var(--color-orange)`, `var(--color-yellow)`,
  `var(--color-blue)`, with `var(--text-accent)` as the default. Because saturated yellow is unreadable as text on a
  light background, pill **text** color must be `color-mix(in srgb, var(--bob-nh-level-color) 80%, var(--text-normal))`
  — the same mix pattern `plugins/block-id-prompt/styles.css` already uses for its orange text.
- `.bob-nh-notice-relative` — the accent chip: `var(--text-accent)`, `var(--font-semibold)`. This is the only element
  that gets emphasis, which is what makes the day count read first.
- Chip tones: `.is-warn` → `--color-orange`, `.is-ok` → `--color-green`, `.is-info` → `--text-accent`, `.is-muted` →
  `--text-muted` with `--background-modifier-border`, each following the border/background mix pattern of
  `.bob-cnp-pill`.
- No animation, no transition — a toast that is already animating in should not animate its contents.

## Implementation order

1. Pure helpers: `getLocalDayOffset`, `formatRelativeDayOffset`, `formatRelativeDayRange`, `getPriorityLevelIconName`.
   Export from `module.exports.helpers`.
2. Extract `scheduledRecoveryNoticeParts()` and `getCountedTaskNoticeParts()`; redefine the two existing suffix builders
   in terms of them. Run `npm test` here — every existing notice assertion must still pass, unchanged. Do not proceed
   until it is green.
3. `buildPriorityNoticeModel()` + `formatPriorityNoticeText()`. Export both.
4. `renderPriorityNoticeFragment()` + `showPriorityNotice()`. Export the renderer.
5. Wire the three writers through `buildNotice`; delete `noticePrefix`.
6. `styles.css`.
7. Update the priority notice assertions in `scripts/test-navigation-hotkeys.cjs` and add the new tests below.

## Tests

All in `scripts/test-navigation-hotkeys.cjs`, following the file's existing style (`node:test` + `node:assert/strict`,
`helpers.*` for pure functions, `createBulletPropertyPickerHarness()` / `createPriorityPickerConfig()` /
`choosePriorityLevel()` for end-to-end paths, `notices.length = 0` before an assertion on notices).

Pure:

- `getLocalDayOffset` — 0 for the same day, 1 for tomorrow, 42 across a month boundary, and a whole number across a DST
  transition (e.g. base 2026-03-07 → target 2026-03-14 in a DST-observing local zone yields exactly 7).
- `formatRelativeDayOffset` — `today`, `tomorrow`, `in 2 days`, `in 42 days`, `yesterday`, `3 days ago`.
- `formatRelativeDayRange` — equal offsets collapse to the single form; `2,7` → `in 2–7 days`; `0,7` →
  `today to in 7 days`.
- `getPriorityLevelIconName` — 0/1/2 map to `signal-high`/`signal-medium`/`signal-low`; 3 and 9 fall back to `signal`.
- `buildPriorityNoticeModel` — the three scopes produce the documented `pill`, `countPill`, `receipt`, `dateLabel`,
  `dateText`, `relativeText`, `chips`, and `text`; counted rolls that all land on one date collapse to the single-date
  form; an unparseable scheduled value yields `relativeText: ""` and no crash; chip tones are assigned as specified.
- `renderPriorityNoticeFragment` — against a minimal fragment stub implementing `createDiv`/`createSpan`/`appendText`
  and recording a tree: asserts the card class list includes the level class, the accent chip node carries the relative
  text, the chips row is absent when there are no chips, and `aria-label` equals `model.text`.
- `showPriorityNotice` — with no `document` (the default under `node --test`) it emits exactly one notice whose message
  is `model.text`; with a stub root that throws mid-render it still emits exactly one plain-text notice.

End to end (update the existing assertions, do not duplicate them):

- Single inline task, base 2026-08-03, `random: () => 0`, level P2 → exactly one notice equal to
  `priority → P2 (high); scheduled → 2026-08-05 · Wed · in 2 days; marked task Blocked`.
- Counted session, rolls `[0, 0.5, 0.999999]` → one notice naming the span `2026-08-05 to 2026-08-10` and `in 2–7 days`,
  replacing the old `scheduled rolled per task` wording, with the three-task Blocked chip intact.
- `^prj` project task → one notice whose date row carries `scheduled (project)` and the relative count, and whose
  frontmatter/inline write assertions are unchanged.
- Regression: the plain `scheduled`, `dependsOn`, scalar-value, and `Ctrl+D` delete notices are **unchanged** — those
  assertions must pass without edits. Selecting the pinned priority-roll row in the _date_ picker must still produce the
  identical notice a preset date produces (the existing test that compares the two is the guard).

`npm test` must be fully green (268 tests today, plus the additions), and `npm run validate` 6/6.

## Docs, version bump, and deploy

- `plugins/bob-navigation-hotkeys/manifest.json`: bump `version` to `1.15.0` (user-visible feature change).
- `bob-plugins/README.md`: bump the Bob Navigation Hotkeys row's version and mention that priority writes report the
  rolled date's distance from today.
- `bob-cli` `docs/projects.md`: in the priority section (around the P2/P3/P4 table), document what the toast reports —
  level, the `[priority:: …]` field that landed, the rolled date with weekday, the relative day count, the counted span,
  and the status chips.
- `git diff --check` clean; commit in both repos through the normal SASE commit flow.
- Deploy: from the `bob-plugins` checkout, `bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run`, then the
  same without `--dry-run`. If sync reports the vault file dirty, verify the on-disk vault file against the committed
  baseline before considering `--force`.
- Tell the user to reload the plugin in Obsidian — the vault copy of `main.js` and `styles.css` is only picked up on
  reload — and hand them the manual check below.

## Manual verification (for the user, after reload)

The card cannot be verified from a headless agent; the automated tests cover structure and text, not pixels. Ask the
user to confirm:

1. `Ctrl+Shift+P` → `priority` → `P2` on an ordinary task: the toast shows `P2`, `[priority:: high]`, the date with its
   weekday, a highlighted `in N days`, and a `Blocked` chip.
2. The same in the other theme (light and dark) — the level pill and the accent chip stay legible.
3. `3<Ctrl+Shift+P>` → `priority` → `P4`: the toast shows `3 tasks` and a span like `in 34–78 days`.
4. On a `^prj` task: the date row says `scheduled (project)`.

## Deliberately out of scope

- **Every non-priority notice keeps its current plain string.** The `scheduled`, `dependsOn`, scalar, and delete notices
  are untouched; only the shared parts extraction refactors them, and byte-identically. Adopting the card for the plain
  `scheduled` write (which would also gain "in N days") is an obvious follow-up, but it is not what was asked, and doing
  it here would put a rendering change under a dozen unrelated assertions.
- **The pinned priority-roll row in the date picker.** It writes only `scheduled`, and an existing test pins its notice
  to be identical to a preset date's. Adding the relative count to its `detail` line would be a nice consistency touch;
  it is separable and is left out to keep this change reviewable.
- **What gets written to the file.** No change to the rolled dates, the `[priority:: <tasks-native-value>]` mapping, or
  the Blocked/recovery behavior. The Tasks trailing-field constraint documented in the parent epic still governs.
- **Notice duration and dismissal.** The plugin will keep inheriting the user's configured notice timeout.
