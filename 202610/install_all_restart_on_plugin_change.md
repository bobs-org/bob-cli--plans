---
tier: tale
title: Merge install-all-and-restart into install-all, restarting only on plugin changes
goal:
  just install-all restarts a running Obsidian only when bob plugins sync changed the
  vault (or an earlier run's restart is still owed), and just install-all-and-restart is
  gone.
size: medium
proposed_by: bbugyi200.athena.0wh
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0wh](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0wh.md)
- **COMMITS:**
  - [54ff70a](https://github.com/bobs-org/bob-cli/commit/54ff70acf959abd46a30e728cd5e102862546157)
    — feat(install-all): restart Obsidian when plugin sync copies files

# Plan: Fold the Obsidian restart into `just install-all`, gated on plugin changes

## Goal

- Remove the `just install-all-and-restart` recipe.
- Make `just install-all` restart a running Obsidian at the end of a clean run, but
  **only when `bob plugins sync` changed the vault**, meaning it copied at least one
  managed plugin file into `<vault>/.obsidian/plugins/`. It also restarts when an
  earlier run's change is still waiting for a restart (see call-out 3).
- Everything else stays as it is: the per-OS restart mechanics, skipping the restart
  when any step failed, and never launching an Obsidian that isn't running.

## Design call-outs

1. **"Changed" means sync copied a file, not that the bob-plugins pull moved HEAD.**
   This is the literal request ("made changes to my Obsidian vault"), and it is the more
   accurate signal:
   - A pull that only touches sources, docs, or tests leaves the deployed bytes
     identical, so no restart is needed.
   - An up-to-date checkout with a stale vault still needs one.
2. **Detect changes with a new `bob plugins sync --format json`, run as a `--dry-run`
   preview right before the real sync.**
   - The streamed, human-readable sync output (diffs, backup paths) stays exactly as it
     is today.
   - A dry run follows the same decision path as the real sync, dirty-file guard
     included.
   - A real sync that exits 0 has made every copy it planned, because any failed copy
     becomes an issue and the command exits 1.

   Rejected alternatives:
   - **Parsing the table footer:** it depends on color and the separator.
   - **Hashing vault plugin files in shell:** the hash tool differs between macOS and
     Linux, it needs the manifest-id → folder mapping, and `data.json` adds noise.
   - **Diffing `bob plugins list` states before and after:** it misses partial updates
     where one file is dirty-skipped and another is copied. Dirty drift would also
     trigger false restarts.

3. **A pending-restart marker keeps the "only if changed" rule from losing restarts.**
   - **Problem:** suppose a run syncs new plugins but skips the restart because another
     step failed or the restart itself failed. The rerun finds the vault already in sync
     and would never restart.
   - **Marker file:**
     `${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli/install-all/obsidian-restart-pending`.
     This is bob-cli's existing state root (`bob_env::bob_cli_state_dir()`), shared with
     the plugin backups and the completion manifest.
   - **Written** just before a sync that will copy files.
   - **Removed** once Obsidian was restarted, the restart was handed off, or Obsidian
     was found not running.
   - **Kept** when the restart was skipped or failed, so the next `just install-all`
     retries.
4. **Remove `-r/--restart-obsidian` from `scripts/install_all` outright.** There is no
   hidden alias for the recipe or the flag, because Bryan asked to get rid of the
   recipe. The CLI rule "never remove a command" covers `bob` commands, not dev recipes
   or this script's flags. `-r` becomes an unknown argument (usage on stderr, exit 2).
5. **Keep "any failed step skips the restart"** from the original install-all design. It
   avoids loading new plugins against, for example, a failed bob-cli install.

## Part 1 — `bob plugins sync --format json` (Rust)

Files: `src/native/plugins/{cli.rs,model.rs,render.rs,tests.rs}`,
`tests/cli/plugins.rs`, and `docs/plugins.md`.

- **`cli.rs`**
  - Add `.arg(format_arg())` to `sync_command()`, keeping options alphabetical by long
    name: backup-dir, bob-dir, dry-run, force, format, no-pull, plugin, repo. The short
    alias `-f` already exists.
  - Add `bob plugins sync --dry-run -f json` to the sync `after_help` examples.
  - Add one sentence about `-f json` to the sync `long_about`.
  - Completion derives from `build_cli()`, so it picks up the option automatically.
- **`run_sync`:** branch on `OutputFormat::from_matches(matches)`.
  - **Table:** unchanged.
  - **Json:** print no table and no diffs. Stdout carries exactly one JSON object.
    - No issues: print the success object and exit 0.
    - Issues: print `{"ok": false, "error": "<issues joined with '; '>"}`, don't also
      echo the issues to stderr, and exit 1. This mirrors `run_list`'s JSON branch. The
      copies that could happen still happen, as in table mode.
    - Pull diagnostics already go to stderr, so they don't corrupt stdout.
- **`model.rs`:** add `Serialize` result types alongside `PluginsResult`, for example:
  - `SyncResult { ok, dry_run, repo, bob_dir, copied, skipped, unchanged, plugins }`
  - `PluginSyncResult { id, files }`
  - `FileSyncResult { name, action, backup: Option<String> }`

  Derive `Serialize` on `FileAction` with `#[serde(rename_all = "snake_case")]`, giving
  `created`, `updated`, `forced`, `unchanged`, `skipped_dirty`, and `failed`. Add
  `SyncReport::result(dry_run)` and `SyncReport::issue_summary()`, reusing the existing
  `copied()`, `skipped()`, and `unchanged()`. `backup` is the backup path when the file
  is backed up (or, in a dry run, would be), otherwise `null`.

- **`render.rs`:** add `sync_success_json(&SyncResult) -> String` next to
  `success_json`.

Success shape, to document in `docs/plugins.md`:

```json
{
  "ok": true,
  "dry_run": false,
  "repo": "/home/bryan/projects/github/bobs-org/bob-plugins",
  "bob_dir": "/home/bryan/bob",
  "copied": 1,
  "skipped": 0,
  "unchanged": 5,
  "plugins": [
    {
      "id": "bob-project-tasks",
      "files": [
        { "name": "manifest.json", "action": "unchanged", "backup": null },
        {
          "name": "main.js",
          "action": "updated",
          "backup": "/home/bryan/.local/state/bob-cli/plugin-backups/20261004-120000/bob-project-tasks/main.js"
        }
      ]
    }
  ]
}
```

In a dry run, `copied` counts the files that _would_ be copied (the table's "to copy").

### Tests

- **`tests/cli/plugins.rs`.** These use `write_plugins_fixture`: alpha is in sync,
  beta's `main.js` is stale, and gamma is missing from the vault.
  - `plugins_sync_json_reports_file_actions` runs a real
    `sync -f json -B <backups> -r <repo> -b <vault>` with `BOB_NOW` pinned.
    - Stdout parses as JSON, has no ANSI codes, and has no table header text.
    - Counts: `ok: true`, `dry_run: false`, `copied: 3`, `skipped: 0`, `unchanged: 3`.
    - beta `main.js` is `updated`, with `backup` = `<backups>/<ts>/beta/main.js`.
    - Both gamma files are `created` with `backup: null`.
    - The alpha files are `unchanged`.
    - The vault files were actually written.
  - `plugins_sync_dry_run_json_writes_nothing` runs `-d -f json`. Expect `dry_run: true`
    and `copied: 3`, with the vault and the backup dir untouched.
  - `plugins_sync_json_reports_errors_as_object` runs `-p nope -f json`. Expect exit 1
    and stdout `{"ok": false, "error": "…plugin not found in repo: nope"}`.
- **`src/native/plugins/tests.rs`:** add `sync_json_shape_is_stable` next to
  `json_shape_is_stable`, pinning the top-level keys and the snake_case action names.

## Part 2 — `scripts/install_all`

### Usage and arguments

```text
Usage: scripts/install_all [-h]

Pull and install bob-cli from this checkout, then pull and deploy the sibling
bob-plugins and bob-mac-capture checkouts that live next to it. Missing siblings
are skipped. When the plugin sync changes the vault, a running Obsidian is then
restarted so it loads the new plugins. Usually run through `just install-all`.

Options:
  -h, --help  Show this help and exit.
```

Delete the `-r|--restart-obsidian` case and the `restart_obsidian` variable.

### State

- `PLUGINS_CHANGED=0`
- `restart_marker="${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli/install-all/obsidian-restart-pending"`
- **`mark_restart_pending`:** run `mkdir -p` on the marker's directory, then
  `: >"$restart_marker"`. This is best-effort (`|| true`): the current run still
  restarts through `PLUGINS_CHANGED`, and only the cross-run memory is lost.
- **`clear_restart_pending`:** `rm -f "$restart_marker"`.
- Add a short comment explaining why the marker exists (call-out 3).

### Header

Replace the conditional `then` line with an unconditional one:

```text
  then     restart Obsidian if the plugin sync changes the vault
```

### `plugin_step`

Insert these steps after `bob_bin` is resolved and before the existing real sync:

1. Echo the command with
   `command_line "bob plugins sync --dry-run --format json --no-pull --repo <dir>"`.
   Then capture
   `preview="$("$bob_bin" plugins sync --dry-run --format json --no-pull --repo "$plugins_dir")"`.
   On a non-zero exit, record ✗ `"$pull_detail · could not preview plugin sync"`
   (`FAILURES++`, `add_summary … fail`) and return without running the real sync. This
   mirrors the existing "could not verify plugin sync" failure.
2. Set `pending="$(extract_json_integer copied "$preview")"`. If it is empty, fail the
   same way with `"$pull_detail · could not read plugin sync preview"`. `"copied"` is a
   unique top-level key; the per-file objects only use `name`, `action`, and `backup`.
3. If `pending > 0`, set `PLUGINS_CHANGED=1` and call `mark_restart_pending` **before**
   the real sync. A sync that fails partway, or an interrupted run, then still leaves
   the restart owed.
4. Keep the real sync and the post-sync `bob plugins list` drift check exactly as they
   are.
5. **Summary detail:** when `pending > 0`, insert `· N files copied` (`1 file copied`
   when singular) right after the pull detail in the ✓ and ⚠ details, for example
   `up to date · 2 files copied · 6/6 plugins in sync`. Runs that copy nothing keep
   today's text.

### `obsidian_step`

Call it unconditionally after `mac_capture_step` by dropping the
`if (( restart_obsidian ))` guard.

Add this preamble **before** the existing `FAILURES` check:

- If `PLUGINS_CHANGED == 1`, set `reason='plugins changed'`.
- Otherwise, if the marker file exists, set
  `reason='restart pending from an earlier run'`.
- Otherwise, show the row `– plugins unchanged · restart not needed` with
  `add_summary obsidian skip`, and return without calling osascript or pgrep.

Change the failure skip detail to
`$reason; not restarted: a step failed — fix it and rerun just install-all`. The marker
stays in place, so the rerun restarts even though the vault is in sync by then.

Marker lifecycle inside the existing OS branches:

- **Remove the marker** (`clear_restart_pending`) on:
  - **Not running, on both OSes.** Unify the detail to
    `not running · plugins load on next launch`.
  - **Success:** ↻ `restarted` (macOS) and ↻ `restart requested` (Linux).
  - **⚠ `relaunch requested; not confirmed yet`.** Obsidian did quit, so the old plugins
    are already unloaded.
  - **⚠ Linux running-without-CLI manual hint.** The restart is handed to Bryan, and
    removing the marker avoids repeating the hint on every run.
- **Keep the marker** on every ✗ outcome so the next run retries:
  - could not check whether Obsidian is running or quit;
  - quit failed;
  - did not quit within 30s;
  - relaunch failed;
  - Linux CLI restart failed;
  - unsupported OS.
- When the reason is the marker alone (`PLUGINS_CHANGED == 0`), append
  ` · pending from an earlier run` to the ↻ details. The summary then explains a restart
  that had no new sync.

### Unchanged

- The restart mechanics.
- The self-re-exec: the marker is file-based, and `PLUGINS_CHANGED` is computed after
  the exec decision.
- bash 3.2 compatibility, so no new bash-4 features.
- Exit codes.
- The mode stays 100755.

## Part 3 — justfile and docs

- **`justfile`:** delete the `install-all-and-restart` recipe and its comment.
  `just --list` shows only the comment line directly above a recipe, so make that line a
  complete description:

  ```just
  # Missing sibling checkouts are skipped.
  # Pull + install bob-cli, bob-plugins, and bob-mac-capture; restart Obsidian if plugins changed.
  install-all:
      @scripts/install_all
  ```

- **`README.md`, "Install everything":** replace the `just install-all-and-restart`
  paragraph with one that says:
  - After a clean run, `just install-all` restarts a running Obsidian only when
    `bob plugins sync` copied at least one file into the vault. It previews the sync
    with `bob plugins sync --dry-run --format json`.
  - On macOS it gracefully quits and relaunches Obsidian, and the first run may ask for
    Automation permission. On Linux it uses the Obsidian CLI `restart` when that CLI is
    registered at `~/.local/bin/obsidian`.
  - A stopped Obsidian is never launched.
  - If a failed step or a failed restart leaves the new plugins unloaded, the next run
    retries the restart. The pending marker lives under
    `~/.local/state/bob-cli/install-all/`.
- **`docs/getting-started.md`:** extend the install-all sentence with "…from sibling
  checkouts, restarting a running Obsidian when the plugins change."
- **`docs/plugins.md`:**
  - Add `-f, --format <FORMAT>` to the sync Options list, right after `-F, --force`:
    "`table` (default) or `json`".
  - Add a "### JSON output" subsection under Sync with the shape above, the dry-run
    meaning of `copied`, and the `{"ok": false, "error": …}` error object.
  - Extend the existing install-all sentence: it previews the sync with
    `--dry-run --format json` to decide whether to restart Obsidian.
  - Add `bob plugins sync -d -f json` to Examples.
- **Leftovers:** grep for any remaining `install-all-and-restart` or `restart-obsidian`
  mentions outside `sase/` (whose archived plans stay untouched) and remove them.

## Verification

Do **not** run the real `just install-all` or `scripts/install_all` against the real
home. It would replace the installed `bob`, write to `~/bob`, and could restart Bryan's
Obsidian.

1. `just all` passes: fmt, clippy, and the tests, including the new JSON tests.
2. `just check-scripts` and `shellcheck scripts/install_all` are clean.
3. CLI surface:
   - `scripts/install_all --help` shows the new usage.
   - `scripts/install_all -r` and `scripts/install_all --restart-obsidian` exit 2.
   - `just --list` has no `install-all-and-restart` and shows the full `install-all`
     description.
4. **Sandboxed end-to-end run on Linux.** Use a temp dir `T`, following the original
   install-all plan's sandbox recipe:
   - Create `upstream.git`, then a `$T/bob-cli` clone with these changes committed.
   - Make `$T/bob-plugins` a clone of the linked bob-plugins checkout (open it with
     `sase repo open bob-plugins`).
   - Create a fake vault with `mkdir -p $T/vault/.obsidian`.
   - Set `CARGO_HOME` and `RUSTUP_HOME` to the real values **before** overriding `HOME`.
   - Set `CARGO_INSTALL_ROOT=$T/cargo`, `BOB_DIR=$T/vault`,
     `BOB_PLUGIN_BACKUPS_DIR=$T/backups`, `HOME=$T/home`,
     `XDG_STATE_HOME=$T/home/.local/state`, `XDG_DATA_HOME`, `ZDOTDIR`, and
     `SHELL=/bin/zsh`.

   Put a stub dir first on `PATH`:
   - a fake `pgrep` that exits 0 ("running") or 1 ("not running"), chosen per scenario;
   - an executable `$T/home/.local/bin/obsidian` that appends its arguments to a log
     file and exits with a configurable status.

   Because both `HOME` and `pgrep` are sandboxed, the real Obsidian CLI and process are
   never touched. Scenarios:
   1. **Fresh vault, Obsidian "running".**
      - The plugin row shows `N files copied`.
      - The Obsidian row shows `↻ restart requested`.
      - The stub log contains `restart`.
      - The marker is gone afterwards, and the exit status is 0.
   2. **Immediate rerun.**
      - The plugin row has no copied count.
      - The Obsidian row shows `– plugins unchanged · restart not needed`.
      - The stub is not called, and the exit status is 0.
   3. **One vault `main.js` edited, `pgrep` stubbed as "not running".** The fake vault
      isn't a Git repo, so the dirty guard doesn't apply.
      - The plugin row shows `1 file copied`.
      - The Obsidian row shows `– not running · plugins load on next launch`.
      - The marker is removed, and the stub is not called.
   4. **Failed step, then recovery.**
      - Edit a vault `main.js` again and put a failing `just` stub first on `PATH`.
        bob-cli shows ✗, and the plugin sync still runs with `$T/cargo/bin/bob` from
        scenario 1.
      - Expect `– plugins changed; not restarted: a step failed — …`, the marker
        present, and exit 1.
      - Rerun without the `just` stub, with Obsidian "running". The plugins are
        unchanged, the Obsidian row shows
        `↻ restart requested · pending from an earlier run`, the marker is removed, and
        the exit status is 0.
   5. **Restart failure, then retry.**
      - Change a vault file and make the stub CLI exit 1. Expect ✗
        `Obsidian CLI restart failed`, the marker kept, and exit 1.
      - Rerun with the stub exiting 0. Obsidian restarts and the marker is cleared.
   6. **Plain output.** Pipe one run through `| cat` and confirm there are no ANSI
      escapes.

5. **The macOS branch can't be exercised on athena.** Review the marker keep/remove
   placement there by reading the code. Note in the final report that Bryan should do
   one real `just install-all` on the MacBook.
