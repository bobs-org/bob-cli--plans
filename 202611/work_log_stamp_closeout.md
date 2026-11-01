---
tier: tale
title: Finish Work Log clock semantics and land bob-cli-fd
goal:
  Navigation Work Logs use the commit instant, previews refresh their stamps, and
  bob-cli-fd is verified, deployed, and closed.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fd.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fd
status: done
---

- **PARENT:**
  [202611/work_log_timestamps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/work_log_timestamps.md)
- **BEAD:**
  [bob-cli-fd](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fd/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-fd.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-fd.land.md)
- **COMMITS:**
  - [885c560](https://github.com/bobs-org/bob-plugins/commit/885c560f66be3879a8599afcb7e015eaf5f6b1fb)
    — fix(nav-hotkeys)\!: sample Work Log stamps at commit/render time

# Finish Work Log clock semantics and land bob-cli-fd

## Scope and approved contract

Finish the remaining navigation-plugin clock defects, deploy the corrected plugin, and
close epic `bob-cli-fd` in this coding turn. This is one bounded repair, not a new
feature or a new epic. The original plan is `plan:202611/work_log_timestamps.md`; read
it with `sase artifact read` and read `bob-cli-fd` plus its landing notes before
starting. Open `bob-plugins` and `plans` through `/sase_repo`; use only the paths those
commands return.

The epic's decisions remain final: ISO `*YYYY-MM-DD HH:MM* — summary`, host-local plugin
time, minute truncation, no memory changes. Each gesture formats one commit instant and
shares it across all targets. Previews sample the clock when rendered; they may lag
until the next input, but must update on that input. Preserve legacy date-only entries
byte for byte, and preserve the day-only semantics of freshness, Rotten Log, Schedule
Log, and Cancel Log. No version or CHANGELOG edits in bob-cli.

## Audit already completed

The lander read all four phase beads and every note. `fd.1` is Mac commit `01c753b`,
`fd.2` is plugin commit `74ad555`, `fd.3` is CLI commit `50df0e41`, and `fd.4` is plugin
commit `072b25f`. The Mac stripper and its decode tests accept both stamp forms. Rust
formats once from `ClosePlanner::new(now)`, preserves the separate completion date, and
has yesterday-close and human-output coverage. The unlink and checkbox-close plugins
have shared stamp formatters and their planned tests. No epic commit edited memory. The
CLI commit has the required `!` subject.

Post-start drift reviewed through CLI `96155ed5`, plugins `7dbbb6d`, and Mac `01c753b`:
capture keep-moving, event warnings, drawer grammar, merged type planners, inbox
route-before-type, root display, and calendar-aware MEET starts introduce no new Work
Log writer or parser. The later plugin commits preserve the stamp changes. There are no
navigation source changes after `74ad555` at that snapshot. Recheck newer commits when
starting; integrate any newly landed caller that now needs stamps.

The lander's plugin `build:check` and 14 focused suites passed (lane-walk, lane-toggle,
task-link-lane, scheduling Work Logs, Rotten keeps, schedule review, project subtree,
refresh modal, unlink Work Logs/runtime/deletion, checkbox Work Logs, keep-moving, and
inbox-link runtime). These tests miss the timing failures below. Mac phase verification
was `just portable-check` on athena (1917 tests) and docs-check; the CLI phase passed
`just check` and docs-check. The repair still needs its own final verification.
`just check-full` is not authorized.

## Reproduced remaining defects

On plugins `7dbbb6d`, replace global `Date` in an isolated Node harness process with a
subclass whose no-argument constructor reads an adjustable instant. Start at local
2026-11-01 23:59, advance to 2026-11-02 00:01 while answering the prompt:

1. `toggleTaskLane` on `- [/] #task Working ^a1`, with `requestLaneReleaseSummary`
   advancing the clock before returning the summary, writes `*2026-11-01 23:59*`.
   `510-plugin-link-commit-and-lane.js` computes `workLogNow` before awaiting the
   summary and inbox route. The linked release counterpart in
   `520-plugin-lane-links-and-review.js` has the same order.
2. `toggleTaskLinkLane` on a daily-note `[[Tasks#^a1]]` whose target is In Progress,
   with `requestTaskLinkLaneSummary` advancing the clock, also writes 23:59.
   `525-plugin-task-link-lane.js` computes and passes a frozen stamp and `now` before
   the prompt, so its preview is frozen too.
3. Open a Task Card on an In Progress task, choose its lane release action, advance the
   clock, and call `confirmLaneReleaseReason`. It writes 23:59.
   `380-picker-cancel-and-lane.js` freezes `pendingLaneRelease.stampText` at opening and
   passes that value to the writer at confirmation; it overrides the new `now`.
4. Rerender `renderLaneReleaseReasonPreviewItem` or `renderSchedulingWorkLogPreviewItem`
   after advancing the clock and changing the summary. Both still show 23:59 because
   they read frozen pending-state stamps (`380` and `370-picker-schedule-work-log.js`).
   Scheduling's commit can be correct even while this preview is wrong.

The correct output for these commits and rerenders is `*2026-11-02 00:01*`. The existing
midnight test in `test-navigation-hotkeys-schedule-work-log.cjs` updates
`picker.fixedWorkLogNow` and covers a different commit route, so its pass does not
establish the other gestures' correctness.

## Implementation and regression coverage

1. Give navigation's gesture orchestration a consistent injectable clock seam. Keep
   fixed-date injection useful for existing tests, but production defaults must sample
   at render/commit time. In `510`, `520`, and `525`, sample and format once after
   awaited user input and routing are settled and before planning the committed writes.
   Pass that stamp unchanged to every note and target. Do not use a prompt-opening
   snapshot as the writer's default, and do not take a clock read per target. Preserve
   existing cancellation, stale-preimage, and review-walk behavior.
2. In `196-task-link-lane-modal.js`, `370`, and `380`, derive displayed stamps from the
   render-time clock. Remove frozen opening stamps from the production commit payload.
   The Task Card's inbox route can await another user choice: obtain its write stamp
   inside the eventual write callback (`455-picker-inbox-route-gate.js` and
   `540-plugin-decay-picker-and-cancel.js`), not before that wait. Audit the other
   scheduling callbacks for the same boundary while retaining one shared stamp per
   batch. Existing schedule choices and day-only fields keep their own established
   semantics.
3. Finish the stamp-only writer migration in `220-property-targets.js`, `230`, `245`,
   and their callers. `resolveSchedulingWorkLogStampText` still accepts
   `workLog.dateText`, and a compatibility `resolveSchedulingWorkLogDateText` wrapper
   remains; several lane helpers similarly treat bare `dateText` as a stamp. Use
   explicit stamp fields or the gesture clock for new Work Log writes. Update tests that
   still expect new date-only entries, such as the nonblank `planTaskLaneBatch` test in
   `test-navigation-hotkeys-lane-toggle.cjs`. Keep genuinely pre-existing legacy entries
   in fixtures. Do not change other uses of `dateText`, especially freshness and
   Schedule Log dates. Audit every `insertLaneWorkLogEntry` and `formatLaneWorkLogEntry`
   caller in `src/`.
4. Add regression tests through real orchestration and modal rendering, using the
   existing harnesses. Cover all three reproduced commit routes, the linked release
   route, render-after-input for the frozen previews, and a route gate delayed across a
   minute boundary. Include a multi-note/multi-target gesture with one commit-clock
   sample and identical stamps, plus blank/cancel behavior. Use both same-day minute
   advancement and midnight where existing guards permit the write; do not weaken any
   existing midnight refusal merely for these tests. Keep the reader/chip vectors and
   all four formatter vectors passing.

## Verification, deployment, and follow-up disposition

Edit source fragments only and keep each at or below 1000 lines. Run `npm run build`,
`npm run build:check`, the affected navigation and Work Log suites, and `npm test` in
bob-plugins. If changing docs, first read the docs memory and run `npm run docs:check`
with `BOB_CLI_DIR` pointing to the actual bob-cli checkout. Run `just check` in bob-cli
for file-change verification; never `just check-full`. Use `/sase_monitor` for
verification that requires a handoff and preserve this plan's closeout instructions in
its continuation. Do not require this turn's own commit SHA, push, or CI before closing
the epic.

Deploy `bob-navigation-hotkeys` with
`bob plugins sync --repo <opened-plugin-checkout> -p bob-navigation-hotkeys` and verify
the deployed build matches. This also finishes `fd.2`'s refused deploy. The other two
plugins were deployed by `fd.4`. A dirty-origin refusal is not proof that overwriting is
safe: inspect provenance/current bytes and preserve newer work before deciding on a
targeted force. Never force a blanket plugin sync.

Follow-up triage is owned by the lander and recorded on `bob-cli-fd` before this
proposal. `fd.4` note #1 has five failures, grouped by root cause into existing tasks:
`bob-cli-f1` (missing Chromium), `bob-cli-et` (gcal/freshness real-date tests), and
`bob-cli-eh` (note-ready/relation-index single-slot layout-ready fakes). The lander
reproduced all five and corroborated those tasks. No new task is needed. The retained
failure log is `file:explicit:844e735243fe4ec95a86291a` (read through
`sase artifact read`). `fd.2`'s extra wikilink performance failure is tracked by
`bob-cli-f8`; its isolated five-test suite passed in the landing checkout, so the lander
recorded that result without claiming a new reproduction. The active epic sweep found no
credible causal owner for these older infrastructure defects. Keep these outcomes, and
their proposing bead IDs, in the final close note.

## Final step: close bob-cli-fd in this turn

After the repair, verification, integration review, and targeted deployment succeed:

1. Read `bob-cli-fd` and all children/notes again; ensure every open descendant is
   complete. If this tale has its own open child bead, finish and close it normally
   first once all implementation and verification are complete, so it cannot block the
   parent close. Do not wait for the host to commit it.
2. Run `sase bead epic-symbols bob-cli-fd`. Resolve every listed exemption by wiring,
   privatizing, adding a valid non-test pragma, or deleting the symbol; re-key only to a
   genuinely still-open later bead that requires it. At the land audit there were no
   entries. Never defer this judgment or leave a closed bead on a whitelist.
3. Run `sase bead close bob-cli-fd --note "<verification>"`. The note must identify the
   four audited commits and child notes, the repaired prompt/commit timing defects and
   regression evidence, intervening-change integration, plugin deploy, final
   verification results, and all follow-up outcomes above. If rejected for symbols,
   finish cleanup and retry. If a phase is incomplete, complete or reopen it. Never use
   `--force` to make a successful nested landing advance; cancellation or supersession
   requires a real reason and the corresponding resolution.
4. Run `just symvision` if the recipe is available; otherwise explicitly record that
   this repository has no such recipe (none existed at the audit). Resolve any actual
   whitelist failures.
5. Through the opened plans repository, set `status: done` in the frontmatter of
   `202611/work_log_timestamps.md`, the epic's linked plan. Resolve its path from
   `sase bead read bob-cli-fd` rather than assuming a checkout location.
6. Run `sase bead read bob-cli-fd -r "Need the parent link"`. It had no parent at this
   audit, so no ancestor close is expected. If a parent was added, follow the original
   land prompt: close only a completed parent phase, or re-audit a parent plan's
   descendants, notes, linked plan, and drift before normal closure and plan status
   update; stop and note the first incomplete or ambiguous parent.
7. Finish through `/sase_final`, declaring all changed repositories for host commit. No
   separate land agent resumes after this tale: completing this closeout is part of its
   scope.
