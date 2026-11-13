---
tier: tale
title: Remove the macOS-dead command_on_path from desktop_notify
goal:
  macOS builds of bob-cli no longer warn (and macOS `just lint` no longer errors) that
  desktop_notify::command_on_path is never used, because the duplicate helper is gone
  and the Linux notify-send lookup uses the shared bob_env::find_on_path.
size: small
proposed_by: bbugyi200.athena.1c9
proposed_with: claude/opus
create_time: 2026-11-13 17:40:39
status: wip
---

# Remove the macOS-dead `command_on_path` from `desktop_notify`

## Problem

On macOS, every build of bob-cli warns:

```
warning: function `command_on_path` is never used
   --> src/native/desktop_notify.rs:116:15
```

The macOS CI step `Lint macOS-only targets` runs `just lint`, which builds with
`-D unused`. That turns the warning into a hard error:
`error: function command_on_path is never used` (CI run 43513864920, master `21d12f23`).
`release.yml` calls `ci.yml`, so the daily release train cannot ship while this error
remains. Epic `bob-cli-kx` (phase `kx.5`, live Gmail-archive enable) is waiting for that
release.

### Root cause

Commit `cd6aefae` (`bob-cli-kx.3.1.4`) moved `command_on_path` out of
`src/native/vault_sync.rs` into the new shared `src/native/desktop_notify.rs`. In
`vault_sync.rs` the function carried `#[cfg(not(target_os = "macos"))]`. The move
dropped that attribute and widened the function to `pub(crate)`. Its only caller is the
`#[cfg(not(target_os = "macos"))]` branch of `notification_command`, so on macOS the
function compiles with no caller.

### Is anything planning to use it?

No. The only in-progress epic that touches this module is `bob-cli-kx`. Its one
remaining phase, `kx.5`, is live enable and verification: chezmoi config, a released
build, and gated live sync. It adds no Rust callers. No plan file under the plans store
mentions `command_on_path` or `desktop_notify`, and no bead mentions `command_on_path`.
`pub(crate)` was incidental to the move, not a reserved API. Removing the function is
safe.

## Change

Delete the duplicate helper and use the crate's existing shared PATH lookup,
`crate::native::env::find_on_path` (already imported here as `bob_env`). It is used by
`gog`, `xelatex`, `kpsewhich`, `curl`, and `uv` lookups.

All edits are in `src/native/desktop_notify.rs`:

1. In `notification_command`, inside the `#[cfg(not(target_os = "macos"))]` block,
   replace

   ```rust
   if !command_on_path("notify-send") {
       return None;
   }
   ```

   with a `bob_env::find_on_path("notify-send")` check that returns `None` when the
   lookup misses (e.g. `bob_env::find_on_path("notify-send")?;`, or an explicit
   `is_none()` guard if that reads better). Keep the returned program as the bare
   `"notify-send"` string, not the resolved path, so the argv contract and the existing
   test stay unchanged.

2. Delete `pub(crate) fn command_on_path` entirely. Do not re-add it behind a `cfg`
   gate: the shared helper replaces it.
3. Update the `notification_command` doc comment and the module doc (lines 5-6) to say
   Linux needs an executable `notify-send` on `PATH`. `find_on_path` checks the
   executable bit (`is_executable_file`), while the old helper checked only `is_file`.
   The user-visible result doesn't change: a non-executable `notify-send` was already
   un-spawnable, and `send_notification` swallows spawn failures. Now bob just skips the
   spawn attempt.
4. Leave `osascript_escape`'s existing
   `#[cfg_attr(not(target_os = "macos"), allow(dead_code))]` alone. It is correct. Check
   that `bob_env` is still used in non-test macOS builds after the deletion
   (`notifications_enabled` uses `bob_env::var`), so no new unused-import warning
   appears on either platform.

### Test

Extend the existing `#[cfg(not(target_os = "macos"))]` unit test
`linux_builds_notify_send_argv`, or add a sibling test gated
`#[cfg(all(unix, not(target_os = "macos")))]`. It should pin the stricter lookup: a
`notify-send` file on the guarded `PATH` without the executable bit yields `None`, and
the same file with mode `0o755` yields the `notify-send` argv. Use the same
`bob_env::TestEnvGuard` pattern the existing test uses. The existing missing-`PATH` and
present-and-executable assertions must keep passing unchanged.

## Out of scope

- The second hard error in the same macOS lint run,
  `error[E0425]: cannot find function ptsname_r in crate libc`, is in
  `src/native/gcal/ack.rs:2854` and `tests/gcal_ack_tty.rs:86`. It comes from `307616ae`
  (`bob-cli-l5.3.1.3`) and is recorded as a `DISCOVERED ISSUE:` note on the active epic
  `bob-cli-l5`. Do not fix it here.
- The Ubuntu test failure
  `native::gcal::cli::tests::sync_args_read_last_n_days_aliases_and_default_to_none` is
  tracked by `bob-cli-l2`.
- The roughly 670 advisory Clippy warnings in the macOS lint log are default-level
  lints, not part of the `-D` gate.

## Verification

1. `rg -n command_on_path src tests` returns nothing.
2. `cargo test --lib desktop_notify` passes on Linux, including the new or extended
   non-executable case.
3. `just lint` passes on Linux. Linux lint was green before; this must not regress it.
4. `just fmt` leaves the tree clean (or `cargo fmt --check` on the touched file, if
   `just fmt` hits the repo-wide `bob-cli-24` problem).
5. macOS: a local Linux cross-check is not viable, because
   `cargo check --target aarch64-apple-darwin` fails building `rquickjs-sys`. GitHub
   Actions is the macOS gate. After the commit lands on master, wait for the CI workflow
   run for that exact SHA, using `/sase_monitor` with
   `gh run watch <run-id> --exit-status` or an equivalent bounded wait. Then confirm the
   `test (macos-latest)` job's `Lint macOS-only targets` log no longer contains
   `command_on_path` (`gh run view <run-id> --log-failed | grep -c command_on_path`
   prints `0`, or the step passes). That step is expected to stay red until `bob-cli-l5`
   fixes `ptsname_r`. That is not a regression from this change, so report it rather
   than fixing it.

## Files

- `src/native/desktop_notify.rs`: the only source file touched.

No docs, CLI help, JSON contracts, config keys, or memory notes change. The
`notify-send` and `BOB_VAULT_SYNC_NOTIFY` behavior described in `docs/` stays accurate.
Do not edit `Cargo.toml`'s version or `CHANGELOG.md` (release-train decision). Commit as
a `fix(notify):` (or `fix(desktop-notify):`) conventional commit with no `!`, since no
contract changes.
