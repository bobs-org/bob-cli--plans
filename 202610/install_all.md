---
tier: tale
title: Add just install-all and install-all-and-restart
goal:
  One command per machine pulls and installs bob-cli, deploys the sibling bob-plugins
  and (on macOS) bob-mac-capture checkouts, gracefully skips anything absent, and
  optionally restarts a running Obsidian, with a clear and honest summary.
size: medium
proposed_by: bbugyi200.athena.0we
create_time: 2026-10-04 10:32:39
status: wip
---

# Plan: `just install-all` and `just install-all-and-restart`

## Goal

Give Bryan one command per machine that brings the whole Bob toolchain up to date from
the checkouts that sit next to bob-cli:

- `just install-all` pulls and installs bob-cli, then pulls and deploys the sibling
  `bob-plugins` and `bob-mac-capture` checkouts. Siblings that are absent are skipped
  gracefully.
- `just install-all-and-restart` does all of that, then restarts Obsidian if it is
  running, so the freshly synced plugins load.

The main target is the MacBook. On Linux hosts such as athena, the same command must do
the right thing: install bob, sync plugins into the local vault, skip the macOS-only
app, and leave Obsidian alone when it isn't running.

The design aims for intuitive, reliable, and beautiful: a predictable step order, each
step's failure kept to itself, an honest summary, and output styled like the existing
justfile banners.

## Changes to the original requirements (call-outs)

Each item below refines Bryan's spec. The reason follows each one.

1. **The pulls use `git pull --ff-only --no-rebase` instead of a bare `git pull`.** An
   install flow should never create merge commits or start a rebase in a checkout. A
   diverged branch, a dirty tree that blocks the pull, no upstream, or no network
   becomes a visible **warning**. The step then installs the local checkout as-is (the
   same degrade-and-continue rule `bob plugins` already uses for its own pull). The pull
   doesn't set `GIT_TERMINAL_PROMPT=0`, because a human runs this command: an SSH
   passphrase or credential prompt should still work.
2. **The installer updates itself.** `just` parses the justfile before the bob-cli pull
   runs. So the logic lives in `scripts/install_all`. When the bob-cli pull moves HEAD,
   the script re-`exec`s the freshly pulled copy of itself for the remaining steps, so a
   pull that changes the installer takes effect on the same run.
3. **bob-plugins deploys with `bob plugins sync --no-pull --repo <sibling>`, run by the
   bob binary that was just installed.**
   - `--repo` makes it deploy exactly the sibling checkout we pulled. Without it, the
     command would resolve `BOB_PLUGINS_DIR` or the hard-coded default path.
   - `--no-pull` avoids pulling twice, since our explicit pull already ran and reported
     its result uniformly.
   - Running the fresh binary (from the same cargo root `just install` uses) means
     brand-new sync behavior is what deploys.
   - `npm run build` isn't needed: generated `main.js` files are committed, and sync
     copies the committed files.
   - It never passes `--force`, so the dirty-vault guard stays on.
4. **bob-plugins is skipped when there is no Obsidian vault.** If
   `${BOB_DIR:-~/bob}/.obsidian` doesn't exist, the step is skipped instead of letting
   sync create a stray vault folder.
5. **bob-plugins is verified after the sync.** `bob plugins sync` exits 0 even when it
   skips a file that is dirty in the vault. The step therefore runs
   `bob plugins list --no-pull --repo <sibling> --format json` afterwards. If any plugin
   is still in `drift`, the summary shows a ⚠ warning instead of claiming success.
6. **bob-mac-capture is macOS-only.** On Linux the whole project is skipped, pull
   included, with a `skipped · macOS only` row.
7. **bob-mac-capture keeps its signing identity.** The step runs
   `just install "$HOME/Applications" "${CODESIGN_IDENTITY:--}"`. The repo's
   `just install` recipe hard-codes the identity default `-`, which shadows the
   `CODESIGN_IDENTITY` environment variable its `Scripts/install.sh` otherwise honors.
   Passing the variable through means a real Apple Development identity is never
   silently downgraded to ad-hoc. A downgrade would reset notification and
   launch-at-login trust. With the variable unset, the behavior is exactly today's: the
   MacBook's installed `~/Applications/Bob Mac Capture.app` is ad-hoc signed. That
   installer already restarts the menu-bar app if it was running, so install-all doesn't
   restart it again.
8. **A missing sibling is skipped with a hint.** The skip line prints the clone command,
   e.g.
   `git clone git@github.com:bobs-org/bob-mac-capture.git <parent>/bob-mac-capture`. ⚠
   On the MacBook today, `~/projects/github/bobs-org/` contains bob-cli and bob-plugins
   but **not** bob-mac-capture. Until Bryan clones it there, install-all will skip it on
   the Mac.
9. **Failures are isolated.** Each project runs even if an earlier one failed. The
   summary lists every outcome, and the command exits 1 when any step failed. Skips and
   warnings still exit 0.
10. **`install-all-and-restart` is one script run with `--restart-obsidian`, not a
    literal `just install-all` followed by a second command.** The behavior is the same
    (install-all, then restart), but it produces one summary that includes the restart.
    The restart happens only if no step failed, the same as `install-all && restart`. It
    only restarts an Obsidian that is already running and never launches a stopped one.
11. **How Obsidian restarts on each OS.**
    - **macOS:** a graceful AppleScript quit of bundle `md.obsidian`, then a bounded
      wait (no force-kill), then `open -b md.obsidian`, then a check that it came back.
      The first run may show macOS's one-time "allow your terminal to control Obsidian"
      Automation prompt.
    - **Linux:** the official Obsidian CLI `restart` command (documented as "Restart the
      app"), used only when Obsidian is running and the CLI is registered at its
      documented Linux location `~/.local/bin/obsidian`. Otherwise the step prints a
      manual-restart hint.

## Design

### Files

- **New `scripts/install_all`:** an executable bash script that holds all the logic.
- **`justfile`:** two thin recipes, and the new script added to `check-scripts`.
- **`README.md`:** an "Install everything" subsection under Installation.
- **`docs/getting-started.md`:** a one-sentence pointer to `just install-all`.
- **`docs/plugins.md`:** a one-sentence pointer in the Sync section.

No Rust changes and no changes to bob-plugins or bob-mac-capture.

### justfile recipes

Place them directly after `install` and before `install-smoke`. The comments double as
the `just --list` descriptions.

```just
# Pull + install bob-cli, then pull + deploy the sibling bob-plugins and
# bob-mac-capture checkouts (missing siblings are skipped).
install-all:
    @scripts/install_all

# install-all, then restart Obsidian (if it is running) so it loads the new plugins.
install-all-and-restart:
    @scripts/install_all --restart-obsidian
```

Also add `scripts/install_all` to the `bash -n` list in `check-scripts`.

### Script CLI (`scripts/install_all`)

```
Usage: scripts/install_all [-h] [-r]

Pull and install bob-cli from this checkout, then pull and deploy the sibling
bob-plugins and bob-mac-capture checkouts that live next to it. Missing siblings
are skipped. Usually run through `just install-all` or
`just install-all-and-restart`.

Options:
  -h, --help              Show this help and exit.
  -r, --restart-obsidian  After a clean install, restart Obsidian if it is running.
```

- An unknown argument prints the usage to stderr and exits 2.
- Exit codes: 0 when every step succeeded, skipped, or only warned; 1 when any step
  failed.

### Paths

- `root` is the bob-cli checkout: the script's own `..`, resolved with `cd`/`pwd`.
- `parent` is `dirname "$root"`.
- The siblings are `"$parent/bob-plugins"` and `"$parent/bob-mac-capture"`.
- `bob_bin` is `${CARGO_INSTALL_ROOT:-${CARGO_HOME:-$HOME/.cargo}}/bin/bob`, the same
  root `just install` uses (say so in a comment). If that file is missing or not
  executable, fall back to `command -v bob`.
- Display paths with `$HOME` abbreviated to `~`.

### Step flow (sequential, in this order)

bob-cli goes first because plugin sync uses the new `bob`, and Bob Mac Capture shells
out to `bob` at runtime.

1. **bob-cli**
   1. Pull (see "Pull helper").
   2. If HEAD moved and this is the first run, re-`exec` the pulled
      `scripts/install_all` with the original arguments. Hand the pull result to the new
      process through env vars (for example `BOB_INSTALL_ALL_SELF_PULL=<state>` and
      `…_DETAIL=<text>`). The re-exec'd process reads them, unsets them, skips the
      bob-cli pull, and records the handed-over result. Start the run timer before the
      pull and carry its start through the exec too, so durations stay truthful.
   3. Run `just --justfile "$root/justfile" install`. It prints its own 📦 INSTALL
      banner and cargo output. Non-zero means ✗ failed.
   4. Summary detail: the pull result plus `installed @ <short-sha>`. The crate version
      is always 0.1.0, so the SHA is the meaningful identifier.
2. **bob-plugins**
   1. If the directory is missing, skip with a clone hint.
   2. If it exists without a `plugins/` subdirectory, ✗ fail with "doesn't look like
      bob-plugins".
   3. If `${BOB_DIR:-$HOME/bob}/.obsidian` is missing, skip with
      `no Obsidian vault at <path>`.
   4. Pull.
   5. Run `"$bob_bin" plugins sync --no-pull --repo "$dir"` and stream its output.
      Non-zero means ✗.
   6. On success, run `"$bob_bin" plugins list --no-pull --repo "$dir" --format json`
      and extract the top-level `count`, `synced`, and `drift` integers with a grep/sed
      on those keys. Don't depend on `jq`. Plugin rows carry `"sync": "drift"` as a
      value, never as a `"drift":` key, so matching `"drift": *[0-9]+` is safe.
      - `drift == 0`: ✓ `N/N plugins in sync`.
      - Otherwise: ⚠ `K drifted (dirty in vault — review, then bob plugins sync -F)`.
3. **bob-mac-capture**
   1. If not on Darwin (`uname -s`), skip with `macOS only`.
   2. If the directory is missing, skip with a clone hint.
   3. If `Scripts/install.sh` is missing, ✗ fail.
   4. Pull.
   5. Run
      `just --justfile "$dir/justfile" install "$HOME/Applications" "${CODESIGN_IDENTITY:--}"`.
      - Exit 0: ✓ `installed ~/Applications/Bob Mac Capture.app`.
      - Exit 72 (the installer's "installed and verified, but the running-app handoff
        failed"): ⚠ `installed; restart it via Bob → Restart Bob Mac Capture`.
      - Any other non-zero exit: ✗.
4. **Obsidian** (only with `--restart-obsidian`)
   1. If any earlier step failed, show the row as skipped with
      `not restarted: a step failed — fix it and rerun just install-all-and-restart`.
   2. **macOS**
      1. Check whether it's running with
         `osascript -e 'application id "md.obsidian" is running'`. This doesn't launch
         Obsidian and needs no permission. If it isn't running, the row is
         `– not running · nothing to restart`.
      2. Quit with `osascript -e 'tell application id "md.obsidian" to quit'`. If the
         quit fails, for example because Automation permission was denied, ✗ fail and
         name System Settings → Privacy & Security → Automation.
      3. Poll every 0.5s, for up to 30s, until it no longer reports running **and**
         `pgrep -x Obsidian` finds nothing. If time runs out, ✗ fail with "Obsidian did
         not quit within 30s (an open dialog?) — not relaunching". Never force-kill.
      4. Run `open -b md.obsidian`, then poll up to 15s for it to report running again.
         If it does, ↻ `restarted`. If not, ⚠ `relaunch requested; not confirmed yet`.
   3. **Linux**
      1. Check for a running instance with `pgrep -x obsidian`. If none, the row is
         `– not running`.
      2. If it is running and `$HOME/.local/bin/obsidian` is executable (the documented
         Linux registration path of the official Obsidian CLI), run that binary with
         `restart`. Success: ↻ `restart requested`. Failure: ✗.
      3. If it is running and there is no registered CLI, show ⚠
         `running; restart it manually to load the new plugins`.
      4. Don't call an arbitrary `obsidian` from PATH: it could be the desktop app's own
         launcher.

### Pull helper

`pull <dir>` sets a state, either `updated`, `current`, `warn`, or `none`, plus a short
detail string:

- If `git -C dir rev-parse --is-inside-work-tree` fails, the state is `none`, detail
  `not a git checkout`. This is not a warning.
- Otherwise record the old `rev-parse --short HEAD`. Then run
  `git -C dir pull --ff-only --no-rebase` with stdout and stderr captured to a temp
  file. Leave stdin and the TTY alone, so passphrase prompts still work.
  - **Success with a new HEAD:** `updated`, detail like
    `3 new commits · 76df6b6 → a1b2c3d` (count via `rev-list --count old..new`).
  - **Success with the same HEAD:** `current`, detail `up to date · <sha>`.
  - **Failure:** `warn`, detail `pull failed — using local checkout`. Print the captured
    git output indented and dimmed so the cause is visible.

### Output (visual language)

Reuse the justfile `_banner` look: a blank line,
`icon + two spaces + bold colored TITLE`, then the same 48-character `─` rule in that
color. Plain text without ANSI codes when stdout isn't a TTY or `NO_COLOR` is set. Keep
the rule string identical to the justfile's `rule` and note in a comment that it mirrors
`_banner`. Write subprocess output (cargo, swift, sync diffs) through raw, unindented
and unpiped, so progress bars and colors survive. Print only our own lines with a
two-space indent.

Target rendering (TTY colors omitted):

```
🚀  INSTALL ALL
────────────────────────────────────────────────
  from     ~/projects/github/bobs-org
  then     restart Obsidian if running        (only with -r)

🦀  BOB-CLI  1/3
────────────────────────────────────────────────
  ⇣  pull   ✓ 3 new commits · 76df6b6 → a1b2c3d
  $  just install
                                         (just install's 📦 INSTALL banner + cargo output)
🧩  BOB-PLUGINS  2/3
────────────────────────────────────────────────
  ⇣  pull   ✓ up to date · 9f8e7d6
  $  bob plugins sync --no-pull --repo ~/projects/github/bobs-org/bob-plugins
                                         (sync output)
🍎  BOB-MAC-CAPTURE  3/3
────────────────────────────────────────────────
  –  skipped · macOS only

🔄  OBSIDIAN
────────────────────────────────────────────────
  ↻  restarted

✨  SUMMARY
────────────────────────────────────────────────
  ✓  bob-cli          3 new commits · installed @ a1b2c3d   1m 12s
  ✓  bob-plugins      up to date · 6/6 plugins in sync          2s
  –  bob-mac-capture  skipped · macOS only
  ↻  obsidian         restarted                                 4s

  ✓  ALL INSTALLED · 1m 18s
```

- **Glyph colors:** ✓ green (32), ⚠ yellow (33), ✗ red (31), ↻ cyan (36), – dim (2).
- **Banner colors:** header 36, bob-cli 35, bob-plugins 34, bob-mac-capture 32, Obsidian
  36, summary 1 (bold default).
- **Final line**, modeled on `just all`'s `✓  ALL CHECKS PASSED`:
  - `✓  ALL INSTALLED · <total>` when there were no warnings or failures.
  - `⚠  INSTALLED WITH N WARNING(S) · <total>` when there were warnings.
  - `✗  N STEP(S) FAILED · <total>` (red, exit 1) when anything failed.
- **Durations:** measured with bash `SECONDS`, formatted `Ns` or `Mm SSs`. Skipped rows
  show no duration.
- **Row alignment:** pad project names to a fixed width (`printf '%-16s'`). Right-align
  durations only when it's simple. Fixed column padding is acceptable.
- **Commands:** echo each external command we run on a dim `$` line before it runs.

### Implementation constraints

- `#!/usr/bin/env bash` with `set -euo pipefail`. Run every fallible step inside
  `if …; then` or `|| status=$?`, so `set -e` never aborts before the summary.
- Stay bash 3.2-compatible (stock macOS `/bin/bash`), even though the MacBook's PATH has
  Homebrew bash 5:
  - no associative arrays, `mapfile`, `${var,,}`, or `&>>`;
  - no `$EPOCHSECONDS`;
  - guard empty-array expansions under `set -u`. Parallel indexed arrays or per-project
    scalar variables are fine for the summary rows.
- Require `git` and `just` on PATH up front, with a clear message and exit 1 if either
  is missing.
- Clean up the pull temp files with an `EXIT` trap. Keep the trap simple and re-arm it
  correctly across the self-`exec` (the exec replaces the process, so nothing to clean
  in the old one must leak; create temp files only after the exec decision or remove
  them before exec'ing).
- Match the repo's shell style: two-space indent like `scripts/lib/bob_shell.sh`, quoted
  expansions, `local` in functions, and short comments only where intent isn't obvious.
  Keep it self-contained rather than sourcing `bob_shell.sh`, whose `name: level:` log
  format doesn't fit this UI.
- Make the file executable (`chmod +x`, committed with mode 100755).

### Documentation

- **README.md, Installation:** after the `just install` paragraph and before the
  remote-install snippet, add a `### Install everything` subsection. It should say that:
  - `just install-all` pulls (`--ff-only`) and installs this checkout, then pulls and
    deploys `../bob-plugins` (via `bob plugins sync --no-pull --repo`) and
    `../bob-mac-capture` (macOS only, `just install`, honoring `CODESIGN_IDENTITY`).
  - Missing siblings, a missing vault, and non-macOS hosts are skipped.
  - Pull failures warn and install the local checkout.
  - `just install-all-and-restart` additionally restarts a running Obsidian (macOS: a
    graceful quit and relaunch, and a first-run Automation permission prompt; Linux: the
    Obsidian CLI `restart` when registered) and only after a clean install.
  - Exit status: 1 only on a failed step.
  - Include the expected sibling layout as a short tree under
    `~/projects/github/bobs-org/`.
- **docs/getting-started.md:** extend "From a checkout, run `just install`." with: "Run
  `just install-all` to also deploy the Bob plugins (and, on macOS, Bob Mac Capture)
  from sibling checkouts." This answers that section's "installing the CLI alone does
  not install that interface".
- **docs/plugins.md, Sync section:** add one sentence saying that bob-cli's
  `just install-all` runs this sync against the sibling `bob-plugins` checkout after
  pulling it.

## Verification

Don't run `just install-all` for real from an agent workspace. It would replace the
user's installed `bob` and write to the real `~/bob` vault. Run the checks below
instead.

1. `just check-scripts` passes (the new script is in the `bash -n` list).
2. `shellcheck scripts/install_all` is clean (shellcheck is installed on athena).
3. `scripts/install_all --help` renders the usage; `scripts/install_all --bogus`
   exits 2.
4. **Sandbox end-to-end run on Linux.** Everything goes in a temp dir `T`, and the real
   install root, vault, and completion state are never touched.
   1. `git clone --bare <this checkout> "$T/upstream.git"` and
      `git clone "$T/upstream.git" "$T/bob-cli"`. Copy the changed files into the clone
      and commit them there, then push to `upstream.git`.
   2. Clone the linked bob-plugins and bob-mac-capture checkouts (open them with
      `sase repo open`) into `"$T/bob-plugins"` and `"$T/bob-mac-capture"`.
   3. Make a fake vault with `mkdir -p "$T/vault/.obsidian"`.
   4. Run `"$T/bob-cli/scripts/install_all" --restart-obsidian` with this environment
      (mirroring `install-smoke`'s isolation):
      - `CARGO_INSTALL_ROOT="$T/cargo"`
      - `CARGO_HOME="$HOME/.cargo"`
      - `RUSTUP_HOME="$HOME/.rustup"` (if rustup is used)
      - `BOB_DIR="$T/vault"`
      - `BOB_PLUGIN_BACKUPS_DIR="$T/backups"`
      - `HOME="$T/home"`, `XDG_STATE_HOME`, `XDG_DATA_HOME`, `ZDOTDIR="$T/home/.zdot"`,
        and `SHELL=/bin/zsh`, so completion installs into the sandbox
      - `PATH` still containing cargo, just, and git

      Set `RUSTUP_HOME` and `CARGO_HOME` from the real home _before_ overriding `HOME`.
      Expected results:
      - bob-cli is `up to date`, installed into `$T/cargo/bin/bob`.
      - bob-plugins syncs into `$T/vault/.obsidian/plugins/` and reports
        `N/N plugins in sync`.
      - bob-mac-capture shows `skipped · macOS only`.
      - Obsidian shows `not running` (it isn't running on athena).
      - The final line is ✓ and the exit status is 0.

   5. **Self-update re-exec.** Make a second clone of `upstream.git` and commit a
      visible change to `scripts/install_all` there, e.g. a changed header subtitle.
      Push it, then rerun from the first clone. Expect `1 new commit`, the _new_ header
      text (proving the re-exec), and no second bob-cli pull.
   6. **Skips and warnings.**
      - Remove `"$T/bob-mac-capture"`: the row reads skipped with the clone hint.
      - Run with `BOB_DIR` pointing at a directory without `.obsidian`: plugins skipped.
      - Detach HEAD in `"$T/bob-plugins"`: the pull ⚠ warns, the sync still runs, the
        final line is ⚠, and the exit status is 0.
   7. **Failure isolation.** Point `"$T/bob-plugins"` at a directory without `plugins/`:
      ✗ fail, mac-capture still evaluated, Obsidian row skipped because a step failed,
      exit 1.
   8. Pipe one run through `| cat` and confirm there are no ANSI escapes.

5. **macOS-only paths can't be exercised on athena.** That covers the Mac Capture
   install, exit-72 handling, and the AppleScript restart. Review them carefully against
   the bob-mac-capture `Scripts/install.sh` contract described above. Note in the final
   report that Bryan should do one real `just install-all-and-restart` on the MacBook
   after cloning bob-mac-capture into `~/projects/github/bobs-org/`.
6. `just all` still passes (no Rust changes expected; it confirms nothing else broke).
