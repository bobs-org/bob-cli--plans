---
tier: tale
title: Require bob config to list every plugin bob plugins sync deploys
goal:
  bob plugins sync deploys only plugin ids listed in bob's config, and Bryan's chezmoi
  bob config lists the 11 plugins he deploys today.
size: medium
decisions:
  config_key:
    ask: Which bob config key should hold the plugin allowlist?
    choices:
      sync:
        "Ids live under `plugins: sync:`; no clash with the list VAULT column's
        enabled/disabled"
      enabled:
        "Ids live under `plugins: enabled:`, matching the request; `enabled` then means
        two things"
    default: sync
    why:
      bob already says enabled for community-plugins.json; a second meaning invites
      confusion
    answer: sync
  list_marks:
    ask:
      Should `bob plugins list` and install-all show which repo plugins bob's config
      leaves out?
    default: true
    why:
      list is where users find ids to add; install-all shouldn't flag drift sync never
      touches
    answer: true
proposed_by: bbugyi200.athena.1c5.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 08:51:11
status: wip
---

# Require bob config to list every plugin `bob plugins sync` deploys

## Goal

`bob plugins sync` currently deploys every plugin folder under `<repo>/plugins/`. After
this change it deploys only plugin ids that bob's config file lists explicitly. A
missing list is a setup error, not "sync everything". Bryan's chezmoi-managed bob config
gains the list, naming every plugin he deploys today.

Below, `plugins.sync` is the config key (decision `config_key`). If the reviewer picks
`enabled`, use `plugins.enabled` everywhere `plugins.sync` appears: code constants,
messages, docs, tests, and the chezmoi config. The JSON field names `configured` and
`not_configured` stay the same under both choices.

## Facts the implementer can rely on

- Sync lives in `src/native/plugins/` (`cli.rs` builds clap and resolves the inputs,
  `sync.rs::sync_plugins` walks `<repo>/plugins/*`, `scan.rs` holds list's
  `scan_plugins` and `read_manifest`, `model.rs` holds `SyncOptions`, the reports, and
  the JSON result types, and `render.rs` prints tables). Plugin id = manifest `id`,
  falling back to the folder name.
- Bob config loading lives in `src/native/config/`. `mod.rs` holds `config_path()`
  (`BOB_CONFIG_FILE`, then `$XDG_CONFIG_HOME/bob/config.yml`, then
  `~/.config/bob/config.yml`), `ConfigError::{Read, Invalid}`, and `RawConfig`.
  `RawConfig` has no `deny_unknown_fields`, so released bob builds ignore a new
  top-level `plugins:` key. `dependencies.rs` and `archive.rs` are the small per-block
  parser pattern to copy, and `themes.rs::display_config_path()` renders the path with
  `~`.
- `scripts/install_all` (`plugin_step`) runs
  `bob plugins sync --dry-run --format json --no-pull --repo …`, then a real sync, then
  `bob plugins list --format json`. The verify step reads the top-level `count`,
  `synced`, and `drift` values. The `plugin_sync_rows` helper already parses JSON with
  `${BOB_INSTALL_UI_PYTHON}`.
- CLI integration tests (`tests/cli/plugins.rs`) run `bob` through
  `support::bob_command()`, which sets `BOB_CONFIG_FILE` to a missing file. Every
  existing sync test therefore needs a config fixture after this change. The unit tests
  (`src/native/plugins/tests.rs`) build `SyncOptions` through the `options()` helper.
- Shell completion for `-p` (`completion/providers.rs::plugins`) lists repo plugin
  folders. It is out of scope; see Non-goals.
- The live vault on athena has all 11 bob-plugins plugins deployed (each with a
  `.bob-deploy.json` stamp) and enabled in `community-plugins.json`: `block-id-prompt`,
  `bob-articles`, `bob-command-mode`, `bob-emoji`, `bob-ledger-tools`, `bob-mail`,
  `bob-navigation-hotkeys`, `bob-project-tasks`, `bob-vim-surround`,
  `bob-vim-yank-flash`, and `task-status-cycler`. Every host shares the same vault
  through Git and the same plain (non-template) chezmoi file
  `home/dot_config/bob/config.yml`.

## Config contract

```yaml
plugins:
  sync:
    - bob-ledger-tools
    - bob-navigation-hotkeys
```

- `plugins.sync` is a YAML sequence of plugin ids. Each entry must be a string, and is
  trimmed. A blank entry, a non-string entry, or a duplicate (after trimming) is a
  config error that names `plugins.sync`, the config path, and the offending entry or
  index.
- `plugins` must be a mapping when present, and `plugins.sync` must be a sequence when
  present. Anything else is a config error.
- If the config file is missing, the `plugins` block is absent or null, or
  `plugins.sync` is absent or null, the list is **unset**. Sync treats an unset list as
  a setup error (see below).
- `plugins.sync: []` is valid and means "deploy nothing".
- Unknown sibling keys under `plugins:` are ignored, not rejected. Hosts on the daily
  release train can run different bob versions against one shared config, and this
  matches the forward-compatible `capture:` precedent. A typo in the `sync` key itself
  still fails loudly, because the list is then unset.
- Add `src/native/config/plugins.rs` with `PluginsConfig { sync: Option<Vec<String>> }`
  (config order kept for messages), `load_plugins_config(path)` (a missing file gives an
  unset list, and other read errors give `ConfigError::Read`), and a
  `parse_plugins_config(text, path)` that unit tests can call. Add
  `plugins: Option<serde_yaml::Value>` to `RawConfig`, and re-export from
  `config/mod.rs` like the sibling blocks.

## Sync behavior

1. **Unset list or invalid config.** Before the repo pull, `run_sync` resolves the repo,
   then loads the plugins config from `config_path()`. On a config error, or an unset
   list, it exits **2** before any pull, lock, read of vault files, or write:
   - Table mode writes the message to stderr with the usual `bob plugins: ` prefix.
   - JSON mode prints `{"ok": false, "error": "<message>"}` on stdout.
   - For an unset list, the message names the config path (via `display_config_path()`),
     states that `bob plugins sync` deploys only the ids listed in `plugins.sync`, and
     ends with a ready-to-paste YAML snippet listing every plugin id in the current
     checkout (no pull). Add a small id-only helper beside `read_manifest` in `scan.rs`,
     and reuse it in sync. If the repo is unreadable, the snippet shows one placeholder
     id and points at `bob plugins list`.
   - Invalid config gets the `ConfigError` text unchanged.
2. **Filtering.** `SyncOptions` gains `configured: BTreeSet<String>` and the config path
   used in messages. In `sync_plugins`, a repo plugin whose id is not in `configured`
   gets no copies, stamps, backups, or guard evaluation. Its id goes to a new
   `SyncReport.not_configured` list (sorted). Manifest read and parse errors stay issues
   for every folder, as they are today.
3. **Listed id missing from the repo.** When `-p` is not given, each id in
   `plugins.sync` with no matching repo plugin adds one issue:
   `plugins.sync in <config> lists <id>, but <repo>/plugins has no plugin with that id`.
   The other plugins still sync, and the run exits 1. This matches how one plugin's
   failure leaves the rest running today.
4. **`-p/--plugin ID`.** The flag still selects one id:
   - If the id is in the repo but not in `plugins.sync`, sync copies nothing and adds
     one issue:
     `plugin <id> is not listed in plugins.sync in <config>; add it there to sync it`.
     The run exits 1.
   - If the id is not in the repo, the existing `plugin not found in repo: <id>` issue
     applies unchanged.
   - With `-p`, `not_configured` stays empty, and the missing-from-repo check runs only
     for the selected id.
5. **Dry run.** The same decision path applies. A dry run reports `not_configured` and
   every issue above, and writes nothing.
6. **Output.**
   - Table: after the plugin rows, print one row per unlisted id, using a neutral/dim
     glyph and the text `not in plugins.sync; skipped`. Then append
     ` · N not configured` to the summary line when N > 0.
   - JSON: `SyncResult` gains a top-level `not_configured: [ids]` that is always present
     (possibly `[]`). This is an additive field.
   - The existing `skipped` count keeps its meaning (dirty vault files).
7. **Exit codes.** `0` means success, including an empty list. `1` means any issue,
   including a listed id missing from the repo and an unlisted `-p`. `2` means a config
   error or an unset list (and clap usage errors, as today).

## List behavior

> [!decision] list_marks `scan_plugins` takes the loaded allowlist as an `Option` and
> still lists every repo plugin. `PluginEntry` gains `configured: bool` (always
> serialized). Table rows whose id the list leaves out get a dim detail line at
> `DETAIL_INDENT`: `not in plugins.sync; bob plugins sync skips it`. Header, footer, and
> counts stay unchanged.
>
> When the list is unset or the config is invalid, list does not fail. Instead:
>
> - It writes one stderr line: for an unset list,
>   `bob plugins: <config> has no plugins.sync list; bob plugins sync deploys nothing until you add one`,
>   and for an invalid config, the error text.
> - Every row gets `configured: false`, and the per-row detail lines are suppressed.
> - The exit status is unchanged.
>
> List stays a read-only discovery tool that helps repair the config, and sync remains
> the gate.
>
> In `scripts/install_all` `plugin_step`, the verify step computes the in-sync and drift
> figures from rows with `configured: true` only. Parse them with the same
> `${BOB_INSTALL_UI_PYTHON}` pattern as `plugin_sync_rows`, keeping the existing detail
> wording and the `$synced/$count` form over configured rows. When the sync preview
> reports a non-empty `not_configured`, also print one `dim_row` line naming those ids.

> [!decision] list_marks = no `scan_plugins`, `PluginEntry`, list output, and the
> install_all verify step stay unchanged.

Under both choices, update `scripts/install_all` so that a failed dry-run preview no
longer hides bob's reason. When `bob plugins sync --dry-run --format json` exits
non-zero, extract the JSON `error` string (same Python pattern as `plugin_sync_rows`)
and print it with `dim_diagnostics` under the existing `could not preview plugin sync`
row. Then a missing list on a freshly updated host reads as an actionable message. Keep
`bash -n` clean (`just check-scripts`).

## CLI help

Update `cli.rs` help text:

- In the top-level and `sync` long help, state that sync deploys only plugin ids listed
  in `plugins.sync` in bob's config file, and that a missing list exits 2 with a
  ready-to-paste snippet.
- Change the `-p/--plugin` help to say it syncs one id that must also be listed in
  `plugins.sync`.
- Add an example such as `bob plugins sync -d` if it helps.
- If `list_marks` is on, mention the `configured` marking in the list long help.

Keep options alphabetical and do not add flags (`sase/memory/cli_rules.md`). Regenerate
the golden CLI reference with `just docs-gen`.

## Implementation steps

1. Add `config/plugins.rs` and its `RawConfig` field and re-exports, with unit tests:
   - Unset cases: missing file, absent block, `plugins: null`, and `sync: null`.
   - A parsed list, trimming, and the empty list.
   - Errors for a non-mapping `plugins`, a non-sequence `sync`, a non-string entry, a
     blank entry, and a duplicate. Each error message names `plugins.sync` and the path.
   - An unknown sibling key is ignored.
2. Extend `model.rs`: `SyncOptions.configured` and the config path,
   `SyncReport.not_configured`, `SyncResult.not_configured`, and (if `list_marks` is on)
   `PluginEntry.configured`.
3. Implement filtering, missing-id issues, and the `-p` rules in `sync.rs`, and the
   id-only helper in `scan.rs`.
4. Wire `run_sync` in `cli.rs`: load the config before `maybe_pull_repo`, give exit 2 on
   an unset list or error (snippet from the helper), and fill `SyncOptions`. If
   `list_marks` is on, wire `run_list` to load the config tolerantly as described above.
5. Render the `not_configured` rows and summary clause in `render.rs`. If `list_marks`
   is on, also render the list detail line.
6. Make the `scripts/install_all` changes above.
7. Tests:
   - Unit (`plugins/tests.rs`): change `options()` so existing tests keep configuring
     every plugin they write. For example, derive the set from the repo's plugin folders
     at call time, or pass ids explicitly. Make sure tests that write plugins after
     building options still configure them. Add tests for:
     - An unlisted plugin is skipped, reported in `not_configured`, and leaves no vault
       folder.
     - A listed id missing from the repo gives an issue while other plugins sync.
     - `-p` on an unlisted id gives an issue and no writes.
     - An empty list gives no issues and nothing synced.
     - A dry run carries `not_configured`.
     - The sync JSON shape includes `not_configured`.
     - If `list_marks` is on, list rows carry `configured`.
   - Integration (`tests/cli/plugins.rs`): add a helper that writes a config file
     listing the fixture ids, and set `BOB_CONFIG_FILE` on every sync invocation. Keep
     the list tests working with the default missing config. Add tests for:
     - With a missing config, table sync exits 2 with the stderr snippet naming the
       fixture ids, and the vault stays untouched.
     - JSON sync with a missing config exits 2 with an `ok: false` object.
     - An invalid `plugins.sync` exits 2 naming the key.
     - JSON `not_configured` is reported.
     - If `list_marks` is on: list JSON `configured`, plus the list stderr notice with
       exit 0 when the config is missing.
8. Docs (follow `sase/memory/docs.md`; contracts own exact behavior):
   - `docs/plugins.md`: Commands intro, Discovery (the config file as a third input),
     Sync section, `-p` option, Exit status (new exit 2), sync JSON (`not_configured`),
     and, if `list_marks` is on, Columns and list JSON (`configured`).
   - `docs/reference/configuration.md`: a new `### plugins` section after `### copy`
     with a key table for `plugins.sync`, and a sentence in the "To install plugins"
     paragraph that sync needs the list.
   - `docs/reference/json-outputs.md`: the `bob plugins sync` row mentions the unlisted
     plugins.
   - `docs/guides/deploy-obsidian-plugins.md`: "Before you start" adds the
     `plugins.sync` step with a YAML snippet, and "If it goes wrong" covers the exit-2
     setup error.
   - `docs/start/obsidian.md` step 3: show the YAML to add before the sync command. Use
     the two plugins that page already tells readers to enable, and keep any bash block
     that touches real config under a `tutorial-test: skip` marker.
   - `docs/guides/install-everything.md`: one sentence that the plugin step deploys the
     ids in `plugins.sync`.
   - Run `just docs-gen`, then `just docs-check`.
9. bob-plugins docs: open the linked repo with `/sase_repo`
   (`sase repo open bob-plugins`). In `docs/develop/deployment.md`, add a short
   paragraph near the top: sync deploys only ids listed in `plugins.sync` in bob's
   config, so a new plugin must be added there before `bob plugins sync` deploys it, and
   an unlisted `-p` id is refused. Do not name Bryan's chezmoi paths in that public doc.
   bob-cli mounts this page at build time (git-ignored in bob-cli), so no bob-cli copy
   needs editing.
10. chezmoi config: open it with `sase repo open chezmoi` (read its `AGENTS.md`). In
    `home/dot_config/bob/config.yml`, add a commented `plugins:` block next to the
    existing `copy:` block (or in the same style). It lists all 11 ids above, sorted,
    under `sync:`. The comment explains that `bob plugins sync` deploys only these ids,
    that a new bob-plugins plugin must be added here first, and points at "Bob Plugins"
    in bob-cli `docs/plugins.md`. Note in the comment that older bob builds ignore the
    key, so it is safe on every host before the release train ships the new bob.

## Verification

Run these gates in bob-cli:

- `just all` (fmt, lint, test)
- `just check-scripts`
- `just docs-gen`, leaving a clean diff afterward except for intended golden updates
- `just docs-check`

Then run read-only checks against the real vault and plugins repo. Dry runs take no lock
and write nothing.

- `BOB_CONFIG_FILE=<opened chezmoi checkout>/home/dot_config/bob/config.yml cargo run -q -- plugins sync --dry-run --no-pull -f json`
  should report `ok: true` with `"not_configured": []` and all 11 plugins.
- `BOB_CONFIG_FILE=/nonexistent/bob.yml cargo run -q -- plugins sync --dry-run --no-pull`
  should exit 2 and print the snippet listing all 11 ids.
- If `list_marks` is on: with the chezmoi config,
  `cargo run -q -- plugins list --no-pull -f json` should show `configured: true` on
  every row.

Do not run a real (non-dry-run) sync for verification.

## Commits and rollout

- The bob-cli commit is a CLI contract break. Use a `!` subject, for example
  `feat(plugins)!: require plugins.sync allowlist for bob plugins sync`. Do not edit
  `Cargo.toml` versions or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- The chezmoi and bob-plugins edits are separate repository obligations in the same
  turn's final declaration. Commit each with a conventional subject, such as
  `feat(bob): list synced plugins in plugins.sync` and
  `docs(deploy): note plugins.sync allowlist`.
- The chezmoi repo requires `chezmoi update -a --force` after its commit lands. The
  finalizer lands it after the coding turn, so the final response must tell Bryan to run
  that on each host (athena, apollo, and the Mac). Do not hand-copy the file into
  `~/.config/bob/`. Order is safe either way: old bob ignores the new key, and new bob
  without the key fails with an actionable exit-2 snippet.

## Non-goals

- Enabling or disabling plugins in Obsidian's `community-plugins.json`. Sync's existing
  `disabled_warning` is unchanged.
- Removing or uninstalling vault copies of plugins dropped from the list. They are
  simply no longer updated.
- Filtering `-p` shell completion to listed ids.
- Changing the bob-plugins Obsidian smoke gate. It reads list JSON rows and keeps
  passing while every repo plugin is listed. A new repo plugin left out of the list will
  fail that gate until it is listed, which is the intended reminder.
- A `--all` or other bypass flag. The point of the change is an explicit list.
