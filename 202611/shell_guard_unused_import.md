---
tier: tale
title: Fix the macOS-only unused AsRawFd import in the capture shell guard
goal:
  macOS builds of bob-cli compile with no unused-import warning, and master CI's macOS
  lint step passes again.
size: small
decisions:
  macos_gate:
    ask:
      Should the coder prove the fix on a real macOS runner with
      scripts/dev/macos-ci-check before finishing?
    choices:
      run:
        Push a throwaway ci/precommit branch and wait up to ~90 min for real macOS lint
        proof
      skip: Finish after Linux checks; the next master CI run is the first macOS proof
    default: run
    why:
      Linux never compiles this cfg branch; macOS lint has hidden its tests for 20+
      master pushes
    answer: skip
proposed_by: bbugyi200.athena.17x
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 09:39:50
status: wip
---

# Fix the macOS-only unused `AsRawFd` import in the capture shell guard

## Problem

On macOS, every build of bob-cli prints this warning, for example from
`cargo install --profile local-install`:

```
warning: unused import: `std::os::unix::io::AsRawFd`
   --> src/native/capture/shell_guard.rs:124:13
```

It is more than noise. CI's `test (macos-latest)` job runs `just lint`, whose
`rustc_gate` passes `-D unused`. That turns this warning into a hard error
(`-D unused-imports` implied by `-D unused`). Since commit e00f9074 ("feat(capture): add
transfer lexer and shell redirect guard"), the "Lint macOS-only targets" step has failed
on every master push. On the latest completed run (41839828207, cb9dd8f9), this import
is the only error; the other ~450 diagnostics are advisory Clippy warnings. Because lint
fails first, "Run tests" and "Run archive-move presenter cases" are skipped on macOS.
Earlier master runs failed macOS lint for other reasons that have since been fixed, so
macOS tests have not run for 20+ master pushes.

## Root cause

`fd_is_regular_file()` in `src/native/capture/shell_guard.rs` has a
`#[cfg(target_os = "macos")]` branch that imports `std::os::unix::io::AsRawFd`. That
branch then calls `libc::fstat(1, &mut stat)` with a literal file descriptor, so the
trait is never used. The branch is compiled only on macOS, so `just check` on Linux
(where the change was written) never saw it.

## Changes

Edit only `src/native/capture/shell_guard.rs`, inside `fd_is_regular_file()`:

1. Delete the line `use std::os::unix::io::AsRawFd;` from the
   `#[cfg(target_os = "macos")]` block. Keep the literal `1` in `libc::fstat(1, ...)`.
   It matches the Linux branch (`/proc/self/fd/1`) and `macos_stdout_path()`
   (`libc::fcntl(1, ...)`), which both address fd 1 directly. Do not switch to
   `std::io::stdout().as_raw_fd()`. Removing the import cannot change behavior, because
   rustc already reports the trait as unused.
2. In the same function, change the tail of the
   `#[cfg(not(any(target_os = "linux", target_os = "macos")))]` block from
   `return false;` to the tail expression `false`. This is the same kind of hidden
   problem: `clippy::needless_return` is denied in the justfile's `clippy_gate`, but
   this branch is compiled on no CI platform. Leave the early `return false;` inside
   `if ret != 0 { ... }` in the macOS branch alone; it is a real early return.

Make no other changes. Do not deduplicate the repeated `starts_with_at` logic, edit
other files, change `Cargo.toml` or `CHANGELOG.md` (the release train owns them), or
touch docs. This is not a user-visible CLI or JSON contract change.

Suggested commit subject: `fix(capture): drop unused AsRawFd import in shell guard`.

## Verification

### Linux (always)

- `cargo fmt --all -- --check`
- `just lint`, which must pass with the same gate as CI.
- `cargo test --lib shell_guard` for the module's unit tests. Running the full
  `just check` is fine too, but it does not exercise the macOS branch.
- `git diff` shows only the two edits above.

Note: a Linux cross-check (`cargo check --target aarch64-apple-darwin`) cannot verify
the macOS branch here. The target is installed, but the `rquickjs-sys` build script
needs a macOS C cross-compiler (`cc: error: unrecognized command-line option '-arch'`).
Do not try to set up a cross toolchain.

> [!decision] macos_gate = run After the Linux checks pass, use your `/sase_monitor`
> skill to run `scripts/dev/macos-ci-check` from the repo root through
> `sase monitor start` with profile `verify` and a timeout longer than 92 minutes (for
> example `--timeout 95m`), as `docs/develop/ci.md` ("Precommit macOS check")
> prescribes. The script pushes a throwaway snapshot to `refs/heads/ci/precommit/<id>`,
> waits for that SHA's `ci.yml` run, prints each macOS step's verdict, and deletes the
> branch. Give `--next` the follow-up: inspect the verdicts and record the evidence.
>
> Success for this fix means the "Lint macOS-only targets" step passes and its log no
> longer reports `shell_guard.rs`. Read the result as follows:
>
> - **Lint passes, everything else passes:** done.
> - **Lint passes, but "Run tests", "Run archive-move presenter cases", or a non-macOS
>   job fails:** these steps have been skipped on macOS for 20+ pushes, so such failures
>   existed before this fix and are not caused by it. Do not widen this tale to fix
>   them. Use `/sase_new_task` to file each distinct failure as a `ci` task bead, citing
>   the run URL and failing test names, then finish.
> - **Lint still fails:** if the error is another macOS-only denied lint in code that
>   was hidden behind the lib failure (for example in `tests/cli/capture/clip.rs`,
>   `tests/cli/highlights/*.rs`, or a bin), fix it the same minimal way, then rerun the
>   gate once. If it is anything larger, file a `ci` bead and report it.
> - **Exit code 2 (could not finish: deadline, no matching run, missing prerequisites,
>   or cleanup failure):** report the script output faithfully, including any recovery
>   command it printed for an undeleted ref. Do not claim the fix is verified on macOS.

> [!decision] macos_gate = skip Finish after the Linux checks. In the final report, say
> that the macOS branch was verified by reasoning only (the compiler itself reported the
> import as unused), and that the "Lint macOS-only targets" step of the next master CI
> run is the first real macOS proof.

## Expected outcome

- Building on the Mac (`cargo install --profile local-install`, `cargo build`) no longer
  prints the `AsRawFd` warning.
- Master CI's "Lint macOS-only targets" step stops failing on this import, so the macOS
  test steps run again.

## Out of scope

- A Linux-side macOS lint gate (cross-target Clippy). The `rquickjs-sys` C build blocks
  it, and the existing `scripts/dev/macos-ci-check` already covers this need.
- The ~450 advisory Clippy warnings, which stay at their default levels by policy.
- Any failures in the macOS test suite that this fix lets run again. File them as `ci`
  beads (see above).
