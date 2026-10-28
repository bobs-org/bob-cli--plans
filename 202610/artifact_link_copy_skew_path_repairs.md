---
tier: epic
title: Repair same-payload ancestry and remaining quarantine path gaps
goal: Preserve earliest provable history across skewed copies and reject unsafe quarantine
  paths before reads or writes are accepted.
parent_bead: bob-cli-df.2.1.4.3
phases:
- id: same-payload-ancestry
  title: Preserve ancestral provenance across skewed identical copies
  depends_on: []
  description: 'same-payload-ancestry: repair surviving-history selection and prove
    the public partition and binding behavior.'
  size: medium
- id: quarantine-paths
  title: Pin the core repair and reject every unsafe quarantine path
  description: 'quarantine-paths: consume the landed core repair, reject malformed
    operation paths and unsafe staging, and verify integrated recovery.'
  size: medium
  depends_on:
  - same-payload-ancestry
proposed_by: bbugyi200.athena.bob-cli-df.2.1.4.3.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-28 07:11:11
status: done
bead_id: bob-cli-df.2.1.4.3.3
---

- **PARENT:** [202610/artifact_link_provenance_closeout.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/artifact_link_provenance_closeout.md)
- **BEAD:** [bob-cli-df.2.1.4.3.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.2.1.4.3.3.md)

# Remaining work for bob-cli-df.2.1.4.3

This is remaining epic work. Do not redo completed collision, cross-root ancestry,
import-cleanup, or recovery work. The current epic and directly parented plans
**bob-cli-df.2.1.4** and **bob-cli-df.2.1** remain open. Their original coding phases
are closed. Read their notes and plans through audited commands.

Two sequential phases are required because the Python consumer must pin the Rust
repair's actual committed SHA and verify its wheel. A tale cannot wait for its own
future commit, and externally opened repos have no assumed automatic pin update. There
is no parent-closeout coding phase; `parent_bead: bob-cli-df.2.1.4.3` returns this
child's lander to the interrupted landing.

## Context and constraints

Read:

- `sase bead read bob-cli-df.2.1.4.3 -r "Need landing findings and follow-up dispositions"`
- `sase artifact read file:explicit:65cfd4bea965ad1e3eef0b63 "Need detailed landing audit, verified scope, limitations and dispositions"`
- `sase artifact read plan:202610/artifact_link_provenance_closeout.md "Need governing provenance and path contract"`
- `sase artifact read plan:202610/artifact_link_quarantine_landing_repairs.md "Need parent ordering and immutable replay contract"`
- `sase artifact read plan:202610/artifact_link_collision_quarantine.md "Need original selection and retention contract"`
- `sase artifact read file:explicit:2e6b68e6e526d103d9c39738 "Need exact copy-skew and actual Git-history reproduction"`
- `sase artifact read file:explicit:0bb00b945d1059bc24e04ae8 "Need operation-path and staging IO reproductions"`

There are no discretionary DECISIONS or authorized memory changes. Keep event and
collision/quarantine wire schema 1, registered Python binding names, strict low-level
reduction, Rust-owned winner policy, and Python-owned filesystem/Git IO. No release
versions, changelogs, published dependency windows, real outage data, memory, or
unrelated CLI options change.

Open source repos through `/sase_repo`, use only printed paths, and read AGENTS.md:

- `sase repo open gh:sase-org/sase-core -r "<reason>"`: clean audited tip
  **00fe29d5436cb391f78c4bfb1df2214dfa0e09e5**.
- `sase repo open sase -r "<reason>"`: clean audited tip
  **49ebcc5eba99d8ac30ce4072fa6e43f30b20b00a**, pinning that exact core SHA.

Fresh fetch during this audit found both tips equal to origin/master. No PR branch is in
use. Post-start non-epic **95bd848d26** adds Footprint derived-source folding and
history backfill, with no competing artifact-link reduction path. Its author timestamp
precedes this epic but its commit time is 04:55 EDT. Earlier Footprint and recovery
drift remains unrelated. Preserve sibling **1b1162b313** plan-inlet rollback for other
publication failures and its formatting/handler regressions. Recheck drift.

The landing probes are bounded evidence, not full feature tests: Rust selection
functions were extracted unmodified with minimal structs/integer instants; the actual
Python history adapter used synthetic Git files; actual quarantine IO used stubbed Rust
validation/path bindings. Add regressions using real public APIs and records. The
lander's external Python checkout has no venv and exact Python 3.12 wheel-cache lookup
missed. No fresh check pass or global binding use is implied.

## Phase same-payload-ancestry

Repair `crates/sase_core/src/artifact_link/collision/mod.rs`. The completed
`representative_entry` calls `earliest_provable_history`, which correctly excludes
same-payload descendants before comparing independent times. But `select_winner` now
uses `best_surviving_instant`; its `occurrence_is_eliminated` ignores entries with the
same digest. Consequently it reintroduces later same-payload histories and lets a skewed
clock on a later copy replace that payload's earliest provable history.

Reproduction: A first appears at a0, 12:00; identical A appears at a1, 10:00, with a0 an
ancestor of a1; independent B appears at b0, 11:00. A's representative first-add is
correctly 12:00, but final selection chooses A via a1 at 10:00. Expected B. The actual
Python adapter supplies this evidence for two store roots at separate subdirectories of
one full Git repo, plus an independent B repo.

Apply occurrence-level ancestry consistently, including ancestors of the same payload,
so only genuinely independent surviving histories participate in time ordering. Keep
ancestry evidence needed to reject competitor descendants; do not erase the completed
opposite-ancestry behavior. Ensure winner, group basis, per-rejection detail, and
ambiguity all describe the actual selection over the same surviving evidence. Review the
runner-up/group-basis path while making this bounded repair; it currently orders by the
representative evidence rather than the new surviving-history comparator. Fix an
inconsistency only if regression evidence demonstrates it, without introducing a second
policy or broad rewrite.

Add public `partition_artifact_link_collision_events` regression tests for this
same-payload skew case, its permutations/root labels, equal/offset instants,
multi-generation identical copies, and a three-payload case. Preserve every existing
competitor-descendant-copy test, the case where a later identical A copy is an ancestor
of B, tier/evidence/admission precedence, missing evidence, exact retries, malformed
evidence and strict reducer rejection. Extend the real PyO3 round-trip. Use module-path
imports; do not restore removed lib/prelude facade additions.

Run `just test -p sase_core artifact_link`,
`just test -p sase_core_py artifact_link_collision_bindings_round_trip`, and governed
`sase tool run check` (the `just check` gate). Never run bare cargo or
`just check-full`. Use `/sase_monitor` for long commands and wait for its start command
to exit. Fix epic-caused failures, record independent baseline failures precisely,
resolve/re-key the assigned phase's epic symbols, and close only that phase. Host
finalization creates the SHA consumed by phase two; no manual commit.

## Phase quarantine-paths

Open both repos again, inspect the actually landed Rust phase, and update SASE's
`sase-core-revision.txt` to that SHA or a later containing commit. Follow
`docs/contribute/rust-backend.md` and verify the extension loaded in an isolated
checkout venv comes from that source. Use explicit valid `SASE_CORE_DIR` or a verified
exact wheel to avoid independent installer issue **bob-cli-di**; verify again after
editable install. Do not install globally or repair the unrelated installer in this
phase.

### Complete quarantine path validation

In `_artifact_link_event_quarantine.py`:

1. `_scan_quarantine_tree` validates `dirnames` but skips non-`.json` entries in
   `filenames`. An operation directory replaced with a plain file, or a broken symlink
   at the operation path, is therefore ignored. The actual loader returns zero findings
   and zero errors. Recognize invalid directory-level entries before applying record
   filename filtering. Preserve intentional staging temporary-file handling without
   letting staging or operation directory corruption disappear.
2. `persist_artifact_link_quarantine_record` validates its final record path but never
   checks `_QUARANTINE_STAGING` before calling the immutable installer. An existing
   `.staging` symlink to an outside directory lets `os.open` create and write the
   temporary record there, then hard-link it into the store and return success. Validate
   all quarantine directories used by first persistence/reuse before side effects.
   Reject symlink, broken-symlink, and non-directory staging paths with typed
   corruption. Keep the fix scoped; reuse safe IO conventions without a general artifact
   installer redesign.

Use valid records and real bindings for tests. Cover namespace, operation-directory,
leaf and staging paths; plain files where directories are required; live and broken
symlinks; redirects outside the canonical supplied root; and legitimate symlinked
ancestors of that supplied root (macOS `/tmp`). Read-only load, snapshot, and doctor
must report blocking corruption without mutating files/Git status. Persistence must make
no outside write, including temporary writes: observing only the final outside directory
contents is insufficient because the existing installer removes its temp. Instrument
creation or a write boundary in the test to detect a transient escape.

Preserve validated immutable record reuse across basis/locator changes, original
bytes/provenance, project/operation/digest/path checks, and rejection of incompatible
accepted/rejected identities. Verify an unsafe path cannot produce a quarantine
settlement receipt or drop a loser from the outbox; after safe repair of the fixture,
retry and restart must settle exactly once without losing another payload or multiplying
observations. Keep all completed empty-store restart, direct/outbox crossover,
ownerless/document, malformed-line, concurrent append/publication and
quarantine-write-failure coverage.

### Verify the integrated core and consumer

Add a real store/partition regression for the new copy-skew case using full Git history:
two store roots in separate directories of one repo, with A added at 12:00 then copied
at a descendant commit whose clock is 10:00, plus an independent B root at 11:00. Assert
that ancestry is gathered and B wins, that source bytes remain unchanged, and that exact
copies do not multiply observations. Keep Rust authoritative.

Run relevant history/quarantine IO/replay-evidence, store/core/health/projection/
reconcile, publisher, outbox drain/IO/parked, CLI add/doctor, binding-validation and
`tests/test_plan_command_handler_formatting.py` suites against the verified wheel.
Retain the temporary-store CLI acceptance where historical collision files remain
byte-identical, unrelated link add works, and doctor names both payloads. Read
`lint_and_test.md` with the memory skill, run `just fix`, then governed
`sase tool run check`. Use `/sase_monitor` for long setup/checks; no `just check-full`.
Fix epic-caused regressions; classify unrelated failures only with clean-base
reproduction or a valid independent witness. Resolve/re-key this phase's symbols, record
the exact source/wheel and check results, and close only this phase.

## Follow-up triage already completed

The current lander used `/sase_new_task`; outcomes are on **bob-cli-df.2.1.4.3**. Do not
duplicate them:

- Proposing **bob-cli-df.2.1.4.3.1 #1**: +1 on **bob-cli-dh** for the phase's report of
  27 pre-existing Clippy errors; original clean-base witness is 83b010dd.
- **bob-cli-df.2.1.4.3.2 #3**: +1 on **bob-cli-dg** for reported clean-HEAD missing
  plans sidecar, run c22e8134aaf51624cc76b121e7e284db.
- **bob-cli-df.2.1.4.3.2 #4**: corroborating DISCOVERED ISSUE on causal B4
  **bob-cli-df**, whose sibling recovery owns the private `_RecoveryAssessment` import;
  KNOWN witness 13ea2a464abef73b3eab4bfee9e624d7.
- No new proposal was declined. Prior stale Footprint exemption proposal remains
  declined as fixed by c80a6c791a, and installer override work remains **bob-cli-di**.

Workers record new unrelated discoveries as PROPOSED FOLLOW-UP notes on their own
phases. Defects caused by this epic remain its work.

## Land-agent handoff, not a coding phase

This child's lander verifies its own children/notes/DECISIONS, actual commits,
pin/wheel, current source, fresh drift and readiness. It resolves every own
`sase bead epic-symbols` entry, closes this child normally, runs `just symvision` where
available, and marks its plan done. Then follow `parent_bead` to resume
**bob-cli-df.2.1.4.3**; no parent closure depends on the lander's future commit.

For each plan in this order, reread every descendant and note, previous landing
findings, linked plan, and post-child drift; rerun descendant and linked-plan readiness
checks. Only when fully complete, resolve every listed symbol (or re-key only to a
still-open later bead that genuinely needs it), close normally, run `just symvision`
where available, and use `/sase_repo` to open plans before marking its frontmatter
`status: done`:

1. `sase bead epic-symbols bob-cli-df.2.1.4.3`, then
   `sase bead close bob-cli-df.2.1.4.3 --note "<verified source/commits, repaired provenance and path cases, source/wheel checks, integration and all follow-up outcomes>"`;
   mark `plan:202610/artifact_link_provenance_closeout.md` done.
2. `sase bead epic-symbols bob-cli-df.2.1.4`, then
   `sase bead close bob-cli-df.2.1.4 --note "<rechecked ancestor ordering, replay, descendants, linked-plan readiness, drift and dispositions>"`;
   mark `plan:202610/artifact_link_quarantine_landing_repairs.md` done.
3. `sase bead epic-symbols bob-cli-df.2.1`, then
   `sase bead close bob-cli-df.2.1 --note "<rechecked original retention, canonical selection, unrelated writes and doctor contracts plus all repairs>"`;
   mark `plan:202610/artifact_link_collision_quarantine.md` done.

Finally read phase **bob-cli-df.2**, verify these child plans complete its scope,
resolve/re-key `sase bead epic-symbols bob-cli-df.2`, and close **only that phase** with
`sase bead close bob-cli-df.2 --note "<verified child plans, canonical original, quarantine retention, unrelated append and doctor evidence>"`.
Leave containing **bob-cli-df**, its B4 plan, and original **bob-cli-5m** to its
already-waiting land agent. At the first incomplete or ambiguous ancestor, stop, append
a concrete blocker note and report it. Never use `--force` merely to make a close
succeed or to advance a successful nested landing; finish or reopen required phases
instead. Do not leave stale symbol judgment to another agent.
