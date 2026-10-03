---
tier: tale
title: Split capture completion into focused modules
goal:
  Refactor capture completion into cohesive Rust modules of at most 1500 lines while
  preserving all completion behavior and test coverage, then close only bob-cli-3s.1.
size: medium
proposed_by: bbugyi200.athena.bob-cli-3s.1
bead: bob-cli-3s.1
create_time: 2026-10-03 05:25:47
status: wip
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/bob-cli-3s.1.md)

# Split capture completion into focused modules

Implement phase bead `bob-cli-3s.1` from `plan:202610/split_largest_rust_files.md`. This
is a structural refactor of `src/native/capture_complete.rs`, preserving behavior and
coverage while making every resulting Rust file at most 1500 physical lines after
formatting. One coding agent can perform the bounded extraction, so this is a `tale`
sized `medium`.

## Inspected baseline and scope

The planning checkout is clean at `619201720934e641db800d7d2e86a8f6ac714193`. The
assigned file still has 4656 lines: 2488 lines before its outer test gate and a
2168-line test section containing 61 unit tests. Two additional test-only scan wrappers
appear in the production section. Runtime baseline checks have not been run during
planning; capture them before changing source as described below.

There are three production callers, all of whose paths must remain valid:

- `src/native.rs` calls `capture_complete::run`.
- `src/native/completion/tree.rs` calls `capture_complete::build_cli`.
- `src/native/completion/capture_text.rs` calls `capture_complete::shell_completion` and
  consumes the returned public fields.

The module owns completion, not capture parsing, task status, or vault writes. Refactor
only this component and necessary module/import wiring. Do not change dependencies, CLI
flags/help, docs contracts, neighboring components, or existing integration-test
expectations. Do not perform live-vault or real-clipboard tests.

## Final production layout

Retain `src/native/capture_complete.rs` as the sole small facade; add real child modules
under `src/native/capture_complete/`. Do not also create a competing
`capture_complete/mod.rs`, and do not use textual `include!` fragments. The following
estimates include space for imports and ordinary formatting and leave substantial
headroom. Final physical counts are authoritative.

| File                             | Responsibility                                                                                                                                     | Expected lines |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------: |
| `capture_complete.rs`            | Module declarations, shared command-name constant, explicit facade exports, test gate                                                              |       under 60 |
| `capture_complete/cli.rs`        | Existing command builder, Clap errors/arguments, input collection, cursor validation, output-format parsing, and `run`                             |      about 320 |
| `capture_complete/model.rs`      | Serialized result/candidate types, replacement spans, schema constant, `Candidates::len`, empty results, output-format and completion-error types  |      about 280 |
| `capture_complete/shell.rs`      | Existing shell row/result types, `shell_completion`, continuation handling, and shell Pomodoro descriptions                                        |      about 400 |
| `capture_complete/engine.rs`     | `build_result`, close-log context check, wikilink precedence, block-ID handling, and candidate dispatch                                            |      about 230 |
| `capture_complete/candidates.rs` | Route, section, task, task-section, active-task, and task-link providers; task ranking/search, routed-note reads, and task-section lookup warnings |      about 550 |
| `capture_complete/pomodoros.rs`  | Name/start discovery, scan-to-candidate conversion, creation eligibility/order, and plan-budget hints                                              |      about 520 |
| `capture_complete/support.rs`    | Shared case-insensitive `rank` and `bounded_warning` helpers                                                                                       |       under 80 |
| `capture_complete/render.rs`     | Existing human output/labels/badges/warnings, success serialization, and error output                                                              |      about 370 |

Keep `run`, `build_cli`, `shell_completion`, `ShellRow`, and `ShellCompletion` available
at `native::capture_complete` with their existing visibility/signatures and field
access. Re-export shell types explicitly with appropriate handling of unused re-export
warnings; do not broaden visibility or suppress unrelated lints.

The dependency direction is: CLI uses engine/model/render; engine and shell use
providers/model; candidate and Pomodoro providers use model/support and existing native
scanners; rendering uses model and existing style/link types. Model must not call
providers. Put the shared `COMMAND_NAME` in the facade so CLI and render can use it
without depending on each other. Put `SCHEMA_VERSION` and the Serde `is_false` predicate
with the model. Keep `OutputFormat::from_matches` with CLI parsing (an impl there is
fine), preserving its default logic.

Use explicit imports from sibling modules and existing `crate::native` modules.
`pomodoros.rs` deliberately avoids a name collision with `native::pomodoro`. Expose
types, fields, methods, and cross-module functions only as narrowly as needed, normally
`pub(super)` within the component. Keep single-module helpers private. The facade must
not expose candidate-provider internals to other native components just to make tests
compile.

Useful boundaries in the inspected file, offered as navigation rather than fragile
line-based instructions:

- `run` through `raw_text_from_matches` is CLI/input code.
- `Replacement` through the result type plus its empty constructor are models; move
  completion error types/constructors and their exit-code method here too.
- `ShellRow` through `pomodoro_description` is shell adaptation.
- `build_result` is the completion engine; move `cursor_in_close_log_text` here.
- Most providers run from `route_candidates` through `task_link_candidates`; move the
  intervening `pomodoro_name_candidates` into `pomodoros.rs` instead.
- `pomodoro_name_candidates_at` through `pomodoro_name_candidate_with_next_up` belongs
  in `pomodoros.rs`.
- Later task-section warnings, task-search fields, and `read_target` belong with the
  non-Pomodoro providers. `rank` and `bounded_warning` are shared support.
- `print_success` through `success_json`, plus `print_error`, is rendering.

Move bodies, comments, derives, and Serde attributes intact. Change paths, visibility,
and imports as needed; do not consolidate duplicated algorithms or redesign logic during
this extraction.

## Test layout and isolation

Replace the inline test block with a gated `tests` module at
`capture_complete/tests/mod.rs` and these suites:

- `routes_tasks.rs`: route/section discovery, identified task and all-task
  grouping/search, task metadata/sections/warnings, explicit toggle suffixes, and
  authored block-ID contexts.
- `active_tasks.rs`: active-task fixture, queue ordering, ranking, status eligibility,
  suffix preservation, and active-task human rows.
- `task_links.rs`: task-link fixture/result helper, picker order, JSON omissions, query
  ranges, batch scoping, human rows, and missing-day-file behavior.
- `pomodoros.rs`: name/nameable/create candidates, named-start discovery and row kinds,
  the named-start ledger fixture, plan-budget previews, and corresponding human/JSON
  assertions. This suite should remain under 1000 lines.
- `output.rs`: CLI builder smoke test, empty results, non-completable close/hash text,
  Work Log wikilinks, general wikilink precedence/ranges/warnings, stable result
  serialization, and plain human output.

Keep all 61 existing test function names, bodies, assertions, and fixtures; module path
changes are expected. Route each test exactly once. Keep common `result`, `result_all`,
`write_settings`, `write_file`, `with_env`, `TempDir`, and time helpers in
`tests/mod.rs`. The fixture counter `TEMP_COUNTER` and process-global `DAY_FILE_LOCK`
must each have exactly one shared instance there; all suites use the existing
`day_file_guard` whenever they currently override the daily file. Do not replace that
lock with one per suite or remove any guard.

Preserve `#[cfg(test)]` on `pomodoro_name_candidates_from_scan` and
`pomodoro_start_name_candidates_from_scan`; they can stay narrowly visible in the
provider module. Keep `PlanCreationHint` and any scan conversion helpers needed by moved
tests accessible within this component only. Use deliberate test imports instead of
production-wide visibility. Add new tests only if the split exposes a material coverage
gap, not to test module organization.

The unchanged integration coverage includes 8 tests in `complete_block_id.rs`, 22 in
`complete_editor.rs`, 12 in `complete_query.rs`, 5 in `complete_task_link.rs`, and 19 in
`completion/capture_text.rs`. These source counts guide discovery checks; they are not
claims of passing runtime tests.

## Implementation and verification

1. Re-read `bob-cli-3s.1` and the audited epic plan in the implementation turn, inspect
   current source and status, and preserve unrelated work. Honor the epic's serial
   dependency ordering; implement only this phase.
2. Before source edits, capture test discovery and focused outcomes on the clean base.
   Use `cargo test --bin bob native::capture_complete::tests -- --list` and
   `cargo test --test cli -- --list`; save the relevant names/counts and command results
   as evidence. Run `cargo test capture_complete`, `cargo test --test cli complete`,
   `cargo test --test cli completion`, and
   `cargo test --test cli capture::complete_block_id`. The explicit block-ID suite is
   necessary because its test names do not all match `complete`. Read and use
   `/sase_monitor` for any long-running command handoff, retaining this plan and phase
   context in its continuation. Baseline check execution must not trigger implementation
   edits before plan approval.
3. Extract the production modules and facade as above. Move result/model and shared
   helpers first, then provider, dispatch, shell, CLI, and rendering bodies, resolving
   imports and narrowly scoped visibility. Preserve the current bytes/ordering of help
   and serialized output.
4. Move the test suites and shared support without losing discovery or test
   synchronization. Run `cargo fmt`, then compile/run the focused commands. Resolve
   refactor-caused compiler/lint/test failures without changing expected behavior.
   Compare unit test discovery by leaf test name to the baseline: exactly the same 61
   names must remain, without duplicates. Confirm the CLI suites above execute nonzero
   tests, including all four `complete_*` modules and the shell capture-text suite.
5. After final formatting, audit `src/native/capture_complete.rs` plus every `*.rs`
   recursively beneath `src/native/capture_complete/`, including tests and support.
   Count physical lines including blanks/comments; every file must be at most 1500.
   Verify there is exactly one root and cohesive real modules.
6. Run `just all`: `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   full `cargo test`. Use `/sase_monitor` when needed. Checks that passed need not be
   repeated absent later changes. Preserve existing cfg gates; report only platform
   paths actually compiled/tested (planning source has no target-specific gates here).
7. Review the final diff for unchanged public entry points, no dropped tests or
   comments, preserved schema version 1 and omission attributes, candidate
   ranking/discovery order, UTF-8/replacement/suffix behavior, wikilink precedence,
   warning text/bounds, creation eligibility, shell safe-row filtering, early wikilink
   deferral, exit codes, stdout/stderr routing, and read-only behavior. Record final
   file map/counts, before/after discovery, actual command outcomes, and any verified
   clean-base/platform limitations on `bob-cli-3s.1`.
8. Before closing, run `sase bead epic-symbols bob-cli-3s.1` again. Planning found no
   entries. If entries exist at completion, resolve them or re-key the corresponding
   Justfile lines to a still-open bead (parent epic or later phase); do not leave a
   closed-phase exemption. Close **only** this assigned phase with
   `sase bead close bob-cli-3s.1 --note "<what was verified>"`.

Do not set the phase's status by hand: it is already assigned and `in_progress`. Do not
close parent epic `bob-cli-3s` or any ancestor plan bead. Do not create task beads.
Record unrelated discoveries via
`sase bead note bob-cli-3s.1 'PROPOSED FOLLOW-UP: <summary — detail>'` for the epic's
land agent. A failure reproduced identically on the clean base is not a reason to leave
this phase open: document its reproduction and any existing tracking bead, then close
once the structural work and applicable verification are complete. Fix failures
introduced by this phase before closing. Follow the required SASE final declaration for
an ordinary implementation-turn ending; use only successfully completed handoffs as the
documented exemption.
