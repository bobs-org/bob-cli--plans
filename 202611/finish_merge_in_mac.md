---
tier: tale
title: Finish and close the Bob Mac Capture merge-in epic
goal:
  Complete the remaining merge-in acceptance audit and repairs, record exact-tree
  verification, and close bob-cli-hl.5.1.
size: medium
proposed_by: bbugyi200.athena.19a
proposed_with: codex/gpt-6-astra
create_time: 2026-11-06 15:10:32
status: done
---

# Finish and close bob-cli-hl.5.1

## Outcome and tier

Complete the remaining acceptance audit for **bob-cli-hl.5.1**, repair any confirmed
merge-in regression or missing acceptance coverage, preserve durable verification
evidence, and close that existing epic. Its five implementation phases are already
closed and their code has landed. This is a **medium tale**: one coding/landing agent
can finish the bounded integration audit and closeout. Do not create a replacement
feature epic or restart the completed phases.

The user requested a validated proposal before implementation. This plan authorizes
implementation only after plan approval. No product files changed during planning. There
are no new product or memory decisions to ask the reviewer.

## Authority and inspected state

Read these through their audited commands at execution start:

- `sase bead read bob-cli-hl.5.1 -r "Review merge-in closeout state"`,
  `bob-cli-hl.5.1.5`, and `bob-cli-hl.5`; check all five child statuses again.
- `sase artifact read plan:202611/capture_merge_in_mac.md "Review the complete Mac acceptance contract"`.
- `sase artifact read plan:202611/capture_merge_in_arrow.md "Review inherited merge-in behavior and baseline policy"`.
- Read `sase_beads.md`, `sase_artifacts.md`, `decisions:mac-capture-is-a-thin-client`,
  and `decisions:similar-tasks-are-a-settled-heads-up` through `/sase_memory_read`.

Use `/sase_repo` to open `bob-mac-capture`, use only its returned checkout, and read its
`AGENTS.md`. All Swift paths below are relative to that repository. Bob-cli paths are
explicitly prefixed. Record both repositories' HEADs and initial dirty state. Integrate
current upstream changes through the normal workspace workflow before the final checks;
never erase another agent's changes.

Planning inspected clean Bob Mac Capture master
`cbb50dc909f61f6ce4aacae86b394d11b43e02d2`. It includes the five phase commits and the
lander's `e9d3ae2fdade28d031811b458c5f18633f38e023` repair: pending target identity,
refusal submit protection, spoken source transitions, settled announcements, hosted
tests, and documentation. Do not reapply those changes.

The last recovery response (`sase chat show --agent bob-cli-hl.5.1.land--4 -f response`,
using `/sase_chats`) left the epic open because run 42230645404 at snapshot
`6b9c715a9e61047a733950ca5c201f09397c0268` had two unclassified wait timeouts:

- `CaptureRelationDashAliasTests.testSecondDashRestoresDoubleDashForTheTabEmDashSnippet`.
- `CaptureSeparatorToggleTests.testUndoAfterHashAliasRestoresLiteralWithoutRetriggering`.

New evidence establishes a bounded closeout path. Planning fetched the full failure logs
and compared test identities **and each assertion message**, ignoring log timestamps and
source line numbers:

| Evidence                                                                                               | SHA                                        | Result                                                                                     |
| ------------------------------------------------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Pre-epic run [42161345004](https://github.com/bobs-org/bob-mac-capture/actions/runs/42161345004)       | `6f4e279`                                  | 16 failing test cases, 33 assertions                                                       |
| Landed repair run [42234819040](https://github.com/bobs-org/bob-mac-capture/actions/runs/42234819040)  | `e9d3ae2fdade28d031811b458c5f18633f38e023` | 14 failing cases, 30 assertions; all identities and assertion lists match the pre-epic run |
| Current master run [42278025697](https://github.com/bobs-org/bob-mac-capture/actions/runs/42278025697) | `cbb50dc909f61f6ce4aacae86b394d11b43e02d2` | 12 failing cases, 28 assertions; all identities and assertion lists match the pre-epic run |

Both previously unclassified tests pass in **both** later runs. Both runs also pass
`CaptureMergeInDecodingTests`, `CaptureMergeInPresentationTests`,
`CaptureMergeInPanelTests`, `CaptureMergeInDesignTests`, `CaptureGrammarGateTests`, and
`CaptureGrammarGateHostedTests`. Current master passed script checks, formatting, and
Build; its full test suite completed (4,287 tests, one skipped), without a watchdog
timeout. Bundle, signature/plist, launch, and install/reinstall steps were skipped after
Test failed. These results do not constitute a green full workflow.

The remaining 12 failures are the already tracked `bob-cli-h7` cluster:

- `CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched`.
- `CapturePanelModelTests.testCommaCommitsPartialRouteThenBobOpensParentTaskPicker` and
  `testCommaOnExactRouteUsesBobRewriteLane`.
- Nine `CaptureSeparatorToggleTests` methods:
  `testBackslashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses`,
  `testCommaAfterCompleteTargetRunOpensNextParentPicker`,
  `testDelayedAcceptedAliasSupersedesGatedCompletion`,
  `testNativeUndoRestoresLiteralSigilAliasWithoutRetriggering`,
  `testPeriodAliasAfterRouteOpensNewIDBlockPicker`,
  `testPeriodAliasOnMoveBulletOpensMovePicker`,
  `testSemicolonAliasOnOfferMoveBulletOpensMoveAndLinkPicker`,
  `testSlashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses`, and
  `testUndoAfterMoveAliasRestoresLiteralWithoutRetriggering`.

`bob-cli-hs` still owns the known README contract-table digest failure. The earlier
lander already triaged all phase follow-ups and recorded them on `bob-cli-hl.5.1`: `h7`,
`hs`, `ge`, `hu`, and `hv` are existing owners. Both
`sase bead epic-symbols bob-cli-hl.5.1` and `sase bead epic-symbols bob-cli-hl.5` were
empty during planning. Refresh rather than assuming any of this state is frozen.

## 1. Establish one acceptance and evidence ledger

Create a concise closeout report in the working checkout, then publish it with
`sase artifact create -p <report> -l "Merge-in Mac closeout evidence" --bead bob-cli-hl.5.1`.
Use `/sase_memory_read` for artifact rules first. Record the exact tested SHA/tree,
commands, exit results, CI URLs, render artifact names, and each acceptance area's test
or inspected implementation. Update the report after repairs and final checks; publish
the final version as a new immutable snapshot if necessary.

Audit the original plan's matrix against current code and tests. Start with:

| Area                      | Implementation and existing evidence to inspect                                                                                                                                |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Additive wire contract    | `CaptureModels.swift`, `CaptureMergeInModels.swift`, `CaptureRelatedModels.swift`; decoding tests and `capture-merge-in-*` fixtures                                            |
| Picker and explicit edits | `CapturePanelModel.swift`, picker helpers and editor; pending/revision, diagnostic rewrite, and ID-less cancellation/splice tests in `CapturePanelModelTests.swift`            |
| Band and outcomes         | `CaptureMergeInPresentation.swift`, `CaptureMergeInBandView.swift`, `CapturePanelView.swift`, notification integration; presentation, panel, notification, and design tests    |
| Landing block             | `CaptureTaskBlockPresentation.swift` and block-row views; contiguous-origin, unmatched-row, both-arrow and duplicate-target tests in `CaptureTaskBlockPresentationTests.swift` |
| Compatibility             | `CaptureGrammarGate.swift`, gate tests, `Tests/Fixtures/fake-bob`; absent/old/current/newer grammar and executable changes                                                     |

Map all original requirements, including vault/note/route pickers, comma continuation,
all three rewrite labels with Unicode and undo, stale requests/actions, source exclusion
from Similar tasks, mixed/task-list captures, label-driven merges, one aggregate submit,
explicit unselected Pull-In choices, reset/disclosure state, and settled accessibility
copy. Reuse meaningful existing tests. Add a regression only for a demonstrated missing
behavior or a material acceptance gap; do not duplicate implementation details in tests.

Review `docs/capture-previews.md`, `docs/task-pickers.md`, and `docs/keyboard.md`
against the shipped behavior. No changes are required solely to make the closeout
produce code.

## 2. Repair only confirmed merge-in gaps

If the audit exposes a gap, make the smallest correction in its owning layer with a
regression that fails before the fix. Preserve these boundaries:

- Bob owns syntax, eligibility, replacement bytes/ranges, merge effects, and consequence
  text. Swift owns presentation and request lifetime. Do not parse capture grammar or
  write the vault in Swift.
- `<` keeps ordinary `task`/`pomodoro_task` cards. Each merged item gets its band;
  mixed/bulk drafts retain the aggregate `Capture` action. Transfers retain their
  existing whole-card treatment.
- Refusals and incomplete input cannot submit. Rewrites require explicit actions, exact
  snapshot matching, and valid UTF-8 boundaries. Preserve normal Return behavior.
- Keep all blocking errors and warnings visible when source rows fold. Target blocks
  appear once; group only contiguous identical `origin` values, never matching titles.
- Preserve grammar gate 15 or the actual shipped compatible revision, schema 1,
  optional-field compatibility, pending target identity, source status transitions, and
  current Pull-In Done/Cancel behavior.

The two historical timeouts are currently passing; do not invent a fix or call them
confirmed flakes on the strength of different-tree runs. If either recurs, inspect its
request records and bounded wait conditions. Compare against the clean baseline and,
only when necessary, rerun the identical tree to distinguish an intermittent failure. Do
not weaken assertions, add arbitrary sleeps, suppress tests, or broaden this work into
repairing the entire `h7` cluster.

For documentation corrections, read `docs.md` first, edit the page that owns the
behavior, and keep `docs/protocol.md` unchanged. Do not edit memory, generated agent
instructions, Cargo versions, changelogs, or release configuration. Do not install the
app or use Bryan's vault. If a required correction unexpectedly changes the CLI contract
or exceeds this bounded closeout, record the concrete issue and replan before expanding
the architecture.

## 3. Verify the final integration and inspect its renders

Run `just portable-check` in Bob Mac Capture and `just docs-check` in both repositories.
Run focused tests while developing; the final integrated check must cover the current
tree, not an earlier intermediate snapshot. Run `just check-scripts` if a generator or
script changes. If bob-cli implementation changes prove necessary, run its `just check`
as well as its documentation gate.

Never land uncompiled Swift. For changed `Sources/BobMacCapture*` or
`Tests/BobMacCapture*`, read `tailnet.md`, then follow the linked `AGENTS.md` for a
best-effort scratch-copy Mac compile: exclude `.git`/`.build`, run `Scripts/bundle.sh`
and `just typecheck-tests`, and remove the scratch copy. Do not build in Bryan's
checkout or install there. If unreachable, record that the scratch check was unavailable
and use exact-SHA hosted macOS results to describe what was verified.

For any changed Swift tree, run `Scripts/macos-ci-check` before landing. Watch every
pushed SHA through `/sase_monitor`. Use that skill for long checks and CI waits; wait
for the monitor-start command itself to exit and finish the turn through `/sase_final`.
Never end with an unmonitored subprocess still running. With an unchanged tree, the
existing completed exact-SHA run can supply hosted evidence; retrieve and inspect its
logs and artifacts instead of rerunning solely to obtain a different verdict.

Inspect render artifacts from the final tree at 620 and 760 points, light and dark. At
minimum review long/duplicate titles, five-source disclosure, the folded-source warning,
pending target plus resolved/error rows, all three native rewrite buttons, and
collapsed/expanded origin groups for both arrows. Review actual images with native
controls where provided; successful image generation alone is insufficient. Check
keyboard/accessibility behavior in hosted tests. Repair clipping, hidden warnings, stale
actions, or duplicate blocks, then recompile and repeat the affected checks.

Use the **already approved original plans' baseline-only exception**, not a new blanket
CI waiver. The parent Mac phase says pre-existing failures are not this phase's work
when no new failures are proved; the child plan requires exact baseline evidence. Record
a green full workflow when available. If it is red:

1. Enumerate every failed test, assertion, and gate from the final exact-SHA run.
2. Compare with the clean pre-epic evidence and current owning beads. A matching suite
   name alone is insufficient. A timeout is not automatically a known failure.
3. Fix every new merge-in regression. An unexplained failure or an incomplete/hung run
   cannot be used as a passing closeout receipt.
4. Only established baseline failures may remain under that existing exception.
   Explicitly list their owners, and list skipped packaging/install steps as unverified.
   Do not alter CI to bypass Test or describe the red workflow as green.

For the known docs failure, compare the exact
`test_readme_table_matches_original_contract_index_snapshot` assertion with
`bob-cli-hs`; any additional docs failure needs investigation. Preserve existing
follow-up ownership. Use `/sase_new_task` before recording any genuinely new follow-up;
check duplicates and active epics first. A new bead is never a substitute for fixing an
in-scope regression.

## 4. Complete the existing bead lifecycle

Before closure, the report must demonstrate all original acceptance areas, successful
merge-in tests and render inspection on the integrated tree, and either full green
checks or only the specifically proven baseline exceptions above. Confirm all five child
phases are closed, all proposed follow-ups are triaged, and no epic-symbol exemptions
remain. Resolve each exemption or re-key it to a valid open owner.

Attach the final report and append a concise completion note to **bob-cli-hl.5.1** with
commit/tree, checks, run URLs, render review, and any baseline-only limitations. Then
close the requested epic with
`sase bead close bob-cli-hl.5.1 --note "<concrete evidence>"` as the recovery closeout
owner, or use its assigned primary finalizer's `bead_action: close` if that is the
host-owned closure path. Never force-close unfinished descendants. If approval creates
an additional descendant bead, finish that work through its normal lifecycle before the
existing epic's close; do not force around the descendant guard.

The original Mac epic explicitly delegates completion of **bob-cli-hl.5** to its landing
owner. After `.5.1` is actually closed, re-read `.5`, verify it has no other open
children or exemptions, and complete that delegated phase if the current recovery owns
its landing lifecycle; otherwise leave the evidence for its assigned lander. Do not
close **bob-cli-hl** or unrelated CI/task beads. Re-read the requested bead to confirm
its closed/done state and report that state with the verification limitations.

Finalize all repository changes through `/sase_final`. If the host will perform the
close after finalization, describe it as submitted for host closure until the resulting
state is observable. A plan submission, passing portable tests, or the presence of five
closed children alone is not completion of this closeout.
