---
tier: tale
size: medium
title: Finish and land the snooze plugin plumbing epic
goal:
  Fix the remaining snooze plugin defects the bob-cli-q5.4.1 landing review found, add
  the missing gesture regression tests, then close bob-cli-q5.4.1 and its parent phase
  bob-cli-q5.4.
proposed_by: bbugyi200.athena.bob-cli-q5.4.1.land
proposed_with: claude/opus
bead: bob-cli-q5.4.1
create_time: 2026-11-26 12:15:53
status: wip
---

- **PARENT:**
  [202611/snooze_plugin_plumbing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snooze_plugin_plumbing.md)
- **BEAD:**
  [bob-cli-q5.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.4.1.md)

# Finish and land the snooze plugin plumbing epic (bob-cli-q5.4.1)

This tale is the remaining work, plus the closeout, for epic `bob-cli-q5.4.1`
(`plan:202611/snooze_plugin_plumbing.md`). That epic implements the `plugin-plumbing`
phase `bob-cli-q5.4` of the outer snoozed-tasks epic `bob-cli-q5`. All five child phases
(`bob-cli-q5.4.1.1` to `.5`) are closed. The epic's land agent verified them and found
the gaps below. Nothing else resumes this landing, so the coder of this tale **must
finish the landing itself** (see "Closeout"). It must close the epic and its parent
phase in the same turn as the final code, never waiting on this work's own commit, push
or CI.

All code changes are in the linked `bob-plugins` repository. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first. Edit `src/`
fragments only, register any new fragment in `src/fragments.json`, run `npm run build`,
and never hand-edit generated `main.js`. Each hand-edited fragment stays ≤ 1000 lines,
and new suites go in `package.json`'s `test:unit` list. All paths below are relative to
bob-plugins unless they say bob-cli. No bob-cli code, fixture, version or `CHANGELOG.md`
change is expected (`decisions:bob-cli-releases-ride-a-daily-train`). If one turns out
to be necessary, run bob-cli `just check` (never `just check-full`) and
`just docs-check`.

Governing decisions (unchanged from the epic):
`decisions:today-is-read-from-the-ledger`, `decisions:task-lanes-are-sticky`,
`decisions:consumption-keeps-notes-and-recorded-time`,
`decisions:unblocked-tasks-resurface-as-ticklers`,
`decisions:unblocked-tasks-are-offered`, and
`decisions:calendar-events-are-daily-note-event-tasks`. The outer epic's reviewer
choices stay final (`uncommitted_rows = ignore`, `wake_notice = hud`,
`hold_in_meetings = yes`). This tale edits no memory files.

## What the land agent already verified (do not redo)

- The source, commits and tests of all five phases, on bob-plugins `a14ccc46` through
  `e36ddc68`. The only drift during the epic was `bob-cli-qi.3`'s Bank Card refactor
  (`8e6a5a61`). Its dangling `arbitration.owner` reference was already repaired in
  `86f5f340`/`0d822970`. The two concurrent bob-cli commits (`478d5233` bank/props,
  `4dd787bc` drawers) do not touch snooze contracts, and the canonical fixtures are
  unchanged since the epic started.
- Integrated gates on `e36ddc68` with `BOB_CLI_DIR` set to the bob-cli checkout:
  - `npm run validate`: 12/12 plugins.
  - Targeted snooze suites: 140/140.
  - `npm test`: 9902 passed, 4 skipped, 0 failed, plus perf 26/26.
  - `npm run docs:check` passed, and `bob plugins sync --dry-run` reported 0 to copy.
- The live vault `dash.md` carries the `isSnoozedToday` exclusions in the PENDING, NEXT
  and READY queries and in the DataviewJS counts (vault commits `bc6607cb8`,
  `6bae272e5`). The TODAY query is unchanged.
- In an isolated vault, `bob snooze wake -q -f json` (bob 0.6.0) made one placement. The
  repeated trigger reported `nothing_due` and left the daily note byte-identical.
- Follow-up triage is done and recorded on the epic bead.

## 1. Make the minute waker accept the CLI's real reports

`plugins/bob-ledger-tools/src/290b-snooze-waker.js` `snoozeWakeDecodeReport` rejects two
legal `bob snooze wake` reports as `bad-shape`. Both were reproduced with node against
`helpers.snoozeWakeDecodeReport`:

1. **Untyped wake into an unnamed Pomodoro.** bob-cli `WakeDestination.name` is
   `Option<String>` (`src/native/snooze/plan.rs` `untyped_destination` returns
   `name: None`). The real report for an untyped task woken into `- [ ] ()` is:

   ```json
   {
     "daily_file": "2026/20261008.md",
     "date": "2026-10-08",
     "dropped": [],
     "ignored": [],
     "next_wake_at": null,
     "notified": "none",
     "now": "2026-10-08T12:00:00-04:00",
     "pending": [],
     "plan": { "cap": 10, "count": 1, "over": false },
     "schema_version": 1,
     "status": "woke",
     "wake_id": "18eba710ff76f2a0-0-668023",
     "woken": [
       {
         "block_id": "job",
         "destination": { "created": false, "line": 4, "name": null },
         "over_cap": false,
         "route": "jobs.md",
         "target": "1100",
         "title": "Review the launch plan",
         "woke_at": "12:00"
       }
     ]
   }
   ```

   The decoder requires a string `name`, so this common case gets no post-wake cache
   refresh and no fallback Notice. Accept `name: string | null`. For a single wake with
   a null name, the Notice copy drops the destination segment:
   `⏰ Ready now: <title> · woke <HH:MM>`. Named destinations keep
   `⏰ Ready now: <title> · woke <HH:MM> · <NAME>`, and the batch copy is unchanged.

2. **Deferred.** bob-cli `deferred_action` (`src/native/snooze/cli.rs`) emits only
   `schema_version`, `date`, `now`, `status: "deferred"`, `daily_file`, `notified` and
   `wake_id`. Accept that slim envelope for `status: "deferred"` only. Validate those
   base fields, and validate any list or plan fields only when they are present. Treat
   it as a no-op: no refresh, no Notice, no session disable, and a retry on a later
   eligible minute. Keep the full-envelope validation for `woke`, `nothing_due` and
   `skipped`. The doc/code mismatch on the CLI side is tracked as a DISCOVERED ISSUE
   note on `bob-cli-q5`. Do not change bob-cli here.

Add tests in `scripts/test-ledger-tools-planned-today.cjs` that use the exact JSON above
and the exact deferred shape:

- Decoder acceptance.
- A tick whose woken row has a null destination name with `notified: "none"`. It must
  produce exactly one 8 s Notice without the name segment, plus one cache refresh.
- A deferred tick. It produces no Notice, does not disable the waker, and retries on the
  next minute.

If the null-name behavior needs a sentence, update `docs/develop/plugin-apis.md`.

## 2. Task-status-cycler `[unblocked::]` recovery honors planned-today (SP14)

Rust `mark_unblocked` skips the automatic `[unblocked::]` write for both Today-linked
and snoozed-today tasks. Its callers in bob-cli `task_complete/project_close.rs` and
`task_status_hooks/sync.rs` pass `is_snoozed_today` from the pre-recovery snapshot. The
ledger `api.freshness.markUnblockedLine` already honors an `isSnoozedToday` setting
(`plugins/bob-ledger-tools/src/080-freshness-placement.js`, SP14). Phase
`bob-cli-q5.4.1.2` flagged the gap below for the later phases of this epic, but they
never closed it.

task-status-cycler never passes that membership:

- `src/088-project-blocking.js` `projectRecoveryTaskIsToday` reads only
  `ledger.api.isToday`. It feeds `rewriteRecoveredTaskLine` from the project-resident
  recovery (~L266) and from `src/130-plugin-references.js`'s blocked-dependent recovery,
  on both the open-editor path (~L424) and the `vault.process` path (~L459).
- The successor and offer-close recovery paths derive `isToday` only from a live ledger
  link: `hasLiveLink` in `src/089b-successor-verdicts.js` (~L212/219/433/440),
  `src/089-successor-project.js` (~L741), `src/089i-successor-offer-close-apply.js` and
  `src/137b-successor-offer-close-api.js`.

As a result, closing a prerequisite from Obsidian stamps `[unblocked:: today]` on a
snoozed Blocked dependent, which the CLI never does. Thread snoozed membership through
each of these markUnblockedLine call sites. Prefer passing
`isSnoozedToday: ledger.api.isSnoozedToday(task)` alongside the existing `isToday`, with
the task shape `snoozedCacheKey` accepts (`path` plus block id/`blockLink`). Read it
before the write, and fall back to `false` when the API is missing so older ledgers keep
today's behavior. Keep `isToday` itself ledger-only
(`decisions:today-is-read-from-the-ledger`). Recovery stays automatic
(`decisions:unblocked-tasks-are-offered`), and only the field write is skipped.

Tests (task-status-cycler suites) cover these cases on the editor and vault paths:

- A snoozed `[?]` dependent with an older `[fresh::]` recovers to `[ ]` without
  `[unblocked::]`.
- The same dependent with no snooze still gets the field.
- A Today-linked dependent keeps today's behavior.
- The successor offer-close path skips the field for a snoozed dependent.

Navigation's `navNoteScopeFlags` gap predates this epic and is tracked as task
`bob-cli-ql`. Leave it alone.

## 3. Structural Event detection in the snoozed cache

`plugins/bob-ledger-tools/src/290a-snoozed-today-cache.js` `snoozedIsEventTask` treats
any task whose text contains a standalone `MEET` as an Event Task. Rust's rule is
structural (bob-cli `src/native/event_task.rs` `event_task_at`): the target note is a
canonical daily note, the line lies inside its `## Events` H2 section (up to the next
H2, outside fences), and the line parses as an Event Task row. Under the heuristic:

- A MEET-typed task in a project note is wrongly ignored. Rust classifies it as snoozed,
  so JS fails to exclude it from walks and lanes.
- An Events-section task without a `MEET` token is wrongly treated as an ordinary row.

Replace the heuristic with the structural rule and keep it synchronous within the
existing rebuild. Usable inputs:

- The Tasks fact's `taskLocation` path and `precedingHeader`/section data.
- Today's already-loaded daily content.
- Existing ledger-tools Events helpers such as `gcalEventsFindSection` in
  `src/035-meet-event-run.js`, or `metadataCache` headings.

Undeterminable cases stay non-Event, as the current heuristic does. Keep the export in
`src/381-snoozed-cache-exports.js` working, and rename it if the semantics change. Add
cache tests for the two misclassified cases above, plus a hand-authored row that targets
a real Events-section Event Task, which classifies `event_task` and is ignored. All SP
parity tests must stay green.

## 4. Missing public-gesture regression tests

The epic plan required public-path regression tests that the phases did not add. Add
them beside the existing suites and drive the real entrypoints, such as
`openPomodoroTaskLink`, the cycler's Ctrl+Enter command path and the nav gesture
commands, not only the pure planners. Fix any defect a test exposes, inside this tale.

block-id-prompt (`scripts/test-block-id-prompt-pomodoro-link-runtime.cjs`; reuse its
`createSnoozedLinkNowHarness`):

- **Definition link-now.** Ctrl+Shift+Enter on the task definition, when the task's only
  presence is an active Snoozed row (`pomodoroTaskLinkPresence` → `"snoozed"`), links
  rather than unlinks. The row's authored notes move under the new link, the row (and an
  emptied heading) goes away, and Ready becomes Next.
- **Same note.** A task defined in today's daily note links now in one composed editor
  transaction (one logical undo).
- **Unlink.** Unlinking a task that has both a live ledger link and an active row
  removes both and leaves the lane unchanged (`decisions:task-lanes-are-sticky`).
- **Cancel.** Cancelling the route picker or the Pomodoro picker leaves the daily note
  and task bytes unchanged.
- **Ambiguous target.** An ambiguous basename row target refuses with a Notice and
  writes nothing.
- **Blocked target.** A row whose task is `[?]` follows the existing blocked-link
  consumption, and no orphan row remains.

task-status-cycler (`scripts/test-task-status-cycler-pomodoro-completion.cjs`):

- **Row Done.** Ctrl+Enter on an active Snoozed row goes through the public gesture. It
  completes the task and removes the row and its subtree rather than striking the link
  in place. Struck history rows stay.
- **Definition Done.** Done on the definition in another note removes today's active
  row.
- **Unchanged paths.** Event Task completion and ordinary struck-history behavior stay
  unchanged.

Trailing sections: add a trailing `## Snoozed` section after `## Pomodoros` to the
ledger insertion and retirement tests (BIP `planPomodoroLinkInsertion`, the cycler's
retirement and archive appenders, and ledger-tools appenders that have tests). The
assertions must prove that no appender writes past the next H2 and that row link
rewrites keep row times.

bob-navigation-hotkeys (`scripts/test-navigation-hotkeys-snoozed-cleanup.cjs`): add one
public-path test for each of these:

- Cancel.
- Roll.
- The schedule setter.
- A move into a blocked project (`src/250-task-move.js`).
- Project conversion (`src/665-plugin-project-conversion.js`).
- Relation cancellation (`src/596-relation-cancel-compose.js`).

Each uses a Snoozed row and no ledger link, and asserts that the row is removed in the
same write. Also add one ordinary move or rename that rewrites the row link while
keeping its notes and time.

## Gates and deploy

In bob-plugins, with `BOB_CLI_DIR` set to the bob-cli checkout:

1. `npm run build`.
2. The touched suites.
3. `npm run validate`.
4. `npm test`, which takes about a minute of wall time. Run it in the foreground with a
   generous timeout.
5. `npm run docs:check`.

Then run `bob plugins sync`. If sync refuses another checkout's deploy, follow
bob-plugins `AGENTS.md` and never pass `-F`. Keep sources and generated bundles in the
same declared commit.

## Closeout (final step: finish the landing in this same turn)

Do these only after every gate above passes:

1. Run `sase bead epic-symbols bob-cli-q5.4.1`. Resolve every listed `--epic-symbol`
   entry (wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy), or re-key it to a still-open bead that needs the
   exemption. At landing review there were none.
2. Close the epic with `sase bead close bob-cli-q5.4.1 --note "<verification>"`. The
   note combines the land agent's verification listed above with this tale's results:
   the decoder fixes, the SP14 cycler recovery, structural Event detection, the new
   gesture tests, and the gate counts and sync result. Never use `--force` to make the
   close succeed.
3. In the bob-cli checkout, run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file, the `PLAN` path that
   `sase bead read bob-cli-q5.4.1 -r "<why>"` prints
   (`plan:202611/snooze_plugin_plumbing.md`). Its current value is `status: wip`.
5. The epic's `parent_bead` is the **phase** bead `bob-cli-q5.4`. Confirm that this
   child plan completed the phase's described work: the JS mirror on the shared vectors,
   the `isSnoozedToday`/`isPlannedToday` API, the `]s`/`]w`/dash exclusions and the
   `dash.md` edit, cleanup parity in every link-removing gesture, Snoozed-row
   Ctrl+Enter/Alt+N/Ctrl+Shift+Enter, and the minute wake spawn with its Notice
   fallback. Then close only that phase with
   `sase bead close bob-cli-q5.4 --note "<what you verified>"`. Do not close the outer
   epic `bob-cli-q5`; its own land agent owns it.
