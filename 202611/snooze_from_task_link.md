---
tier: tale
title: Snooze a task from its Task Link via Task Card z
goal:
  Task Card z on a single Task Link opens the same Snooze Card that Alt+Z opens on that
  link, instead of refusing with "That row is not an open task".
size: medium
proposed_by: bbugyi200.athena.1h1
proposed_with: claude/opus
create_time: 2026-11-27 09:12:18
status: wip
---

# Plan: Snooze a task from its Task Link via Task Card `z`

## Problem

Bryan opens the **Task Card on a live Task Link** in today's ledger and presses `z`
(Snooze). The card refuses with `That row is not an open task`. The screencast
`~/tmp/screencasts/20261127_090251.mp4` shows this. The cursor is on
`[[bob#^one-equals-not-two]]` under an open `BUILD` entry. The Task Card was opened on
that link: its last row is "Cancel linked task…". Bryan then pressed `z`. The vault task
tracking this request is `bob#^snooze-from-task-link`.

The target task is snooze-eligible. It is `- [*] #task … ^one-equals-not-two` in
`bob.md`, and both of its `dependsOn` prerequisites are `[x]`. `Alt+Z` on the same link
line already works.

This is a defect in the Snooze Card phase of the snoozed-tasks epic (`bob-cli-q5.6`,
child epic `bob-cli-q5.6.1`, plan `plan:202611/snooze_card.md`). That phase promised
that Task Card `z` "opens the same Snooze Card as `Alt+Z` for the card's single task".
Its acceptance run only exercised Task Card `z` from a task **definition**, never from a
Task Link.

## Root cause (reproduced)

All paths are in bob-plugins, under `plugins/bob-navigation-hotkeys/src/`.

1. `745-snooze-card-commands.js`, `openSnoozeCardFromTaskCard(taskCard)`, takes the Task
   Card's raw session targets and passes them to `openSnoozeCardCommon` as
   `explicitTargets`.
   - For a Task Link card these are `linkSession.resolved` entries, built in
     `500-plugin-transclusion-and-link-picker.js` `resolveLinkPickerTargets`. Their
     shape is `{path, file, content, line, rawLine, blockId, displayText}`. There is no
     `status`, `block`, `route`, or `is_task`.
   - For a counted definition card they are `taskSession.targets`, built by
     `220-property-targets.js` `discoverCountedObsidianTaskTargets`. Their shape is
     `{line, rawLine}`.
   - The `model.session.targetsForSnooze` branch is dead: nothing sets it.
2. `683-snooze-runtime-resolve.js` `resolveSnoozeCursorTarget` adopts a single explicit
   target directly as kind `definition` through `snoozeRuntimeAdoptTask`. Then
   `snoozeRuntimeEligibility` reads the missing `status` as `""`. That is not in
   `[" ", "*", "/"]`, so it returns `not_task`, which produces "That row is not an open
   task".
3. A one-off node repro through `scripts/navigation-hotkeys-harness.cjs` confirms both
   sides:
   - The link-session target shape returns
     `{ok:false, refusal:"not_task", message:"That row is not an open task"}`.
   - The same link line through the cursor path (Alt+Z) returns
     `{ok:true, kind:"live-link", identity:{route:"bob.md", block:"one-equals-not-two"}, sourcePomodoro:"BUILD"}`.

Even with a valid status, the explicit-target path would still be wrong for a link. It
would label the context `definition` and drop the link context: the source Pomodoro
behind the footer's `leaves BUILD`, and woken versus history link handling.

## Fix: Task Card `z` resolves exactly like Alt+Z at the card's single source line

The Snooze runtime already has one cursor resolver for every context: definition,
live-link, woken-link, history-link, and snoozed-row. The Task Card knows which source
line it was opened for. So the bridge should hand the resolver that source line, not
pre-resolved Task Card targets.

### 1. Task Card source projection (745, or a new small fragment if 745 would exceed 1000 lines)

Add a pure helper, for example `snoozeTaskCardSource(taskCard)`, that returns either
`{ok:true, editor, sourcePath, cursorLine, expectedLine, expectedIdentity}` or a snooze
refusal:

- **Link session.** Use `taskCard.linkSession`.
  - If it holds more than one target, return `multi_task`.
  - If it holds exactly one valid target, `cursorLine = linkSession.targets[0].line`.
    These discovery targets carry the **source** link line, not the target note's line.
  - `expectedLine = linkSession.targets[0].rawLine`.
  - `expectedIdentity` is taken from `linkSession.resolved[0]`, as
    `{route: path, block: blockId}`.
  - If the session is still resolving or invalid, return `unavailable`.
- **Counted task session.** Use `taskCard.taskSession` with `explicit`.
  - More than one target: return `multi_task`.
  - Exactly one target: `cursorLine = targets[0].line` and `expectedLine = rawLine`.
- **Otherwise**, a plain single-definition card: use `taskCard.cursor.line` and
  `taskCard.lineText`.
- In every case `sourcePath = taskCard.filePath` and `editor = taskCard.editor`.

Rewrite `openSnoozeCardFromTaskCard` to use this projection. Remove the raw-target
extraction, including the dead `targetsForSnooze` branch.

Then let `openSnoozeCardCommon` accept a cursor override (`{sourcePath, cursorLine}`)
and pass `explicitTargets: null`. Resolution then goes through the same
`resolveSnoozeAtCursor` → `resolveSnoozeCursorTarget` → `readSnoozeTaskForIdentity` path
Alt+Z uses. Context kind, eligibility refusals, `sourcePomodoro`, and card facts are
then identical by construction.

Before opening, check for staleness:

- The active markdown view must still be the Task Card's editor and file.
- The editor's current text at `cursorLine` must equal `expectedLine`.
- When `expectedIdentity` is known, the resolved identity must equal it.

Any mismatch refuses with the existing `stale` message ("The note changed while the card
was open; try again") and writes nothing.

Keep everything else as it is:

- The wrapper's existing guards: repeat key, `taskCardDispatching`, `linkResolving`, and
  closing the Task Card first.
- `plugin.reviewAnchor` handling, unchanged.
- The `needs_naming` / block-ID prompt fallback for definitions.

The Task Card row in `310-task-card-previews.js` already enables `snooze` only when
`targetCount === 1`. Keep that.

### 2. Defensive hardening of the pure resolver (683)

`snoozeRuntimeEligibility` must not call a record without task facts "not an open task".
When `status` is absent or empty and `is_task` is not explicitly `false`, return
`missing_task`.

Records from `readSnoozeTaskForIdentity` and the `tasks` snapshots always carry a status
character, so no current caller changes behavior. This only stops a future shape
mismatch from producing misleading copy.

Keep the `explicitTargets` API for callers that pass full task records, which the
existing tests use.

### Governing decisions

- `decisions:consumption-keeps-notes-and-recorded-time`: snoozing from the link moves
  the live link and its authored notes as one subtree, through the existing planner. A
  running timed entry is never deleted, and closed history is untouched. This fix reuses
  that planner unchanged and adds no new write path.
- `decisions:today-is-read-from-the-ledger`: once snoozed, the link has left
  `## Pomodoros`, so the task is no longer Today. The planner already handles this;
  nothing is stored on the task line beyond the existing Ready→Next promotion rule.
- `decisions:answering-advances-the-walk`: a successful snooze committed from a `]s`
  landing still goes through the existing review capture/continue once. Refusals and
  cancellation stay put. The bridge must keep passing the same `reviewAnchor`.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the Snooze Card's close keys are unchanged.

## Tests

Add every new test runner to `package.json` `test:unit`. Extending the existing runners
listed below is preferred.

- **`scripts/test-navigation-snooze-runtime-resolve.cjs`**
  - A raw link-session-shaped explicit target (no status) is no longer reported as
    `not_task`.
  - `snoozeRuntimeEligibility` truth table, including the empty-status → `missing_task`
    case.
- **`scripts/test-navigation-snooze-card.cjs`**, or
  `scripts/test-navigation-task-card-view.cjs` if the stubs live there. Cover Task Card
  `z` with a stubbed plugin and editor:
  1. A link-session card with one resolved live link under an open entry opens the card
     with `kind:"live-link"`, the target identity, and `sourcePomodoro` equal to the
     entry name.
  2. A woken link (`⏰ woke HH:MM`) resolves as `woken-link`. A struck history link
     resolves as `history-link`.
  3. A counted definition session with one target resolves as `definition`.
  4. The plain definition card is unchanged.
  5. A link or counted session with two targets refuses `multi_task`.
  6. Each staleness case refuses `stale` with zero editor writes: a changed source line,
     a different active file, or a resolved identity that differs from
     `linkSession.resolved[0]`.
  7. An Event Task link, a Blocked target, and a done target show their existing
     teaching refusals, not "not an open task".
  8. A repeat `z` still dispatches once.
- **`scripts/obsidian-snooze-card-e2e.cjs`** (real Obsidian, isolated fixture vault).
  Add a "Task Card `z` from a live Task Link" case next to the existing
  `task-card-z-opens-same-card` check:
  1. Open the daily fixture and put the cursor on a dedicated live link under an open
     entry.
  2. Open the Task Card with Ctrl+Shift+P. Use the same command fallback the existing
     case uses.
  3. Press `z`. Assert that the Snooze Card opens with the task title and the footer
     names the entry it leaves.
  4. Press Enter. Assert that the link and its nested notes moved exactly once into
     `## Snoozed` as `- **HHMM** [[…#^id]]`.
  5. Assert that the source entry kept its other links, and that one daily-editor Ctrl+Z
     restores the preimage.

  Keep `scripts/test-obsidian-snooze-card-e2e.cjs` (the offline self-test) passing. Run
  it with `xvfb-run -a node scripts/obsidian-snooze-card-e2e.cjs --out <dir>`.

## Docs

First read `docs.md` with `/sase_memory_read`.

- In bob-plugins `docs/task-card.md` § Snooze, add one sentence:
  - On a single dedicated Task Link (live, woken, or struck history), `z` resolves that
    link exactly as `Alt+Z` does.
  - A live link and its notes move into today's Snoozed row.
  - The footer names the entry it leaves.
- Mirror the same sentence in `docs/bob-navigation-hotkeys.md` if it has a Task Card `z`
  entry.
- bob-cli `docs/snooze.md` already says Task Card `z` "opens the same card". Leave
  bob-cli untouched unless docs-check requires otherwise.
- Run `npm run docs:check` with `BOB_CLI_DIR` pointing at the bob-cli checkout.

## Work location, gates, and deploy

- Open bob-plugins with `/sase_repo` and read its `AGENTS.md` / `CLAUDE.md`.
- Edit only `src/` fragments: keep each at or under 1000 lines and update
  `src/fragments.json` if you add a fragment. Then run `npm run build`; never hand-edit
  `main.js`.
- Gates:
  - focused runners above;
  - `npm run build:check`;
  - `npm run validate`;
  - `npm test`;
  - `npm run docs:check` (with `BOB_CLI_DIR`);
  - the real-Obsidian e2e above.

  Known clean-base `npm test` noise: the `/tmp` ENOSPC e2e self-tests (`bob-cli-qt`) and
  timing thresholds (`bob-cli-io`). If either reproduces on a clean base, record the
  evidence and do not fix it here. If the e2e cannot start (no Xvfb or Obsidian binary),
  say so plainly in the final report, with the error. Do not claim it passed.

- Deploy with `bob plugins sync`. If sync refuses to overwrite another checkout's
  deploy, follow the bob-plugins CLAUDE.md procedure and `docs/develop/deployment.md`
  exactly. Never pass `-F` on your own.
- Commit in bob-plugins as
  `fix(snooze): open the Snooze Card from a Task Link's Task Card`. There are no bob-cli
  code changes, so the release-train rules do not apply. Never edit versions or
  `CHANGELOG.md` anywhere.
- Add a note on `bob-cli-q5.6` with the fix commit and the e2e result, so its pending
  closeout covers Task Card `z` from a Task Link. Do not close or change the status of
  any `bob-cli-q5*` bead, and do not edit the vault task `bob#^snooze-from-task-link`.
  Bryan closes it.

## Acceptance

1. In the real vault, with Bryan's exact case (Task Card on
   `[[bob#^one-equals-not-two]]` under the open BUILD entry, then `z`), the Snooze Card
   opens with `⏰ Snooze “Change the == keymap to =!”` and a `leaves BUILD …` footer.
2. Enter moves the link into `## Snoozed`. The task stays `[*]` and is not re-stamped
   (rule 4: an already-Next task writes only the daily note).
3. Alt+Z on the same link and Task Card `z` produce identical resolutions and card
   facts.
4. Counted or multi-target cards still refuse with single-task copy. Event, Blocked,
   done, and cancelled targets keep their teaching refusals.
