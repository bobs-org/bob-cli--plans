---
tier: tale
size: medium
title: Split the projects command into cohesive Rust modules
goal: Keep the bob projects command and its public native-module API unchanged while
  splitting its 4,652-line implementation and 53 unit tests into files of at most
  1,500 lines.
proposed_by: bbugyi200.apollo.bob-cli-2f.7
bead: bob-cli-2f.7
status: done
---

- **PARENT:**
  [202609/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-2f.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2f/bob-cli-2f.7.md)

# Scope

Complete assigned phase `bob-cli-2f.7` of the approved epic design
`plan:202609/split_largest_rust_files.md`. This phase owns only
`src/native/projects.rs`. It is a structural refactor: retain the CLI contract,
diagnostics, sorting, Markdown edits, formatting, and all test assertions. The existing
`tests/cli/projects/` integration tests stay in place.

# Implementation

1. Record the starting 4,652 source lines and 53 `#[test]` functions. Review the actual
   top-level item boundaries before moving code; the line ranges below describe the
   current file and are guides rather than fixed cut points. Use
   `git mv src/native/projects.rs src/native/projects/mod.rs` first.
2. Keep the constants, clap builders, `run`, `run_list`, `run_sync`, and
   `bob_dir_from_matches` in `mod.rs`. Give each child a short module doc line. Move
   code in source order within these cohesive groups, with no algorithm or assertion
   changes:

   | File                 | Responsibility                                                                                            | Current source landmarks                                   |
   | -------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
   | `projects/model.rs`  | Scan and project structs, status and task models, subproject and frontmatter types                        | 182–627                                                    |
   | `projects/scan.rs`   | Directory scan and project/frontmatter/task/wikilink parsing; file and directory helpers used by scanning | 628–717, 2178–2632, 3209–3227                              |
   | `projects/sync.rs`   | Sync report, event/change models, sync walk, project planning, subproject aggregation                     | 718–1492, except the test-only `plan_project_sync` wrapper |
   | `projects/edits.rs`  | Apply planned edits, schedule/status/subproject rendering, frontmatter and line layouts                   | 1493–2177                                                  |
   | `projects/tags.rs`   | Tag and inline-field spans, descriptions, block IDs, whitespace helpers                                   | 2633–2863                                                  |
   | `projects/output.rs` | List/report printing, event rendering, style extension, names and display paths                           | 2864–3208                                                  |

   Keep any helper with its callers when that avoids unnecessary coupling. Use narrow
   sibling visibility (`pub(super)` or the narrowest required scope); preserve existing
   `pub(crate)` visibility only for the API already called by `capture_targets`,
   `capture_active_tasks`, and `task_status_hooks`. Re-export `ProjectStatus`,
   `Frontmatter` if needed, `parse_frontmatter`, `frontmatter_is_project`,
   `frontmatter_is_area`, `frontmatter_value`, `trim_yaml_scalar`, and
   `is_markdown_file` through `projects/mod.rs` so the existing `pub(crate)` API retains
   its current paths. Organize imports per module without adding warnings.

3. Replace the 1,425-line inline test block with `projects/tests/mod.rs` for shared
   fixtures and three files holding the same 53 tests: `tests/parse.rs` (initial
   parser/tag tests, about lines 3284–3537), `tests/sync.rs` (planning/subproject tests,
   about 3538–4179), and `tests/edits.rs` (edit/schedule/frontmatter tests, about
   4180–4652). Move the test-only `plan_project_sync` wrapper to test support. Keep
   every test name, body, and assertion. Explicit imports or narrow `pub(super)` access
   may be needed after `use super::*` changes scope.
4. Run `cargo fmt`. Count lines in every touched Rust file; re-cut a cohesive module if
   any file exceeds 1,500 lines, preferably leaving growth room. Verify all 53 unit test
   functions remain and the integration tests still compile and run. Run `just all`
   (`fmt`, clippy, and `cargo test`) and fix any warnings caused by the move. Confirm
   `git diff` contains only the intended refactor and required path/import adjustments.
5. Run `sase bead epic-symbols bob-cli-2f.7` before closure. Resolve every remaining
   symbol or re-key its Justfile line to the parent epic or a later open phase. Record
   any unrelated clean-base check failure as `PROPOSED FOLLOW-UP:` on this phase rather
   than leaving it open. Close only `bob-cli-2f.7` with
   `sase bead close bob-cli-2f.7 --note "<verified results>"`; do not close the parent
   epic or set this phase's status by hand. Report the final file layout and line count
   for each created or touched file.

# Acceptance

- Every resulting projects source or test file is at most 1,500 lines.
- The same 53 unit tests are present, all existing tests pass, and `just all` reports no
  new compiler or clippy warnings.
- Outside users of `native::projects` still compile through their current paths.
- `sase bead epic-symbols bob-cli-2f.7` has no entries when the phase closes.
