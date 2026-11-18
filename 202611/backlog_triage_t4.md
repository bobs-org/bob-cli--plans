---
tier: epic
title: 'Backlog triage T4: green macOS master CI and CPU-work Live Preview gates'
goal: Make bob-cli's macos-latest master test job green so the release train can ship
  again, then stop the two most-corroborated bob-plugins wall-clock flakes, and continue
  the five-task backlog chain.
phases:
- id: macos-case
  title: Make case-sensitive collision tests portable to macOS
  size: medium
  description: 'macos-case: Resolve bob-cli-l9 (album, record, and highlights-scan
    collision tests that assume a case-sensitive filesystem) without weakening any
    collision protection.'
  depends_on: []
- id: macos-symlink
  title: Fix the macOS /var symlink path failures in note catalog and note goto
  size: medium
  description: 'macos-symlink: Resolve bob-cli-n6 (note_catalog xlib tier and note_goto
    vault_root under the /var -> /private/var temp symlink), fixing product canonicalization
    where it is a real bug.'
  depends_on:
  - macos-case
- id: pipe-drain
  title: Make the detached-grandchild pipe-drain tests deterministic and portable
  size: medium
  description: 'pipe-drain: Resolve bob-cli-kl so both subprocess_runner pipe-drain
    tests pass deterministically on Linux under full-suite load and on macOS, which
    has no setsid(1).'
  depends_on:
  - macos-symlink
- id: cpu-gates
  title: Gate the 200-link Live Preview budgets on CPU work
  size: medium
  description: 'cpu-gates: Resolve bob-cli-ju (off-clock link chips, 35 ms) and bob-cli-ki
    (keep-moving marks surface, 250 ms) by measuring CPU work with the convention
    bob-cli-3w established, with unchanged budgets and workloads.'
  depends_on:
  - pipe-drain
proposed_by: bbugyi200.athena.bob-cli-n5.land
proposed_with: claude/opus
create_time: 2026-11-18 01:55:50
status: wip
bead_id: bob-cli-n7
---

- **PROMPT:** [prompts/202611/backlog_triage_t4.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/backlog_triage_t4.md)
- **PARENT:** [202611/backlog_triage_t3.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backlog_triage_t3.md)
- **BEAD:** [bob-cli-n7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-n7/README.md)

# Backlog triage T4

## Scope and selection

This is the next independent backlog epic after `bob-cli-n5` /
`plan:202611/backlog_triage_t3.md`. It is not a child required to finish T3. It takes
exactly five existing task beads: bob-cli-l9, bob-cli-n6, bob-cli-kl, bob-cli-ju, and
bob-cli-ki. Run the four phases strictly in order. Read each assigned bead with
`sase bead read ID -r "Need scope and notes"`, including every note, before you
implement. Check its status first so you do not duplicate work. Close a task as already
fixed only with current-source evidence and passing regression evidence.

Why these five. The T3 lander re-swept the 232 active task rows on 2026-11-18. No
well-scoped wrong-write bug remains that an agent can fix without a live-vault or design
decision; see "Not selected" below. The top deterministic red gate is bob-cli's own
master CI. Every completed master run since about 2026-11-14 fails `test (macos-latest)`
or is cancelled, while ubuntu, lint, and package pass. No release has shipped since
v0.5.0 on 2026-11-07, because the release train needs full CI on the exact SHA. Run
44227364061 (master 2f14eef5) fails exactly twelve macOS-only tests:

- **bob-cli-l9** (case-insensitive filesystem): albums::create
  `stale_plan_notices_a_collision_introduced_after_planning`, albums::registry
  `stem_noun_and_case_insensitive_folder_collisions_invalidate_all_claims`,
  albums_create `create_normalizes_mixed_case_stems_and_takes_path_qualified_parents`,
  capture::record
  `record_exists_fix_uses_a_staged_target_but_not_an_incomplete_or_unsafe_one`, plus
  three same-class nodes in `tests/cli/highlights/scan_interleave.rs`. Those three were
  recorded on l9 by its 2026-11-18 +1:
  `queued_existing_output_collision_rejects_without_overwrite`,
  `queued_queued_output_collision_rejects_without_publication`, and
  `post_recovery_replan_failure_keeps_direct_output`. Each writes `clash.pdf` and
  `clash.PDF`.
- **bob-cli-n6** (the `/var` → `/private/var` temp symlink): note_catalog
  `golden_listed_set_and_canonical_order` and
  `shared_path_tiers_agree_with_the_living_walk` (`xlib/q.md` is Living, not Hidden),
  and capture::note_goto `capture_complete_note_goto_reports_shape_without_query`
  (`vault_root` is canonicalized).
- **bob-cli-kl**: both `subprocess_runner` pipe-drain tests. They fail deterministically
  on macOS because its fixture runs `setsid sleep 8`, and macOS has no `setsid(1)`. On
  Linux they are a +12 full-suite flake.

The two highest-corroborated landing-blocking flakes are bob-cli-ju (+20) and bob-cli-ki
(+20). Both are single-sample `performance.now` gates on 200-link Live Preview builds in
bob-plugins, the same pattern T3 fixed for the stage ranker (bob-plugins 87c7b72).
Fixing them together reuses one measurement approach.

Not selected, with reasons:

- bob-cli-io, bob-cli-km, and bob-cli-ej are the same Live Preview pattern but exceed
  the five-bead cap. Leave them for the next chain step, which can reuse this epic's
  helper.
- bob-cli-it (+6): docs-check fails only on stale gitignored mount output in reused
  workspaces. It is green on clean trees.
- bob-cli-jp: a transfer rewrite gets refused; this is daily-workflow pain, not a wrong
  write.
- bob-cli-ks: the plugin test lingers about 30 s and is not a hang. Its 2026-11-18 note
  has the details.
- bob-cli-cd: an untracked new plugin folder in the live vault needs a coordinated,
  Bryan-gated untrack on three hosts.
- bob-cli-mv and bob-cli-mq: design changes.
- bob-cli-f8 (large), bob-cli-fz (+10, a headless Chromium hang): outside this batch.
- Feature and memory beads: lower priority.

No CLI option, JSON schema, release, live-vault, real Gmail, or memory change is
authorized. There are no optional feature, UX, or memory decisions in this batch, so
this plan has no `decisions:` block.

## Shared rules

- Never weaken a collision, safety, or correctness assertion to get green. Do not raise
  a time budget, shrink a workload, skip a whole test on macOS, gate a test behind
  opt-in, or add retries-until-green.
- If the macOS run shows correct product behavior that differs only because the
  filesystem differs, make the fixture or assertion portable. If it shows a real product
  bug, fix the product and keep the Linux assertion.
- Verify on Linux with focused nodes and `just check` (the named `sase tool run check`).
  Never run `just check-full`.
- Verify on macOS by running `scripts/dev/macos-ci-check` from the bob-cli checkout
  under `/sase_monitor`, since it can take up to 90 minutes. It snapshots the working
  tree to a throwaway `ci/precommit/*` branch and prints every macOS step's verdict.
  Never push master yourself.
- Phase workers record unrelated failures as `PROPOSED FOLLOW-UP:` notes on their own
  phase bead, with baseline evidence. They create no beads and make no manual git
  commits.

## Phase macos-case — bob-cli-l9

Follow l9's own repair direction. Assert the filename actually stored, using directory
enumeration. Keep collision rejection, and accept the read-dependency guard that
correctly fires first on a case-insensitive volume (for example,
`eat.base appeared on disk after planning`). Build case-folded ambiguity with a portable
fixture, or split coverage that depends on real-volume case sensitivity (detected at
runtime by a probe file) from the core assertions that always run. In the three
scan_interleave nodes, a case-insensitive volume cannot hold both `clash.pdf` and
`clash.PDF`. Keep the Linux assertions unchanged. On a case-insensitive volume, assert
the actual single-file outcome and prove that no output was overwritten or silently
published. Check the production collision guards with the same probe. If a
case-insensitive volume would let a real collision through, that is a product bug: fix
it.

Run the seven focused nodes and `just check` on Linux. Then run
`scripts/dev/macos-ci-check` and confirm that all seven pass on macos-latest; the
remaining n6 and kl failures are expected there. Close l9 with the Linux and macOS
evidence.

## Phase macos-symlink — bob-cli-n6

Find out why the macOS temp vault (`/var/folders/...`, where `/var` is a symlink to
`/private/var`) breaks these:

- the `xlib` library-root derivation behind `TierContext::library_roots` in
  `src/native/note_catalog/tiers.rs`;
- the `note_goto.vault_root` that capture-complete reports.

Inspect how the library roots and the vault root are resolved, and whether one side
canonicalizes while the other does not. A vault reached through a symlinked path is a
real user configuration (for example, `BOB_DIR` pointing through a symlink). Where
canonical and un-canonical roots disagree inside the product, fix the product so tier
classification and prefix stripping use one consistent root form. Cover that fix with a
Linux test that builds a vault behind a symlink, which reproduces the bug without macOS.
Where only the test's expected path is un-canonical, compare canonical forms. Keep the
reported `vault_root` contract unchanged unless the docs already define it. Read
`docs.md` through `/sase_memory_read` before you change any documented output.

Run the focused nodes, the new symlink regression, `just check`, and
`scripts/dev/macos-ci-check`. Close n6 when its three nodes pass on macos-latest.

## Phase pipe-drain — bob-cli-kl

`src/native/subprocess_runner.rs` tests
`run_bounds_pipe_drain_when_detached_grandchild_holds_output` and
`raw_run_bounds_pipe_drain_when_detached_grandchild_holds_output` race a detached
`setsid sleep 8`. On macOS that command does not exist. On Linux, 92813d08's `sleep 0.1`
narrowed the race without removing it.

Replace the fixture with one where the test controls the lifetime of a holder of the
inherited stdout pipe. For example, the holder can block on a sentinel (a FIFO or a
file) that the test releases after asserting. The holder must escape the runner's
process-group kill portably. Either use a POSIX mechanism available on both Linux and
macOS, or have the test spawn a holder with the pipe fd from Rust, so that no
platform-only binary is needed. Keep the bounded-drain contract and both assertions
exactly: `output truncated` in stderr, `output_incomplete` true, and completion under 4
s. Always clean up the holder, even when an assertion fails.

Run the pair repeatedly in isolation and under parallel load, for example while the full
lib suite runs. Then run `just check` and `scripts/dev/macos-ci-check`. After this
phase, the macos-latest test job should be fully green. If any other macOS-only failure
remains, record it as a `PROPOSED FOLLOW-UP:` with its node and log line. Close kl with
the Linux repetition counts and the macOS result.

## Phase cpu-gates — bob-cli-ju and bob-cli-ki

Open bob-plugins with `/sase_repo` and read its AGENTS.md first. Then inspect
`scripts/test-ledger-tools-off-clock-link-chips.cjs` (the 200-link build under 35 ms)
and `scripts/test-ledger-tools-keep-moving-marks-surface.cjs` (the 200-link, 50-target
Live Preview build under 250 ms, about line 486).

Adopt the convention from bob-plugins 87c7b72
(`scripts/test-navigation-dependencies-stage-entry-and-view.cjs`), which itself mirrors
`scripts/task-wikilink-index-perf-support.cjs`:

- warm up the build;
- take the minimum CPU work over several samples, using `process.threadCpuUsage` when
  available and `process.cpuUsage` otherwise;
- keep the wall samples, clock source, and host load as failure diagnostics.

Put the small CPU-clock helper in one shared test module rather than copying it a third
time. Leave `task-wikilink-index-perf-support.cjs` limited, as its header asks. Keep
both budgets, both workloads, and every functional, memoization, and decoration
assertion unchanged.

Add a deterministic seam check. Parking (Atomics.wait) must not consume a budget, and
real CPU work beyond the budget must fail the same predicate the gates use. If the
measurement exposes a real production regression, fix the bounded cause instead.

Run both files standalone repeatedly, then run the full `npm test` once under
representative parallel load, and keep the samples and the load average. Do not take on
io, km, ej, m1, or f8; record any new wall-clock misses as PROPOSED FOLLOW-UPs naming
those beads. A test-only change needs no plugin version bump. Run `bob plugins sync` as
AGENTS.md requires, and follow its protected-deploy instructions without an unapproved
`-F`. Close ju and ki with CPU and wall measurements and the seam evidence.

## Verification and closeout

Each phase closes only its own assigned task beads and its phase, with concrete source
and test evidence. Use `sase_monitor` for long gates and finish staged handoffs through
`sase_final`. Include every edited repository (bob-cli, and bob-plugins in cpu-gates) in
the final declaration.

The lander verifies all five tasks and the phase notes. It confirms the macOS result
with a fresh `scripts/dev/macos-ci-check` on the landed tree, or with a green master
`test (macos-latest)` job if one has already finished. These steps do not wait for the
lander's own future commit, push, or CI. The lander then rechecks post-start commits,
resolves every `sase bead epic-symbols` entry, closes this epic normally, runs
`just symvision` if available, and marks this plan done. This plan has no parent_bead.

The inherited Landing section below is copied unchanged to continue the backlog chain.
Its T1-specific scan_lock and plugin examples are historical. This T4 scope has no
scan-lock exception, and its only plugin work is the cpu-gates phase.

## Landing

The land agent:

1. Verifies each phase's beads are closed, or left open as the `scan_lock` decision
   directs. Spot-check the evidence: the bob-plugins red nodes now pass, and the bob-cli
   tests named in the close notes exist and pass.
2. Triages every `PROPOSED FOLLOW-UP:` note into task beads through `/sase_new_task`,
   which catches duplicates.
3. **Continues the chain.** List the open task beads with `sase bead list -t task -n 0`.
   Leave out snoozed beads that have not reached their wake time, and beads already
   owned by an active epic. If any eligible beads remain:
   1. **Re-triage.** Close every bead that is verifiably obsolete or duplicate. Concrete
      evidence means a fixing commit SHA, a current `file:line`, a passing targeted
      test, an existing memory key, or the duplicate bead id; "probably fixed" is never
      enough. Use the default resolution for already-fixed beads, `-R superseded` for
      duplicates and overtaken designs, and `-R canceled` for no-longer-applicable
      beads. Read each bead's `2026-11-17 triage` note, and any later triage notes,
      first. Parallel read-only helper agents are fine for this sweep; the land agent
      does the closing.
   2. **Pick the five most important** remaining beads, in this priority order:
      1. wrong or lost vault writes;
      2. deterministic red gates on master;
      3. high-+1 flakes that keep blocking landings;
      4. daily-workflow pain;
      5. everything else.

      Several beads may share one phase, but there are at most five beads per epic.

   3. **Author the next epic plan with `/sase_plan`**:
      - at most five phases, run strictly in sequence (each phase depends on the one
        before);
      - a `decisions:` block kept to a minimum, but one that lets Bryan opt out of work
        that may not be wanted, especially feature beads, UX polish, and memory beads
        (for example the "Record the decision …" beads, and beads whose triage notes say
        they were overtaken or need re-scoping);
      - this whole Landing section copied forward unchanged, so the chain continues.

      Then `sase plan propose` it. When it reports Staged, finish through `/sase_final`.

   4. If no eligible beads remain, say so in the final report. List any snoozed beads
      with their wake times, and stop the chain.
