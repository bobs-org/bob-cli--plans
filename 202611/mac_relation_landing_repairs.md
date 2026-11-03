---
tier: tale
title: Repair Mac relation presentation and finish the nested landing
goal:
  Resolve the remaining Mac relation contract gaps, verify the combined implementation,
  and close bob-cli-fs.10.1 and parent phase bob-cli-fs.10 normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-fs.10.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fs.10.1
status: done
---

- **PARENT:**
  [202611/mac_relation_successors.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_relation_successors.md)
- **BEAD:**
  [bob-cli-fs.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.10.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-fs.10.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-fs.10.1.land.md)
- **COMMITS:**
  - [17bd529](https://github.com/bobs-org/bob-mac-capture/commit/17bd529a2d80c46be39bc18e99be0b101701eefa)
    — feat(capture): repair Mac relation presentation and finish nested landing

# Repair Mac relation presentation and finish bob-cli-fs.10.1 landing

## Scope and ownership

Finish the remaining work found while landing `bob-cli-fs.10.1` (Bob Mac Capture
relation successors), then close that epic and its directly parented phase
`bob-cli-fs.10`. This is one bounded coding assignment, not another phased epic. The
coder must finish the landing in the same turn as the final repairs, before host
finalization commits them. Do not wait for this work's commit SHA, push, or CI result
before closing. Do not close the containing epic `bob-cli-fs`; its land agent owns that
work.

Use `/sase_repo` to open `bob-mac-capture` before reading or editing it; follow its
`AGENTS.md` and use only the returned checkout. All product changes belong there. Use
`sase artifact read` for both `plan:202611/mac_relation_successors.md` and
`plan:202611/starts_after_relation.md` as needed. Read the current epic and parent phase
with `sase bead read` for any new notes. No memory edits are required or authorized
here. The parent's final DECISIONS permitted only three glossary edits elsewhere and
refused a decision record; retain those decisions. Do not change the live vault,
installed app, Bob configuration, Cargo version, or changelog.

Bob remains the authority for eligibility, destinations, verdicts, labels, and group
copy. This repair only presents its JSON. Preserve additive schema 1 decoding, opaque
offer keys, explicit selection, and legacy payload behavior.

## Verified starting point

The lander read every note on all four closed phases and reviewed their commits and
combined source: `4b8db34` (protocol), `52a222a` (chooser), `e3f0817` (reporting), and
`5e95b8f` (authoring). The fixture generator uses isolated vaults, config/state/cache, a
pinned clock, path-only normalization, and provenance at Bob SHA
`71907dab7ceb7dab45a401a700e7069cd767bc24`. Existing real fixtures cover parked, Done,
toggle unlink, nested completion, held-only, mixed, selected, none, refusal, and
authoring. Keep this work; fix the gaps below.

Follow-up triage is complete and recorded on `bob-cli-fs.10.1`: there were **zero
PROPOSED FOLLOW-UP entries**, no unrelated findings, and no declined proposals or new
task beads. Every issue below is epic-caused work. Prior failed Mac connections are an
explicitly permitted best-effort limit, not a separate task.

Both origins were fetched during review. Mac master at `5e95b8f` contains only this
epic's four commits since `a58e4f0`. Primary master at `807e1816` contains five
post-start external commits: `15d18e90`, `817ba7dd`, `a9a499ac`, `4d5a61de`, and
`807e1816` (clock-tag capture grammar, agenda/settle/stats clocks, gcal clocks,
freshness docs/memory). Inspection found no changed relation wire contract or
integration requiring another Swift planner. Recovery re-fetched both origins: Mac
remains at `5e95b8f`; primary origin/master now also contains `0346d828`, which adds
decay-ladder live-verification documentation, navigation registration, and the matching
registry test update. It does not change relation behavior. Preserve these changes and
recheck any newer drift before closeout. Generator `bash -n` and combined
`git diff --check` passed. `epic-symbols` was empty for both the child epic and parent
phase.

## Remaining implementation

1. **Make the authoring slot valid inside SwiftUI's result builder.** In
   `Sources/BobMacCapture/CapturePanelView.swift`, `RelationLabelPromptCard` declares
   `let color: Color` then assigns it in `switch slot.style` arms inside the view
   builder. These assignments are Void expressions, not Views. A minimal Swift 6
   result-builder reproduction fails typechecking on each assignment. Move the
   style-to-color mapping into a normal computed helper or another expression outside
   builder branches. Preserve the neutral offers, amber caveats, muted hints,
   cancel/complete colors, and the rule that offers never dim or close the source. This
   finding is from source and a language reproduction, not a claimed native app build.

2. **Join groups by their primary reason and consistently present Bob's copy.** In
   `Sources/BobMacCapture/CapturePullInChoice.swift`, `PullInSectionIdentity.init(row:)`
   currently takes `row.label` from `row.relation.label` for every group. Rust's
   `src/native/capture/pull_in_copy.rs::group_key/emit_group` sets group `label` and
   `after` only for a primary relation group; a hand-off/project/keep-moving row may
   separately carry relation metadata. Such a row currently fails to match its Bob
   group, splits the group, loses authored copy, and triggers legacy ordering. Use
   relation discriminators only for primary relation groups; secondary metadata must
   remain attached to its row for display. Keep distinct relation labels at the same
   target distinct, without normalizing labels or reconstructing keys. Preserve Bob
   order for known supplied groups; append any unmatched fallback groups
   deterministically without discarding that order.

   `contextText` still synthesizes `Unblocked by …` and new-project text even when Bob
   supplies group copy. Respect supplied copy in the context line too; an absent `why`
   in a supplied copy object means no invented legacy caption. Keep old strings only for
   absent-copy payloads and do not attribute relation or keep-moving rows to dependency
   recovery.

   `metaDetails` has relation suffix/second-reason text, but non-relation `detailText`
   rebuilds another list, and `accessibilityLabel` ignores `metaDetails` whenever
   `detailText` exists. Passive previews use `detailText`, so both passive output and
   accessibility lose the second reason. Reuse a consistent presentation path without
   duplicate meta. Preserve the lane-only relation status and suppress raw relation
   trigger text. Review the existing
   `testRelationAfterSuffixAndSecondReasonReachMetaAndAccessibility`: its second reason
   assertion is currently contradicted by implementation; its broad trigger assertion
   also conflicts with its synthetic `dependency` trigger. Resolve the latter
   deliberately against the contract rather than weakening the required second-reason
   coverage.

   Add model regressions for a hand-off with a second relation reason alongside another
   hand-off and a relation group: one hand-off section, exact Bob copy, exact Bob group
   order, intact meta in preview/accessibility, and no extra sections for absent
   per-item rows. Include non-relation `why` override/absence, partial group metadata,
   same-target distinct labels, and no-copy legacy cases.

3. **Render held reasons from real wire fields and respect the picker boundary.**
   `CaptureRelationHeldPresentation.heldReasonText` only treats literal
   `reason: scheduled` and `reason: project` specially. Rust
   `src/native/capture/output.rs::relation_held_row_json` emits both as
   `reason: blocked` with `scheduled`/`project` metadata. Its `held_copy_reason` uses
   the precedence scheduled, then project, then dependency count. Mirror this
   presentation precedence in Swift and retain the documented short month/day wording.
   Tests must decode the actual blocked wire shapes, including metadata precedence,
   instead of relying only on synthetic reason strings.

   Centralize or consistently reuse held/check presentation across item-level toggles,
   whole-item task completion, Pomodoro close, nested relation completes, and batches.
   Current task/Pomodoro notification bodies append held rows whenever choice is not
   `none`, including after a picker with selections; item-level and nested routes omit
   them. Show held lines only when no picker is involved, avoid duplicating folded
   diagnostics, and keep explicit `none` silent. Carry the enclosing choice context when
   using nested completion presentation. Aggregate helpers already expose all relation
   held/check locations; reuse them and account for batch roots mirroring the first
   item. Suppress `already_planned` and retain unavailable uncertainty wording and
   old-payload silence. Cover the preview and committed result paths, including a batch
   with offers in one item and held rows in another.

4. **Report selected relation picks on every notification route.**
   `NotificationService.successPresentation` returns early for a single toggle, task
   completion, Pomodoro close, or relationship notification. Only the latter two
   completion models gained relation counts; `pulledInSummary` is used on batch paths. A
   selected item-level toggle or nested relation-complete result therefore loses its
   relation notification. Integrate a shared summary at the right boundary so every
   single/batch route reports selections once, preserving existing titles, dependency
   wording, keep-moving breakdowns, and note targets. Relation-only selections use
   `🪢 Pulled in n · label “T”`. Deduplicate opaque keys within the capture result and
   avoid double-counting a second reason. Avoid appending a second summary to
   task/Pomodoro notifications that already contain one. Unselected offers and explicit
   none must not produce a picked notification.

   Add tests that actually exercise a single toggle, a single nested complete-label
   capture, and multi-item captures; the existing two tests named Batch in
   `NotificationServiceTests` each pass only one task-complete capture. Cover selected
   and none, nested plus top-level rows, duplicate keys, and mixed
   dependency/keep-moving reasons. Extend the isolated real-Bob generator for committed
   selected toggle and nested cases, plus blocked scheduled/project held cases and a
   primary hand-off with a secondary relation reason. Preserve genuine output; do not
   patch fixture semantics. Retain original fixtures whose inputs did not change and
   record exact build/provenance for new fixtures.

## Verification and drift

Add focused regressions for the repaired behavior, then run the complete combined
`just portable-check` and `just docs-check` in the opened Mac checkout. The Mac repo has
no `just check` recipe; portable-check is its required gate. If primary bob-cli code
changes prove necessary, run `just check` there. Never run `just check-full` for this
landing. Format only changed lines. Run `bash -n` for the updated generator and execute
it against an explicitly built Bob from an opened checkout, in isolated scratch state.
Reuse verified retained build provenance where possible; use `/sase_monitor` for long
builds/checks with explicit continuation instructions to finish this tale and its
closeout.

Read `tailnet.md` through `/sase_memory_read`, then attempt the native Mac compile with
a bounded connection timeout using SSH alias `mac`: create a unique scratch copy, rsync
without `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, and remove
only that scratch copy. Never use Bryan's checkout or install the app. The Mac Command
Line Tools lack XCTest, so do not substitute `swift build --build-tests` there. Actual
macOS tests run in CI; do not claim they ran from Linux or stub typechecks. If Mac is
unavailable, record the failed attempt and explicitly report macOS targets unverified.
This does not excuse skipping portable compilation or ignoring a source-proven compile
defect.

Re-read all descendant statuses and new notes and review any commits landed since the
verified heads above. Resolve any new epic-caused integration issue before closing. No
unrelated follow-up triage remains from the lander's review. For any new genuinely
unrelated finding, use `/sase_new_task` and record the result on `bob-cli-fs.10.1`. Do
not file the listed repairs as independent tasks.

## Final step: perform this epic's closeout

This is mandatory coder work; a tale has no land agent to resume it later. Do it after
the repairs and gates pass, in this same turn, before host finalization.

1. Run `sase bead epic-symbols bob-cli-fs.10.1`. Resolve every listed exemption by
   wiring, privatizing, deleting, or using a valid non-test pragma; re-key only to a
   still-open later bead that actually needs it. Run
   `sase bead close bob-cli-fs.10.1 --note "<source fixes, all four original phase notes checked, integration review, actual combined verification results, Mac verification limits, and follow-up triage outcomes>"`.
   All four original phases must remain completed and no unfinished descendants may
   remain. If close rejects a leftover exemption, fix it and retry. If a named phase is
   incomplete, finish/reopen it. Never use `--force` to advance this successful nested
   landing or merely make close succeed.
2. Run `just symvision` if available in the relevant checkout(s); the inspected primary
   and Mac justfiles have no such recipe, so record absence if unchanged. Use
   `/sase_repo` to open `plans`, use `sase artifact read` to audit the linked plan
   content, and set only its frontmatter `status: done` in
   `202611/mac_relation_successors.md` under the returned plans checkout (the PLAN file
   shown by `sase bead read bob-cli-fs.10.1`).
3. Run `sase bead read bob-cli-fs.10.1 -r "Need the parent link"` and
   `sase bead read bob-cli-fs.10 -r "Verify delegated Mac phase completion"`. Verify
   that the repaired child fulfills the parent's Mac phase and final DECISIONS. Run
   `sase bead epic-symbols bob-cli-fs.10` and resolve/re-key any entries. Close **only
   that phase** normally:
   `sase bead close bob-cli-fs.10 --note "<child epic complete; generic Bob-copy groups, relation decoding/choices/previews/notices/badges/authoring verified; actual checks and Mac limitation>"`.
   Never force it. Leave containing epic `bob-cli-fs` and its linked plan status to its
   already-waiting land agent. If new evidence leaves this parent phase incomplete or
   ambiguous, record the specific blocker on it and report that instead of falsely
   closing it.
4. Finish through `/sase_final`, declaring every edited repository (including the plans
   sidecar) for host commit. Report both close outcomes and actual test results. No
   post-declaration edits and no waiting for this turn's own commit.
