---
tier: tale
size: medium
title:
  "Finish and land epic bob-cli-34: one-undo inline roll writes, decay notice copy,
  end-to-end Ctrl+Enter tests"
goal:
  Every Ctrl+Enter recommended-roll write in the Ctrl+Shift+P picker is one undo step,
  the decay notice text matches the epic plan, end-to-end tests lock in single and
  counted Ctrl+Enter behavior, and epic bob-cli-34 is closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-34.land
bead: bob-cli-34
status: done
---

- **PARENT:**
  [202609/priority_roll_decay.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/priority_roll_decay.md)
- **BEAD:**
  [bob-cli-34](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-34/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-34.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-34.land.md)
- **COMMITS:**
  - [9da50dd](https://github.com/bobs-org/bob-plugins/commit/9da50dd5ea9959823c9a7c288383f8992fbf6b2b)
    — feat(nav-hotkeys): land priority roll decay epic bob-cli-34

# Plan: finish and land epic bob-cli-34 (priority roll decay)

## Context

Epic **bob-cli-34** ("Priority roll decay: Ctrl+Enter takes the recommended roll") has
all five phases closed. Its land agent verified the work and found three gaps the epic
itself caused. Each gap breaks something the epic's plan promised. This tale fixes them
and then closes the epic. Nothing resumes this landing after you, so the closeout in
step 5 is part of this tale.

The epic's design contract is in its plan file. Get the path from the `PLAN` line of
`sase bead read bob-cli-34 -r "<why>"`; it is `202609/priority_roll_decay.md` in the
plan store. The epic's goal says **"every write is a guarded, logged, one-undo edit"**.

What the land agent already verified (do not redo it):

- `npm test` passes 990/990 and `npm run validate` passes 6/6 in `bob-plugins`.
- The vault matches the repo (`bob plugins sync --dry-run` reports everything up to
  date).
- bob-cli docs, the `ignores_decay_and_rolls_keys` Rust test, and the chezmoi commented
  `decay` block are in place and pushed.
- No `--epic-symbol` entries exist for bob-cli-34.
- Follow-up triage is done and recorded on the epic. The flaky `capture_pomodoros` test
  is a duplicate of bug bob-cli-2e and was corroborated there. Do not file it again.
- An ad-hoc smoke run drove the real picker through `handleKeydown`. Single-task roll,
  decay and cancel, stage-two Ctrl+Enter, stale refusal, the `^prj` decay into
  frontmatter, P0 fall-through to ↵, and a counted mixed batch all write the exact lines
  the plan specifies. The counted batch, cancel and `^prj` writes are already one editor
  transaction each.

All code work is in the linked **`bob-plugins`** repo. Open it with
`sase repo open bob-plugins -r "Finish epic bob-cli-34 landing"`, read its `AGENTS.md`,
and use only the path that command prints. Work only in
`plugins/bob-navigation-hotkeys/` (`main.js`, `manifest.json`),
`scripts/test-navigation-roll-decay.cjs` and `README.md`. No bob-cli source changes are
needed.

## 1. One undo step for inline single-task roll and decay writes

**Problem.** `setInlineBulletPropertyValues` in `plugins/bob-navigation-hotkeys/main.js`
writes in two separate edits:

1. It writes the task line with `replaceEditorLine`, or for the same-file Pomodoro prune
   it uses `applyEditorContentTransaction`.
2. Afterwards it plans the Schedule Log entry with `planScheduleLogEntry` against
   `cm.getValue()` and inserts it with a second edit, `insertEditorLine`.

In Obsidian these are two CodeMirror history events, because the changes are not
adjacent. One Ctrl+Z therefore removes the `🎲 P2 roll …` entry but keeps the new date
and priority. That erases the step the derived roll streak depends on. The Ctrl+Enter
roll (`applyRecommendedRollWrite` → `setBulletPropertyValue`) and the inline decay
(`applyRecommendedDecayWrite` → `setBulletPriorityValue` →
`setInlineBulletPropertyValues`) both go through this writer. The pre-existing pinned
roll row and priority row use it too, and they benefit from the fix.

**Fix.** Make the task-line edit, any folded same-file Pomodoro prune, and the Schedule
Log entry land as **one** editor change set:

- Plan the Schedule Log entry in memory **before** touching the editor:
  - Build the postimage lines: the task line replaced by `nextLine`, or
    `foldedDailyContent` when the prune is folded.
  - Run `planScheduleLogEntry(postimage, effectiveCursorLine, options.scheduleLog)`.
- **No folded prune** (the common case). The line count is unchanged before the insert,
  so `scheduleLogPlan.insertLine` is valid in the original document's coordinates.
  Dispatch one `cm.transaction({ changes: [...], selection })` that holds:
  - the task-line replacement;
  - the insertion of `scheduleLogPlan.lineText + "\n"` at `{ line: insertLine, ch: 0 }`.
    When `insertLine` is past the last line, insert `"\n" + lineText` at the end of the
    last line instead, mirroring `insertEditorLine`.

  Add a small helper next to `applyEditorLineChanges` / `insertEditorLine` for this.

- **Folded prune.** Splice the log line(s) into the in-memory folded content and apply
  the result with the existing single `applyEditorContentTransaction`.
- **Editors without `transaction`.** Keep the current sequential `replaceRange`
  fallback.
- **Keep everything else unchanged:**
  - `scheduleLogOutcome` comes from the same plan, through `getScheduleLogWriteOutcome`.
    An invalid plan still writes the property and reports `guard-failed`.
  - `shouldWriteAutomaticScheduleLog` skipping must still behave the same.
  - Keep the cursor placement, the notice text, the deferred (other-file) prune, and the
    recovery and Blocked handling.
- Run the existing suites. Tests that pin the old two-edit sequence through
  `RecordingFallbackEditor.replaceCalls` or `TransactionEditor.transactions` should only
  change where the new single-transaction shape is the intended behavior.

## 2. Decay notice text header copy

The plan specifies the decay notice's text header as
`priority → P3 (low) · decayed from P2`. `buildPriorityNoticeModel` (the `textHeader`
for `rollKind === "decay"`) instead interpolates the pill `P2 → P3`, which produces
`priority → P2 → P3 (low) · decayed from P2`.

- Use the target level's label there instead of `pill`. The pill itself stays `P2 → P3`,
  as the plan requires.
- Update the assertion in `scripts/test-navigation-roll-decay.cjs` (currently
  `/priority → P2 → P3 \(low\) · decayed from P2/`) to the plan's exact text.

## 3. Plan-required end-to-end Ctrl+Enter tests

Phases picker-single and picker-counted shipped only pure-helper tests. Add modal-driven
tests to `scripts/test-navigation-roll-decay.cjs`. Drive `BulletPropertyPickerModal`
through `handleKeydown` with an editor that records undo groups.

**Harness.**

- Copy `TransactionEditor` (it counts `undoGroups`) and the
  `createBulletPropertyPickerHarness` shape from `scripts/test-navigation-hotkeys.cjs`.
- Open with
  `plugin.openBulletPropertyPicker(editor, { config, baseDate: new Date(2026, 8, 30), random: () => 0 })`.
  Use `{ countExplicit: true, additionalTaskCount: N }` for counted sessions.
- Select the `scheduled` property row. Send
  `{ key: "Enter", ctrlKey: true, preventDefault() {}, stopPropagation() {} }`.
- Writes are async (`void this.applyRecommendedRoll()`), so flush timers before
  asserting.
- `^prj` fixtures need `type: [[project]]` frontmatter. This file's obsidian stub throws
  in `parseYaml`, so replace it with the tiny `parseTestYaml` from
  `scripts/test-navigation-hotkeys.cjs`.

**Expected values** with `random: () => 0` and a base date of 2026-09-30. The land
agent's smoke run observed these:

- **P2 roll, no log.**
  - Task line: `- [?] #task A [priority:: medium] [scheduled:: 2026-10-08] ^a`.
  - Log entry: `\t\t- *2026-10-01 → 2026-10-08* — 🎲 P2 roll · in **8** (8–30) days`.
- **P2 with a `🎲 P2 roll` entry.**
  - Task line: `[priority:: low] [scheduled:: 2026-10-31]`.
  - Log entry: `🎲 P2 → P3 decay · in **31** (31–90) days`.
- **P4 with a `🎲 P4 roll` entry.**
  - Task line: `- [-] … [cancelled:: 2026-09-30]`.
  - First-child `❌ **CANCEL LOG**` with
    `*2026-09-30* — 🍂 decayed past P4 after 1 roll`.

**Cover, for single tasks:**

- Roll, decay and cancel from stage one. Each one closes the picker and writes the exact
  lines above.
- Each roll, decay and cancel write has `undoGroups === 1`. Roll and decay need step 1.
- Ctrl+Enter in stage two with a date preset row highlighted, not the pinned row. It
  writes the recommended roll. The pinned `🎲 P2 roll` row's value equals the
  recommendation's date.
- ↵ on `scheduled` still opens the date list. Ctrl+Enter on the priority row behaves
  like ↵.
- A P0 task (no priority field) and a closed `[x]` task have no preview, and Ctrl+Enter
  behaves like ↵.
- A recurring (`🔁`) P4 task past its limit shows the recurring Notice, leaves the
  content unchanged, and keeps the picker open.
- A stale task refuses:
  - After opening, edit the editor content to add a `🎲 P2 roll` entry.
  - Ctrl+Enter shows `Task changed while the picker was open; nothing was written` and
    writes nothing.
  - The picker stays open, and the stage-one preview is now the decay.
- Ctrl+R in stage one re-rolls the recommendation's date. Use a `random` that returns
  different values per call.
- Ctrl+R in stage two keeps the pinned row and the recommendation on the same date.
- `^prj` roll and `^prj` decay:
  - frontmatter `scheduled:` is updated;
  - the inline `[priority:: low]` is set for the decay;
  - the log entry is written;
  - `undoGroups === 1`.

**Cover, for counted sessions:**

- A mixed batch with A roll, B decay, C cancel, and D closed `[x]`, which is skipped. It
  writes the exact lines with `undoGroups === 1`. The notice reads
  `Rolled 3 tasks; 1 rolled; 1 decayed; 1 cancelled; …; 1 task skipped`.
- A target line edited after open refuses the batch and writes nothing.
- A recurring cancel target refuses the whole batch and writes nothing.

## 4. Ship the plugin

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.47.0` to `1.48.0`.
- Update the version cell in that plugin's `README.md` Plugins-table row to match.
- In `bob-plugins`, run `npm test` and `npm run validate`. Both must pass.
- Deploy from the opened checkout, dry run first:
  - `bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run`
  - `bob plugins sync -p bob-navigation-hotkeys -r "$PWD"`

  Do not use `--force` unless the vault file is verified byte-identical to the pre-edit
  baseline.

- The `bob-plugins` changes are a repository obligation of this turn. Commit them
  through the normal final declaration.

## 5. Close out epic bob-cli-34 (final step)

Do this in the same turn as the code. Do not wait for, or order it after, this work's
own commit SHA, push or CI.

1. Run `sase bead epic-symbols bob-cli-34`. It reported none at planning time. For any
   entry now listed, resolve it (wire it up, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy). Re-key a Justfile line only to a
   still-open later bead that needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-34 --note "<verification>"
   ```

   The note should summarize:
   - phases .1–.5 verified against code, tests and docs;
   - integration review found no conflicts with the concurrent commits (bob-plugins
     0b6c847 project promotion and eff561e bob-cli-31 freshness landing; bob-cli 3dd833f
     park-links and 6710c74 bob-cli-31 test fixes), and no other plugin or bob-cli code
     classifies Schedule Log reasons;
   - the follow-up triage (bob-cli-2e +1; nothing else filed);
   - the three fixes in this tale, with the test and validate counts.

   Never use `--force` merely to make the close succeed. If the close is refused for
   leftover epic symbols, finish that cleanup and close again.

3. Run `just symvision` in bob-cli if the recipe exists. bob-cli's `justfile` had no
   `symvision` recipe at planning time; if it is still absent, say so in your final
   response.
4. Set `status: done` in the frontmatter of the epic's plan file, at the path from the
   `PLAN` line of `sase bead read bob-cli-34 -r "<why>"`. That store
   (`sase/repos/plans`) is a git repo, so commit the edit in place:
   - `git -C <plans store> add <file>`
   - `git -C <plans store> commit -m "chore(sdd): mark bob-cli-34 done"`

   Then run any `sase bead` command and confirm there is no pull-rebase error.

5. bob-cli-34 has no `parent_bead`; there is no parent to close.

## Acceptance

- One Ctrl+Z fully reverts an inline single-task Ctrl+Enter roll or decay: date,
  priority, Blocked mark, stamp and Schedule Log entry together. The tests assert
  `undoGroups === 1`.
- The decay notice text header reads `priority → P3 (low) · decayed from P2`.
- The new end-to-end tests cover the step 3 list. `npm test` and `npm run validate`
  pass.
- The plugin is deployed at 1.48.0.
- bob-cli-34 is closed with a verification note, and its plan file is `status: done` and
  committed.
