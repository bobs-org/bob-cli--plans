---
tier: epic
title: Finish collision provenance and quarantine path safety
goal: Preserve earliest durability across copied histories and prove quarantine recovery
  without accepting unsafe paths.
parent_bead: bob-cli-df.2.1.4
phases:
- id: copy-chronology
  title: Preserve earliest history when a later identical copy descends from a competitor
  size: medium
  depends_on: []
  description: 'copy-chronology: repair complete-group selection across copies and
    finish the required Rust import cleanup.'
- id: adapter-recovery
  title: Gather cross-root ancestry and finish safe quarantine recovery coverage
  size: medium
  depends_on:
  - copy-chronology
  description: 'adapter-recovery: pin the core repair, gather cross-root provenance,
    reject unsafe quarantine paths, and prove restart settlement.'
proposed_by: bbugyi200.athena.bob-cli-df.2.1.4.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-28 04:35:23
status: wip
bead_id: bob-cli-df.2.1.4.3
---

- **PROMPT:** [prompts/202610/artifact_link_provenance_closeout.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/artifact_link_provenance_closeout.md)
- **PARENT:** [202610/artifact_link_quarantine_landing_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/artifact_link_quarantine_landing_repairs.md)
- **BEAD:** [bob-cli-df.2.1.4.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.2.1.4.3.md)

# Remaining work for bob-cli-df.2.1.4

This is remaining epic work, not a new feature. **bob-cli-df.2.1.4** and directly
parented plan **bob-cli-df.2.1** remain incomplete; their original phases are closed.
The current landing audit found defects after the previous ordering/replay repair. Do
not repeat that completed work or change the original contracts.

Read these with audited commands:

- `sase bead read bob-cli-df.2.1.4 -r "Need landing findings and dispositions"`
- `sase artifact read file:explicit:4ea0ec4a426fca44ad0e64e8 "Need detailed landing evidence"`
- `sase artifact read file:explicit:93c39bb2954df550e568ab38 "Need the bounded reproduction probes"`
- `sase artifact read plan:202610/artifact_link_quarantine_landing_repairs.md "Need the governing repair contract"`
- `sase artifact read plan:202610/artifact_link_collision_quarantine.md "Need original canonical selection and quarantine contracts"`

There are no discretionary DECISIONS or authorized memory edits. Keep event schema 1,
the existing registered Python bindings, strict low-level reducer behavior, and the Rust
policy/Python IO boundary. Do not edit release versions, changelogs, published
dependency windows, real historical outage data, or unrelated CLI options.

Two sequential medium coding phases are required: the Rust behavioral repair must land
before Python can pin its actual SHA and verify the matching wheel. A tale cannot depend
on its own future commit. No phase in this child epic performs the parent closeout;
`parent_bead: bob-cli-df.2.1.4` hands this child lander back to the interrupted landing.

## Source baseline and integration

Use `/sase_repo` before each external checkout and only the path it returns:

- `sase repo open gh:sase-org/sase-core -r "<reason>"`: audited clean tip **f4bf6074**,
  including original partition **6613c9bb**. Read its AGENTS.md.
- `sase repo open sase -r "<reason>"`: audited clean tip **ac23f63ee4**, including store
  **6470f7d7a0**, publication/outbox **d59c46296f**, and replay repairs **ac23f63ee4**.
  Read its AGENTS.md. Its pin is `f4bf6074cc7651225e968bb6767c2c91eb432b55`.

Fresh fetch during this audit found both tips equal to origin/master. Child-start drift
**dae555aa20** and **0028e6765d** adds Footprint evidence/review UI and no competing
artifact-link reducer. Preserve sibling **1b1162b313** plan-inlet rollback: other
publication failures still need it. Recheck post-audit drift before landing and run its
relevant formatting/plan-propose regression suite.

The audit ran bounded probes, not full feature tests: extracted unmodified Rust
selection helpers with minimal structs/integer instants; the actual Python history
adapter against full synthetic Git repositories; actual quarantine load/reuse with Rust
record validation/path construction stubbed to isolate path handling. Workers must turn
these into real public API/binding/IO regression tests. The current external Python
checkout has no provisioned venv, and exact wheel-cache lookup missed for available
interpreters. Never substitute the globally installed binding or claim historical phase
checks are fresh verification.

## Phase copy-chronology

Repair `crates/sase_core/src/artifact_link/collision/mod.rs` without moving policy to
Python. The existing `representative_entry` now retains every complete history and
chooses the earliest evidence, but `is_proven_ancestor` asks whether **any** history of
one digest precedes **any** history of another. `selection_pool` then eliminates the
entire latter digest. This makes a later exact retry erase earlier durability.

Concrete failing group:

1. A is first added in independent history a0 at 10:00.
2. B is first added in independent history b0 at 11:00.
3. An identical copy of A is added at a1 at 12:00; a1 descends from b0.

Current selection returns B because b0 precedes a1. Expected A: A's independent a0 still
proves earlier durability. The audit's exact helper probe returns B. Preserve
occurrence-level first-add evidence when eliminating later occurrences; do not treat
proof about one later copy as proof about every occurrence of that payload. Select over
the complete group, not a nontransitive pairwise sort. Keep the existing rules for
tier/evidence precedence, actual RFC3339 instants, admission sequences and genuinely
unordered digest ties. Report basis/detail/ambiguity consistent with the actual choice.

Add tests through `partition_artifact_link_collision_events` for this case and all input
permutations/root-label variations, plus interleaved repeated payloads and three or more
distinct payloads. Retain the opposite existing case: A has an independent early copy
and a later copy that is an ancestor of B; B must not win merely through clock skew.
Cover same-commit evidence, equal instants, missing evidence and exact retries. Do not
invent ancestry or let duplicate copies add observations. Preserve malformed-evidence
rejection and strict reducer collision rejection. Update the real PyO3 round-trip
regression for the new case.

Finish the explicit import cleanup from the governing repair plan. Original collision
and quarantine names remain in `crates/sase_core/src/lib.rs` and the retiring
`crates/sase_core_py/src/prelude.rs` (`core_*` aliases and wire/constants). Remove only
those original additions from these facades and move their actual binding/test consumers
to `sase_core::artifact_link::<name>` imports. Preserve domain module exports and every
registered Python binding name. Do not perform a general prelude rewrite.

Run `just test -p sase_core artifact_link` and
`just test -p sase_core_py artifact_link_collision_bindings_round_trip`, then governed
`sase tool run check`. Never run bare cargo or `just check-full`. Use `/sase_monitor`
for long commands; wait for the monitor-start command itself to exit. Fix regressions
from this work, and distinguish recorded clean-base failures from current results.
Resolve/re-key this phase's epic-symbol exemptions, close only this assigned phase, and
let the host finalizer commit. Phase two consumes that actual landed commit.

## Phase adapter-recovery

Open both repositories anew. Inspect the landed core phase and advance SASE's
`sase-core-revision.txt` to its real SHA or a later containing commit. Follow
`docs/contribute/rust-backend.md`; provision an isolated checkout venv and verify the
extension actually loaded is built from that opened source. `bob-cli-di` tracks the
independent installer override bug: use an explicit valid `SASE_CORE_DIR` or verified
wheel and verify the installed binding after editable install; do not repair that
unrelated installer inside this phase. No global install.

### Cross-root history evidence

In `_artifact_link_event_history.py`, `attach_collision_history` currently groups
candidates by root and `_root_history` only compares first-add commits represented
inside that root. Thus it never collects ancestry between one candidate in each of two
related full repositories. The audit's fixture has A at 12:00 in an original repo; a
full clone deletes its copy of A and adds B at 10:00 in a descendant commit. Both
candidates receive complete history without ancestors despite
`git merge-base --is-ancestor` proving A precedes B. The core wrongly sees independent
clocks.

Gather first-add evidence across the whole collided group, then query the available
repositories for ancestry between its known commits wherever the objects are present.
Carry that provenance through the existing wire to Rust. Keep missing/unavailable
ancestry honest, never list the same commit as its own ancestor, and avoid Git history
work for noncolliding groups. Preserve admission evidence and the completed shallow
history fix; do not invent boundary first-add timestamps or redo winner selection in
Python. A root's inability to resolve another commit is missing evidence, not a proof of
independence or a reason to block an otherwise valid store.

Use temporary full Git stores to test the reproduction through actual store/partition
paths, not only `_root_history`. Add a truly independent-root timezone-offset test; the
existing `test_timezone_histories_order_by_instant` places both events in the same
ancestral history, which masks time-comparison mistakes. Retain shallow clone and
duplicate-root tests. Preserve read-only filesystem/Git status.

### Quarantine path safety

In `_artifact_link_event_quarantine.py`, matching-record reuse checks only the leaf
file's symlink status; loading starts recursive scanning at the unchecked namespace
root. A symlink at `link-events/quarantine/v1` makes current load return a valid-looking
finding with no errors and allows reuse of bytes outside the store. The bounded path
probe demonstrates this with stubbed wire validation; implement regressions using real
valid canonical records and the matching Rust binding.

Reject symlinks and non-directory components below the canonicalized caller-supplied
root on load, first persistence and reuse. Cover the namespace and operation directory,
leaf files, broken symlinks, and paths redirected outside the root. Do not reject
legitimate symlinked ancestors of the root such as macOS `/tmp`; compare consistently
after root normalization. Reuse existing safe IO/path conventions where appropriate,
without a broad unrelated installer refactor. Unsafe/malformed records remain typed
blocking corruption, read-only load/doctor stay side-effect free, and publication cannot
report a settlement receipt from unsafe evidence. No writes outside the store.

Keep the completed immutable reuse behavior: a validated existing record with the same
project/operation, accepted/rejected identity and full rejected canonical payload keeps
its original bytes/provenance across basis/locator changes. Reject incompatible
decisions, payload mismatches, wrong operation/digest path and wrong project. Never
swallow corruption simply to remove an outbox entry.

### Finish the required recovery acceptance cases

Add the cases omitted by the prior repair, fixing any epic-caused regressions they
expose:

- Start with **no durable winner** and an outbox holding both same-ID payloads plus an
  unrelated event. Persist quarantine evidence, interrupt before outbox rewrite,
  recreate the store/restart, and drain twice. Existing collision drain tests publish
  the original winner before queueing, so they do not exercise this transition.
- Exercise rejected payload arrival through direct publication then outbox and the
  reverse. Changed provenance labels must preserve the first immutable evidence.
- Cover document-sidecar and local ownerless event lanes. Assert the original accepted
  bytes/projection survive, unrelated events publish, the loser settles exactly once, no
  unseen payload disappears, quarantine bytes never change, observations do not
  multiply, and the second drain has no new settlement.

Retain quarantine-write failure, malformed outbox lines, concurrent appends/publication,
same-path corruption, and rejected alias/remove/put/observation coverage. Preserve the
real temporary-store CLI acceptance: historical colliding files remain byte-identical,
ordinary unrelated link add succeeds, doctor names both payloads, and valid quarantine
does not make the store unwritable.

Run relevant store/core/health/quarantine IO/projection/reconcile, publisher, outbox
drain/IO/parked, CLI add/doctor, binding-validation suites, and
`tests/test_plan_command_handler_formatting.py`. Read `lint_and_test.md` through
`sase memory read`, run `just fix`, and then `sase tool run check` against the verified
source/wheel combination. Use `/sase_monitor` for long setup/checks. Do not run
`just check-full`. Do not classify a new failure as unrelated without a clean-base
reproduction or valid independent witness. Resolve this phase's exemptions and close
only its assigned phase after recording exact results.

## Completed unrelated follow-up triage

The parent landing agent already used `/sase_new_task`, searched across statuses, swept
last-week tasks and inspected causal active epics. Outcomes are on **bob-cli-df.2.1.4**;
do not repeat them as new tasks:

- Proposing bead **bob-cli-df.2.1.4.1**, note #1: additional evidence on existing medium
  ready CI task **bob-cli-dh**, which owns the reported identical 27 Clippy errors on
  clean base 83b010dd. No fresh gate pass/failure is implied.
- **bob-cli-df.2.1.4.2**, note #1: additional evidence on **bob-cli-dg** for missing
  external plans-sidecar validation. Keep this environment repair separate.
- **bob-cli-df.2.1.4.2**, note #2: supplementary DISCOVERED ISSUE on causal containing
  epic **bob-cli-df** for sibling recovery's private `_RecoveryAssessment` import;
  witness `13ea2a464abef73b3eab4bfee9e624d7`. B4 land owns it.
- **bob-cli-df.2.1.4.2**, note #3: new medium ready bug task **bob-cli-di**,
  independently reproduced loss of effective Justfile core settings in the nested
  override helper; the phase reported wheel downgrade. No duplicate or causal active
  epic was found.
- Original **bob-cli-df.2.1.2** note #1 stale Footprint exemptions remain declined as
  already removed by c80a6c791a, with historical impact on sase-o7. No reopening.

Phase workers record new unrelated discoveries as PROPOSED FOLLOW-UP notes on their
assigned phase. Do not hide any defect caused by this epic in a new task.

## Land-agent handoff, not a coding phase

This child lander first audits its own phases, commits, notes, decisions, verification,
pin/wheel and post-start drift; resolves all its epic-symbol entries; closes its own
epic normally; runs `just symvision` where available; and marks its own plan done. Then
follow the direct `parent_bead` link to resume **bob-cli-df.2.1.4**.

For **bob-cli-df.2.1.4**, reread all descendants/notes, the prior landing audit and
dispositions, linked plan and current source. Rerun descendant and linked-plan readiness
checks. When complete, resolve or justifiably re-key every entry from
`sase bead epic-symbols bob-cli-df.2.1.4`, close normally with
`sase bead close bob-cli-df.2.1.4 --note "<verified source, commits, checks, integration and all follow-up outcomes>"`,
run `just symvision` where available, and set `status: done` in
`plan:202610/artifact_link_quarantine_landing_repairs.md` through the opened plans
sidecar. Use audited artifact reads and `/sase_repo` before edits.

Next resume directly parented plan **bob-cli-df.2.1**. Review its previous landing
notes, every descendant/note, `plan:202610/artifact_link_collision_quarantine.md` and
drift. Rerun descendant/linked-plan readiness checks. If complete, retire all entries
from `sase bead epic-symbols bob-cli-df.2.1`, close with
`sase bead close bob-cli-df.2.1 --note "<rechecked original contracts and repairs, verification, drift and dispositions>"`,
run `just symvision`, and mark its linked plan done.

Finally reread parent phase **bob-cli-df.2** and verify the original earliest-durable,
quarantine retention, unrelated append and doctor contract. Resolve its epic symbols and
close **only that phase** with
`sase bead close bob-cli-df.2 --note "<verified completed child plans and original phase acceptance>"`.
Leave containing **bob-cli-df**, its B4 plan, and original task **bob-cli-5m** to the
waiting B4 land agent. At the first incomplete/ambiguous ancestor, leave it open, append
the concrete blocker and report it. Never force a successful nested landing; never force
merely to make a close succeed. No closure step depends on the lander's own future
commit, push or CI result.
