---
tier: tale
size: medium
title: Finish the Bob Mac Capture merge-in landing and close bob-cli-hl.5.1
goal:
  Merge-in VoiceOver copy, the pending card's new-task header, the refusal submit guard,
  and the plan's missing hosted tests land in bob-mac-capture, then epic bob-cli-hl.5.1
  and its parent phase bob-cli-hl.5 close with verified evidence.
proposed_by: bbugyi200.athena.bob-cli-hl.5.1.land
proposed_with: claude/opus
bead: bob-cli-hl.5.1
create_time: 2026-11-06 07:50:44
status: done
---

- **PARENT:**
  [202611/capture_merge_in_mac.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_merge_in_mac.md)
- **BEAD:**
  [bob-cli-hl.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hl/bob-cli-hl.5.1.md)

# Finish the Bob Mac Capture merge-in landing

## Why this tale exists

The land agent for epic **bob-cli-hl.5.1** (`plan:202611/capture_merge_in_mac.md`)
checked all five closed phases against the code on `bob-mac-capture` master `a3e533b`
(the wire, editor, band, landing, and rollout commits `b4b343f`, `8c956ca`, `e30a2f4`,
`6d659eb`, and `a3e533b`). It confirmed these as done: tolerant decoding and 31
fixtures, the merge-in picker role and Bob-authored splices, span colors, revision-bound
pending state, explicit Unicode-safe rewrites, the complete band with **Show all N** and
de-duplicated consequences, the primary action, live and committed status, notification
title and body, origin folds for both arrows with the exact target block shown once,
grammar gate 15 with old/missing rejection, docs, and 84 reviewed renders. The macOS CI
run for `a3e533b` (42216904060) has no new deterministic failures. Its 13 recurring
failures are a subset of the 16 that pre-epic master `6f4e279` already failed (run
42161345004), and bob-cli-h7 tracks them. Its two other failures are flakes on an
identical tree, now tracked as bob-cli-hu and bob-cli-hv. bob-cli itself has had no
capture changes since the epic started: `GRAMMAR_REVISION` is still 15, and only Gmail
and successor-verdict copy changed.

The landing found four gaps the epic plan required but its phases did not deliver. This
tale fixes them and then performs the whole landing closeout.

1. **VoiceOver copy is off-spec.** `CapturePanelModel` posts
   `"Now ⤵ merging in N tasks into “T”"`. It has no source summaries and no spoken
   status transitions, and it includes the glyph. The band's
   `previewAccessibilitySummary` puts the header before the new-task identity. The
   contract (parent plan copy table) is:
   `New task Some great new task!, foo.md, new. Merging in 2 tasks: Fix the flaky login test, Next to Cancelled, 2 notes; …`.
   It is announced only when settled, never per keystroke, and never moves focus.
2. **The pending card omits the new task's parsed header.** The plan requires the parsed
   header plus `⤵ Merge tasks into this one — type + to pick` and the resolved sources.
   `CaptureMergeInPendingCardView` shows only the prompt and sources.
3. **Refused input stays submittable.** The plan says to disable submission for
   incomplete _and refused_ input. `primaryActionEnabled` and `submit` guard only
   `mergeInNeedsSource`. With a merge-in refusal card on screen, ⌘↩ still runs a real
   `bob capture`. Bob refuses it, but the panel then flashes a red failure.
4. **Hosted coverage is missing.** The epic added only three hosted model tests: pending
   revision binding, the explicit rewrite action, and ID-less pick/cancel. The plan's
   required hosted cases are absent: complete single-item action, status, and
   announcement; mixed bulk keeping **Capture**; the label-driven merge band; the model
   path from a `merge_in_*` dry-run failure to the refusal card with Bob's rewrite
   buttons; executable-switch invalidation; and notification strings.

## Rules

- Open `bob-mac-capture` with `/sase_repo` and work only in the path it prints. Read its
  `AGENTS.md` first. Paths below are relative to that checkout. Never touch Bryan's live
  install, his Mac checkout, or the vault.
- `decisions:mac-capture-is-a-thin-client` holds. Read it with `/sase_memory_read`.
  Every spoken or displayed fact comes from Bob's parse or capture JSON. Swift arranges
  it and never scans the draft for `<` or computes merge effects.
- No memory edits, no bob-cli code changes, no Cargo version or changelog edits, no
  grammar changes. Format only changed lines (`.swift-format`, 4 spaces, 100 columns).
- Do not reopen product decisions from the epic plan or `bob-cli-hl` DECISIONS. In
  particular, `pull_in_title_count = yes` stays final.
- Do not fix the bob-cli-h7 baseline cluster, the bob-cli-hu or bob-cli-hv flakes, the
  bob-cli-hs docs snapshot, or the bob-cli-ge stale fixture audit here. Each is tracked.
  Record any genuinely new follow-up through `/sase_new_task`.

## Work

### 1. Spoken merge-in copy (`Sources/CaptureCore/CaptureMergeInPresentation.swift`)

- Add a public `settledAnnouncement` built only from existing presentation fields:
  - Sentence 1 is the new-task identity:
    `New task <title>, <file name of relativeTarget>`. Append `, new` when
    `destination.isNew`, and `, <statusName>` when Bob supplied one.
  - Sentence 2 is the sources: `Merging in <N> task(s): <src>; <src>; …`, using singular
    `task` for one.
  - Each `<src>` is the title, then `, <previous> to <target>` when both status names
    are present and differ, then `, <moved-note text>` (`1 note` / `N notes` /
    `title only`).
  - Fallback rows (no enriched status) speak the title only.
  - Join the sentences with `. `. No `⤵` or other glyphs.
- Reorder `previewAccessibilitySummary` to the same identity-first form: identity, then
  the sources sentence, then each consequence text, then the Pull-In offers sentence.
  The band summary may keep each source's locator after its transition. Remove the glyph
  from spoken strings only. `headerText` keeps `⤵` for display.
- Give `CaptureMergeInPendingPresentation` an optional `targetTitle: TaskDisplayText?`,
  passed through `init?(pending:targetTitle:)` with a default of `nil` so the existing
  call sites and tests still compile. When it is present, its accessibility summary
  starts with `New task <title>. ` before the prompt. Strip the `⤵` glyph from the
  spoken prompt there too.
- Extend `Tests/CaptureCoreTests/CaptureMergeInPresentationTests.swift` to pin:
  - the exact announcement for `capture-merge-in-complete-two.json` and
    `capture-merge-in-complete-one.json` (singular);
  - title-only and fallback rows;
  - no `⤵` in any spoken string;
  - pending summaries with and without a target title.

### 2. Model wiring (`Sources/BobMacCapture/CapturePanelModel.swift`)

- In the preview-apply path that currently builds `mergeInAnnouncement`, assign
  `nextMergeIn.settledAnnouncement`. Change `lastPreviewMergeInKey` to `relativeTarget`
  plus the ordered source `stableID`s and each source's target status name, and leave
  out the new-task title. Re-typing the title over settled sources must not re-announce,
  and an identical re-preview must not bump `mergeInAnnouncementTick`. A changed source
  set or count must announce once. Keep `.onChange` posting through
  `AccessibilityNotification.Announcement` with no focus change.
- Add a `currentMergeInPendingTargetTitle: String?` helper. It reads Bob's parse `body`
  for the pending item: the one-based `pending.item` into `latestParse.items`, or
  `latestParse.body` when `items` is empty, trimmed, `nil` if empty. It returns `nil`
  whenever `currentMergeInPending` is `nil`. Reuse the scope logic of
  `mergeInTargetTitle(in:replacement:)` where practical, and never scan the draft.
- Refused input: add `&& mergeInRefusalPresentation == nil` to `primaryActionEnabled`.
  In `submit(openAfterCapture:)`, right after the `mergeInNeedsSource` guard, return
  early while a merge-in refusal for the current draft is showing. Set `statusText` to
  the refusal message and do not run `bob capture`. `setPlainDraft` already clears the
  refusal on any edit, so a rewrite or edit re-enables Return after the next analysis.
  Leave transfer refusal behavior unchanged.

### 3. Views

- `Sources/BobMacCapture/CaptureMergeInBandView.swift`: `CaptureMergeInPendingCardView`
  shows the target title, when present, as a one-line semibold row above the calm
  prompt, using tail truncation, and drops the prompt's weight to regular. Keep the
  secondary (non-red) styling for a source-less draft. Source errors stay red inline.
- `Sources/BobMacCapture/CapturePanelView.swift`: build the pending presentation with
  `targetTitle: model.currentMergeInPendingTargetTitle`.
- `Tests/BobMacCaptureTests/CaptureMergeInDesignTests.swift`: update
  `testRenderPendingSourcesAndErrors` so its presentation carries a target title (for
  example `New outcome`). Re-render the pending fixtures at the existing widths and
  appearances, and inspect the 620-pt light and dark results in the CI render artifact
  for truncation and contrast.

### 4. Hosted tests (new `Tests/BobMacCaptureTests/CaptureMergeInPanelTests.swift`)

Follow `CapturePanelModelTests` patterns: the fake-bob `processClient`,
`CapturePanelModel(debounceNanoseconds: 0)`, and bounded `waitUntil` with an explicit
timeout. Never await an untimed continuation. Type each parse fixture's exact `input` as
the draft so revision guards match. Use the existing fake-bob knobs:
`FAKE_BOB_PARSE_FIXTURE`, the capture dry-run fixture/stdout variables, and
`FAKE_BOB_CAPTURE_DRY_RUN_EXIT`. Read `Tests/Fixtures/fake-bob` for the exact variable
that serves a dry-run fixture file. Cover:

1. **Complete single item.** Use `capture-merge-in-parse-two.json` with
   `capture-merge-in-complete-two.json`. Assert:
   - `primaryActionTitle == "Capture & merge 2 tasks"` and `primaryActionEnabled`;
   - `statusText` equals the presentation's `statusLineText`
     (`Would create … and merge 2 tasks into it · …`);
   - `mergeInAnnouncement` equals the pinned settled string and the tick is 1;
   - re-running analysis on the identical draft leaves the tick at 1.
2. **Mixed bulk.** Use `capture-merge-in-parse-bulk-mixed.json` with
   `capture-merge-in-complete-task-list.json` (two captures). Assert
   `primaryActionTitle == "Capture"`, `mergeInPresentation == nil`, and no announcement
   tick.
3. **Label-driven merge.** `capture-merge-in-complete-label.json` yields a band
   presentation and `Capture & merge 1 task`.
4. **Refusal end to end.** Use `capture-merge-in-parse-rewrite-spacing.json` (input
   `text<@inbox+aaa`) with a dry-run stdout of
   `{"ok":false,"code":"merge_in_arrow_spacing","error":"capture item 1 starting on line 1: put spaces around <: New task < @a+x"}`
   and exit 1. This is real Bob 0.4.0 output for that draft.
   - `mergeInRefusalPresentation?.rewrites.map(\.label) == ["Add spaces"]` and
     `primaryActionEnabled == false`.
   - `submit(openAfterCapture: false)` leaves the draft unchanged and records no
     non-dry-run `capture` argv.
   - `applyDiagnosticRewrite` clears the refusal and yields `text < @inbox+aaa`.
5. **Executable switch.** After pending state is established (reuse the
   `capture-merge-in-parse-pending-plus.json` setup from
   `testMergeInPendingPreviewStaysCalmAndIsBoundToItsDraftRevision`), assigning a
   different `processClient` makes `currentMergeInPending` and
   `currentMergeInPendingTargetTitle` nil. Before the switch, assert the target title is
   Bob's parsed body (`New outcome`).
6. **Notification.** In `NotificationServiceTests`, add a case through the existing
   `successContent` seam for `capture-merge-in-complete-two.json`. Its title is
   `Created “<T>” · merged 2 tasks` and its body has one line per source,
   `<title> · <prev> → Cancelled · <n notes>`, matching `CaptureMergeInPresentation`.

### 5. Docs

`docs/capture-previews.md` § Merge-in band needs one short sentence each:

- Pending names the new task above the prompt.
- VoiceOver announces the new task, then each source's transition and note count, once
  per settled source change.
- A refusal disables Return until a rewrite or edit.

Read `docs.md` memory with `/sase_memory_read` before editing, and stay within the
mounted page's word budget. Run `just docs-check` in `bob-mac-capture` and in bob-cli.
In both, the only accepted failure is the known bob-cli-hs README contract-table
snapshot (`test_readme_table_matches_original_contract_index_snapshot`). Report any
other failure.

## Verification

- `just portable-check` on Linux must pass (2239+ tests). Also run the focused
  `swift test --filter CaptureMergeInPresentationTests`.
- Mac scratch compile per `AGENTS.md`: read `tailnet.md` memory, rsync without
  `.git`/`.build`, run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the
  copy. If the Mac is offline, say so and rely on CI.
- Run `Scripts/macos-ci-check` through `/sase_monitor` on the exact working tree, and
  wait for the monitor handoff command itself to exit. Lint and Build must pass. Every
  new and changed merge-in test must pass. Any remaining Test failures must be among the
  bob-cli-h7 cluster of 13 names (word-motion undo, two comma-route tests, Pull-In
  footer hints, nine `CaptureSeparatorToggleTests`) or the bob-cli-hu/hv flakes. Diff
  the failing-name list against that set and record the run ID and snapshot SHA. Any
  other failure is this tale's work. Never call a red run green.
- This exact-tree check is the Mac gate for the landing. The closeout below must not
  wait on the CI run for this work's own pushed commit.

## Closeout (final step; do it in the same turn as the final code)

1. Run `sase bead epic-symbols bob-cli-hl.5.1` and
   `sase bead epic-symbols bob-cli-hl.5`. Both were empty at landing. Resolve any new
   entry (wire, privatize, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy) rather than re-keying it, since no later open bead owns this
   work.
2. Close the epic: `sase bead close bob-cli-hl.5.1 --note "<verification>"`. The note
   summarizes this tale's changes and evidence (portable-check count, scratch-compile
   result, exact-tree CI run ID and snapshot SHA with the baseline-subset comparison,
   docs-check results). It also restates the landing triage already recorded on the
   epic's notes:
   - hl.5.1.2 #1: declined, fixed in-epic by `6d659eb`.
   - Baseline Mac failures: +1 bob-cli-h7.
   - Docs snapshot: +1 bob-cli-hs.
   - clip-history regeneration: declined, as those fixtures carry no grammar field and
     are unaffected.
   - Stale grammar-6/12/14 fixtures: +1 bob-cli-ge.
   - New flakes: bob-cli-hu and bob-cli-hv.

   Never use `--force` to make the close succeed.

3. Run `just symvision` if either repository defines that recipe. Neither did at landing
   time; say so if that is still true.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path printed
   by `sase bead read bob-cli-hl.5.1 -r "<why>"` (`202611/capture_merge_in_mac.md` in
   the plans sidecar). Change nothing else in that file.
5. The epic's `parent_bead` is the **phase** bob-cli-hl.5 ("mac: render the Merging-in
   band inside the new-task card, pending and refusal states with rewrite buttons,
   merge-in pickers, folded landing groups, submit/status/notification/VoiceOver copy,
   and raise the grammar gate"). Check that bob-cli-hl.5.1 now delivers every clause of
   that description. Then run
   `sase bead close bob-cli-hl.5 --note "<implemented behavior and concrete verification>"`.
   Do **not** close `bob-cli-hl` or any other ancestor, because its own land agent is
   already waiting.
6. Finish through `/sase_final` so the `bob-mac-capture` changes and the plan-file
   status edit are committed with Conventional Commit messages, for example
   `feat(capture): finish merge-in VoiceOver, pending header, and refusal guard`.
