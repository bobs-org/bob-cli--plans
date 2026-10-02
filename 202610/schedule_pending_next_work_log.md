---
tier: tale
title: Optional Work Log when scheduling Pending or Next tasks
goal:
  Offer an optional work summary before Ctrl+Shift+P schedules Pending or Next tasks and
  save nonblank summaries atomically with the scheduling edits.
size: medium
proposed_by: bbugyi200.apollo.4c
create_time: 2026-10-02 18:27:40
status: wip
---

# Optional Work Log when scheduling Pending or Next tasks

## Goal and scope

When `Ctrl+Shift+P` schedules an Obsidian task that starts in Pending (`[/]`, internally
In Progress) or Next (`[*]`), offer an optional work summary before committing the
scheduling gesture. A nonblank summary becomes a locally dated, newest-first entry under
that task's managed `🛠️ **WORK LOG**`. Enter with a blank summary schedules normally
without touching its Work Log. Escape cancels the entire gesture, including any chosen
date, priority, schedule reason, and ledger cleanup. The existing Schedule Log continues
recording scheduling reasons.

This is one medium tale: a bounded change to one plugin's modal and scheduling writers,
with regression coverage, documentation, versioning, and deployment. It needs no
independently implemented epic phases.

Implement in the `bob-plugins` source-of-truth linked repository, opened through
`/sase_repo`
(`sase repo open bob-plugins -r "Implement optional scheduling Work Log prompt"`). Read
its `AGENTS.md`. Update scheduling documentation in this `bob-cli` checkout. Never edit
deployed plugin files directly. This planning turn must change only the scratch plan;
implementation starts after plan approval.

## Findings and implementation entry points

The current `bob-plugins` navigation plugin is version `1.51.0`. In
`plugins/bob-navigation-hotkeys/main.js`:

- `BulletPropertyPickerModal.showValueStage` sends explicit `scheduled` dates through
  `showScheduleReasonStage` and `confirmScheduleReason`. Nothing is written until that
  reason is confirmed. Priority picks and pinned roll rows currently apply immediately
  through `applySelectedValue`.
- `applyRecommendedRoll`, `applyCountedRecommendedRoll`, and `applyLinkRecommendedRoll`
  are additional entry points for `Ctrl+Enter` / the existing Cmd equivalent.
  Recommendations can roll, decay, cancel, or skip a target. They bypass the
  schedule-reason stage and therefore need explicit integration with the new prompt.
- `applySelectedValue` dispatches to inline, project-frontmatter, counted, and Task Link
  writers. It currently drops options in several priority branches; adding a prompt only
  to `confirmScheduleReason` would miss these paths.
- `normalizeLaneWorkSummary`, `formatLaneWorkLogEntry`, `findLaneWorkLogParent`, and
  `insertLaneWorkLogEntry` already implement the task-local Work Log grammar for lane
  release. Reuse these pure helpers rather than calling the lane-release operation,
  which would change task lanes.
- `planCountedBulletPropertyBatch` builds scheduled/priority edits, propagates project
  schedules, then inserts Schedule Log entries. `planRecommendedRollBatch` composes
  cancellations with scheduling and remaps original task lines after Cancel Log
  insertion. These are the natural places to compose batch Work Logs.
- Single-task writes run through `applyInlineTaskPropertyEdits`,
  `setBulletPropertyValue`, `setBulletPriorityValue`, and
  `setProjectNoteScheduledValue`. Counted writes use `setCountedBulletPropertyValue` and
  `setCountedBulletPriorityValue`; Task Links use `applyLinkPickerPropertyValue`,
  `applyLinkPickerPriorityValue`, and the shared `commitLinkPickerNoteWrites` /
  `commitLinkPickerPlans` machinery.
- Existing safeguards include stale task/note checks, one editor undo transaction,
  link-note preimage validation and rollback, future-date Blocked transitions, freshness
  stamping, and pruning live links from today's open Pomodoros. Same-note daily cleanup
  folds into the target edit. Preserve these safeguards.

Tests already provide Obsidian modal/editor/vault stubs and exported helpers in
`scripts/test-navigation-hotkeys.cjs`, `scripts/test-navigation-roll-decay.cjs`, and
`scripts/test-navigation-stamps.cjs`. Work Log grammar and existing unlink behavior also
have coverage in `scripts/test-block-id-prompt.cjs`. The plugin README and
`bob-cli/docs/projects.md` currently say automatic rolls write immediately, which will
need qualified wording.

Before implementing, use `/sase_memory_read` to read the canonical guidance in one
batch: `glossary:work-log`, `glossary:schedule-log`, `glossary:pomodoro`,
`decisions:task-lanes-are-sticky`, and `obsidian.md`, with a specific reason. These were
reviewed when authoring this plan. Pending is the display name for the existing `[/]` In
Progress status; this feature introduces no new status.

## Behavior contract

Determine eligibility from each explicit target's validated task line **before** any
schedule propagation, checkbox recovery/blocking, freshness stamp, or cancellation. The
target must be a real open `#task` in `/` or `*`, and its action must actually be a
scheduling action. Do not infer eligibility from links or from the resulting checkbox,
which may already be `[?]`.

| Gesture / target                                                                                                 | Prompt behavior                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Typed or preset `scheduled` date on Pending/Next                                                                 | Existing schedule-reason stage, then the optional Work Log stage, then one commit                                                  |
| Pick a configured priority level, or choose the pinned date-roll row                                             | Keep its deterministic Schedule Log reason; offer Work Log before committing                                                       |
| Recommended roll or decay via `Ctrl+Enter`                                                                       | Keep the chosen recommendation and deterministic reason; offer Work Log before committing                                          |
| Counted tasks or Task Links, including mixed statuses                                                            | Ask once if any scheduling target qualifies; identify the qualifying task count and apply the shared summary only to those targets |
| Mixed recommendation batch with roll/decay/cancel/skip                                                           | Only eligible roll/decay targets receive Work Logs; the final confirmation commits the existing whole batch                        |
| A directly selected Pending/Next `^prj` task                                                                     | Prompt and log on that lifecycle task; keep frontmatter scheduling and propagation behavior                                        |
| Tasks affected only by project propagation                                                                       | Do not copy the summary onto them unless they were also explicit scheduling targets                                                |
| Ready, Blocked, closed tasks, ordinary non-task bullets, or no qualifying scheduling targets                     | Keep existing flow with no additional prompt                                                                                       |
| Cancel-only recommendation, Cancel row, lane row, refresh, dependsOn, property deletion, or unrelated properties | Keep existing flow; no scheduling Work Log stage                                                                                   |

Apply the prompt to accepted scheduling gestures regardless of whether the date is
future, today, past, or unchanged. The request is keyed to scheduling a Pending/Next
task, not just the future-date prune case. A typed work summary records actual work even
if the schedule itself is unchanged. Existing Schedule Log rules still decide whether an
automatic entry would be redundant.

For counted and linked sessions, use one optional shared summary, consistent with the
current shared schedule-reason and lane-release prompts. The preview must explicitly
name the qualifying count so it does not imply every selected task gets the entry.
Resolve/deduplicate repeated Task Links by target note and task identity so each
qualifying task receives at most one entry.

Use a distinct stage, for example `schedule-work-log`, titled `Schedule task` (pluralize
for a batch), with `Work summary` as its input label and placeholder
`What did you get done? (optional · ↵ to skip)`. Preview the dated entry, where it will
be saved, the frozen scheduling result or batch date span, and that nothing has been
written yet. Reuse established picker layout, focus behavior, and keyboard handling.
Warn on `::` as existing summary/reason previews do; normalize whitespace while
preserving Markdown. No new settings are needed.

Enter with text commits scheduling and one `*YYYY-MM-DD* — <summary>` entry per eligible
target; Enter with empty/whitespace-only text commits scheduling with zero Work Log
writes. A skipped Work Log never creates an empty marker and never adds the Schedule
Log's `🤷 no reason given` fallback. Escape or modal dismissal at either prompt discards
all pending state and performs zero writes. Both prompts remain independent: the
scheduling reason is not silently reused as the work summary, and automatic scheduling
reasons never count as work.

## Implementation sequence

1. **Prepare and retain one pending scheduling action.** Add a small explicit
   pending-action structure to `BulletPropertyPickerModal` that can resume the existing
   single/count/link/project dispatch after the final summary. Capture validated target
   identities and original statuses, task/note preimages, chosen value/level, Schedule
   Log payload, and the scheduling base date. For priority picks, materialize each
   target's independent roll once; for recommended actions retain the cached
   recommendation's exact dates, offsets, reasons, and cancel/skip decisions. Pass
   precomputed rolls/maps into the existing writers instead of rolling again after the
   prompt. Do not build a new transaction framework. Keep existing recommendation
   refresh behavior before the action is chosen, and preserve independent rolls and
   original priority reason text.

2. **Add the optional stage and route every scheduling entry point through it.**
   Explicit dates retain the Schedule Log reason payload, then enter the new stage when
   any target qualifies. Priority and recommended actions enter it directly without
   gaining a human schedule-reason prompt. No eligible targets means immediate dispatch
   through the existing path. The prompt-entry result must keep the modal open; only a
   successful final commit closes it. Add the pending action to `clearPendingBatch` /
   `onClose` cleanup. Guard asynchronous preparation and confirmation against duplicate
   Enter/click, cancellation during preparation, and late callbacks after dismissal.
   Confirmation must resume the prepared action once, without reentering either prompt.
   Discard obsolete pending state when returning to property selection after a failure.

3. **Compose Work Logs with scheduling in memory.** Thread an optional scheduling Work
   Log payload through every dispatcher/writer branch, carrying the normalized summary,
   a local-calendar date, and original eligible targets. Use existing Work Log insertion
   helpers; retain the direct-child ownership test so nested tasks' logs are never
   mistaken for their parent's. Prepend beneath an existing marker with its established
   indentation/list style, or append a new marker and entry after that task's existing
   children. Preserve the other children, both logs, legacy accepted markers, and line
   endings.

   In counted planners, compose Schedule and Work Log insertions bottom-up per original
   target, or maintain an explicit line map when applying the two log passes. Separate
   bottom-up passes that reuse stale task indices are unsafe. For mixed recommendations,
   map eligibility through Cancel Log and project frontmatter line shifts; preserve
   original-line conventions used for prune targets. Use the original `/` or `*` even
   when the scheduling postimage is Blocked. Work Log-only changes must count as a real
   planned write when a chosen date was already present. Carry accurate Work Log counts
   through outcome/notice builders; add a concise `N Work Log` chip only when entries
   were written, without losing existing roll/date/prune/freshness feedback.

4. **Commit through existing guarded transactions.** Revalidate the frozen source
   context before any final write, including target status, scheduling inputs, target
   children, and affected note preimages. A changed task, moved cursor/active file where
   the current guards require them to stay, modified linked note, or stale
   recommendation must refuse the complete action with a useful notice and no scheduling
   or log writes. Never carry a stale summary forward to a newly refreshed set of tasks
   automatically.

   Fold schedule, priority, Schedule Log, Work Log, and same-note daily cleanup into the
   existing one-step editor transaction. Build cross-note Work Logs into each target
   note's postimage and use the existing preimage and rollback core; do not perform a
   separate Work Log write after scheduling succeeds. Preserve the existing contract
   where a separate daily-note prune can fail after durable target edits and is reported
   without undoing those target edits. Recompute/remap cursor locations and cleanup
   against the final content when either log adds lines before the cursor or a later
   target. Preserve recurring-task refusals, configured property handling, Blocked
   derivation, sticky lanes, freshness stamping rules, closed-ledger history, and undo.

5. **Document, version, verify, and deploy.** Update `bob-cli/docs/projects.md` near the
   schedule-reason and Pomodoro-prune sections, and the navigation row in
   `bob-plugins/README.md`. Explain Pending and Next eligibility, prompt ordering,
   blank/escape behavior, shared batch summary scope, independent Schedule/Work Logs,
   and priority/recommendation behavior. Include one small Markdown example showing both
   managed logs. Bump the navigation manifest minor version (currently `1.51.0`, so
   `1.52.0` unless intervening work changed it), keeping README version/copy consistent.
   Add CSS only if established picker classes do not support the new preview. No Rust
   CLI, capture grammar, Mac Capture, keybinding, or durable memory changes are needed.

## Verification and acceptance

Extend existing meaningful planner and modal/runtime tests instead of relying on
helper-only coverage. Use fixed local dates and deterministic or counted random
generators so tests prove that opening/submitting the prompt does not consume extra
randomness or change the chosen schedule.

- Single Pending and Next tasks: explicit date -> schedule reason -> Work Log; no writes
  after either initial choice; typed, blank, and whitespace summary; Escape at each
  prompt; same-date work entry; Ready/Blocked bypass. Cover future-date Blocked/prune
  behavior and a today/past date with existing status recovery, showing that only the
  prior qualifying status controls logging.
- Scheduling paths: chosen priority, pinned roll, recommended roll, and decay from
  property and value stages; the exact date/offset and deterministic Schedule Log remain
  the ones chosen before the summary. Ctrl+Enter in the Work Log stage confirms rather
  than initiating another recommendation.
- Ownership/format: existing and missing Work Log, empty and legacy accepted marker,
  tabs and spaces, nested child tasks, preexisting Schedule Log and freeform children,
  Markdown, `::` warning, and CRLF preservation. Summary-only changes must persist when
  date/property edits are otherwise a no-op.
- Counted mixed-status batches: only explicitly scheduled `/` and `*` targets receive
  the shared summary; include multiple log insertions above later targets,
  project-frontmatter shifts, an explicitly targeted `^prj`, and propagation-only tasks.
  One confirmation and one undo step.
- Task Links: target logs are written in their source notes, not on the link bullet;
  cover multiple notes, duplicate links, and today's daily note as a target. Mixed
  recommended roll/decay/cancel/skip batches log only eligible scheduling targets.
  Cancel-only and recurring refusal behavior stay intact.
- Guard failures: target line/status, child log, frontmatter, linked-note preimage, and
  active source change while prompting; async dismissal and duplicate confirmation;
  later target-write failure rolls back scheduling and both logs. A separate daily prune
  failure retains committed logs/date and the existing failure notice. Same-note prune
  plus both logs is atomic.
- Existing Alt+N and Ctrl+Shift+Enter behavior remains covered by their current
  regression suites; no scheduling prompt leaks into other picker operations.

Run in the opened `bob-plugins` repository:

```sh
node --test scripts/test-navigation-hotkeys.cjs scripts/test-navigation-roll-decay.cjs scripts/test-navigation-stamps.cjs scripts/test-block-id-prompt.cjs
npm test
npm run validate
```

Review diffs in both repositories and run `git diff --check` in each. No bob-cli Rust
test run is required if its only change is Markdown documentation.

After verification, fulfill the linked repo's deployment instruction using the path
returned by `sase repo open`, assigned to `task_plugins_repo`:

```sh
bob plugins sync --no-pull --repo "$task_plugins_repo" --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo "$task_plugins_repo" --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo "$task_plugins_repo" --format json
```

Confirm the final list reports `bob-navigation-hotkeys` synced at the expected version;
sync's exit code alone is insufficient because dirty files may be skipped. Preserve the
dirty-file guard and report any deployment skip. Reload the plugin when an Obsidian
session is available and smoke-check a Pending and a Next task, blank submission,
Escape, and one Task Link/count batch. If this environment cannot drive Obsidian, report
that manual UI verification remains unperformed; do not claim the live keymap was
exercised. The implementation is accepted when every entry path follows the contract,
tests/manifest validation pass, docs agree, and deployment results are accurately
reported.
