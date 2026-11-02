---
tier: tale
size: medium
title: Finish landing bob-cli-fl.6.1 — Stats clock fixes, docs, deploy, and closeout
goal:
  Fix the two epic-caused Stats clock defects, make the Stats preview harness complete,
  finish the clock user docs in both repositories, deploy bob-ledger-tools, and close
  bob-cli-fl.6.1 and its parent phase bob-cli-fl.6.
decisions:
  force_sync_ledger_tools:
    ask:
      Force-sync bob-ledger-tools after re-proving its refused dirty deploy is on
      master?
    choices:
      force:
        Obsidian runs the finished Stats clocks; sync backs up the overwritten files
      leave: The vault keeps the phase-2 build and the refusal is recorded for Bryan
    default: force
    why: The deployed build is 3a2b03b minus 7a647e8, both on master, so nothing is lost
    answer: force
proposed_by: bbugyi200.apollo.bob-cli-fl.6.1.land
proposed_with: claude/opus
bead: bob-cli-fl.6.1
decided_by: auto
status: done
---

- **PARENT:**
  [202611/stats_clock_ui.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/stats_clock_ui.md)
- **BEAD:**
  [bob-cli-fl.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fl/bob-cli-fl.6.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-fl.6.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-fl.6.1.land.md)
- **COMMITS:**
  - [1bad28f](https://github.com/bobs-org/bob-cli--plans/commit/1bad28f1e5d632cf4a83f33750f578c471907009)
    — docs(plans): mark stats_clock_ui epic plan done

# Finish landing bob-cli-fl.6.1

The bob-cli-fl.6.1 land agent checked every phase of `plan:202611/stats_clock_ui.md`
against the plan and the source code. The phases match their notes except for the gaps
below, all caused by this epic. This tale finishes them and then performs the epic's
closeout, so the coder must complete every step in one turn.

Work in bob-cli (this checkout) and in the linked bob-plugins checkout opened with
`/sase_repo` (`sase repo open bob-plugins -r "..."`). Use only the path that command
prints, and read its `AGENTS.md`. Read `docs.md` with `/sase_memory_read` before you
edit any docs. In bob-plugins, edit `plugins/bob-ledger-tools/src/` and `styles.css`,
then run `npm run build`. Never hand-edit `main.js`. Each hand-edited fragment must stay
at or below 1000 lines. `238-plugin-stats-block.js` is already at 967 lines, so put any
new logic in `236-plugin-stats-clock.js` (412 lines) or another small fragment.

Make no memory edits, CLI changes, Cargo version or CHANGELOG edits, or ledger-writer
changes. The epic's contracts still apply: Bob owns every value and display string, a
missing breakdown is never read as zero, and the Day strip's default request key and
Worked lanes stay unchanged.

## 1. Stop the endless refetch when an Off-clock or Total fetch fails (phase 2 bug)

`paintStatsBlock` in `238-plugin-stats-block.js` handles a settled result that has no
report while `args.clock !== "worked"` and the view supports clocks. It sets
`view.statsClockSupported = false` and repaints on Worked. The Worked report, cached or
fetched, is clock-capable, so `statsClockObserveReport` sets supported back to `true`.
`state.clock` still holds the failed lens, so it returns `"retry"`. The repaint then
requests the failed lens again, fails again, and repeats forever. The block never leaves
its skeleton, no error card renders, and every pass spawns a new `bob stats`, because
errors are never cached. Any persistent failure triggers this: a `-k total` timeout, a
crash, or an older bob after a downgrade.

Reproduction (the land agent ran it): build a plugin from `ledger-tools-harness.cjs`.
Stub `statsPeek` to return a clean capable Worked report only for `clock: "worked"`, and
stub `apiStatsFetch` to always resolve `{report: null, error: {...}}`. Set
`state.clock = "off-clock"` with `statsClockSupported = true` and
`statsClockPreferenceApplied = true`, then call `paintStatsBlock(view)`. The result is
500+ fetches and 0 `renderStatsResult` calls, as a microtask loop.

Fix: a failed nondefault-lens fetch must settle once. Fall back to Worked for that view
by setting the view's in-memory `state.clock` to `"worked"`. Do not persist it, and do
not erase the saved device preference. Then render, and never automatically re-request
the failed lens. When the user picks Off-clock or Total again (radio, `c`, or a Day
strip hint), the plugin makes exactly one new attempt. Keep the existing behavior for a
stale cached report that arrives with an error, which still renders the stale report.

Add a regression test to `scripts/test-ledger-tools-stats-clocks.cjs` that covers this
scenario. Bound the test so a reintroduced loop fails the assertion instead of hanging
the test runner, and assert one render, at most two fetches, `state.clock === "worked"`,
and an unchanged stored preference.

## 2. Fix the 360 px overflow from the clock row values (phase 4 bug)

At 360 px the Stats preview page is 367 px wide; phase 3 (33318e0) was exactly 360. The
only unclipped overflow is `.bob-stats-home-value`. It is `white-space: nowrap` in an
`auto` grid track, and phase 4's `statsPanelClockText` made its text long, for example
`1h 05m Total · 50m worked · 15m off-clock · 54%`. Under Worked with off-clock time
present it becomes `… Worked · 15m off-clock excluded`. Let the value wrap only at its
`·` separators: keep each segment unbreakable and allow breaks between segments, using
non-breaking spaces or per-segment nowrap spans. That shrinks its min-content without
breaking `1h 05m` apart. Check that the window-lanes meta text and the panel titles fit
at 360 px too. Add a preview or DOM assertion that
`document.documentElement.scrollWidth` does not exceed the viewport at 360 px with the
clock fixtures.

## 3. Make `scripts/preview-stats-block.cjs` complete again (phase 4 regression)

On a healthy host the harness at 33318e0 finishes in about 14 s with 30 interaction
checks. From 75e7875 through HEAD 806ba69 it hangs forever in `Page.captureScreenshot`
(`fromSurface: true, captureBeyondViewport: true`) on the first 1000 px matrix cell
(`stats-1000-light.png`). The land agent established:

- The hang is deterministic and caused by phase 4's JS. HEAD `main.js` with phase-3
  `styles.css` hangs, and phase-3 `main.js` with HEAD `styles.css` passes.
- It hangs with or without the hover and with or without the earlier 360 px cells. A
  viewport-only `{format: "png"}` capture of the same page succeeds in about 70 ms.
- `Debugger.pause` shows an idle page. No rAF or mutation churn follows the render, no
  inline style holds NaN or Infinity, and document height does not depend on viewport
  height (33864 px at 900 and at 20000 px tall).
- Deleting every `.bob-stats-lanes` element, or every `.bob-stats-panel`, before the
  capture lets it succeed. Deleting only the clock tail and split spans does not.
- Separately, apollo sometimes wedges every Chromium capture, even of a trivial page.
  The land agent filed that as task bob-cli-fz. Calibrate first: if the 33318e0 page
  also hangs, the host is in that state, so wait and retry before you bisect.

Find which phase-4 render change wedges the full-page capture, using a scratch copy of
the generated `index.html` or by bisecting the 75e7875 hunks. Fix the product cause if
there is one. If it proves to be a Chromium compositor quirk with no product defect,
switch this harness to a capture strategy that works and say why in a comment. One
option is to set the device-metrics height to the content height and then take a normal
capture. Either way, add a per-call timeout to this harness's CDP `call()` and kill
Chromium's process group on exit (spawn it detached, then kill `-pid`), so a stalled
capture fails with a clear error instead of hanging. The other harnesses belong to
bob-cli-fz; do not widen this change to them. The finished harness must exit 0 with
every interaction check passing. Inspect the clock screenshots it writes: off-clock,
total, forced-colors, and the 360/1000 light/dark cells.

## 4. Finish the clock user docs (phase 5 gap)

Phase 5 updated only the Stats-block header paragraph and the Day strip paragraph of
bob-cli `docs/stats.md`. In bob-plugins it changed only the docs inventory row. Read the
current code, then document what it actually does.

- bob-plugins `docs/bob-ledger-tools.md`: in the Day strip paragraph, add the quiet
  `◌ +<display> off-clock` hint under WORKED and say that it opens Stats on that day
  with the Off-clock lens. In the `bob-stats` block paragraph, add the Worked ·
  Off-clock · Total clock control: per-device memory, the `c` shortcut, and Worked when
  an older bob reports no clocks. Explain what the marks mean: dashed outlines in the
  item's hue mean off-clock time; under Total, a solid worked span ends in a dashed
  off-clock tail; off-clock sessions get a dashed edge. Total sums session minutes, so
  overlapping sessions count twice; it is not occupied or free time. Link to the bob-cli
  Stats page rather than repeating the contract.
- bob-cli `docs/stats.md`, Obsidian Stats block: add the dashed-meaning paragraph for
  tiles, sparklines, focus bars, root, type, and mini bars, panel bars, and session
  rows. Include the spoken Total sentence and the `No off-clock time in this window`
  empty state, which is distinct from pre-adoption `not_tracked`.
- bob-cli `docs/stats.md`, Window bars: replace the hard-coded Worked contract
  (`<summary.total.display> worked`, title `Worked <range.label>`, `Time by root parent`
  / `Time by type` / `Time by area`, and the group aria label) with the clock-aware text
  the code produces. Read `statsWindowLanesModel` and `statsPanelClockText` /
  `statsPanelClockName` for the exact strings. Also cover the Total split segments and
  legend tails, Off-clock dashed segments, the clock in the morph `windowKey`, and the
  forced-colors treatment. Keep the statement that the Day strip lanes are unchanged.

## 5. Ship it

1. Bump bob-ledger-tools' `manifest.json` version by one minor, following repository
   convention. Run `npm run build`, then `npm run docs:gen` if the inventory changes.
2. bob-plugins checks: run the focused suites first (the stats-_ and day-strip_ test
   files, `test-ledger-tools-pomodoro-clock.cjs`,
   `test-ledger-tools-pomodoro-home-chips.cjs`), then `npm test`, `npm run build:check`,
   `npm run validate`, and `BOB_CLI_DIR=<this bob-cli checkout> npm run docs:check`.
   These `npm test` failures are known and tracked, so do not fix them here:
   schedule-work-log (bob-cli-ff), command-mode-sheet-sequence-dom Chromium discovery
   (bob-cli-f1), freshness-namespace (bob-cli-et), note-ready and relation-index startup
   (bob-cli-eh), freshness Live Preview perf (bob-cli-ej), and Chromium capture hangs
   (bob-cli-fz). Any other failure must be shown to reproduce on clean 806ba69 before
   you call it pre-existing. `node scripts/preview-day-strip.cjs` has four known 410 px
   `screenshot-day` failures (bob-cli-g0); everything else in it must stay green.
3. bob-cli checks: run `just check` and `just docs-check`. Do not run `just check-full`.
4. Deploy. `bob plugins sync -n` refuses bob-ledger-tools because the vault holds a
   dirty build from workspace bob-cli_14 on 3bab3c0 (deploy digest `6c146eaa…`,
   2026-11-02T11:43Z). The land agent proved that build's `styles.css` equals commit
   3a2b03b's and its `main.js` equals 3a2b03b's minus the 36 lines that 7a647e8 added.
   Both commits are on master, so nothing uncommitted would be lost.

> [!decision] force_sync_ledger_tools = force Re-check that
> `~/bob/.obsidian/plugins/bob-ledger-tools/.bob-deploy.json` still shows that
> bob-cli_14 / 3bab3c0 deploy with the same digest. If anything changed, do not force:
> record the refusal and go to the closeout. If it is unchanged, run
> `bob plugins sync --repo <linked bob-plugins checkout> -p bob-ledger-tools -F`, then
> confirm with `bob plugins sync --repo <same> -n -p bob-ledger-tools` that it is up to
> date.

> [!decision] force_sync_ledger_tools = leave Do not force. Record the refused sync and
> its provenance evidence in the close note.

In either branch, never force bob-navigation-hotkeys or task-status-cycler. Their
refusal is tracked by bob-cli-e4 and bob-cli-fy.

## 6. Close out bob-cli-fl.6.1, then its parent phase

1. Run `sase bead epic-symbols bob-cli-fl.6.1`. At planning time it reported no entries.
   Resolve any entry that has appeared (wire it up, privatize it, add a non-test pragma,
   or delete it), or re-key it only to a still-open bead that needs it.
2. Close the epic with `sase bead close bob-cli-fl.6.1 --note "<verification>"`. The
   note must cover the five phase commits (bob-cli 247440cc and 69c6458c; bob-plugins
   3a2b03b, 33318e0, 75e7875, and 806ba69) and this tale's fixes: the refetch loop and
   its test, the 360 px overflow, the harness root cause and fix, and the docs. Include
   the test, check, and preview results, which browser scenarios ran and any that could
   not, the deploy outcome, and the follow-up triage already recorded on the epic
   (bob-cli-ff, -f1, -et, -eh, -ej, -e4 corroborated; bob-cli-fy, -fz, -g0 filed). Never
   use `--force`.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202611/stats_clock_ui.md`. Its absolute path is the EPIC PLAN path printed by
   `sase bead read bob-cli-fl.6.1 -r "Need the plan path for closeout"`.
5. The parent `bob-cli-fl.6` is a phase bead of epic `bob-cli-fl`. Confirm that this
   child plan covered its scope: the segmented clock control with per-view totals,
   Off-clock and Total renderings in hue with dashed tails, and the quiet off-clock hint
   under the Day strip's WORKED fact. Run `sase bead epic-symbols bob-cli-fl.6` and
   resolve any entry it lists. Then close only that phase with
   `sase bead close bob-cli-fl.6 --note "<what you verified>"`. Leave `bob-cli-fl` and
   every other ancestor open for their own land agent.

## Acceptance

- A failing Off-clock or Total fetch falls back to Worked once and renders. It never
  loops, and the regression test proves it.
- At 360 px the Stats preview is no wider than the viewport. The unmodified
  `node scripts/preview-stats-block.cjs` exits 0 with every interaction check passing.
- Both repositories document the clock control, the dashed meaning, Total overlap,
  clock-aware window bars, and the Day strip hint. Docs gates pass.
- bob-ledger-tools is version-bumped, built, tested, and deployed, or the refusal is
  recorded under the `leave` decision.
- bob-cli-fl.6.1 and bob-cli-fl.6 are closed with verification notes, `just symvision`
  is clean, and `plan:202611/stats_clock_ui.md` has `status: done`.
