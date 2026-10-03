---
tier: tale
title: Split clipboard capture into focused Rust modules
goal:
  Complete bob-cli-3s.4 with cohesive clipboard capture modules under 1500 lines while
  preserving behavior and existing coverage.
size: medium
proposed_by: bbugyi200.athena.bob-cli-3s.4
bead: bob-cli-3s.4
create_time: 2026-10-03 06:48:47
status: wip
---

- **PARENT:**
  [202610/split_largest_rust_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_rust_files.md)
- **BEAD:**
  [bob-cli-3s.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3s/bob-cli-3s.4.md)

# Split clipboard capture into focused Rust modules

Complete phase bead `bob-cli-3s.4` from `plan:202610/split_largest_rust_files.md`. This
is a behavior-preserving structural refactor of `src/native/capture_clip.rs`, including
its existing tests. Keep every resulting Rust file at most 1500 physical lines after
formatting. Implement this tale as one bounded work unit; do not split it into another
epic.

## Inspected baseline and scope

The planning checkout is clean at `b4a5022515e08c764299614c1c473a5199867608`. It
includes the three preceding phase commits, most recently the focused
`task_status_hooks_write` facade and child modules. `capture_clip.rs` remains 2239
lines: 1394 lines before the outer test module and an 845-line test section. Source
inspection finds 19 unit tests, 15 clip integration tests on Linux (one is excluded on
macOS), and 11 batch integration tests. These are source counts, not claimed test
executions; capture executable discovery and focused outcomes before implementation.

The current callers are:

- `src/native/capture/plan.rs`: one `ClipReservations` instance across the whole batch,
  clipboard reads, individual/history planning, `ClipPlan.output`, and cloning output.
- `src/native/capture/commit.rs`: `ClipPlan::save`, cleanup of files from earlier batch
  items, and cleanup if note writes fail.
- `src/native/capture/output.rs`: serialized `ClipOutput` and file confirmations.
- `src/native/capture/cli.rs` and `src/native/capture_language/markers.rs`: header
  validation.
- `src/native.rs`: the existing `mod capture_clip;` declaration.

Preserve these entry points and signatures. The clipboard/JSON contracts in
`docs/capture.md` remain unchanged. No CLI options, dependencies, documentation
contracts, plugin code, Mac client code, or memory files need changes. Keep changes
outside this component limited to wiring that the compiler actually requires.

An initial `sase bead epic-symbols bob-cli-3s.4` found no entries; check again
immediately before closure. The current `justfile` supplies `just all` (fmt, clippy,
test) and has no `--epic-symbol` line.

## Selected file layout and dependency direction

Retain `src/native/capture_clip.rs` as the single module root. Create the children below
under `src/native/capture_clip/`; do not also create a competing `mod.rs` root. Counts
are approximate placement budgets, not targets to obtain by squeezing formatting.

| File              | Responsibility                                                                              | Approximate lines |
| ----------------- | ------------------------------------------------------------------------------------------- | ----------------: |
| `capture_clip.rs` | Small facade, child declarations, explicit existing API re-exports                          |             40–70 |
| `model.rs`        | Output/plan models, planned-file kinds, and shared reservation state                        |           140–180 |
| `clipboard.rs`    | Live providers, command execution/errors, normalization, history override and merge         |           240–280 |
| `clipy.rs`        | Read-only SQLite history, schema validation, ordered asset/plist decoding                   |           220–250 |
| `plan.rs`         | Entry/history planners, classification dispatch, list/structure decisions and thresholds    |           250–300 |
| `render.rs`       | Header validation/rendering, child-line indentation, inline and line outputs                |            75–100 |
| `files.rs`        | Path/file-URI classification, attachment/snippet planning, destination selection and naming |           420–470 |
| `persist.rs`      | Plan saves, exclusive temporaries, rollback cleanup/messages, shared temporary counter      |           140–170 |
| `tests.rs`        | All 19 current tests, wrappers, temporary fixtures, and Clipy fixture helpers               |           850–900 |

Splitting rendering into a small dedicated module allows both content planning and file
planning to reuse the same formatting without a `plan`/`files` cycle. Likewise, keep
reservation types in `model.rs`, rather than making persistence depend on file-planning
internals. Production dependencies should flow as follows:

- `plan` uses `model`, `files`, and `render`.
- `files` uses `model`, `render`, and the existing `native::env` helpers.
- `render` and `persist` use `model`.
- `clipboard` uses `clipy` only through the macOS history path.
- `clipy` is independent of planning and persistence.

Use explicit imports in each child, adjusting the original `super::env` import to
`crate::native::env as bob_env` where needed. The root should expose the existing
crate-visible types (`ClipMode`, `AttachmentKind`, `AttachmentOutput`, `ClipOutput`,
`ClipPlan`, `ClipReservations`) and public-in-crate functions through explicit
re-exports. Use those names internally where that avoids unused facade re-exports.
Retain the test-only `plan` and `plan_history` wrappers and their original signatures
under `#[cfg(test)]`; they are not new production APIs.

Internal planned-file/reservation types, fields, and helpers shared between children
need at most `pub(super)`, not `pub(crate)` or `pub`. Keep `FileReservations`' lookup
map private and its existing methods with the type. Expose only the reservation vector
and fields needed by the unchanged planners, persistence, and tests to this component.
Keep `ClipOutput::file_confirmations` with the output type; implement `ClipPlan::save`
in `persist.rs` without changing the plan's data model.

## Implementation steps

1. Re-read `bob-cli-3s.4` and its epic artifact with audited commands. Inspect the
   current tree and counts before adopting the layout; do not undo another phase's work.
   Read the applicable SASE bead/artifact memory through `sase memory read`. Record the
   base revision and pre-existing changes, if any. Capture the baseline discovery and
   focused test outcomes listed below before moving production or test code.
2. Extract the shared models and reservations, then rendering and file planning. Move
   complete definitions and preserve comments/derives/serde attributes. Keep numeric
   thresholds beside the decisions that use them. Move path classification and URI
   decoding into `files.rs`; retain their filesystem metadata reads. Planning is free of
   clipboard commands and writes, but still performs the same file inspections and
   attachment reads as before.
3. Extract provider access and Clipy decoding. Declare the whole `clipy` child under
   `#[cfg(any(target_os = "macos", test))]`. Preserve existing function/import gates and
   macOS-only call sites so non-test Linux builds do not require macOS production
   dependencies. Leave the target-specific and dev dependencies in `Cargo.toml` intact.
4. Extract persistence, retaining exactly one `TEMP_COUNTER` instance. Place it in
   `persist.rs` with component-scoped visibility; the moved test fixture imports that
   same counter rather than creating another. Keep the save loop and all error/cleanup
   paths intact. Add the facade's explicit re-exports and retain the current root.
5. Move the whole test body to `tests.rs` under the root's `#[cfg(test)] mod tests;`.
   Keep test function names, assertions, helper behavior, and fixture cleanup. Replace
   reliance on the old broad parent imports with explicit standard-library, chrono,
   rusqlite, and component imports. Private helpers tested from this sibling can be
   `pub(super)`; if used only by tests, gate any helper visibility/import accordingly.
   Keep the convenience wrappers calling the parent test-only planners. The 845-line
   suite already has ample headroom, so further test subdivision is unnecessary.
6. Format, compile, and resolve only relocation/import/visibility regressions. Run the
   focused checks and full repository checks. Compare test discovery and review the diff
   for changes to literals, control flow, cfg gates, serialization, or assertions. Add a
   new regression test only if the split reveals a material missing behavior check; do
   not add tests of module organization.
7. Record the final file map/counts, baseline versus final test discovery, commands and
   results, and actual platform coverage on `bob-cli-3s.4`. Complete the symbol audit
   and close only this phase as described below.

## Behavior that must survive the move

- Provider precedence: a nonblank `BOB_CLIPBOARD_CMD` override wins; macOS uses
  `pbpaste`; Linux checks Wayland before X11; `xsel` replaces `xclip` only when the
  executable is missing, not when it exits unsuccessfully; non-macOS uses tmux only
  after applicable display providers. Keep command argument splitting, error text,
  stderr detail, and exit handling unchanged.
- History: one entry reads only the live clipboard; larger counts use the configured
  history command or macOS Clipy database. Normalize each value, remove only the first
  matching live candidate, preserve later duplicates/newest-first order, and reject
  insufficient or invalid entries with the same indexed errors.
- Clipy: read-only SQLite flags, required column names/types, history ordering by
  `updateAt DESC, id DESC`, asset ordering by `index ASC, id ASC`, UTF-8/plist fallback,
  legacy filename arrays, and binary-only representation errors remain unchanged. Linux
  fixture tests must still compile the Clipy/plist path.
- Content: header grammar/casing, flat unordered list recognition, all length/line and
  attachment thresholds, structural-text fallback, file URI/quote/tilde handling,
  missing-path errors, and tab/two-space rendering remain unchanged.
- Files/reservations: preserve image/file selection and 400px embeds, sanitized names,
  eight-character hashes, content reuse and collision refusal, snippet timestamps,
  Unicode slug limits, final snippet newline, numeric collision suffixes, and
  attachment/snippet kind distinctions. Preserve one reservation object across every
  batch item and the per-plan vector slice from `start` onward; a later plan must not
  save files owned by an earlier plan. History entries share the same reservations.
- Output: preserve lowercase enum serialization, nullable header, required `lines`,
  `attachments`, and `entries` even when empty, omitted absent `snippet`, history
  aggregation order, and deduplicated file confirmations. The comment explaining Mac
  client's required collection fields stays with `ClipOutput.entries`.
- Persistence: keep reused-file skipping and destination deduplication, create-new
  temporary allocation with the same counter/retry limit, write and sync order,
  destination-appeared refusal, rename behavior, temporary cleanup, reverse-order
  rollback, ignored NotFound errors, and exact cleanup messages. Do not redesign the
  save algorithm or alter which directories remain on failure. Remove only files created
  by the failed capture; dry-run must still perform no writes.

## Verification and handling unrelated failures

Run these discovery and execution commands on the clean implementation baseline, and
again after extraction. Save their output in disposable logs or bead notes; do not
commit generated logs. Each filter must execute relevant nonzero tests:

```sh
cargo test --lib capture_clip -- --list
cargo test --test cli capture::clip -- --list
cargo test --test cli capture::batch -- --list
cargo test capture_clip
cargo test --test cli capture::clip
cargo test --test cli capture::batch
```

Expect all 19 original unit tests, 15 Linux clip tests (14 on macOS), and 11 batch tests
at the inspected baseline. `cargo test capture_clip` also matches some CLI function
names; compare the `--lib` inventory separately. Source and executable test counts may
legitimately differ by platform gates; explain any difference and compare the same
target before and after. With the single `tests.rs` module, the existing
`native::capture_clip::tests::*` unit paths should remain stable.

Use existing command overrides, temporary vaults/files, and SQLite fixtures only; do not
test with the real clipboard or Bryan's vault. The existing suites cover required JSON
collections, normalization/history errors, dry runs, attachment reuse/collisions,
snippet counters, cross-kind reservations, and cleanup after a later save failure.

After the final edit, run:

```sh
cargo fmt
just all
git diff --check
```

`just all` must actually run formatting checks,
`cargo clippy --all-targets --all-features`, and the full `cargo test`. Use
`/sase_monitor` for any command requiring a long-running handoff; wait for the handoff
command itself to exit before ending its turn. Report checks actually run. On Linux,
production provider branches and the test-gated Clipy helpers are compiled/tested; macOS
`pbpaste`, automatic Clipy path selection, and other OS-only branches receive cfg/import
inspection unless the target and runner are available. Do not claim native macOS
execution from Linux fixtures.

Count physical lines with this audit after formatting; every listed file must be present
and at most 1500 lines, including tests:

```sh
python3 - <<'PY'
from pathlib import Path
root = Path('src/native')
paths = [root / 'capture_clip.rs', *(root / 'capture_clip').rglob('*.rs')]
assert (root / 'capture_clip.rs').is_file()
assert not (root / 'capture_clip/mod.rs').exists()
for path in sorted(paths):
    count = len(path.read_bytes().splitlines())
    print(f'{count:5} {path}')
    assert count <= 1500, path
PY
```

The preceding phase recorded a pre-existing process-global `BOB_DAY_FILE` race in
`capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
(`bob-cli-3s.3`, notes 1 and 3): it failed intermittently under default full-suite
parallelism and passed in isolation and with four test threads. This is prior evidence,
not permission to assume any failure has the same cause. If a check fails, retain the
exact diagnostic and reproduce it on the unchanged base without disturbing the
implementation. A failure reproduced identically on the clean base does not keep this
phase open; append `PROPOSED FOLLOW-UP: <summary — reproduction/evidence>` to
`bob-cli-3s.4`, citing any existing tracking bead or the preceding phase's evidence.
Record unresolved flakes and platform limits precisely; fix failures caused by this
refactor. Do not add unrelated environment-lock/lint cleanup or create task beads.

## Phase completion

After all work and verification, run:

```sh
sase bead epic-symbols bob-cli-3s.4
```

Resolve every remaining symbol, or re-key its `justfile` line to a still-open parent
epic or later phase when that outstanding work belongs there. Do not close while any
entry remains keyed to this phase. Record the before/after file map/counts, discovered
test preservation, commands/results, limitations, and follow-ups with
`sase bead note bob-cli-3s.4` as needed, then run:

```sh
sase bead close bob-cli-3s.4 --note "<final file counts, preserved tests, checks and limitations actually verified>"
```

Never hand-set this bead's status. Close only `bob-cli-3s.4`; do not close the parent
`bob-cli-3s` or an ancestor plan bead. Leave cumulative epic landing and phase 5 to
their owners. Do not invoke a manual Git commit skill unless explicitly instructed;
follow the required `/sase_final` declaration for any normal response ending the coding
turn.
