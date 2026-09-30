---
tier: epic
title: Cancel tasks with an optional reason from the Ctrl+Shift+P picker
goal: "In Obsidian, Ctrl+Shift+P on a #task line (bare or counted) or on a dedicated
  Task Link offers a pinned Cancel row. It asks for an optional reason, then closes the
  task(s) as Cancelled `[-]` with a `[cancelled:: YYYY-MM-DD]` stamp and records the
  reason under a managed `❌ **CANCEL LOG**` child. It also removes the tasks' links
  from today's open Pomodoros, unblocks their dependents right away, and confirms with a
  rich notice card.

  "
phases:
  - id: tsc-recovery-api
    title:
      "Task Status Cycler: versioned dependent-recovery API and cancelled-link guard"
    depends_on: []
    size: small
    description: "tsc-recovery-api: expose `api.recoverBlockedDependents` (version 1)
      from task-status-cycler. Keep a dependent Blocked while it has a strictly future
      `scheduled` date. When Ctrl+Enter lands on a Task Link to a Cancelled task, show a
      notice instead of completing the owning Pomodoro. Bump the version, update the
      README, add tests, and sync to the vault.

      "
  - id: cancel-planner
    title: "Navigation Hotkeys: Cancel Log grammar and pure cancel planner"
    depends_on: []
    size: medium
    description: "cancel-planner: add the `❌ **CANCEL LOG**` marker/entry grammar, a
      `cancel` managed-log kind (so project conversion carries the log), and a pure
      batch planner. The planner sets `[-]`, upserts `[cancelled::]`, writes the Cancel
      Log first-child/prepend/fallback entry, refuses recurring tasks, and returns the
      cancelled identities. Export the helpers and unit-test them. No UI yet.

      "
  - id: cancel-picker
    title:
      "Navigation Hotkeys: Cancel row, reason stage, guarded writes, and notice card"
    depends_on:
      - tsc-recovery-api
      - cancel-planner
    size: medium
    description: "cancel-picker: wire the pinned Cancel row and the live-preview reason
      stage into BulletPropertyPickerModal for single, counted, and Task Link sessions.
      Commit through the existing guarded editor and cross-note write paths, including
      the today's-Pomodoro prune. Call the TSC recovery API, show the Cancelled notice
      card, add CSS, bump the version, update the README, add tests, and sync.

      "
  - id: cancel-docs
    title: bob-cli documentation for the cancel gesture and the Cancel Log
    depends_on:
      - cancel-picker
    size: small
    description:
      "cancel-docs: document the gesture, the written shape, and its side effects in
      docs/projects.md. Add a Cancel Log row to the README glossary and update
      docs/task-status-hooks.md for immediate cancel pruning, the recovery API, and both
      new guards. Add a parity comment in bob-cli's managed-log parser and propose a
      glossary follow-up."
proposed_by: bbugyi200.athena.0ug
create_time: 2026-09-30 13:42:47
status: wip
---

- **PROMPT:**
  [prompts/202609/cancel_task_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/cancel_task_picker.md)

# Plan: Cancel tasks with an optional reason from the Ctrl+Shift+P picker

## Context

- **Which keymap.** The vault has no plain Ctrl+P binding. The task-property keymap is
  **Ctrl+Shift+P**: `bob-navigation-hotkeys:set-bullet-property`, bound in
  `~/bob/.obsidian/hotkeys.json` and implemented by `BulletPropertyPickerModal` in
  `plugins/bob-navigation-hotkeys/main.js` of the linked `bob-plugins` repo. This
  feature extends that picker.
- **Target modes the picker already has.** "The selected task, or the corresponding task
  when a Task Link is selected" maps onto the picker's three existing modes, and cancel
  must support all of them:
  1. **Single.** The `#task` line under the cursor.
  2. **Counted.** `N<Ctrl+Shift+P>` builds a `taskSession` of the current task plus the
     next N tasks (`discoverCountedObsidianTaskTargets`).
  3. **Link session.** On a dedicated Task Link bullet, `openLinkPicker` builds a
     `linkSession` whose `resolved` targets are the linked, _open_ tasks in their own
     notes (`discoverLinkPickerTargets`, `resolveLinkPickerTargets`,
     `findUniqueLinkPickerTargetLine`).

  The picker already pins a `#now` toggle row (`describeNowToggleRow`,
  `renderNowToggleItem`, `applyNowToggleFromPicker`). It also has a free-text reason
  stage for `scheduled` (`showScheduleReasonStage`, `renderScheduleReasonPreviewItem`,
  `getBulletPropertyScheduleReasonHints`) and a rich notice card
  (`buildPriorityNoticeModel`, `renderPriorityNoticeFragment`, `showPriorityNotice`, CSS
  `.bob-nh-notice*`). The cancel row, stage, and notice copy these patterns.

- **How cancelling works today.**
  - The only ways to cancel are Task Status Cycler's `<option+[>` cycle and Obsidian
    Tasks' own status command. Nothing records _why_ a task was cancelled.
  - Bryan writes reasons by hand as the task's **first** child bullet, e.g.
    `- OBSOLETE: See [[sase_art_links_panel#^agents-sub-tab]]!` in `sase_art.md`.
  - Obsidian Tasks is configured with `taskFormat: dataview` and
    `setCancelledDate: true`, so its own cancels write `[cancelled:: YYYY-MM-DD]`. The
    ~400 existing `[-] #task` lines carry that stamp.
- **Rules this design must respect.** Read these with `sase memory read decisions:<key>`
  before phases `tsc-recovery-api` and `cancel-picker`:
  - `decisions:task-status-is-derived`: keymaps may apply the hooks' rules immediately
    for feedback, and `bob task-status-hooks` has the final word.
  - `decisions:now-tag-is-user-owned`: no automation adds or strips `#now`.
- **What `bob task-status-hooks` already does with cancelled tasks.** See
  `docs/task-status-hooks.md`, "Canceled Task References" and the dependency section.
  - It deletes list items under today's _open_ Pomodoros whose links resolve only to
    CANCELLED tasks.
  - CANCELLED dependencies never block.
  - Terminal tasks are never re-derived, so a cancelled task that keeps a future
    `scheduled` date is harmless.
  - The NOW query uses `not done`, and Cancelled counts as done. A cancelled `#now` task
    therefore drops out of NOW by itself.
- **Two sharp edges found while designing.**
  1. **Ctrl+Enter on a Task Link to a Cancelled task completes the Pomodoro.** Task
     Status Cycler's handler resolves targets with `isOpenDoneTaskStatus`, which only
     accepts `" "`, `*`, `/` and `x`. A plain Task Link to a Cancelled task inside an
     open Pomodoro therefore fails to resolve, and `handleVimTaskToggleOpenDone` falls
     through to **completing the whole owning Pomodoro**.
  2. **Ctrl+Enter recovery ignores future schedules.**
     `buildBlockedDependentRecoveryPlan` reopens a Blocked dependent to Ready even when
     that dependent still has a strictly future `scheduled` date. The hooks then
     re-block it.

  Phase `tsc-recovery-api` fixes both, because cancel shares this recovery and produces
  Cancelled Task Link targets.

## Design contract (all phases implement exactly this)

### The gesture

1. **Pinned Cancel row.** Ctrl+Shift+P shows the property list as today, plus a pinned
   **Cancel** row as the **last** row. It gets danger styling and a hairline separator
   above it.
2. **Choosing the row.** Choose it by arrowing to it or by typing a filter such as
   `can`, `drop` or `obsolete`, then press ↵. This opens the **reason stage**; nothing
   is written yet.
3. **Confirming.** Type an optional reason and press ↵. The task(s) are cancelled, the
   modal closes, and one notice card confirms what happened.
4. **Backing out.** Esc at either stage writes nothing.

**When the row appears.**

- It appears only when at least one target is an open `#task` (`" "`, `*`, `/` or `?`).
- It is hidden on closed tasks, on plain bullets, and when the cursor is not on a task
  or Task Link.
- In a counted session, closed targets (`x`, `X`, `-`) are skipped and reported. The row
  is hidden if every target is closed.
- Link sessions are open by construction.
- `^prj` lifecycle tasks are allowed. `bob projects sync` already maps `[-]` to
  `status: canceled`.

**Recurring tasks are refused.** A target counts as recurring when its line has
`[repeat:: …]`, `(repeat:: …)` or `🔁`.

- On such a target the row still shows, with the detail `recurring · use Obsidian Tasks`
  and a muted pill.
- Choosing it keeps the property list and shows the Notice
  `Recurring tasks are cancelled with Obsidian Tasks so the next occurrence is handled; no tasks were updated`.
- In counted and link sessions, one recurring target refuses the whole batch.

No new default hotkey and no separate command are added.

### What is written on each cancelled task

1. The checkbox becomes `[-]`.
2. `[cancelled:: YYYY-MM-DD]` is upserted with `upsertBulletProperty`, before the
   trailing `^block-id`, replacing any existing `cancelled` field.
   - The date is the picker's local base date (`valueBaseDate`).
   - This matches what Obsidian Tasks itself writes, so Tasks queries, the `done`
     filters, `### Done & Canceled` grouping and `move-done-tasks` treat it exactly like
     a Tasks-command cancel.
3. The reason is recorded in the **Cancel Log** (grammar below).
4. **Nothing else on the line changes.** `scheduled`, `dependsOn`, `id`, `priority`,
   `created`, `#hide` and `#now` are untouched. `#now` is user-owned, and cancelled
   tasks leave NOW by themselves.

Example: a task with notes and a Schedule Log, cancelled on 2026-09-30 with a reason.

```markdown
- [-] #task Add new `Agents` sub-tab to `Artifacts` tab! [priority:: high] [created::
  2026-08-14] [cancelled:: 2026-09-30] ^agents-tab
  - ❌ **CANCEL LOG**
    - _2026-09-30_ — Superseded by [[sase_art_links_panel#^agents-sub-tab]]
  - The new agents sub-tab on the artifacts tab should show all agents filtered by
    project…
  - 🗓️ **SCHEDULE LOG**
    - _2026-09-02_ — 🎲 P0 → P1 · in **5** (2–7) days
```

### Cancel Log grammar

**Marker.** `❌ **CANCEL LOG**`.

- `❌` is U+274C. It is written without a variation selector; the parser also accepts an
  optional U+FE0F after it.
- Build the regex with the existing `buildManagedTaskLogParentRe`. That accepts the
  emoji-less `**CANCEL LOG**` and an optional colon, and rejects a mismatched emoji such
  as `🗓️ **CANCEL LOG**`. There are no legacy labels.

**Entry.** `*YYYY-MM-DD* — <reason>`.

- Use `SCHEDULE_LOG_ENTRY_EMPHASIS` (`*`) and `SCHEDULE_LOG_SEPARATOR` (`—`).
- The date is the cancel date.
- Entries are newest first.

**Placement.**

- A new log is inserted as the task's **first direct child**, directly below the task
  line. This deliberately differs from Schedule and Work Logs, which are appended last:
  for a closed task the reason is its verdict and belongs on top, where Bryan already
  writes such reasons by hand.
- If the task already has a direct-child Cancel Log (it was cancelled, reopened, then
  cancelled again), the new entry goes directly under that marker, wherever the marker
  sits. The log is never moved and never duplicated.
- A marker nested under a grandchild does not count.

**Indentation.**

- The marker uses the task's existing direct-child indent (`getDependencyChildIndent`),
  otherwise the task's indent plus one tab.
- An entry uses an existing entry's indent, otherwise the marker's indent plus one tab.
- The list marker is `-`, or the existing marker's own character when prepending.
- CRLF line endings are preserved.

**Empty reason.**

- No log is created.
- If the task already keeps a Cancel Log, prepend `*YYYY-MM-DD* — 🤷 no reason given`
  (`SCHEDULE_LOG_SKIPPED_REASON_TEXT`). This is the same "a kept history has no gaps"
  rule the Schedule Log uses.

**Reason text.**

- Normalize with `normalizeScheduleReasonText`: collapse whitespace and keep wikilinks,
  backticks and markdown verbatim.
- `::` produces a warning but is not blocked.

**Recognition.**

- `parseManagedTaskLogParentBullet` gains the kind `cancel`
  (`MANAGED_TASK_LOG_KIND_CANCEL`). As a result, `Ctrl+Shift+Alt+N` project creation
  carries a Cancel Log under `^prj` like the other logs, instead of seeding a bogus
  `- [ ] #task ❌ **CANCEL LOG**`. `getProjectFromTaskNoticeText` then reports
  `cancel log moved`.
- Check that `convertProjectNoteToTask` round-trips the log. Keep
  `findScheduleLogParent` and the Work Log helpers schedule-only and work-only.

**bob-cli needs no code change.**

- `capture_task_sections` skips emoji-led bullets.
- Sub-bullet capture anchors only on Schedule and Work Logs, so new captured notes land
  below a first-child Cancel Log.
- `capture_project_note` only seeds brand-new tasks.

The NAV parity comment above `buildManagedTaskLogParentRe` must say that the Cancel Log
is plugin-only, and why.

### Side effects (applied immediately, for feedback)

**Today's open Pomodoros.**

- Every live link to a cancelled task that has a block ID is removed with
  `planDeferredPomodoroLinkCleanup`, exactly as future-scheduling does:
  - A dedicated link bullet is removed with its subtree.
  - Otherwise only the link token is removed.
  - Struck links, closed Pomodoros and other days' notes are untouched.
- This mirrors the hooks' Canceled Task References rule.
- When the picker was opened on a Task Link in today's open Pomodoro, that bullet itself
  is removed.

**Dependents.** After the writes land, call
`app.plugins.plugins["task-status-cycler"].api.recoverBlockedDependents(identities, { activePath, editor })`.

- Only when `api.version >= 1`.
- Each identity is `{ path, blockId, taskId }`, where `taskId` is the task's `[id::]`
  value.
- A Blocked dependent with no remaining open dependency and no future `scheduled` date
  becomes Ready. This is the same routine Ctrl+Enter uses.
- If the API is missing, or it throws or rejects, skip it silently. The notice then
  omits the chip, and the hooks recover the dependent later.
- No reference strike-through: the hooks strike only DONE references.

**NOW and plan budget.**

- Read `bob-ledger-tools` `api.nowBudget()` (`readNowBudgetValue`) _before_ the write.
  When any cancelled target carried `#now`, the notice shows `NOW (count − k)/cap`.
- When links were pruned and `api.planBudget` exists, append the plan chip computed with
  `planBudget({ content: <post-prune daily note text> })`, the same way
  block-id-prompt's unlink notices do.
- Both chips are omitted when the API is unavailable.

### Atomicity and write paths

Nothing is written until the reason stage is confirmed. Every refusal ends with
`…; no tasks were updated`.

**Single and counted sessions (active editor).**

- Re-validate against the live buffer: `getCountedTaskWriteContext`, and for a single
  task an equivalent line-and-status check.
- Plan with the pure planner and apply **one** `applyEditorContentTransaction`. That is
  one undo step, with the cursor clamped onto the post-image line.
- **Today's daily note is the active note** (`pomodoroSnapshot.sameFile`): fold the
  prune into the same transaction and shift the cursor for removed lines.
- **The daily note is a different file:** write the prune afterwards through
  `readDeferredPomodoroSnapshot` and `writeDeferredPomodoroCleanup`, following the
  pattern in `setCountedBulletPropertyValue`.
- A failed prune is reported as a warning chip `Pomodoro links not removed`. It is never
  rolled back.

**Link sessions.**

- Group targets by note (`groupLinkPickerTargetsByNote`) and plan each note.
- Commit with the `commitLinkPickerPlans` guarantees:
  - Re-verify every preimage before the first write.
  - Write the targets first, then the daily note.
  - Roll back target writes if one fails.
  - Fold the prune into the daily note's own write when the daily note is a target note.
  - Report a failed prune; never roll it back.
- Extract the shared preimage, write, rollback and prune core of `commitLinkPickerPlans`
  into a helper that both scheduled/priority and cancel use. Existing behavior and tests
  must stay byte-for-byte unchanged.

**Ordering.** The modal closes as soon as the writes land. Dependent recovery then runs
(TSC serializes it on its own mutation queue), and exactly one notice card is shown
after it settles.

### Look and feel

**Cancel row (property stage).**

- Item: `{ kind: "cancel-task", property: { name: "cancel" }, … }`. It needs the
  `property` object so `selectPropertyName` and the filters keep working.
  `showValueStage` and `deleteSelectedProperty` must ignore this kind, as they do for
  `now-toggle`.
- Class `bob-cnp-cancel-row` with a top hairline separator and `var(--text-error)` on
  the icon and pill. When selected, it gets a red-tinted left border like
  `.bob-cnp-schedule-reason-row.is-warning.is-selected`.
- Icon: Lucide `ban`.
- Title: `Cancel task`, `Cancel N tasks`, `Cancel linked task` or
  `Cancel N linked tasks`.
- Detail line:
  - single: `Next → Cancelled · asks why`, with status names Ready (`" "`), Next (`*`),
    In Progress (`/`) and Blocked (`?`);
  - counted: `N tasks → Cancelled · 1 already closed`;
  - link: `↗ <note> · <task> → Cancelled`, reusing `getLinkPickerSessionSubtitle`;
  - recurring: `recurring · use Obsidian Tasks`.
- Pill: `cancel`, muted when recurring.
- Filter text:
  `cancel cancelled canceled abandon drop obsolete wontfix won't do close ❌` plus the
  detail text.

**Reason stage** (`stage = "cancel-reason"`, state in `pendingCancel`, cleared by
`clearPendingBatch` and `onClose`).

- Title: the row's title.
- `headerIcon`: `ban`.
- `inputLabel`: `Cancel reason`.
- Placeholder: `Why cancel it? (optional · ↵ to skip)`.
- Subtitle: `Next → Cancelled · Wed 2026-09-30 · nothing written yet`. Counted and link
  sessions use their counts or `↗` subtitle in the same shape.
- `getFilteredItems` returns one live preview item. `renderResults` refreshes the footer
  as the user types, the way the schedule stage does.

The preview row renders the **exact** text that will be written
(`formatCancelLogEntryText`):

- **Reason typed:** `check-circle-2`, title `*2026-09-30* — <reason>`, meta
  `Adds ❌ CANCEL LOG as the first child`, `Prepends to the existing ❌ CANCEL LOG`, or
  (counted/link) `Adds or prepends a ❌ CANCEL LOG entry on each task`.
- **Empty, no log:** `minus-circle`, title `No reason`, meta
  `[-] + [cancelled:: 2026-09-30] only; no Cancel Log`.
- **Empty, a log exists:** the muted fallback entry `*2026-09-30* — 🤷 no reason given`,
  meta `Logged because this task already keeps a Cancel Log`. For counted sessions:
  `…on tasks that already keep one`.
- **Contains `::`:** `alert-triangle`, meta
  `"::" creates a Dataview inline field on this bullet`.
- Always a muted effects line built from cheap synchronous facts:
  - `Removes its links from today's open Pomodoros` when a target has a block ID;
  - `unblocks dependents`;
  - `keeps #now` when any target carries `#now`.

Footer hints:

- ↵ is `Cancel task` or `Cancel tasks` (empty, no fallback), `Cancel & log reason`
  (typed), or `Cancel & log 🤷` (fallback).
- esc is `Keep open`, which avoids the double meaning of "Cancel".

**Notice card.** `buildCancelNoticeModel` is pure and exported;
`renderCancelNoticeFragment` and `showCancelNotice` mirror the priority card.

- The card is `.bob-nh-notice.is-cancel`.
- **Header:** `ban` icon, level pill `Cancelled`, count pill (`3 tasks`,
  `via Task Link`, `2 tasks via Task Links`), receipt `[cancelled:: 2026-09-30]`.
- **Body:** `.bob-nh-notice-reason`, a quote block with a left rule showing
  `❌ <reason>` in italics. Without a reason it shows muted `No reason recorded`, or
  `🤷 no reason given · logged` when the fallback was used.
- **Chips:**
  - `removed N Pomodoro links` (neutral);
  - `unblocked N dependents` (positive);
  - `NOW c/cap` (neutral, 🔴 when over the cap);
  - the plan-budget chip;
  - `skipped N closed` (muted);
  - `Pomodoro links not removed` (warning).
- `model.text` is the aria-label and the plain-text fallback, e.g.
  `Cancelled 3 tasks via Task Links · “Superseded by …” · removed 2 Pomodoro links · unblocked 1 dependent · NOW 11/15`.
- The CSS in `plugins/bob-navigation-hotkeys/styles.css` uses only theme variables and
  `color-mix`, and must read well in both light and dark themes.

### Deliberate choices (and the rejected alternatives)

- **A picker row, not a new chord.** The picker already owns task-level edits for all
  three target modes. A row inherits counted and Task Link targeting, stale-preimage
  guards and cross-note writes for free, and keeps one mental model.
- **Pinned last, with danger styling.** Pinned first like `#now` was rejected: the first
  row is the ↵ default. Destructive actions sit last by convention, and filtering
  reaches them instantly.
- **A managed Cancel Log.** Rejected alternatives:
  - an inline `[cancel_reason:: …]` field: an unknown trailing field breaks Tasks' and
    bob's right-to-left field parsing (see `decisions:now-tag-is-user-owned`);
  - the Work Log: it records work performed, not a verdict;
  - a bare prose bullet: it can't be recognized, carried, or kept truthful after a
    reopen. A dated log stays accurate across cancel → reopen → cancel.
- **First-child placement.** The verdict reads first; see the grammar section.
- **Keep `scheduled`, `dependsOn` and `#now`.** Terminal tasks are never re-derived, the
  history stays intact, and `#now` is user-owned.
- **One recovery implementation, via a versioned TSC `api`.** Copying the vault-wide
  recovery into NAV was rejected. Plugins never import each other's `main.js`;
  `bob-ledger-tools` already set this `api` pattern.
- **Recurring tasks refused.** A text rewrite would silently drop the next occurrence
  that Obsidian Tasks creates.
- **No un-cancel in the picker.** `<option+]>` already reopens a task, and the Cancel
  Log remains as history.

## Phase tsc-recovery-api

Repo: `bob-plugins`. Open it with `sase repo open bob-plugins -r "<reason>"` and use the
printed path. File: `plugins/task-status-cycler/main.js`.

1. **Versioned API.** In `onload`, set
   `this.api = Object.freeze({ version: 1, recoverBlockedDependents })`.
   - `recoverBlockedDependents(closedIdentities, context = {})` normalizes identities
     with `normalizeClosedTaskIdentities`.
   - It runs `recoverBlockedDependentsNow(closed, context)` inside
     `enqueueTaskReferenceMutation`, **without** `retireClosedTaskReferencesNow`.
   - It resolves to `{ reopened, failures }` and never throws: failures are returned,
     not raised.
   - An empty list resolves to `{ reopened: 0, failures: [] }`.
   - Document the API in the bob-plugins README next to the `bob-ledger-tools` api
     paragraph.
2. **Future-schedule guard.**
   `buildBlockedDependentRecoveryPlan(documents, closedIdentities, options = {})` gains
   `options.today` (a local `YYYY-MM-DD` string, defaulting to `formatLocalDate()`).
   - A dependent whose own line has a valid, strictly future `[scheduled:: …]` or
     `(scheduled:: …)` stays Blocked. Reuse `findSingleFutureScheduledField` or the same
     parsing.
   - This deliberately applies to the Ctrl+Enter path too, aligning both with
     `bob task-status-hooks`.
3. **Cancelled Task Link guard.** When Ctrl+Enter's selected Task Link resolves to a
   task whose status is `-`, the key is consumed. It shows the Notice
   `Task is cancelled; reopen it with ⌥] first` and never falls back to completing the
   owning Pomodoro.
   - Resolve once more with a `symbol === "-"` predicate, or an equivalent, only after
     the normal resolution fails.
   - Links that do not resolve to a task keep their current fallback.
4. **Tests** in `scripts/test-task-status-cycler.cjs`:
   - the API exists and is frozen with version 1;
   - the API recovers a dependent of a Cancelled task and never strikes references;
   - the future-scheduled dependent stays Blocked, on both the API and Ctrl+Enter paths;
   - Ctrl+Enter on a Task Link to a Cancelled task, inside an open Pomodoro, leaves the
     Pomodoro open and changes nothing.
5. **Version and deploy.**
   - Bump `manifest.json` to 1.17.0, and update the manifest description and the README
     Plugins row to mention the API and both guards.
   - Run `npm test` and `npm run validate`.
   - Run `bob plugins sync -p task-status-cycler -r <bob-plugins path> --dry-run`, then
     run it for real.

## Phase cancel-planner

Repo: `bob-plugins`. File: `plugins/bob-navigation-hotkeys/main.js`. Pure functions
only; no modal or command wiring.

1. **Grammar constants and helpers.**
   - `CANCEL_LOG_EMOJI = "❌"`, `CANCEL_LOG_LABEL = "CANCEL LOG"`,
     `CANCEL_LOG_MARKER_TEXT`, `CANCEL_LOG_PARENT_RE` (with optional U+FE0F),
     `MANAGED_TASK_LOG_KIND_CANCEL`.
   - `parseManagedTaskLogParentBullet` returns the `cancel` kind.
   - `findCancelLogParent(lines, taskLine)` finds a direct child only.
   - `formatCancelLogEntryText({ date, reason })` returns `*date* — reason`.
   - `planCancelLogEntry(content, taskLine, { date, reason, fallbackReason })` returns
     the same shape as `planScheduleLogEntry`, with `createdParent`, `usedFallback`,
     `insertLine` and `lineTexts`. It applies the first-child placement, prepending,
     fallback and indentation rules from the contract.
   - Update the parity comment to say the Cancel Log is plugin-only.
2. **Status helpers.**
   - `isRecurringTaskLine(line)`.
   - `getTaskCancelStatusLabel(symbol)`, returning Ready, Next, In Progress or Blocked.
   - `describeCancelTaskRow(content, { cursorLine | taskSession | linkResolved })`
     returns `null` or
     `{ kind, count, openCount, closedCount, recurring, fromStatus, detail }`, in the
     style of `describeNowToggleRow`.
3. **The planner.**
   `planTaskCancelBatch(content, session, { date, reason, fallbackReason })` takes a
   session shaped like the counted and link-group sessions
   (`targets: [{ line, rawLine }]`).
   - Validate staleness the way `validateCountedTaskSession` does.
   - Refuse the whole batch if any open target is recurring
     (`{ valid: false, recurring: true, error }`).
   - Skip closed targets.
   - Process the remaining targets bottom-up so insertions never shift pending lines.
     For each one: checkbox → `-`, `upsertBulletProperty(line, "cancelled", date)`, then
     `planCancelLogEntry`.
   - Return:

     ```text
     { valid, content, cancelledCount, skippedClosedCount, loggedCount, createdLogCount,
       fallbackLoggedCount, nowTaggedCount, cancelled: [{ originalLine, blockId, taskId,
       fromStatus }], cursorLineShift(line) }
     ```

     `cursorLineShift` (or the equivalent post-image line mapping) lets callers clamp
     the cursor.

   - It must be pure and CRLF-preserving.

4. **Project conversion.** Make `getProjectFromTaskNoticeText` report
   `cancel log moved`, and confirm `createProjectNoteFromTask` and
   `convertProjectNoteToTask` carry a Cancel Log as a managed log.
5. **Tests.** Export every new helper on `module.exports.helpers` and cover them in
   `scripts/test-navigation-hotkeys.cjs`:
   - first-child insertion, including tabs versus spaces and tasks with and without
     children;
   - prepending to an existing log positioned last;
   - a grandchild marker is ignored;
   - fallback only with an existing log;
   - the `::` flag;
   - `[cancelled::]` placed before `^id` and replaced when present;
   - `scheduled`, `dependsOn`, `#now` and `#hide` preserved byte-for-byte;
   - counted batches with mixed open and closed targets and bottom-up correctness;
   - recurring refusal in all three syntaxes;
   - CRLF;
   - the `❌️` (with FE0F) marker parses;
   - `🗓️ **CANCEL LOG**` does not parse;
   - project conversion carries the log.
6. **Checks and deploy.**
   - Run `npm test` and `npm run validate`.
   - Run `bob plugins sync -p bob-navigation-hotkeys -r <bob-plugins path>`. Dry-run
     first; do not bump the version, since phase `cancel-picker` bumps it.

## Phase cancel-picker

Repo: `bob-plugins`. Files: `plugins/bob-navigation-hotkeys/main.js` and `styles.css`.
Before starting, read `decisions:task-status-is-derived` and
`decisions:now-tag-is-user-owned`.

1. **Cancel row.** In `showPropertyStage`, append the item built from
   `describeCancelTaskRow` for the current mode, after the property items. Pass
   `linkResolved` for link sessions, `taskSession` for counted sessions, and
   `cursorLine` otherwise.
   - Extend `filterItem`, `renderItem` (`renderCancelTaskItem`) and `openItem`.
   - Choosing a recurring row shows the refusal Notice and stays on the property stage.
   - Guard the `cancel-task` kind in `showValueStage` and `deleteSelectedProperty`.
2. **Reason stage.** Add `showCancelReasonStage(item)`, with the preview item from
   `getFilteredItems`, `renderCancelReasonPreviewItem`, `getCancelReasonHints` and
   footer refresh in `renderResults`.
   - `confirmCancelReason(item)` calls
     `this.plugin.applyTaskCancelFromPicker(this, { reason, fallbackReason: SCHEDULE_LOG_SKIPPED_REASON_TEXT })`.
   - The modal closes only on success. On failure it stays open on the reason stage with
     the Notice shown.
3. **`applyTaskCancelFromPicker(picker, options)` on the plugin.**
   - Local single and counted sessions follow the contract's editor path, including the
     same-file fold and the separate daily-note prune.
   - Link sessions go through the extracted `commitLinkPickerPlans` core. Refactor it
     without behavior change; the existing scheduled and priority tests must pass
     unchanged.
   - After the writes, close the modal, read the NOW budget (captured before the write),
     run TSC `api.recoverBlockedDependents` when available, compute the plan-budget
     chip, and show the notice card.
4. **Notice card.** Add `buildCancelNoticeModel`, `renderCancelNoticeFragment` and
   `showCancelNotice`, with the text fallback. Add CSS for the cancel row, the preview
   row states (`is-valid`, `is-empty`, `is-fallback`, `is-warning`, `is-selected`), and
   the `.bob-nh-notice.is-cancel` card, reason block and chip tones.
5. **Tests** in `scripts/test-navigation-hotkeys.cjs`, following the existing `#now`-row
   and schedule-reason picker tests:
   - The row appears or is hidden correctly: open vs. closed, plain bullets, all-closed
     counted sessions.
   - Row ordering is last.
   - The filter synonyms work.
   - The reason-stage preview texts and hints cover the typed, empty, fallback and `::`
     cases.
   - Esc writes nothing.
   - A single cancel is one editor transaction, and the cursor is clamped.
   - Today's daily note as the active note folds the prune; a separate daily note gets
     the prune.
   - A link session updates the target notes and removes the invoking link bullet with
     its subtree.
   - A stale preimage refuses and writes nothing.
   - A recurring target refuses.
   - The TSC API is called with the right identities; when it is absent, the flow still
     succeeds.
   - `#now` is untouched and the NOW chip shows `count − k`.
   - The notice model text covers each chip.
6. **Version and deploy.**
   - Bump `manifest.json` to 1.41.0.
   - Add one concise sentence about the Cancel row to both the manifest description and
     the README Plugins row.
   - Run `npm test` and `npm run validate`, then
     `bob plugins sync -p bob-navigation-hotkeys -r <bob-plugins path>` (dry-run first).
   - Remind Bryan in the phase's final note to reload the plugin in Obsidian.
   - Manually smoke-test in a scratch copy of a note if an Obsidian runtime is
     available. Otherwise say that only the automated coverage ran.

## Phase cancel-docs

Repo: `bob-cli`, the primary workspace.

1. **`docs/projects.md`.** Add `### Cancelling a task` after
   `### Deferring a task prunes it from today's open Pomodoros`. Cover:
   - the gesture and the three target modes;
   - the written shape, with the example above;
   - the Cancel Log grammar and first-child placement;
   - the empty-reason and fallback rule;
   - the side effects: the prune, dependent recovery, and NOW/plan chips;
   - the refusals (recurring, stale) and the undo scope.

   Add the link to `## Contents` if subsections are listed there.

2. **README glossary table.** Next to the Schedule Log and Work Log rows, add
   `| Cancel Log | A managed ❌ **CANCEL LOG** child (placed first) that records why a task was cancelled |`.
3. **`docs/task-status-hooks.md`.**
   - In "Canceled Task References", note that the Ctrl+Shift+P cancel applies the same
     removal at once, using the deferral prune's dedicated-bullet/token semantics, and
     that the hooks stay authoritative.
   - In the Ctrl+Enter recovery paragraph, document the future-schedule guard, the
     Cancelled Task Link guard, and that the picker's cancel reuses the same recovery
     through Task Status Cycler's `api.recoverBlockedDependents`.
4. **Parity comment.** Near `MANAGED_TASK_LOG_LABELS` in
   `src/native/capture/sub_bullet.rs`, add a comment explaining that the plugin-only
   `❌ **CANCEL LOG**` is deliberately not a managed-log anchor. It is placed first, so
   new captured notes land below it, and emoji-led bullets are never task sections.
5. **Checks.** Run `just all`. If `just lint` fails only on the pre-existing `|| true`
   clippy deny tracked on `bob-cli-28`, record that instead of fixing it here.
6. **Follow-up note.** Record a `PROPOSED FOLLOW-UP:` note on this phase bead for a new
   `glossary:cancel-log` term. Memory changes need Bryan's authorization, so do not edit
   `sase/memory/`.

## Acceptance

- **Single task.** Ctrl+Shift+P → Cancel → reason → ↵ on a Next task yields `[-]`,
  `[cancelled:: <today>]`, and a first-child `❌ **CANCEL LOG**` with
  `*<today>* — <reason>`. The task's links in today's open Pomodoros are gone. A Blocked
  dependent with no other blockers is Ready. `#now` is unchanged, and one Cancelled
  notice card summarizes the result.
- **Counted and link sessions.** `2<Ctrl+Shift+P>` and Task Link sessions cancel every
  open target atomically, with one shared reason. A recurring or stale target refuses
  with nothing written.
- **Empty reason.** An empty reason writes no log unless one already exists, in which
  case it adds the 🤷 entry.
- **Task Status Cycler guards.** Ctrl+Enter on a Task Link to a Cancelled task never
  completes the Pomodoro, and future-scheduled dependents stay Blocked.
- **Checks and docs.** `npm test` and `npm run validate` pass in bob-plugins, both
  plugins are synced to the vault, and the bob-cli docs describe the behavior.
