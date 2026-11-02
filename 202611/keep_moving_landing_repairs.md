---
tier: tale
title: Repair and land Rust keep-moving offers
goal:
  Prevent incomplete-snapshot writes, remove duplicate routing work, correct keep-moving
  copy, and close bob-cli-fb.2.1 and phase bob-cli-fb.2.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fb.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fb.2.1
create_time: 2026-11-01 19:04:04
status: wip
---

- **PARENT:**
  [202611/leaf_keep_moving_rust.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/leaf_keep_moving_rust.md)
- **BEAD:**
  [bob-cli-fb.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/bob-cli-fb.2.1.md)

# Repair and land Rust keep-moving offers

## Objective and scope

Finish the remaining work found while landing **bob-cli-fb.2.1**, then close that epic
and its parent **phase bob-cli-fb.2** in this same implementation turn. This is one
medium, directly implementable tale: three bounded repairs in existing Rust and
documentation seams, focused regression coverage, verification, and closeout. There is
no later land agent for this tale. Do not recreate the completed engine, capture wiring,
or net-pass phases.

Read `sase bead read bob-cli-fb.2.1 -r "Need landing audit and triage"` and its three
children `.1`, `.2`, and `.3`, including every note. Read
`plan:202611/leaf_keep_moving_rust.md` through `sase artifact read`; that is the epic
plan whose status must eventually become done. The original parent phase is
`bob-cli-fb.2`, belonging to the separately managed epic `bob-cli-fb` and
`plan:202611/leaf_project_keep_moving_offers.md`.

The audited source is master **5ac2fbc2**, equal to fetched origin/master. The epic
implementation commits are **f6d3084b**, **ab32d58a**, and **8347a46d**. All three child
phases are closed, but source inspection found the blockers below. The audit did not
rerun compilation or performance tests; previous phase results are historical evidence,
not a fresh pass.

Binding decisions remain: offer Pending, Next, Ready in that order; keep-moving is
explicit selection only; unpicked rows are unchanged; JSON stays additive at schema 1.
No memory edits, new CLI flags, version changes, or CHANGELOG changes.
`docs/task-dependencies.md` §§12.15–12.17 and the child plan govern behavior.

## 1. Prevent legacy writes when dependency data is unavailable

This is the primary correctness repair. In both `src/native/capture/task_complete.rs`
and `capture/pomodoro_close.rs`, unavailable snapshots now become an empty `snapshot`
plus `snapshot_complete: false`. The caller invokes `engine::plan_successors`, clears
`unblocked`, `still_blocked`, and `project_folds`, then stages the engine's
`changed_files` and `new_day_text`. Clearing reporting does not prevent legacy recovery,
minting, freshness changes, or links. In `task_complete/successors.rs`, snapshot
completeness is currently passed only to the keep-moving classifier; the legacy
classifiers still run on the held staged notes. Missing external IDs become unresolved
rather than open in `task_dependency_states`.

Implement the safety boundary before legacy verdicts and writes are constructed. An
unavailable snapshot suppresses all legacy dependency recovery, project-member recovery,
and project-finished pull-ins, including their effects on the day file. Keep KM31's
independently safe keep-moving rows: dependency-bearing residents fold blocked, a root
whose prerequisite safety is unknown suppresses its group, and safe residents may still
be explicitly picked. Do not simply discard all engine changes, because that would
discard legitimate keep-moving picks too. Keep cancellation and complete-snapshot
behavior intact. Remove or clarify the misleading caller-side reporting cleanup after
the engine enforces this rule.

Use a regression fixture extending `tests/cli/capture/keep_moving.rs`:

- N is the existing leaf `sase_badges`, with open `^prj`, linked predecessor T carrying
  `[id:: t]`, one dependency-free candidate A, and resident D:
  `- [?] #task Needs both [dependsOn:: t,q] [id:: d] ^d`.
- `q.md` holds an open task with `[id:: q]`. An unrelated marker-positive invalid UTF-8
  note poisons the dependents snapshot, as in the existing KM31 fixture.
- Close T with `!` and with a completing Pomodoro close, both without selection and with
  `-p all`. The old implementation can classify D as a legacy hand-off because q is
  absent from the empty snapshot, then write D while hiding its row.

Assert exact task bytes for D (no status, tags, IDs, freshness or unblocked writes), no
D link in the day postimage, no legacy report/count/diff reason, and
`unblocked_check: "unavailable"`. A must still offer and apply when picked, and D must
contribute to the blocked fold if a keep-moving group survives. Verify dry-run and
real-run agreement. Cover a held dependent outside a keep-moving-triggering project and
an unavailable project pull-in as well, so the safety boundary cannot depend on KM1
firing. Add a pure engine assertion that no legacy change escapes with
`snapshot_complete: false`, plus complete-snapshot controls. Preserve the existing
controlled poison fixture; do not depend on Unix permissions.

## 2. Remove the avoidable catalog cost from keep-moving routing

Phase bob-cli-fb.2.1.3 note #1 reported triggering p50 272/314 ms versus control 174/254
ms and pre-feature base **83e11967** p50 203/220 ms on a 6,826-note copy, 50 repetitions
on loaded apollo. It attributed approximately 45–55 ms of added cost to a second
root-note scan. The source confirms that keep-moving newly opens the successor path,
whose `SuccessorTypeRouting::load` unconditionally calls `Catalog::load`. That calls
`pomodoro_themes::scan::scan_root_areas_matching` without a key restriction. Capture
already has a lazy batch-scoped `CaptureTaskThemeContext`, and successor planning
currently constructs its own routing context per call, even when no verdict ultimately
has an offer.

Make routing inputs lazy and reusable for the capture batch. Skip catalog/home setup
when there is no routable offer. Reuse the loaded catalog/config and resolver where
practical, or provide an equivalently narrow shared routing context that avoids repeated
root-note scans across direct, Pomodoro, and nested completions. Keep the engine
testable with in-memory data. Respect staged task and home facts; do not introduce a
daemon, persistent cache, or a second home-resolution algorithm. Retain D7 destination
behavior, known type spelling, configured sources, exclusions, and existing hue metadata
wherever it is actually emitted. Do not optimize by omitting required routing or by
changing selection semantics.

Add a meaningful read/load-count regression for nontriggering closes and multiple
triggering items. It must catch the duplicate catalog/root scan, not just test the
counter implementation. Exercise typed keep-moving offers sharing a destination with
legacy offers and ensure stable keys, lane order, and Pending preservation.

Measure a release build on apollo against a temporary full-vault copy, following vault
reference memory before accessing it. Never mutate the live vault. Keep scratch files in
the current workspace, isolate config and clock, and record the exact commands, tested
revision plus dirty diff identity, fixture size, repetitions, load conditions,
distributions, and JSON proof of a real keep-moving group. Compare matched triggering
and nontriggering runs, and the same-path pre-feature baseline if needed. The documented
target remains 70 ms; do not call a miss a pass. Fix the epic-added duplicate work here.
A remaining, demonstrated pre-feature latency floor belongs to **bob-cli-5w**, which
already owns that unmet acceptance target; report it there and in the landing evidence
rather than reopening unrelated fixed tasks. Do not close while an unexplained or
avoidable epic-added regression remains.

## 3. Use lane names and align the CLI examples

`capture/output.rs::print_human_keep_moving_row` currently calls
`style_task_status_marker`, which emits raw checkbox symbols. The human tests even pin
`linked [ ] → [*]`. This contradicts the approved child plan and §12.15's requirement
that keep-moving rows use Pending/Next/Ready lane names.

Render named lanes on unpicked and picked keep-moving rows, including the before/after
lane for a changed status. Preserve the approved why header, folds, `keeps N moving`
suffix, number/key behavior, and existing destination labels such as `(next up)`. Leave
ordinary completion and legacy dependency-row formatting unchanged. Update tests for
direct, Pomodoro, nested-completion, and choice-refusal human output, checking that
keep-moving rows contain lane names and no raw marks or unblocked/unlock labeling.

Read the docs memory before editing. Correct the CLI example and picked-row line in
`docs/task-dependencies.md` §12.15 to match those named lanes and the retained
destination capsule. This addresses phase `.3` note #2 without deleting useful
destination information or weakening the lane-name rule. Do not change the shared
selection/trigger contract.

## 4. Recheck integration and verify the repaired tree

The landing audit reviewed these intervening commits:

- **b71a6c15** Today roots: `root_label` became crate-visible; the leaf resolver's
  algorithm stayed unchanged. Keep using that resolver.
- **eda4dfc3**, **273e09a7**: calendar-aware MEET starts and event warnings. Closes
  intentionally stay quiet for the start-conflict warning. Preserve trackedness and the
  closing-anchor baseline; add coverage for carry/drop suppression (KM10/KM12) through
  the current close path.
- **50df0e41**: stamped Work Logs. Confirm a completing close with a Work Log still
  offers the correct residents without interpreting log bullets as tasks.
- **bbd26d97**, **5ac2fbc2**: drawer grammar and multi-note existing-task captures.
  Their outer item tracker still records the day after each item. Verify a later
  multi-note item that inserts lines before an ID-less keep-moving candidate does not
  lose its identity or invent a link; preserve later-link/reblock suppression.
- **675330dd**: merged-Pomodoro pure planners. Active **bob-cli-fe.2** owns the capture
  landing migration. Preserve its landed helpers and recheck any further changes when
  sharing the routing context. Do not implement that separate epic.
- **f490d88f** already fixed the proposed agenda lint failure, and **39804896** fixed
  keep-moving numeric-test lint and the close JSON golden.

Read any newer commits on master/base since **5ac2fbc2** and integrate changes touching
these paths. Run focused keep-moving, successor, project-pull-in, selection/net, and
affected capture integration tests, then **`just check`** and **`just docs-check`**. Do
not run `just check-full`. Use `/sase_monitor` for long commands as required, carrying
the remaining closeout instructions across a monitor continuation. Do not confuse a
running command with a completed gate. Compare unrelated failures with the clean base
and route them through `/sase_new_task`; never hide a new failure as baseline. The
already reported typeless landing failures from 675330dd belong to active bob-cli-fe,
but verify their current state rather than assuming the historical failure persists.

## 5. Finish this epic's landing in the implementation turn

All proposal triage is already recorded on bob-cli-fb.2.1. Include every outcome in the
eventual close note:

- `.2` note #1: declined as already fixed by f490d88f, no new task.
- `.3` note #1: added catalog cost repaired here; the separate pre-feature latency miss
  was attached to active bob-cli-5w with proposer attribution after duplicate searches
  and its earlier landing/repair evidence were read. No duplicate task.
- `.3` note #2: CLI example mismatch repaired here, no deferred task.

Re-read this epic, all descendants and notes, and its linked plan. Confirm the three
repairs, intervening integration, selected decisions, test results, exact JSON example,
and honest timing evidence. Resolve any remaining epic-caused issue before closing.
Complete the tale's own bead first if it is a descendant that would otherwise block the
normal close; no SHA, push, or CI for this turn's own commit is a prerequisite. The host
commits after the turn ends.

Run **`sase bead epic-symbols bob-cli-fb.2.1`** immediately before closing. Resolve each
entry by wiring, privatizing, deleting, or an appropriate non-test pragma; re-key only
to a still-open later consumer that actually needs it. Then run:

```bash
sase bead close bob-cli-fb.2.1 --note "<source, decisions, integration, tests, performance, and every proposal outcome verified above>"
```

Run **`just symvision`** if available (the audited checkout has no such recipe; check
again and record explicit unavailability if still absent). Through `/sase_repo`, open
the plans repo, use the printed path, and set **`status: done`** in
**`202611/leaf_keep_moving_rust.md`**, the plan identified by
`plan:202611/leaf_keep_moving_rust.md`. Do not hand-edit bead status.

After this epic closes, run `sase bead read bob-cli-fb.2.1 -r "Need the parent link"`
and verify the parent is still **phase bob-cli-fb.2**. Re-read that phase; confirm this
child plan completes its Rust engine/capture-reporting scope. Run
**`sase bead epic-symbols bob-cli-fb.2`**, retire or legitimately re-key leftovers, and
close **only that parent phase**, normally:

```bash
sase bead close bob-cli-fb.2 --note "<verified Rust scope, child completion, JSON/selection/net behavior, integration, checks, and timing limitations>"
```

Leave containing epic **bob-cli-fb** and its plan to its waiting land agent. If parent
readiness changed, record the precise blocker on bob-cli-fb.2 and report it. If either
close rejects stale symbols or unfinished descendants, fix the cause and retry normally.
Never force to advance a successful nested landing. Use `--force` only for a deliberate
canceled/superseded resolution with its real reason, never to bypass completion checks.
Finish with `/sase_final`, declaring all repositories changed, including the plans repo,
for host commit.
