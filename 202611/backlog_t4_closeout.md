---
tier: tale
title: Finish the T4 pipe-holder cleanup and land bob-cli-n7
goal:
  Fix the holder lifecycle and integration formatting, obtain Linux and macOS evidence,
  and complete bob-cli-n7 closeout in the coder turn.
size: medium
status: draft
proposed_by: bbugyi200.athena.bob-cli-n7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-n7
create_time: 2026-11-18 10:56:55
---

- **PARENT:**
  [202611/backlog_triage_t4.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backlog_triage_t4.md)
- **BEAD:**
  [bob-cli-n7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-n7/README.md)

# Finish the T4 pipe-holder cleanup and land bob-cli-n7

This is the remaining work from the land audit of `bob-cli-n7`, not a new backlog batch.
One coder owns the repairs, verification, and the epic's closeout. There is no later
land agent waiting for this tale. The target epic has no `parent_bead`; its linked plan
is `plan:202611/backlog_triage_t4.md` (repository-relative file
`202611/backlog_triage_t4.md` in the plans sidecar). No reviewer DECISIONS apply.

Read the epic's LANDING AUDIT AND FOLLOW-UP TRIAGE note, reopened phase `bob-cli-n7.3`,
and task beads `bob-cli-n6`, `bob-cli-kl`, and `bob-cli-n9` before starting. Read the
original plan through `sase artifact read` for its complete acceptance criteria and
inherited backlog-chain instructions. Use `/sase_repo` before opening linked
repositories or the plans sidecar, and audited artifact reads for sidecar context.

## What the lander already verified

- `509d8572` retains the collision protections while making the fixtures portable. All
  seven required collision nodes explicitly pass in macOS job `134695683540`, run
  `44273496460`, and current-master Ubuntu job `134910729425`, run `44335332552` at
  `f0857be8`. The lander closed original task `bob-cli-l9`.
- `1f6cc7be` canonicalizes library/ref roots consistently with a symlinked vault root.
  Its Linux symlink regression passes in current-master CI. The three macOS regressions
  still lack a completed post-fix macOS verdict. `bob-cli-n6` remains open even though
  phase `bob-cli-n7.2` is closed.
- Plugin `b2a6d2b` keeps both workloads, functional/memoization assertions, and 35/250
  ms budgets; it adds shared warmup/min-of-7 CPU measurement and the parking-versus-spin
  seam. A new landing run passed 71/71 focused tests, with best CPU times 4.48 ms for
  off-clock chips and 54.74 ms for keep-moving marks, load 14.0/18.7/27.9 on 64 CPUs.
  Original tasks `ju` and `ki` and phase `n7.4` are closed. Its full-suite and
  plugin-sync evidence is in their close notes.
- Post-start integration reviewed bob-cli `ce6425f5` (headlines), `2f80442b` (H1-less
  records), `aea85529` (walk attention), and plugin `04ef18f` (headlines) and `c7e6d3e`
  (attention). No new consumer conflicts with the epic's features. The H1-less record
  commit did introduce the formatting blocker below.
- `sase bead epic-symbols bob-cli-n7` reports no entries. The current bob-cli justfile
  has no `symvision` recipe. Recheck both at final closeout.

## 1. Give the detached pipe holder a test-controlled lifetime

Repair the test-only fixture in `src/native/subprocess_runner.rs` introduced by
`f0857be8`. Keep production subprocess semantics unchanged unless a regression proves a
product defect.

The current `PIPE_DRAIN_HOLDER_SCRIPT` calls Python `os.setsid()`, creates a ready file,
and then `time.sleep(8)`. `DetachedHolderGuard::armed` obtains its PID from the runner's
returned stdout. But `receive_before` times out on the intentionally held pipe; `run`
and `run_raw` replace that unread result with empty stdout. Thus the guard has
`pid: None` on the intended successful assertion path. It is also created only after
`run(...).expect`, so an error before that point bypasses it. The phase's claim that its
guard cleans up the holder is not supported by the source. Phase `bob-cli-n7.3` was
reopened for this work.

Use one shared test fixture with control/identity outside the pipe being tested.
Establish its cleanup owner before invoking the runner. The holder must detach portably,
signal readiness only after detaching, and retain the inherited pipe until the test
explicitly releases it. A unique release sentinel/FIFO with an out-of-band identity and
exit acknowledgement is suitable; directly owning a child handle is preferable if the
fixture can preserve the runner scenario. Do not recover cleanup identity from truncated
stdout. A bounded fail-safe may prevent leaks after a harness crash, but a fixed sleep
must not determine the normal holder lifetime. Do not depend on macOS having
`setsid(1)`.

Preserve both existing tests, the exact truncation assertions (`output truncated` and
`output_incomplete`), the 4-second bound, and the production 2-second drain grace. Add
meaningful cleanup coverage for normal return and assertion unwind, plus an early runner
failure where applicable. Prove the holder is gone after cleanup rather than only
inspecting a Drop implementation. Do not claim that SIGKILL alone reaps a child. Use
unique per-test resources and leave no holders or reader threads behind after fixture
teardown.

Run the pair repeatedly without retries-until-green and under parallel lib-suite load,
recording counts and durations. Investigate any failure rather than weakening the bound
or suppressing the assertion.

## 2. Clear integration blockers and obtain the missing verdicts

Apply the rustfmt-only join of the `.all(...)` closure at
`tests/cli/albums_create.rs:1149`. `cargo fmt --all -- --check` independently fails on
clean `f0857be8` with that one diff. Master run `44335332552`, lint job `134910728869`,
shows the same diff; Ubuntu/package pass and macOS is canceled. Keep the H1-less
assertion introduced by `2f80442b`. The lander attached this causal issue to active epic
`bob-cli-ms.8`; append the repair evidence there.

Run the named `sase tool run check` (`just check`) on the repair tree, following
`/sase_monitor` if it escalates or needs a long verification handoff. Never run
`just check-full`. A check-full-only problem is infrastructure, not remaining epic work.
Keep focused evidence for note_catalog's symlink/golden/tier tests, capture note_goto
shape, the seven collision regressions, and the pipe-drain pair.

Run `scripts/dev/macos-ci-check` under `/sase_monitor` on this working tree. It
snapshots uncommitted files to a throwaway `ci/precommit/*` ref, so verification does
not require this tale's eventual commit, push, or post-commit CI. Never push master
manually. Capture the exact snapshot SHA, run/job URLs, required-step coverage, and
named test results. The required macOS job must really pass, including its platform
lint, workspace tests, and archive-move presenter cases.

The separate infrastructure task `bob-cli-n9` records the 45-minute compilation
deadline, proposed by `n7.2` note #3. Evidence is
`file:explicit:5633c7224fa64f51f01da38a`: clean master run `44308576558` at `1f6cc7be`,
macOS job `134816454974`, starts tests at 13:11:54Z, is still compiling bob-obsidian
after 13:14:08Z, and cancels at 13:40:50Z without a test verdict. The same-content
snapshot `44308634344` also timed out. This predates the fixture repair and is not a
failing product test. Distinguish actual job-deadline failures from a canceled
superseded run or the precommit helper stopping on a lint failure.

If the repaired snapshot still cannot finish because of this compile cost, make the
smallest evidence-backed CI build/cache repair needed to run the existing coverage.
Start by inspecting cache restoration and the costs of platform Clippy and test
compilation. A CI-only reduction of dev/test debug-information generation is a bounded
option if compilation evidence supports it; preserve optimization, debug assertions,
overflow checks, all targets, and the full test set. Do not increase time budgets,
shrink workloads, skip tests, weaken the helper's required steps, or rerun unchanged
failures until green. Read `docs.md` through the memory skill before changing
documentation. Coordinate with any worker already assigned to `n9`; reuse its finished
repair if one has landed. Close `n9` only if its own infrastructure remediation is
actually verified; one unrelated green run is not proof of a durable timeout fix. Record
infrastructure limits honestly.

Refresh origin and review any later bob-cli and opened-plugin commits before the final
gate; integrate concrete conflicts without redoing unrelated feature work. No plugin
edits or deploy are presently needed. If a plugin edit becomes necessary, follow its
AGENTS.md and protected sync procedure; never add an unapproved `-F`.

## 3. Finish bob-cli-n7's closeout in this tale

Do this in the same turn as the final code after working-tree verification. Never wait
for this work's own future commit SHA, push, or CI to close the epic. A staged monitor
may resume verification through its successor; that successor still owns this complete
closeout. Do not hand a successful tale back to a nonexistent lander.

1. Read current `bob-cli-n7`, every descendant and note, and the linked plan again.
   Confirm all remaining acceptance criteria, source changes, post-start drift, and the
   exact verified tree. Close `bob-cli-n6` and `bob-cli-kl` with their concrete
   Linux/macOS evidence. Close reopened `bob-cli-n7.3` normally after proving holder
   cleanup and its portable behavior. Do not reopen or re-close already-complete `l9`,
   `ju`, or `ki` without new evidence.
2. Carry all follow-up outcomes into the epic's final note. The lander already completed
   `/sase_new_task` triage: `n7.1` note #1 is existing epic work on `n6`/`kl`, not a new
   task; `n7.2` note #3 became ready CI task `bob-cli-n9`; `n7.3` notes #1/#5 are one
   formatting defect attached to causal epic `bob-cli-ms.8`, repaired as integration
   here; `n7.4` note #1 corroborated `bob-cli-io` (+16) and `bob-cli-m1` (+2). Do not
   create duplicates. The fixture cleanup defect is caused by this epic and is resolved
   here. Triage any genuinely new unrelated discoveries through `/sase_new_task` with
   proposing-bead details.
3. Run `sase bead epic-symbols bob-cli-n7`. For every returned entry, wire up,
   privatize, add a justified non-test pragma, or delete it under Symvision policy;
   re-key only if a specific still-open later bead needs the exemption. Do not leave
   entries keyed to this epic or its closed phases.
4. Close normally with `sase bead close bob-cli-n7 --note "<verification>"`, where the
   note states the code audit, integration repair, holder cleanup proof, passing
   `just check`, exact macOS SHA/job evidence, CPU-gate evidence, and every follow-up
   disposition above. If closing reports stale epic symbols, finish their cleanup and
   retry. If it reports incomplete descendants, finish them; never use `--force` just to
   make the close succeed or advance a successful nested landing. Canceled/superseded
   work requires a deliberate reason and the corresponding non-done resolution, not a
   fabricated success.
5. After closing, run `just symvision` if the recipe is available; otherwise record its
   absence. Open the plans sidecar with `/sase_repo`, and set `status: done` in the
   frontmatter of `202611/backlog_triage_t4.md`. Re-read
   `sase bead read bob-cli-n7 -r "Need the parent link"`; this audit confirms no parent,
   so there is no ancestor to close. If that relationship changes, follow the user's
   parent-phase/parent-plan readiness checks, never force an ancestor.
6. Preserve the original plan's inherited backlog-chain obligation after this epic is
   complete: re-triage eligible tasks, excluding snoozed and active-epic owned work, and
   use `/sase_plan` to propose the next independent batch if any remain (at most five
   tasks, at most five sequential phases, copy its Landing section unchanged). Do not
   delay this epic's close waiting for the next batch. Keep the timeout tracker `n9` in
   that sweep if unresolved. This is continuation of the original plan, not
   authorization to implement the next batch here.
7. Finish through `/sase_final`, declaring every edited repository. If any proposal was
   staged, let its staging command exit and then finalize without waiting on the staged
   successor. Report the epic closed only when the normal close has succeeded; otherwise
   record the actual blocker on the epic.
