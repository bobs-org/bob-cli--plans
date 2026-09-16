---
tier: tale
title: Fix failing dotfiles CI test job
goal:
  "The bbugyi200/dotfiles CI `test (ubuntu-latest)` job passes again: chezmoi is
  installed in the job, and install_luarocks_test.sh uses the real chezmoi helpers and
  no longer depends on the host machine."
size: small
proposed_by: bbugyi200.athena.0ly
create_time: 2026-09-16 10:14:15
status: wip
---

# Plan: Fix the failing dotfiles (chezmoi) GitHub Actions `test` job

## Target repository

All changes belong to the `chezmoi` linked repository (GitHub `bbugyi200/dotfiles`), not
to bob-cli. Open it with the `/sase_repo` skill (`sase repo open chezmoi -r "<reason>"`)
and use only the printed path for reads, edits, and test runs. The chezmoi checkout then
becomes a commit obligation for the turn's final declaration.

## Diagnosis

`actstat` reports that the `CI` workflow on `bbugyi200/dotfiles` master fails in job
`test (ubuntu-latest)`, step `Run tests` (`just test`). The `lint` job passes. Every
master push has failed since `a01facce` (2026-09-13); the last green run was `ed48a190`.
The latest run (35100526604, commit `696615c`, bashunit 0.50.1) reports
`146 passed, 5 skipped, 9 failed`. The 9 failures have two independent root causes.

### Root cause 1 — `tests/bash/poseidon_chezmoi_isolation_test.sh` (4 failures)

Every test fails with `chezmoi: command not found`. Commit `a01facce` added this file,
and its `render_ignore` / `render_cargo` helpers pipe `home/.chezmoiignore` and
`home/dot_cargo/config.toml.tmpl` through `chezmoi execute-template`. The CI `test` job
never installs chezmoi. The tests pass locally only because chezmoi is installed on the
dev machines. This failure has kept CI red since 2026-09-13, which hid root cause 2.

Checked: chezmoi v2.72.2 (the current latest release, which `get.chezmoi.io` installs)
renders both templates correctly with an empty `HOME`, no chezmoi config, and no
`~/.local/share/chezmoi`. All 4 tests pass under those CI-like conditions.

### Root cause 2 — `tests/bash/install_luarocks_test.sh` (5 failures)

Every installer run exits `127` (command not found), including on athena. This is a
merge race between two commits that landed 90 seconds apart on 2026-09-15:

- `49178c5a` ("guard chezmoi install scripts for headless sudo") added `chez::has_tty`,
  `chez::can_sudo`, and `chez::exit_install` to `lib/chezmoi_utils.sh`. It also changed
  `home/.chezmoiscripts/run_onchange_install_luarocks.tmpl` so that it calls
  `chez::can_sudo` before `apt-get` and ends with `chez::exit_install "$?"`.
- `02abd5ba` ("preserve rock tree and summarize failures") added the test. The test's
  `write_chezmoi_utils` writes a hand-written stub `chezmoi_utils.sh` that defines only
  `chez::build_dir_root` and `chez::log`. The installer's last line therefore fails with
  `chez::exit_install: command not found`, and every run exits `127`.

Just adding the missing functions to the stub is not a robust fix, because the same test
has three hidden dependencies on the host machine:

1. `run_installer` uses `PATH="${fake_bin}:/usr/bin:/bin"`. As a result:
   - `install_lua51` passes or fails depending on whether the host has a Lua 5.1
     interpreter in `/usr/bin`. athena has `/usr/bin/lua5.1`; the GitHub ubuntu runner
     does not.
   - On the runner, the missing interpreter makes the installer take the
     `chez::can_sudo` → real `sudo apt-get install lua5.1` path.
   - The bootstrap-failure test also assumes that no `luarocks` exists in `/usr/bin`.
2. The real `chez::exit_install` exits `0` (with a warning) when stdin is not a TTY.
   That is always the case under bashunit and CI. Tests that expect `rc=1` therefore
   need an explicit TTY model rather than the real `[[ -t 0 ]]` check.

The product code (`run_onchange_install_luarocks.tmpl`, `lib/chezmoi_utils.sh`) is
correct. The test is stale. `tests/bash/chezmoi_utils_test.sh` already tests the helpers
themselves, including `chez::exit_install`'s soft-exit behavior.

### Downstream note

`just test` runs `test-nvim test-hammerspoon test-bash test-python` in order, so
`test-python` has not run in CI since 2026-09-13. It passes locally (`26 passed`), so no
further CI failure is expected after the bash fixes.

## Changes

### 1. `.github/workflows/ci.yml` — install chezmoi in the `test` job

In the `test` job only, add this step directly after the existing `Install bashunit`
step and before `extractions/setup-just@v2`. It follows the same unpinned curl-installer
pattern as the bashunit step:

```yaml
- name: Install chezmoi
  run: |
    curl -fsLS https://get.chezmoi.io -o "$RUNNER_TEMP/chezmoi-install.sh"
    sh "$RUNNER_TEMP/chezmoi-install.sh" -b "$HOME/bin"
    echo "$HOME/bin" >> "$GITHUB_PATH"
    "$HOME/bin/chezmoi" --version
```

Indent it to match the sibling steps. Leave the `lint` job unchanged. Do NOT make the
poseidon test skip when chezmoi is missing, because that would silently remove the
test's coverage in CI.

### 2. `tests/bash/install_luarocks_test.sh` — use the real helpers and isolate the test from the host

Keep the five existing tests and their assertions unchanged. Make these edits:

- **`set_up`**: add `sys_bin="${test_tmp}/sysbin"`, add `"${sys_bin}"` to the
  `mkdir -p`, and call a new `write_host_stubs` right after `write_chezmoi_utils`.
- **`write_chezmoi_utils`**: replace the hand-written stub. Copy the repo's real
  helpers, then append a TTY override so the test decides whether the apply is
  interactive:

  ```bash
  function write_chezmoi_utils() {
    # Use the real helpers so the test tracks the installer's actual contract.
    cp "${PWD}/lib/chezmoi_utils.sh" \
      "${test_home}/.local/share/chezmoi/lib/chezmoi_utils.sh"
    cat >>"${test_home}/.local/share/chezmoi/lib/chezmoi_utils.sh" <<'EOF'

  # Test override: model an interactive apply unless FAKE_HAS_TTY=0.
  function chez::has_tty() {
    [[ "${FAKE_HAS_TTY:-1}" == 1 ]]
  }
  EOF
  }
  ```

- **New `write_host_stubs`** (place it next to the other `write_*` helpers). It creates:
  - an executable `${fake_bin}/lua5.1` that prints `5.1`, so `install_lua51` succeeds
    without apt, brew, or sudo;
  - an executable `${fake_bin}/sudo` that appends `sudo <args>` to `${CALLS_FILE}` and
    exits `1`, as a safety net against real privileged commands;
  - symlinks in `${sys_bin}` for only `bash`, `mkdir`, and `rm`, each pointing at
    `$(command -v <tool>)`.

  `bash` must be linked because `PATH=... bash "${INSTALL_SCRIPT}"` looks up `bash`
  using the temporary PATH. The prototype exited `127` without it. `mkdir` and `rm` are
  the only external commands that the installer and the luarocks stub use on the tested
  paths.

- **`run_installer`**: change `PATH="${fake_bin}:/usr/bin:/bin"` to
  `PATH="${fake_bin}:${sys_bin}"` and also pass `FAKE_HAS_TTY="${FAKE_HAS_TTY:-1}"` in
  the command's environment.
- **New regression test** at the end of the file, covering how the two racing commits
  interact:

  ```bash
  function test_non_interactive_failure_soft_exits_with_summary() {
    write_luarocks_stub
    fail_rock busted 37

    local rc
    if FAKE_HAS_TTY=0 run_installer; then rc=0; else rc=$?; fi

    assert_same "0" "${rc}"
    assert_contains "- busted (exit status 37)." "$(final_summary)"
    assert_contains "non-interactive chezmoi apply" "$(stdout_text)"
    assert_not_contains "sudo" "$(calls_text)"
  }
  ```

A prototype of exactly these edits, run in a scratch copy of the repo, passed all 6
luarocks tests with local bashunit 0.36.0. With CI's bashunit 0.50.1, chezmoi v2.72.2,
and an empty `HOME`, all 10 tests in the two files passed.

## Out of scope

- Do not edit `home/.chezmoiscripts/run_onchange_install_luarocks.tmpl` or
  `lib/chezmoi_utils.sh`.
- Do not pin new tool versions or restructure the workflow.
- The other failing repos that `actstat` lists (sase, bob-mac-capture, sase-telegram,
  sase-core) are unrelated to this task.

## Verification

Run everything from the chezmoi checkout root:

1. `bashunit tests/bash/install_luarocks_test.sh tests/bash/poseidon_chezmoi_isolation_test.sh`
   → 10 passed, 0 failed.
2. CI-like run with an empty home:
   `env HOME="$(mktemp -d)" bashunit tests/bash/install_luarocks_test.sh tests/bash/poseidon_chezmoi_isolation_test.sh`
   → all pass.
3. `just test-bash` → 0 failures (5 skips are expected). Then `just test-python` →
   passes. Run the full `just test` too if `busted` / `nlua` are available locally.
4. Optionally run `git diff` and a YAML parse of `.github/workflows/ci.yml` (for example
   `python3 -c 'import yaml,sys; yaml.safe_load(open(sys.argv[1]))' .github/workflows/ci.yml`)
   to confirm the workflow is still valid YAML.
5. After the chezmoi commit lands on `bbugyi200/dotfiles` master, wait for the new `CI`
   run with the `/sase_monitor` skill (for example by monitoring
   `gh run watch <run-id> -R bbugyi200/dotfiles --exit-status`). Then confirm with
   `actstat --only-failures` that the `bbugyi200/dotfiles` head commit is green. If the
   `test` job still fails, fetch the log with
   `gh run view <run-id> -R bbugyi200/dotfiles --log-failed`, then fix and repeat.
