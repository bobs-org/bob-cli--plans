---
tier: tale
title: Preserve opened repository obligations across monitor continuations
goal:
  A continuation discovers and commits its predecessor's external repository edits with
  attributable repo-open evidence and the original protection baseline.
size: medium
proposed_by: bbugyi200.athena.bob-cli-dp.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-dp.4
create_time: 2026-10-28 11:35:47
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b5.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b5.md)
- **BEAD:**
  [bob-cli-dp.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dp/bob-cli-dp.4.md)

# Preserve opened repository obligations across monitor continuations

## Scope and completion authority

Implement phase `bob-cli-dp.4` of `plan:202610/backlog_batch_b5.md`, addressing
`bob-cli-dn`. This is one bounded host lifecycle repair in the external `sase`
repository, plus focused regression tests. The phase has no earlier implementation notes
or recorded unfinished attempt. One coding agent can complete this tale.

Read the phase and its current notes again before implementing. Do not manually set its
status, reopen it without evidence of unfinished work, or redo a fix that has landed
since this plan was written. The epic has no optional decisions and authorizes no memory
changes. Leave memory files, generated skills, global installs, the Rust wire contract,
and unrelated workspace recovery behavior outside this change.

Open `sase` through `/sase_repo` from the host project and use only the returned path.
Read its `AGENTS.md` and required memory. Reopen every edited external repository on
every actual continuation, including a continuation after verification, until the host
fix is deployed. This operational workaround must not be used inside the regression to
hide the defect. Include every edited repository in `/sase_final`.

Only close `bob-cli-dp.4` once this phase is verified. Do not close `bob-cli-dp`, any
ancestor plan bead, or `bob-cli-dn`; the outer lander owns task and epic closure. If
SASE associates this tale with a separate plan bead, follow its normal completion flow
without expanding this closure authority. Record discovered unrelated work as
`PROPOSED FOLLOW-UP:` notes on `bob-cli-dp.4`, never create task beads.

## Evidence and root cause

Audited context:

- `bead:bob-cli-dn` records external edits stranded after a monitor handoff. Its
  original implementation diff was later rescued and committed; this plan fixes the
  still-separate obligation-discovery defect, not that recovered implementation.
- `file:explicit:e79819a22e42975e194405f7` shows the continuation baseline containing
  two `opened_repo` repositories while `final_context.json` has no obligations or
  repository evidence, `final_context_host.json` has no repositories, and
  `commit_results.json` contains only beads-sidecar commits.
- `file:explicit:ab5d6c3294bc3abbcfb7bf7b` preserves the stranded diff. Read evidence
  through `sase artifact read`, never by opening its historical workspace paths.
- `bead:bob-cli-cv` is closed with independent dirty-tree preservation evidence. Do not
  reopen or reimplement that adjacent recovery fix.

Source inspection at SASE `3d52c6ce74` finds:

- `src/sase/monitor/followup.py` launches the successor through an agent-session attach
  plan naming its predecessor's artifacts and workspace.
- `src/sase/axe/run_agent_runner_bootstrap.py` copies only the canonical or legacy
  baseline in `_inherit_parent_commit_finalizer_baseline`.
- `src/sase/axe/run_agent_helpers_artifacts.py` similarly seeds only the baseline for
  in-process followups in `_seed_followup_commit_finalizer_baseline`.
- `src/sase/_linked_repo_markers.py` stores discovery separately in
  `opened_linked_workspaces.json`, with legacy `opened_siblings.json` support.
  `opened_repo_records` reads these markers from the current run's artifacts.
- `src/sase/llm_provider/commit_finalizer_state/` and
  `src/sase/finalizers/_prepare_observe.py` discover external repositories from those
  markers. Merely inheriting `finalizer_baseline.json` cannot make them visible.
- `src/sase/repo_open_log.py` provides attributable audit events, independently of
  discovery. Copying markers alone would repair discovery but leave continuation audit
  attribution incomplete; writing audit rows alone would not fix discovery.

This repair is host filesystem, identity, and launch wiring, which can remain in Python
under SASE's Rust boundary. Reuse existing marker readers, baseline loaders, audit event
builders, and workspace validation rather than introducing a second repository inventory
or a new core domain.

## Implementation

1. Add a failing regression using disposable real Git repositories. Open and
   marker-record an external repository under a predecessor's artifacts, capture its
   initial baseline, then make tracked and untracked edits. Exercise the actual
   monitor/session-attach inheritance path into a fresh continuation artifacts
   directory. Without explicitly reopening the repository in the child, demonstrate the
   current omission from dirty-state collection and final declaration context.

2. Add a shared host helper to inherit opened-repository responsibility together with
   the baseline. Wire the session-attach launch and in-process followup paths through
   it, at a point where the destination artifacts, successor identity, and bound
   workspace are known. Preserve canonical names, repository kind, provider ref,
   normalized path, original open metadata, and baseline-error diagnostics. Merge with
   existing destination markers instead of overwriting newly opened repos. Use the
   existing legacy marker reader. A valid canonical `opened_repo` baseline can recover a
   missing marker for an already-inherited external repository; do not treat every
   baseline entry or every checkout on disk as an opened repository.

3. Preserve the original baseline's HEAD, fingerprints, scope, and capture time. Never
   recapture a dirty predecessor checkout as a fresh protected baseline, clear
   pre-existing protection, or replace the baseline during an idempotent retry. Keep the
   existing first-open and normalized-path deduplication rules. Preserve genuine foreign
   dirt while making changes performed after the predecessor's open eligible for the
   continuation's commit.

4. Record the inherited open responsibility under the continuation's own identity and
   artifacts directory through the existing repo-open audit API. Use an explicit reason
   naming the predecessor and inheritance; retain the predecessor's events. Do not
   resolve identity from stale ambient predecessor environment during in-process
   followups. Repeated initialization of one continuation must not produce duplicate
   inherited entries; a later continuation must get its own attributable record. Keep
   audit bookkeeping separate from baseline capture.

5. Respect workspace ownership. The ordinary same-workspace monitor handoff is the
   acceptance case. If a continuation is bound to a different checkout, use existing
   workspace resolution/validation and report an unresolved obligation where needed;
   never silently import a predecessor's old path as a writable child repository, scan
   another workspace, or turn its dirty contents into protected foreign work. Keep
   missing or malformed optional metadata handling compatible with existing readers,
   with useful diagnostics when known responsibility cannot be transferred.

6. Test a second handoff and repeated initialization, linked-repo/legacy-marker
   compatibility, merging an existing child marker, clean repositories, preserved
   pre-existing dirty paths, and workspace mismatch isolation. Extend existing test
   modules where natural; split new coverage into focused files rather than growing
   already large runner modules unnecessarily.

## Verification and acceptance

Use the established real-Git controller fixtures in
`tests/finalizers_live_e2e_test_helpers.py` for end-to-end coverage. Drive the real
monitor followup/session-attach wiring and final declaration lifecycle; substitute only
process/provider boundaries so no real agent launch or remote service is required. The
child must not manually call `sase repo open` before the assertions. Do not settle for
tests that only assert a helper copied a JSON file.

The acceptance test must prove all of these:

1. The continuation's inherited baseline retains the external `opened_repo` entry and
   original protection evidence.
2. Child discovery and `final_context.json` contain the external repository's commit
   obligation and repository evidence, and `final_context_host.json` names the correct
   checkout. Audit filtering by the child identity returns its inherited open.
3. A valid continuation declaration runs the host finalizer and writes a real commit
   containing the predecessor's tracked and untracked changes; commit results name the
   external repository. Verify the commit's bytes and clean owned paths, with
   pre-existing protected dirt excluded. Use disposable local bare remotes where the
   fixture requires publication, never a production repository for the experiment.
4. Repeated startup and a second continuation retain responsibility without duplicate
   obligations or changed protection; a clean repo does not spuriously require a commit,
   and an unrelated workspace is not adopted.

Relevant existing suites include:

- `tests/test_run_agent_runner_bootstrap_baseline.py`
- `tests/test_run_agent_workspace_identity.py`
- `tests/test_axe_run_agent_exec_plan_followup_artifacts.py`
- `tests/llm_provider/test_commit_finalizer_external_repos.py`
- marker and repo-open audit tests selected from the touched imports
- the new monitor-to-finalizer acceptance test and affected finalizer tests

Before editing or verifying, read `lint_and_test.md` in the opened SASE checkout. Run
focused tests with that checkout's supported environment and then `sase tool run check`
there. Do not run `check-full`. Read `/sase_monitor` before starting a long setup or
check; format first and include an explicit reminder to reopen external repos in its
continuation. Never globally reinstall SASE to run this test. Record exact revisions,
commands, ToolRun IDs, outcomes, and the distinguishing context/baseline/commit-results
evidence on `bob-cli-dp.4`.

If a gate failure reproduces identically on the clean base, record that independent
evidence as a `PROPOSED FOLLOW-UP:` note, citing an existing tracking task if present,
and close the completed phase anyway. NEW/UNKNOWN failures require investigation, not an
unsupported claim that the gate passed. Keep external-sidecar validation and
editable-install repairs owned by sibling phases `bob-cli-dp.1` and `bob-cli-dp.3`.

Immediately before closing, run `sase bead epic-symbols bob-cli-dp.4`; there were no
entries at planning time, but recheck. Resolve each remaining symbol or re-key its
Justfile line to a still-open owner. Close only this phase with
`sase bead close bob-cli-dp.4 --note "<concrete verification evidence>"`. Finish with
the host-owned `/sase_final` declaration, committing the edited external SASE repo.
