---
tier: tale
title: Verify and close the landed macOS caret-underline repair
goal:
  Close bob-cli-dw.3 with verified macOS CI evidence for the existing repair and
  preserve the unrelated relation-row failure for epic-land triage.
size: small
proposed_by: bbugyi200.athena.bob-cli-dw.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-dw.3
create_time: 2026-10-28 15:45:18
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b6.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b6.md)
- **BEAD:** bob-cli-dw.3

# Verify and close the already-landed macOS caret-underline repair

## Objective and scope

Complete assigned phase `bob-cli-dw.3` of `plan:202610/backlog_batch_b6.md`. The source
repair has already landed and passed the required macOS CI. This tale is a bounded
verification and bead closeout, not a second implementation. No source, workflow, docs,
memory, or version changes are expected. No reviewer decisions are needed: the parent
epic authorizes no memory work and has no unresolved decisions.

The phase was in progress when planning began, with no earlier implementation or
unfinished-work notes. Planning added two notes: verified landed evidence and a
`PROPOSED FOLLOW-UP:` for an unrelated failure in the latest master run. There is no
unfinished coding remainder requiring a child epic.

## Verified evidence

Read the assigned phase and original task `bob-cli-dr` with `sase bead read`, and the
parent design through `sase artifact read`. Opened `bob-mac-capture` using `/sase_repo`,
read its `AGENTS.md`, and inspected its local history and source.

- The initial runs 40724077224, 40728493998, and 40729693701 failed the model and hosted
  caret tests. Job 122962340765 from run 40729693701 confirms the failures are the
  **return-to-name clearing assertions**, after successful underline appearance. They
  are not timeouts waiting for the first underline.
- `4053edb8dac2db59dbcb0b04b4a56b03de4d992a` tried explicit nil clearing.
  `2c3407d6c096bbc7617313b63434dcae7cd2654b` then made the effective repair:
  `CapturePanelModel.paintHighlighting` overwrites the theme range with
  `NSUnderlineStyle([])` while the caret is in the name, and clears underline color.
  This preserves the async selection-triggered analysis path and its event-time caret;
  selection callbacks still do not rewrite the editor.
- [Run 40746228778](https://github.com/bobs-org/bob-mac-capture/actions/runs/40746228778),
  job 123020643000, is a successful **master** run for full SHA
  `2c3407d6c096bbc7617313b63434dcae7cd2654b`. Its `macOS 26 SwiftPM` job passed all
  3,265 tests, including:
  - `CapturePanelModelTests.testCaretLeavingUnconfiguredThemeUnderlinesAfterAnalysis`
    (0.067 seconds).
  - `CaptureUnconfiguredThemeHostedTests.testTypingUnconfiguredThemeThenMovingCaretUnderlinesWithoutFreezing`
    (1.351 seconds).
  - `CapturePanelModelTests.testCaretMoveWithUnconfiguredThemeDoesNotRewriteEditor`.
  - `CaptureEditorHighlightTests.testUnconfiguredThemeHasNoMarkWhileCaretIsInTheName`.
    Build, Bundle, Verify plist and signature, Launch smoke test, and Install and
    reinstall also passed. This is the actual Mac CI signal required by the phase.
- Current inspected master is `c8c4366d897baab836b4f05693aede4ca0a01a51`, a descendant
  of the repair. Its only model changes concern relation-card rows, not caret
  highlighting.
  [Run 40750655496](https://github.com/bobs-org/bob-mac-capture/actions/runs/40750655496),
  job 123036129488, again passes both target tests (0.108 and 1.405 seconds), the
  no-selection-write regression, and the direct painter test.
- The latter run has **one unrelated failure among 3,267 tests**:
  `CaptureRelationPanelTests.testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible`,
  `CaptureRelationPanelTests.swift:1566`, expects `Optional("1 link")` but gets
  `Optional("one-off · 1 link")`. The failing test was added by c8c4366. It is a
  clean-base CI failure, with no changes made by this phase. It is already recorded as a
  proposed follow-up on `bob-cli-dw.3`; the epic land agent should triage it against
  `plan:202610/relation_card_refused_cancel_rows.md`. Searches for `RefusedCancel` and
  that plan name found no tracking bead at planning time.
- `sase bead epic-symbols bob-cli-dw.3` reported no entries. Both source checkouts were
  clean. `sase tool list` in bob-mac-capture reports no named tools; invoking its
  nonexistent named `check` is not a substitute for the observed workflow.

## Execution

1. Use `/sase_memory_read` for bead/artifact rules, then re-read
   `sase bead read bob-cli-dw.3 -r "Verify the landed caret repair and closeout notes"`
   and the parent design through `sase artifact read`. Use `/sase_repo` to reopen
   `bob-mac-capture` and read its `AGENTS.md`. Do not hand-set bead status or reopen an
   already-complete phase. If it was closed by another authorized worker, corroborate
   the existing evidence instead of manufacturing work.

2. Recheck the repair's ancestry and inspect any new master changes. Verify the full
   SHA, completed job/step conclusions, zero-failure summary, and both target test pass
   lines in run 40746228778. Reconfirm the target tests on the current inspected master
   run 40750655496, or a newer run if master has advanced. Use available GitHub run/job
   tools or `gh run view`; repository source/history must still be read from the
   checkout opened through `/sase_repo`. During planning, the connector's
   commit-workflow lookup returned empty arrays even for full SHAs, so
   `gh run list -R bobs-org/bob-mac-capture --branch master` supplied run identities and
   the connector's run-job/log calls supplied evidence.

3. Leave the already-verified source repair intact. The recorded successful full macOS
   workflow covers unchanged Swift and its hosted tests; do not replace it with Linux
   portable results or claim the newer unrelated red run is green. No fresh build is
   necessary for this evidence-only closeout. If new evidence actually requires Swift
   edits, compile them before landing: follow `AGENTS.md` (`just portable-check` on
   Linux, best-effort scratch Mac bundle and `just typecheck-tests`, plus actual
   `macOS 26 SwiftPM` execution of changed targets). Use `/sase_monitor` before starting
   long commands or CI waits. Never run `just check-full`, edit Bryan's Mac checkout, or
   install there.

4. Preserve the existing proposed follow-up note for the unrelated relation-row failure.
   Add only new corroborating evidence if needed; do not duplicate the note or create
   task beads. A clean-base failure outside the caret scope does not keep this phase
   open. Do not repair the relation behavior in this tale.

5. Immediately before closing, run `sase bead epic-symbols bob-cli-dw.3` again. Resolve
   any newly appeared symbols or re-key their Justfile entries to a still-open owner
   before proceeding. Then close **only** `bob-cli-dw.3` with
   `sase bead close bob-cli-dw.3 --note "<verified commit, Mac run, target test and full workflow results, plus latest-master unrelated failure disposition>"`.
   Leave `bob-cli-dr`, `bob-cli-dw`, and all ancestor plan beads to the epic land agent.
   Do not mark the parent epic design done.

6. Use `/sase_final` as the last action before the normal final response. Report that
   the repair was already landed, link the successful Mac run, and distinguish the
   unrelated current-master failure. Do not claim to have implemented changes.

## Acceptance

- Both named caret tests have real macOS CI pass evidence for the existing repair, and
  the repaired master commit has the complete successful packaging/launch/install
  workflow required by the original task.
- The later clean-base relation assertion is recorded for epic-land triage.
- No redundant implementation or unauthorized memory changes are made.
- No epic-symbol entries remain assigned to this phase, and only `bob-cli-dw.3` is
  closed with concrete evidence.
