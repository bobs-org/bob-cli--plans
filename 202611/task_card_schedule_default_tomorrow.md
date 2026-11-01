---
tier: tale
title: Task Card Schedule stage drops the Today preset and defaults to Tomorrow
goal:
  On the Task Card, Enter then Enter picks Tomorrow for scheduled; no Today preset row
  is offered.
size: small
decisions:
  priority_roll_default:
    ask:
      On a task with a priority, should Tomorrow be preselected instead of the pinned
      priority roll?
    choices:
      roll:
        Priority tasks keep the pinned roll preselected; only no-priority tasks default
        to Tomorrow
      tomorrow: Roll row stays pinned on top, but Tomorrow is preselected on every task
    default: roll
    why:
      The ask targets Today; the roll is a deliberate pick and Ctrl+Enter / Alt+P
      already take it
    answer: roll
proposed_by: bbugyi200.apollo.bu
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 10:29:06
status: wip
---

# Task Card Schedule stage: drop the Today preset, default to Tomorrow

## Goal

On the Obsidian Task Card (`Ctrl+Shift+P`), `Enter` opens the Schedule stage (Schedule
is the card's default action). That stage lists fixed date presets and preselects row 0.
For a task with no priority, row 0 is **Today**, so `Enter Enter` heads toward
scheduling the task for today. Bryan almost never schedules a task for today.

After this change:

- The `scheduled` stage has **no "Today" preset row**.
- **Tomorrow** is the first preset row and is preselected. For a task with no priority,
  `Enter Enter` picks Tomorrow.
- Today is still reachable on purpose: type `0`, or type today's ISO date. The typed
  resolver already supports both (`0` means today), so no resolver change is needed.

Nothing changes after the date is picked. A plain `Enter` on a preset still goes through
the usual Schedule Log reason / Work summary review, so a third `Enter` (blank reason)
commits. `Shift+Enter` on the date row still skips the reason under the blank-reason
rule. Leave that flow alone.

## Where the code lives

All work is in the linked **`bob-plugins`** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and use only the path it prints. Read that
repo's `AGENTS.md` first. `bob-navigation-hotkeys` uses the committed fragment build:
edit `plugins/bob-navigation-hotkeys/src/*.js`, run `npm run build`, and never hand-edit
`main.js`. Each hand-edited fragment must stay at or below 1000 lines;
`src/300-dependency-stage.js` is at about 964, so keep that edit small.

Relevant code (paths relative to `plugins/bob-navigation-hotkeys/`):

- `src/270-counted-batches.js`: `createBulletPropertyDateItems(baseDate, currentValue)`
  builds the fixed preset list: Today, Tomorrow, In 2 days, In 3 days, This Saturday,
  This Sunday, Next Monday, In 1 week, In 2 weeks, In 1 month.
- `src/300-dependency-stage.js`:
  `createBulletPropertyValueItems(propertyItem, baseDate)` calls it for every
  `values: "date"` property. Its other caller, `commitTaskCardPriority` in
  `src/350-picker-task-card.js`, only looks up priority values, so it is unaffected.
- `src/360-picker-schedule-review.js`: `showValueStage` sets `this.selectedIndex = 0`,
  builds the items, and for a task with a priority `unshift`s a pinned priority-roll row
  (unless a counted batch suppresses it). It then calls `applyOptions` and
  `renderAll({ clearQuery: true })`. `renderAll` does not reset `selectedIndex`; the
  input listener resets it to 0 when the query changes.
- `src/700-exports.js` exports both helpers for the test harnesses.

`~/.config/bob/config.yml` currently defines only one date property, `scheduled`. The
More menu can still open other configured date properties, and a hypothetical `due` date
for today is reasonable, so scope the removal to `scheduled` only.

## Changes

1. **`createBulletPropertyDateItems` option.** Add an optional third parameter,
   `options = {}`. When `options.includeToday === false`, leave the `["Today", today]`
   row out. Everything else stays the same (labels, order, `current` marking, weekday
   detail). The default (no options) still returns Today, so other callers and the
   exported helper keep their behavior.

2. **Scope to `scheduled`.** In `createBulletPropertyValueItems`, for
   `property.values === "date"`, pass
   `{ includeToday: normalizeBulletPropertyName(property.name) !== "scheduled" }`.
   `normalizeBulletPropertyName` is a hoisted function declaration in
   `src/020-config-load-and-tasks.js`, and fragments share one script scope. This one
   call site covers single-task cards, counted batches, Task Link sessions, and `^prj`
   inline schedule targets, because all of them build their rows here.

3. **Preselection.** For a task with no priority, and for a counted batch whose pinned
   roll is suppressed, nothing more is needed: Tomorrow is now row 0 and `selectedIndex`
   is already 0.

   > [!decision] priority_roll_default = roll No further change. The pinned
   > priority-roll row stays at index 0 and stays preselected; Tomorrow sits directly
   > below it.

   > [!decision] priority_roll_default = tomorrow In `showValueStage`, when
   > `isScheduledProperty` is true and a pinned roll row was `unshift`ed, set
   > `this.selectedIndex` to the index of the first non-`priorityRoll` row whose `value`
   > equals tomorrow's ISO date (that is, the Tomorrow preset, index 1). Do this after
   > the items are built and before `renderAll`. Keep the roll row pinned at the top so
   > `Ctrl+P`/`↑` reaches it; `Ctrl+Enter` and `Alt+P` still apply the recommendation.
   > Typing still resets the selection to 0 through the existing input listener. Update
   > the nearby comment that says Enter on the pinned row writes the same date as
   > Ctrl+Enter, so it no longer implies the pinned row is the Enter default.

4. **Build.** Run `npm run build` in the `bob-plugins` repo to regenerate
   `plugins/bob-navigation-hotkeys/main.js`.

## Tests

Add or extend tests in the `bob-plugins` repo, using the existing harness patterns in
`scripts/test-navigation-task-card-schedule.cjs` (`openScheduleStage`,
`BASE_DATE = 2026-10-03`, `typeQuery`, `pressInputKey`). Note that
`openPropertyPicker`'s default content has `[priority:: medium]`. Pass
`content: "- [ ] #task Ready work ^ready"` for the no-priority cases.

- **No Today row:** on a task with no priority, the Schedule stage's `visibleItems`
  contain no item labelled `Today`. Row 0 is `Tomorrow` with value `2026-10-04`, and
  `selectedIndex === 0`.
- **Enter on the default goes to Tomorrow:** from the opened Schedule stage on a Ready
  task with no priority, plain `Enter` with an empty query moves to the
  `schedule-review` stage carrying `2026-10-04`. `Shift+Enter` instead commits
  `2026-10-04` with a blank reason. Stub `plugin.setBulletPropertyValue` the way the
  neighbouring tests do.
- **Today stays reachable by typing:** `typeQuery(modal, "0")` yields a valid typed row
  for `2026-10-03`.
- **Priority task, matching the chosen decision:** on the default `[priority:: medium]`
  content, row 0 is the `priorityRoll` row. Under `roll`, `selectedIndex === 0`. Under
  `tomorrow`, the selected item is the Tomorrow preset (`2026-10-04`). In both cases
  there is still no `Today` row.
- **Helper scope:** in `scripts/test-navigation-hotkeys-priority-notice.cjs` (it already
  exercises `helpers.createBulletPropertyValueItems`), or in the schedule test file,
  assert three things: `helpers.createBulletPropertyDateItems(base, "")` still starts
  with `Today`; `createBulletPropertyDateItems(base, "", { includeToday: false })`
  starts with `Tomorrow`; and `createBulletPropertyValueItems` returns no Today row for
  a `{ name: "scheduled", values: "date" }` property but keeps it for a non-scheduled
  date property such as `{ name: "due", values: "date" }`.

Then run the full suite: `npm test` in the `bob-plugins` repo. Its first step is
`build:check`. Some existing tests may silently depend on row 0 being Today in a
no-priority date stage, such as counted-batch, link-picker, schedule-log, or
review-advance tests. If one fails for that reason only, update it to select its
intended row by label or value. Do not restore the Today row.

## Docs

Update `docs/task-card.md` in the `bob-plugins` repo:

- In **Scheduling input**, say that the Schedule stage's presets start at Tomorrow,
  which is preselected, and that there is no Today preset. Today is still reachable by
  typing `0` or its ISO date. Keep the existing "Other date properties opened from
  More…" sentence accurate: those properties keep the Today preset.
- In the **Card gestures** row for `Enter`, add that the Schedule stage preselects
  Tomorrow, or, under the `roll` decision, the pinned priority roll when the task has a
  priority. That makes `Enter Enter` choose Tomorrow before the usual reason review.

Run `npm run docs:check` (plus `npm run docs:inventory:check` if the inventory
complains).

## Deploy

After tests and docs pass, run `bob plugins sync` so the vault picks up the rebuilt
plugin, as `bob-plugins/AGENTS.md` requires. Then commit the `bob-plugins` changes
through the normal SASE final flow. There are no bob-cli source changes and no memory
changes.

## Out of scope

- The Schedule Log reason / Work summary review after a date is picked (unchanged;
  Shift+Enter remains the skip-reason path).
- `Ctrl+Enter` / `Alt+P` recommended-roll behavior and priority-level keys `1`–`4`.
- Other plugins' date surfaces and the typed-date resolver (`0` keeps meaning today).
