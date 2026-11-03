---
tier: tale
title: Fix sase Master Gate marker path-passing audit after the readout split
goal:
  The sase repo's Master Gate test (5) shard passes again because the two read-only
  readout contexts are reviewed in the marker path-passing audit, without weakening the
  audit's safe-callee list.
size: small
proposed_by: bbugyi200.athena.179
proposed_with: claude/opus
create_time: 2026-11-03 09:35:27
status: wip
---

# Fix sase Master Gate: review the split readout contexts in the marker path-passing audit

## Problem

The sase repo's (`sase-org/sase`) `Master Gate` workflow is red on `master` at
`323a91d69f refactor(instructions): split readout.py into focused submodules` (run
41647853290, job `test (5)`, step "Run sharded fast suite"). Every other job (lint,
docs, core-wheel, test shards 1-4 and 6-8) passed. The single failure is deterministic,
not a flake:

```
FAILED tests/test_agent_artifact_marker_path_passing_audit.py::test_tracked_marker_path_passing_sites_are_reviewed
  Extra items in the left set:
  'src/sase/instructions/_readout_classify.py:classify_artifact_dir'
  'src/sase/instructions/_readout_build.py:build_readout'
```

## Root cause

`tests/_agent_artifact_marker_audit_helpers.py` records a "path-passing context" for
every function that passes an expression containing a tracked marker filename
(`agent_meta.json`, `done.json`, ...) into a callee whose name is not in
`_SAFE_PATH_CALLEES`. `tests/test_agent_artifact_marker_path_passing_audit.py` then
asserts that the set of detected contexts equals its `_REVIEWED_PATH_PASSING_CONTEXTS`
dict exactly.

Before the split, `src/sase/instructions/readout.py` read markers through a private
`_read_json(...)` helper, and `_read_json` is on the safe-callee list. So
`build_readout` and `_classify_artifact_dir` were never detected. The split moved that
helper into `src/sase/instructions/_readout_shared.py` and made it public as `read_json`
so the new submodules don't import private names across modules. `read_json` is not
safe-listed, so two call sites are now detected:

- `src/sase/instructions/_readout_build.py:build_readout`, which calls
  `read_json(artifact_dir / "agent_meta.json")`. It also probes
  `artifact_dir / "done.json"` with `.is_file()` only.
- `src/sase/instructions/_readout_classify.py:classify_artifact_dir`, which calls
  `read_json(artifact_dir / "agent_meta.json")` when the caller passed no `meta`.

Neither readout module writes anything. The only file open in them is `open(path, "rb")`
in the tail reader. The split commit updated `tests/instructions/test_readout.py` but
not this audit, and nothing caught it before push.

This was reproduced locally (Python 3.13, AST-only, no venv needed): the detected set
minus the reviewed set is exactly those two contexts, and no reviewed context is
missing.

## Fix

Edit only `tests/test_agent_artifact_marker_path_passing_audit.py` in the sase repo. Add
two `PathPassingReview(exemption=...)` entries to `_REVIEWED_PATH_PASSING_CONTEXTS`. Put
them right after the existing `src/sase/main/instructions_handler.py:_agent_base_facts`
entry, which is the other instructions/readout-family entry. Follow the surrounding
formatting: wrap the value in parentheses when the key line is long, as the
`agents_detail.py` and `facts.py` entries do. Suggested text, adjusted to fit:

- `"src/sase/instructions/_readout_build.py:build_readout"` → `exemption=` "Read-only
  finalization readout: agent_meta.json is read through the readout read_json helper and
  done.json is only probed with is_file() to select and classify recent runs for the
  readout table; no marker is written."
- `"src/sase/instructions/_readout_classify.py:classify_artifact_dir"` → `exemption=`
  "Read-only readout classification: agent_meta.json is read through the readout
  read_json helper only when the caller supplied no meta, to classify one artifact dir's
  declaration and handoff evidence; no marker is written."

Each entry must set exactly one of `lifecycle_coverage` / `exemption`.
`test_reviewed_path_passing_sites_declare_coverage` enforces this.

### Rejected alternative: safe-list `read_json`

Do **not** add `read_json` to `_SAFE_PATH_CALLEES`. Several unrelated modules define
their own public `read_json`: the healer `_healer_common`, `launch_admission_store`,
`axe/state`, `agents_sync`, and the `ArtifactFileCache.read_json` method. The audit
matches callees by name only. A local simulation showed that safe-listing `read_json`
would silently drop 9 existing reviewed contexts. Among them is
`healer_relaunch.py:_write_evidence`, which writes a copy of done.json. That would
weaken the audit. Per-site review is also this test's established pattern after module
splits (see commit `82cf081fc4`, which retargeted a reviewed key after a split). Do not
rename the helper to a safe-listed name just to get past the audit, and do not change
the readout source modules.

## Steps

1. From your workspace, open the sase repo with your `/sase_repo` skill
   (`sase repo open sase -r "Fix Master Gate marker path-passing audit after readout split"`).
   Work only in the printed path. Read that checkout's `AGENTS.md`. Confirm `HEAD` is at
   or after `323a91d69f` and that the two contexts are still unreviewed. If `master` has
   moved on and someone already fixed the audit, stop and report that instead of
   duplicating the fix.
2. Add the two reviewed entries described above.
3. Verify in the sase checkout:
   - Run the focused tests: `tests/test_agent_artifact_marker_path_passing_audit.py` and
     `tests/instructions/test_readout.py`. Use the checkout's venv; run
     `just install-venv` first if `.venv` is missing. That step can be long, so route it
     through `/sase_monitor` if it won't fit inline. Both must pass.
   - Read the sase repo's `lint_and_test.md` reference memory with
     `sase memory read lint_and_test.md` from the sase checkout, then run `just fmt` and
     `sase tool run check` as it directs. Do not run `just check-full`. This prompt does
     not request it.
4. Finish through `/sase_final`. The opened sase checkout is a repository obligation and
   needs a `commit` decision. Suggested message:
   `test(audit): review split readout read_json marker contexts`.

## Acceptance

- `tests/test_agent_artifact_marker_path_passing_audit.py` passes. The detected
  path-passing set equals the reviewed set, and every entry declares exactly one
  coverage kind.
- `_SAFE_PATH_CALLEES` and the `src/sase/instructions/` modules are unchanged.
- `sase tool run check` passes in the sase checkout, so the next `Master Gate` run on
  `master` should be green for `test (5)`.
