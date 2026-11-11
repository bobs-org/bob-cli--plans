---
tier: tale
title: Verify the landed alias-tip timeout fixes
goal:
  Complete bob-cli-kj.5 with current CI evidence and disposition of separate failures.
size: small
proposed_by: bbugyi200.athena.bob-cli-kj.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-kj.5
create_time: 2026-11-11 17:47:11
status: wip
---

- **PARENT:**
  [202611/impact_ranked_bead_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/impact_ranked_bead_fixes.md)
- **BEAD:**
  [bob-cli-kj.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kj/bob-cli-kj.5.md)

# Verify the landed alias-tip timeout fixes and complete bob-cli-kj.5

## Objective and scope

Complete phase `bob-cli-kj.5` of `plan:202611/impact_ranked_bead_fixes.md` by verifying
the existing fix for the three deterministic alias-tip timeouts, checking the named
intermittents for a shared cause, and recording accurate closeout evidence. The
implementation already landed in linked repository `bob-mac-capture`; do not reimplement
it. Preserve every assertion and the first-frame tip contract.

This is a small tale because the remaining work is a bounded source/evidence audit and
CI verification, not another feature implementation. The phase had no prior work notes
when read on 2026-11-11. Its dependency `bob-cli-kj.1` is closed; its teardown fix
landed as `1e83c032217a9d6c90827ee467e5c60b57069d88`.

Read `bob-cli-kj.5`, `bob-cli-jx`, and `bob-cli-jk` through `sase bead read` at the
start. Honor the parent epic's final DECISIONS. None requires memory or documentation
changes for this phase. Do not change memory, release versions, or changelogs.

## Findings already verified

Open `bob-mac-capture` with `/sase_repo` and use only its printed path. Read its
`AGENTS.md`. The planning checkout was clean at
`1e83c032217a9d6c90827ee467e5c60b57069d88`.

Commit `ee26941dedaeff5cbfae6640daee18000b668e84`, already an ancestor of that HEAD,
contains the relevant repairs:

- `Tests/BobMacCaptureTests/CaptureAliasTipPanelTests.swift` enables
  `FAKE_BOB_ALIAS_TIP_ANCHOR=1` in both route-query/filter tests. The fixture branches
  had been conditional since `aafed83`, but these tests had not enabled them.
- `Tests/Fixtures/fake-bob` serves `task-type-complete-incomplete-marker.json` for
  `jot idea #`, giving the option-typed hash test the intended completion surface.
- `Sources/BobMacCapture/CapturePanelModel.swift::isRecordCompletion` no longer
  classifies an ordinary route list as a Record Card merely because a candidate has kind
  `record`. This preserves the route completion surface on which the tip is anchored.
  The route tests additionally assert that generic record fallback is off; the filtering
  test still requires the record candidate.

The alias-tip test file has no changes between `ee26941` and planning HEAD. The fix
added assertions rather than weakening them.

Evidence retrieved directly from GitHub Actions during planning:

| Run / SHA                                                                                                                       | Relevant result                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [43018693025](https://github.com/bobs-org/bob-mac-capture/actions/runs/43018693025), `ee26941dedaeff5cbfae6640daee18000b668e84` | Fully successful workflow: Build, Test, Bundle, Verify plist and signature, Launch smoke test, and Install and reinstall all executed successfully. Test log: 4,710 tests, one skipped, zero failures. The entire alias-tip suite passed, including all three deterministic tests and all three named intermittents. macOS job ID `130516138966`.                                                                                          |
| [43104946525](https://github.com/bobs-org/bob-mac-capture/actions/runs/43104946525), `6cbbddf6eb9192e8d20ed17cb0206fc257a91102` | The deterministic trio passed again: filtering 0.338 s, option-typed hash 0.284 s, typing query before open 0.325 s. Changed-separator and second-eligible-sigil tests also passed. Edit-before-reply failed at line 336 (`nil` versus `Optional(1)` for the `sigilEdited` drop counter). Three notebook/drawer test cases supplied the other seven assertion failures. Later application gates were skipped. macOS job ID `130805719698`. |
| [43112612175](https://github.com/bobs-org/bob-mac-capture/actions/runs/43112612175), `1e83c032217a9d6c90827ee467e5c60b57069d88` | Still running when inspected; lint and build passed, Test was in progress. This is the prerequisite teardown commit's exact-SHA run, not a new run launched by this planner. Read its final result before deciding whether another run is necessary.                                                                                                                                                                                       |

Thus the bead's historical claim that the trio fails on every run is no longer true. Do
not assume a red workflow still indicates these three failures.

## Work

1. **Confirm the fix is still present.** Inspect the three changes above in the current
   linked checkout and establish ancestry of `ee26941`. Verify that the original
   assertions and anchoring ranges remain intact. Inspect the final logs of the latest
   run covering that exact tree; retain run URL, SHA, individual test outcomes, and
   required step outcomes in the phase's notes. Do not alter source merely to produce a
   new commit.

2. **Classify the named intermittents.** Examine `testEditBeforeTheReplyDropsTheTip`,
   `testChangedSeparatorReplyDoesNotDisableTheNextTipRequest`, and
   `testSecondEligibleSigilInASessionShowsNoTip` against the landed fixture and
   completion-routing causes. Passing once does not establish that a flake is fixed.
   Record each as shared-and-fixed only with causal evidence; otherwise record that its
   separate cause remains unresolved. The edit-before-reply failure is already
   reproduced on the unchanged phase base in run `43104946525`. A distinct hypothesis
   visible in source is that `editorTextDidChange` marks an in-flight request
   `.sigilEdited`, but `handleMissingCompletion` can call
   `deferAliasTipOpportunity(.unanchored)` and overwrite that reason before the gated
   rewrite reply arrives. Inspect this ordering; do not report the hypothesis as a
   confirmed runtime diagnosis without evidence. It is separate from missing query
   fixtures, hash completions, and record-list classification.

3. **Verify the current SHA and record unrelated failures.** If the existing run is
   still active, use `/sase_monitor` to watch its exact run ID with
   `gh run watch 43112612175 --repo bobs-org/bob-mac-capture --exit-status`, provided
   that its SHA still matches the checkout. Use the skill's `WATCHING CI` / `WATCHED CI`
   labels, weight zero, and an explicit continuation to inspect logs and complete this
   plan. If a fresh snapshot check is necessary, use `Scripts/macos-ci-check` from the
   linked checkout under `/sase_monitor`. Never wait inline for CI, bypass required
   steps, or rerun indefinitely until a flaky test happens to pass. Wait for the
   monitor-start command itself to exit with `Staged`, then finalize the turn; the host
   resumes verification afterward.

   The normal target is a green exact-SHA run with Test, Bundle, Launch smoke test, and
   Install and reinstall actually executed. The known fully green `ee26941` run proves
   the landed fix passed those gates; it must not be mislabeled as a green run for
   today's newer SHA. If today's clean base is red only from failures outside the fixed
   trio's causes, apply the explicit phase-worker exception: record the identical
   clean-base failures as `PROPOSED FOLLOW-UP:` notes and complete this phase instead of
   leaving it stranded. Record all failed test names and skipped downstream steps. The
   observed notebook/drawer failures are
   `CaptureEditorHighlightTests.testNotebookCreateSectionUsesDottedUnderlineAndExistingClearsIt`,
   `CapturePanelModelTests.testAcceptingDrawerCandidateInsertsSlugAndLandsCaret`, and
   `CapturePanelModelTests.testNotebookNewSectionUsesDottedPurpleUnderlineAndRepaintClearsIt`.
   Check existing bead ownership before describing follow-ups; the parent plan
   identifies active epic `bob-cli-ka` as the notebook/place-menu work owner.

4. **Finish the assigned phase.** Append evidence and any discovered work only with
   `sase bead note bob-cli-kj.5`; discovered work uses the required
   `PROPOSED FOLLOW-UP: <summary — detail>` prefix and cites existing tracking IDs where
   applicable (`bob-cli-jx` already mentions edit-before-reply; `bob-cli-jk` tracks
   second-eligible-sigil). Do not create task beads or close those tasks. Run
   `sase bead epic-symbols bob-cli-kj.5` immediately before closeout; it reported no
   entries during planning, but recheck and resolve/re-key any new entries to a
   still-open owner. When executing as the worker assigned to `bob-cli-kj.5`, close only
   that bead with
   `sase bead close bob-cli-kj.5 --note "<verified commit, exact run results, and any clean-base exceptions>"`.
   Do not reopen or hand-set status. Do not close `bob-cli-kj`, any ancestor,
   `bob-cli-jx`, or `bob-cli-jk`; task-bead disposition belongs to the parent epic's
   land agent. A differently assigned descendant worker records evidence for the phase's
   owner instead of closing an ancestor. End through `/sase_final`.

## Boundaries and acceptance

No source edits are expected: this plan recognizes already-landed work. If new evidence
shows an actual regression of the original trio, diagnose that bounded regression before
changing code, preserve all assertions, and follow the linked repo's compilation
requirements. Any Swift changes require `just portable-check`, best-effort Mac
scratch-copy `Scripts/bundle.sh` and `just typecheck-tests`, and exact-SHA macOS CI
through `Scripts/macos-ci-check`; never use Bryan's checkout or install into it. Consult
tailnet memory before remote work. No timeout inflation, test removal, unconditional
skip, or production timing change is authorized as a substitute for proving the cause.

Completion requires evidence that all three deterministic tests pass with the landed fix
intact, an explicit disposition of all three named intermittents, the latest exact-SHA
gate result with any clean-base exceptions documented, and the assigned phase closed
without closing an ancestor. A historical green run alone does not claim that every
current failure is fixed.
