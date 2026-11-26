---
tier: tale
title: Finish Mac snooze scheduling and land bob-cli-q5.7.1
goal:
  Preserve authoritative wake outcomes and bounded retries, render the folded alarm
  icon, verify the combined Mac tree, and close bob-cli-q5.7.1 and parent phase
  bob-cli-q5.7.
size: medium
proposed_by: bbugyi200.athena.bob-cli-q5.7.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-q5.7.1
create_time: 2026-11-26 10:57:48
status: wip
---

- **PARENT:**
  [202611/mac_snoozed_agenda.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_snoozed_agenda.md)
- **BEAD:**
  [bob-cli-q5.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.7.1.md)

# Finish Mac snooze scheduling and land bob-cli-q5.7.1

Finish the remaining work found while landing **bob-cli-q5.7.1**, then complete its
landing and close its containing phase **bob-cli-q5.7**. This tale has no separate land
agent: its coder owns the final closeout below. One coder can make the bounded scheduler
and view changes directly. Do not create another epic or defer closeout until this
work's own commit, push, or post-commit CI. Use the precommit macOS gate on the final
working tree, then close in the same provider turn that finalizes the verified code.

## Context already verified

Read `sase bead read bob-cli-q5.7.1 -r "Need the landing audit and remaining work"` and
its notes. Read `plan:202611/mac_snoozed_agenda.md` and `plan:202611/snoozed_tasks.md`
through `sase artifact read`. Open bob-mac-capture with `/sase_repo`, and read its
`AGENTS.md` before touching it. Product paths below are relative to that opened checkout
unless prefixed with `bob-cli:`.

The lander reviewed all three closed children and every note, their commits `c4a30591`,
`e387187f`, `e4a97f52`, and `a4a7020d`, and the actual models, fixtures, presentation,
folding, views, measurement, process client, lifecycle, and tests.

- Optional decoding, unknown-state fallback, `snooze` span tint, same-day filtering,
  separate budget counts and lineup numbers, first-to-fold persistent summary, and
  sixty-minute wake accents are implemented. The four `agenda-snoozed*.json` files
  exactly match Bob's fixtures.
- The earlier height and UTC-countdown failures were fixed by `e4a97f52`. CI run
  [45847440694](https://github.com/bobs-org/bob-mac-capture/actions/runs/45847440694)
  passed all required macOS steps. Its snapshot
  `f923059807e9efa942476d1a51bb7388d26c9efe` and Mac master `a4a7020d` have the same
  tree, `b52b21dbc584c2217c5613fe0aa06f9c4dbb8189`. Phase .3 also records 2,837 portable
  tests, one skipped, scratch bundle/typecheck, and docs-check passes. New edits still
  need their own verification.
- Drift since the first epic commit consists of Mac `aa233750` and `38dc0dee`; since
  creation, also Mac `6ea5b8f6` and Bob `4dd787bc`. These repair drawer fixtures and
  tests and add no competing agenda/wake consumers. The fetched primary and Mac master
  branches were current at those reviewed heads. Repeat the drift check before closeout
  and incorporate any new relevant changes.
- Follow-up triage is **finished**. The only `PROPOSED FOLLOW-UP:`, phase
  **bob-cli-q5.7.1.2 note 1**, concerned the place-menu test's literal “3 weeks ago”
  becoming “4 weeks ago.” `/sase_new_task` searches across types/statuses, the recent
  task sweep, and active-epic review found no task needed: `aa233750` and `38dc0dee`
  already repair the assertion and compile it, and the combined tree passed CI. The
  lander recorded **declined as resolved** on bob-cli-q5.7.1. Preserve that outcome and
  the proposing bead in the final close note; do not file or reopen this issue.
- `sase bead epic-symbols bob-cli-q5.7.1` returned no entries at audit time. Recheck
  during final closeout.

## Governing rules

Honor `decisions:mac-capture-is-a-thin-client`,
`decisions:idle-capture-shows-ledger-agenda`, and
`decisions:today-is-read-from-the-ledger`. Bob owns vault writes, task identities,
blocking, wake destinations, and host pinning. Swift owns orchestration and rendering.
Keep the command exactly `bob snooze wake -q -f json`, on its independent,
non-cancelling lane. Retain additive schema-1 decoding and precise old-Bob rejection.
Preserve the eye line, measured layout, existing folding order after the snoozed band,
and separation from Today/budget/close numbering.

The outer plan choices remain final: ignore uncommitted rows; notification delivery uses
a HUD and holds during meetings, owned by **bob-cli-q5.8**. This tale creates no notice,
claim, receipt, or notification policy. It changes no memory, release version, or
CHANGELOG. The authorized glossary work belongs to the outer epic's closeout.

## 1. Preserve wake outcome provenance and retry limits

The defects are in `Sources/CaptureCore/CaptureSnoozeWake.swift`,
`Sources/BobMacCapture/SnoozeWakeScheduler.swift`, and the new snooze method in
`Sources/CaptureCore/BobProcessClient.swift`.

The lander executed the unmodified production policy with only DTO/protocol stubs:

1. Update a snapshot for `2026-08-28`, target `2026-08-28T09:00:00-04:00`, at 09:00:05.
2. `beginAttempt` returns true.
3. At +1 second, settle with `nextWakeAt: nil`; repeat separately with a future
   `2026-08-28T19:00:00-04:00` target.
4. At +2 seconds, update with the same old snapshot and call `beginAttempt` again.
5. It returns **true in both cases**. The old snapshot overwrites the settled target and
   clears the retry deadline. `refreshScheduleFromSnapshot` performs this update on
   timer and lifecycle callbacks, so this is reachable beyond a unit-test call.

There is also an actual command-contract interaction. Inspect
`bob-cli:src/native/snooze/cli.rs` (`wake` and `wake_report`): deferred and wrong-host
skipped replies can carry the _prospective_ postimage `next_wake_at`, including null,
although no write happened. The current Mac adapter treats all result statuses alike. A
deferred/null reply therefore disarms the only retry, and the required refresh can be
byte-identical and publish no snapshot. Existing scheduler tests cover deferred only
with the original non-null target.

Implement a coherent policy for snapshot facts versus wake outcomes:

- Recomputing a timer from retained facts must not count as a new authoritative
  snapshot. Track enough snapshot/attempt provenance to retain an authoritative
  successful outcome through clock/wake callbacks and unchanged or failed refreshes. A
  completed null target stays retired; a returned future target stays scheduled.
- A genuinely newer authoritative agenda snapshot can retime, drop, or add work,
  including a new task at the same clock target. Avoid permanently suppressing that
  legitimate case by remembering only a timestamp forever. Unrelated agenda changes must
  not erase the same-target retry deadline.
- Treat `deferred` and `skipped` as non-writing outcomes. Preserve the previously due
  target and its reserved retry deadline until fresh agenda facts retire or retime it;
  neither a speculative null nor future postimage proves that due work was processed.
  Unknown statuses should not assert a successful write. Transient errors likewise
  retain bounded retries. Keep the minimum 60-second interval for repeated attempts on
  the same target, while a genuinely new future target arms normally.
- Continue to refresh the agenda after every settled attempt, including non-writing
  outcomes and failures. Consume Combine-emitted values, never willSet-stale reads. One
  writer may run at a time, with at most one pending follow-up. Timer cancellation,
  hide, Agenda off, or reconfiguration never kills an already-running writer.
- Day rollover must invalidate the old day's facts and in-flight result authority. An
  old callback cannot re-arm yesterday's timer. Vault/executable changes likewise
  invalidate callbacks. Cache unsupported-command results against the executable
  identity **captured at launch**: `snoozeWake()` currently stores `identityToken` again
  in its error path and can poison a replacement executable. Preserve Recheck Bob and
  executable-replacement recovery, and normal bounded process cleanup at quit.

Keep this correction in the Mac client; Bob's existing status is enough to distinguish a
non-write. Do not expand the tale into a new Rust command contract. Reuse the shared
offset-aware parser where appropriate and retain target+5-second scheduling.

Add meaningful regressions in `CaptureSnoozeWakePolicyTests.swift`,
`SnoozeWakeSchedulerTests.swift`, and `BobProcessClientTests.swift`, extending
`Tests/Fixtures/fake-bob` only as needed:

- The exact two-second reproductions above must fail on the audited implementation and
  pass after the fix. Cover successful null and future outcomes with unchanged and
  failed refreshes, plus lifecycle callbacks before 60 seconds.
- Deferred and skipped replies with null/future advertised postimages keep due work
  retrying at the 60-second deadline, never immediately. Then a changed agenda without
  due work cancels that retry. Ordinary successful null settlement disarms.
- Interleavings: unrelated snapshot publication during a write, retime/drop while in
  flight, fresh same-target work after retirement, forward/backward clock jumps,
  midnight during a write, and old-vault/executable results arriving late.
- Unsupported old Bob remains disabled until recheck/replacement; a rejection from the
  old executable does not disable the replacement. Preserve hidden-panel and Agenda-off
  behavior, exact argv/lane isolation, and no overlapping or cancelled writer. Use
  injected time and bounded async waits, not real minute-long sleeps.

## 2. Render the folded alarm as an icon

In `Sources/BobMacCapture/CaptureAgendaView.swift`, the `.snoozedSummary` branch calls
`chipButton(text: "alarm ...")`, and `CaptureAgendaChipButton` renders `Text(text)`. The
shipped collapsed band therefore literally says “alarm 2 later …”. Render
`Image(systemName: "alarm")` beside the summary text, retaining its button action,
accessible state/count/next-time label, subdued styling, and full-to-summary expansion.
Use a narrowly scoped chip extension or summary view; keep existing chip appearance.
Measurement must use the same view so icon width and row height agree.

Cover full, folded, expanded, and snoozed-only layouts at 724/584 widths with the
existing 2-point height tolerance. Inspect synthetic light/dark renders containing
asleep, due, blocked, fresh-woken, and long-title rows. Confirm the collapsed chip has
an actual alarm icon and remains visible under a tiny budget. Record reviewed visual
evidence through the artifact workflow; do not silently substitute screenshot generation
for inspection.

## 3. Verify the final working tree before closeout

Update `docs/idle-agenda.md` and `docs/develop/architecture.md` only where needed to
explain retry/retirement behavior; keep command/optional-field details consistent in
`docs/protocol.md`. Use `/sase_memory_read` for `docs.md` before documentation edits.

Follow bob-mac-capture's `AGENTS.md` and the original child plan's gates:

1. Run focused regressions, then `just portable-check` on the combined final tree. The
   linked Mac repo has no `just check` recipe at audit time; its documented portable
   gate is the applicable file-change check. If Bob source changes become necessary, run
   **`just check`** there. Never run `just check-full`.
2. Read tailnet memory, compile a Mac scratch copy with `Scripts/bundle.sh` and
   `just typecheck-tests`, and remove it. Exclude `.git` and `.build` during transfer;
   never use Bryan's live working checkout or install the scratch app. If the Mac is
   unavailable, record that and still require successful app compilation in CI.
3. Run `just docs-check` in bob-mac-capture with `BOB_CLI_DIR` pointing to this coder's
   primary Bob checkout, avoiding the wrapper's external clone fallback.
4. Run `Scripts/macos-ci-check` from the opened Mac checkout through `/sase_monitor`. It
   verifies the final working tree through a throwaway snapshot before the actual work
   commit. Wait for the monitor **staging command** to exit, then finish through
   `/sase_final`; do not wait for a staged monitor in the same turn. Its continuation
   must resume this tale at the closeout below after a green result. Require the exact
   snapshot's lint/build/test/bundle/signature/launch/install gates. Any subsequent
   source edits require re-verification; record the SHA, run URL, and tree identity.

The earlier baseline CI pass is useful evidence, not a waiver for new code. Resolve
epic-caused failures before closing. Route genuinely unrelated discoveries through
`/sase_new_task`, recording the outcome on bob-cli-q5.7.1. A `just check` pass with a
`just check-full` failure is infrastructure work, never remaining scope for this epic.

## 4. Finish this landing and the containing phase

This is the tale's final step; no other lander will resume it. Complete it against the
verified final working tree **before** the host commits this tale's code. It must not
wait for that commit's SHA, push, or post-commit CI.

1. Re-read bob-cli-q5.7.1 and all its descendants/notes with audited `sase bead read`.
   Confirm the three original phases remain closed, the two landing findings are
   addressed, and later drift integrates. Check its linked plan readiness using
   `sase artifact read plan:202611/mac_snoozed_agenda.md`. Preserve the resolved
   place-menu follow-up outcome above in the close note.
2. Run **`sase bead epic-symbols bob-cli-q5.7.1`**. Resolve every listed exemption by
   wiring, privatizing, using a valid non-test pragma, or deleting it under the
   Symvision policy. Re-key only when a genuinely still-open later bead needs the
   exemption. Leave none keyed to this epic or its closing phases.
3. Close normally:
   `sase bead close bob-cli-q5.7.1 --note "<verified original scope, scheduler repairs, summary icon, drift integration, exact verification evidence, and resolved bob-cli-q5.7.1.2 note-1 follow-up>"`.
   If rejected for symbols, clean them and retry. If named descendants are unfinished,
   finish or reopen them; never force a successful nested landing. `--force` is only for
   a deliberate canceled/superseded outcome with the required reason and resolution.
4. Run **`just symvision`** in each relevant checkout where that recipe exists. The
   audited Bob and Mac justfiles have no such recipe; check availability again and
   explicitly record absence if still unavailable. An available failing recipe must be
   resolved.
5. Open the plans repository with `/sase_repo`. In the linked plan
   **`plan:202611/mac_snoozed_agenda.md`**, path **`202611/mac_snoozed_agenda.md`**
   relative to that opened plans checkout, set frontmatter **`status: done`**, retaining
   other metadata. Use the PLAN path returned by the bead/artifact tools; never guess
   another workspace's path. Include this edit in finalization.
6. Run `sase bead read bob-cli-q5.7.1 -r "Need the parent link"`. Its parent is phase
   **bob-cli-q5.7**, not a plan. Read that phase and confirm this child plan satisfies
   its full Mac agenda scope: optional facts/span tint, dim Later today with first-fold
   persistent summary, separate snoozed count, expiring wake accents, and an always-on
   +5-second non-cancelling waker with bounded retries. Run
   `sase bead epic-symbols bob-cli-q5.7` and resolve/re-key any entries first, then
   `sase bead close bob-cli-q5.7 --note "<verified completed Mac agenda scope and this child's final passing evidence>"`
   normally. Leave **bob-cli-q5** to its already-waiting land agent; do not mark the
   outer snoozed_tasks plan done.
7. Finish through `/sase_final`, declaring commits for every repository changed,
   including the plans checkout. Report the closed epic and phase, the checks, and any
   actual limitation. Never end with a running foreground process.
