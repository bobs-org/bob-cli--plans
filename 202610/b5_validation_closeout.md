---
tier: tale
size: medium
title: Finish B5 nested validation integration and land the batch
goal:
  Validate the external checkout's own committed plans, verify the combined B5 work, and
  close bob-cli-dp with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-dp.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dp
create_time: 2026-10-28 14:31:02
status: done
---

- **PARENT:**
  [202610/backlog_batch_b5.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b5.md)
- **BEAD:**
  [bob-cli-dp](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dp/README.md)

# Finish B5 nested validation integration and land the batch

## Scope and verified context

Finish the remaining integration in epic `bob-cli-dp`, whose plan is
`plan:202610/backlog_batch_b5.md`, then close that epic in this coding turn. The epic
has no parent bead. Its four phases are closed, but its original task beads
`bob-cli-dg`, `bob-cli-dh`, `bob-cli-di`, and `bob-cli-dn` remain ready pending combined
acceptance. Read the epic's landing audit note for the completed audit and all five
follow-up dispositions; no fresh follow-up triage of those proposals is needed unless
new evidence changes a disposition.

The lander reviewed every phase note, the actual implementation and tests, and
post-start drift. SASE commits `7531093713`, `e6d44254b0`, and `a5cb6724b2` implement
nested plans provisioning, editable reinstall override forwarding, and continuation
obligation inheritance. Core commit `ed8328f9` fixes the Clippy environment mutations.
At audit time SASE `d12d712e79` and core `ed8328f9` matched their fetched
`origin/master`. Later Footprint export/import, thin snapshot packs, and test splits
were reviewed without finding a conflicting integration.

One integration is still missing. In the opened SASE checkout,
`src/sase/scripts/validate_committed_plans.py::_resolve_committed_plans_root` returns
`SASE_SDD_PLANS_DIR` before considering checkout identity. A Bob agent inherits the Bob
plans directory, so the required SASE `just check` stage scans the host's plans despite
the external checkout now having its own plans sidecar. The lander reproduced this on
clean `d12d712e79` by importing that resolver with `PYTHONPATH=src` and printing its
result from the external checkout.

This tale is limited to fixing that read-context integration, proving the combined B5
acceptances, and completing the landing. No release version changes, memory edits, new
CLI options, or general changes to bead/plan write ownership.

## 1. Open the implementation repositories and review current drift

Use `/sase_repo` to open `sase`, `gh:sase-org/sase-core`, and `plans` from the current
host workspace. Use only the returned paths and read each applicable `AGENTS.md`. Reopen
them in every monitor continuation so the host discovers their commit obligations. Read
SASE's `lint_and_test.md` through `/sase_memory_read` before finishing source changes.

Read `bob-cli-dp`, all four phase beads and their notes, the four original task beads,
and `plan:202610/external_sidecar_validation.md` through audited commands. Check for
newer upstream changes and integrate relevant drift before editing. Keep all
modifications in these opened checkouts.

## 2. Make committed-plan validation use the nested checkout's read context

Reuse the nested repo-open checkout detection and local-record resolution added in
`src/sase/sdd/_link_files.py`. Share or expose a narrow helper when needed; do not
import another module's private symbol or copy the path/marker detection. Keep all
public helpers used so no epic whitelist is needed.

For validation launched inside a repo-open external or linked checkout, including its
subdirectories, resolve that checkout's plans using its own materialized record. An
inherited host `SASE_SDD_PLANS_DIR` must not redirect the scan to the host. The
no-environment case must use the same local read context. If the nested checkout is
unprovisioned, preserve a failing missing-root result with an actionable
`sase repo open` remedy; never silently validate the host's plans or skip the validation
stage.

Outside nested repo-open checkouts, preserve current semantics: explicit plans
environment overrides, primary checkouts, ordinary numbered workspaces, and the owner's
sidecar clones must retain their current behavior. Do not change the default
`resolve_sdd_store` ownership resolver, `get_primary_workspace_dir`, or bead/plan write
paths. Host-scoped writes are an accepted invariant.

Add focused regression coverage using temporary repositories and plans. Prove that a
broken external plan fails even when host plans are valid and named in the inherited
environment, and that broken host plans cannot fail a valid external scan. Cover the
unset environment, nested subdirectory, missing local sidecar, and non-nested behavior.
Assert the selected root and actual validation outcome, not only a mocked helper call.
Reuse the existing nested-sidecar fixtures and committed-plan validator tests where
practical.

## 3. Verify against an explicitly selected local core

Use only `just check`, wrapped as `sase tool run check` in SASE and core. Never run
`just check-full`. Hand long install/build/check commands to `/sase_monitor` with a
concrete continuation, and wait for the monitor-start command itself to exit before
ending the provider turn.

The previous SASE checks ran with published `sase-core-rs==0.42.7` because no local core
was selected. This is not evidence that the override fix failed: B5 explicitly preserves
the published-pin fallback when no source or wheel is requested. Set `SASE_CORE_DIR` to
the opened core path for local installation and verification, using the governed
`install-venv` tool. Verify the installed package version and source stamp/provenance
before interpreting tests. This also exercises the real editable reinstall, beyond the
existing helper tests. If an explicitly selected local build is downgraded, it is
remaining B5 work. Do not bump package versions or published pins to bypass this
verification.

Run the new tests plus these B5 suites through the governed test tool:

- `tests/main/test_repo_open_external_plans_sidecar.py`
- `tests/sdd_store/test_nested_repo_open_sdd_root.py`
- `tests/test_justfile_sase_core_dir.py`
- `tests/llm_provider/test_commit_finalizer_inheritance.py`
- `tests/test_run_agent_runner_bootstrap_baseline.py`
- `tests/test_followup_commit_finalizer_inheritance.py`
- `tests/test_linked_repo_markers.py`
- `tests/llm_provider/test_commit_finalizer_external_repos.py`
- `tests/llm_provider/test_commit_finalizer_baseline.py`
- `tests/llm_provider/test_commit_finalizer_resume_provenance_e2e.py`
- `tests/test_monitor_continuation_finalizer_obligations.py`
- `tests/test_repo_open_log.py`

Run the changed checkout's own editable SASE executable from the host workspace to
reopen/provision `sase`; do not globally reinstall. From the external checkout, run its
`sase validate`, `sase plan links validate`, and committed-plan validation. Confirm both
validators select the external SASE plans while the inherited host environment remains
present. The earlier 5589-file count is evidence, not a fixed assertion: plans may have
been added. Confirm only intended source edits appear in checkout porcelain.

Run the required SASE gate and retain its stage evidence. Core's unchanged `ed8328f9`
already has passing governed check `a1a04aa7d44e17653dcb9387a8718271` (exit 0, 519484
ms). Verify that receipt still covers the current core; rerun the core gate if code or
relevant inputs changed, or if coverage cannot be established. Do not repeatedly rerun
unchanged checks without new evidence.

There is one separately tracked, deterministic pre-B5 Symvision failure: `bob-cli-dt`,
proposed by `bob-cli-dp.3`, for the private `_RecoveryAssessment` import in
`_failed_close_recovery_snapshots.py`. Independent witness
`13ea2a464abef73b3eab4bfee9e624d7` and later ToolRuns classify it KNOWN. Inspect its
latest status before reporting it. Do not suppress it, claim a green gate when it fails,
or fold unrelated cleanup into this tale. A remaining known unrelated failure can be
recorded honestly; every new or unknown failure needs classification, and every
B5-caused failure must be fixed before closeout.

## 4. Close B5 in this turn, then continue the backlog chain

This is the final step. The coder's source commit is host-owned and occurs after its
turn ends. Do not wait for this tale's own SHA, push, or CI before closing beads. Do not
hand the landing back to a nonexistent tale lander.

Record combined acceptance evidence on `bob-cli-dp`: selected plans roots, focused test
and gate results, core provenance, continuation E2E evidence, post-start drift, and the
retained follow-up dispositions. Normally close `bob-cli-dg`, `bob-cli-dh`,
`bob-cli-di`, and `bob-cli-dn` with evidence only when their respective acceptance is
met. If this tale has a separate assigned child bead, normally close that completed
child before the epic. If its assigned bead is `bob-cli-dp`, close it only through the
epic close procedure below.

Run `sase bead epic-symbols bob-cli-dp` before closing. Resolve every listed entry by
wiring it up, privatizing it, adding a policy-compliant non-test pragma, or deleting it.
Re-key only when a specifically identified still-open later bead needs the exemption.
The initial audit found none. Never use `--force` to advance a successful nested landing
or merely to make close succeed.

Close with
`sase bead close bob-cli-dp --note "<combined verification, integration review, and all follow-up dispositions>"`.
If a remaining descendant or symbol blocks close, finish it and retry normally. Then run
`just symvision` where available (use SASE's governed tool path if required);
distinguish the absence of stale whitelist entries from any independently tracked lint
failure.

Set `status: done` in the frontmatter of the epic's plan in the opened `plans` repo at
`202610/backlog_batch_b5.md`. Use audited artifact reads for its content, and include
the plan change in the host-owned commit obligations. Re-read `bob-cli-dp` for its
parent link: the audited parent is null, so no ancestor close is required unless the
link has deliberately changed.

The five proposals are already triaged in the epic landing note: tier-3 provisioning
declined as outside supported scope without a concrete failure; general nested store
ownership declined as intentional; the committed-plan environment leak is fixed by this
tale; the unselected published-core fallback is declined as intentional and replaced
with explicit-core verification; the private import is ready task `bob-cli-dt`. Include
all five outcomes in the close note and update them only if evidence changed.

Finally honor B5's existing successor rule using `/sase_plan`. List all unresolved tasks
with `sase bead list -t task -n 0`, re-read candidates with audited reads, and propose
the next bounded backlog epic of at most five important beads in at most five phases
with unique slugs and explicit dependencies. Prioritize red gates/flakes (including
`bob-cli-dt` if still open), then correctness/data safety, misleading memory/docs, then
features. Check duplicates and active epic ownership. Close only verified fixed tasks or
duplicates with evidence. Do not reopen unrelated completed work based only on a title.

Copy this successor rule into that next epic: repeat bounded, freshly triaged backlog
batches until no unresolved task remains; feature work needs a local yes/no decision,
and memory work needs `/sase_memory_write` with its decisions. Respect live snoozes
(historically ah through November 4, aa through November 6, cp through November 16, and
6q around November 9). Human-dependent Mac acceptance ay/cx, crontab 30, sudo installs
ax, approved write passes cm/co, and running-app UX review 20 need an appropriate
gate/question/decision, never invented evidence. Open external repositories through
`/sase_repo`. Validate the successor with `--explain`, repair, revalidate without
`--explain`, and propose only a valid plan. Wait for proposal exit; its successful
handoff ends the turn. Include all source and plan changes in host completion; if no
successor handoff is possible, use `/sase_final` as the last action of the normal
response.
