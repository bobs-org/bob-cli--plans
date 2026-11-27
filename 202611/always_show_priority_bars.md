---
tier: tale
title: Always show the priority bars on Blocked digit rows
goal:
  Every task with a canonical priority field shows its 1–4 bar priority mark in Live
  Preview, Reading view, and Tasks query results, even when its Blocked checkbox shows a
  P1–P4 digit or the raw [?] is revealed.
size: small
proposed_by: bbugyi200.athena.1he
proposed_with: claude/opus
create_time: 2026-11-27 14:34:09
status: wip
---

# Always show the priority bars on Blocked digit rows

## Goal

Every task with a canonical `[priority:: …]` field shows its priority mark (the 1–4
filled signal bars, or the urgent `!` for `highest`). The mark shows in Live Preview,
Reading view, and Tasks query results, whatever blocked-reason mark its checkbox wears.
The hollow digit inside a Blocked `[?]` box (`p1`–`p4` block marks) stays exactly as it
is. Only the rule that hides the bars when the box shows a digit goes away.

## Why (root cause)

bob-plugins commit `b64a4332` ("fold digit bars and project-blocked pills") made the
priority bars fold to zero width on any Blocked row whose block-mark key is `p1`–`p4`,
reasoning that the hollow digit in the box already states the priority. But the digit is
not reliably on screen while the fold is:

- **Live Preview, cursor on the checkbox.** The fold is decided from the line text alone
  (`priorityMarkFoldedForLine`) and is revealed only when the selection touches the
  `[priority:: …]` field itself. When the cursor sits at the start of the line, Obsidian
  reveals the raw `- [?]` markup, so the digit disappears while the bars stay folded.
  The row then shows no priority at all. This is what Bryan's screenshot shows (row 143,
  `[?] #FIX Fix stale ?N markers…`, with `⌛ Tue` and no bars).
- **Tasks query results.** The CSS clip on `.task-priority` fires for every
  `[data-task="?"]` row with a future `data-task-scheduled` and a ladder priority. The
  later `.task-recurring` loop rule in the cascade can replace the digit with the loop
  glyph, but the bars stay clipped, so the row again shows no priority.
- **Any other moment** the box paint is missing or lagging (quoted forms that render no
  checkbox, unpainted rows) has the same failure: the fold does not check whether the
  digit is actually painted.

Folding to save one glyph's width is not worth a row that can lose its priority. Remove
the fold everywhere. The digit stays as the teaching link between the box and the P1–P4
ladder.

## Governing decisions

No decision record governs block-mark or priority-mark display: the roster was checked,
and the nearest is `decisions:task-decay-is-one-ladder`, which only defines the P1–P4
ladder this mark reads. The authoritative display contracts are bob-cli
`docs/blocked-marks.md` and the _Priority marks_ section of bob-cli `docs/projects.md`;
both are updated below.

## Out of scope (keep unchanged)

- The hollow digit (`p1`–`p4`) block marks themselves, in every surface and in the Task
  Card.
- The `#prj_is_blocked` pill fold on `prj` rows and the `#awaiting_reply` pill fold on
  `awaiting` rows (Live Preview widget, Reading-view `data-bob-block-folded`, and the
  Tasks-results tag clips). Only the priority-bar part of the fold machinery goes away.
- The `foldSpace` / `data-fold-space` space-folding of the priority mark. That is
  unrelated: it swallows the space before the field.
- Block-mark classification, tooltips, the Tasks CSS digit cascade, and the session
  toggles.

## Implementation

### bob-plugins (linked repo): open it with `/sase_repo` (`sase repo open bob-plugins`)

Read its `AGENTS.md` first. `bob-ledger-tools` uses the fragment build: edit `src/`
fragments, run `npm run build`, and never hand-edit `main.js`.

1. `plugins/bob-ledger-tools/src/265-plugin-priority-marks.js`
   - Live Preview (`buildPriorityMarkDecorations`, the block near the
     `priorityMarkFoldedForLine` call): drop the `folded` computation and build the
     widget unfolded.
   - Reading view (`renderPriorityMarksIn`, the block near the
     `priorityMarkFoldedForRow` call): drop the `folded` computation and stop passing
     `folded`.
2. `plugins/bob-ledger-tools/src/135-priority-marks.js`: remove the now-dead `folded`
   concept from `buildPriorityMarkElement` (the `settings.folded` read and the
   `data-folded` attribute) and from `PriorityMarkWidget` (the constructor argument,
   `this.folded`, the `"1"/"0"` folded component of its `eq`/key strings, and the
   `toDOM` pass-through). Update the comments that mention folding the bars.
   `api.priorityMarks.render` never documented `folded`, so its public shape is
   unchanged.
3. `plugins/bob-ledger-tools/src/272d-plugin-block-mark-folds.js`: delete
   `priorityMarkFoldedForLine` and `priorityMarkFoldedForRow`. Rewrite the header
   comment so it describes only the `#prj_is_blocked` / `#awaiting_reply` tag folds.
4. `plugins/bob-ledger-tools/src/142h-block-mark-folds.js`: drop the header sentence
   about digit-row priority bars reusing `PriorityMarkWidget` with `data-folded`.
5. `plugins/bob-ledger-tools/src/142c-block-marks-model.js`: delete
   `blockMarkIsDigitKey` and its comment (`272d` is its only caller), and remove it from
   `src/350-exports.js`.
6. `plugins/bob-ledger-tools/src/142e-block-marks-runtime.js`: in
   `runBlockMarksRefresh`, remove the `refreshPriorityMarkEditors()` call. Its own
   comment says it exists only so digit-row bar folds rebuild with the block-marks gate.
   Keep every other step, including `sweepBlockMarkFoldsReading`, which still serves the
   tag folds. Before removing it, grep the other callers of `refreshPriorityMarkEditors`
   to confirm that priority marks still refresh on their own ladder, toggle, and
   rollover paths.
7. `plugins/bob-ledger-tools/styles.css`
   - Delete the three `body.bob-block-marks .bob-priority-mark[data-folded="true"] …`
     rules and their comment (around the `.bob-priority-mark[data-fold-space="true"]`
     rule).
   - Delete the source-note clip
     `body.bob-block-marks li.task-list-item[data-bob-block="p1"…"p4"] .task-priority`
     and its comment.
   - In the Tasks-results rule list that starts with the comment "Digit rows already say
     the priority on the box; …", delete only the first selector: the
     `…[data-task-scheduled^="future-"]:is([data-task-priority="high"], …) … .task-priority`
     one. Keep the `#prj_is_blocked` and `#awaiting_reply` tag-clip selectors and the
     shared clip declarations. Reword the comment so it no longer claims digit rows clip
     their bars.
8. Bump `plugins/bob-ledger-tools/manifest.json` by one patch version (this repo's
   convention for ledger-tools fixes). Then run `npm run build`.
9. Tests (all under `scripts/`):
   - `test-ledger-tools-priority-marks.cjs`: replace the
     `"PM18/PM19 digit rows fold only while block marks are on"` test with one asserting
     that a `p2` digit row (PM18) builds an unfolded priority widget in Live Preview
     with block marks on and off, and that `renderPriorityMarksIn` on an `li` carrying
     `data-bob-block="p2"` emits a visible mark (no `data-folded="true"`). Drop the
     `data-folded` assertions in the `buildPriorityMarkElement` test.
   - `test-ledger-tools-block-marks.cjs`: in
     `"fold helpers: digit keys, BM1/BM8/BM9 bars, BM7/BM19 pills"`, remove the
     `blockMarkIsDigitKey` assertions and the import. Flip the CSS assertion so
     `styles.css` must **not** contain
     `li.task-list-item[data-bob-block="p2"] .task-priority` and must not clip
     `.task-priority` for Tasks-results digit rows. Rename the test to match. In the
     awaiting-vectors loop, drop the `blockMarkIsDigitKey(…) … "bars stay shown"`
     assertion.
   - `test-ledger-tools-block-marks-editor.cjs`: update the `runBlockMarksRefresh`
     call-order expectation (it currently expects
     `["editors", "priority", "reading", "folds", "chips"]`) to drop `"priority"`.
   - `obsidian-block-marks-e2e.cjs`: flip `lp-BM1-bars-folded` and
     `reading-BM1-bars-folded` to `lp-BM1-bars-shown` and `reading-BM1-bars-shown`
     (`folded === false`, `markWidth > 4`). In the Tasks-results BM24 check, pass
     `hidePriority: false`. Add one Live Preview case: put the cursor at the start of
     the BM1 line, so the raw `- [?]` is revealed, and assert BM1's priority mark still
     has width > 4. That case is the screenshot regression.
10. Docs: `docs/bob-ledger-tools.md`, in the blocked-reason mark and priority mark
    bullets. Remove "While on, digit checkboxes (`p1`–`p4`) fold that row's priority
    bars to zero width" (keep the `prj` / `awaiting` pill-fold wording), and remove
    "Digit block-mark rows fold those bars while blocked-reason marks are on". State
    that the bars always show. Then run `npm run docs:inventory` if the inventory check
    asks for it.
11. Gates: `npm test`, `npm run validate`, and `npm run docs:check` must pass. Run the
    real-Obsidian e2e
    (`xvfb-run -a node scripts/obsidian-block-marks-e2e.cjs -r /tmp/block-marks-e2e.json`)
    if the harness can launch here. If it cannot, say so plainly in the final report
    rather than claiming it passed.
12. Deploy: run `bob plugins sync` after the bob-plugins changes. If it refuses to
    overwrite another checkout's deploy, follow that repo's `AGENTS.md` procedure. Never
    pass `-F` on your own.

### bob-cli (this repo): the authoritative display contracts

13. `docs/blocked-marks.md`
    - _Surfaces and interaction_ table: change the
      `Priority bars / #prj_is_blocked / #awaiting_reply pills` row to cover only the
      two pills ("folded where the box already says it"). Add a sentence after the
      table: priority bars are never folded; a digit row shows both its digit and its
      bars.
    - _Tasks query results and their known limits_: rewrite the paragraph that starts
      "Digit rows clip `.task-priority`" so only the `prj` and plane tag clips remain
      and every row keeps its bars.
    - _Conformance vectors_: BM1's expected cell ends in `bars shown`, not
      `bars folded`.
    - _Live verification_: update the folds bullet to say that BM1, BM8, and BM9 bars
      all stay shown (including with the cursor revealing BM1's raw `[?]`), while the
      BM7 pill folds and BM19 keeps it. Do not change the historical pass count unless
      you actually rerun the e2e; if you do, update it to the new real total.
    - _Rejected alternatives_: add one sentence on why folding the bars on digit rows
      was dropped (2026-11-27). The digit vanishes whenever the raw `[?]` is revealed,
      for example with the cursor on the checkbox. In Tasks results the loop glyph can
      win the box. Either way the row had no priority on screen.
14. `docs/projects.md`, _Priority marks_ section
    - Replace the paragraph "While blocked-reason marks are on, a row whose block mark
      is a digit … folds its priority mark to zero width instead …" with one sentence:
      the mark always renders, including on Blocked rows whose box shows a P1–P4 digit.
    - Vectors PM18/PM19: PM18 (block marks on) expects `3 bars ·` plus the same tooltip
      as PM19, so the two rows read identically. Merge them into one vector, or keep
      both with matching expectations.
15. Run `just docs-check`. The `docs/reference/plugins/` tree is a generated mount of
    bob-plugins `docs/`, so do not hand-edit it.

Do not edit `Cargo.toml` versions or `CHANGELOG.md` (release-train rule). No CLI or JSON
contract changes, so no `!` marker.

## Verification checklist

- In Live Preview, a `- [?] #task A [priority:: medium] [scheduled:: <future>]` row
  shows the hollow `2` box **and** 3 bars. With the cursor at the start of that line
  (raw `[?]` revealed), the 3 bars still show.
- Reading view: the same row shows the `2` box and 3 bars.
- Tasks query results: the same row shows the CSS `2` box and the CSS bars. A recurring
  future row with a ladder priority shows the loop box and its bars.
- `#prj_is_blocked` and `#awaiting_reply` pills still fold on `prj` and `awaiting` rows.
- Toggling blocked-reason marks off and on never changes whether the bars show.
- `npm test`, `npm run validate`, `npm run docs:check` (bob-plugins), and
  `just docs-check` (bob-cli) pass, and `bob plugins sync` deployed the new ledger-tools
  version.
