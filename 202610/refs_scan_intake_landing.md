---
tier: tale
title: Finish the Bob Refs scan intake contract and land bob-cli-5x
goal:
  Report PDF intake moves accurately, match the Swift wire fixtures, and complete
  bob-cli-5x closeout in the same coding turn.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5x.land
bead: bob-cli-5x
create_time: 2026-10-09 14:47:07
status: wip
---

- **PARENT:**
  [202610/bob_refs_scan_keymap.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_refs_scan_keymap.md)
- **BEAD:**
  [bob-cli-5x](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5x/README.md)

# Finish the Bob Refs scan intake contract and land bob-cli-5x

## Objective

Complete three narrowly bounded gaps found by the land audit, then close epic
`bob-cli-5x` in this same coding turn. This tale is standalone: do not create a child
epic or defer landing to another agent. Its final code need not be committed before the
epic closes; the host commits after the turn ends.

## Context already verified

Read `sase bead read bob-cli-5x -r "Need the landing audit and remaining scope"` and
`sase artifact read plan:202610/bob_refs_scan_keymap.md "Need the original scan contract"`.
The original plan is the authority; implement only the remaining work below.

All four phases are closed. The lander read every child note and the implementations and
commits in both repositories. bob-cli's scan report/lock is commit `7fe88b7`. Mac scan
work is `6d98f23`, `37b914c`, and `d808c6d`, with compilation/test expectation fixes
`fec4293` and `fe27cd4`. The thin-client and sticky-lane decisions were checked: bob
owns scan facts and vault writes, the app decodes and presents, and opening a reference
changes no vault state. There are no additional epic DECISIONS overrides. The required
bookkeeping note on pending decision task `bob-cli-5u` is present.

Audited HEADs were bob-cli `02029a7` and bob-mac-capture `8c10d52`, matching their
origin/master refs. Non-epic changes since the first epic commit were inspected:
reference freshness/marks, Successor Links, task_link_count, and == capture support in
bob-cli; capture reset and close-comma assist in the Mac repository. Shared
BobProcessClient, AppDelegate, and fake-bob changes preserve the scan contract and
independent lane. No other integration changes were needed at that point.

Verification at these HEADs: `just check` passed (formatting, clippy, 3418 tests across
all test binaries/doctests); a scratch-copy Linux Swift run filtered to Refs and
BobProcessClient passed 218 tests; fake-bob passes `bash -n`. Mac core/service CI is
green. Scan UI lint/build and render upload passed, and phase bob-cli-5x.4 recorded
review of all 16 scan renders in light/dark with no visual defects. Its sole test
timeout is separately triaged as described below. Neither repository currently has a
`symvision` recipe, and `sase bead epic-symbols bob-cli-5x` was empty.

## Follow-up triage is finished

The only `PROPOSED FOLLOW-UP:` was bob-cli-5x.4 note #2:
`RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection` intermittently
misses a 5-second deadline on macOS. All-status duplicate searches, a last-week task
sweep, and active-epic causal checks found no duplicate or responsible active epic. It
is recorded as **bob-cli-61**, type flake, size large, ready, with a related link to the
proposing phase. Evidence is `file:explicit:c6560e48933887da529d30d2`. Failures occurred
on fec4293 and both d808c6d attempts; the same unchanged test passed on 37b914c and on
newer 8c10d52 (0.048 seconds). These are cross-SHA observations, not a claimed same-SHA
green rerun. The Refs source subtree and fixture branch did not change between d808c6d
and 8c10d52. Its possible initial git-lane/manual-rerank race needs investigation. Do
not expand this tale to fix bob-cli-61. No proposals were declined.

The newer 8c10d52 CI failure is the close-comma test sequence owned by active phase
bob-cli-60.2, which already recorded its diagnosis and test-only fix. Do not take over
that separate work. Include both facts honestly in the final verification note.

## Implementation

1. **Fix JSON intake membership in bob-cli.**
   - Files: `src/native/highlights_ref/scan_json.rs`,
     `src/native/highlights_ref/doctor.rs`, and relevant existing scan tests.
   - The original JSON contract says `intake` contains PDF moves only, in move order;
     sidecar and audio moves are excluded. `plan_xlib_intake` also produces standalone
     audio moves for existing library PDFs, but `scan_json::intake_moves` serializes
     every top-level move today. Filter that serializer to PDFs using existing
     extension/path conventions. Apply this to success, partial, dry-run, and
     hard-failure reports without changing human output or actual file moves.
   - Audited synthetic reproduction: `file:explicit:1d014ee19d2c86caa490cd53`. A scratch
     vault with `lib/chat/synthetic.pdf` and `xlib/chat/synthetic.mp3`, scanned using
     `bob ref scan --no-hooks -f json`, moved the MP3 and reported
     `intake: [{"from":"xlib/chat/synthetic.mp3","to":"lib/chat/synthetic.mp3"}]`. The
     PDF used in that minimal reproduction was intentionally invalid, so exit 1 is
     expected; the membership defect is independent of PDF validity. Use existing valid
     synthetic-PDF test helpers for regression coverage.
   - Add a meaningful CLI regression for a PDF intake plus a standalone audio intake,
     asserting the audio still moves but JSON `intake` lists only the PDF. Cover dry-run
     as well so a planned audio move cannot appear there either. Existing
     late-audio-pair tests in `tests/cli/highlights/scan.rs` show the setup.

2. **Report an already moved PDF when its companion fails.**
   - `execute_xlib_intake_counting` currently increments its prefix count only after all
     companions move. If the PDF rename succeeds and a companion rename fails, the error
     envelope slices away that already completed PDF move. The contract explicitly
     requires hard failures to report every PDF intake move that already happened.
   - Count a successfully renamed top-level move immediately after its rename, before
     attempting companions. Keep the prefix count in units of top-level IntakeMove
     entries, including standalone audio, because the error path slices the original
     list before the PDF-only serializer filters it. Do not replace this with a PDF-only
     count that would break indexing. Preserve first-error behavior.
   - Add a deterministic filesystem regression: construct a planned move with an
     existing synthetic PDF and a missing companion source, run the counting helper, and
     assert the error, PDF's destination existence, and completed prefix including that
     PDF. Check that conversion of the completed prefix to JSON intake keeps the PDF.
     Use private/unit-test access; add no public test-only seam or epic whitelist. Also
     distinguish failure of the main rename, which must not count that move.
   - Remove the duplicated adjacent doc-comment lines on `execute_xlib_intake` while
     clarifying that its count measures completed top-level renames.

3. **Match Swift scan fixture key order to the wire contract.**
   - Open the linked repo via `/sase_repo`:
     `sase repo open bob-mac-capture -r "Finish bob-cli-5x scan fixture parity"`. Work
     only at its printed path, and obey any applicable AGENTS.md.
   - `Tests/Fixtures/refs-scan-created.json` and `refs-scan-dirty.json` currently use
     alphabetically sorted keys. The original phase-4 contract explicitly requires
     key-set AND key-order parity with the real bob envelopes. Reorder those fixtures
     and the other `refs-scan-*.json` fixtures consistently, retaining their existing
     synthetic values and semantic scenarios, including the intentionally unsupported
     schema version in `refs-scan-schema2.json`. Keep compact JSON plus one newline.
   - Success/partial top-level order:
     `ok, schema_version, command, generated_at, mode, write_pdfs, hook, intake, summary, notes, failures`.
     Hard-failure order:
     `ok, schema_version, command, generated_at, mode, write_pdfs, intake, error`.
     Nested order: hook `status, command`; intake `from, to`; summary
     `pdfs, created, updated, unchanged, markers, tasks, failures`; notes
     `action, path, title, ref_type, source_pdf, marker`; failures
     `pdf, stage, message`; error `code, message, hint, paths`.
   - Compare with the phase-1 INTERFACE SAMPLE and actual Rust envelope structs/tests,
     reading the phase with
     `sase bead read bob-cli-5x.1 -r "Need the real scan envelope for fixture parity"`.
     Verify raw JSON ordering with a one-off ordered-object parse, rather than a decoder
     or a sorted-map round trip that hides order. Fixture-only changes need no new
     permanent tests or render rerun; existing decoding tests must pass.

## Verification and drift

Run `just check` in bob-cli after the fixes. Never run `just check-full`. Run the
existing Refs scan/ranking/fetching and BobProcessClient tests in a scratch copy of the
current linked checkout on Linux. No AppKit source or rendered pixels change in this
tale; use existing macOS CI evidence for those portions and state that limitation. Do
not wait for this turn's own commit, SHA, push, or CI run before closing the epic. Host
finalization commits the code and linked fixture updates.

Before final closure, inspect commits that landed in either repository after the audited
HEADs above, including remote/base-branch drift when present. Integrate any changes that
should use or conflict with these fixes; leave unrelated active-epic work to its owner.
Do not create follow-up tasks for defects caused by this epic. For a newly discovered
distinct unrelated issue, use `/sase_new_task` and record its triage outcome on
bob-cli-5x, in addition to the already completed bob-cli-61 triage.

## Final step: finish this epic's closeout in this coding turn

This step is mandatory and follows verification; it does not depend on the coder's own
commit existing.

1. Re-read the epic and child readiness with audited `sase bead read` commands; confirm
   every descendant is closed and each remaining gap above is fixed. The parent-link
   reread during landing showed **no parent bead for bob-cli-5x**. Recheck with
   `sase bead read bob-cli-5x -r "Need the parent link"`.
2. Run `sase bead epic-symbols bob-cli-5x`. For every entry keyed to this epic or its
   phases, resolve it under Symvision policy (wire, privatize, non-test pragma, or
   delete), or re-key it only to a genuinely still-open later bead needing the
   exemption. No stale epic-symbol line may survive the close.
3. Close normally with `sase bead close bob-cli-5x --note "<verification>"`. The note
   must summarize the land audit, post-start and post-audit integration review, the
   PDF-only/completed-move corrections, raw fixture parity, checks and their results,
   prior render review, every follow-up outcome (bob-cli-61 accepted; none declined
   unless new proposals arose), and remaining manual Mac smoke checks. If symbols block
   closure, finish cleanup and retry. If unfinished named phases appear, complete/reopen
   them; use force only for a deliberate canceled/superseded outcome with an explicit
   reason. Never force to make a successful landing advance.
4. After closure run `just symvision` wherever the recipe is available, and record its
   result or the confirmed absence. Both audited Justfiles lacked it.
5. Open the plans sidecar through `/sase_repo` and resolve the original plan with
   `sase artifact path plan:202610/bob_refs_scan_keymap.md`. Its audited content has
   already been read; if more content is needed, use `sase artifact read`, never direct
   sidecar artifact reads. Change only its frontmatter to `status: done`. The PLAN is
   `plan:202610/bob_refs_scan_keymap.md`; do not mark only this tale done and omit the
   original epic plan. Include this sidecar in the final commit declaration if
   opening/modifying it creates that repository obligation.
6. With no parent bead, finish normally. If the parent link has changed, obey the
   originating land instructions: close only a completed phase parent normally; for a
   directly parented plan, re-audit its previous landing note, descendants, linked plan,
   readiness, and post-child drift, retire its symbols, close normally, run available
   symvision, mark its plan done, and repeat only while fully complete. Stop and note
   the first incomplete or ambiguous parent. Never force a successful nested landing.

Manual Mac checks to record honestly as not performed on Linux: create real intake on
athena and scan/select/open it on the Mac; scan again with nothing new; hide during a
scan and click the notification; click a Refs notification while Capture holds a draft
and confirm retention; dirty-note failure and Scan Again after cleaning; overlap a
terminal/cron scan and confirm serialized completion; confirm pre-scan SSH pull works in
the app environment. The plan intentionally leaves these real-device checks for Bryan
after installing both bob and the app.

Use `/sase_final` as the last normal-turn action so all changed repositories are
committed by the host. Do not manually commit merely to obtain this tale's SHA.
