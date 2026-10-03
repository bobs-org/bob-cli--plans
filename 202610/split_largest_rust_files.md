---
tier: epic
title: Split the five largest Rust files into maintainable modules
goal: Refactor the five Rust files identified in this plan, in sequence, into cohesive
  modules whose production and test files each contain at most 1500 physical lines,
  preserving existing behavior, interfaces, and test coverage. Each large phase owns
  planning its final split against the code present when it starts.
phases:
- id: split-capture-complete
  title: Split capture completion into focused modules
  size: large
  depends_on: []
  description: 'split-capture-complete: Reinspect src/native/capture_complete.rs and
    plan its final split; consider CLI/model, shell completion, candidate providers,
    rendering, and test modules. Implement the split with every resulting Rust file
    at most 1500 lines and preserve completion behavior and coverage.'
- id: split-capture-task-toggle
  title: Split task toggle and link planners into focused modules
  size: large
  depends_on:
  - split-capture-complete
  description: 'split-capture-task-toggle: After split-capture-complete, reinspect
    src/native/capture_task_toggle.rs and plan its final split; consider task updates,
    link insertion/removal, relocation, ledger planning, text edits, and tests. Keep
    every resulting Rust file at most 1500 lines and preserve pure-planner behavior
    and coverage.'
- id: split-task-status-hooks-write
  title: Split guarded task status writes into focused modules
  size: large
  depends_on:
  - split-capture-task-toggle
  description: 'split-task-status-hooks-write: After split-capture-task-toggle, reinspect
    src/native/task_status_hooks_write.rs and plan its final split; consider model,
    snapshots/preflight, apply orchestration, recovery, filesystem staging, and tests.
    Keep every resulting Rust file at most 1500 lines and preserve guarded-write sequencing,
    platform behavior, and coverage.'
- id: split-capture-clip
  title: Split clipboard capture into focused modules
  size: large
  depends_on:
  - split-task-status-hooks-write
  description: 'split-capture-clip: After split-task-status-hooks-write, reinspect
    src/native/capture_clip.rs and plan its final split; consider clipboard providers/history,
    content planning/rendering, attachment reservations, persistence, and tests. Keep
    every resulting Rust file at most 1500 lines and preserve platform gates, rollback
    behavior, and coverage.'
- id: split-plugins
  title: Split plugin management into focused modules
  size: large
  depends_on:
  - split-capture-clip
  description: 'split-plugins: After split-capture-clip, reinspect src/native/plugins.rs
    and plan its final split; consider CLI, discovery/models, Git refresh, sync, diff/rendering,
    and tests. Keep every resulting Rust file at most 1500 lines, preserve plugin
    management behavior and coverage, and verify the cumulative file-size and test
    results for all five refactors.'
proposed_by: bbugyi200.athena.0vn
create_time: 2026-10-03 05:16:49
status: wip
bead_id: bob-cli-3s
---

- **PROMPT:** [prompts/202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/split_largest_rust_files.md)
- **BEAD:** [bob-cli-3s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/README.md)

# Split the five largest Rust files

## Scope and measured baseline

The inventory below was measured on 2026-10-03 at commit
`619201720934e641db800d7d2e86a8f6ac714193`. All 344 tracked `*.rs` files were
considered, including integration tests. These are the five largest by physical line
count; blank lines, comments, and inline tests count toward the limit.

| Phase | Assigned file                           | Total lines | Before inline test module | Inline test section |
| ----- | --------------------------------------- | ----------: | ------------------------: | ------------------: |
| 1     | `src/native/capture_complete.rs`        |        4656 |                      2488 |                2168 |
| 2     | `src/native/capture_task_toggle.rs`     |        2507 |                      1683 |                 824 |
| 3     | `src/native/task_status_hooks_write.rs` |        2398 |                      1529 |                 869 |
| 4     | `src/native/capture_clip.rs`            |        2239 |                      1394 |                 845 |
| 5     | `src/native/plugins.rs`                 |        2215 |                      1526 |                 689 |

The final two columns mark the outer `#[cfg(test)] mod tests` boundary; some earlier
helpers are also test-only. They are planning estimates, not prescribed cut points.
Moving only the inline test section would leave phases 1, 2, 3, and 5 with an oversized
production file, and phase 1 with oversized tests too.

There are exactly five phases, all `size: large`, with this dependency chain:

`split-capture-complete` → `split-capture-task-toggle` → `split-task-status-hooks-write`
→ `split-capture-clip` → `split-plugins`.

The first phase has no dependency. Every other phase depends on completion of the
immediately preceding phase. Preserve this serial ordering even if the work looks
independently implementable. Large sizing deliberately gives each worker a planning
handoff before implementation; use size-derived model routing.

## Instructions shared by every phase

1. **Own the final plan.** At phase start, inspect the current assigned file, its
   callers, tests, module conventions, and the preceding phase's changes. Recount lines
   and identify cohesive boundaries before proposing the implementation split through
   `/sase_plan`. The layouts below are recommendations, not required filenames or frozen
   boundaries. Adjust them to the code actually present when your phase runs, and
   explain the selected layout and validation in your plan. Keep ownership of the
   originally assigned responsibility even if that file has already moved; do not
   substitute the next-largest unrelated file.
2. **Split one assigned file's responsibilities.** Create multiple focused Rust files
   and a small module root/facade. Either retain `<name>.rs` with child modules under
   `<name>/`, or replace it with `<name>/mod.rs`; never leave both competing roots.
   Existing examples include `src/native/projects/` and `src/native/task_dependencies/`.
   Preserve existing `native::<name>` entry points with explicit re-exports where
   practical. Make only necessary wiring/import changes outside the assigned component.
3. **Keep every resulting file at most 1500 physical lines after formatting.** This
   includes the retained root, extracted production modules, test modules, and test
   support. Prefer useful headroom over landing exactly at the ceiling. Do not move a
   large block into an already oversized neighbor, compress formatting, drop comments or
   tests merely to pass, or substitute textual `include!` chunks for real module
   boundaries. Unrelated pre-existing files over the limit are outside this epic's
   scope.
4. **Preserve behavior.** Keep CLI options/help, exit codes, stdout/stderr routing, JSON
   schemas and omission rules, candidate ordering, errors, file bytes, and side effects
   unchanged. This is a structural refactor; it does not authorize redesigning capture
   grammar, task policies, write semantics, or plugin syncing. Use the narrowest
   necessary visibility for sibling modules; do not broadly make implementation details
   public to satisfy the compiler. Preserve `cfg` gates, module-level attributes,
   test-only helpers, and platform-specific dependencies.
5. **Preserve tests and isolation.** Move existing tests into discoverable modules,
   retaining assertions and fixture coverage. Compare test discovery before and after;
   module-path changes are expected, lost tests are not. Keep one shared instance of any
   test environment lock or counter. Add a regression test only for a material coverage
   gap exposed by the split, not for module organization.
6. **Validate before phase completion.** Use the checks below and record the final file
   map, counts, actual commands/results, and any platform limitations. Fix regressions
   caused by the refactor. Epic phase workers record unrelated findings as
   `PROPOSED FOLLOW-UP:` notes on their own phase bead, following project policy.

## Phase 1: split-capture-complete

**Assigned file:** `src/native/capture_complete.rs` (4656 lines). **Size:** large.
**Depends on:** none.

The current file combines the Clap command, serialized completion result models, shell
completion adaptation, completion dispatch, candidate selection/ranking, human
rendering, and 2168 lines of tests. This needs both production and test decomposition.

Recommended starting layout under `src/native/capture_complete/`:

- A small root exposing `run`, `build_cli`, `shell_completion`, and the existing shell
  row/result types needed by callers.
- `cli.rs` for command construction, argument parsing, input and command dispatch;
  `model.rs` for result/candidate/error types and schema constants.
- `shell.rs` for `shell_completion`, replacement spans, insertion behavior, and
  shell-specific candidate adaptation; `engine.rs` for `build_result` and context
  dispatch.
- `candidates.rs` (or a few cohesive provider modules) for route, section, task,
  task-section, active-task, and task-link candidates and ranking; `pomodoro.rs` for
  named/start candidates, creation rows, and plan-budget hints.
- `render.rs` for human rows, badges, warnings, labels, and JSON output.
- `tests/mod.rs` for shared fixtures and synchronization, with suites such as
  `routes_tasks.rs`, `task_links.rs`, `pomodoros.rs`, and `output.rs`. Subdivide further
  if any test file approaches the ceiling.

Reassess which types and helpers belong together: avoid forcing a model/provider cycle
to match these names. Preserve the shared `DAY_FILE_LOCK` used for `BOB_DAY_FILE`
overrides when tests become sibling modules. Test-only helpers currently appear before
the outer test module and must remain test-only.

Keep the interfaces used by `src/native.rs`, `completion/tree.rs`, and
`completion/capture_text.rs` stable. Preserve cursor/replacement semantics, wikilink
precedence, ranking, identified/unidentified task grouping, creation candidate
eligibility, warning bounds, and schema version 1 serialization.

Focused validation: `cargo test capture_complete`, `cargo test --test cli complete`, and
`cargo test --test cli completion`, plus the common checks below. Review the
`complete_*` files under `tests/cli/capture/` and `tests/cli/completion/`; make sure the
filters actually execute these tests after any module renaming.

## Phase 2: split-capture-task-toggle

**Assigned file:** `src/native/capture_task_toggle.rs` (2507 lines). **Size:** large.
**Depends on:** `split-capture-complete`.

This component contains pure string-to-plan transformations for task status and
scheduling updates, link insertion/removal, relocation, and daily ledger edits. It has
many callers, including capture orchestration, active-task discovery, Pomodoro
start/close, and randomization.

Recommended starting layout under `src/native/capture_task_toggle/`:

- A root facade retaining the existing planners, result types, constants, and helpers
  consumed by sibling `native` modules.
- `task_update.rs` for `plan_task_next`, `plan_task_link`, status mutation,
  future-schedule handling, and pull-forward Schedule Log text.
- `links.rs` for insertion/removal planners and link discovery; `relocation.rs` for
  relocation selection, destinations, and result types; `ledger.rs` for
  `plan_pomodoro_link_ledger` and open-entry link enumeration.
- `text.rs` for the shared line/subtree edits, indentation and newline handling, and
  placeholder insertion if that fits its actual dependencies better than
  `relocation.rs`.
- `tests.rs` initially fits comfortably; separate task-update and relocation suites if
  that gives cleaner ownership in the final plan.

Keep closely related types next to their planner or use a small shared model module if
needed. The worker must resolve the actual helper dependency graph; these suggested
buckets do not justify duplicating link scans or text edits.

Preserve pure planning with no disk I/O, sticky lane semantics, future-date retirement
rules, Schedule Log emphasis, named/implicit entry selection, idempotence, subtree
ownership, CRLF handling, and a missing final newline. Do not use this refactor to
reconcile old comments or policies with a different behavior. If consulting the
referenced plugin implementation, first open `bob-plugins` with `/sase_repo`; no plugin
modification is needed for this phase.

Focused validation: `cargo test capture_task_toggle`, `cargo test --test cli capture`,
and `cargo test --test randomize`, plus common checks. Ensure task-toggle, Ensure Next,
Pomodoro-link, start/drop, and close integration coverage remains present.

## Phase 3: split-task-status-hooks-write

**Assigned file:** `src/native/task_status_hooks_write.rs` (2398 lines). **Size:**
large. **Depends on:** `split-capture-task-toggle`.

This module implements guarded writes and recovery, used by both task-status hooks and
randomization. Simply extracting its 869-line test section still leaves 1529 lines
before the test module, so extract production responsibilities as well.

Recommended starting layout under `src/native/task_status_hooks_write/`:

- A root facade keeping existing types/functions/constants available to callers.
- `model.rs` for plans, snapshots, sessions, error/reason types, and outcomes.
- `snapshot.rs` for capture/recapture, identities, hashing, and stable reads;
  `preflight.rs` for read-set checks, quiet-period checks, and replacement checks.
- `apply.rs` for lock/run setup and the ordered apply operation.
- `recovery.rs` for manifests, original-byte retention, updates, and pruning;
  `staging.rs` for exclusive temporaries, permissions, metadata/xattrs, cleanup, and
  relevant platform-specific helpers.
- `tests.rs`, or separate snapshot/apply/recovery suites sharing the existing
  fault-injection fixtures, if that improves clarity.

The worker owns whether these boundaries need merging or adjustment to avoid circular
dependencies and excessive visibility. Keep the ordered transaction flow easy to audit:
quiet period and preflight, recovery creation, staging, revalidation, per-file
replacement checks, replacement, manifest updates, and retention cleanup. Preserve
callback timing and the shared run counter.

Preserve lock behavior, content and identity validation, symlink/hardlink rejection,
private permissions, Linux `O_NOFOLLOW` and xattr gates, non-Linux fallbacks, recovery
schema/path conventions, partial-apply reporting, cleanup, and no-op behavior. Preserve
tool-specific recovery roots used by randomization.

Focused validation: `cargo test task_status_hooks_write`,
`cargo test --test cli task_status_hooks`, and `cargo test --test randomize`, plus
common checks. Existing tests must continue to cover edits before/after staging and
before replacement, changed scan membership, replacement inode, symlink substitution,
staging failure, retention, and quiet-period behavior.

## Phase 4: split-capture-clip

**Assigned file:** `src/native/capture_clip.rs` (2239 lines). **Size:** large. **Depends
on:** `split-task-status-hooks-write`.

This file combines clipboard access/history, Clipy decoding, capture planning,
rendering, attachment/snippet reservations, file persistence, and tests. Although test
extraction alone would bring the current production body below the limit, prefer
separating the major I/O and planning responsibilities for useful headroom.

Recommended starting layout under `src/native/capture_clip/`:

- A root facade for the existing `ClipPlan`, `ClipOutput`, `ClipReservations`,
  planning/provider functions, header helpers, and cleanup functions.
- `model.rs` for output and plan types; `clipboard.rs` for provider selection, command
  output normalization, history override and merge behavior; `clipy.rs` for SQLite/plist
  history decoding with its existing platform gates.
- `plan.rs` for content classification, aggregate planning and output rendering;
  `files.rs` for attachment/snippet naming, deduplication, collision handling, and
  shared reservations. Keep reservation ownership clear across batch items.
- `persist.rs` for saving planned files, temporary-file handling, and cleanup;
  `tests.rs` or suites grouped around providers/planning/persistence.

The worker may choose fewer files where cohesion is better, but should keep pure
planning distinct from clipboard reads and writes. Retain the
`cfg(any(target_os = "macos", test))` coverage for Clipy/SQLite/plist helpers: their
fixture tests intentionally compile on Linux through dev dependencies.

Preserve clipboard provider precedence, live/history ordering, normalization, header and
Markdown indentation rules, attachment limits, stable names, reuse and collision
behavior, JSON collection fields, dry-run behavior, and removal of only the files
created by the failed capture. Keep the interfaces consumed by capture planning/commit
and capture-language marker validation stable.

Focused validation: `cargo test capture_clip`, `cargo test --test cli capture::clip`,
and `cargo test --test cli capture::batch`, plus common checks. Exercise existing
command/SQLite fixtures and failure tests; do not use the real clipboard or live vault
as a refactor test fixture. Report which target-specific paths were actually
compiled/tested; preserve other gates by inspection rather than claiming unrun platform
coverage.

## Phase 5: split-plugins

**Assigned file:** `src/native/plugins.rs` (2215 lines). **Size:** large. **Depends
on:** `split-capture-clip`.

This module implements the plugin CLI, Git refresh, discovery and state models, guarded
syncing/backups, diff calculation, human/JSON output, and tests. Its 1526-line pre-test
body also needs production extraction.

Recommended starting layout under `src/native/plugins/`:

- A root exposing `run` and `build_cli`; `cli.rs` for Clap construction, option
  resolution, list/sync dispatch, and exit status selection.
- `model.rs` for manifests, reports, state enums, and sync outcomes; `scan.rs` for
  discovery, enabled-plugin state, and managed-file comparisons.
- `git.rs` for worktree detection, refresh/summaries, and dirty-file checks; `sync.rs`
  for per-plugin/per-file decisions, backups, and copies.
- `diff.rs` for textual/binary/minified diff classification and calculations;
  `render.rs` for table widths, styling, sync reports, and JSON output.
- `tests.rs` initially fits, or `tests/` with discovery, sync, and Git suites and shared
  temporary fixture helpers.

Choose final boundaries from the current code; keep report methods together where
splitting them would increase coupling. Preserve command defaults and help, managed file
selection, default paths/environment overrides, pull-before-analysis and `--no-pull`,
warning/exit-code behavior, stdout/stderr separation, dirty-file refusal/force,
backup-before-overwrite, dry-run, diff limits, and untouched runtime `data.json`.
`docs/plugins.md` describes these contracts.

Focused validation: `cargo test plugins` and `cargo test --test cli completion`, plus
common checks. Existing plugin tests use temporary repositories/vaults, local Git
remotes, and isolated backup roots; retain that isolation. This phase refactors
bob-cli's plugin management code; it does not change plugin source or require deployment
with `bob plugins sync`.

At completion, also verify all five components' resulting file maps and line counts and
run the full checks on the cumulative tree. This is the final phase's validation duty,
not a sixth implementation phase.

## Validation and completion criteria

Before editing, each phase captures its applicable baseline test discovery and focused
test outcome. After moving modules, run the focused commands listed for the phase and
confirm they execute nonzero, relevant tests. Fix filters if module paths change.
Existing CLI/integration tests serve as behavior checks; do not rewrite expected results
merely to accommodate a refactor.

Run the repository's `just all` before completing each phase: it runs
`cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test`. Use
`cargo fmt` while editing as needed. Do not add unrelated lint cleanup or claim
unavailable platform checks passed. Follow `/sase_monitor` for commands that need a
long-running handoff.

For the assigned component, count the root plus every extracted Rust file recursively,
including tests/support, after formatting. The following command audits the conventional
root/child layouts for all five components; during an intermediate phase, apply the
ceiling to completed components only. If a final layout differs, include its additional
paths explicitly in the phase's report.

```bash
python3 - <<'PY'
from pathlib import Path

names = (
    'capture_complete', 'capture_task_toggle', 'task_status_hooks_write',
    'capture_clip', 'plugins',
)
for name in names:
    root = Path('src/native')
    paths = [root / f'{name}.rs', *(root / name).rglob('*.rs')]
    files = sorted(path for path in paths if path.is_file())
    if not files:
        print(f'MISSING COMPONENT: {name}')
    for path in files:
        count = len(path.read_bytes().splitlines())
        marker = ' OVER LIMIT' if count > 1500 else ''
        print(f'{count:5} {path}{marker}')
PY
```

Each phase is complete only when its assigned responsibility is split into cohesive
files, every resulting file is at most 1500 lines, its existing behavior and test
coverage remain intact, and required checks pass (or an independently established
baseline/environment limitation is explicitly documented). Record before/after file
counts and any justified change to the suggested layout.

The epic is complete when all five serial phases satisfy those criteria and cumulative
verification passes. Other oversized Rust files are intentionally outside this five-file
effort.
