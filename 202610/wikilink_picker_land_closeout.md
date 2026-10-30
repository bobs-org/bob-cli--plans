---
tier: tale
title: Finish the Task Wikilink Picker repairs and close bob-cli-e7
goal:
  Apply the never-executed landing repairs for the `((` Task Wikilink Picker, fix the
  e7-caused old-Bob test fixture regression, verify macOS behavior and renders, and
  close bob-cli-e7.
size: medium
bead: bob-cli-e7
proposed_by: bbugyi200.athena.163.f0
proposed_with: claude/opus
create_time: 2026-10-30 12:28:09
status: done
---

- **BEAD:**
  [bob-cli-e7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e7/README.md)

# Finish the Task Wikilink Picker repairs and close bob-cli-e7

## Why this tale exists

An audit on 2026-10-30 found that `bob-cli-e7` (the `((` Task Wikilink Picker epic,
approved plan `plan:202610/wikilink_task_picker.md`) is still **IN_PROGRESS**. Its four
phases (`bob-cli-e7.1` through `bob-cli-e7.4`) are closed, but the feature is not done:

- The approved landing tale `plan:202610/wikilink_picker_landing.md` (status `wip`) was
  never executed. Neither repository has a commit implementing it after its approval.
- The later tale `plan:202610/wikilink_task_parse_spans.md` landed as bob-cli `4b4a4a34`
  and bob-mac-capture `ef07ce2`. It fixed the root cause of "typing `((` does nothing"
  (missing `capture-parse` spans plus the Mac completion gate). It also absorbed only
  two pieces of the landing tale: step 1's suffix-punctuation boundary fix
  (`Follow up ((foo and [existing](url)` now yields a field) and step 3's test
  initializer-order compile fix. Everything else in the landing tale remains.

This tale **supersedes** the unexecuted landing tale. Read it with `sase artifact read`
as the detailed specification for steps 2 through 4 and the closeout. Where the two
differ, this tale wins, because it reflects the current source.

The epic's reviewer decisions remain final: **history = all** and **link_form =
shortest**. Bob owns grammar, candidates, query, ranges, replacements, and vault writes;
Swift only presents (`decisions:mac-capture-is-a-thin-client`). Do not change release
versions, `CHANGELOG.md`, memory, or unrelated task-type, mention, or gcal features.

## Verified current defects (bob-cli `a844d3e1`, bob-mac-capture `ef07ce2`)

All of these are in `Sources/BobMacCapture/CapturePanelModel.swift` unless noted.

1. **Multiword full-pool refetch never matches.** `handleWikilinkTaskCompletion`
   refetches at `r.start + 2` and accepts the snapshot only if
   `snapshot.replacement == r`. Bob's unclosed replacement end is caret-dependent by
   design: `non_whitespace_run_end` in `src/native/capture_task_wikilinks.rs` extends
   from the caret to the next whitespace. For `Follow up ((foo bar` with the caret after
   `bar`, the range is `(10,19)`, but at the refetch cursor 12 it is `(10,15)`. The
   equality fails, so auto-open falls back to the partial, filtered pool. Chip reopen
   for multiword queries has the same problem.
2. **The query is reconstructed from the draft.** `handleWikilinkTaskCompletion` falls
   back to `wikilinkTaskPickerQuery(in:range:cursor:)`. The `.wikilinkTask` branch of
   `openPickerFromChip` always slices the draft, so reopening a closed `((foo))` chip
   with the caret at the end seeds `foo))`. `CapturePickerChipState` stores no Bob query
   or descriptor for this source.
3. **The descriptor ranges are synthesized.** `removePickerTrigger` deletes
   `CaptureRange(start: r.start + 1, end: r.end)` instead of Bob's
   `trigger_removal_range`. `pickerMarkerHighlightRange` returns `replacementRange` for
   `.wikilinkTask` instead of Bob's `marker_range`. `CapturePickerSource.wikilinkTask`
   carries no context, unlike `.parentTask(context)`, which already threads
   `triggerRemovalRange` and `markerRange`; mirror that pattern. The decoded descriptor
   (`isWikilinkTask` in `Sources/CaptureCore/CaptureModels.swift`) is discarded.
4. **`cursor_after` is ignored on ordinary acceptance.** `acceptPickerRow` places the
   caret at `replacementRange.start + insertionText.utf8.count`, not at the selected
   candidate's `cursor_after`.
5. **Two hosted CI failures are e7-caused, not unrelated.**
   `CaptureDependencyPanelTests.testOldBobBlocksIDAssignmentShowsExactUpdateMessage` and
   `testOldBobRelationIDAssignmentShowsExactUpdateMessage` fail because `031f100` (e7.3)
   added `blocked_task_replacement`, `relation_to_replacement`,
   `relation_from_replacement`, and `wikilink_replacement` to the generic
   `capture-task-id --note-path` branch of `Tests/Fixtures/fake-bob`. The tests' block
   IDs (`call-bank-old`, `idless-old`) fall through to that branch. They now receive
   replacements and insert, instead of showing "This Bob is too old for …". Pre-picker
   master run 40968776328 at `d41bb61` failed only
   `testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible` (tracked by `bob-cli-dz`).
   The test target did not compile again until the parse-spans work, which hid this. App
   behavior is correct; this is a fixture regression.
6. **Coverage is missing.**
   `Tests/BobMacCaptureTests/CaptureWikilinkTaskPanelTests.swift` has 14 tests. It
   lacks: chip instead of auto-open on Delete and Backspace; second-parenthesis burst
   auto-open; closed and multiword chip reopen; full-pool filter clearing without extra
   subprocesses; descriptor-controlled Backspace and highlight; caret at `cursor_after`;
   ID-prompt failure, plus Escape and filter restoration; missing-replacement
   compatibility; Command-Return and Control-Return follow-ups; and `--allow-closed` for
   both Done and Cancelled ID-less rows. Neither repository has a `((` regression
   combined with trailing `@@@` or `@@template:`.
7. **The UI and render review are unfinished.** Per the epic plan's card spec: `#hide`
   rows are sorted last and badged "Hidden" but are not shown subdued; completed rows
   need a quiet **Done** / **Cancelled** capsule; ID-less rows need the dim
   `plus.circle` and **Add ID** on the locator; the identified action line must be
   link-colored like the editor's wikilink spans; and section headers must not fall
   through to `CapturePickerView`'s "Not in a Pomodoro" subtitle. No one has reviewed
   the 20 `wikilink-task-*` render PNGs.

## Implementation

Before working, use `/sase_repo` to open `bob-mac-capture` and `plans`. Read the linked
repo's `AGENTS.md`. Read `plan:202610/wikilink_task_picker.md` and
`plan:202610/wikilink_picker_landing.md` with `sase artifact read`, and read
`bob-cli-e7` with `sase bead read`. Use `/sase_memory_read` for
`decisions:mac-capture-is-a-thin-client`, `docs.md`, and `tailnet.md` as needed. Never
use Bryan's live vault for fixtures, or his Mac checkout for scratch compilation.

1. **Thread Bob's picker contract through the Mac** (fixes defects 1–4). Give the
   `.wikilinkTask` source a context carrying Bob's decoded descriptor (`marker_range`,
   `trigger_removal_range`) and Bob's `query`, following the `.parentTask(context)`
   pattern. Thread it through picker state, chip state, Escape/reopen, and ID-prompt
   restoration.
   - Apply `marker_range` and `trigger_removal_range` verbatim, with range and draft
     validation.
   - Place the caret at the selected candidate's `cursor_after` on ordinary acceptance.
     Keep splicing `wikilink_replacement` after ID assignment, with the caret after the
     returned link.
   - Do not synthesize grammar or ranges as a fallback for missing or invalid descriptor
     data. Treat that as an older Bob and keep the token plain, consistent with
     `testOlderBobWithoutTaskWikilinkParseSpansKeepsTokenPlain`.
   - For the full-pool refetch at the opener, require an unchanged draft, the same
     generation, the `wikilink_task` context, and the same token start (the same
     opener). Do not require end-offset equality. Keep the original Bob-authored
     replacement range, descriptor ranges, and query, and take only the complete
     candidate pool from the refetch. Never replace the original insertion range with
     the shorter refetch range.
   - Closed-token chip reopen must seed Bob's `query` (no closing parentheses). Clearing
     or changing the filter must reveal every eligible task locally, without
     per-keystroke subprocesses.
   - Preserve the stale-response and stale-draft guards.

   Keep the Bob-side boundary as is: the caret-dependent end is intentional, and the Mac
   handles it. If Bob's existing tests lack a dry-run capture assertion for valid source
   drafts, as landing step 1 asked, add only that.

2. **Repair the fake-bob old-Bob regression** (defect 5). In `Tests/Fixtures/fake-bob`,
   route the old-Bob block IDs (`*-old`) to a response shaped like a Bob that predates
   these fields. Omit `blocked_task_replacement`, `relation_to_replacement`,
   `relation_from_replacement`, and `wikilink_replacement`; keep
   `dependency_replacement` and `complete_replacement`, matching the branch before
   `031f100`. Keep `FAKE_BOB_WIKILINK_OLD` working for `testOldBobKeepsPromptOpen`. Do
   not change the app's old-Bob guards. Confirm both `CaptureDependencyPanelTests`
   old-Bob tests and every other `-old`/`FAKE_BOB_*` test pass in hosted CI.

3. **Complete the Mac tests** (defect 6). Add the missing behavioral cases listed above
   to `CaptureWikilinkTaskPanelTests`, using bounded async waits (never an untimed
   continuation; see `AGENTS.md`). Before relying on the fake-bob JSON fixtures,
   regenerate or reconcile them against a freshly built bob-cli (`cargo build`) and a
   synthetic temp vault. Their UTF-8 ranges, `query`, descriptor ranges, and
   `cursor_after` values must be real, not arbitrary constants. Include a multiword
   fixture whose refetch range differs from the original. Add any new Mac fixtures the
   same way.

4. **Finish the UI and review real renders** (defect 7). In
   `Sources/CaptureCore/WikilinkTaskPickerPresentation.swift` and the shared picker
   views, scoped so existing pickers keep their behavior:
   - render hidden rows subdued;
   - show completed rows with a Done or Cancelled capsule;
   - show the ID-less Add ID affordance on the locator;
   - make the identified action line link-colored;
   - give task-wikilink section headers appropriate subtitles.

   Reuse Bob's metadata for note icons; do not infer vault semantics in Swift. Format
   only changed lines.

   Then download the successful hosted run's `render-fixtures` artifact
   (`gh run download`). View all five `wikilink-task-*` states at 760 and 620 points, in
   light and dark (20 images). Fix any alignment, truncation, selection, locator-color,
   or contrast defects, and repeat the affected checks and renders after each change.

5. **Add the integration regressions.**
   - In bob-cli, add focused tests showing that trailing `@@@` template rewrite and the
     `@@template:` completion gate coexist with a `((` token. Draft-global byte ranges
     and the `((` parse spans must stay correct after the rewrite.
   - In bob-mac-capture, add the matching panel-level case.
   - Confirm that the inserted `[[target#^id]]` still feeds the landed mention scanner
     and the type-aware capture path unchanged. Reuse that behavior; do not add a second
     parser or writer.
   - Update only documentation that this repair makes inaccurate, in `docs/capture.md`,
     `docs/wikilink-completion.md`, and the Mac docs.

6. **Verify.**
   - bob-cli: run `just check` (prefer `sase tool run check`) and `just docs-check`. Do
     not run `just check-full`.
   - bob-mac-capture: run `just portable-check` (on athena if needed), and
     `just docs-check` with `BOB_CLI_DIR` pointing at this bob-cli checkout.
   - Run `Scripts/macos-ci-check` from the Mac checkout through `/sase_monitor` for the
     long CI wait. Recheck every required step even after a failure.
   - Best effort, if the Mac is reachable: compile a scratch copy on the Mac with
     `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. If it is
     offline, say that the macOS targets are verified only by hosted CI.
   - Fix every e7-caused failure. The hosted Test step must have no wikilink failures
     and no old-Bob failures.

   Known independent failures, to triage under the existing task policy and not fix
   here:
   - `bob-cli-87`: bob-cli lib test
     `table_layout_records_prose_tables_on_snapshot_route`.
   - `bob-cli-dz`: Mac `testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible`.
   - Active epic `bob-cli-eg` (`bob-cli-eg.4` owns the final gate): bob-cli `docs-check`
     lint on `BOB_GCAL_BROWSER` and `BOB_GCAL_CONSENT_SECS` in
     `src/native/gcal/signin.rs`, and `gcal_auth::login_browser_access_denied`. Check
     eg's notes first; if eg has landed fixes, they should simply pass.
   - Mac
     `CapturePanelModelTests.testPomodoroNamePromptUsesConfiguredThemesInConfigOrder`,
     which expects "isn't one of your configured themes" but gets the task-types copy
     "isn't one of your types; tasks here keep their own types" (from
     `7ac2bd9`/`2f9a70a`). No bead tracks it. Use `/sase_new_task`: corroborate onto the
     owning task-types work if it is still active, otherwise file a `ci` task.

   If any of these remains, record it on `bob-cli-e7` with its owner and continue.

## Final step: close bob-cli-e7 in this coder's turn

There is no later land agent. Do not wait for this turn's own commit, push, SHA, or
post-commit CI before closing.

- Re-read `bob-cli-e7` and every descendant and note. Confirm this tale, together with
  the landed parse-spans tale, fulfills every remaining requirement of
  `plan:202610/wikilink_task_picker.md`. Recheck commits that landed in either
  repository since `a844d3e1` and `ef07ce2`, and integrate any direct conflict.
- Run `sase bead epic-symbols bob-cli-e7` (empty at audit time; recheck) and resolve any
  entries under Symvision policy.
- Run `sase bead close bob-cli-e7 --note "<verification>"`. The note must name:
  - the repaired defects, including the fixture regression's cause;
  - compliance with history = all, link_form = shortest, and the thin-client rule;
  - the source and commit drift review;
  - the actual check, CI run, and render-review results;
  - each independent failure with its owner;
  - the follow-up triage already recorded on e7 (note #1: e7.1 #2 and e7.2 #2 were one
    defect, repaired by `c0f94390` under active `bob-cli-e6`; no task filed).

  Never force-close.

- Run `just symvision` if either repository defines it. Neither did at audit time; if
  still absent, record that.
- In the `plans` repo opened with `/sase_repo`, set `status: done` in the frontmatter of
  `plan:202610/wikilink_task_picker.md`, `plan:202610/wikilink_picker_landing.md`
  (superseded by this tale), and `plan:202610/wikilink_task_parse_spans.md` (landed).
  Resolve each path with `sase artifact path`, never a hard-coded workspace path.
- `bob-cli-e7` had no `parent_bead` at audit time. Confirm with `sase bead read`; if one
  now exists, follow the normal non-forced ancestor readiness rules.
- Finish with `/sase_final`, declaring every modified repository (bob-cli,
  bob-mac-capture, plans).
