---
tier: tale
title: Finish ledger frees marks and land bob-cli-g1
goal:
  Repair identity, midnight refresh, and rendering verification gaps, deploy the
  integrated plugin, and close bob-cli-g1.
size: medium
proposed_by: bbugyi200.athena.bob-cli-g1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-g1
create_time: 2026-11-02 15:44:15
status: wip
---

- **PARENT:**
  [202611/ledger_frees_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/ledger_frees_marks.md)
- **BEAD:**
  [bob-cli-g1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-g1/README.md)

# Finish ledger frees marks and land bob-cli-g1

Repair the remaining defects in the existing feature, verify the integrated plugin,
deploy it, and close bob-cli-g1 in this same coding turn. This tale has no land agent;
its coder owns the closeout below. Do not wait for this work's own commit, SHA, push, or
CI run: the host commits after the turn ends.

## Context already reviewed

Read `sase bead read bob-cli-g1 -r "Need landing scope and decisions"` and its notes,
and audit-read `plan:202611/ledger_frees_marks.md`. All three phases are closed. The
lander reviewed every phase note, the CLI contract commits b508f0b6 and 07edcdfa, and
the plugin implementation commits f824dd1 and a28d1af. Keep the final approved
`pill_text = compact` decision: the visible mark is `🔑 N`; the tooltip says frees. The
dependency-only count must continue to match BLOCKS. Relation successors, keep-moving,
project-finished pull-ins, Mac echoes, and joint unlocks remain outside this feature. No
memory edits or new CLI/API surface are needed.

Open bob-plugins and plans through `/sase_repo`; use only the returned checkouts. Read
bob-plugins/AGENTS.md. Edit plugin source fragments and generate main.js with
`npm run build`, never edit main.js directly. Keep fragments at most 1000 lines. Before
changing contract documentation, read docs.md through `/sase_memory_read`.

The reviewed baseline is bob-cli 07edcdfa and bob-plugins a28d1af. The lander examined
intervening CLI capture/relation, merge, and GCal changes: none expands the frees count
rule. Plugin commits 289f764, 93d3859, and d42b951 added Tasks block marks and
dependency-chip miniatures in shared files. They are present in a28d1af; preserve them,
including the BLOCKS self-exclusion added by f824dd1 and the wrapped Reading list fixes
added by a28d1af. Inspect any newer commits before finishing.

The lander's focused frees/progress/unblocked/e2e-self-test run passed 56/56.
`npm run validate` passed (8/8 plugins), and `npm run docs:check` passed (447 tests, one
skipped). These passes do not cover the defects below. No `just check` pass is claimed
for the landing. Never run `just check-full`.

## 1. Fix task identity in motion state and widget reuse

In `plugins/bob-ledger-tools/src/271b-plugin-frees-marks.js`,
`buildFreesMarkDecorations` keys counts by the daily file path plus blockId. Two
different target notes with the same blockId, especially `^prj`, collide. A probe using
the helpers in `scripts/test-ledger-tools-frees-marks.cjs` reproduced:

- owners `one.md#^prj` (id x, one dependent) and `two.md#^prj` (id y, two dependents);
- first build: counts 1/2, ticks false/true;
- unchanged second build: ticks true/true.

Carry the resolved owner's canonical path and blockId from the pure candidate model in
`141b-frees-marks.js` and use that identity for the bounded per-editor count map.
Aliases to the same owner share identity; distinct notes do not. Compute transitions
once per owner per build so repeated visible links do not consume or fabricate a
transition. Preserve zero-count history, the 2000-key bound, first-paint silence, and no
animation on selection-only updates, unchanged rebuilds, or scrolling.

`FreesMarkWidget.eq` currently compares only blockId, count, and tooltip. Widgets with
linktext `one#^prj` and `two#^prj`, equal counts, and equal freed-task copy compare
equal even though their mousedown closures open different notes. Include navigation
identity (linktext and sourcePath, and any other behavior-bearing identity required) in
equality. Test replacing a link with a different target that has the same blockId and
display data, and verify Mod-click opens the new target. Keep plain-click caret
placement and read-only behavior.

Add regression tests for two notes sharing `^prj`, duplicate and aliased links,
unchanged rebuilds, a real 0-to-N/count change, and a target replacement. Pin the clock
in tests wherever production code reads Date.

## 2. Clear yesterday's editor marks at midnight

`refreshFreesMarksForRollover` advances freesMarkDay and calls
`refreshFreesMarksEditors`, but that method skips every leaf not matching the NEW daily
path. A probe with yesterday's editor still open returned rollover=true and dispatched
zero refreshes to that editor. Its decorations remain until another edit or viewport
change, violating today's-note-only gating.

Refresh the formerly eligible editor as well as today's editor during rollover, or
dispatch the cheap invalidation to all Markdown editors and let the builder's existing
path gate reject them. Ensure count-history handling does not make a new day's first
paint animate. Test two open daily leaves across a pinned midnight: yesterday becomes
bare immediately without typing or scrolling; today renders the correct marks. Verify
Reading hosts also disappear from yesterday's note and that toggle/source-mode
transitions still work.

## 3. Finish the promised bounded rendering and verification

`freesMarkCandidateLinks` currently resolves and counts every ledger row on every build,
before viewport filtering. A 200-link/2000-task probe with 200 links to one owner
invokes the reverse-index owner lookup 200 times (10.56ms on this host). The approved
plan requires one resolution per unique target per build, visible-range rendering, and a
200-link/2000-task performance test; that test is absent.

Add build-local resolution/count caching keyed by resolved target and preserve
memo-generation freshness. Avoid expensive target/count work for offscreen rows in Live
Preview while retaining correct ledger section/context parsing, count-change semantics,
and silent scroll-in. Reuse the same optimization for repeated Reading targets where
appropriate. Keep counts driven by freshnessEnsureMemo and the shared BLOCKS helpers.
Add a meaningful synthetic performance regression near the existing 25ms budget, and an
instrumented assertion that repeated targets resolve/count once per build. Do not mask
regressions by loosening the promised bound. Test first paint and warm rebuilds, memo
replacement, zero counts, and visible-range behavior.

Tighten `scripts/obsidian-frees-marks-e2e.cjs`: `tick-count-updates` currently waits
four seconds but prints `within=1s`, and `tick-class-once` accepts two events. Measure
elapsed update time and enforce the contract's approximately one-second target with an
explicit documented tolerance. Observe actual animation events without double-counting
DOM mutations, and require one animation for the single changed row. Assert plain
click's exact link-end caret column, not just the presence of `fc1` somewhere on the
line. Add offline checks for rejection of late updates and duplicate animations so the
harness cannot silently weaken its own assertions.

## 4. Verify the integrated build and deploy it

Run the focused regression tests, BLOCKS/chip tests, and the related progress and
unblocked-glyph tests. Run `npm run build`, `npm test`, `npm run validate`, and the
documentation gate. Set `BOB_CLI_DIR` to the current bob-cli checkout for
`npm run docs:check`; otherwise that script falls back to a separate cached clone. Run
`just check` in bob-cli and `just docs-check` for the contract update. Use
`/sase_monitor` before starting long gates, with continuation instructions retaining
this tale's final closeout; do not finish a turn with a running command.

Run the isolated real-Obsidian harness under xvfb-run with /snap/bin/obsidian, using
`node scripts/obsidian-frees-marks-e2e.cjs -e /snap/bin/obsidian -o <output> -r <report>`.
Keep it on its scratch vault. Review light/dark Live Preview and Reading screenshots,
exact tooltip and link/unblocked/frees order, caret navigation, toggle and row-height
behavior, and measured animation/update results. Update `docs/frees-marks.md` with what
actually passed and any remaining deferred checks.

Finish the deployment integration from g1.2 note #3. The source is integrated, but the
lander's explicit `bob plugins list --no-pull --repo <opened-bob-plugins> --format json`
still reports ledger-tools drift. The vault stamp is an uncommitted f824dd1 build from
g1.3; a dry run shows vault manifest 1.92.0 versus source 1.93.0 and main.js/styles.css
differences. Therefore g1.3's forced sync did NOT deploy the final combined tree.
Preserve d42b951's block-reason miniatures while repairing frees marks, then run
`bob plugins sync --no-pull --repo <opened-bob-plugins> --plugin bob-ledger-tools` and
confirm synced with an explicit-repo list. Put options after the list/sync subcommand.
Inspect a refusal and reconcile its actual provenance; never force a stale build over
newer or unrelated vault work. Do not require the coder's own commit to exist before
deployment or epic closure.

## Follow-up triage already completed by the lander

No new task is needed for the phase proposals. The lander recorded outcomes on
bob-cli-g1 before proposing this tale:

- g1.2 #1, generated glossary: declined as resolved; current docs gate passes.
- g1.2 #2 and g1.3 #1: corroborated existing CI tasks bob-cli-f8 (two wikilink CPU
  budgets), bob-cli-et (freshness reload clock), bob-cli-f7 (Keep UTC/local time),
  bob-cli-eh (two startup callback fakes), and bob-cli-ff (fixed future date). Landing
  independently reproduced exactly these seven failures, 110/117 passing, on a28d1af.
  Evidence: `file:explicit:1c86942de5e48357b13f472a`.
- g1.2 #3, stale base and sync: retained as epic integration work in step 4, not a
  separate task. The stale source base is fixed, but deployed files still drift.
- g1.2 #4's isolated-passing Chromium timeout is not an unresolved epic defect; preserve
  the evidence distinction from the deterministic baseline failures.

The active-epic sweep found no causal owner for the five existing CI tasks. Do not fold
those unrelated test-infrastructure fixes into this tale. Record any changed
verification outcome honestly; every new failure caused by this work must be fixed.

## 5. Close bob-cli-g1 and finish its plan in this turn

Re-read bob-cli-g1 and all three child scopes/notes; verify each remaining issue in the
lander's notes is resolved, all descendants are closed, the compact decision still
holds, and intervening commits are integrated. The audited parent-link read found NO
parent_bead; recheck it with `sase bead read bob-cli-g1 -r "Need the parent link"`. No
ancestor close is required unless that link has changed, in which case apply the
original landing prompt's phase-versus-plan ancestor rules.

Run `sase bead epic-symbols bob-cli-g1` (none at landing review). Resolve every entry
keyed to this epic or its phases by wiring it, privatizing it, adding a valid non-test
pragma, or deleting it per Symvision policy; re-key only if a still-open later bead
truly needs the exemption. Do not leave stale entries for another agent.

Close normally with `sase bead close bob-cli-g1 --note "<verification>"`. The note must
name the identity/midnight/performance/harness repairs, reviewed commits and
integration, passing checks and measured real-Obsidian results, deployment result, and
every follow-up disposition above. If symbols or unfinished phases reject the close,
finish the required work and retry. Never use --force to make a successful landing
advance. Cancellation/supersession requires an actual deliberate outcome, --reason, and
the appropriate resolution, not a workaround.

After closure, run `just symvision` wherever that recipe is available; currently bob-cli
has no symvision recipe and bob-plugins has no Justfile, so explicitly record
unavailability if that remains true. In the opened plans repository, set `status: done`
in `202611/ledger_frees_marks.md` (canonical artifact
`plan:202611/ledger_frees_marks.md`). Preserve its other frontmatter. Finish through
`/sase_final`, declaring all changed repositories, including the plans sidecar.
