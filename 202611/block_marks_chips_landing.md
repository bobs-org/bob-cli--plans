---
tier: tale
size: medium
title: Finish blocked-reason chip marks and land epic bob-cli-ft
goal: "DEPENDS ON and BLOCKS chips show a Blocked target's block mark and reason line.
  They refresh on every block-marks trigger and fall back to the plain ? as soon as the
  marks are toggled off. Real Obsidian e2e screenshots verify them in light and dark.
  The contract's live checklist is final, and epic bob-cli-ft is closed with its plan
  marked done.

  "
proposed_by: bbugyi200.athena.bob-cli-ft.land
proposed_with: claude/opus
bead: bob-cli-ft
create_time: 2026-11-02 14:08:38
status: wip
---

- **PARENT:**
  [202611/blocked_reason_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/blocked_reason_marks.md)
- **BEAD:**
  [bob-cli-ft](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ft/README.md)

# Tale: Finish blocked-reason chip marks and land epic bob-cli-ft

## Context

Epic `bob-cli-ft` (Blocked-reason marks) is in landing. The authoritative design is the
epic plan `plan:202611/blocked_reason_marks.md`. Read it with
`sase artifact read plan:202611/blocked_reason_marks.md "<why>"`. Focus on the `chips`
and `visual_review` phases and the cross-cutting rules. The display contract is bob-cli
`docs/blocked-marks.md`.

The land audit verified phases `.1` contract, `.2` model, `.3` editor marks (child epic
`.3.1`), `.4` redundancy folds, `.5` Tasks results, `.7` Task Card, and `.8` visual
review against source and commits. The final DECISIONS were honored: own_first, chevron,
quiet, and memory_decision_record = no (no decision strand exists). The only drift since
the epic started is gcal, capture, relation-successor, Stats, cycler, and the new
frees-marks contract (`bob-cli-g1`), and none of it touches block marks. Follow-up
triage is already done and recorded on the epic (see the notes in
`sase bead read bob-cli-ft`). Do **not** redo it, but copy its outcomes into the close
note.

The chips phase (`bob-cli-ft.6`, bob-plugins commit `93d3859`) left these epic-caused
gaps. They are this tale's work.

1. **Chips never refresh on block-mark triggers.** `runBlockMarksRefresh` in
   `plugins/bob-ledger-tools/src/142e-block-marks-runtime.js` refreshes block-mark
   editors, priority marks, reading sweeps, and folds. It never dispatches the
   dependency-chips refresh effect. The DEPENDS ON ViewPlugin
   (`270-plugin-dependency-model.js`, `dependencyChipShouldRebuild`) and the BLOCKS
   ViewPlugin (`272-plugin-blocks-model.js`, `blocksChipShouldRebuild`) both rebuild on
   `ensureDependencyChipsRefresh()`, but that effect is only sent from the Tasks
   cache-update fan-out. So toggle, ladder change, midnight rollover, and resume leave
   chip marks stale.
2. **Toggling off blanks marked chip boxes.** A marked chip box holds only the miniature
   `span.bob-block-mark`. That element has no children, and only the gated
   `body.bob-block-marks …` CSS paints it. Once `toggle-block-marks` removes the body
   class, the box shows nothing instead of `?`. In Live Preview this lasts until the
   next doc or selection change. Reading-view chips are rendered once by the
   post-processor (`280-plugin-dependency-render.js`, `281-plugin-blocks-render.js`), so
   there the box stays blank until the view re-renders.
3. **The chip tooltip appends the whole block-mark tooltip.** `dependencyChipModel`
   (`140-dependency-model.js`) and `blocksBuildChip` (`141-blocks-model.js`) do
   `chip.tooltip += " · " + blockMark.tooltip` (likewise `ariaLabel`). That drags in
   line 2 of the tooltip, such as `Ctrl+Shift+B to edit prerequisites`,
   `Ctrl+Shift+P to reschedule…`, or `Clears on the next reconcile`, plus `Also …`
   lines, all joined with `\n`. Those hints describe the target row, not the chip. The
   epic plan specifies the target's **reason line**, for example
   `Blocked · P2 deferral · until Fri, Nov 20 · in 19 days`.
4. **Chips skipped the phase's ship steps.**
   - No manifest bump: bob-ledger-tools is still `1.92.0` from `tasks_results`.
   - No bob-plugins docs mention chip marks.
   - The e2e never got the planned `DEPENDS ON` line with `p2`, `dep`, and `prj`
     targets, so no screenshot has ever shown a chip mark.
5. **The contract's live checklist is not final.** `docs/blocked-marks.md` → _Live
   verification_ still ends with "Still pending for the parent visual-review phase: …".
   It also lists neither folds, chips, nor the Task Card. Visual_review step 3 required
   a final checklist with only the human or device items left pending.

## Work

### bob-plugins (linked repo)

Open it with your `/sase_repo` skill: `sase repo open bob-plugins -r "<why>"`. Read its
`AGENTS.md` and work only in the printed path. Edit only `src/` fragments, never
`main.js`. Keep every hand-edited fragment at or below 1000 lines.
`142g-block-marks-reading.js` is already at 990, so do not grow it.

1. **Refresh wiring.** In `runBlockMarksRefresh`, add a guarded call to
   `this.refreshDependencyChipEditors()` (defined in `280-plugin-dependency-render.js`).
   It dispatches `ensureDependencyChipsRefresh()` to every markdown leaf, which rebuilds
   both DEPENDS ON and BLOCKS widgets. The widgets' `eq()` keys are
   `JSON.stringify(model)`, so a changed or removed `blockMark` re-renders. Add a node
   test proving that `runBlockMarksRefresh` (and therefore the toggle) calls it.
   `scripts/test-ledger-tools-block-marks-editor.cjs` already stubs this runtime.
2. **One chip-box helper with a CSS fallback glyph.** The four chip-box builders repeat
   the same try/`blockMarkRender`/text-node block:
   - `buildDependencyChipElement` in `150-dependency-element.js`;
   - `buildBlocksChipElement` in `141-blocks-model.js`;
   - the reading anchor box in `280-plugin-dependency-render.js`;
   - `blocksAppendAnchorContent` in `281-plugin-blocks-render.js`.

   Replace those copies with one never-throwing helper, for example
   `dependencyChipBoxFill(box, chip, renderFn, doc)` in `150-dependency-element.js`.
   - When the miniature renders, it appends the mark **and then** the chip's plain
     symbol in `span.bob-dep-chip-fallback`.
   - Otherwise it appends the plain symbol text exactly as today, so unmarked chips keep
     their current DOM.

   In `plugins/bob-ledger-tools/styles.css`:
   - `body.bob-block-marks .bob-dep-chip-box > .bob-block-mark + .bob-dep-chip-fallback`
     hides the fallback glyph (clip pattern or `display: none`, never `!important`);
   - `body:not(.bob-block-marks) .bob-dep-chip-box > .bob-block-mark` is
     `display: none`.

   Toggling off then restores `?` immediately in both views through CSS alone, and the
   Live Preview rebuild from step 1 drops the mark. Reading-view chips keep the reason
   text in their tooltip until the view re-renders. Document that as a known limit;
   don't build a reading-chip sweep.

3. **Reason line only.** Add a pure helper, for example
   `blockMarkReasonLine(model) → string` (the first `\n`-separated line of
   `model.tooltip`, trimmed, `""` on bad input). Put it in `140-dependency-model.js` so
   both 140 and 141 see it. Make both chip builders append only `" · " + reasonLine` to
   `tooltip` and `"; " + reasonLine` to `ariaLabel`, and only when the line is
   non-empty.
4. **Unit tests.** Extend `scripts/test-ledger-tools-dependency-chips.cjs` and
   `scripts/test-ledger-tools-blocks-chips.cjs`:
   - use a two-line `dep` tooltip and assert the chip tooltip and aria hold line 1 and
     no `Ctrl+Shift`;
   - cover `prj`, `p2`, and `stale` miniatures;
   - assert a marked box holds `span.bob-block-mark` followed by
     `span.bob-dep-chip-fallback` with the plain symbol;
   - assert a neutral (null) model keeps today's lone text glyph;
   - assert that a throwing render falls back to the text glyph.

   Add a CSS assertion, in the existing block-marks contrast/CSS test or the chips test,
   that both fallback rules exist and are body-gated.

5. **Real-Obsidian e2e.** Extend `scripts/obsidian-block-marks-e2e.cjs`.
   - Seed a **separate** note so existing BM markers and cases stay untouched. Give it:
     - a source task with a first-child `⛓️ **DEPENDS ON:**` line whose links target
       three `[?]` rows carrying block ids. Use a BM1-style `p2` row, a BM4-style `dep`
       row (it needs its own open prerequisite), and a BM7-style `prj` row. The line
       format is in bob-cli `docs/task-dependencies.md` §2.
     - one Blocked dependent so that a target's generated `🧱 **BLOCKS:**` row shows a
       marked chip.
   - Assert in Live Preview and in reading view:
     - `.bob-dep-chip-box > span.bob-block-mark[data-bob-block=…]` carries the expected
       key per target;
     - the chip tooltip or aria holds the reason line and no `Ctrl+Shift`;
     - the chip row height is equal with marks on and off.
   - Toggle off with the real command and assert:
     - no visible `.bob-block-mark` (computed `display: none`, or removed after the Live
       Preview rebuild);
     - each Blocked chip box visibly reads `?`.
   - Toggle back on and assert the marks return.
   - Capture light and dark screenshots of the chip rows in both views and **look at the
     PNGs**. Fix polish (miniature size and baseline inside `.bob-dep-chip-box`, pip
     clipping, chip height) until they look right.
   - Keep the offline `--self-test` wrapper
     (`scripts/test-obsidian-block-marks-e2e.cjs`, in `npm test`) passing.
   - Run with `xvfb-run -a node scripts/obsidian-block-marks-e2e.cjs` (see `parseArgs`
     for the output dir and options).
   - If Chromium screenshots hang (`bob-cli-fz`), rerun once. Record any remaining
     harness failure honestly; never weaken an assertion to pass.
6. **Ship.**
   - Bump `plugins/bob-ledger-tools/manifest.json` minor (`1.92.0` → `1.93.0`, or the
     next minor if master moved). In its description, extend the Depends-On sentence:
     Blocked chips draw the target's miniature block mark and reason line.
   - Update the chips paragraph in `docs/bob-ledger-tools.md` the same way, including
     the toggle fallback and the reading-tooltip limit.
   - Run `npm run build`, then the focused chip, block-marks, and editor suites, then
     `npm test`. The only expected failures are the pre-existing `bob-cli-et`,
     `bob-cli-f7`, `bob-cli-eh`, and `bob-cli-ff`, plus a possible `bob-cli-f8`/
     `bob-cli-ej` load flake. Confirm any other failure reproduces on clean master
     before calling it unrelated.
   - Run `npm run validate`, `npm run docs:gen`, and `npm run docs:check`. If a bare
     docs:check hits a stale bob-cli cache clone, set `BOB_CLI_DIR` to the bob-cli
     checkout.
   - Run `bob plugins sync`. From inside the linked checkout, use
     `--repo <printed bob-plugins path>`. If sync refuses over another checkout's
     provenance (`bob-cli-fy`, `bob-cli-e4`), read the refusal and resolve it
     deliberately. Never `--force` over someone else's uncommitted deploy, and record
     the outcome.

### bob-cli (this repo)

7. **Contract** `docs/blocked-marks.md`:
   - _Surfaces and interaction_: under the chips row, state that the box holds the
     target's miniature, the tooltip gains the target's reason line (line 1 only), and
     toggling off restores the plain `?` immediately through CSS. Add the
     reading-tooltip known limit.
   - _Reliability and invalidation_: the block-marks refresh also rebuilds Live Preview
     DEPENDS ON and BLOCKS chips.
   - _Live verification_: add verified bullets only for what actually ran:
     - folds (from the existing e2e fold assertions; check
       `obsidian-block-marks-e2e.cjs` `collectFolds`);
     - chips (step 5);
     - the Task Card, verified by node view/model suites rather than the e2e, and say
       so.

     Replace "Still pending for the parent visual-review phase: …" with a final
     **Pending (Bryan, in Obsidian)** list of exactly the human or device items:
     - iOS glyph rendering;
     - Style Settings and third-party checkbox packs;
     - a 100-task project note scrolling smoothly;
     - a real reconcile turning a row `stale` and then clear.

   - Touch `docs/task-dependencies.md` (the chips bullet near "draws its own miniature
     block mark and reason") only if its wording now disagrees.
   - Run `just docs-check` until it passes.

8. **Verify** with `just check` in bob-cli. Do not run `just check-full`.

### Close out epic bob-cli-ft (final step, same turn)

Nothing here waits for this tale's own commit, SHA, push, or CI. The finalizer commits
bob-cli and bob-plugins after the turn.

9. Run `sase bead epic-symbols bob-cli-ft`. It currently reports none. If entries
   appear, resolve each one (wire up, privatize, pragma, or delete per the Symvision
   epic-whitelist policy), or re-key it only to a still-open bead that needs it.
10. Close the epic with `sase bead close bob-cli-ft --note "<verification>"`. The note
    must state:
    - phases .1–.8 (and child epic .3.1) were verified against source and commits, and
      DECISIONS own_first, chevron, quiet, and memory_decision_record = no were honored
      (no decision strand);
    - post-epic drift was reviewed with no block-marks conflict, and frees-marks
      (`bob-cli-g1`) is independent;
    - the follow-up triage outcomes:
      - `bob-cli-et`, `-f7`, `-eh`, and `-ff` were corroborated with a reproduction at
        bob-plugins `93d3859`;
      - `bob-cli-f8` passed in isolation, so it got no new +1;
      - new feature tasks `bob-cli-g6` (Rust BlockReason mark key plus shared JSON
        fixture) and `bob-cli-g7` (move `task-statuses.css` into bob-ledger-tools);
    - this tale's fixes and the checks you actually ran, with results, including the e2e
      pass count, the manifest version, and the sync outcome.

    Never use `--force` to make the close succeed. If the close is rejected, fix the
    cause and close again.

11. Run `just symvision` if the recipe exists. bob-cli's justfile has none today, so
    confirm with `just --list` and record that it is unavailable.
12. Set `status: done` in the frontmatter of the epic's plan file. That is the absolute
    path printed under **PLAN** by
    `sase bead read bob-cli-ft -r "Need the plan path for closeout"`, in the `plans`
    sidecar. Use `/sase_repo` (`sase repo open plans -r "<why>"`) if you need the
    checkout.
13. `bob-cli-ft` has no `parent_bead`, so nothing further is needed after the close.

## Done when

- Marked DEPENDS ON and BLOCKS chips refresh on toggle, ladder, rollover, and resume.
- Toggling off shows `?` in both views, and the chip tooltip carries only the reason
  line.
- The unit suites and the extended real-Obsidian e2e pass, and the screenshots were
  reviewed.
- bob-ledger-tools is bumped, documented, and synced.
- The contract's live checklist is final.
- `bob-cli-ft` is closed, and its plan file says `status: done`.
