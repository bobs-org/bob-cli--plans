---
tier: tale
title: install-all offers to clone missing sibling checkouts over SSH
goal:
  When run from a terminal, just install-all asks [y/N] before the bob-cli build whether
  to clone a missing bob-plugins or bob-mac-capture sibling from
  git@github.com:bobs-org/<repo>.git, then installs the fresh clone; declined or
  non-interactive runs skip it as today.
size: medium
proposed_by: bbugyi200.athena.0wh.f0.f0
create_time: 2026-10-04 14:53:16
status: wip
---

# Plan: `just install-all` offers to clone missing sibling checkouts over SSH

## Goal

`scripts/install_all`, which `just install-all` runs, skips a missing `../bob-plugins`
or `../bob-mac-capture` checkout and prints a `git clone` hint. Change it so that, when
run from a terminal, it first asks `[y/N]` whether to clone each missing sibling into
the exact path where install-all looks for it. Clone from
`git@github.com:bobs-org/<repo>.git` over SSH. If Bryan says yes, the rest of the run
installs the fresh clone as usual. If he says no, or the run is not interactive,
install-all skips the sibling exactly as it does today.

This builds on the installer from `plan:202610/install_all.md`,
`plan:202610/install_all_restart_on_plugin_change.md` (commit `54ff70a`), and
`plan:202610/obsidian_restart_notification.md` (commit `cd2dd77`). Keep everything those
plans built: the self-update re-exec, step isolation, summary rows, the Obsidian restart
policy, and the macOS banners.

The motivating case: the MacBook's `~/projects/github/bobs-org/` has bob-cli and
bob-plugins but no bob-mac-capture, so install-all skips Bob Mac Capture there today.

## Design call-outs

1. **Ask up front, before the bob-cli build, and clone right after each answer.**
   - The questions go in a new `MISSING CHECKOUTS` section, between the `INSTALL ALL`
     header and the `BOB-CLI  1/3` banner.
   - bob-cli's `just install` is a release `cargo install` that can take minutes. If the
     question waited inside each step, it would arrive after Bryan had walked away. So
     would SSH's first-contact host-key confirmation and any key passphrase prompt.
     Asking first keeps the rest of the run unattended.
   - The clone runs right after its answer, while Bryan is still at the terminal. Its
     output streams raw so those SSH prompts work. Do not set `GIT_TERMINAL_PROMPT=0` or
     SSH `BatchMode`. This follows the original plan's rule for pulls.
   - Rejected: asking inside `plugin_step` / `mac_capture_step` at the point where they
     skip. That is less code, but it puts an interactive prompt (and a possible SSH
     passphrase prompt) in the middle of a long unattended run.

2. **Prompt only when both stdin and stdout are terminals (`[[ -t 0 && -t 1 ]]`).**
   - Otherwise ask nothing and print no section. The steps keep today's
     `skipped · missing checkout · git clone …` hint.
   - Requiring stdout as well avoids an invisible prompt that hangs a run redirected to
     a file. It also keeps SASE agents, which run without a TTY, from ever cloning into
     a workspace's parent directory.
   - `[y/N]`: only `y`, `Y`, `yes`, `Yes`, or `YES` means yes. Anything else is no,
     including Enter and EOF (Ctrl-D). On EOF, print a newline so the next line starts
     clean.

3. **Offer only siblings this run would install.**
   - Offer bob-plugins only when the vault `${BOB_DIR:-$HOME/bob}/.obsidian` exists,
     because `plugin_step` skips without one.
   - Offer bob-mac-capture only on Darwin, because `mac_capture_step` skips elsewhere.
   - Offer a sibling only when nothing exists at its path (`! -e`). A stray file there
     keeps today's skip.
   - On athena (Linux) this means only bob-plugins can ever be offered.

4. **SSH only, no HTTPS fallback.**
   - The URL is `git@github.com:bobs-org/<repo>.git`, built from one
     `clone_base='git@github.com:bobs-org'` constant. The skip hints, the prompt, and
     the clone all use it.
   - Rejected: falling back to `https://github.com/bobs-org/<repo>.git` after an SSH
     failure. A silent switch would leave an HTTPS `origin` that every later install-all
     pull uses. It could also stop for a GitHub username or password mid-run. A failed
     SSH clone shows git's own error, and the summary row prints the exact SSH command
     to retry.
   - Rejected: deriving the URL from bob-cli's own `origin`. Bryan asked for SSH, and
     the existing hints already use these URLs.

5. **A declined clone is a skip. A failed clone is a failed step.**
   - Decline: the step row is today's `– skipped · missing checkout · git clone …` hint.
     Exit status is unaffected.
   - Failure (Bryan asked for the clone and it did not happen): the step row is
     `✗ clone failed · retry: git clone <url> <path>`. `FAILURES` goes up, so the run
     exits 1, the same as any other failed step.
   - Success: the step does not pull the fresh clone. The pull line and the summary say
     `cloned · <sha>`. Everything after that runs unchanged: a fresh vault sync copies
     files, which can trigger the Obsidian restart and banner.

6. **Carry clone outcomes across the self-update re-exec.**
   - The clones happen before the bob-cli pull. That pull may `exec` the newer
     installer, which starts with fresh variables, so pass three more env vars the same
     way `BOB_INSTALL_ALL_SELF_PULL*` is passed:
     - `BOB_INSTALL_ALL_CLONES_OFFERED=1`
     - `BOB_INSTALL_ALL_PLUGINS_CLONE=<cloned|failed|empty>`
     - `BOB_INSTALL_ALL_MAC_CAPTURE_CLONE=<cloned|failed|empty>`
   - Without them, a failed clone would turn back into a plain skip after the re-exec.
   - **Re-exec from an older installer.** On a host that hasn't pulled this change, the
     first `just install-all` runs the old script. That script pulls this change and
     re-execs the new one with `BOB_INSTALL_ALL_HEADER_DONE=1` but without
     `BOB_INSTALL_ALL_CLONES_OFFERED`. The new script must still offer the clones then,
     so the first real run on the MacBook prompts. In that case, offer them right after
     the re-exec. If the section printed anything, print the `BOB-CLI  1/3` banner again
     before `just install`.

7. **No new options and no auto-yes.** No `--yes` or `--clone` flag and no env opt-in
   for non-interactive cloning. Nobody asked for one, and a non-TTY run should never
   create checkouts.

## Part 1 — `scripts/install_all`

Stay on bash 3.2: no `${var,,}`, no `mapfile`, no associative arrays. Keep mode
`100755`, the two-space indent, and short comments only where the intent isn't obvious.

### Globals and env handoff

- Next to `plugins_dir` / `mac_capture_dir`, add:
  - `clone_base='git@github.com:bobs-org'`
  - `obsidian_dir="${BOB_DIR:-$HOME/bob}/.obsidian"`

  Then drop `plugin_step`'s `bob_root` / `obsidian_dir` locals and their assignments,
  and use the global.

- With the other env reads (`self_pull_state=…`), read `BOB_INSTALL_ALL_CLONES_OFFERED`,
  `BOB_INSTALL_ALL_PLUGINS_CLONE`, and `BOB_INSTALL_ALL_MAC_CAPTURE_CLONE` into
  `clones_offered`, `plugins_clone_state`, and `mac_capture_clone_state`, each
  defaulting to empty. Add all three to the existing `unset`.
- Initialize the globals `CLONE_STATE=""` and `CLONE_SECTION_SHOWN=0` next to
  `PULL_STATE` etc.
- In the re-exec block, next to the existing exports:

  ```bash
  export BOB_INSTALL_ALL_CLONES_OFFERED=1
  export BOB_INSTALL_ALL_PLUGINS_CLONE="$plugins_clone_state"
  export BOB_INSTALL_ALL_MAC_CAPTURE_CLONE="$mac_capture_clone_state"
  ```

### Helpers

Put these near `pull_repo`. The sketch below shows the intended behavior; match the
surrounding style.

```bash
# Asks [y/N] and clones on yes. Sets CLONE_STATE to cloned, failed, or "".
offer_clone() {
  local name="$1" dir="$2" url answer sha
  url="$clone_base/$name.git"
  CLONE_STATE=''
  if (( CLONE_SECTION_SHOWN == 0 )); then
    banner 33 '📥' 'MISSING CHECKOUTS'
    CLONE_SECTION_SHOWN=1
  fi
  printf '  %s is missing at %s\n' "$name" "$(display_path "$dir")"
  printf '  Clone it from %s? [y/N] ' "$url"
  if ! read -r answer; then
    answer=''
    printf '\n'
  fi
  case "$answer" in
    y|Y|yes|Yes|YES) ;;
    *)
      glyph_line 2 '  ' '–' "not cloned · skipping $name"
      return 0
      ;;
  esac
  # Streams raw so SSH host-key and passphrase prompts reach the terminal.
  command_line "git clone $url $(display_path "$dir")"
  if git clone "$url" "$dir"; then
    sha="$(git -C "$dir" rev-parse --short HEAD 2>/dev/null || printf 'unknown')"
    glyph_line 32 '  ' '✓' "cloned · $sha"
    CLONE_STATE=cloned
  else
    glyph_line 31 '  ' '✗' 'clone failed'
    CLONE_STATE=failed
  fi
}

# Offers only siblings this run would install, and only when a person can answer.
offer_missing_clones() {
  CLONE_SECTION_SHOWN=0
  [[ -t 0 && -t 1 ]] || return 0
  if [[ ! -e "$plugins_dir" && -d "$obsidian_dir" ]]; then
    offer_clone bob-plugins "$plugins_dir"
    plugins_clone_state="$CLONE_STATE"
  fi
  if [[ "$(uname -s)" == Darwin && ! -e "$mac_capture_dir" ]]; then
    offer_clone bob-mac-capture "$mac_capture_dir"
    mac_capture_clone_state="$CLONE_STATE"
  fi
}

fresh_clone_result() {
  PULL_OUTPUT=''
  PULL_STATE=cloned
  PULL_DETAIL="cloned · $(git -C "$1" rev-parse --short HEAD 2>/dev/null || printf 'unknown')"
}
```

In `show_pull_result`, add a `cloned)` case that prints
`glyph_line 32 '  ⇣  clone  ' '✓' "$PULL_DETAIL"`. `clone  ` is the same width as
`pull   `. Nothing else compares `PULL_STATE` against `cloned`, and the bob-cli re-exec
checks only for `updated`.

### Main flow

Replace the header block with:

```bash
if [[ "$header_done" != 1 ]]; then
  banner 36 '🚀' 'INSTALL ALL'
  printf '  from     %s\n' "$(display_path "$parent")"
  printf '  then     restart Obsidian if the plugin sync changes the vault\n'
  offer_missing_clones
  banner 35 '🦀' 'BOB-CLI  1/3'
elif [[ "$clones_offered" != 1 ]]; then
  # Re-exec'd by an installer that predates the clone offer.
  offer_missing_clones
  if (( CLONE_SECTION_SHOWN == 1 )); then
    banner 35 '🦀' 'BOB-CLI  1/3'
  fi
fi
```

### Steps

**`plugin_step`.** Keep the check order: missing dir → not bob-plugins → no vault. The
vault check now uses the global `obsidian_dir`.

- In the missing-dir branch, before today's skip: if `plugins_clone_state` is `failed`,
  record a ✗ row with detail
  `clone failed · retry: git clone $clone_base/bob-plugins.git $(display_path "$plugins_dir")`.
  Increment `FAILURES`, call `add_summary 'bob-plugins' fail … "$(format_duration …)"`,
  and return 0.
- Build the existing skip hint from `$clone_base` and leave its text unchanged.
- Replace `pull_repo "$plugins_dir"` with: `fresh_clone_result "$plugins_dir"` when
  `plugins_clone_state` is `cloned`, else `pull_repo "$plugins_dir"`. Then
  `show_pull_result` and the rest as today.

**`mac_capture_step`.** Same pattern after the Darwin check: the `failed` row in the
missing-dir branch, the hint built from `$clone_base`, and `fresh_clone_result` instead
of `pull_repo` when `mac_capture_clone_state` is `cloned`. The step row and summary then
read `cloned · <sha> · installed ~/Applications/Bob Mac Capture.app`.

### Usage text

Replace `Missing siblings are skipped.` in the `--help` prose with:

```text
When a sibling it would install is missing and the run is interactive, it first
asks whether to clone it over SSH from git@github.com:bobs-org/<repo>.git;
otherwise that sibling is skipped.
```

Wrap it to match the surrounding paragraph. Leave the header `then` line as it is.

## Part 2 — justfile and docs

- **`justfile`.** Change the plain comment above the `install-all` recipe,
  `# Missing sibling checkouts are skipped.`, to
  `# Offers to clone missing sibling checkouts over SSH; otherwise they are skipped.`
  Leave the doc-comment line directly above the recipe, which `just --list` shows, and
  the recipe body unchanged.
- **`README.md`, "Install everything".** Replace the sentence
  `Missing siblings, a missing Obsidian vault, and the macOS-only app on other hosts are skipped.`
  with the following:
  - When a sibling it would install is missing and the command runs in a terminal, it
    asks `[y/N]` whether to clone it over SSH from `git@github.com:bobs-org/<repo>.git`
    into that sibling path. It asks before the bob-cli build starts.
  - bob-plugins is offered only when the Obsidian vault exists, and bob-mac-capture only
    on macOS.
  - A declined clone, a non-interactive run, a missing Obsidian vault, and the
    macOS-only app on other hosts are skipped. A failed clone is a failed step.

  Keep the following "Pull failures warn…" and "A failed step makes the command exit 1"
  sentences.

- **`docs/getting-started.md`.** In the install-all sentence, change "from sibling
  checkouts" to "from sibling checkouts (offering to clone any that are missing)".
- Do not edit `docs/plugins.md`, archived plans, `sase/` memory, bob-plugins, or Bob Mac
  Capture.

## Verification

Do **not** run the real `just install-all` or `scripts/install_all` against the real
home, and never clone into real directories. A real run would replace the installed
`bob`, write to `~/bob`, could restart Bryan's Obsidian, and would contact GitHub.

1. `just check-scripts` and `shellcheck scripts/install_all` are clean. Do not run
   `just all`; this change has no Rust.
2. `scripts/install_all --help` shows the new clone sentence, and
   `scripts/install_all --bogus` exits 2. `just --list` still shows the `install-all`
   description.
3. **Sandbox.** Reuse the temp-dir recipe from `plan:202610/install_all.md` and
   `plan:202610/install_all_restart_on_plugin_change.md`:
   - A temp dir `T` holds `upstream.git` and a `$T/bob-cli` clone with these changes
     committed. The siblings resolve to `$T/bob-plugins` and `$T/bob-mac-capture`.
   - Fake vault `$T/vault/.obsidian`.
   - Capture `CARGO_HOME` / `RUSTUP_HOME` before setting `HOME=$T/home`. Set
     `CARGO_INSTALL_ROOT=$T/cargo`, `BOB_DIR`, `BOB_PLUGIN_BACKUPS_DIR`,
     `XDG_STATE_HOME`, `XDG_DATA_HOME`, `ZDOTDIR`, and `SHELL`.
   - Use a `pgrep` stub reporting "not running", so no restart is attempted on Linux.

   Additions for this plan:
   - **No network, no SSH.**
     - `git clone --bare` the linked bob-plugins checkout (open it with
       `sase repo open bob-plugins`) to `$T/remotes/bob-plugins.git`.
     - Create a small fake bare repo `$T/remotes/bob-mac-capture.git` containing an
       executable `Scripts/install.sh` and a `justfile` whose `install dest identity:`
       recipe only echoes its arguments. Do not use the real bob-mac-capture; it builds
       Swift.
     - Export
       `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0="url.$T/remotes/.insteadOf" GIT_CONFIG_VALUE_0='git@github.com:bobs-org/'`
       so the SSH URLs resolve to those bare repos. `origin` still records the SSH URL.
     - Also export `GIT_SSH_COMMAND=false`, so any URL that escapes the rewrite fails
       instead of reaching GitHub.
   - **TTY.** Drive interactive runs with util-linux `script`:
     `printf 'y\n' | script -qec "$T/bob-cli/scripts/install_all" /dev/null`. This gives
     the script a pty on stdin and stdout. The pty echoes piped answers before the
     prompts appear, so assert on prompt text and outcomes, not on where the answer
     appears. Non-interactive runs use `</dev/null` without `script`.

   Scenarios (Linux unless noted; delete `$T/bob-plugins` before each one that needs it
   missing):
   1. **Non-interactive.** `</dev/null`, bob-plugins missing. No `MISSING CHECKOUTS`
      section and no `git clone`. The row is
      `– skipped · missing checkout · git clone git@github.com:bobs-org/bob-plugins.git …`.
      `$T/bob-plugins` is still absent. Exit 0. Repeat under `script` with
      `… install_all | cat` (stdin a TTY, stdout a pipe) and expect the same: no prompt,
      no hang.
   2. **Decline.** The answer is `n`, then a second run where the answer is an empty
      line. The section shows `bob-plugins is missing at …` and the `[y/N]` prompt, then
      `– not cloned · skipping bob-plugins`. No clone. Same skip row. Exit 0.
   3. **Accept.** The answer is `y`.
      - Exactly one prompt, for bob-plugins only (no bob-mac-capture prompt on Linux).
      - `$ git clone git@github.com:bobs-org/bob-plugins.git …/bob-plugins`, then
        `✓ cloned · <sha>`, all before the `BOB-CLI  1/3` banner.
      - The plugin step shows `⇣  clone  ✓ cloned · <sha>` and no
        `git -C …bob-plugins pull` command line. The sync copies files.
      - The summary row starts `cloned · <sha>`.
        `git -C $T/bob-plugins remote get-url origin` is
        `git@github.com:bobs-org/bob-plugins.git`. Exit 0.
      - Rerun interactively: no prompt, and the plugin row is `up to date`.
   4. **Clone fails.** Move `$T/remotes/bob-plugins.git` away. The answer is `y`.
      - git's error, then `✗ clone failed`.
      - The plugin row is
        `✗ clone failed · retry: git clone git@github.com:bobs-org/bob-plugins.git …`.
      - The final line is `✗  1 STEP FAILED`. `$T/bob-plugins` is absent. Exit 1.
      - Restore the remote afterwards.
   5. **No vault.** `BOB_DIR` points at a directory without `.obsidian`, bob-plugins is
      missing, interactive. No prompt; today's skip hint row. Exit 0.
   6. **Self-update re-exec.**
      - From a second clone of `upstream.git`, push a harmless commit.
      - Run interactively with bob-plugins missing; the answer is `y`.
      - Exactly one `MISSING CHECKOUTS` section and one `git clone`, both before the
        bob-cli pull line. The pull shows `1 new commit`. After the re-exec, there is no
        second prompt, and the plugin row still says `cloned · <sha>`.
      - Repeat with another pushed commit and the remote moved away. After the re-exec,
        the plugin row is still `✗ clone failed …`. Exit 1.
   7. **Re-exec from an older installer.**
      - Run the script under `script` with `BOB_INSTALL_ALL_HEADER_DONE=1`,
        `BOB_INSTALL_ALL_SELF_PULL=current`, and
        `BOB_INSTALL_ALL_SELF_PULL_DETAIL='up to date · test'`, without
        `BOB_INSTALL_ALL_CLONES_OFFERED`, and with bob-plugins missing. The answer is
        `y`.
      - No `INSTALL ALL` header. The section appears, followed by a `BOB-CLI  1/3`
        banner, then the `just … install` command line.
      - Rerun with `BOB_INSTALL_ALL_CLONES_OFFERED=1` added: no prompt.
   8. **Darwin.**
      - Put a `uname` stub printing `Darwin` and an `osascript` stub first on `PATH`.
        The `osascript` stub prints `false` for any `is running` script, logs its argv,
        and never calls the real `/usr/bin/osascript`.
      - Both siblings missing; answers `y\ny\n`.
      - Two prompts, bob-plugins first, then bob-mac-capture. Both clone.
      - The mac-capture row is
        `✓ cloned · <sha> · installed ~/Applications/Bob Mac Capture.app`. The fake
        justfile echo appears. Obsidian is `not running`. Exit 0.
   9. **`NO_COLOR=1` interactive run.** The section, prompt, and glyph lines contain no
      ANSI escapes.

4. **The real SSH clone and the real Mac can't be exercised on this host.** Say so in
   the final report. Ask Bryan to run `just install-all` once on the MacBook, where
   bob-mac-capture is missing. Even if that first run goes through the older-installer
   re-exec, it should ask to clone bob-mac-capture into
   `~/projects/github/bobs-org/bob-mac-capture` over SSH and then install it.
