---
tier: epic
title: 'Priority roll decay: Ctrl+Enter takes the recommended roll'
goal: 'In the Ctrl+Shift+P picker, Ctrl+Enter on `scheduled` takes the recommended
  roll in one keypress, and the date it will write is shown on the `scheduled` row.
  The roll follows a configurable decay ladder read from the task''s Schedule Log.
  A level is re-rolled `rolls` times (default 1), the next recommended roll moves
  the task one level down (P2 → P3), and past the last level it cancels the task.
  This works for single tasks, `^prj` tasks, counted sessions and Task Link sessions,
  and every write is a guarded, logged, one-undo edit.

  '
phases:
- id: decay-core
  title: 'Navigation Hotkeys: decay config, Schedule Log roll streak, and pure recommendation
    planner'
  depends_on: []
  size: medium
  description: 'decay-core: add the `decay`/`rolls` config grammar, the roll-streak
    reader with its entry classification, the date roll that avoids the current date,
    the recommendation planner, the decay and cancel reason formatters, and the preview
    model. Includes a new conformance test file. No UI changes.

    '
- id: picker-single
  title: 'Navigation Hotkeys: Ctrl+Enter recommended roll for single and ^prj tasks'
  depends_on:
  - decay-core
  size: medium
  description: 'picker-single: add the roll preview line on the `scheduled` row, Ctrl+Enter
    in both picker stages, and Ctrl+R re-roll in stage one. The value stage''s pinned
    row shares the previewed date. Writes are guarded and re-verified for roll, decay
    and cancel, with decay-aware notice cards, CSS, a manifest bump and a vault deploy.

    '
- id: picker-counted
  title: 'Navigation Hotkeys: recommended roll for counted N<Ctrl+Shift+P> sessions'
  depends_on:
  - picker-single
  size: medium
  description: 'picker-counted: plan a recommendation for each target, show a batch
    preview line, and compose the cancel and set-priority plans into one undoable
    editor transaction. Also adds the priorityValueByLine and reasonByLine planner
    extensions, the batch notice card, and a manifest bump.

    '
- id: picker-links
  title: 'Navigation Hotkeys: recommended roll for Task Link sessions'
  depends_on:
  - picker-counted
  size: medium
  description: 'picker-links: read each linked task''s streak from its own note and
    reuse the composed batch planner for each note group. Commits go through commitLinkPickerNoteWrites
    with one shared Pomodoro prune and dependent recovery for cancelled tasks. Also
    bumps the manifest.

    '
- id: decay-docs
  title: bob-cli docs, config guard test, and chezmoi config for roll decay
  depends_on:
  - picker-links
  size: small
  description: 'decay-docs: add the bob-cli docs/projects.md section, the reason-table
    rows, and notes in randomize.md and freshness.md. Also adds a Rust test that keeps
    `decay`/`rolls` keys parseable, and a commented `decay` block in the chezmoi-managed
    config.'
proposed_by: bbugyi200.athena.0um
create_time: 2026-09-30 23:56:47
status: done
bead_id: bob-cli-34
---

- **PROMPT:** [prompts/202609/priority_roll_decay.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/priority_roll_decay.md)
- **BEAD:** [bob-cli-34](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-34/README.md)

# Plan: Priority roll decay — Ctrl+Enter takes the recommended roll

## Context

- **Which keymap.** `Ctrl+Shift+P` is `bob-navigation-hotkeys:set-bullet-property`. It
  is implemented by `BulletPropertyPickerModal` in
  `plugins/bob-navigation-hotkeys/main.js` in the linked **`bob-plugins`** repo. Open
  that repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`), read its
  `AGENTS.md`, and use the printed path for every read and write. Phases 1–4 change only
  that repo. Phase 5 changes bob-cli (this repo) and the linked **`chezmoi`** repo, also
  opened through `/sase_repo`.
- **How the picker works today.**
  - **Stage one** (`showPropertyStage`) lists the pinned lane row, the refresh row, the
    property rows (`renderPropertyItem`) and the pinned Cancel row.
  - **↵ on `scheduled`** opens **stage two** (`showValueStage`). When the task has a
    configured priority, `getPriorityRollLevel` + `createPriorityRollDateItem` pin a
    `🎲 P2 roll` row above the date presets. `Ctrl+R` (`rerollPriorityDateSuggestion`)
    re-rolls it. Choosing it writes immediately through
    `buildPriorityRollScheduleLogForItem` with the reason
    `🎲 P2 roll · in **17** (8–30) days`.
  - **Today it takes ↵, ↵ (plus navigation) to reach the recommended roll.**
    `FilteredPickerModal.handleKeydown` treats any `Enter` (including Ctrl+Enter) as ↵.
- **Building blocks to reuse.**
  - Priority config: `normalizeBulletPriorityProperty`, `normalizeBulletPriorityLevel`,
    `validateBulletPropertyConfig`, read from `~/.config/bob/config.yml`.
  - Rolling and reasons: `rollPriorityScheduledDateWithOffset`,
    `formatPriorityRollScheduleReason`, `buildPriorityRollScheduleLog`.
  - Schedule Log grammar: `findScheduleLogParent`, `parseScheduleLogEntryBullet`,
    `SCHEDULE_LOG_ENTRY_RE`, `getScheduleLogEntryIndent`, `planScheduleLogEntry`.
    Entries are direct children of the marker, newest first.
  - Writers: `setInlineBulletPropertyValues` / `setBulletPropertyValue`,
    `setProjectNoteScheduledValue` and `setBulletPriorityValue`. All accept
    `scheduleLog` and `buildNotice`. Counted writers are
    `planCountedBulletPropertyBatch` (operation `set-priority`) and
    `setCountedBulletPriorityValue`; the link writer is `applyLinkPickerPriorityValue`.
  - Cancel: `applyTaskCancelFromPicker`, `planTaskCancelBatch`, `planCancelLogEntry`,
    `buildCancelNoticeModel`.
  - Notices: `buildPriorityNoticeModel`, `renderPriorityNoticeFragment`,
    `showPriorityNotice`.
- **Config consumers.** bob-cli's `src/native/config/mod.rs` (`RawProperty`, `RawLevel`)
  deserializes with serde and no `deny_unknown_fields`, so new `decay` and `rolls` keys
  are ignored by `bob capture p:<N>` and `bob randomize`. bob-ledger-tools does not read
  `properties`.
- **Rules this design respects.** Read them with
  `sase memory read decisions:<key> -r "<why>"`.
  - `decisions:task-status-is-derived`: a roll to a future date marks the task Blocked
    through the existing writers, exactly as the pinned roll does today.
  - `decisions:task-lanes-are-sticky`: this is not age-based lane decay, which that
    record rejects. Priority decay never happens on its own. It happens only when the
    user takes the recommended roll.

## Design contract (all phases implement exactly this)

### Vocabulary

| Term                 | Meaning                                                                                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Recommended roll** | What `Ctrl+Enter` on `scheduled` does: a same-level roll, a decay, or a cancel                                                                       |
| **Roll streak**      | The number of `🎲 <level> roll` Schedule Log entries at the task's current level, counted from the newest entry until the first entry that breaks it |
| **Roll limit**       | How many same-level rolls a level allows before the next recommended roll decays: `levels[].rolls`, else `decay.rolls`, else `1`                     |
| **Decay**            | Moving the task one level down the ladder (P2 → P3) and rolling a fresh date in the new level's window                                               |
| **Ladder**           | P1 → P2 → P3 → P4 → cancelled; the configured level order, then cancellation                                                                         |

### The gesture: ⌃↵ follows the ladder, ↵ stays explicit

1. **`Ctrl+Enter` with the `scheduled` row selected in stage one takes the recommended
   roll** and closes the picker. `Cmd+Enter` is accepted too, for macOS. Alt and Shift
   variants are not.
2. **`Ctrl+Enter` anywhere in stage two** (the `scheduled` date list) takes the same
   recommended roll, whichever row is highlighted.
3. **`↵` never decays or cancels.** ↵ on `scheduled` still opens the date list, whose
   pinned row is still the same-level `🎲 P2 roll`. That pinned row is the explicit
   "keep this level" choice: it writes the same `🎲 P2 roll` entry and therefore counts
   toward the streak.
4. **Fallback.** `Ctrl+Enter` on any other stage-one row, or on `scheduled` when there
   is no recommendation, behaves exactly like ↵, as it does today. When a recommendation
   exists but is unavailable (the recurring case below), `Ctrl+Enter` shows the Notice,
   writes nothing and keeps the picker open.
5. **`Ctrl+R` in stage one** re-rolls the recommendation's date when it has one. That is
   any recommendation except a cancel; nothing else uses Ctrl+R in stage one. In stage
   two, Ctrl+R re-rolls the pinned row as today and keeps the shared date in sync (see
   Look and feel).
6. No new command and no new default hotkey are added.

**When a recommendation exists.** All of these must hold:

- The picker targets an open Obsidian task (`" "`, `*`, `/` or `?`).
- A configured `values: priority` property names this date property in `schedules`.
- The task's priority value is one of that property's configured levels.

There is no recommendation for implicit P0 (no priority field), an unconfigured value,
plain bullets, or closed tasks. Before phases `picker-counted` and `picker-links` land,
counted and link sessions have no recommendation either.

### The ladder

Let `level` be the current level, `streak` its roll streak and `limit` its roll limit.

| Condition                                 | Recommendation                                     | Writes                                                                                                                       |
| ----------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| decay disabled (`decay: false`)           | **roll** at `level`                                | date in `level`'s window, `🎲 P2 roll …`                                                                                     |
| `streak < limit`                          | **roll** at `level` (step `streak + 1` of `limit`) | date in `level`'s window, `🎲 P2 roll …`                                                                                     |
| `streak >= limit` and a next level exists | **decay** to the next level                        | priority field → next level's value, date in the next window, `🎲 P2 → P3 decay …`                                           |
| `streak >= limit` at the last level       | **cancel**                                         | `[-]`, `[cancelled:: today]`, Cancel Log `🍂 decayed past P4 after 1 roll`                                                   |
| cancel recommended on a recurring task    | **unavailable**                                    | nothing; Notice `Recurring tasks are cancelled with Obsidian Tasks so the next occurrence is handled; no tasks were updated` |

A decay **does not** count as the first roll at the new level. The user asked that a
task's first roll at a level recommends that same level. So with the default `rolls: 1`,
every level grants two windows: entering it, then one roll.

Lifecycle with the default config (the ladder is P1–P4 with windows 2–7, 8–30, 31–90 and
91–365):

| #   | Gesture           | Entry written                                            | Next ⌃↵ recommends |
| --- | ----------------- | -------------------------------------------------------- | ------------------ |
| 0   | Priority row → P2 | `🎲 P0 → P2 · in **12** (8–30) days`                     | P2 roll (1/1)      |
| 1   | ⌃↵                | `🎲 P2 roll · in **20** (8–30) days`                     | P2 → P3            |
| 2   | ⌃↵                | `🎲 P2 → P3 decay · in **45** (31–90) days`              | P3 roll (1/1)      |
| 3   | ⌃↵                | `🎲 P3 roll · in **60** (31–90) days`                    | P3 → P4            |
| 4   | ⌃↵                | `🎲 P3 → P4 decay · in **120** (91–365) days`            | P4 roll (1/1)      |
| 5   | ⌃↵                | `🎲 P4 roll · in **200** (91–365) days`                  | Cancel task        |
| 6   | ⌃↵                | Cancel Log `*<today>* — 🍂 decayed past P4 after 1 roll` | —                  |

### Reading the roll streak from the Schedule Log

The streak is **derived, never stored**. There is no `[rolls:: N]` field.

- Find the task's managed Schedule Log with `findScheduleLogParent`. Legacy labels are
  accepted, and a marker nested under a grandchild is ignored.
- Walk the marker's **direct child** bullets in document order (newest first). Ignore
  bullets nested under an entry.
- Parse each one with `parseScheduleLogEntryBullet` and classify its `reason` with a
  pure `classifyScheduleLogRollReason(reason)`.
- A reason is **automatic** when it starts with `🎲 ` (U+1F3B2 then a space). Its
  **head** is the text after that prefix, up to the first `·`
  (`SCHEDULE_LOG_AUTO_REASON_SEPARATOR`), or the rest of the reason when there is no
  separator. The head is trimmed.

| Entry (head or reason)                                                                                        | Class       | Effect on the streak                  |
| ------------------------------------------------------------------------------------------------------------- | ----------- | ------------------------------------- |
| `🎲 <L> roll` where `<L>` equals the current level's label and the head has no `→`                            | `roll`      | counts; keep walking                  |
| `🎲 <anything> randomize` (from `bob randomize`, with or without ` from <date>`)                              | `randomize` | transparent: skip it and keep walking |
| `🎲 <from> → <to> decay`                                                                                      | `decay`     | stops                                 |
| `🎲 <from> → <to>` or `🎲 <L>` (a priority row pick, including an unchanged re-pick)                          | `other`     | stops                                 |
| `🎲 <other label> roll` (the priority was hand-edited since)                                                  | `other`     | stops                                 |
| a typed reason, `🤷 no reason given`, `🔓 unblocked by hand`, a pull-forward entry, or any unparseable bullet | `other`     | stops                                 |

The rules in one sentence: **only reasonless, same-level recommended rolls build a
streak; any deliberate scheduling decision resets it.** Picking the priority level
again, or picking a date with a typed reason, are therefore the documented ways to keep
a task at its level. Legacy pinned-roll reasons such as
`🎲 P2 roll · random in 8–30 days` still have head `P2 roll`, so they count.

**Conformance vectors.** Current level P2, limit 1 unless noted. Entries are listed
newest first. These go verbatim into the tests and the docs.

| #   | Schedule Log entries (reason only)                         | Streak    | Recommendation                                  |
| --- | ---------------------------------------------------------- | --------- | ----------------------------------------------- |
| 1   | (no log)                                                   | 0         | roll P2, step 1/1                               |
| 2   | `🎲 P1 → P2 · in **9** (8–30) days`                        | 0         | roll P2                                         |
| 3   | `🎲 P2 roll · in **17** (8–30) days`, `🎲 P1 → P2 · …`     | 1         | decay P2 → P3                                   |
| 4   | `🎲 P2 randomize · in **9** (8–30) days`, `🎲 P2 roll · …` | 1         | decay P2 → P3                                   |
| 5   | `waiting on the API review`, `🎲 P2 roll · …`              | 0         | roll P2                                         |
| 6   | `🤷 no reason given`, `🎲 P2 roll · …`                     | 0         | roll P2                                         |
| 7   | `🎲 P2 · in **12** (8–30) days`, `🎲 P2 roll · …`          | 0         | roll P2                                         |
| 8   | `🎲 P1 roll · …`                                           | 0         | roll P2                                         |
| 9   | `🎲 P2 roll · random in 8–30 days`                         | 1         | decay P2 → P3                                   |
| 10  | current P4: `🎲 P4 roll · …`                               | 1         | cancel; a recurring task is unavailable instead |
| 11  | `decay.rolls: 3`: two `🎲 P2 roll` entries, then three     | 2, then 3 | roll P2 step 3/3, then decay                    |
| 12  | `decay: false`: five `🎲 P2 roll` entries                  | 5         | roll P2, no step                                |
| 13  | P2 has `rolls: 0`, no log                                  | 0         | decay P2 → P3                                   |
| 14  | current P3: `🎲 P2 → P3 decay · …`                         | 0         | roll P3                                         |
| 15  | P4 has `rolls: 0`, current P4, no log                      | 0         | cancel                                          |

### What each recommendation writes

All of these go through the existing writers, so the existing side effects come for
free: Blocked marking for a future date, pruning of today's open-Pomodoro links,
recovery for a due date, freshness stamping via `api.freshness.stampLine` (not on
cancel, matching the Cancel row), and a single editor transaction (one undo) in single
and counted sessions.

- **Roll.**
  - Set `scheduled` to the previewed date: the inline field, or `^prj` frontmatter via
    `setProjectNoteScheduledValue`.
  - Log the unchanged existing reason from
    `formatPriorityRollScheduleReason({ source: "scheduled", … })`, for example
    `🎲 P2 roll · in **17** (8–30) days`. The format does not change, so bob-cli's
    `capture_schedule_log.rs` parity is untouched.
- **Decay.**
  - Set the priority field to the next level's `value`, and `scheduled` to the previewed
    date in the next level's window. Use `setBulletPriorityValue`'s inline and `^prj`
    paths, extended to take a precomputed roll and a reason source.
  - Log a new `source: "decay"` reason, for example
    `🎲 P2 → P3 decay · in **45** (31–90) days`. The general form is
    `${SCHEDULE_LOG_AUTO_REASON_EMOJI} ${from}${SCHEDULE_LOG_TRANSITION}${to} decay${SCHEDULE_LOG_AUTO_REASON_SEPARATOR}in **N** (min–max) days`.
- **Cancel.**
  - Use the existing cancel path (`applyTaskCancelFromPicker`) with
    `reason = formatPriorityDecayCancelReason({ level, streak })` and no fallback.
  - The reason is `🍂 decayed past P4 after 1 roll`, or `… after 3 rolls`. When the
    streak is 0 it is just `🍂 decayed past P4`. `🍂` is U+1F342, one code point.
  - This writes `[-]`, `[cancelled:: <picker date>]` and a first-child
    `❌ **CANCEL LOG**` entry. It prunes today's open-Pomodoro links and recovers
    Blocked dependents through task-status-cycler's api, exactly like the Cancel row.
  - No Schedule Log entry is written, because the date does not change.
- **The date never lands on the task's current `scheduled` date when the window has
  room.**
  - `rollPriorityRecommendationDate(level, baseDate, random, avoidValue)` draws
    uniformly from the window minus the current date's offset. It does so only when that
    offset is inside the window and the window spans more than one day.
  - Otherwise it behaves exactly like `rollPriorityScheduledDateWithOffset`.
  - Why: an automatic entry is skipped when the date is unchanged
    (`shouldWriteAutomaticScheduleLog`), and a skipped entry would silently lose a
    streak step.
  - The pinned stage-two roll row uses the same function.

### Configuration

The `decay` block lives on the priority property in `~/.config/bob/config.yml`. It is
chezmoi-managed (`home/dot_config/bob/config.yml` in the `chezmoi` repo).

```yaml
- name: priority
  values: priority
  schedules: scheduled
  # Ctrl+Enter on `scheduled` takes the recommended roll. Each level allows `rolls`
  # same-level rolls (read from the task's Schedule Log); the next recommended roll
  # decays the task one level (P2 → P3), and past the last level it cancels it.
  # `decay: false` makes Ctrl+Enter a plain same-level roll that never decays.
  decay:
    rolls: 1
  levels:
    - label: P1
      value: high
      min_days: 2
      max_days: 7
      # rolls: 3   # optional: this level's own roll limit
```

Validation goes in `validateBulletPropertyConfig`. Errors use the existing
`showBulletPropertyNotice` style and invalidate the config, as other bad entries do.

- `decay` absent, `true`, or `{}` → `{ enabled: true, rolls: 1 }`
  (`DEFAULT_PRIORITY_DECAY_ROLLS = 1`).
- `decay: false` → `{ enabled: false, rolls: null }`.
- `decay.rolls` must be a non-negative integer. `0` means every recommended roll decays.
  Any other key under `decay` is an error naming the key, so a typo such as `roll: 3` is
  never silently ignored.
- `levels[].rolls` is optional and must be a non-negative integer. It overrides
  `decay.rolls` for that level. When decay is disabled it is accepted and ignored.
- `decay` or `rolls` on a non-priority property is an error, mirroring the existing
  `levels` rule.
- The normalized property gains a frozen `decay: { enabled, rolls }`, and each frozen
  level gains `rolls` (`null` when not set).
  `getPriorityLevelRollLimit(property, level)` returns `level.rolls ?? decay.rolls`, or
  `null` when decay is disabled.

### Look and feel

**Stage one: the `scheduled` row carries the preview.** When a recommendation exists,
`renderPropertyItem` adds a third line under the row's title and `Current value: …`
path. It is `div.bob-cnp-roll-preview.is-<kind>`, where `<kind>` is `roll`, `decay`,
`cancel` or `unavailable`. In order, it holds:

- a small `kbd.bob-cnp-kbd.bob-cnp-roll-kbd` reading `^↵`;
- an icon `span.bob-cnp-roll-icon` (Lucide via `applyIcon`);
- the action `span.bob-cnp-roll-action`;
- for dated kinds, the date `span.bob-cnp-roll-date`;
- an optional `span.bob-cnp-roll-meta` pill.

All copy comes from a pure `buildPriorityRollPreviewModel(recommendation, baseDate)`.
The relative day uses `formatRelativeDayOffset` and the weekday uses
`getBulletPropertyDateWeekday`:

| Kind             | Icon            | Tone   | Action          | Date                            | Meta                                                     | Footer `^↵` label |
| ---------------- | --------------- | ------ | --------------- | ------------------------------- | -------------------------------------------------------- | ----------------- |
| roll (decay on)  | `dices`         | accent | `P2 roll`       | `2026-10-17 · Sat · in 17 days` | `roll 1/1`; warn tone when this is the last allowed roll | `Roll P2`         |
| roll (decay off) | `dices`         | accent | `P2 roll`       | same                            | —                                                        | `Roll P2`         |
| decay            | `trending-down` | warn   | `P2 → P3`       | `2026-11-29 · Sun · in 60 days` | `decay`                                                  | `Decay to P3`     |
| cancel           | `ban`           | danger | `Cancel task`   | —                               | `decayed past P4`                                        | `Cancel task`     |
| unavailable      | `circle-slash`  | muted  | `Cannot cancel` | —                               | `recurring · use Obsidian Tasks`                         | (no hint)         |

- **Tones.**
  - accent = `--text-accent`, matching the existing `.is-priority-roll` row;
  - warn = `--color-orange`;
  - danger = `--text-error`;
  - muted = `--text-muted`.
  - Tints use the stylesheet's existing `color-mix(in srgb, <tone> N%, transparent)`
    idiom.
- **Size and emphasis.** The line uses `--font-ui-smaller` and `tabular-nums` for the
  date. It is at 0.8 opacity on an unselected row and full opacity on the selected row,
  so the date is always visible near `scheduled` but only loud when it is actionable.
- **Accessibility.** The line gets an `aria-label` with the full sentence, for example
  `Ctrl+Enter: P2 roll to Sat 2026-10-17, in 17 days, roll 1 of 1`.
- **Filtering.** The row's `filterItem` text also includes `roll`, the action and the
  meta text. Typing `roll` or `decay` therefore lands on `scheduled`.
- **Stage-one footer.** When the recommendation is actionable, the footer is
  `↑↓ Navigate · ^N ^P Move · ↵ Choose · ^↵ <label> · ^R Re-roll · ^D Delete · esc Dismiss`.
  The `^R Re-roll` hint appears only for dated kinds (roll and decay). The footer
  already wraps.

**Stage two.**

- The pinned `🎲 P2 roll` row stays the same-level roll, as today.
- When the recommendation is a **roll**, the pinned row is created from the
  recommendation's date and offset (one shared roll, so ⌃↵ and ↵ on the pinned row write
  the identical date). Ctrl+R re-rolls both together.
- When the recommendation is a decay or cancel, the pinned row keeps its own same-level
  roll. Its detail gains ` · over P2's roll limit`, so the user can see it overrides the
  ladder.
- The footer gains `^↵ <label>` before `esc`.

**Notices.**

- **Roll or decay.** Reuse the priority notice card. `buildPriorityNoticeModel` takes an
  optional `roll: { kind, fromLevel, step, limit, next }` and changes these parts:
  - **Pill.** Roll → `P2 roll`; decay → `P2 → P3`, using the new level's `is-level-N`
    class.
  - **Text header.** Roll → `scheduled → P2 roll`; decay →
    `priority → P3 (low) · decayed from P2`.
  - **Leading chips**, before the existing outcome chips:
    - roll → `roll 1/1` (info);
    - decay → `decayed from P2` (warn);
    - plus a **next** chip only when the _next_ ⌃↵ would decay or cancel: `next ^↵ → P3`
      (warn) or `next ^↵ cancels` (warn).
  - With decay disabled there are no roll or next chips.
- **Cancel.** Use the existing Cancelled card. Its reason quote already reads
  `🍂 decayed past P4 after 1 roll`.

### Reliability guarantees

- **What you see is what you get.** The preview's date is rolled once when the picker
  opens (or on Ctrl+R), using `this.priorityRandom` and `this.valueBaseDate`. The write
  uses exactly that date and offset.
- **Re-verify before writing.**
  - Re-read the live content and recompute the recommendation from the live task line
    and Schedule Log.
  - If the kind, the from or to level, or the target line differs from the preview,
    write nothing.
  - Show `Task changed while the picker was open; nothing was written`, then re-render
    stage one with the fresh recommendation, so the next ⌃↵ does what is now shown.
  - The writers' own `expectedLine` and preimage guards still apply on top of this.
- **No double fire.** Ctrl+Enter honours `this.opening` like `openItemAtIndex` does.
- **One undo step** in single and counted sessions. Link sessions keep the existing
  per-note preimage, write and rollback core.
- **Pure core.** Config, streak, recommendation, reasons and preview models are pure
  functions. They are exported on `module.exports.helpers` and covered by a new
  `scripts/test-navigation-roll-decay.cjs`, which is registered in `package.json`'s
  `test` script.

### Batch sessions (counted and Task Link)

- **Recommendations.**
  - Every open target with a configured priority gets its own recommendation from its
    own line and Schedule Log. Link targets are read from their own note's content.
  - Each one has its own pre-rolled date, using the counted priority writer's
    one-roll-per-target approach.
  - Targets with no recommendation (closed, no priority, unconfigured value) are
    **skipped** and reported. If no target has a recommendation, there is no preview and
    ⌃↵ behaves like ↵.
- **Preview.** One line on the `scheduled` row, for example
  `^↵ 4 tasks · 2 roll · 1 decay · 1 cancel · 2026-10-05 → 2026-12-19`, plus
  ` · 1 skipped` (muted) when targets are skipped.
  - The tone and icon follow the most severe kind present: cancel > decay > roll.
  - The footer label is `Roll N tasks` when every target is a roll, otherwise
    `Apply N recommendations`.
- **Unavailable.** If any target recommended for cancel is recurring, the whole batch is
  unavailable. It shows a muted line
  `Cannot apply · a recurring task would be cancelled`, and ⌃↵ shows the recurring
  Notice and writes nothing. This matches `planTaskCancelBatch`'s whole-batch refusal.
- **One composed write.** For each note:
  1. Run `planTaskCancelBatch` for the cancel targets, with a new `reasonByLine` option
     so each target gets its own streak count.
  2. Map the remaining target lines through its `cursorLineShift`.
  3. Run `planCountedBulletPropertyBatch` (`set-priority`) for the roll and decay
     targets, with a new `priorityValueByLine` option. It holds each target's current
     value for a roll and the next value for a decay, along with `scheduledValueByLine`
     and `scheduleLog.reasonByLine`.
  4. Prune the union of block IDs from today's open Pomodoros in one pass: cancelled
     targets, plus targets now scheduled in the future.

  Counted sessions apply the final content in one `applyEditorContentTransaction`. Link
  sessions commit the per-note postimages through `commitLinkPickerNoteWrites`, then run
  task-status-cycler dependent recovery for the cancelled identities.

- **Stage two in batch sessions.**
  - When every target is a same-level roll at one shared level, the recommendation
    replaces the existing shared-date pinned roll row. Rolls are per-target, so this
    avoids a duplicate row.
  - Otherwise the existing pinned roll row stays as the explicit override.
  - Link sessions still have no pinned roll row; ⌃↵ works there through the
    recommendation.
- **Batch notice.** One priority-style card:
  - header icon `dices`;
  - pill `Rolled N tasks`;
  - leading chips `2 rolled` (info), `1 decayed` (warn) and `1 cancelled` (warn);
  - the rolled date span;
  - then the existing outcome chips (Blocked, removed Pomodoro links, skipped,
    prune-failed).

### Deliberate choices (rejected alternatives)

- **A derived streak, not a stored counter.**
  - A `[rolls:: N]` field was rejected. Tasks parsers read trailing inline fields right
    to left and stop at the first one they do not recognize, which is why priority
    values must stay Tasks names. A stored counter also drifts from the history it
    summarizes.
  - The Schedule Log already records every roll, and deriving the streak follows the
    project's "derived, not authored" decisions.
- **Only recommended rolls count.** Counting every reschedule would punish deliberate
  decisions. Decay targets reasonless rollovers.
- **A decay is not a roll at the new level.** Counting it would contradict the request
  that a task's first roll at a level recommends the same level.
- **`bob randomize` is transparent.** It is a catch-up after time away. Counting it
  would decay the whole backlog after a vacation, and letting it reset the streak would
  erase real rollovers.
- **Only ⌃↵ decays or cancels.** A pinned decay or cancel row in the date list would
  turn the habitual ↵, ↵ into a cancel. The user scoped the ladder to Ctrl+Enter, and ↵
  stays the explicit, safe path.
- **No reason prompt at the end of the ladder.** The request is one keypress. The red
  preview warns first, the `🍂` reason documents the cancel, and Ctrl+Z restores single
  and counted edits.
- **Never on its own.** No automation ages, decays or cancels tasks. That keeps
  `decisions:task-lanes-are-sticky`'s rejection of age-based decay intact.

### Non-goals

- `bob randomize`, `bob capture p:<N>` and Bob Mac Capture are unchanged. They only
  ignore the new keys. A decay-aware `bob randomize` is a possible follow-up, not part
  of this epic.
- No change to the priority row, the Cancel row, the lane row, or ↵ behaviour anywhere.
- No SASE memory changes.

## Phase decay-core

Work in the linked `bob-plugins` repo (`/sase_repo`). Change only
`plugins/bob-navigation-hotkeys/main.js`, a new
`scripts/test-navigation-roll-decay.cjs`, and `package.json`.

1. **Config.**
   - Add `DEFAULT_PRIORITY_DECAY_ROLLS` and
     `normalizeBulletPriorityDecay(name, raw, options)`, and per-level `rolls` parsing
     in `normalizeBulletPriorityLevel`.
   - Wire them into `normalizeBulletPriorityProperty` and the non-priority rejection in
     `validateBulletPropertyConfig`.
   - Add `getPriorityLevelRollLimit(property, level)`. Follow the Configuration section
     exactly, including error wording in the existing
     `Bullet property "<name>" … must …` style.
2. **Streak.**
   - Add `classifyScheduleLogRollReason(reason)`, returning
     `{ kind: "roll", label } | { kind: "randomize" } | { kind: "decay", from, to } | { kind: "other" }`.
   - Add `readPriorityRollStreak(content, taskLine, level)`, returning
     `{ count, logLine }` per the classification table. Only the marker's direct child
     bullets are read, newest first.
3. **Date.** Add `rollPriorityRecommendationDate(level, baseDate, random, avoidValue)`,
   returning `{ date, value, offset }`, per "What each recommendation writes".
4. **Recommendation.** Add
   `planPriorityRollRecommendation({ property, lineText, content, taskLine, currentScheduled, baseDate, random })`.
   It returns `null`, or a frozen record:
   `{ kind: "roll"|"decay"|"cancel", level, levelIndex, toLevel, toLevelIndex, streak, limit, step, date, value, rolledDays, unavailableReason }`.
   - `toLevel` is the same level for a roll, the next level for a decay, and `null` for
     a cancel.
   - `step` is set only for a roll with a limit.
   - `unavailableReason` is `"recurring"` for a cancel on a recurring line (use
     `isRecurringTaskLine`), else `null`.
   - It returns `null` exactly in the "When a recommendation exists" cases.
5. **Reasons.**
   - Extend `formatPriorityRollScheduleReason` with `source: "decay"` (head
     `${fromLevelLabel} → ${level.label} decay`).
   - Let `buildPriorityRollScheduleLog` accept it.
   - Add `formatPriorityDecayCancelReason({ level, streak })` with the
     `PRIORITY_DECAY_CANCEL_EMOJI = "🍂"` constant.
6. **Preview model.** Add `buildPriorityRollPreviewModel(recommendation, baseDate)`,
   returning
   `{ kind, tone, iconName, actionText, dateText, metaText, metaTone, hintLabel, ariaLabel, searchText }`,
   with the exact copy from the Look-and-feel table.
7. **Tests.**
   - Export every new helper on `module.exports.helpers`.
   - Create `scripts/test-navigation-roll-decay.cjs`. Copy the Obsidian stub preamble
     from `scripts/test-navigation-stamps.cjs`.
   - Cover:
     - config defaults, `false`, overrides and every error;
     - all 15 conformance vectors, built as real note text with a `🗓️ **SCHEDULE LOG**`
       marker, including a legacy `**Schedule log:**` marker and an ignored nested
       grandchild marker;
     - date avoidance, including a 1-day window that cannot avoid, and determinism with
       an injected `random`;
     - the exact reason and preview strings.
   - Add the file to `package.json`'s `test` script.
8. **Verify.**
   - Run `npm test` and `npm run validate` (no manifest bump; nothing user-visible
     changes).
   - Run `bob plugins sync -p bob-navigation-hotkeys -r <bob-plugins path>`, with
     `--dry-run` first.

## Phase picker-single

Work in the linked `bob-plugins` repo. Single tasks and `^prj` tasks only: no
recommendation when `isCountedSession()` or `isLinkSession()`.

1. **State.**
   - In `BulletPropertyPickerModal`, compute `this.rollRecommendation` when building
     stage one. Use the `scheduled` property item's current value: the inline value, or
     the frontmatter value for `^prj`.
   - Keep it across stage changes until a write or a Ctrl+R.
   - Make `getPriorityRollLevel` and `createPriorityRollDateItem` use
     `rollPriorityRecommendationDate` (avoiding the current date). When the
     recommendation is a roll, seed the pinned item from it.
2. **Stage one.**
   - Render the preview line in `renderPropertyItem` for the recommendation's date
     property row.
   - Extend that row's filter text.
   - Build the footer from a new `getBulletPropertyStageOneHints(previewModel)`.
3. **Keys.** In `BulletPropertyPickerModal.handleKeydown`, before `super.handleKeydown`:
   - **Ctrl/Cmd+Enter** (no Alt or Shift):
     - in stage `properties` with the date property row selected, or in that property's
       stage `value`: run `applyRecommendedRoll()`;
     - otherwise: fall through to ↵.
   - **Ctrl+R in stage `properties`:** re-roll a dated recommendation and re-render.
   - **Ctrl+R in stage `value`:** keep `rerollPriorityDateSuggestion`, and sync the
     recommendation when it is a roll.
4. **`applyRecommendedRoll()`.**
   - Honour `this.opening`, and re-verify per Reliability.
   - **unavailable:** show the Notice and return.
   - **roll:**
     - inline: `setInlineBulletPropertyValues` with the date, `expectedLine`, `today`,
       the `source: "scheduled"` schedule log, and a roll `buildNotice`;
     - `^prj`: `setProjectNoteScheduledValue` with the same.
   - **decay:** `setBulletPriorityValue`, extended with `context.roll`
     (`{ date, offset }`), `context.reasonSource: "decay"` and `context.decay` (for the
     notice). Both its inline and `^prj` paths must honour them.
   - **cancel:**
     `this.plugin.applyTaskCancelFromPicker(this, { reason: formatPriorityDecayCancelReason(…), fallbackReason: "" })`.
   - Close the picker on success, as the other writers do.
5. **Notices.** Extend `buildPriorityNoticeModel`, `formatPriorityNoticeText` and
   `renderPriorityNoticeFragment` with the `roll` option. Keep the existing text
   unchanged when `roll` is absent: existing tests pin it.
6. **CSS.** In `styles.css`, add `.bob-cnp-roll-preview` and its `is-*` tones,
   `.bob-cnp-roll-kbd`, `.bob-cnp-roll-icon`, `.bob-cnp-roll-action`,
   `.bob-cnp-roll-date` and `.bob-cnp-roll-meta` (`is-warn`), plus selected/unselected
   opacity. Match the existing `.is-priority-roll` and `.bob-cnp-cancel-row` treatments.
7. **Tests** (in `scripts/test-navigation-roll-decay.cjs`). Drive the modal through
   `handleKeydown` with a `TestEditor`, as `scripts/test-navigation-hotkeys.cjs` does.
   - roll, decay and cancel from stage one, and from stage two with a preset row
     highlighted;
   - ↵ unchanged; Ctrl+Enter on another row equals ↵; no recommendation (P0, closed
     task) equals ↵; a recurring cancel shows the Notice and writes nothing;
   - a stale task (line or log edited after open) refuses and refreshes;
   - Ctrl+R in both stages, and the shared pinned date;
   - `^prj` roll and decay into frontmatter;
   - one undo step;
   - exact Schedule Log and Cancel Log lines;
   - notice text, footer hints and the preview `aria-label`.
8. **Ship.**
   - Bump the manifest's minor version (1.44.0 → 1.45.0 at the time of writing) and
     mention roll decay in its description.
   - Add a clause to the `README.md` Plugins-table row describing ⌃↵, the preview and
     the ladder.
   - Run `npm test` and `npm run validate`, then
     `bob plugins sync -p bob-navigation-hotkeys -r <bob-plugins path>` (dry-run first).

## Phase picker-counted

Work in the linked `bob-plugins` repo.

1. **Batch planning.** Add a pure
   `planPriorityRollRecommendationsForTargets(content, targets, property, options)`. It
   returns per-target recommendations or skip reasons, kind counts, the date span, the
   skipped count and a whole-batch `unavailableReason`. Also add a pure
   `buildBatchPriorityRollPreviewModel(summary)` that renders the batch copy from "Batch
   sessions".
2. **Planner extensions.**
   - Add `priorityValueByLine` to `planCountedBulletPropertyBatch` (`set-priority`).
     When a line has an entry, it overrides the shared `priorityValue`.
   - Add `reasonByLine` to `planTaskCancelBatch`. When a line has an entry, it overrides
     `reason`.
   - Callers that do not pass the new options must get byte-identical output.
3. **Composed planner.** Add a pure
   `planRecommendedRollBatch(content, session, recommendationsByLine, details)`. It runs
   cancel first, then line remapping, then `set-priority`, and returns one postimage
   plus prune targets, recovery inputs and counts.
4. **Editor write.** Add `applyCountedRecommendedRoll(picker)`, mirroring
   `setCountedBulletPriorityValue` and `applyTaskCancelFromEditor`. It keeps their
   guards (`getCountedTaskWriteContext`, stale-session refusal), does a same-file or
   deferred Pomodoro prune, applies one `applyEditorContentTransaction`, and calls
   task-status-cycler recovery for cancelled identities.
5. **UI.**
   - Enable the recommendation for counted sessions: preview, footer, ⌃↵ in both stages,
     and Ctrl+R re-roll of every pre-rolled date.
   - Apply the stage-two pinned-row rule from "Batch sessions".
   - Add the batch notice.
6. **Tests.** Cover:
   - a mixed roll, decay and cancel batch with exact output lines and one undo step;
   - a batch with skipped (P0 and closed) targets;
   - recurring-cancel refusal;
   - a stale session;
   - planner byte-identity without the new options;
   - the preview and notice copy.
7. **Ship.** Bump the minor version, update the README row, then run `npm test`,
   `npm run validate` and `bob plugins sync` (dry-run first).

## Phase picker-links

Work in the linked `bob-plugins` repo.

1. **Recommendations.** For a `linkSession`, compute each resolved target's
   recommendation from its own note's content (`groupLinkPickerTargetsByNote`). Reuse
   `planPriorityRollRecommendationsForTargets` across groups.
2. **Write.** Add `applyLinkRecommendedRoll(picker)`.
   - Plan each group with `planRecommendedRollBatch`.
   - Build `recoveryByLine` where a rolled date is due, as
     `applyLinkPickerPriorityValue` does.
   - Commit through `commitLinkPickerPlans` / `commitLinkPickerNoteWrites` with one
     daily-note prune, then run task-status-cycler dependent recovery for cancelled
     identities, as `applyTaskCancelFromLinkPicker` does.
   - Any changed preimage refuses the whole operation.
3. **UI.**
   - Enable the preview, footer and ⌃↵ in link sessions, using the single-task copy when
     there is exactly one target and the batch copy otherwise.
   - Link sessions still have no pinned stage-two roll row.
4. **Tests.** Cover:
   - a single Task Link (roll, decay and cancel) into another note;
   - several links across two notes;
   - a stale preimage refusing everything;
   - the prune of today's open Pomodoro links.
5. **Ship.** Bump the minor version, update the README row, then run `npm test`,
   `npm run validate` and `bob plugins sync` (dry-run first).

## Phase decay-docs

1. **`docs/projects.md`** (bob-cli).
   - Add `### Recommended roll and priority decay` after "Priority property and
     scheduled rolls", and add it to the contents list. It covers the gesture table, the
     ladder and lifecycle tables, the streak classification table and conformance
     vectors, the configuration block, batch behaviour, and the documented ways to keep
     a task at its level.
   - In "Schedule-log reason prompt", add a reason-table row for
     `🎲 <from> → <to> decay · in **<chosen>** (<min>–<max>) days`. Note that the
     recommended same-level roll writes the existing `🎲 <level> roll` reason.
   - In "Cancelling a task", mention the `🍂 decayed past <level> after <n> roll(s)`
     reason.
2. **`docs/randomize.md`.** Add one sentence: `🎲 <level> randomize` entries are
   transparent to the roll streak.
3. **`docs/freshness.md`.** In the Surfaces table, add the ⌃↵ recommended roll (roll and
   decay) to bob-navigation-hotkeys' stamping surfaces, and the decay cancel to its
   non-stamping list, matching the Cancel row.
4. **Rust guard test** (`src/native/config/mod.rs` tests). Assert that
   `parse_priority_property` accepts a property with `decay: { rolls: 2 }`, with
   `decay: false`, and with per-level `rolls: 3`, and returns the same levels as without
   them. Run `cargo test`.
5. **chezmoi.**
   - Open `chezmoi` with `/sase_repo`. In `home/dot_config/bob/config.yml`, add the
     commented `decay: { rolls: 1 }` block shown in Configuration. It matches the
     default, so the deployed file needs no `chezmoi apply`.
   - Validate the edited YAML through the plugin's
     `helpers.validateBulletPropertyConfig`, using a short node script fed by
     `python3 -c 'import yaml, json, sys; …'`.
   - Commit it with that repo's obligations.
6. Run `just all` (fmt, lint, test) in bob-cli.

## Acceptance

- **Single task.** On a P2 task with no log, the `scheduled` row shows
  `^↵ 🎲 P2 roll · <date> · <weekday> · in N days · roll 1/1`.
  - ⌃↵ writes exactly that date and `🎲 P2 roll · in **N** (8–30) days`.
  - Reopening the picker shows the orange `P2 → P3` decay preview. ⌃↵ writes
    `[priority:: low]`, a 31–90-day date, and `🎲 P2 → P3 decay …`.
  - After a P4 roll, the red `Cancel task` preview appears, and ⌃↵ cancels with
    `🍂 decayed past P4 after 1 roll`.
- **↵ is unchanged.** ↵, ↵ on the pinned row still rolls the same level, never decays,
  and counts toward the streak.
- **Configurable.** `decay.rolls: 3` gives three same-level rolls per level.
  `levels[].rolls` overrides one level. `decay: false` makes ⌃↵ a plain same-level roll.
  Invalid values raise a config Notice.
- **Resets.** Re-picking the priority, or a typed-reason reschedule, resets the streak.
  `bob randomize` entries do not.
- **Batches.** `N<Ctrl+Shift+P>` and Task Link sessions preview and apply a mixed batch
  in one guarded write: one undo step in counted sessions, all-or-nothing preimages in
  link sessions.
- **Reliability.** A stale task refuses with a refreshed preview. A recurring cancel
  writes nothing. Every write is logged. The date never silently equals the current
  date.
- **Done.** `npm test`, `npm run validate` and bob-cli's checks pass. The plugin is
  deployed with `bob plugins sync`, and the docs and chezmoi config describe the
  feature.
