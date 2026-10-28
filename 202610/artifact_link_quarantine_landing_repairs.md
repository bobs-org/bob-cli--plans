---
tier: epic
title: Repair artifact-link durability ordering and quarantine replay
goal: Preserve the earliest provable durable payload and make quarantined replays
  settle idempotently without weakening corruption checks.
parent_bead: bob-cli-df.2.1
phases:
- id: ordering-policy
  title: Repair Rust collision ordering across history and duplicate roots
  size: medium
  depends_on: []
  description: 'ordering-policy: preserve earliest evidence, compare actual instants,
    and respect ancestry across the complete group.'
- id: replay-and-evidence
  title: Integrate the repaired core and finish quarantine recovery
  size: medium
  depends_on:
  - ordering-policy
  description: 'replay-and-evidence: pin the Rust repair, detect incomplete history,
    reuse immutable quarantine evidence, and validate its path and project.'
proposed_by: bbugyi200.athena.bob-cli-df.2.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-28 02:07:51
status: done
bead_id: bob-cli-df.2.1.4
---

- **PROMPT:** [prompts/202610/artifact_link_quarantine_landing_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/artifact_link_quarantine_landing_repairs.md)
- **PARENT:** [202610/artifact_link_collision_quarantine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/artifact_link_collision_quarantine.md)
- **BEAD:** [bob-cli-df.2.1.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.2.1.4.md)

# Remaining artifact-link quarantine work

This is remaining work for **bob-cli-df.2.1**, not a new feature. Its three original
phases are closed, but its land audit found correctness gaps. Keep that epic and its
delegating parent phase **bob-cli-df.2** open until these repairs are verified.

Read `bead:bob-cli-df.2.1`, its phase notes, and
`plan:202610/artifact_link_collision_quarantine.md` through the audited commands. The
detailed landing evidence and follow-up dispositions are
`file:explicit:15c5f62a7ee80433016f062f`. Read it with `sase artifact read`. The
original epic has no discretionary DECISIONS or authorized memory edits.

Two sequential coding phases are needed because the Rust policy repair must be committed
before the Python consumer can pin that actual revision and test its wheel. A tale that
needs its own future core SHA before verification and closeout cannot finish this
dependency correctly. There is no separate closeout coding phase; this child epic's
`parent_bead` hands its land agent back to the interrupted landing.

## Audited baseline and boundaries

The source implementation is in two repos, opened through `/sase_repo`:

- `sase repo open gh:sase-org/sase-core -r "<reason>"`: audited tip `31deb0d2`,
  collision implementation commit `6613c9bb`.
- `sase repo open sase -r "<reason>"`: audited tip `a9abdd5c97`, store integration
  `6470f7d7a0`, publication/outbox integration `d59c46296f`.

Use only each returned checkout path and read its AGENTS.md. Do not assume the bare
`sase-core` repo alias exists in this bob-cli host. Fresh fetch at audit time found both
tips equal to origin/master. SASE's source pin is
`6613c9bb917092fece75249cb47a2e8a5236d8fd`, containing all six new bindings.

Keep event schema 1, the strict low-level reducer, the existing binding names, the
Rust-owned winner policy, and Python's filesystem/Git adapter boundary. No release
versions, changelogs, memory, real outage records, or unrelated CLI options change.
Follow the current core convention of module-path imports rather than adding root
exports or prelude aliases; clean up the original collision additions in those retiring
facades while moving their real binding consumers to the module path.

Integration review found no later reducer caller that bypasses the partition. Other
landed changes concern Footprint, recovery snapshots/finalizers, and splits. The sibling
`1b1162b313` plan-inlet rollback is still needed for publication failures other than
collisions; preserve it and run its relevant regression tests when checking publication
integration. Recheck drift before landing.

Unrelated follow-up triage is already complete and recorded on bob-cli-df.2.1:

- Phase `.1` note #1: medium ready CI task **bob-cli-dh** owns the phase-reported
  identical 27 Clippy errors on clean base `83b010dd`. Do not assume a current
  reproduction; record the actual gate result and leave independent lint cleanup there.
- Phase `.2` note #1: closed-phase Footprint exemptions were removed by `c80a6c791a`;
  this proposal was declined as already addressed. Supplementary historical impact is
  recorded on **sase-o7**, with prior causal routing to **sase-1p1**. Do not recreate a
  task or remove its still-needed open exemptions.
- Phase `.3` close note: missing external plans sidecar belongs to **bob-cli-dg**. Its
  private `_RecoveryAssessment` import comes from sibling recovery commit `dcb7d2a6e7`;
  a DISCOVERED ISSUE is on active containing epic **bob-cli-df**. Do not fold either
  unrelated repair into this child.

The audit ran small extracted-function Rust probes and the actual Python history adapter
against a synthetic shallow clone. Those proved the cases below, but are not full
feature or binding test runs. No fresh `just check` pass is claimed. The scratch venv is
unprovisioned, and the globally installed core lacks the new binding. Provision the
current checkout's venv and correct wheel; never use a global install or silently run
tests against that stale wheel.

## Phase ordering-policy

Repair `crates/sase_core/src/artifact_link/collision/` in sase-core. The current policy
has three related defects:

1. `compares_greater` compares `first_commit_at` strings. For independent commits,
   `2026-10-27T12:00:00+02:00` is earlier than `2026-10-27T10:30:00+00:00`, but the
   current function selects the latter. Compare validated actual instants, including
   offsets and equivalent spellings; use existing core timestamp facilities. Malformed
   evidence must produce an explicit validation error, never a fabricated order.
2. The pairwise tournament in `select_winner` is nontransitive when ancestry and skewed
   clocks mix. With digest order a,b,c, a at 12:00Z, independent b at 11:00Z, and c at
   10:00Z whose ancestors include a, the code selects c. It must never select a
   descendant replay while its provably earlier ancestor is present. Select over the
   complete group: retain durability/evidence precedence, eliminate candidates proven
   later by ancestry, then apply the documented independent-history time and
   genuinely-unordered digest rules. Ensure reported basis/ambiguity agrees with the
   actual decision; do not use an inconsistent comparator in a sort.
3. `representative_entry` chooses one identical-digest copy by tier, history flag,
   admission sequence, then source locator. A complete later copy at `a-late` (12:00Z)
   replaces complete earlier evidence at `z-early` (10:00Z). That can make a competing
   payload first added at 11:00Z win. Preserve the earliest provable evidence across
   identical copies; root labels and input order must not change the winner. Retain
   enough ancestry evidence from copies for the complete-group decision. Do not move
   winner selection into Python.

Add regression tests through the real public partition API for all three cases, every
candidate permutation, duplicate roots with different evidence, three or more payloads,
same-commit/tied/unordered evidence, and exact retries. Preserve the
durable-over-pending/incoming guarantee and prove rejected aliases, removals, puts, and
observations stay out of reduction. Retain strict reducer collision rejection. Avoid a
wire break if existing evidence fields suffice; no new binding is needed. Update the
existing PyO3 round-trip coverage so the consumer sees repaired ordering.

Run `just test -p sase_core artifact_link`, the focused sase_core_py artifact-link
tests, and governed `sase tool run check`. Use `just` wrappers, never bare cargo. Use
`/sase_monitor` for long commands, and record outcomes precisely. Do not run
`just check-full`. Close only this assigned phase after its epic-symbol audit. The host
finalizer supplies the committed SHA for phase two; do not manually commit.

## Phase replay-and-evidence

Open both repositories again. Inspect the actual landed core repair and advance SASE's
`sase-core-revision.txt` to that revision or a later containing revision. Follow
`docs/contribute/rust-backend.md`, and verify the wheel used in the isolated SASE venv
comes from that opened source. This pin update uses phase one's already landed commit,
never this phase's future commit.

Repair the following bounded IO/publication gaps in SASE:

1. `_artifact_link_event_history.py::_root_history` sets `history_complete=True`
   whenever `git log --diff-filter=A` returns a commit. In a depth-1 clone, Git treats
   the shallow boundary as the file's add, even when the file was added earlier. The
   audit's actual-module probe returned complete history and the wrong first-add commit
   for a two-commit fixture. Detect incomplete/shallow history and mark evidence
   honestly. Preserve any reliable evidence without elevating a boundary guess over an
   established original. Gather cross-root provenance needed by the repaired core
   without implementing a second ordering algorithm. Normal reads must not walk Git
   history for every noncolliding event.
2. `_persist_incoming_quarantine_findings` writes a loser before the first winner is
   installed. `persist_artifact_link_quarantine_record` regenerates metadata on each
   retry. Once the winner is durable, selection changes from
   `digest_tie_break_unordered` to `durable_beats_pending` or `durable_beats_incoming`,
   with changed detail/ambiguity/locator. Immutable no-replace installation then reports
   different bytes and gives no quarantine settlement receipt, so the loser stays queued
   indefinitely. Reuse a validated existing immutable decision record when its
   accepted/rejected identity and full rejected payload match the same decision.
   Preserve its original bytes and provenance. Different payloads, malformed evidence,
   or incompatible decisions remain corruption; do not simply catch and ignore byte
   mismatches. Ensure partial publication and retries cannot retire an unseen payload or
   claim success before its required evidence is durable.
3. `load_artifact_link_quarantine_records` validates JSON fields but not the record's
   operation/digest path or expected store project. Enforce both, plus the existing
   symlink/corruption rules, in the new namespace. Keep valid quarantine diagnostic but
   nonblocking; malformed or misfiled records remain blocking. Read-only store and
   doctor operations must remain side-effect free.

Test same-batch collisions starting with **no durable winner**, both direct publication
and an outbox containing winner, replay, and an unrelated event. Exercise document
sidecars and local ownerless events. Inject failure after evidence persistence and
before outbox rewrite, restart, and drain twice. Verify the unrelated event remains
published, original bytes and projection stay correct, the loser settles exactly once,
quarantine record bytes never change, and observations do not multiply. Cover the same
rejected payload arriving through direct publication then outbox (and the reverse), so
changing provenance labels cannot strand it. Retain existing tests for quarantine-write
failure, malformed lines, concurrent appends, same-path corruption, rejected
aliases/removals, and concurrent publication.

Add temporary-Git-store tests for shallow history and differing time-zone histories, and
wrong-project/wrong-path quarantine records. Keep the real CLI acceptance test that
reads two historical colliding objects, successfully adds an unrelated link, reports
both payloads in doctor, and preserves both original event files.

Run the relevant suites in the checkout venv, including:

- `tests/sdd/test_artifact_link_event_store_quarantine.py`, store core/health,
  quarantine IO, publisher, and related projection/reconcile tests.
- `tests/main/test_artifact_link_outbox_drain.py`, outbox IO and parked coverage,
  artifact link add and link health doctor.
- Binding validation tests and `tests/test_plan_command_handler_formatting.py` for
  sibling inlet rollback compatibility.

Read `lint_and_test.md`, run formatting (`just fix`) before the governed
`sase tool run check`, and use `/sase_monitor` for long setup/check commands. Do not run
`just check-full`. Reproduce any newly attributed unrelated failure on an unchanged base
or cite a valid independent witness; fix all regressions caused by this work. Each
worker records other discoveries as PROPOSED FOLLOW-UP notes, audits its own epic
symbols, and closes only its assigned phase.

## Landing handoff

This child's land agent audits both phases, actual landed commits, all notes, the SASE
pin/wheel combination, test evidence, and post-start drift. It follows the normal close,
post-close `just symvision` where available, and plan-status protocol for **this child
epic**, then resumes directly parented plan **bob-cli-df.2.1**. There is no parent
closeout phase in this plan.

For that resumed parent audit, reread every descendant and note, the previous landing
audit and dispositions above, and the original plan
`plan:202610/artifact_link_collision_quarantine.md`. Rerun descendant and linked-plan
readiness checks. When fully complete, resolve/re-key every
`sase bead epic-symbols bob-cli-df.2.1` entry to a justified still-open owner, close
normally with
`sase bead close bob-cli-df.2.1 --note "<verified behavior, commits, checks, drift and all follow-up outcomes>"`,
run `just symvision` where available, and set `status: done` in that original plan
through the opened plans sidecar. Use `sase artifact read` to read sidecar artifacts and
`/sase_repo` before edits.

Then reread parent phase **bob-cli-df.2**, verify that these repairs complete its
original quarantine-and-unrelated-append contract, resolve its epic-symbol entries, and
close **only that phase** normally with the verification note. Do not close containing
epic **bob-cli-df** or original task **bob-cli-5m**: the waiting B4 land agent owns them
and the unrelated recovery lint issue recorded there. Never force a successful nested
landing. At the first incomplete/ambiguous parent, leave it open, append the concrete
blocker, and report it.
