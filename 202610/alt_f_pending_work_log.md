---
tier: tale
title: Optional Work Log when Alt+F refreshes a Pending task
goal:
  Ask for an optional work summary before Alt+F or Alt+Shift+F refreshes a Pending task,
  and save a nonblank summary in that task's Work Log in the same write as the freshness
  stamp.
size: medium
proposed_by: bbugyi200.apollo.4v
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4v](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4v.md)
- **COMMITS:**
  - [223974d](https://github.com/bobs-org/bob-cli/commit/223974dca6d703d11428d363f50f0a78d1e8b5a1)
    — docs(freshness): document Pending refresh Work Log prompt

# Optional Work Log when Alt+F refreshes a Pending task

## Goal and scope

When Alt+F refreshes a Pending Obsidian task (`[/]`, the In Progress checkbox), ask once
for an optional work summary before anything is written. A nonblank summary becomes one
locally dated, newest-first entry under that task's managed `🛠️ **WORK LOG**`. Enter on
an empty summary still refreshes freshness and writes no Work Log. Escape cancels the
refresh: no stamp, no log, and no advance to the next due task.

Alt+Shift+F is the same refresh with advance. Both commands, the Vim normal-mode capture
path, and a counted `N` prefix call `refreshTaskFreshness`, so the prompt lives there.
The morning lane keep is Alt+Shift+F; limiting the prompt to the unshifted chord would
skip the walk that actually reviews Pending tasks.

This is one medium tale. One implementation agent can land the prompt, the combined
stamp-and-log write, regression coverage, documentation, the version bump, and
deployment. It does not need epic phases.

Implement in the linked `bob-plugins` repository. Open it with
`sase repo open bob-plugins -r "Implement the optional Alt+F Pending Work Log prompt"`,
use the printed path, and read its `AGENTS.md`. Update the freshness and scheduling docs
in this `bob-cli` checkout. Never edit deployed copies under `~/bob/.obsidian/plugins/`.
This planning turn changes only the scratch plan.

## Findings that determine the implementation

Navigation hotkeys is `1.71.1`. In `plugins/bob-navigation-hotkeys/main.js`:

- `refresh-task-freshness` (Alt+F) and `refresh-task-freshness-and-advance`
  (Alt+Shift+F) both call `refreshTaskFreshness`. Vim normal mode reaches the same
  method from `handleReviewRefreshPhysicalKeydown`, because CodeMirror swallows Alt
  chords before Obsidian's hotkey dispatcher.
- The method discovers the cursor task, or a dedicated Task Link's target, plus the next
  N targets when a count is explicit. `planFreshStampBatch` classifies every target
  first: one closed, recurring, or non-task line refuses the whole batch with the note
  unchanged. Surviving open lines (` `, `/`, `*`, `?`) stamp through
  `api.freshness.keepLine` when the namespace can count, and through `stampLine` only on
  a pre-v5 namespace. A v5 namespace missing `keepLine` fails with no write.
- A single source-task press on an exact, due, at-limit Ready row opens
  `FreshnessDecayCardModal` and writes nothing. Counted sessions and every Task Link
  session skip those rows. Pending rows are not that card. The card's own Alt+F means
  Keep and is not this prompt.
- Pending is the `[/]` checkbox. Queue tier labels are a different fact: an existing
  keep-counting fixture names a `[*]` line "pending lane". Eligibility for this prompt
  is the pre-write checkbox, not `tier === "pending"`.
- `normalizeLaneWorkSummary`, `formatLaneWorkLogEntry`, `findLaneWorkLogParent`, and
  `insertLaneWorkLogEntry` already prepend a Work Log entry for Alt+N release of an In
  Progress task. `applySchedulingWorkLogsToLines` applies that helper bottom-up for the
  Ctrl+Shift+P scheduling prompt. Reuse them. Do not call `planTaskLaneBatch` or the
  scheduling writers; those change lanes or dates.
- Alt+N asks through `requestLaneReleaseSummary` / `LaneReleaseSummaryModal`, whose copy
  is "why is this pending?" and whose failure path continues the release. Scheduling
  asks through `showSchedulingWorkLogStage` with placeholder
  `What did you get done? (optional · ↵ to skip)`, a dated preview, a `::` warning, and
  "nothing written yet". The refresh prompt should use the scheduling wording. It should
  not reuse the release modal.
- `finishFreshStamp` builds the Fresh notice from the pre-write queue. Pending and Next
  stamps do not increase upkeep. Work log lines sit under the task, so the stamped
  task's pre-write line number stays valid for queue matching when insertions are
  applied bottom-up after the line stamp.
- `scripts/test-navigation-keep-counting.cjs` drives `refreshTaskFreshness` on Ready,
  Next, and NEW lines and must keep doing so with no modal. `scripts/modal-harness.cjs`
  can open a real modal stub. `scripts/test-navigation-hotkeys.cjs` and
  `scripts/test-navigation-roll-decay.cjs` already cover lane-release and scheduling
  Work Logs.

Before editing, read this canonical memory in one batch with `/sase_memory_read`:
`glossary:work-log`, `glossary:task-freshness`, `glossary:schedule-log`, and
`decisions:task-lanes-are-sticky`, with a specific reason. Do not edit those files. This
request does not authorize a memory change, and an approved plan may edit memory only
when its steps name that edit.

## Behavior contract

Decide eligibility from each target's validated pre-write line, before the stamp. The
line must be a real open `#task` whose checkbox is `/`, and the target must be one this
gesture will actually stamp. A decision-card skip is not stamped and gets no entry.

| Target or gesture                                                                                                | Prompt                                                                                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Alt+F or Alt+Shift+F on one Pending `[/]` task, including a Pending `^prj` lifecycle task                        | Ask, then stamp that task and log only on it                                                                                                                                                |
| Counted `N` Alt+F / Alt+Shift+F, or a Task Link batch, when at least one stamp target is `[/]`                   | Ask once. The shared summary is written only on the qualifying Pending targets. Other stamp targets still refresh                                                                           |
| Ready, Next, Blocked, NEW, or already-fresh non-Pending targets, including a batch with zero `[/]` stamp targets | Today's refresh, with no prompt                                                                                                                                                             |
| Single exact due Ready task at its keep limit                                                                    | Decision card only. No Work Log stage                                                                                                                                                       |
| Closed, recurring, or non-task refusal                                                                           | Existing refusal notice. No prompt and no write                                                                                                                                             |
| Escape or dismissal                                                                                              | No stamp, no log, no advance, no Pomodoro or lane edit                                                                                                                                      |
| Enter with an empty or whitespace-only summary                                                                   | Stamp as today. Create no marker and write no `🤷` fallback                                                                                                                                 |
| Enter with text                                                                                                  | Stamp, and prepend one `*YYYY-MM-DD* — <summary>` entry on each qualifying Pending target                                                                                                   |
| Same-day Pending refresh whose `fresh` is already today                                                          | Still ask. A typed summary is written even when the stamp leaves the task line byte-identical. A blank submit stays a successful refresh and still advances when the chord asked to advance |
| Ctrl+Shift+P refresh row, Alt+N, Ctrl+Shift+Enter unlink, scheduling Work Log                                    | Unchanged                                                                                                                                                                                   |

The date on the entry is the same local calendar day as the freshness stamp
(`options.dateText` in tests, otherwise today). Whitespace collapses the way
`normalizeLaneWorkSummary` already does. A `::` in the summary warns in the preview and
still submits. The entry is prepended under the task's direct Work Log child, including
a legacy accepted marker, using that marker's indent and list marker. A missing log
appends a new `🛠️ **WORK LOG**` child after the task's existing children. A nested
marker owned by a child task is not the parent's log.

Title the modal `Refresh task`, or `Refresh N tasks` when the stamp batch contains N
tasks. Show the qualifying count when it differs from N (`1 of 3 tasks qualify`), the
dated preview, where it will be saved, and that nothing is written yet. Footer hints
follow the scheduling pair: `Refresh without a summary` while the field is empty,
`Refresh & log summary` once it has text, and `esc` cancels. Opening the modal must not
stamp.

A thrown or missing modal refuses the gesture with a notice and writes nothing. Tests
pass an explicit summary, or drive the modal. They do not depend on a silent blank
fallback.

The Fresh notice gains a `1 Work Log` / `N Work Logs` chip only when entries were
written. Upkeep, `kept N×`, decision-skip tails, and lane-stamp accounting stay as they
are.

## Implementation sequence

1. **Add a pure stamp-plus-log plan.** Add a helper next to `planFreshStampBatch`,
   exported on `module.exports.helpers`, that stamps with the existing planner and then
   inserts Work Log entries. Stamp every accepted line first so line counts stay put,
   then call `insertLaneWorkLogEntry` bottom-up only for stamped targets whose pre-write
   checkbox is `/` and whose normalized summary is nonblank. Return the existing plan
   fields plus `workLogWrittenCount`. A blank summary returns the stamp plan with a zero
   count. A stamp refusal returns before any insertion. Preserve CRLF, quote prefixes,
   and child blocks the existing inserter already preserves.

2. **Ask before either write path.** In `refreshTaskFreshnessOnTasks` and
   `refreshTaskFreshnessOnLinks`, after discovery, the v5 `keepLine` guard, and the
   decision-card branch, collect the targets that will be stamped. If any has checkbox
   `/`, await one prompt. `options.summary` as a string skips the modal and uses that
   string, including `""`. Escape returns `false` before `planFreshStampBatch` runs.
   Re-read the editor, linked-note preimages, and target raw lines after the prompt; a
   change refuses the whole gesture with the existing stale-note notice and zero writes.
   Then build one postimage per note with the new helper and commit it through the
   existing single-note transaction or `commitLinkPickerNoteWrites`. Do not write the
   log in a second transaction. Cursor placement stays the current stamp behavior:
   insertions are children at or below the cursor task. Advance only after that commit
   succeeds, including a blank summary whose stamp is a no-op.

3. **Add the modal.** Add a refresh-specific modal beside `LaneReleaseSummaryModal`. Use
   the scheduling placeholder, preview, `::` warning, qualifying-count subtitle, and
   "nothing written yet" note. Enter confirms once. A second Enter, click, or late
   callback after close cannot stamp twice. Dismissal calls the cancel path. Do not
   route this through `BulletPropertyPickerModal` or change
   `showSchedulingWorkLogStage`.

4. **Surface the count.** Thread `workLogWrittenCount` into `finishFreshStamp` and
   append the chip with the existing count formatter. A zero count adds nothing.

5. **Document, version, and deploy.** In `docs/freshness.md` §5 and §6, say that Alt+F
   and Alt+Shift+F ask before stamping when a Pending target will be stamped, that a
   blank Enter still keeps, and that Escape leaves the task due. Leave the decision-card
   paragraph as the Ready at-limit path. In `docs/projects.md`, under Scheduling Work
   Log, state that this Pending refresh prompt is separate and that the picker refresh,
   lane, cancel, dependsOn, and property-deletion rows still do not ask. Extend the
   Alt+F sentence in `bob-plugins/README.md`, set that row's version to `1.72.0`, and
   update the following "ahead of the others" version so it matches the table. Bump
   `plugins/bob-navigation-hotkeys/manifest.json` from `1.71.1` to `1.72.0` and add one
   description clause. Add CSS only when existing picker classes cannot render the
   preview.

## Verification and acceptance

- One Pending task: typed summary prepends under a new log and under an existing log;
  blank and whitespace refresh with no marker; Escape writes nothing and does not
  advance; same-day `fresh` plus a typed summary adds the entry and leaves the task line
  unchanged when the stamp is already canonical; `::` warns and still saves; tabs,
  spaces, a legacy marker, a nested child task's own log, a Schedule Log sibling, and
  CRLF stay intact.
- Next, Ready, Blocked, and the existing `[*]` "pending lane" keep-counting case refresh
  with no modal. A Ready at-limit single press still opens only the decision card.
  Closed and recurring batches still refuse before a prompt.
- Counted and Task Link batches with mixed statuses ask once, log only `[/]` targets,
  dedupe repeated links to one entry in the source note, and keep one undo step. A batch
  whose only `[/]` targets were decision-skipped does not arise for Pending; a batch
  with no `[/]` stamp target does not ask.
- A note or linked preimage that changes while the modal is open writes nothing. A modal
  that fails to open writes nothing. Duplicate confirmation cannot double the entry or
  the stamp.
- Alt+Shift+F advances only after blank or typed confirmation. The Fresh notice includes
  the Work Log chip only when an entry was written, and Pending stamps still do not grow
  upkeep.
- Existing Alt+N, Ctrl+Shift+Enter, and scheduling Work Log tests stay green.

Run from the opened `bob-plugins` checkout:

```sh
node --test scripts/test-navigation-freshness.cjs scripts/test-navigation-keep-counting.cjs scripts/test-navigation-hotkeys.cjs scripts/test-navigation-decision-card.cjs scripts/test-navigation-decision-card-handlers.cjs scripts/test-navigation-stamps.cjs
npm test
npm run validate
```

Run `git diff --check` in `bob-plugins` and in this `bob-cli` checkout. No Rust test run
is required when bob-cli changes only Markdown.

Then deploy from the path `sase repo open` printed, as `task_plugins_repo`:

```sh
bob plugins sync --no-pull --repo "$task_plugins_repo" --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo "$task_plugins_repo" --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo "$task_plugins_repo" --format json
```

Confirm the list shows `bob-navigation-hotkeys` synced at `1.72.0`. Sync's exit code
alone is not enough, because a dirty vault file can be skipped. If Obsidian is open,
reload the plugin and exercise one Pending Alt+F: typed summary, blank Enter, and
Escape. If this environment cannot drive Obsidian, say that the live keymap was not
exercised.

## Out of scope

- Next (`[*]`) refreshes. Scheduling already prompts for Next; this tale follows the
  requested Pending keymap.
- Decision-card Keep, the Ctrl+Shift+P refresh row, Alt+N, and Ctrl+Shift+Enter.
- `bob capture`, ledger-tools' stamp API, and any Rust writer.
- `sase/memory/`, including `glossary:work-log`. Do not edit it, and do not regenerate
  `AGENTS.md`, to record this prompt. The product contract for the change is
  `docs/freshness.md`.
