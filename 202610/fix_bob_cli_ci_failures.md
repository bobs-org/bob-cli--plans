---
tier: tale
title: Fix bob-cli CI release-tool, macOS clippy, and zsh zpty failures
goal:
  "bob-cli CI and Release go green on master: release-tool tests pass under GitHub
  Actions, macOS clippy passes, and the unmasked macOS zsh completion test stops racing."
size: small
proposed_by: bbugyi200.athena.16c
proposed_with: claude/opus
create_time: 2026-10-30 12:10:47
status: wip
---

# Fix bob-cli GitHub Actions: release-tool test, macOS clippy, and the hidden macOS zsh flake

## Goal

Make the bob-cli `CI` workflow (and the `Release` workflow, which calls it as `ci / …`)
green on master again. Two jobs fail today, and fixing them uncovers a third failure
that the macOS lint error has been hiding since 2026-10-28. All three root causes are
diagnosed and reproduced below. The fix touches only three files and changes no product
behaviour.

Current state (`actstat` at 0e178061, CI run 41076178662 / Release run 41076987862):

- `lint`, step "Format, clippy, scripts, and release-tool tests": `just fmt`,
  `just lint`, and `just check-scripts` pass, and then `just check-release-tools` fails.
- `test (macos-latest)`, step "Lint macOS-only targets": clippy fails, so the "Run
  tests" step is **skipped**.
- `test (ubuntu-latest)` and `package` pass.

The last fully green CI run was 6b0a30bf (2026-10-27).

## Diagnosis

### 1. Linux `lint` job: `test_package_check.py` assumes it is not on GitHub Actions

- Failing test:
  `tools/release/test_package_check.py::PackageCheckTests::test_main_warns_above_warning_threshold_but_succeeds_under_failure_threshold`
  (added in 232d68c0; it has failed in every CI run since 40596916186).
- `tools/release/package_check.py` `main()` prints `::warning::<message>` (a GitHub
  annotation) when `GITHUB_ACTIONS` is set and `WARNING: <message>` otherwise. The test
  asserts the local `WARNING:` form, and its `_run_main_with_archives` helper never
  controls the environment, so the test fails on CI only.
- Reproduced locally:
  `env -u GITHUB_ACTIONS python3 -m unittest discover -s tools/release -p 'test_*.py'`
  passes 110 tests (OK), and the same command with `GITHUB_ACTIONS=true` reproduces the
  one failure.
- `package_check.py` itself is correct: the annotation is intended CI behaviour. Fix the
  test, not the tool.

### 2. macOS `test` job, "Lint macOS-only targets": `needless_return`

- `src/native/vault_sync.rs:703`, the `return;` at the end of the
  `#[cfg(target_os = "macos")]` block in `send_clobber_notification` (added in
  16f3da48). `just lint` denies `clippy::needless_return`. The block only compiles on
  macOS, which is why the Linux lint job never sees it.
- Verified on the tailnet Mac: the full `just lint` clippy gate
  (`--workspace --all-targets --all-features`) at 0e178061 passes when only
  `-A clippy::needless_return` is added. This is therefore the only macOS lint error.

### 3. Hidden: macOS "Run tests" fails on `completion::zsh_adapter::real_zsh_first_tab_completes`

- The step has been skipped since the lint broke. In the last nine CI runs where it did
  run (through 9e814f0d, 2026-10-28), it failed seven times, always on this test, with
  `FORMAT-TAB MISS` and then `PROMPT-READ-FAILED`. Bead `bob-cli-da.1` note #2 recorded
  the same failure as a proposed follow-up, but no bead tracks it. Linux CI never sees
  it because `ubuntu-latest` has no zsh outside the full-test mode, so the test skips
  there.
- **Root cause.** f00854be ("make PTY drivers tolerate load") replaced `sleep 0.5` after
  each `^C` in `ZPTY_DRIVER` with `wait_for 'ZP>'`. `wait_for` clears its buffer on
  entry. The previous TAB's listing is followed by a redraw of `ZP> bob capture -…`, and
  that redraw often arrives after the previous `wait_for` has already matched. The
  post-`^C` wait can then match the **stale** redraw. The driver types the next scenario
  before zsh has handled SIGINT, the line being aborted swallows the first byte (`b`),
  and the fresh prompt receives `ob capture --format `. Completion then falls back to
  files. The CI log shows exactly this:
  `READINESS TIMEOUT: b^[[?2004l … ZP> ^[[?2004hob capture --format ^G <file listing>`.
  `^C` is asynchronous relative to the input stream, so any prompt-text readiness check
  races.
- **Reproduced on the tailnet Mac** with 12 `yes` CPU hogs: the current driver failed 6
  of 20 runs with the identical signature. A prototype of the replacement driver below
  passed 40 of 40 under the same load. It also passed 15 of 15 under load with a
  806-character PATH and `EDITOR=VISUAL=nvim`. On Linux (athena, zsh 5.9, `EDITOR=nvim`)
  it passed 10 of 10.
- **Secondary fragility (same driver).** The setup line types `export PATH="$TMP:$PATH"`
  into the pty. With a long PATH (about 970 characters, as on the Mac under `bash -l`
  plus cargo), the setup line is lost and the driver fails with `SETUP-READ-FAILED`. The
  current driver failed 3 of 3 runs that way and 12 of 12 under `cargo test`; the new
  driver passed 10 of 10. The likely cause is the macOS tty input queue (about 1 KB). CI
  does not hit this, but it blocks running the test on Bryan's Mac.
- A full `cargo test --workspace` on the Mac at 0e178061 found nothing else
  macOS-specific. Every other failure was `needspace.sty` not found, an artifact of the
  diagnostic overriding HOME and hiding the user-mode TeX tree in `~/Library/texmf`.
  CI's macOS runner has no pandoc, so those tests skip there. After these three fixes,
  the macOS job is expected to go green.

## Changes

### A. `tools/release/test_package_check.py` (make the helper hermetic, cover both forms)

1. Add `import os` alongside the existing imports, keeping the file's import ordering.
2. Give `_run_main_with_archives` a new keyword argument,
   `github_actions: bool = False`. Inside its existing `ExitStack`, before `main()`
   runs, add `stack.enter_context(mock.patch.dict(os.environ))`. Then set
   `os.environ["GITHUB_ACTIONS"] = "true"` when `github_actions` is true, and
   `os.environ.pop("GITHUB_ACTIONS", None)` otherwise. `patch.dict` restores the
   original environment on exit. The default therefore models a local run, whatever the
   host environment.
3. Leave `test_main_warns_above_warning_threshold_but_succeeds_under_failure_threshold`
   asserting the `WARNING:` lines. It now passes on CI too, unchanged.
4. Add `test_main_emits_github_warning_annotations_under_actions`, mirroring that test
   (same threshold derivation) with `github_actions=True`. Assert:
   - exit code 0;
   - `::warning::bob-obsidian is above the 8 MiB warning threshold` and
     `::warning::bob-obsidian-assets is above the 8 MiB warning threshold` are both in
     stdout;
   - `"WARNING: "` is not in stdout.
5. Do not modify `tools/release/package_check.py`.

### B. `src/native/vault_sync.rs` (macOS clippy)

Delete the `return;` line at the end of the `#[cfg(target_os = "macos")]` block in
`send_clobber_notification`, as clippy suggests. The `#[cfg(not(target_os = "macos"))]`
block is compiled out on macOS, so the function already ends after the `osascript` call.
Behaviour is unchanged on every platform.

### C. `tests/cli/completion/zsh_adapter.rs` (`ZPTY_DRIVER`)

Replace the body of the `ZPTY_DRIVER` raw string with the text below. This is the
prototype validated above.

What changes:

- A new `reset_line` helper replaces each `zpty -w -n pb $'\x03'` + `wait_for 'ZP>' ''`
  pair. It sends `^U` (kill-whole-line), types a marker command, and waits for the
  marker's _output_. ZLE reads `^U` and the command in order, so the next scenario
  always starts on an empty line, and stale prompt text cannot satisfy the wait. The
  `SY''NC` quoting keeps the marker out of the echoed command line.
- `bindkey -e` in the setup line pins `^U` to `kill-whole-line` even when
  `$EDITOR`/`$VISUAL` would make zsh pick vi keys.
- The driver exports `PATH` before `zpty -b`, so the pty shell inherits it instead of
  having it typed.
- Unchanged: the scenario order, every `*-TAB` / `FORMAT-HEADER` marker and its expected
  text, the 25 s deadline, the `PROMPT-READ-FAILED` / `SETUP-READ-FAILED` labels, and
  the Rust test function.
- Keep the existing doc comment above `ZPTY_DRIVER`; it stays accurate.
- The raw string must not contain the sequence `"#`. The text below does not; recheck
  after pasting.

```zsh
zmodload zsh/zpty || { print "ZPTY-MISSING"; exit 2 }
zmodload zsh/zselect || { print "ZSELECT-MISSING"; exit 2 }
TMP=$1
shift
typeset -i deadline=$((SECONDS + 25))
typeset -i sync_count=0
READ_RESULT=
PTY_FD=
wait_for() {
  local expected=$1 expected2=$2 remaining chunk
  READ_RESULT=
  while true; do
    if chunk=$(zpty -rt pb); then
      READ_RESULT+=$chunk
      if [[ $READ_RESULT == *"$expected"* && ( -z "$expected2" || $READ_RESULT == *"$expected2"* ) ]]; then
        return 0
      fi
    fi
    remaining=$((deadline - SECONDS))
    (( remaining > 0 )) || { print -r -- "READINESS TIMEOUT: $READ_RESULT"; return 1; }
    zselect -t 10 "$PTY_FD" || true
  done
}
# Empty the edit buffer in-band, then run a marker command and wait for its
# output. ZLE reads ^U and the command in order, so the next scenario always
# starts on an empty line once the marker prints; the quotes keep the marker
# out of the echoed command line. (A ^C here would race: the shell can still
# read the next scenario's first byte into the doomed line.)
reset_line() {
  (( sync_count += 1 ))
  zpty -w -n pb $'\x15'
  zpty -w pb "print -r -- SY''NC-$sync_count"
  wait_for "SYNC-$sync_count" ''
}
# The pty shell inherits PATH instead of having it typed: a long PATH can
# overflow the tty input queue (1024 bytes on macOS) and lose the setup line.
export PATH="$TMP:$PATH"
zpty -d pb 2>/dev/null
zpty -b pb zsh -f
PTY_FD=$REPLY
# bindkey -e pins ^U to kill-whole-line even when $EDITOR selects vi keys.
zpty -w pb "PS1='ZP> '; bindkey -e"
zpty -w pb "fpath=($TMP \$fpath)"
# -i skips compaudit's interactive insecure-directory prompt so the test is
# hermetic on any host; it does not change how completion renders.
zpty -w pb "autoload -Uz compinit && compinit -D -i && print -r -- COMPINIT-DONE"
wait_for COMPINIT-DONE '' || { print -r -- "SETUP-READ-FAILED"; zpty -d pb; exit 1 }
print -r -- "SETUP OK"
zpty -w -n pb $'bob capture-task\t'
if wait_for capture-task-sections ''; then print -r -- "FIRST-TAB OK"; else print -r -- "FIRST-TAB MISS"; fi
reset_line || { print -r -- "PROMPT-READ-FAILED"; zpty -d pb; exit 1 }
zpty -w -n pb $'bob capture -\t'
if wait_for --route ''; then print -r -- "DASH-TAB OK"; else print -r -- "DASH-TAB MISS"; fi
reset_line || { print -r -- "PROMPT-READ-FAILED"; zpty -d pb; exit 1 }
zpty -w -n pb $'bob capture --format \t'
if wait_for human ''; then print -r -- "FORMAT-TAB OK"; else print -r -- "FORMAT-TAB MISS"; fi
reset_line || { print -r -- "PROMPT-READ-FAILED"; zpty -d pb; exit 1 }
zpty -w -n pb $'bob capture --format \t'
if wait_for format ''; then print -r -- "FORMAT-HEADER OK"; else print -r -- "FORMAT-HEADER MISS"; fi
zpty -d pb
```

(The raw string starts with a newline after `r#"` and ends with a newline before `"#;`,
as it does today.)

## Validation

Run these on Linux (athena) from the repo root:

1. `just fmt`, `just lint`, and `just check-scripts`: all pass, and the lint output has
   no new warnings in the touched files.
2. `env -u GITHUB_ACTIONS just check-release-tools` **and**
   `GITHUB_ACTIONS=true just check-release-tools`: both report OK, with one more test
   than the current 110.
3. `cargo test --locked --test cli completion::zsh_adapter`: all pass (zsh is installed
   on athena, so the real-zpty test runs rather than skipping).

Then run the macOS checks on the tailnet `mac` (`ssh mac`, user `bbugyi`). This is
best-effort: the Mac is offline unless its lid is open. If it is unreachable, say so in
the final report and rely on CI.

- Do **not** touch the Mac's `~/projects/github/bobs-org/bob-cli` checkout or its
  `target/`.
- Use a fresh scratch dir, for example `~/tmp/bobci-<sha>-<timestamp>`. Copy the working
  tree including uncommitted edits, for example
  `git ls-files -z | tar --null -T - -cf - | ssh mac "mkdir -p <dir> && tar -x -C <dir>"`.
  `git archive HEAD` would miss uncommitted changes.
- Build with
  `CARGO_TARGET_DIR=<dir>/target CARGO_INCREMENTAL=0 CARGO_PROFILE_DEV_DEBUG=0` and
  `PATH=$HOME/.cargo/bin:/usr/bin:/bin:/usr/sbin:/sbin`. The pinned 1.95.0 toolchain and
  `just` are in `~/.cargo/bin`. A build uses about 2.5 GB.
- The Mac's login shell is zsh: avoid unmatched globs in remote commands, because zsh
  aborts the whole command on them.
- Checks:
  1. `just lint` (the CI "Lint macOS-only targets" step): exit 0.
  2. Start 12 `(yes > /dev/null) &` CPU hogs, with a `trap` that kills them on exit.
     Then run
     `cargo test --locked -q --test cli completion::zsh_adapter::real_zsh_first_tab_completes`
     at least 20 times. All must pass; the old driver failed about 30% of runs under
     this load.
  3. Run that test once more under `bash -lc` (the Mac's long login PATH) to confirm the
     setup-line hardening. It must pass.
- Afterwards, `rm -rf` the scratch dir (the Mac disk is 97% full, about 14 GiB free) and
  confirm no `yes` processes remain.

After landing, confirm with `actstat` that CI `lint` and `test (macos-latest)` pass,
including the macOS "Run tests" step, which has been skipped since 2026-10-28.

## Out of scope

- The roughly 140 non-gated clippy warnings that both lint jobs print. They are not
  denied and do not fail CI.
- `FORMAT-HEADER`'s `format` marker also matches the echoed command line, so that check
  is weak. Leave the assertion semantics unchanged.
- The failing `bob-mac-capture` CI shown by `actstat`. It is a different repo and
  already has bead `bob-cli-dz`.
- Release-train state: do not edit the `Cargo.toml` version or `CHANGELOG.md`. The next
  scheduled train run picks up the green SHA without any workflow change.

Suggested commit subject: `fix(ci): restore release-tool and macOS lint/test jobs`.
