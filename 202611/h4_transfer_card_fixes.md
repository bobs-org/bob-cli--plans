---
tier: tale
size: medium
title: Finish transfer previews — Mac card fixes and the bob-cli-h4 closeout
goal:
  Bob Mac Capture transfer previews meet the epic plan with no stale or misleading copy.
  Flipping a draft between move and merge announces once to VoiceOver, the footer never
  keeps a pending or refusal title under a valid card, section merges read `into db.md §
  Future Work`, renamed moves show an orange detail, unchanged-status moves show no
  transition chip, and the pending card has a real VoiceOver label. Epic bob-cli-h4 is
  then closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-h4.land
proposed_with: claude/opus
bead: bob-cli-h4
status: done
---

- **PARENT:**
  [202611/transfer_previews.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/transfer_previews.md)
- **BEAD:**
  [bob-cli-h4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h4/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-h4.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-h4.land.md)
- **COMMITS:**
  - [a4aa18a](https://github.com/bobs-org/bob-cli--plans/commit/a4aa18ad7d6b0592be6c1aae3295dd56859bd92e)
    — chore(plans): mark transfer_previews done for bob-cli-h4

# Finish transfer previews: Mac card fixes and the bob-cli-h4 closeout

## Context

Epic `bob-cli-h4` (plan `plan:202611/transfer_previews.md`) gave Bob Mac Capture a
designed preview for `>` transfers. All six phases are closed, and the epic's lander
verified the bob-cli side end to end. All three bob phases are on master: landing
blocks, enriched records with titled consequences and `merge_offers`, and the
`transfer_pending` dry-run context. Every transfer, merge, move, and note-pull CLI test
passes. A code audit of the three Mac phase commits in `bob-mac-capture` (`e9dc5bd`,
`dd4a8bf`, `6e09f2c`) found five defects the epic caused, plus one VoiceOver nit. This
tale fixes them and then closes the epic. It is the epic's landing, so it must finish
with the closeout in "Step 7".

Work in the linked `bob-mac-capture` repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<why>"`) and read its `AGENTS.md` first. Follow
these house rules from it:

- `.swift-format` sets 4-space indentation and 100-column lines. Format only the lines
  you change.
- Never land Swift you have not compiled.
- Never hand-edit the recorded fixtures under `Tests/Fixtures/`. Build ad-hoc decoded
  values inline in a test instead.
- **Thin client** (decision `mac-capture-is-a-thin-client`): Swift never resolves tasks
  or writes sentences that Bob owns. Every fix below only re-arranges or re-styles facts
  Bob already sent.

Line numbers below are from `bob-mac-capture` `6e09f2c` and may drift; search for the
quoted symbols.

## Step 1: Make the VoiceOver verb-flip announcement fire

The plan requires one announcement when the verb flips for the same draft. Typing
`> @db` previews a move; then `+upgrade` turns it into a merge, and VoiceOver should say
`Now merging into “Upgrade Postgres to 17”`.

**Today it never fires:**

- `CapturePanelModel.assignPreviewOutputs` compares the new transfer with
  `previewResults`.
- `scheduleAnalysis` calls `clearPreviewOutputs()` on every edit (the comment begins "A
  changed draft can still have the prior response…"). So `previewResults` is always
  empty by the time a new response arrives.
- The intermediate `> @db+` pending state also clears it, through `showTransferFailure`
  → `clearPreviewOutputs()`.

**Fix:**

- Keep a model-owned `lastPreviewTransferVerb` (or an equivalent small struct) that
  `clearPreviewOutputs()` does **not** reset.
  - Set it whenever a live preview assigns a single-transfer result.
  - Leave it untouched through transfer pending and refusal states.
  - Reset it to nil when the draft stops being a transfer: a successful non-transfer
    preview, an emptied draft, a submit, a stash or discard, or a panel reset.
- Announce only when both the remembered verb and the new verb exist and they differ.
  Never announce on the first transfer preview.
- Use the destination wording from Step 3, so a section reads
  `Now merging into db.md § Future Work`.
- **Tests:** add a `CapturePanelModel` test using the existing fake-bob preview harness,
  as in `CaptureTransferPendingTests`.
  - Drive move preview → `incomplete_transfer_block_id` pending → merge preview.
  - Assert `transferVerbAnnouncementTick` increments exactly once, and assert the
    announcement text.
  - Add a negative case: a first transfer preview, and a merge → merge re-preview, never
    announce.

## Step 2: Clear the stale footer after a pending or refusal state

**The bug:** `showTransferFailure` sets `statusText` in three places: the pending title
(`⇢ 1 task ready — finish the destination`), the older-bob diagnostic, and the refusal
title (`Can't merge`). Nothing resets that text when the draft becomes a valid transfer:

- The live-preview success chain (the `soleTogglePresentation` / `soleLinkPresentation`
  / … ladder) only clears `"Preview failed"`.
- `applyParse` only normalizes `"Ready"`, `""`, and `"Target cache stale"`.

So typing `> @db+` and then `upgrade` leaves the pending title under a valid merge card.

**Fix:**

- In the live-preview success ladder, add a transfer branch: when the batch is a single
  transfer item, `statusText = CaptureTransferPresentation(capture:)!.statusLineText`.
  The explicit Preview path already does this through `captureStatus`.
- Remember the text that `showTransferFailure` wrote, for example in
  `transferOwnedStatusText`. On any later successful preview or non-transfer parse,
  reset `statusText` to `""` (reads as Ready) when it still equals that remembered text.
  Never clobber unrelated notices, such as rewrite notices.
- The empty note-pull card has the same pattern. It predates this epic, so leave its
  behavior unchanged unless one shared helper fixes both without other changes.
- **Tests:** add model tests for pending → valid merge (the footer shows
  `Would merge 1 task into “…” · …`) and refusal → edited-away non-transfer draft (the
  footer reads Ready). Keep the existing non-transfer status tests unchanged.

## Step 3: Section-merge copy must not quote the file name

**The bug:** a section merge's `merges[0].target` has no `title`. One example is
`Tests/Fixtures/capture-transfer-section-merge.json`:
`{"path":"db.md","section":"Future Work","kind":"section","line":7,"created":true}`.
`CaptureTransferPresentation` then falls back to the file name as if it were a task
title:

- the status line reads `Would merge 1 task into “db.md” · db.md § Future Work`;
- the notification title reads `Merged 1 task into “db.md”` and drops the section;
- `alreadyMergedText` would read `Already merged into “db.md”`.

**Fix** in `Sources/CaptureCore/CaptureTransferPresentation.swift`:

- Track whether the destination title came from Bob, for example
  `DestinationRow.hasTitle`.
- When it did not (section kind, or any merge target without a title), the destination
  phrase is the unquoted `db.md § Future Work`: the display path plus the container
  label. Use that phrase in:
  - `statusLineText` → `Would merge 1 task into db.md § Future Work` / `Merged …`;
  - `notificationTitle`;
  - `alreadyMergedText`;
  - the VoiceOver `previewAccessibilitySummary`;
  - the Step 1 announcement.
- Titled targets keep today's copy exactly: `“T” · bob.md`, plus ` › FUTURE WORK` for a
  drawer.
- The card's destination row is unchanged: it still shows `db.md`, the `§ Future Work`
  capsule, and the dashed `new` capsule.
- **Tests:** update `Tests/CaptureCoreTests/CaptureTransferPresentationTests.swift`
  expectations for the section fixture (status line, notification title, and
  accessibility). Assert that the drawer and task merges are unchanged. If
  `docs/capture-transfers.md` quotes the section copy, update it.

## Step 4: Color a renamed move's block-ID detail orange

The plan requires the move detail to read `^bench`, or `^bench → ^bench-2` **in orange
when renamed**. Today `CaptureTransferCardView.sourceRow` always draws `detailText` in
`.secondary`, and `SourceRow` has no rename flag.

**Fix:**

- Add `detailIsRename: Bool` to `CaptureTransferPresentation.SourceRow`. It is true for
  a move whose `old_block_id` and `new_block_id` are both present and differ, and false
  for merges and fallbacks.
- Draw the detail `.orange` when the flag is set.
- Keep the words, so color is never the only signal.
- **Tests:** add a presentation test with an inline-decoded move record whose
  `new_block_id` differs from `old_block_id`. Do not edit the recorded fixtures.

## Step 5: Drop the `Ready → Ready` chip on moves

**The bug:** moves pass the same Bob-reported status as source and target, so every move
row shows a `Ready → Ready` transition chip. The plan's move mockup shows no chip, since
a move does not change status.

**Fix:**

- When source and target status names and symbols are equal, draw no transition chip.
  The leading status glyph already carries the status.
- Omit the `X → X` / `X to X` fragment from the notification line and the VoiceOver line
  too.
- Real changes, such as every merge's `Next → Cancelled`, are unchanged.
- **Tests:** add a presentation assertion on the move fixture (no transition text in
  `notificationBody` or `previewAccessibilitySummary`). The design-render test needs no
  new images; CI re-renders them.

## Step 6: Name the pending card for VoiceOver

`CaptureTransferFailureCardView` gives the pending card the generic
`.accessibilityLabel("Transfer destination pending")`, and
`CaptureTransferPendingPresentation.accessibilitySummary` is computed but never read.

**Fix:**

- Keep `.accessibilityElement(children: .contain)`, so the rows still read as a list.
- Label the container with the presentation's `title`, and attach `accessibilitySummary`
  as the container's `.accessibilityHint`.
- If you find a clearer equivalent, deleting the unused property instead is fine, as
  long as nothing is left unused.

## Not in scope (decided by the lander; do not change)

- **The pending header count includes unresolved sources.** The plan's mockup shows
  `2 tasks ready` above one `✕` typo row, so `CaptureTransferPendingPresentation`'s
  count of named sources is intended.
- **The repair line (`N links repaired in M notes`) and the form-legend tinting.** The
  plan explicitly asked Swift to produce both from Bob's fields.
- **The 13 macOS Test-step failures on clean master** (backslash, word-motion,
  comma-route, and separator-alias tests). They are tracked by task `bob-cli-h7`. Do not
  fix or skip them here.

## Verification (all before the closeout; none needs this work's own commit)

1. Run `just portable-check` on athena. It must pass with 0 failures.
2. Compile a scratch copy on the mac over the tailnet, as `AGENTS.md` describes:
   - rsync without `.build`/`.git`;
   - run `Scripts/bundle.sh` and `just typecheck-tests`;
   - delete the copy, even on failure.

   Quote remote paths so the remote shell sees the literal path. Never use Bryan's
   checkout or `just install`. If the mac is offline, say in the final report that the
   macOS targets are unverified.

3. Run `Scripts/macos-ci-check` from the opened `bob-mac-capture` checkout. This is the
   pre-landing exact-snapshot CI gate.
   - Build must pass.
   - `CaptureTransferPresentationTests`, `CaptureTransferPendingTests`,
     `CaptureTransferCardDesignTests`, the transfer `NotificationServiceTests`, and the
     new model tests must pass.
   - The Test step may still be red, but only on the clean-base set tracked by
     `bob-cli-h7`. Compare failed test names against master run `42079206476`
     (`6e09f2c`).
   - If a hang or watchdog appears, treat it as yours until the logs show otherwise.
4. If any doc changed, run `just docs-check` in `bob-mac-capture`.

Do not wait for, or order the closeout after, the post-push CI run of this tale's own
commit. `/sase_final` commits after the closeout.

## Step 7: Close out epic bob-cli-h4 (this tale's final step)

Run these from the bob-cli workspace:

1. `sase bead epic-symbols bob-cli-h4`. The lander found no entries. If any appear,
   resolve each one: wire it up, privatize it, add a non-test pragma, or delete it per
   the Symvision epic-whitelist policy. Re-key the Justfile line to a bead only when a
   still-open later bead needs the exemption.
2. Close the epic with `sase bead close bob-cli-h4 --note "<verification>"`. Never use
   `--force`. Base the note on this text, adjusted to what you actually ran:

   > Lander verified all six phases against plan:202611/transfer_previews.md and the
   > tombstones=cards decision.
   >
   > - **bob-cli** (3ad23a73 / decd465c / 029b4266): transfer task_blocks roles, the
   >   moved_from before-images, section_blocks, MergeRecord status names and titled
   >   consequences, the move destination fields, merge_offers with the merge-only
   >   Pull-In copy, and transfer_pending are all present. Every transfer, merge, move,
   >   and note-pull CLI test passes on master.
   > - **bob-mac-capture** (e9dc5bd / dd4a8bf / 6e09f2c): the card, landing cards,
   >   pending card, and refusal card are complete.
   > - **Integration:** the 35 bob-cli and 4 Mac commits since the epic started do not
   >   conflict. hd.5's batch-level type_agreement is additive and does not touch the
   >   transfer blocks.
   > - **Fixed in this tale:** the verb-flip VoiceOver announcement, the stale footer
   >   after pending/refusal, the section-merge copy, the orange rename detail, the
   >   removed same-status move chip, and the pending-card VoiceOver label. Verified
   >   with <portable-check result>, <mac scratch result or "mac offline: macOS targets
   >   unverified">, and macos-ci-check <run id> (only the bob-cli-h7 clean-base
   >   failures).
   > - **Follow-ups:** bob-cli-h7 +1 for the macOS CI failures. All other phase
   >   proposals were already fixed on master (see epic note). The red master just check
   >   from hd/hc/he is recorded on those epics.

3. Run `just symvision` in bob-cli if the recipe exists. At the lander's check it did
   not, so note that and skip it.
4. Set `status: done` in the frontmatter of the epic's plan file
   `plan:202611/transfer_previews.md`. Its path is the `PLAN` path that
   `sase bead read bob-cli-h4 -r "<why>"` prints. Today it reads `status: wip`.
5. `bob-cli-h4` has no `parent_bead`, so nothing further is needed. Finish through
   `/sase_final`, which must commit both the `bob-mac-capture` changes and the
   plans-repo status edit.
