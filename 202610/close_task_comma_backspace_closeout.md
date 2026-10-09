---
tier: tale
title: Finish auto-comma Backspace and retire bob-cli-60
size: medium
goal: Backspace removes an assist-generated comma together with its task index, stale
  clients cannot re-arm the assist, and the verified epic bob-cli-60 closes normally
  in this coding turn.
proposed_by: bbugyi200.apollo.bob-cli-60.land
bead: bob-cli-60
status: done
---

- **PARENT:**
  [202610/mac_capture_auto_comma_land.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/mac_capture_auto_comma_land.md)
- **BEAD:**
  [bob-cli-60](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-60/README.md)

# Finish the remaining auto-comma work and close bob-cli-60

Implement this bounded remainder directly. This tale has no separate land agent: its
coder must finish the epic closeout below before ending the turn. Do not wait for this
turn's own future commit SHA, push, or CI run before closing the epic.

## Completed audit and boundaries

The land agent read bob-cli-60, both children and every child note, the epic plan
`plan:202610/mac_capture_auto_comma_land.md`, and the original contract
`plan:202610/mac_capture_close_list_auto_comma.md`. The findings and follow-up outcomes
are recorded in bob-cli-60's LAND AUDIT and FOLLOW-UP TRIAGE notes. Read those notes
with `sase bead read bob-cli-60 -r "Need the audited remainder and closeout"` and reread
children 60.1 and 60.2 for later notes before closing.

Already verified:

- Mac commit `8c10d52` salvaged the feature; `f949417` repaired the key-driven test's
  setter/request ordering. Both are on Mac origin/master; the audited tip is
  `f9494175f54e57e8353a1a7256bded7122a0c994`.
- GitHub CI run `37990648063`, attempt 2, passed all macOS 26 SwiftPM steps, including
  lint, build, tests, bundle, launch, and install. PR #4 is CLOSED
  (2026-10-09T21:15:09Z); its remote `close-list-auto-comma` branch is gone.
- The duplicate `CapturePomodoroEntry` declaration was fixed by naming the minimal
  Decodable model `CapturePomodorosListEntry`. Count decoding tolerates older bob; panel
  show and visible-vault-change refreshes are wired; immediate parse requests come from
  digit keys; the normal count-refresh generation guard and current-draft snapshot
  checks are present.
- bob-cli commit `5601235` supplies `task_link_count`. It uses the shared
  `capture_pomodoro_close::number_task_links`; the CLI parity test in
  `tests/cli/capture/pomodoro_name.rs` compares the count to the dry-run lineup.
- All unrelated Mac commits after the first epic commit were reviewed: successor
  presentation `f4a36e3`, override presentation/picker `6fc7b00` and `0cebe63`, fixture
  repair `0f2f40c`, reference tasks `aa47c1f`, ViewBuilder build fix `ee19240`, and
  hotkey change/merge `1c161d4`/`bc10188`. They preserve the assist helper, key router,
  controller, transport, and count/span contracts. Later model edits concern
  presentation. bob-cli's later override, successor (including `9b44dc6`), and ref-sync
  commits do not duplicate the assist; the count still shares the authoritative close
  lineup. Recheck subsequent drift, integrating actual conflicts if any, without
  repeating completed work.
- The thin-client decision applies; read
  `sase memory read decisions:mac-capture-is-a-thin-client -r "Preserve the assist's span-driven contract"`.
  There are no additional frontmatter DECISIONS in the epic plan.
- Both phases are closed and there are no existing `--epic-symbol` entries for
  bob-cli-60. The epic currently has no parent bead.

Follow-up triage is finished; preserve these outcomes in the final close note:

- Neither child contains a `PROPOSED FOLLOW-UP:` entry.
- bob-cli-60.2 notes #4/#5's unrelated Refs refresh-reorder timeout duplicates task
  `bob-cli-61`; the land agent added +1 identifying those notes. Independent CI log
  evidence on the identical f949417 SHA: attempt 1 failed
  `RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection` at line 825 in
  5.849s; attempt 2 passed in 0.081s. No duplicate task was filed.
- Epic notes #2/#3 and child 60.2 #1/#2 describe the same resolved assist test-order
  defect, fixed by f949417; the concurrent ViewBuilder compile failure is fixed by
  bob-cli-5z landing ee19240. No new tasks for resolved reports.
- Backspace and the stale-client race below belong to this epic, not follow-up tasks. If
  new unrelated work is discovered, triage via `/sase_new_task` and record every outcome
  with the proposing bead and note ordinal.
- Phase 60.2 note #3 reports a historical direct push after six host-finalizer
  submissions failed to materialize. This deviated from the epic's transport rules; the
  resulting source and CI have been verified. Do not repeat it: use `/sase_final`
  repository commit declarations for this turn's own changes.

## 1. Open current source and implement provenance-aware Backspace

Open the Mac repository using
`sase repo open gh:bobs-org/bob-mac-capture -r "Finish bob-cli-60 Backspace and count lifecycle"`.
Use only its printed checkout path, read AGENTS.md if named, and work on current master
after fetching/pulling safely. Do not create branches or PRs, or commit or push by hand.
Read the repo skill first. Preserve concurrent changes.

The missing user requirement is epic note #1: when Backspace deletes a typed task index
that caused a comma to be generated, remove that comma too.

Current paths:

- `Sources/CaptureCore/CaptureCloseTaskCommaAssist.swift` returns the accepted
  `CaptureCloseTaskCommaEdit` containing `,<digit>` and a UTF-16 caret range.
- `CapturePanelController.insertCloseTaskNumberInEditableTextView` requests the parse,
  checks editability/marked text, then uses native `NSTextView.insertText`.
- `CaptureKeyCommandRouter` already routes unmodified editor Backspace to
  `.deleteBackward`, after stash/prompt/picker routing; modified Backspace stays native.
  The controller's `.deleteBackward` handler currently tries only
  `deleteEmptyBulletRowInEditableTextView`, then falls through to AppKit.
- `CapturePanelModel.attributedDraft` is the SwiftUI editor binding;
  `editorTextDidChange` observes character changes. `setPlainDraft` handles wholesale
  programmatic replacement. Highlighting may replace attributes without changing text.
  Account for these actual paths, rather than relying only on a plainDraft setter or on
  transient NSTextStorage attributes that SwiftUI/highlighting might discard.

Record provenance only when the controller actually applies an accepted auto-comma edit.
Track the inserted comma/index pair in UTF-16 coordinates and the text it belongs to; a
small private tracker in the model, with pure range logic in CaptureCore if useful, is
sufficient. Reconcile ordinary text edits so unaffected pairs survive edits before/after
them; invalidate pairs when an edit replaces their comma/index or provenance cannot be
established. Clear tracking on wholesale draft replacement, discard, submit/clear, and
stash restore. Attribute-only highlighting and caret movement must preserve it. Native
undo/redo must not produce stale offsets or let provenance attach to unrelated text; use
the existing native editing/undo path and test it.

Add a Backspace helper before empty-bullet deletion in the existing `.deleteBackward`
handler. With an editable main text view, no marked text, a collapsed caret immediately
after an intact recorded `,<digit>` pair, it deletes both characters as one native
`insertText("", replacementRange: ...)` edit, sets the caret before that comma, updates
provenance, and returns true. Otherwise try the existing empty-bullet helper and then
let AppKit handle the key. Do not infer provenance by matching every comma-digit
substring or by recognizing close grammar in Swift. The recorded insertion was already
authorized by Bob's semantic spans. Removal of that inserted pair should work
immediately, without waiting for a new parse or current count refresh.

Examples: assisted `=x1,2` Backspace becomes `=x1`; repeated assisted digits `=x1,2,3`
delete back to `=x1,2`, then `=x1`; deleting an assisted index in the middle before `!3`
removes only its generated comma/index. A manually typed or pasted `=x1,2` remains
native and becomes `=x1,`. Preserve group, link-close, new-task-close, multibyte-prefix,
picker/prompt, modified-key, and IME behavior. Add concise README wording beside the
close assist and existing Backspace row.

## 2. Invalidate in-flight count refreshes across client changes

This is an epic-caused lifecycle gap in `CapturePanelModel`:
`refreshCurrentPomodoroTaskLinkCount` captures an old client and generation;
`setProcessClient(nil)` clears the count but does not increment
`pomodoroCountGeneration`. The old request's success can then pass the guard and restore
a count, re-arming the assist after Bob becomes unavailable. Replacing a client for a
different configured executable/vault can similarly reuse an old count.
AppDelegate.configureProcessClient uses this setter.

Invalidate the count generation whenever setProcessClient changes the client, clear the
old count, and ensure only refreshes belonging to the active client can publish success
or failure. A non-nil replacement should fetch its count when appropriate, so an
already-visible panel does not stay stale until an unrelated vault change. Preserve
silent failure behavior and existing show/watcher refreshes. Also invalidate obsolete
assist-parse publication across client replacement if the same old-client callback
pattern can republish after clearing; keep this limited to the new assist state.

## 3. Verify behavior and integrate any new drift

Add focused regression coverage using existing CaptureCloseTaskComma tests and fake-bob
conventions. Exercise actual accepted controller insertion followed by routed/helper
Backspace, asserting text and caret; repeated pairs; a pair in the middle; Unicode
prefix; manual/pasted commas; noncollapsed selection; marked text; modified Backspace
and modal routing; unchanged empty-bullet deletion; ordinary edits moving or
invalidating records; programmatic reset; and native undo/redo safety. Keep tests
meaningful and avoid duplicating every existing insertion case.

For count lifecycle, deterministically hold or observe an old request across
setProcessClient(nil) and client replacement, then let it finish and assert no old count
publishes; cover the active replacement's successful refresh. Reuse record/fixture
helpers or a narrowly scoped fake-bob synchronization control, not timeout-only tests
that can pass before the old request actually finishes. Retain the f949417 test-order
fix and all accepted thin-client/count rules.

Run `just check` in the primary bob-cli checkout for file-change verification. Do not
run `just check-full`. If just check passes but some independently observed check-full
fails, treat that as test infrastructure, not epic work. The Mac justfile currently has
format-lint/build/test recipes, no check or symvision recipe; do not add a pretend
verification alias. Run its relevant Swift checks/tests if a usable macOS toolchain is
available. This Linux host currently has no Swift executable: otherwise perform careful
Swift 5-mode, MainActor, UTF-16, native edit, and formatting review, `git diff --check`,
and `bash -n Tests/Fixtures/fake-bob`, and state the limit precisely. The existing
CI-green feature at f949417 is verified; this turn's new code gets its own
post-finalizer CI, which must not be a prerequisite for this turn's closeout. Use
`/sase_monitor` for any long verification command and wait for the monitor start command
itself to exit. Do not end a turn with a tool session still live.

Before finishing, fetch Mac origin/master again and integrate new commits, preserving
both sides of any overlap. Re-run appropriate checks only if new changes justify it.
Record verification and any remaining unrelated failures with their triage outcomes on
the epic.

## 4. Close this epic in this same coding turn

Complete this step before the final declaration; it is not contingent on this tale's
future commit, push, CI, or a separate land agent.

1. Read bob-cli-60 and every descendant again for later notes and readiness. Verify both
   existing phases remain complete. If this tale is represented by an open descendant
   plan bead, close that completed tale bead normally first with its verification note,
   so descendant readiness can pass. Do not force a successful nested landing.
2. Run `sase bead epic-symbols bob-cli-60`. For every entry keyed to this epic or one of
   its phases, wire it up, privatize it, add a valid non-test pragma, or delete it under
   Symvision policy. Re-key only when an identified still-open later bead genuinely
   needs the exemption. Leave no stale entries. The prior audit found none, but the
   final check is required.
3. Close normally with `sase bead close bob-cli-60 --note "<concrete verification>"`.
   Include the completed source/commit/CI/PR audit, later-commit integration,
   implemented Backspace and client-lifecycle fixes, actual checks and limits,
   thin-client compliance, and all follow-up outcomes above. If whitelist cleanup blocks
   close, finish it and close again. If a named phase is incomplete, finish or reopen
   it. Never use --force merely to succeed or to advance a successful nested landing;
   --force --reason with canceled or superseded resolution is only for a deliberate real
   non-done disposition.
4. After close, run `just symvision` wherever the recipe is available to confirm the
   whitelist is clean. Current bob-cli and Mac justfiles have no symvision recipe;
   record unavailability if that remains true.
5. Open the plans repository with
   `sase repo open plans -r "Mark completed bob-cli-60 epic plan done"`, read its
   instructions, then set only `status: done` in the frontmatter of
   `202610/mac_capture_auto_comma_land.md` under that printed path. This is the PLAN
   path resolved by `sase bead read bob-cli-60`.
6. Run `sase bead read bob-cli-60 -r "Need the parent link"`. The audited epic has no
   parent. If unchanged, finish normally. If a parent was added, read it: for a phase
   verify this child's work fulfills its scope and close only that phase normally,
   leaving its epic to its waiting land agent. For a plan parent, recheck its previous
   landing note, every descendant/note, linked plan readiness, and post-child drift;
   close only while fully complete, retiring its epic-symbols first, running available
   symvision, and marking its linked plan done. Repeat directly parented plan ancestors.
   At the first incomplete or ambiguous parent, record its blocker and report it.

End with `/sase_final` as the last action. Give a Conventional Commit `commit` decision
to every repository this turn changed, including Mac and plans; set the completed
primary assigned bead's bead_action appropriately. Do not invoke git commit/push
manually. Final report should say what closed, summarize the fixes and validation
limits, and give Mac reinstall/manual-check steps: `just install` in bob-cli,
pull/reinstall Mac Capture with `just install`, then try assisted `=x12` and Backspace
with a sub-10-link running Pomodoro.
