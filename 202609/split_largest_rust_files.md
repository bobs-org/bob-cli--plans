---
tier: epic
title: Split the ten largest Rust files into modules of at most 1500 lines
goal: 'Each of the ten largest Rust source files in bob-cli is split into cohesive
  modules in which every resulting file has at most 1500 lines. Behavior does not
  change, the test count stays the same, and `just all` stays green after every phase.

  '
phases:
- id: split-cli-tests
  title: Split tests/cli.rs
  depends_on: []
  size: large
  description: 'split-cli-tests: turn the 35k-line CLI integration test file into
    a single `tests/cli/` test target with a shared support module and per-command
    test modules, each at most 1500 lines.'
- id: split-capture
  title: Split src/native/capture.rs
  depends_on:
  - split-cli-tests
  size: large
  description: 'split-capture: turn the capture executor into a directory module (CLI,
    planning, Pomodoro operations, commit/staging, markdown placement, output, and
    split unit tests), each file at most 1500 lines.'
- id: split-capture-language
  title: Split src/native/capture_language.rs
  depends_on:
  - split-capture
  size: large
  description: 'split-capture-language: split the pure capture grammar into model,
    draft/item parsing, token parsers, markers, editor parse, completion, rewrite,
    and split unit tests, each file at most 1500 lines.'
- id: split-highlights-ref
  title: Split src/native/highlights_ref/mod.rs
  depends_on:
  - split-capture-language
  size: large
  description: 'split-highlights-ref: thin the existing highlights_ref directory root
    into sync, reporting, sidecar, annotation-task, note, marker, and frontmatter/IO
    submodules plus split tests, each file at most 1500 lines.'
- id: split-dataview
  title: Split src/native/dataview.rs
  depends_on:
  - split-highlights-ref
  size: large
  description: 'split-dataview: move the Obsidian engine, native evaluator, function
    library, lexer/parser, sources, errors, and CLI out of the dataview root into
    new files under the existing dataview directory.'
- id: split-task-status-hooks
  title: Split src/native/task_status_hooks.rs
  depends_on:
  - split-dataview
  size: large
  description: 'split-task-status-hooks: turn the task status hook engine into a directory
    module (model, retry, sync, pomodoro, settings, structure, references, compose,
    output, split tests).'
- id: split-projects
  title: Split src/native/projects.rs
  depends_on:
  - split-task-status-hooks
  size: large
  description: 'split-projects: turn the projects command into a directory module
    (model, scan/parse, sync planning, edits, tags/inline fields, output, split tests).'
- id: split-collect-done
  title: Split src/native/collect_done.rs
  depends_on:
  - split-projects
  size: large
  description: 'split-collect-done: turn move-done-tasks collection into a directory
    module (plan, git, archive, link repair, markdown transform, split tests) and
    fix relative include_str! fixture paths.'
- id: split-task-status-groups
  title: Split src/native/task_status_groups.rs
  depends_on:
  - split-collect-done
  size: large
  description: 'split-task-status-groups: apply a light three-to-five-file split to
    the task status grouping transform (model/transform, parse, emit/headings, tests).'
- id: split-capture-pomodoro-close
  title: Split src/native/capture_pomodoro_close.rs
  depends_on:
  - split-task-status-groups
  size: large
  description: 'split-capture-pomodoro-close: separate the ledger close planner, link
    and marker parsing, the linked-task close planner, and their two test modules
    into a directory module.'
proposed_by: bbugyi200.apollo.2u
create_time: 2026-09-28 16:49:28
status: wip
bead_id: bob-cli-2f
---

- **PROMPT:** [prompts/202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/split_largest_rust_files.md)
- **BEAD:** [bob-cli-2f](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/README.md)

# Plan: Split the ten largest Rust files into modules of at most 1500 lines

## Context

Line counts at commit `d0c1692` (`git ls-files '*.rs' | xargs wc -l`):

| #   | File                                   | Lines  | Code / tests (approx.)                           |
| --- | -------------------------------------- | ------ | ------------------------------------------------ |
| 1   | `tests/cli.rs`                         | 35,334 | 515 integration `#[test]`s + ~1,000 helper lines |
| 2   | `src/native/capture.rs`                | 12,340 | 1–9370 code, 9371–12340 tests                    |
| 3   | `src/native/capture_language.rs`       | 11,615 | 1–7034 code, 7035–11615 tests                    |
| 4   | `src/native/highlights_ref/mod.rs`     | 10,240 | 1–7682 code, 7683–10240 tests                    |
| 5   | `src/native/dataview.rs`               | 7,088  | 1–6847 code, 6848–7088 tests                     |
| 6   | `src/native/task_status_hooks.rs`      | 6,107  | 1–4204 code, 4205–6107 tests                     |
| 7   | `src/native/projects.rs`               | 4,652  | 1–3227 code, 3228–4652 tests                     |
| 8   | `src/native/collect_done.rs`           | 4,344  | 1–2564 code, 2565–4344 tests                     |
| 9   | `src/native/task_status_groups.rs`     | 2,991  | 1–2021 code, 2022–2991 tests                     |
| 10  | `src/native/capture_pomodoro_close.rs` | 2,982  | code and tests interleaved (see phase)           |

There is one phase per file, run strictly in sequence (each phase depends on the
previous one), ordered largest first. Every phase is a pure structural refactor of
**one** file.

## Rules for every phase

### You own the final split

The layout recommended in each phase section comes from a quick outline of the file at
`d0c1692`. It is **a starting point, not a spec.** Before you move anything:

1. Re-measure the file (`wc -l <file>`). Earlier phases or unrelated commits may have
   changed it. Treat every line number in this plan as an approximate landmark. If the
   file is already at most 1500 lines, say so in your summary and finish without
   changes.
2. Read the file's actual structure (top-level items, `#[cfg(test)]` blocks, `//`
   section comments, and who calls what). Then plan your own split. Merge, rename, or
   re-cut the recommended modules whenever the code's real cohesion points somewhere
   else. For example, if one recommended module would exceed 1500 lines, cut it again.
   If two tiny ones belong together, merge them.
3. State the final layout (each file, its responsibility, and its line count) in your
   final summary so later phases and reviewers can see what you chose.

### Hard requirements

- **At most 1500 lines per resulting file, test modules included.** Aim well below the
  limit (roughly 300–1200 lines) so normal growth doesn't push a file back over right
  away. Don't over-fragment either: avoid files under about 80 lines unless the unit is
  genuinely standalone.
- **No behavior changes.** Move code verbatim: keep doc comments, `//` section comments,
  and the relative order of items within each moved group. Change only what the move
  needs: `mod`/`use` declarations, visibility, and `super::` paths. Don't rename items
  or "clean up" logic on the way.
- **Don't lose or weaken tests.** Record the test count before you start and compare it
  at the end. For example: `cargo test -- --list 2>/dev/null | grep -c ': test$'`. The
  count must match, and no assertion may change.
- **Stable external paths.** Other modules import items through the old module path (for
  example `super::capture::…`, `super::task_status_hooks::…`, or
  `super::collect_done::…`). The new parent module should re-export what outsiders use
  (`pub(crate) use child::Item;`) so callers outside the new directory need no edits, or
  only minimal ones.
- **Keep visibility narrow.** Items that are shared only between the new sibling
  submodules get `pub(super)` (or `pub(in crate::native::<module>)` for deeper trees).
  Don't widen anything to `pub(crate)` unless it already was.
- **Stay in scope.** Only split your assigned file. Leave other oversized files alone,
  even when they're adjacent. Later phases own the other nine, and anything else is
  outside this epic. Touch other files only for the path or import updates your move
  requires.
- **`just all` passes** (`cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, and `cargo test`), with **no new clippy
  or compiler warnings**. Watch for `dead_code` and `unused_imports` warnings caused by
  moving items or `#[cfg(test)]`-only helpers. Run `cargo fmt` (the repo pins
  `max_width = 80` in `rustfmt.toml`).

### Conventions and gotchas

- **Module layout.** Turn `foo.rs` into a directory module. New directory modules should
  use `foo/mod.rs`, which matches `gkeep/` and `highlights_ref/`. `dataview` already
  uses the `dataview.rs` + `dataview/` style, so keep that style there.
  `git mv foo.rs foo/mod.rs` first (before editing), which helps `git log --follow`.
- **Unit tests.** Split each large `#[cfg(test)] mod tests` block by the behavior it
  covers. You can co-locate tests with the submodule they exercise
  (`#[cfg(test)] mod tests;` inside `foo/bar.rs` → `foo/bar/tests.rs`) or use a
  `foo/tests/` directory (`foo/tests/mod.rs` for shared fixtures plus
  `foo/tests/<area>.rs`). Choose whichever keeps things clearer. `use super::*;` in a
  test module only sees items that are nameable in its parent, so import from sibling
  submodules explicitly where needed. Top-level `#[cfg(test)]` helper functions (several
  files have them) move with the code they wrap, or into test support.
- **`include_str!` / `include_bytes!` paths are relative to the file that contains the
  macro.** Fix them whenever a moved test changes directory depth. Known cases:
  `tests/cli.rs` (5 × `fixtures/task_status_hooks/...`) and the `collect_done` unit
  tests (`../../tests/fixtures/collect_done/...`).
- **Module docs.** Keep the original `//!` docs on the parent module. Give each new
  submodule a one-line `//!` summary, matching the repo's sparse comment style.
- **References to old paths.** Update doc comments and docs that name the old file where
  they would now mislead. For example, `capture_language.rs` and
  `capture_task_toggle.rs` mention `capture.rs`, and `tests/randomize.rs` mentions
  `tests/cli.rs`. Use `grep -rn '<old file name>' src tests docs` to find others.
- **Final check.** `find src tests -name '*.rs' | xargs wc -l | sort -rn | head -40`.
  Confirm that every file your phase created or touched has at most 1500 lines.

## Phase 1 — Split tests/cli.rs

`tests/cli.rs` (~35.3k lines, 515 `#[test]`s) is one integration-test crate that runs
the built binaries (`env!("CARGO_BIN_EXE_*")`).

**Recommended target shape: one test binary, many modules.** Replace the file with
`tests/cli/main.rs`. Cargo auto-discovers `tests/<name>/main.rs` as the test target
`cli`, so `cargo test --test cli` still works and only one binary is linked. **Don't**
fan it out into ~25 separate `tests/*.rs` targets, because each would be its own crate
and binary and would multiply link time. `main.rs` should hold only `mod` declarations
(plus any crate-level attributes).

Rough landmarks at `d0c1692`, grouped by test-name prefix:

| Lines           | Content                                                                            | Suggested home                                                                    |
| --------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| 1–38            | imports, bin consts, `TEMP_COUNTER`, `LegacyHelpCase`                              | `support.rs`                                                                      |
| 27876–28880     | shared helpers (`bob_command`, `TempDir`, git/vault/PDF/plugin helpers, asserts)   | `support.rs`, or a `support/` dir if >1500 (e.g. `support/{mod,git,pdf,fs}.rs`)   |
| 6070, 6830–6850 | `write_priority_config`, `priority_scheduled_offset_days`, `date_offset_days`      | with the capture-priority tests                                                   |
| 33653–33740     | `close_worked_vault`, `run_close_json`, `run_close_expect_error`                   | with the Pomodoro-close tests                                                     |
| 40–900          | cache extraction, top-level and per-command help, legacy binaries, script fallback | `help.rs`                                                                         |
| 901–4110        | `task_status_*` (32 tests, ~3.2k lines)                                            | `task_status_hooks/` (2–3 files)                                                  |
| 4111–16750      | `capture_*` (~12.6k lines)                                                         | `capture/` (~9–11 files, below)                                                   |
| 16750–17750     | `dataview_*`                                                                       | `dataview.rs`                                                                     |
| 17752–19290     | `projects_*`                                                                       | `projects.rs` (split in two if it's close to 1500)                                |
| 19292–19950     | `plugins_*`                                                                        | `plugins.rs`                                                                      |
| 19953–25847     | `highlights_create_*`, `highlights_ref_*` (~5.9k lines)                            | `highlights/` (`create.rs` + ~4 ref files by scan/sync/doctor/intake/marker area) |
| 25847–26746     | `move_done_*`                                                                      | `move_done.rs`                                                                    |
| 26747–26960     | pomodoro / tmux / script pomodoro                                                  | `pomodoro.rs`                                                                     |
| 26961–27875     | `vault_sync_*`, conflict dirs, nightly, executable stubs                           | `vault_sync.rs` (+ `nightly.rs` if it helps)                                      |
| 28884–35334     | second capture block, mostly `capture_pomodoro_*` (~6.4k lines)                    | `capture/pomodoro_*.rs` (start, adjust/shift, link, close, …)                     |

Notes:

- The capture tests are **interleaved**. `capture_parse_*`, `capture_complete_*`, and
  `capture_task_*` show up in many separate ranges. Group them by the feature they
  exercise (parse/JSON contract, routing and scheduled/priority, project notes, rewrite,
  task toggles and sub-bullets, batch/global destinations, clip/history, authored/bare
  items, complete/sections/targets, and each Pomodoro operation), not strictly by
  prefix. Keeping the original relative order inside each destination file is fine.
  Nested directories (`tests/cli/capture/mod.rs` + children) are encouraged.
- Make helpers `pub(crate)` in `support` and import them with `use crate::support::*;`
  or explicit lists. Keep the `#[cfg(unix)]` / `#[cfg(not(unix))]` twin helpers
  (`assert_unix_mode`, `set_mode`) together.
- Fix the 5 `include_str!("fixtures/task_status_hooks/...")` paths for the new depth
  (for example `../fixtures/...` or `../../fixtures/...`). `fixture()` uses
  `CARGO_MANIFEST_DIR` and needs no change.
- `tests/randomize.rs` intentionally keeps its own copies of a few cli helpers. Don't
  make it share `tests/cli/support`. Only update its comments that cite `tests/cli.rs`
  (including `tests/cli.rs::init_vault_sync_pair`).
- Confirm that `cargo test --test cli -- --list` reports the same 515 tests as before,
  and that `cargo package --list` (the `include` globs cover `/tests/**`) still lists
  the new files.

## Phase 2 — Split src/native/capture.rs

The `bob capture` executor (~12.3k lines: code 1–9370, one test module 9371–12340).
Recommended: `src/native/capture/mod.rs` plus these submodules. Line ranges are from
`d0c1692`.

- `mod.rs`: `COMMAND_NAME`, `INBOX_FILE`, `run`, `capture` (the entry point, ~694),
  `mod` declarations, and re-exports. Keep
  `pub(crate) use super::capture_language::is_route_token` and every item imported by
  `capture_schedule_log.rs`, `capture_sections.rs`, `capture_work_log.rs`, and
  `randomize_plan.rs` reachable at `super::capture::…`.
- `cli.rs`: `print_clap_error`, `build_cli`, the `*_arg` builders, `OutputFormat`,
  `CaptureRequest`, and the `*_from_matches` / forced-destination helpers (71–694,
  ~620).
- `plan.rs`: `PlannedCaptureBatch` / `PlannedCaptureItem`, `plan_capture_batch`, and
  `plan_capture_item` (718–1380; the item planner alone is ~600 lines). Put the
  format/priority/date helpers (1692–1918) in `format.rs` if that keeps `plan.rs`
  comfortably small.
- `project_note.rs`: project-note conflicts, validation, planning, and its Pomodoro link
  (1381–1692).
- `batch.rs` (or keep it in `plan.rs`): `CaptureBatchPlanner`, the staged-file structs,
  `plan_capture_to_target`, and the shared plan/details/summary structs (1918–2311).
- `pomodoro_start.rs`: start range/planning, placeholder replacement, child links,
  placement scan, move-to-current-slot, and start conflicts/items (2311–2927,
  3505–3693).
- `pomodoro_adjust.rs`: toggle/adjust/shift conflicts, `RunningSessionTarget` /
  `select_running_session`, adjust and shift items (2927–3505), and the
  adjustment-range/duration parsing helpers (4956–5326).
- `pomodoro_close.rs`: `SnapshotCloseVault` and its `capture_pomodoro_close::CloseVault`
  impl, close timing/summary JSON builders, and close/link/task items (3693–4956,
  ~1260).
- `task_toggle.rs`, `ensure_next.rs`, `pomodoro_link.rs`: 5326–5570, 5570–5704, and
  5694–6561 respectively.
- `sub_bullet.rs`: note/sub-bullet capture, parent-section resolution, and managed
  Schedule/Work Log marker parsing and indentation (6561–7009).
- `commit.rs`: batch commit, clip saving, pending/applied text files, staged writes,
  rollback, and temp-file helpers (7009–7189, 7561–7645).
- `pomodoro_insert.rs`: `PomodoroSelection`, block-link/child-block insertion,
  named-Pomodoro selection and verification, and insertion-index helpers (7189–7561).
- `sections.rs`: task/bullet insertion, `MarkdownSection`, headings, frontmatter span,
  tasks section, fences, and line spans (7699–8216). Don't name it `markdown.rs`,
  because `super::markdown` already exists.
- `output.rs`: `Placement`, the Pomodoro close JSON structs, `CaptureResult`,
  `CaptureItemResult`, and all `print_*` human/JSON output (8216–9335, ~1120).
  `CaptureError` / `CaptureErrorKind` (9335–9370) can go here, in `error.rs`, or stay in
  `mod.rs`.
- The test-only parse wrappers (7645–7699) move beside the tests.
- Tests (~2970 lines): split into 2–3 files by area (for example planning and routing,
  Pomodoro operations, markdown placement and commit), or co-locate them with their
  submodules.

`capture_pomodoro_close.rs` is split later, in the last phase. Keep referring to
`capture_pomodoro_close::…` by its current path. Update the `capture.rs` mentions in
`capture_language.rs` (module docs, lines 5–13) and `capture_task_toggle.rs` (~278).

## Phase 3 — Split src/native/capture_language.rs

This is the pure capture grammar (no I/O, ~11.6k lines: code 1–7034, one test module
7035–11615 of ~4.6k lines). Recommended: `capture_language/mod.rs` plus submodules:

- `mod.rs`: the existing `//!` docs, re-exports, and the public model types (23–420:
  `CaptureKind`, the Pomodoro spec structs, `SessionOperator`, `ParsedCaptureText`,
  `Token` / `ParseToken`, and so on). Use `model.rs` if you want `mod.rs` to stay a thin
  root.
- `draft.rs`: physical-line and item splitting, authored-line classification, the
  `parse_capture_text*` / `parse_capture_draft*` entry points, and global-destination
  resolution, shadow warnings, and inheritance (423–921).
- `item.rs`: `parse_capture_item` (921–1222), sub-bullet kind resolution, session
  operator and equals tokens, and Pomodoro adjust/equals items (1222–1634).
- `line.rs`: `LineOutcome`, `resolve_line`, `AggregateMarkers`, and the `*_error`
  constructor functions (1634–1954).
- `tokens.rs`: route, selector, block-id, colon-link, Pomodoro-route/start, and caret
  token parsers and classifiers, including the caret item / Pomodoro link item parsers
  (1954–2940, ~1000).
- `markers.rs`: terminal marker extraction; schedule, priority, and clip tokens; legacy
  markers; and the `*_ERROR` message constants (2940–3265).
- `editor/`: span/diagnostic/editor model types (3265–3600), `parse_for_editor` and the
  per-item editor parsers (3601–5008, ~1400, which probably needs two files), and editor
  token classification / `marker_parse` / diagnostics (5008–5900).
- `completion.rs`: completion context and fields (5900–6512).
- `rewrite.rs`: draft rewrite, absorption, text edits, and deletion spans (6512–7035).
- Tests (~4.6k lines): split into 4+ files by area, for example draft/item parse, tokens
  and markers, editor diagnostics, and completion/rewrite.

Notes: `parse_capture_text_with_clip_control` (~657) is `#[cfg(test)]`-only.
`use super::{capture_clip, collect_done}` must still resolve (`collect_done` gets split
later). Update the module docs, which explain how `capture.rs` layers on top of this
module, to name the capture module's new layout.

## Phase 4 — Split src/native/highlights_ref/mod.rs

This is `bob highlights` (~10.2k lines: code 1–7682, tests 7683–10240). The directory
already exists with `create.rs` (~1.4k lines, which stays as-is). Thin `mod.rs` down to
a root:

- `mod.rs`: constants (32–120), `run` / subcommand dispatch (577–677), `mod`
  declarations, and re-exports. Put the shared model types (120–577: `Config`,
  `PdfMarker`, `ParsedNote`, `PdfTaskStatus`, the sync/scan plan structs,
  `CommandError`, and so on) in `model.rs`.
- `hooks.rs`: pre-scan hook configuration, running, and checking (677–845).
- `sync.rs`: `sync_pdf`, `scan_library`, `plan_pdfs`, `plan_pdf_sync`, annotation-task
  plan finalization, `execute_pdf_sync`, image asset writes, and the unchanged-for-write
  guards (845–1537).
- `report.rs`: the PDF sync, scan plan, and write reports; `ScanLine`; `ScanCounts`; and
  summaries (1537–2386, ~850).
- `doctor.rs` / `intake.rs`: `show_marker`, `doctor_vault`, library layout validation,
  xlib intake and moves, and PDF path collection (2386–2810).
- `guard.rs`: output/asset collision checks, `ensure_safe_to_write`, dirty-note
  allowances, and git status/HEAD helpers (2810–3164).
- `sidecar.rs`: wikilink/parent canonicalization, sidecar discovery and parsing, image
  annotations and assets, `render_sidecar_highlights`, callouts, and the line
  classifiers/strippers (3164–4251, ~1090). Split parse and render if it's close to the
  limit.
- `text.rs`: PDF text artifact cleanup and reflow (4251–4432).
- `annotation_tasks.rs`: candidates, sources, routes, identities, the processed-task
  index, PDF task lines, and status signals/conflicts (4432–5232, ~800).
- `projection.rs`: sync projection resolution and merge, and diagnostics (5232–5462).
- `cli.rs`: `print_config_report`, `build_cli`, and the arg builders (5462–5653).
- `note.rs`: the `Config` / `Prefer` / `SyncSource` / `MarkerValue` / `ParsedNote`
  impls, pipeline metadata, the default note body, the managed region, and tasks-section
  insertion (5653–6675, ~1020).
- `marker.rs`: PDF marker read/decode/write/parse/validate/render and inline value
  parsing (6675–7233).
- `frontmatter.rs` / `io.rs`: note read/parse, frontmatter entries, field predicates,
  projection snapshot JSON, path metadata, atomic write/copy, and string rendering
  (7233–7683).
- Tests (~2.6k lines): 2–3 files by area.

Watch the `#[cfg(test)]` helper functions near 3011, 4024, 4775, and 4782.

## Phase 5 — Split src/native/dataview.rs

This is `bob query` (~7.1k lines: code 1–6847, only ~240 lines of tests). The module
already uses the `dataview.rs` + `dataview/` layout with `index.rs`, `value.rs`, and
`tasks/`. **Keep `dataview.rs` as the root** and add new files beside the existing ones.
Don't clobber `index.rs` or `value.rs`, because `value.rs` already owns `DataviewValue`.

- Root `dataview.rs`: constants, `run`, `run_request`, `run_native`, the
  request/input/engine enums (1–300 minus the JS), `mod` declarations, and re-exports.
- `obsidian.rs`: `OBSIDIAN_EVAL_SCRIPT` (40–205), the Obsidian
  process/eval/failure/protocol parsing (293–470), and `ObsidianEvalRequest` /
  `ProtocolEnvelope` / `protocol_error` (5982–6100).
- `output.rs`: `emit_*_output`, warnings, JSON printing, and source path extraction
  (469–641). Also markdown table and task rendering and display helpers (1731–2030 and
  3362–3403), or give those their own `render.rs`.
- `native.rs`: native query model types (641–869), the `NativeVault` impl (869–1584,
  ~715; consider `vault.rs`), `NativeRow`, and `NativeSelect` (1584–1731).
- `eval.rs`: the `NativeExpression` / `NativeExpr` impls, `EvalContext`, `evaluate_call`
  dispatch, and value coercion helpers (2030–2570).
- `functions/`: the built-in function library (2570–4142, ~1570). Split it by family:
  links/embeds, numeric and aggregate, collections and lambdas (filter/map/reduce/…),
  strings and regex, and dates/durations/currency formatting, plus comparison and
  arithmetic (3885–4142).
- `lexer.rs` / `parser.rs`: `NativeLexer` (4142–4302) and `NativeParser` plus the
  expression helpers (4302–5344, ~1040).
- `sources.rs`: source tags, markdown path collection, frontmatter scalars, links,
  identifiers, and result path collection and grouping (5344–5982).
- `error.rs`: `DataviewError` and the output excerpts (6084–6336).
- `cli.rs`: `build_cli`, the arg builders, the `Request` / `QueryInput` / `DqlInput` /
  `TasksInput` / `OutputFormat` / `Engine` / `VaultConfig` impls, and path validation
  (6336–6848).
- Keep the small test module in the root, or move its tests beside the code they cover.

## Phase 6 — Split src/native/task_status_hooks.rs

This is `bob task-status-hooks` (~6.1k lines: code 1–4204, tests 4205–6107 with 60
tests). Recommended: `task_status_hooks/mod.rs` plus submodules. `randomize_plan.rs` and
`note_tasks.rs` import from this module, so keep those paths via re-exports. The sibling
`task_status_hooks_write.rs` is out of scope.

- `mod.rs`: constants, `run`, `print_clap_error`, `build_cli`, `OutputFormat`, and
  `Request` (1–240).
- `model.rs`: report, result, and plan structs/enums; `TasksSettings`; `TaskStatusType`;
  and the Pomodoro/link/reference models (240–673).
- `retry.rs`: daily anchor / note kind, `RetryEnv`, retry delay/logging/loop, and
  `run_with_retries` (673–973).
- `sync.rs`: `sync_task_statuses` (973–1465) and the dependency state/transition helpers
  (1465–1569).
- `pomodoro.rs`: `scan_pomodoros`, indentation, block link occurrences, and marker
  expectations (1569–1871), plus the empty-Pomodoro plan/apply (2417–2459).
- `settings.rs`: reading, parsing, and validating task settings (1871–2009).
- `structure.rs`: fenced/bullet block extents, duplicate-line removal, and the
  structural plan/apply (2009–2475).
- `parse.rs`: markdown file collection, task line/metadata parsing, dates, and block ids
  (2475–2773).
- `references.rs`: the archive reference catalog, `ReferenceContext`, resolution,
  dependency edges, reachability, and desired statuses (2773–3344).
- `compose.rs`: `compose_outputs`, grouping eligibility and reports, and
  `apply_guarded_outputs` (3344–3649).
- `output.rs`: `print_*`, `SyncError`, and `print_error` (3649–4205).
- Tests (~1.9k lines): 2 files by area (for example sync/reference behavior and
  structural/pomodoro/grouping behavior).

## Phase 7 — Split src/native/projects.rs

This is `bob projects` (~4.7k lines: code 1–3227, tests 3228–4652 with 53 tests).
Recommended: `projects/mod.rs` plus submodules:

- `mod.rs`: constants, `run`, the CLI builders, and `run_list` / `run_sync` (1–182).
- `model.rs`: `ScanReport`, `Project`, schedules, `PrjTask`, subproject state, and the
  frontmatter types (182–628).
- `scan.rs`: directory scanning (628–761) plus project, schedule, frontmatter, wikilink,
  prj sub-block, and task line parsing (2178–2633).
- `sync.rs`: `SyncReport`, `SyncFile`, `SyncEvent`, `ProjectPlan`, and `ProjectChange`;
  the sync directory walk; `plan_project_sync_at`; and subproject entry/display planning
  (761–1493). The `#[cfg(test)]` `plan_project_sync` wrapper (~1190) goes with the
  tests.
- `edits.rs`: `apply_project_changes`, task schedule/hide/status edits, subprojects line
  rendering, and frontmatter layout (1493–2178).
- `tags.rs`: tag spans, hide tags, inline fields, and description/block-id stripping
  (2633–2864).
- `output.rs`: the project list and sync report, the `SyncEvent` display impl,
  `Summary`, `ProjectStyleExt`, and name/path helpers (2864–3228).
- Tests (~1.4k lines): split into 2 files for headroom.

## Phase 8 — Split src/native/collect_done.rs

This is `bob move-done-tasks` (~4.3k lines: code 1–2564, tests 2565–4344 with 67 tests).
Recommended: `collect_done/mod.rs` plus submodules. `capture_language.rs` imports this
module, so keep its paths stable.

- `mod.rs`: constants, `DEPENDENCY_FIELD_RE`, `Args`, `run`, `run_collection`, output
  merging, and arg parsing/help (2465–2565).
- `plan.rs`: the `CollectionPlan` / `FilePlan` / `LinkRepairPlan` / write / git-state
  types and impls (29–288), apply file/link plans (492–528), `build_collection_plan`,
  and dependency identity validation (1047–1248).
- `git.rs`: prepare/detect/touched paths/finish/stage/commit/push and the commit message
  (528–718).
- `archive.rs`: archive block-id deduplication, archive/source frontmatter, append, and
  atomic writes (718–1047).
- `link_repair.rs`: moved-block targets, `NoteIndex`, link repair in plans and notes,
  dependency metadata repair, and wiki/markdown link rewriting (1248–2096, ~850).
- `transform.rs`: markdown file collection, archive paths and wiki links,
  `transform_markdown`, block-id scanning, and collectible task lines (2096–2465).
- Tests (~1.8k lines): 2 files. **Fix the `include_str!` fixture paths**
  (`../../tests/fixtures/collect_done/...`) for the new directory depth.

## Phase 9 — Split src/native/task_status_groups.rs

This is the task status grouping transform (~3.0k lines: code 1–2021, tests 2022–2991
with 36 tests). It's only moderately over the limit, so prefer a light split of about
3–5 files. Don't fragment it. Recommended: `task_status_groups/mod.rs` plus submodules:

- `mod.rs`: title/marker constants, `StatusBucket`, `Destination`, `GroupKind`, the skip
  codes, report structs, `TaskClassification`, `transform`, and the container rewrite /
  child classification (1–713, roughly 700–800 lines).
- `parse.rs`: group markers, badge audit, container regions, direct pieces, task blocks,
  subtrees, and list item parsing (713–1367).
- `emit.rs`: the grouping plan, grouped body / badges / anchors emission, blank-line
  normalization, source lines, and the heading tree (1367–2021).
- `tests.rs` (~970 lines).

## Phase 10 — Split src/native/capture_pomodoro_close.rs

This is the Pomodoro close planners (~3.0k lines). Layout at `d0c1692`:

- 1–1207: the ledger close planner. It covers running Pomodoro discovery, close timing,
  ledger link classification (545–683), wikilink / marker / link-target parsing
  (683–1033), and Work Log note grouping (1033–1207).
- 1209–1681: `mod tests` (~470 lines).
- 1684–2629: the linked-task close planner. It covers `CloseTaskRole`,
  `PomodoroClosePlan`, the `CloseVault` trait, `ClosePlanner` (1771–2391),
  `plan_pomodoro_close`, `lookup_task`, and completion-field helpers.
- 2630–2982: `mod linked_task_tests` (~350 lines, with a `MemoryVault` test double).

Recommended: `capture_pomodoro_close/mod.rs` (shared constants, re-exports) +
`ledger.rs` (the ledger planner) + `links.rs` (wikilink / marker / link-target parsing,
if `ledger.rs` would otherwise be crowded) + `linked_tasks.rs` + `tests.rs` +
`linked_task_tests.rs`. The capture module implements
`capture_pomodoro_close::CloseVault` for its `SnapshotCloseVault`. Keep that trait path
and every other item capture uses reachable via re-exports.
