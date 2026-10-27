---
tier: tale
title: Verify uncommitted changes on macOS CI before landing
goal:
  Give bob-cli and bob-mac-capture a bounded precommit CI check that preserves the
  checkout and removes its scratch branch.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1
create_time: 2026-10-27 06:35:02
status: wip
---

- **PARENT:**
  [202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1.md)

# Verify uncommitted changes on macOS CI before landing

Implement the approved `macos_gate = scratch_ci` decision for phase
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1`, addressing task `bob-cli-by`. The parent
design is
`plan:202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md`.
Its decision is final. This tale needs no new decisions or memory edits.

## Scope and current evidence

Linux checks cannot compile or run the AppKit targets or macOS-only Rust tests.
`bob-cli-by` and the repair notes on `bob-cli-bj.1`, `bob-cli-bj.12.2`, and
`bob-cli-bj.12.3` document failures that escaped into master. Provide an executable
developer check in each repository that snapshots the current working tree, pushes a
temporary commit to `ci/precommit/<unique-id>`, waits for that exact CI run, reports its
macOS results, and removes the remote branch.

The assigned phase is in progress and has no previous implementation or remainder notes.
No equivalent helper exists in either checkout. At planning time:

- bob-cli HEAD is `c2b6908913d8350cbefa9115da676499259e9f50`; CI run `40485333363`
  passed. `.github/workflows/ci.yml` already triggers on `ci/**` pushes. The macOS job
  is `test (macos-latest)`, with required steps `Lint macOS-only targets`, `Run tests`,
  and `Run archive-move presenter cases`.
- bob-mac-capture HEAD is `0e7eba439a5c59b2fecc67917b8f36dcf479e2f1`; CI run
  `40469875355` passed. Its workflow only triggers on master pushes and PRs. Its job
  `macOS 26 SwiftPM` runs on `macos-26`. Required coverage includes
  `Lint Swift formatting`, `Build`, `Test`, `Bundle`, `Verify plist and signature`,
  `Launch smoke test`, and `Install and reinstall`.
- bob-cli `just check-scripts` is already called by its CI lint job. Mac Capture has
  `docs-test` for offline Python script tests but no general script-check recipe.
- Neither checkout has implementation changes. The assigned phase currently owns no
  `--epic-symbol` entries; check again before closing.

This is one medium implementation: one small standard-library Python orchestration
script, mirrored into two standalone repositories with the same behavioral tests, plus
workflow wiring and two existing documentation pages. There are no Rust/Swift product
changes, host provisioning, new services, vault edits, app installations on Bryan's
machines, release changes, or AGENTS/memory changes.

## Repository access and files

Before working, re-read the assigned bead and this plan through the audited commands.
Open `bob-mac-capture` through `/sase_repo`, use only the returned checkout, and read
its `AGENTS.md`. Recheck both current trees/history so already-landed work is reused. Do
not use Bryan's personal Mac checkout.

Use these paths:

| Repository      | Files                                                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------------------------------------ |
| bob-cli         | `scripts/dev/macos-ci-check`, `scripts/dev/test_macos_ci_check.py`, `justfile`, `docs/develop/ci.md`                     |
| bob-mac-capture | `Scripts/macos-ci-check`, `Scripts/test_macos_ci_check.py`, `justfile`, `.github/workflows/ci.yml`, `docs/develop/ci.md` |

Use Python 3's standard library for JSON, subprocess deadlines, temporary indexes, and
signal handling. The executable scripts should be identical self-contained copies with a
small explicit profile table for the two supported GitHub repositories. Select the
profile from the script's own checkout and validated origin push URL; do not depend on
the invocation directory, another installed checkout, jq, or a network-fetched helper.
Keep the same self-test cases in both repositories. Avoid a separate packaging or
synchronization framework for these two copies.

## Implementation

### 1. Define the invocation and snapshot contract

Normal invocation is `scripts/dev/macos-ci-check` in bob-cli or `Scripts/macos-ci-check`
in Mac Capture. Provide `-h`/`--help` describing exactly what gets uploaded,
prerequisites, deadlines, result codes, cleanup, and the offline test command. No bob
CLI command or contract changes are needed.

Require `python3`, `git`, authenticated `gh`, repository push access, and a supported
GitHub origin push URL. Validate that both the push destination and explicit `gh --repo`
target identify the same supported repository. Reject ambiguous multiple push URLs, an
unresolved merge, unsupported checkout state, or missing prerequisites before pushing.
Never print credentials. Do not log in or provision credentials for the user.

Resolve HEAD once, create a temporary directory outside the checkout, and set
`GIT_INDEX_FILE` only in the child Git environment. Seed that index with
`read-tree HEAD`, stage the working tree into it with `add -A`, and use `write-tree` and
`commit-tree -p <original-head>` to make the verification object. Include a fresh
invocation identifier in the message so a clean or repeated invocation still has a
unique SHA. The parent explicitly authorizes this verification-only commit object; it
never becomes repository history and is not a substitute for `/sase_git_commit`.

The snapshot includes tracked modifications/deletions and non-ignored untracked files,
including the new gate/workflow/test files. It represents current file contents, not an
alternate staged-only version of partially staged files. Preserve HEAD, local branches,
the caller's index bytes and staged/unstaged split, and all working files. Use no
checkout, switch, stash, reset, normal commit, or update-ref operation. Respect ignored
files, including bob-cli's linked repositories. Fail clearly for unsupported dirty
submodules rather than claiming their uncommitted content was tested.

### 2. Push only a unique scratch ref and bound every wait

Create a high-entropy `refs/heads/ci/precommit/<id>` ref without creating a local
branch. Push only the snapshot SHA to that ref, using a create-only lease or equivalent
collision guard. Never force-overwrite an existing branch or push master. Print the
repository, base SHA, snapshot SHA, and remote branch before the network mutation so the
run remains recoverable after a hard interruption.

Use monotonic deadlines with documented defaults: five minutes for first-run discovery
after push, 90 minutes for the overall check, at most 60 seconds per external command
capped by the remaining deadline, and a separate two-minute cleanup budget. Allow
positive, finite numeric overrides through documented `BOB_MACOS_CI_*` environment
variables so tests and callers can shorten them. Print occasional bounded progress while
waiting. Network calls and diagnostic retrieval must have timeouts, not just the polling
loop. Disable Git terminal prompts and gh pagers/prompts for noninteractive execution.

Poll
`gh run list --repo <repo> --workflow ci.yml --event push --branch <branch> --commit <sha> --json ...`,
then verify the returned SHA, branch, and event before choosing a run. Verify that
identity again with `gh run view`. Never use the latest master run or an unrelated PR
run. Retry transient absence/API errors only within the relevant deadline; missing auth
or invalid JSON must never yield success.

Wait for the selected workflow to finish, keeping each macOS job's steps available for
reporting. Waiting for workflow completion also makes failed-job logs available. This
check conservatively requires the selected workflow to pass, including its other jobs,
as well as the required macOS coverage. State this in help/docs.

### 3. Report coverage and distinguish failure from unrun

Print the run URL and snapshot SHA, every macOS step with its actual verdict (including
skipped/cancelled steps), and any failed non-macOS jobs. Require exactly the expected
macOS job and the mandatory profile steps to have completed successfully before
returning pass. Known conditional diagnostic/upload/Linux-only steps may be skipped;
missing or skipped required build/test steps are never green coverage.

For failure, fetch the relevant failed job logs with bounded gh calls. Print Rust
`test ... FAILED`/failure-list names and Swift XCTest/Swift Testing failing test names
when present, retaining relevant compiler errors and watchdog/hang diagnostics. When
failure occurs before tests or names cannot be extracted, say so and print useful
failed-step diagnostics plus the run URL; do not invent a test verdict. Inspect later
steps after a red lint/build instead of silently dropping them from the report.

Result codes are:

- `0`: the exact selected workflow and mandatory macOS checks passed, and remote branch
  deletion was verified.
- `1`: a completed check reports a real CI failure. Preserve that failure verdict even
  if optional failure-log retrieval is unavailable, and explain missing logs.
- `2`: verification is incomplete or could not run: prerequisites/auth/push errors, no
  matching run, deadline, cancellation/interruption, malformed/missing coverage, or
  cleanup failure. Cleanup failure takes precedence and returns 2, while also reporting
  any observed CI failure. Missing optional failure logs alone do not change a known
  failure from 1 to 2.

### 4. Always enter cleanup and verify remote deletion

Register cleanup before starting the push, because a failed/timed-out push may have
created the remote ref. On success, CI failure, exceptions, timeouts, SIGINT, and
SIGTERM, stop/reap active child processes before cleanup so a late push cannot recreate
the branch after deletion. Handle subprocess process groups and bounded termination
grace periods. Cleanup itself must not be skipped by a second ordinary termination
signal.

Delete only this invocation's scratch ref, retry transient errors within the cleanup
budget, and verify its absence on the remote. Treat an already absent ref as success; do
not delete a ref now owned by a different SHA. Remove the temporary index/directory in
all cases. On timeout/interruption, best-effort cancel only the selected unfinished CI
run within the cleanup budget before deleting the ref.

Network loss, permission revocation, SIGKILL, or machine loss cannot be made into an
absolute remote-deletion guarantee. Do not conceal this: preserve the printed
repository/ref, report unverified cleanup as exit 2, and give the exact narrow branch
deletion/recheck command for recovery. Tests must prove deletion on all supported exit
paths and that a deletion error cannot be reported as a pass. Do not add a janitor
service or broaden this task into infrastructure provisioning.

### 5. Wire CI and offline behavioral tests

In Mac Capture, add `"ci/**"` to the workflow's push branches while preserving its
existing macOS runner, toolchain checks, PR trigger, steps, and timeout. Add a
`check-scripts` recipe for the new offline test and the existing documentation-script
checks, and invoke the offline tests in CI with the runner's Python before the Swift
checks. The CI test command must never recursively invoke the live gate. In bob-cli,
extend the existing `check-scripts` recipe with the Python test; do not feed an
extensionless Python program to `bash -n`.

Use `unittest` with temporary directories and stubbed `git`/`gh` executables, following
Mac Capture's existing `Scripts/test_docs_scripts.py` approach. Assert public behavior,
exit codes, exact command destinations/identity filters, and cleanup calls. Cover:

- pass, real Rust/Swift test failure with names, compile/lint failure with later skipped
  steps, and an unrelated job failure;
- wrong SHA/branch/event being ignored, delayed run discovery, no run, queued run
  deadline, cancelled run, missing required job/step, and ordinary conditional skips;
- auth/push/API errors, stalled subprocess, malformed output, failure-log
  unavailability, SIGINT/SIGTERM after push, and interruption during an in-flight push;
- cleanup on every post-push exit, transient delete retry, already absent ref, deletion
  failure, and refusal to delete another SHA; no master push or local branch mutation.

Also use a local temporary Git fixture with a bare remote and stubbed gh to verify the
real temporary-index behavior: mixed staged/unstaged edits, tracked deletion, a new
file, an ignored file, and a path containing spaces. Compare snapshot contents and
assert unchanged HEAD, refs, worktree contents, and caller index bytes on both success
and a failed gate. These offline fixture commits belong only to disposable test repos.
Tests must finish quickly with short injected deadlines and no network access.

### 6. Document and demonstrate the packaged gate

Update the two existing `docs/develop/ci.md` pages with invocation, snapshot semantics,
prerequisites, bounded waits/overrides, exit codes, exact-SHA coverage, cleanup and
interruption recovery, and the offline test command. Explain the expanded Mac Capture
push trigger. No new docs hierarchy is needed. Read `docs.md` and `cli_rules.md` through
`/sase_memory_read` before these changes; provide clear `-h`/`--help` output.

For SASE agents, document running the live gate through `/sase_monitor`, with a monitor
timeout longer than the gate's 90-minute deadline plus cleanup budget. Use a verify
monitor, since the command performs a snapshot and push before waiting. Its `--next`
must resume inspection, repairs if needed, verification evidence, and assigned-bead
closure. Wait for `sase monitor start` itself to exit; do not leave ordinary background
commands running or wait for CI inline. Read the current monitor skill before launch.

Record a `PROPOSED FOLLOW-UP:` on the assigned phase, citing existing memory task
`bob-cli-1x`, to incorporate the packaged precommit gate into agent guidance after the
lander obtains memory authorization. Do not edit `AGENTS.md` or any memory note, and do
not create or close a task bead.

## Verification and completion

Run the focused offline tests in both repositories, then bob-cli `just check-scripts`
and `just check` using the existing `$CARGO_TARGET_DIR`; never use `just check-full`.
Run Mac Capture `just check-scripts` and its required Linux `just portable-check`. These
do not replace the macOS run. Run `just docs-check` for the documentation changes with
the opened Mac Capture checkout mounted through `BOB_DOCS_BOB_MAC_CAPTURE_DIR`; when
using Mac Capture's wrapper, set `BOB_CLI_DIR` to the current bob-cli checkout so it
cannot clone an unapproved repository. Use the SASE monitor/tool workflow for checks
that exceed synchronous limits.

Prove the helper on one real scratch CI run per repository, including the uncommitted
new script/test/workflow files in each snapshot. Do this before landing the changes.
Record base SHA, snapshot SHA, run URL, every macOS step's result, process exit code,
and independent remote-ref absence verification. Compare local HEAD, index, and
working-tree state before and after the invocation. If verification exposes a defect in
the gate, fix it and rerun the affected tests and live proof on the final tree. Both
current baseline workflows are green, so two passing runs are expected.

A check failure reproduced identically on a clean base is not a reason to strand the
assigned phase: record it as a `PROPOSED FOLLOW-UP:` on that phase, cite an existing
task if one tracks it, and include the exact run/check evidence and limitation. The
script must still return its truthful failure/unrun code. Unrelated failures do not
expand this phase. Never treat missing auth, an unrun gate, or absent cleanup evidence
as proof of passing macOS verification; use a supported continuation to complete the
missing acceptance work.

Before closing, record implementation and verification evidence on
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1` and run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.1
```

Resolve each remaining entry or re-key its Justfile line to a still-open owner, then
close only this assigned phase with `sase bead close ... --note "<what was verified>"`.
Leave `bob-cli-by`, `bob-cli-1x`, the parent epic, and all ancestor plan beads for the
parent lander. Never set the phase status by hand. Finish the implementation turn with
`/sase_final`, declaring both changed repositories. Follow the required SASE commit
workflow; the temporary `commit-tree` object above does not authorize ordinary manual
git commits. Preserve the daily release train and leave versions/CHANGELOG untouched.
