---
tier: tale
title: Make the Task Card the only Ctrl+Shift+P surface
goal: 'Ctrl+Shift+P always opens the Task Card. Remove the pilot setting, the 2026-10-19
  date gate, and the classic "Set bullet property" list, including type-to-search.
  Escape, q, and Ctrl+] close the card without writing. Stages the card already opens
  stay. The freshness-decay trial date stays.

  '
size: medium
proposed_by: bbugyi200.athena.0w4
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0w4](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0w4.md)
- **COMMITS:**
  - [b1332af](https://github.com/bobs-org/bob-cli/commit/b1332af2d23335be72d2827eb868193e289d0819)
    — feat(ready): always advertise the Task Card keys in the crowded-note hint

# Make the Task Card the only Ctrl+Shift+P surface

## Outcome

Pressing `Ctrl+Shift+P` (`bob-navigation-hotkeys:set-bullet-property`, palette **Task
card (set properties)**) always opens the Task Card. There is no plugin setting, no
Automatic / Task Card / Classic list choice, and no local-calendar switch on 2026-10-19.
The classic first screen in `~/tmp/screenshots/20261004_051911.png` cannot appear: no
title "Set bullet property", no "Filter properties" box, no property-list footer
(`esc Dismiss`).

On the card, `Escape`, bare `q` / `Q`, and `Ctrl+]` close it and discard uncommitted
state. Nothing is written. `Ctrl+]` also closes stages this same modal has opened. Bare
`q` does not close while a text field is focused.

This is a medium tale. One agent can do it. The root cause and the deletion boundary are
known, and the card, its writers, and its later stages already exist. The work is still
more than a small edit because the classic path is threaded through
`BulletPropertyPickerModal`, the schedule commit branch, the settings tab, plugin tests,
the `bob ready` hint, and the docs.

## Why the screenshot is the old list

Today is 2026-10-04. In `plugins/bob-navigation-hotkeys/main.js`:

- `taskCardPilotEnabled` returns true only when saved `taskCard === true`, or when
  `taskCard` is absent/null and `taskCardDefaultEnabled(now)` is true.
- `taskCardDefaultEnabled` is false before local 2026-10-19 and true from that day. Any
  other saved value, including `taskCard: false` (Classic list), stays classic.
  `loaded !== true` also forces classic, so a settings load that has not finished or has
  failed opens the old list.
- `BulletPropertyPickerModal` sets `taskCardEnabled` from `context.taskCard === true`.
  When that is false it calls `showPropertyStage`, which is the filtered list titled
  "Set bullet property". That matches the screenshot (Commit to Next, Refresh every,
  scheduled, dependsOn, priority, Cancel task, `esc` Dismiss).
- `openBulletPropertyPicker` and `openLinkPicker` pass
  `taskCard: this.isTaskCardPilotEnabled()` unless the caller overrides it. The setting
  tab `TaskCardPilotSettingTab` persists the override through `setTaskCardPreference` /
  `mergeTaskCardPilotPreference`.

The landed epic `plan:202610/ctrl_shift_p_task_card.md` (bead `bob-cli-42`) kept that
gate on purpose, and it kept classic search (`/` or any unbound printable, and the More
row) as a permanent way back to the same list. This tale removes both. The card is the
only surface the keymap supports.

A vault `data.json` was not required to explain the screenshot. The default date gate
alone shows the old list on 2026-10-04. Deleting the gate also ignores a saved Classic
list value. Do not hunt for or rewrite vault `data.json`.

## Repository map

Run from the assigned bob-cli checkout. Before reading or editing bob-plugins, run
`sase repo open bob-plugins -r "Remove the Ctrl+Shift+P classic list and pilot setting"`
and use only the printed path. Read that repo's `AGENTS.md`. Never edit an installed
vault copy of the plugin. After bob-plugins source changes, run `bob plugins sync` so
the vault matches the source. Paths below are relative to the repo that owns them.

Do not edit `sase/memory/`. The schedule-log glossary strand still mentions classic
search; this tale does not authorize that edit. Decision notes that mention the picker
stay as they are. The freshness-decay trial is a different gate
(`FRESHNESS_DECAY_ACTIVE_FROM` in both navigation-hotkeys and bob-ledger-tools,
documented in `docs/freshness.md`). Leave it on 2026-10-19.

## Removal boundary

Delete the user-facing classic list and the opt-in that selects it. Keep the machinery
the card uses to perform an action.

Delete:

- `taskCardDefaultEnabled`, `taskCardPilotEnabled`, `taskCardPreferenceValue`,
  `mergeTaskCardPilotPreference`, `TaskCardPilotSettingTab`, and the plugin methods
  `isTaskCardPilotEnabled`, `loadTaskCardSettings`, and `setTaskCardPreference`,
  including their `onload` registration and test exports.
- Every read and write of the persisted `taskCard` flag. A leftover key in saved plugin
  data is ignored. Do not add a migration.
- The constructor branch that paints `showPropertyStage` when the card is off, and the
  link-resolution branch that paints it after targets resolve. The separate "Resolving
  Task Link" classic shell goes away with them. While targets resolve, the existing card
  loading behavior remains, and `q` and `Ctrl+]` close it the same way `Escape` does.
- `showSearchFromCard` and every `open-search` intent. `/`, unbound letters, and digits
  that do not map to a configured P-level must not open the old list and must not write.
  Configured `1`–`4` (and further configured digits through `9`) still set that level.
  `0` still clears priority.
- The `more-properties` action-list row whose only job is to open the classic list. The
  card's More section stays. Activating one of those rows opens that property's existing
  value stage (`showValueStage` or the schedule commit path when it is a date the card
  already knows how to commit). It does not seed a filter list. `onOpenProperty` today
  calls `showSearchFromCard`.
- The `taskCardEnabled === false` schedule path: serial reason prompt, then Work Log
  prompt, and the classic-only typed-date row that rejects bare `N`, unsigned
  `Nd`/`Nw`/`Nm`, and weekday names. Once the card is the only entry, schedule commits
  always use the concise resolver and the combined Reason / Work summary review. Keep
  `parseBulletPropertyTypedDate` while the concise resolver or another date item still
  calls it. Delete `createBulletPropertyTypedDateItem` only if nothing calls it after
  the classic branch is gone.
- `bob ready`'s date split. `task_card_default_from` and the pre-2026-10-19 string
  `sequence / defer / drop Ctrl+Shift+P` go away. The hint is always the card wording,
  without `type to search`:
  `split Ctrl+Shift+N · defer Ctrl+Shift+P 1–4 · drop Ctrl+Shift+P x · sequence Ctrl+Shift+P b`.
  `ready_gesture_hint` no longer takes a day for this choice.

Keep:

- The Task Card model, view, styles, priority strip, recommendation banner, and dispatch
  into existing writers.
- Value stages the card opens: schedule, Depends on, Review every, cancel reason, lane
  release, and the combined schedule review. Empty-Backspace and the visible Back
  control still return to the card without writing.
- Direct stage skips that are not the property list. `initialProperty: "dependsOn"` from
  a Depends-On line, a dependency chip, and the Edit task dependencies palette command
  still open the dependency value stage without painting the card or the classic list.
  Decay **Less often** still uses `openFreshnessDecayLessOftenPicker`. Do not route
  those through the card.
- Property-item builders (`createBulletPropertyItems` and the counted and link variants)
  and deletion / lane / refresh / cancel / dependency writers.
- `showPropertyStage` only if a remaining caller still needs it as a non-visible item
  setup. No user-visible path may paint the "Set bullet property" list. If the only
  remaining callers immediately switch to a value stage or delete, build the items
  without painting the list and delete the unused presentation.
- Command id `bob-navigation-hotkeys:set-bullet-property` and the palette name **Task
  card (set properties)**.
- `FRESHNESS_DECAY_ACTIVE_FROM` and the decay card's behavior. Its `x` alias for Drop
  already runs with no date check (`FreshnessDecayCardModal.handleKey`). The footer text
  is the only decay use of `taskCardDefaultEnabled`. After that helper is deleted, the
  footer always includes the existing `D / X drops` wording. Do not add `q` or `Ctrl+]`
  to the decay card. Do not change bob-ledger-tools' decay date.

`FreshnessDecayCardModal` must not import a deleted helper. The footer change above is
the whole decay-card edit.

## Close keys

Add a helper next to the existing Ctrl+[ detector in
`isClearSearchHighlightEscapeKeydown`. That helper matches BracketLeft and must stay as
it is. The new helper matches the requested chord only:

- `ctrlKey` set, `altKey`, `metaKey`, and `shiftKey` unset
- `event.code === "BracketRight"` or `event.key === "]"`

Bare `q` means `event.key` is `q` or `Q`, with `ctrlKey`, `metaKey`, and `altKey` unset.
Shift may be set for `Q`. `Ctrl+Q` does not close. Repeat and a second `close()` are
harmless. Composition (`isComposing` or `keyCode === 229`) never closes.

`resolveTaskCardKey` returns `{ type: "close-card" }` for `Escape`, bare `q` / `Q`, and
the `Ctrl+]` chord when the event is not in a text field and not composing. The existing
text-field guard stays in front of `q`, so `q` inside an input, textarea, select, or
contenteditable returns null. Update the card tests that currently expect `/` and `z` to
be `open-search`, `5` on a four-level strip to be `open-search`, and `q` with `ctrlKey`
to be null. `Ctrl+Q` stays null. Bare `q` and `Ctrl+]` become `close-card`.

`BulletPropertyPickerModal` also closes on `Ctrl+]` from every stage this modal shows,
including a focused date, filter, reason, or Work summary field. Closing discards
uncommitted state and writes nothing, which is what `Escape` already does through
Obsidian. Handle the chord even though the pure resolver returns null for text fields.
`preventDefault` and stop the event so it does not reach the editor. Calling `close()`
twice must not write.

The card footer that now says
`type to search · Ctrl+D clear selected property · Esc close` names `Ctrl+D` and close
via `Esc`, `q`, and `Ctrl+]`. It does not mention type-to-search.

## Docs and version

Bump `bob-navigation-hotkeys` from 2.0.1 to 2.1.0 in `manifest.json` and the bob-plugins
README version table. Rewrite the manifest description and the README Task Card section
so they describe one surface: the card opens immediately, More opens a property's value
stage, and `Esc` / `q` / `Ctrl+]` close. Remove Automatic, Classic list, 2026-10-19
activation, classic search, and the classic parser.

In bob-cli, update the same claims in:

- `docs/projects.md` (Task Card setting, classic search gestures, classic parser, serial
  prompts, "Task Card is off")
- `docs/plan.md` (the ready-hint date split and the Classic list exception)
- `docs/freshness.md` (the classic-search compatibility sentence and the stamp cell that
  says classic-search rows; card actions and the stages they open still stamp)
- `docs/task-dependencies.md` (the `#task` line row that offers Depends on via classic
  search; `b` on the card remains)

Leave the 2026-10-19 freshness-decay sentences in `docs/freshness.md`, `README.md`, and
bob-ledger-tools alone. Leave `docs/randomize.md` dates alone. They are not this gate.

`src/native/note_ready/render.rs` and `tests/cli/ready.rs` follow the hint change above.
A `bob ready` run before 2026-10-19 must show the card keys and must not show
`sequence / defer / drop Ctrl+Shift+P` or `type to search`.

## Verification

From the opened bob-plugins checkout:

- `npm test`
- `npm run validate`

From the bob-cli checkout, the ready-hint tests, including
`overview_human_advertises_task_card_keys_from_october_19` renamed to match the
always-on hint, plus the render unit test that currently switches on October 19.
`cargo test` for those targets is enough. Do not treat the known unrelated `just lint`
clippy deny in `tests/cli/capture/pomodoro_name.rs` as this tale's failure.

Then `bob plugins sync` from bob-cli, as bob-plugins `AGENTS.md` requires.

Manual Obsidian GUI acceptance is still unavailable in this environment. Say so. The
automated modal harness is the verification for open, close, and the absence of the
classic list.

## Non-goals

- Do not rewrite priority, schedule, dependency, refresh, cancel, or lane writers, and
  do not change their stored side effects.
- Do not remove concise scheduling input or the combined review.
- Do not change the freshness-decay activation date, decay-card actions, or
  bob-ledger-tools.
- Do not edit SASE memory, vault notes, or plugin `data.json`.
- Do not add `q` or `Ctrl+]` as global closes for other modals.
