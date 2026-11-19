---
tier: tale
title: Finish and close chop launch phase sase-1tc.3
goal:
  The chop-launch phase and its child epic close normally with no epic-symbol rows or
  Python policy copies left.
size: medium
decisions:
  generation_mirror:
    ask:
      Retire the Python copy of clan-generation matching so cancellation trusts the
      pinned Rust policy?
    choices:
      retire:
        Rust answer only; five helpers and two old-pin skips go; sase-1tm closes as done
      keep: Python copy and veto stay; sase-1tm gets a note and stays ready for triage
    default: retire
    why:
      Pinned core 6f8f016e already has the veto; rust-core-required forbids a Python
      copy
    answer: retire
proposed_by: bbugyi200.athena.1e9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 05:05:52
status: wip
---

# Finish and close chop launch phase sase-1tc.3

Phase **sase-1tc.3** (`chop_launch`, target task **sase-1c2**) of outer epic
**sase-1tc** is still open. Its only child is epic **sase-1tc.3.1**, which is also still
open. All six of that epic's children are closed: phases `.1`–`.5` and child epic `.6`.
Production-shaped `TASK RESOLVED: sase-1c2` evidence is already recorded as sase-1tc.3
note #4, superseding note #2. What remains is a small symbol cleanup, a leftover Python
copy of Rust policy, and the bead closes. The outer lander `sase-1tc.land` and phases
`sase-1tc.4` and `sase-1tc.5` are waiting behind this phase. `sase-1tc.3.1.land` is no
longer running. The user asked for this phase to be finished and closed, which requires
closing `sase-1tc.3.1` first, because closes never cascade.

All code work is in the **sase** repository. This bob-cli-hosted agent must open it with
`/sase_repo` (`sase repo open sase -r "<why>"`), read the `AGENTS.md` it names, and work
only in the printed path. No sase-core change is needed. Do not edit
`sase-core-revision.txt`, docs, or memory.

## Read first

- `sase bead read` for sase-1tc.3, sase-1tc.3.1, sase-1tc.3.1.6 and sase-1tm, including
  every note and +1.
- `sase artifact read plan:202611/chop_launch_recovery.md` and
  `plan:202611/chop_recovery_closeout.md`. The closeout tale's final-step walk is the
  model for this landing.
- From inside the sase checkout,
  `sase memory read symvision.md lint_and_test.md sase_beads.md -r "<why>"`.

## Audit baseline and status of every recorded blocker

Baseline: sase `065eda332b` (origin/master at audit). The core pin `6f8f016e8d` equals
sase-core master HEAD. First, fetch and review any commits since the baseline that touch
`src/sase/axe/*chop_batch*`, `tests/axe/*chop_batch*`, or the Justfile symvision recipe,
and adapt the steps below to them.

1. **Two epic-symbol rows (sase-1tc.3.1 note #3). Still open; this tale fixes it.**
   `789e5d0fc5` cleared 13 of the 15 rows. `sase bead epic-symbols sase-1tc.3.1` still
   lists `ChopBatchReplayError` and `ChopBatchReplayResult`, both defined in
   `src/sase/axe/chop_batch_replay.py`. Neither has a non-test consumer:
   - `_chop_batch_recovery_resume_ordinary.py` imports only
     `replay_ordinary_chop_batch`, reads fields off the result, and turns any exception
     into a `blocked` recovery result.
   - Only `tests/axe/test_axe_chop_batch_replay.py` imports the error class.
2. **mypy and oversized-test failures (sase-1tc.3.1 note #4). Already fixed.**
   `5467217dd0` narrowed `_read_blocked_roots_strict` to `Mapping[str, Any]`, and
   `6200573ecc` split `test_axe_chop_batch_recovery.py` into modules under 500 lines.
   The lint job of Master Gate run 44456688642 on `065eda332b` passed.
3. **Generation-cancellation failure (sase-1tc.3.1 note #5 and sase-1tm). Fixed, with
   cleanup left.**
   - The replay and store cancellation nodes were fixed by `4e5c1b9730` (sase-1tm note
     #1).
   - `test_older_generation_does_not_cancel_a_batch_without_recorded_generation` was
     fixed by a Python veto in `5467217dd0` plus core `6f8f016e`. That core commit makes
     Rust `chop_batch_generation_matches` refuse an older kill when the selected clan
     has no recorded generation, and it is now the pin.
   - sase-1tm note #2 asked for exactly that core change "so the wrapper can go back to
     trusting a Rust True".
   - `_chop_batch_recovery_cancel._generation_matches` still keeps a full Python copy of
     the policy: `_scoped_clan_generations`, `_unrecorded_older_generation`,
     `_older_generation_predates_batch`, `_batch_started_at` and
     `_generation_started_at`. It falls back to that copy whenever the Rust call raises.
4. **Unrelated CI failure.** That Master Gate run's only failure is five ffmpeg tests in
   `tests/ace/tui/artifact_file_viewer/test_video_frames.py`. It is not chop work; do
   not fix it here.

## Code changes (sase repo)

### Make the replay symbols private

In `src/sase/axe/chop_batch_replay.py`:

- Rename `ChopBatchReplayResult` to `_ChopBatchReplayResult` and `ChopBatchReplayError`
  to `_ChopBatchReplayError`. The error still subclasses
  `chop_batch_store.ChopBatchError`.
- Update every use in that file, and remove both names from `__all__`.
  `replay_ordinary_chop_batch` stays public.

In `tests/axe/test_axe_chop_batch_replay.py`, stop importing the error. Assert
`pytest.raises(chop_batch_store.ChopBatchError, match=...)` with the existing match
strings, so each test still pins the specific refusal.

Delete the two `--epic-symbol "sase-1tc.3.1(...)"` rows from the `_lint-symvision`
recipe in the `Justfile`. Keep the warning comment against putting comments inside the
continued command.

Re-run `grep -rn ChopBatchReplay src tests tools docs` and expect only the new private
names. Making these private follows symvision.md's order of fixes: a symbol used only in
its own file becomes private. Deleting the class is not an option, because the in-file
code still uses it. A non-test consumer added only to satisfy the linter is not a real
consumer.

### Trust the Rust generation policy (`generation_mirror = retire`)

In `src/sase/axe/_chop_batch_recovery_cancel.py`:

- `_generation_matches` builds the same request (`schema_version`, `manifest`, `clan`,
  `generation`) and returns `bool(chop_batch_generation_matches(...))` directly, with no
  `try`/`except`, no veto, and no fallback.
- Delete the five Python helpers listed above and the misplaced
  `from datetime import UTC` import.
- Rewrite the docstring to name the Rust policy as the single authority. Keep the
  documented rule that selecting either clan of a mixed-clan batch cancels the whole
  batch.

Let Rust errors propagate:

- `load_chop_batch` has already validated the manifest through Rust
  `validate_chop_batch_manifest`.
- `cancel_chop_batches_for_clan` returns early on a blank clan.
- So an exception here means the core install is broken, and that should fail loudly,
  per the rust-core-required decision.

In `tests/axe/test_axe_chop_batch_recovery_cancel.py`, delete the two
`if not hasattr(sase_core_rs, "chop_batch_generation_matches"): pytest.skip("...fallback covers old pins")`
guards. These tests must keep passing with their assertions unchanged:

- `test_cancel_targets_only_matching_clan_generation`
- `test_older_generation_does_not_cancel_a_batch_without_recorded_generation`
- `test_cancel_scoped_to_requested_clan_generation`
- both `test_rust_generation_match_binding_*` tests

If the reviewer picks `keep`, skip this subsection. Then add a `sase bead note` to
sase-1tm saying the core change has landed and the wrapper cleanup was declined, and
leave sase-1tm open.

## Verify

Do not weaken assertions, add skips, change flake baselines, or raise timeouts. Do not
run `just check-full`.

1. If the opened checkout has no `.venv`, run `just install-venv`. It can take more than
   10 minutes, so run it through `/sase_monitor`.
2. Run focused regressions with the checkout's venv pytest:
   - `tests/axe/test_axe_chop_batch_replay.py`
   - `tests/axe/test_axe_chop_batch_store.py`
   - every `tests/axe/test_axe_chop_batch_recovery*.py`
   - `tests/axe/test_axe_chop_batch_process_recovery.py`
   - `tests/ace/tui/test_agent_collapsed_panel_kill.py`

   Then run, by name, the three sase-1tm nodes:
   - `test_axe_chop_batch_replay.py::test_clan_cancellation_during_launch_settles_current_and_stops_next`
   - `test_axe_chop_batch_store.py::test_cancellation_priority_and_reconcile_selection`
   - `test_axe_chop_batch_recovery_cancel.py::test_older_generation_does_not_cancel_a_batch_without_recorded_generation`

3. Re-run the disposable demonstration:
   `python -m tests.axe.chop_batch_process_harness demo`. It uses isolated state only,
   with no live scheduler, job, or provider. Expect stop/resume to spawn 1, then the
   remaining 3, then 0. Expect the clan kill to stay cancelled with a cancelled notice.
4. Run the Symvision gate (`just symvision`, or through `sase tool run` if the recipe
   guard refuses a raw run). It must pass with no `sase-1tc.3.1` rows.
5. Run `sase tool run check` inside the sase checkout, through `/sase_monitor` if it is
   long, with the remaining steps in the follow-up.
   - Any UNKNOWN failure in touched chop or cancellation code is yours to fix.
   - Handle unrelated failures per `/sase_new_task` (corroborate an existing task, or
     file one) instead of fixing them here.

Record the base SHA, ToolRun id, and test counts for the close notes.

## Landing order: commit first, close after the commit is on origin/master

`sase bead close` refuses while any `--epic-symbol` row for the bead or its descendants
remains. It checks two places: the sase primary checkout's working-tree Justfile and
`origin/master:Justfile`. `sase bead epic-symbols sase-1tc.3.1 -f json` shows both
sources. The Justfile change lands only when the host finalizer commits and pushes after
the coding turn ends. So the closes cannot happen in that same turn.

1. In the coding turn, once verification is green, stage a `/sase_monitor`. The host
   starts it after this turn's finalizers commit and push, so `/sase_final` should give
   the sase repo a `commit` decision with bead action `keep`.
   - The monitor command polls `sase bead epic-symbols sase-1tc.3.1 -f json` every 60 s
     until `entries` is empty, with a bound of about 45 minutes.
   - Its follow-up runs the close walk below.
2. If the bound expires while a source still lists the rows, the follow-up adds a
   blocker note to sase-1tc.3.1 naming the stale source and stops. For example, the
   primary checkout may not have auto-synced yet. Never use `--force` and never re-key
   the rows.

## Close walk (monitor follow-up, after the rows are gone everywhere)

1. **sase-1tm** (only with `retire`): close with `--note` citing:
   - `4e5c1b9730` for the replay and store nodes;
   - `5467217dd0` plus core `6f8f016e` for the older-generation node;
   - the landed wrapper commit SHA;
   - the passing three-node run.
2. **sase-1tc.3.1**:
   - Re-read its notes and descendants, and confirm that every descendant is closed.
   - Run `sase plan links validate --show-warnings` and inspect any findings for this
     plan chain.
   - Close with `sase bead close sase-1tc.3.1 --note "<...>"`. The note covers: the six
     closed children; land note #2's eleven defects, fixed by child epic `.6` and its
     closeout repairs; note #1's triage outcomes; note #3's symbols (13 cleared by
     `789e5d0fc5`, 2 by this change); note #4, fixed by `5467217dd0` and `6200573ecc`;
     note #5 and the sase-1tm outcome; drift since `d3fcb19562`; and the verification
     evidence.
   - Run Symvision again after the close.
   - Then set `status: done` in `202611/chop_launch_recovery.md` in sase's plans
     sidecar. Opening the sase project with `/sase_repo` provisions that sidecar at
     `sase/repos/plans/` under the printed checkout. Make sure the follow-up's
     `/sase_final` lists that repo as an obligation with a `commit` decision. If it is
     not listed, revert the edit and add a note on sase-1tc.3.1 that the plan status
     still needs `status: done`.
3. **sase-1tc.3**:
   - Check the phase's "Done when" in
     `plan:202611/backlog_epic_cmdsub_retries_launches.md` against the evidence:
     - stop after k of n launches resumes exactly n − k;
     - a second restart launches nothing;
     - a killed clan's journal is discarded;
     - notices name the clan and counts;
     - check passed;
     - a disposable stop/resume demonstration is recorded (note #4 plus the re-run
       above).
   - `sase bead epic-symbols sase-1tc.3` must be empty.
   - Close with `sase bead close sase-1tc.3 --note "<combined verification>"`.
4. Leave epic **sase-1tc**, its plan file's `wip` status, and task **sase-1c2** to the
   waiting outer lander `sase-1tc.land`. Its plan rule is that the lander closes task
   beads.
5. Finish through `/sase_final`. Report exactly which beads were closed and any that
   remain blocked, with the reason.
