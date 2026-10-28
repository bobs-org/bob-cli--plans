---
tier: epic
title: Quarantine artifact-link collisions and keep unrelated writes available
goal: Preserve the earliest durable artifact-link payload, retain conflicting replays
  as visible quarantine evidence, and allow unrelated reads, writes, and outbox publication
  to complete.
parent_bead: bob-cli-df.2
phases:
- id: collision-policy
  title: Add the Rust collision partition and quarantine contracts
  size: medium
  depends_on: []
  description: 'collision-policy: add canonical selection from durability evidence,
    typed quarantine results, and Python binding coverage in sase-core.'
- id: store-quarantine
  title: Read colliding stores safely and expose quarantine in doctor
  size: medium
  depends_on:
  - collision-policy
  description: 'store-quarantine: add historical durability evidence, accepted-event
    snapshots, quarantine persistence helpers, and doctor reporting in sase.'
- id: publication-quarantine
  title: Quarantine publication replays without losing outbox payloads
  size: medium
  depends_on:
  - store-quarantine
  description: 'publication-quarantine: support sidecar and local installation, mixed
    publication batches, digest-aware outbox settlement, and end-to-end acceptance
    tests in sase.'
proposed_by: bbugyi200.athena.bob-cli-df.2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-27 21:43:09
status: wip
bead_id: bob-cli-df.2.1
---

- **PROMPT:** [prompts/202610/artifact_link_collision_quarantine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/artifact_link_collision_quarantine.md)
- **PARENT:** [202610/backlog_batch_b4.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b4.md)
- **BEAD:** [bob-cli-df.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.2.1.md)

# Artifact-link collision quarantine

## Scope and evidence

This child epic completes phase `bob-cli-df.2` of `plan:202610/backlog_batch_b4.md`.
Read that phase, its parent plan, original task `bob-cli-5m`, historical repair
`bob-cli-21`, and sibling `bob-cli-5n` through audited commands. The parent has no
discretionary decisions or authorized memory changes for this work. Its requirements are
final: keep the earliest durable payload, quarantine the conflicting replay, report it
through `sase artifact doctor`, and allow unrelated link appends. Preserve history; do
not repeat the manual deletion on bob-cli-21.

The phase was in progress with no prior work notes during planning. Both opened source
checkouts were clean. Inspection confirmed the work has not landed:

- At sase `daf54c5b2d39ed483df935fbf98b209b47e1f276`,
  `src/sase/sdd/_artifact_link_event_store.py` passes all durable/pending events to the
  strict reducer. A collision enters `reduction_errors`, and `assert_healthy` rejects
  the whole store.
- `_artifact_link_event_install.py::reject_existing_operation_collisions` rejects an
  incoming conflicting payload. `_artifact_link_event_publish.py` calls it for both
  document roots and local events; `_dedupe_event_objects` also rejects mixed batches
  before the publication lock is acquired.
- At sase-core `83b010dd`, `crates/sase_core/src/artifact_link/events.rs`
  `dedupe_events` rejects different canonical bytes under one operation ID, with an
  explicit `reduction_rejects_operation_id_collisions` test. No quarantine policy is
  present. The PyO3 entry point is in `crates/sase_core_py/src/artifact_links/mod.rs`.
- `_artifact_link_outbox_drain.py` settles batches using operation-ID sets/maps;
  `_artifact_link_outbox_io.py::rewrite_artifact_link_outbox_without_ids` also drops
  duplicate IDs. A fix that merely skips a collision can silently discard its other
  payload or repeatedly fail the same batch.
- Doctor's relevant path is `src/sase/artifact_cli/_link_health_signals.py` through
  `link_health.py` to `doctor.py`, rather than only the general `sase doctor` checks.

Three sequential coding phases keep the Rust contract, store integration, and write
settlement bounded. A single tale would span a new cross-repository contract, historical
provenance, read projections, publication receipts, and queue retention. No production
files were changed and no test suite was run during planning.

## Shared implementation contract

### Canonical selection

Collision policy belongs in sase-core. Python gathers filesystem/Git evidence, performs
authorized side effects, and renders results. It must not implement a second
winner-selection algorithm.

Canonical identity is canonical event bytes/digest, not just operation ID. Exact retries
collapse without quarantine and do not increase observation counts. Different payloads
with the same project/operation ID produce one accepted payload plus typed quarantine
findings for the others. Do not weaken schema, project, path/digest, symlink,
alias-cycle, or other corruption checks to achieve availability.

Preserve the earliest durable object. An already durable payload wins over an incoming
or pending replay, even when that replay claims an earlier `created_at`. For historical
durable collisions, gather first-add Git history only for the collided digests; use
ancestry to distinguish the original from a descendant replay and first-add commit time
where histories are independent. Neither directory enumeration, digest order, checkout
mtime, nor event `created_at` proves which was durable first. Derived events can
deliberately have the same epoch `created_at`.

For future local-only objects, record durable admission evidence under the existing
publication lock so restart retains the first accepted payload. Older local-only
objects, shallow/missing history, and simultaneous independent histories may not provide
a provable ordering. Represent that explicitly in the policy input/output: prefer
established durable evidence, use a stable digest tie-break only among genuinely
unordered candidates, and report the missing/ambiguous evidence with the quarantine. Do
not invent timestamps or make the entire store unusable because legacy chronology is
unavailable. Input order and repeated copies across roots must not alter the result when
the same evidence is supplied. Test the historical original-before-replay case with
identical event timestamps and reversed digest/path order.

### Retention and visibility

Use logical quarantine for already stored event objects: retain their exact bytes at
their existing content-addressed paths, exclude losing payloads from active reduction,
and carry findings to doctor. Read-only snapshots and doctor do not write or commit a
repair. Historical collisions must work before a mutating maintenance pass.

When rejecting an incoming payload or removing it from the active outbox, persist
idempotent quarantine evidence first. Use a versioned namespace outside the active
`link-events/v1` scan, such as
`link-events/quarantine/v1/<operation_id>/<rejected_digest>.json`. Preserve the full
rejected canonical payload, canonical and rejected digests, project/operation ID, source
locator, selection basis, and any ambiguity. Keep provenance separate from the immutable
event schema. A persisted record and its payload must survive restart; repeat replays
must not grow duplicate records. An existing record with different bytes is corruption,
not permission to overwrite it.

Follow existing sidecar ownership/commit rules for sidecar evidence and the existing
machine-local lane for ownerless events. Machine mutations never write through the
primary checkout's sidecar clones. Do not modify the real historical outage data as part
of testing.

Quarantine is a diagnostic that does not block unrelated writes. `assert_healthy`
continues to reject actual blocking corruption. Doctor must visibly identify the
operation and both payloads, the accepted payload and selection basis, and where the
rejected bytes remain. Include quarantine in the artifact doctor's unhealthy findings
and exit status without conflating that diagnostic status with store writability.

### Work and verification rules

Open sase with `/sase_repo` and use only its returned path. Open sase-core through that
skill too. In this bob-cli host context, bare `sase repo open sase-core` was not in the
host inventory; `sase repo open gh:sase-org/sase-core -r "<reason>"` successfully opened
it. Do not assume changing directories changes the host project inventory. Read both
AGENTS.md files and applicable reference memory before editing.

Follow the Rust backend boundary. Prefer an additive partition API/wire contract so the
strict low-level reducer remains a useful validator; all store and publication callers
must partition before reducing. Preserve immutable event schema 1 and existing bindings.
Register and test any added binding. Do not change release versions, changelogs,
published dependency windows, memory, or unrelated CLI options.

Use `/sase_monitor` for long setup/check commands, wait for each handoff command to
exit, and resume from its recorded results. In sase-core never run bare cargo: use
`just test -p sase_core <filter>` and `just test -p sase_core_py <filter>`. Run
`sase tool run check` in every changed source repo; it is the governed `just check`
gate. Do not run `just check-full`. In sase read `lint_and_test.md`, format before the
final check, and run focused Python suites using its checkout virtualenv and the
matching core wheel. No global install.

Core changes must land before a sase consumer calls the new binding. After the core
phase's host finalization, inspect the landed SHA and advance sase's
`sase-core-revision.txt` to a revision containing it, following
`docs/contribute/rust-backend.md`. Check the actual pin and wheel; do not assume an
automatic sibling pin update works for externally opened repos. Never invent a SHA or
manually commit; the finalizer owns commits.

Each worker closes only its assigned child phase after running
`sase bead epic-symbols <assigned-phase>` and resolving/re-keying any entries to a
still-open owner. Put discovered unrelated work in `PROPOSED FOLLOW-UP:` notes on that
phase. Reproduce purported unrelated check failures on the clean base or cite an
independent established witness; record them and close completed phases rather than
leaving them open. Fix regressions caused by this work.

## Phase collision-policy

Implement a deterministic partition function and typed request/result wires in
`crates/sase_core/src/artifact_link/`, exporting through its module facade and a
registered PyO3 binding in the existing artifact-links domain. Put new code/tests in
bounded modules rather than enlarging the already long `events.rs`.

Inputs must distinguish durable, pending, and incoming candidates and carry explicit
durability provenance plus canonical bytes/digest identity. Results must retain accepted
events, rejected payloads/findings, and the selection basis; the consumer must be able
to associate every outcome with a digest, including two outcomes sharing one operation
ID. Define/validate the versioned quarantine record with the same core policy.
Filesystem reads, Git subprocesses, and commits remain in sase's adapter.

Cover exact retries, distinct operations, preexisting durable versus incoming replay,
two historical durable payloads with known ordering, missing/tied evidence, permutations
and duplicate roots, and three payloads under one operation ID. Verify rejected alias,
put, remove, and observation events cannot affect accepted reduction. Invalid individual
events must still error. Round-trip the new binding and its registration; retain strict
reducer collision validation if introducing a partition API. Record the contract and
focused/full-gate evidence on the phase for downstream workers.

## Phase store-quarantine

Consume the landed core API and update the source pin before calling it. Implement the
thin evidence loader and atomic quarantine/admission-record IO shared by reading and
publication. History lookup is exceptional work for actual collided groups, not a Git
walk for every event on every normal read. Reuse established lock ordering and durable
install primitives; quarantine record installation must use no-replace plus fsync.

Integrate partitioning into `_artifact_link_event_store.py`. Choose accepted events once
across the durable/pending union, then derive durable, pending, and combined rows from
that selection. A separately reduced pending replay must not override the durable
winner. Preserve raw counts for diagnosis while exposing accepted events to projection
consumers and rejected identities through a separate snapshot field.

Audit the alternate reduction paths in `_artifact_link_event_canonical.py`,
`_artifact_link_event_project.py` (including active-operation lookup and bead
projections), and `_artifact_link_store_reconcile.py`. They must share the same accepted
selection and provenance rather than re-reducing raw colliding objects or allowing
legacy/aggregate rows to resurrect a rejected edge.

Carry quarantine through `_link_health_signals.py`, `link_health.py`, and `doctor.py`,
including both historical logical quarantine and stored rejected incoming payloads. Keep
blocking errors distinct. Add focused store/health/doctor tests using temporary Git
stores and local roots. Verify strict snapshots and ordinary reads succeed for a
colliding pair, canonical content is preserved, a fresh unrelated link add succeeds,
doctor reports the pair, read-only checks leave files and Git status unchanged, and
malformed/path-corrupt objects remain blocking. Relevant existing suites include
`tests/sdd/test_artifact_link_event_store_core.py`,
`test_artifact_link_event_store_health.py`, and
`tests/main/test_artifact_cli_link_health_doctor.py`.

## Phase publication-quarantine

Replace blanket batch rejection in `_artifact_link_event_publish.py` and
`_artifact_link_event_install.py` with the shared classification and explicit
quarantined outcomes. Cover document-sidecar roots, local ownerless/bead-only objects,
and same-batch collisions. Classify under the publication lock with current durable
evidence before projection or installation, so a rejected event cannot enter another
root or mutate bead/aggregate state first. Recheck under the existing root lock where
needed to preserve concurrent publication guarantees. Quarantine failures retain the
input for retry and never fabricate a published receipt.

Unrelated accepted events in the same batch must still publish. Preserve the original
payload, distinguish a quarantined replay from successful publication, and report its
diagnostic. Update existing tests that expect blanket rejection, including the
committed-sidecar and bead-only collision cases in
`tests/sdd/test_artifact_link_event_publisher.py`; keep same-path byte corruption and
symlink rejection coverage.

Trace enqueue, drain, rewrite, and parked-ledger interactions in
`_artifact_link_outbox_io.py`, `_artifact_link_outbox_drain.py`, and related types. Use
`(operation_id, digest)` where settling payloads would otherwise discard a sibling with
the same ID. Do not reuse the endpoint parked ledger blindly: it also deduplicates by ID
and has separate endpoint-retry semantics. Preserve both canonical and rejected payload
evidence before retiring an active entry. Existing by-ID caller contracts may remain
where IDs are provably unique, but collision settlement must not use them to drop unseen
payloads. Retain concurrent appends and malformed unrelated lines.

Add a mixed-batch/outbox regression with a canonical payload, conflicting replay, and
fresh unrelated event. Verify the fresh event is committed and listed, the canonical
projection stays unchanged, the replay is durably quarantined, and a second drain or
restart does not retry it indefinitely or multiply observations. Inject failure between
quarantine persistence and outbox rewrite to prove recovery is idempotent. Also cover a
conflicting alias/edge-remove so exclusion protects state-changing events, and preserve
the existing publication concurrency test.

Finish a temporary-store CLI acceptance test that starts with two historical durable
objects under one ID, calls the ordinary artifact link add path for an unrelated edge,
and inspects artifact doctor. Check both original files remain intact. Run focused
publication/outbox/CLI tests and the governed sase check.

## Landing and closure

The child epic's land agent reviews all three phases and their actual landed commits,
finishes gaps caused by this epic, verifies the core pin includes the new binding, and
checks the final integrated acceptance evidence. Run governed `just check` in the
changed repos against the final source/wheel combination; reuse valid recorded results
where the relevant tree is unchanged. Confirm no unrelated edits to the sibling 5n
plan-propose ordering fix or cv workspace recovery work slipped into this change.

After all child phases close and this epic is verified, the land agent resolves its
epic-symbol entries, closes this child epic, and marks this child plan done. It then
runs `sase bead epic-symbols bob-cli-df.2` again, resolves/re-keys any remaining
entries, and closes only the delegated parent phase with
`sase bead close bob-cli-df.2 --note "<commits, canonical selection, quarantine retention, unrelated append and doctor evidence, checks and recorded unrelated failures>"`.
Do not close `bob-cli-df`, any other ancestor plan, or original task `bob-cli-5m`: the
B4 land agent owns those task/epic conclusions. Child phase workers provide evidence for
this landing procedure; they do not execute ancestor closure themselves.
