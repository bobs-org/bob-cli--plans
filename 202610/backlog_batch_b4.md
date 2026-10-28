---
tier: epic
title: 'Backlog batch B4: catalog count, artifact-link quarantine, propose inlet,
  and dirty-tree preservation'
goal: Resolve bob-cli-de, bob-cli-5m, bob-cli-5n, and the remaining bob-cli-cv dirty-tree
  loss with reproducible verification, then continue the bounded backlog chain.
phases:
- id: catalog-protocol-count
  title: Align the protocol-1 catalog CLI count with the themes-split row
  size: small
  depends_on: []
  description: 'catalog-protocol-count: resolve bob-cli-de in bob-cli by updating
    the protocol-1 CLI catalog assertion from 54 to the intentional 55-entry shape,
    covering the themes-split row, and verifying with the focused catalog CLI suite
    plus just check.'
- id: artifact-link-quarantine
  title: Quarantine colliding artifact-link operation IDs without blocking later writes
  size: large
  depends_on: []
  description: 'artifact-link-quarantine: resolve bob-cli-5m in sase so one reused
    operation_id with two payloads is quarantined and surfaced by sase artifact doctor
    while unrelated link appends still succeed.'
- id: plan-propose-inlet
  title: Keep a failed links inlet from consuming the scratch plan
  size: medium
  depends_on: []
  description: 'plan-propose-inlet: resolve bob-cli-5n in sase by validating or publishing
    the plan links inlet before move_plan_to_sase, or rolling that move back on inlet
    failure, so a failed propose leaves the scratch plan in place with a clear error.'
- id: dirty-tree-preservation
  title: Preserve tracked and untracked edits across a failed-close relaunch reset
  size: large
  depends_on: []
  description: 'dirty-tree-preservation: resolve the remaining bob-cli-cv loss in
    sase by proving and repairing the relaunch reset that still drops tracked edits
    after sase-1oh reopened failed closes; stash or otherwise preserve the full dirty
    tree before any hard reset.'
proposed_by: bbugyi200.athena.bob-cli-dd.land--3
proposed_with: grok/grok-4.6
create_time: 2026-10-27 21:24:43
status: done
bead_id: bob-cli-df
---

- **PROMPT:** [prompts/202610/backlog_batch_b4.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_b4.md)
- **BEAD:** [bob-cli-df](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/README.md)

# Backlog batch B4

## Context and triage

This is the required successor to `plan:202610/backlog_batch_b3.md`. B3 epic
`bob-cli-dd` is closed, its plan is done, and its three original tasks d4, d9, and dc
are closed with evidence. This successor is independent backlog work, not remaining B3
work; do not reopen B3 or attach a `parent_bead` to it.

The B3 lander listed all 101 unresolved task beads on 2026-10-28 after closing dc and
dd, including the four snoozed tasks. It audited de, 5m, 5n, cv, and 8n rather than
relying on the prior batch's deferred-SASE note as proof. No additional obsolete or
duplicate task was established, and none was closed on speculation. Scope here is the
red `just check` catalog assertion, then three confirmed SASE correctness bugs. Four
independent work units in two repositories justify an epic. There is no feature or
memory work and no discretionary product decision.

- **de:** at bob-cli `0421192684d525f556f44aa35a6e5ff61ef3660b` plus the B3
  remaining-work rename repair, `just check` (monitor `m97qqcmn090r`, tool
  `ba90bfaa3a6cdb825202411ebcff45c1`) failed only
  `catalog::protocol_1_parses_with_schema_version_one` at `tests/cli/catalog.rs:42`
  (`left: 55`, `right: 54`; CLI target 2318 passed / 1 failed). Closed db commit
  `0a896a45` added `themes-split` to `src/native/catalog/table.rs` and updated the lib
  count to 55; the CLI handshake assertion is still pinned at 54. This is a bounded
  stale test expectation, not a protocol change.
- **5m:** at sase `daf54c5b2d39ed483df935fbf98b209b47e1f276`,
  `_artifact_link_event_install.py:121` and `_artifact_link_event_publish.py:607` still
  raise on reused `operation_id` bytes, and `_artifact_link_event_store.py:98`
  `assert_healthy` still turns any such pair into a fatal store for every reader and
  writer. There is still no quarantine path. Related closed bob-cli-21 was a hand repair
  of data only.
- **5n:** at the same sase tip, `plan_propose_handler.py` still calls
  `move_plan_to_sase` before `publish_plan_artifact_link_inlet`. An inlet publication
  failure still consumes the scratch plan with no approval gate and no handoff. Sibling
  of 5m, not a duplicate: any inlet failure (invalid ref, lock timeout, or a future
  store problem) takes this path even after 5m quarantines colliding IDs.
- **cv:** the first requested fix landed in closed sase-yo / sase-1oh.4 (reopen a bead
  whose host commit never lands, with `CompletionFailureRecovered` and a `STITCH FAILED`
  note). Follow-ups sase-1on and sase-1oo are suite flakes. The remaining requested fix
  still stands: a failed-close relaunch reset discarded tracked CLI/docs wiring while a
  later stash kept only leftover untracked files (reset 20:33, stash 20:34 on
  2026-10-26). Current source still has `prepare_workspace` clean-then-checkout,
  `vcs_stash_and_clean` as `git stash push --include-untracked` only, and
  `reset_replay._reset_leased_checkout` doing `git reset --hard` to upstream. No
  in-progress sase epic owns that remaining loss (sase-1nv is publication/gc; sase-1om
  is capacity UI; sase-1oj is closed disk-write diet). Keep cv open; do not close it as
  done against sase-1oh.

Deferred scope: snoozed ah (wake 2026-11-04), aa (2026-11-06), cp (2026-11-16), and 6q
(still snoozed; four-week stats journal incomplete around 2026-11-09). Human-gated Mac
acceptance ay/cx, crontab 30, sudo Prometheus ax, approved write passes cm/co, and
running-app UX review 20 stay gated. Vault AGENTS.md 8n is a small docs lie in `~/bob`
and ranks after these red-gate and data-loss bugs. Docs/memory tasks and features rank
after this batch.

## Shared work rules

Read the assigned phase and original task with `sase bead read ... -r ...`. Open every
linked or external repo through `/sase_repo`, use only its returned path, and read its
AGENTS.md. Do not embed numbered workspace paths in plans.

Use `/sase_monitor` for long checks and CI waits. File-change verification is
`just check` in the repo that changed; do not run `just check-full`. Recheck unrelated
failures, record them, and do not silently fix or duplicate another epic's scope. Any
regression caused by this batch must be finished here. SASE shared backend behavior
belongs in sase-core; follow that checkout's rust-core boundary before reimplementing
store logic in Python.

Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
phase rather than creating tasks. Before closing a phase, resolve or re-key its
`sase bead epic-symbols <phase-id>` entries. Record commands, results, skips, and
relevant commits. The lander closes the four original task beads after reviewing their
acceptance evidence; each worker closes only its assigned phase.

## Phase catalog-protocol-count

Read bob-cli-de and confirm `src/native/catalog/table.rs` still lists `themes-split` as
the 55th public leaf. Update
`tests/cli/catalog.rs::protocol_1_parses_with_schema_version_one` from 54 to 55. Assert
the protocol-1 handshake still reports schema_version 1, ok, protocol 1, and the
existing features list. Cover that the catalog entries include the themes-split leaf
rather than only bumping the count. Do not change protocol versions, catalog generation,
or CLI help.

Run `cargo test --test cli catalog` and `just check`. The B3 just-check failure is this
node only; if a new failure appears, finish any caused by this phase and leave genuine
unrelated failures recorded rather than silently expanding scope.

## Phase artifact-link-quarantine

Use `/sase_repo` to open sase. Read bob-cli-5m, related bob-cli-21, and sibling 5n.
Trace `load_artifact_rows` / `assert_healthy`, `_reject_reused_operation_ids` in
`_artifact_link_event_install.py`, and the matching publish-time check. Keep the
earliest durable object as the canonical payload and quarantine the colliding replay
rather than deleting history by hand.

A store that already holds one colliding pair must still accept an unrelated
`sase artifact link add`. `sase artifact doctor` must surface the quarantined pair.
Whole-store validation must stop treating that pair as fatal for later unrelated writes.
Add regression coverage for install and publish paths. Do not redo the historical
bob-cli-21 hand deletion of replay objects as the product fix.

## Phase plan-propose-inlet

Open sase and read bob-cli-5n. In `src/sase/main/plan_propose_handler.py`,
`move_plan_to_sase` currently consumes the scratch file before
`publish_plan_artifact_link_inlet`. Validate or publish the `links:` inlet first, or
roll the archive move back when inlet publication fails, so the scratch plan remains at
the caller's path with a clear error and no approval gate is implied.

Cover at least one inlet-publication failure (invalid ref, injected store error, or lock
timeout) and prove a successful inlet still archives and hands off. Do not require 5m to
land first; 5n is about propose ordering for any inlet failure. Do not leave a half-done
archived plan without a gate.

## Phase dirty-tree-preservation

Open sase and read bob-cli-cv, including its 2026-10-28 triage note, closed sase-yo, and
closed sase-1oh.4. Do not reopen or reimplement failed-close bead recovery. Prove
whether the remaining tracked/untracked dirty-tree loss still reproduces on a
failed-close relaunch: a workspace with both modified tracked files and new untracked
files must retain both across `prepare_workspace` / `reset_replay` /
`run_sase_hg_clean`. The historical failure was a hard reset followed by a stash of
leftover untracked files.

Preserve the full dirty tree before any hard reset or clean, including untracked files,
and keep a durable locator for that snapshot in the failure path so relaunch does not
depend on reading a partial stash. Add a regression that would have caught the ct.3 loss
(tracked wiring plus untracked sources). Leave sase-1on/sase-1oo as unrelated flakes.

## Landing and successor rule

Review all phases/notes, their source and commits, and changes landed since this batch
began in all touched repositories. Finish epic-caused gaps before closing. Verify de's
focused catalog CLI suite and `just check`, and 5m/5n/cv's focused and repository gates
in the sase checkout. For genuine unrelated proposals use `/sase_new_task`, record every
disposition and proposing bead, and avoid duplicating active epic work. Close de, 5m,
5n, and cv normally with evidence only once each acceptance is met. Never force a
successful close. cv may close only when the remaining dirty-tree preservation is
proven; the sase-1oh failed-close reopen is not sufficient.

Before closing this epic, resolve or re-key every entry from
`sase bead epic-symbols <this-epic>`, close with a detailed verification note, run
`just symvision` if available, and set its plan frontmatter `status: done`. Follow any
parent_bead link under the normal nested-landing rules; this successor was authored
without one.

As the lander's final action, use `/sase_plan` to author and propose the next backlog
epic, and copy this successor rule into it. Continue until no unresolved task beads
remain:

1. List every unresolved task with `sase bead list -t task -n 0`; re-read candidates
   with audited reads. Close only verified duplicates (`superseded`) or work already
   fixed (`done`), with evidence. Recheck this triage instead of trusting it.
2. Choose at most five important remaining beads in at most five phases, with unique
   slugs and explicit depends_on. Rank red gates/flakes, then correctness and data
   safety, misleading memory/docs, then features. Open external repos with `/sase_repo`
   before source investigation, including sase for remaining runner defects and the
   vault for 8n.
3. Feature work gets a local yes/no decision. Memory changes require
   `/sase_memory_write` and its memory decisions; no unsupported permission assumption.
   Keep other decisions to a minimum.
4. Skip snoozed tasks until wake: ah 2026-11-04, aa 2026-11-06, cp 2026-11-16, and 6q
   (still snoozed; journal completeness around 2026-11-09; use live snooze state).
   Human-dependent Mac acceptance ay/cx, crontab 30, sudo installs ax, approved write
   passes cm/co, and running-app UX review 20 need a gate, question, or decision, never
   invented evidence.
5. Validate with --explain, repair, revalidate without --explain, and propose the
   validated plan. Wait for the proposal command itself to exit; its successful handoff
   ends the turn.
