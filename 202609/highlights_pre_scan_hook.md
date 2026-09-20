---
tier: epic
title: Rename highlights pre-scan hook config and auto-scan from bob_xlib_pull
goal: '`bob highlights` reads the hook from `highlights.pre_scan_hook`, a new `-n|--no-hooks`
  flag ignores it, and `bob_xlib_pull` runs `bob highlights --no-hooks scan -w` itself
  on macOS so freshly pulled PDFs sync immediately instead of waiting for the 15-minute
  cron.

  '
phases:
- id: cli
  title: Rename the pre-scan hook config surface and add `--no-hooks`
  depends_on: []
  size: medium
  description: 'cli: rename `highlights.pre_scan_command` to `highlights.pre_scan_hook`
    (config key, env override, report labels, docs), reject the legacy names loudly,
    add the `-n|--no-hooks` flag to `bob highlights`, and export a hook-marker env
    var to the hook child process.'
- id: dotfiles
  title: Auto-scan from bob_xlib_pull and follow the rename in chezmoi
  depends_on:
  - cli
  size: medium
  description: 'dotfiles: update the managed bob config and `maybe_bob_highlights_sync`
    for the new names, make `bob_xlib_pull` run `bob highlights --no-hooks scan -w`
    after its pull on macOS with hook and cron-lock guards, extend the bashunit regression
    tests, and refresh the Highlights bridge documentation.'
proposed_by: bbugyi200.apollo.16
create_time: 2026-09-20 15:38:35
status: wip
bead_id: bob-cli-23
---

- **PROMPT:** [prompts/202609/highlights_pre_scan_hook.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/highlights_pre_scan_hook.md)
- **BEAD:** [bob-cli-23](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-23/README.md)

# Plan: Rename the Highlights pre-scan hook and auto-scan from `bob_xlib_pull`

## Background

`bob highlights scan` runs an optional shell hook before xlib intake. The hook is
configured as `highlights.pre_scan_command` in `~/.config/bob/config.yml` (managed in
the chezmoi repo as `home/dot_config/bob/config.yml`) and currently holds
`PATH="$HOME/bin:$PATH" bob_xlib_pull`. `bob_xlib_pull` (chezmoi
`home/bin/executable_bob_xlib_pull`) is a macOS-only script: it exits 0 immediately on
any non-Darwin `uname -s`, then probes athena and apollo in parallel over Tailscale SSH
and rsyncs their `~/bob/xlib/` queues into the MacBook's intake directory.

Today the only thing that turns a completed pull into reference notes is the MacBook's
15-minute cron job, `~/bin/maybe_bob_highlights_sync -w` (chezmoi
`home/bin/executable_maybe_bob_highlights_sync`), which shells out to
`bob highlights scan -w`. So a manual `bob_xlib_pull` leaves the pulled PDFs sitting in
`xlib/` until the next cron tick. The user wants the pull script to scan immediately.

Running a scan from inside the pull script is only safe if the scan does not re-enter
the hook, which is what the requested `--no-hooks` flag provides. Note the inverse
direction as well: when `bob_xlib_pull` is itself running _as_ the pre-scan hook (the
cron path), an inner scan would be pure duplicated work — bob is about to scan anyway,
and the inner scan would run with `-w` even when the outer scan did not ask for PDF
writes. That case needs a guard, which this plan implements with a marker environment
variable bob exports to the hook child.

## Decisions and assumptions

These go beyond the literal request; they are called out so they can be rejected at plan
review rather than discovered in the diff.

1. **The env override and report labels are renamed too.**
   `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND` becomes `BOB_HIGHLIGHTS_PRE_SCAN_HOOK`, and the
   `pre_scan_command:` lines that `scan` and `doctor` print become `pre_scan_hook:`. The
   env var exists only to override the renamed config field and its name embeds the old
   field name; leaving either behind would make the surface inconsistent.
2. **The legacy names are rejected, not silently ignored.** `RawHighlights` does not use
   `deny_unknown_fields`, so after the rename an unmigrated `pre_scan_command:` key
   would be silently dropped and the hook would quietly stop running. Both the legacy
   config key and the legacy env var therefore become hard errors that name the new
   spelling.
3. **`--no-hooks` disables the hook from every source** — config field and env override
   alike — and applies to `scan` and `doctor` (the only two subcommands that consult the
   hook).
4. **`--no-hooks` is accepted before or after the subcommand.** The requested invocation
   is `bob highlights --no-hooks scan -w`, so the flag is declared on the root
   `highlights` command; it is also declared on `scan` and `doctor` so the natural
   `bob highlights scan --no-hooks` works and the flag shows up in those subcommands'
   help. It is deliberately _not_ a clap `global(true)` arg: a global arg cannot also be
   declared per-subcommand, and it would advertise a meaningless `--no-hooks` in
   `create`, `sync`, and `marker` help.
5. **Short alias `-n`**, per `sase/memory/cli_rules.md` ("give every public long option
   a short alias"). `-n` is unused by the root command and by `scan` and `doctor`.
6. **bob exports `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1` to the hook child**, so a hook
   script can tell it is running inside a scan. `bob_xlib_pull` uses it to skip its own
   scan. This is preferred over encoding the guard in the config command line (e.g.
   adding a variable assignment to the `pre_scan_hook` value) because it cannot be
   forgotten when the config is edited, and it is covered by bob's own test suite.
7. **The pull-triggered scan serializes with the cron scan** by acquiring the same
   `${TMPDIR:-/tmp}/maybe_bob_highlights_sync.lock` directory that
   `maybe_bob_highlights_sync` uses, and skips scanning when it is already held. Without
   this, a manual pull could run a second concurrent `bob highlights scan -w` against
   the same vault and git worktree. Skipping is not a regression: that is exactly
   today's behavior when the cron lock is contended.
8. **The pull-triggered scan always runs** (on macOS, when not invoked as the hook and
   the lock is free) — including when both remote queues were empty and when a probe or
   transfer failed — because local `xlib/` can still hold leftovers from an earlier run.

## Non-goals

- No change to the MacBook's crontab; the 15-minute `maybe_bob_highlights_sync -w` job
  stays as the fallback and is not managed in the chezmoi repo.
- No scan-level locking inside `bob` itself. Serialization stays in the shell scripts.
- No back-compatible acceptance of `pre_scan_command` (see decision 2).
- `bob-mac-capture` is a capture-only frontend and is not expected to touch
  `bob highlights`; its checkout is not available locally, so no change is planned
  there.

## Rollout order (for the user, after both phases land)

Apply the chezmoi change _before_ installing the new `bob` binary on the MacBook. In
that order the worst case is a short window where an old `bob` ignores the new
`pre_scan_hook` key and skips the pull; in the opposite order a new `bob` would
hard-fail every cron scan until `chezmoi update -a --force` runs. The same applies on
any other machine that runs `bob highlights scan`.

---

## Phase `cli`: Rename the pre-scan hook config surface and add `--no-hooks`

All work in this phase is in the bob-cli repo.

### `src/native/config.rs`

- Rename `HighlightsConfig::pre_scan_command` (field and accessor) to `pre_scan_hook`.
- In `RawHighlights`, rename the deserialized field to `pre_scan_hook: Option<String>`
  and add a detection-only field for the legacy key. Use
  `pre_scan_command: Option<serde_yaml::Value>` so a legacy key with a non-string value
  is still detected instead of producing a confusing serde type error.
- In `parse_highlights_config`, return `ConfigError::Invalid` when the legacy key is
  present, with a message that names the file and the new spelling, e.g.
  `"highlights.pre_scan_command in {path} was renamed; use highlights.pre_scan_hook"`.
  Keep the existing trim-and-drop-blank handling for the new key.
- Update the unit tests in this file: rename `parses_highlights_pre_scan_command` and
  `blank_highlights_pre_scan_command_disables_file_hook` to the `pre_scan_hook`
  spelling, and add a test asserting the legacy key is rejected with a message
  mentioning `pre_scan_hook`.

### `src/native/highlights_ref/mod.rs`

- Constants: rename `ENV_PRE_SCAN_COMMAND` to `ENV_PRE_SCAN_HOOK` with value
  `"BOB_HIGHLIGHTS_PRE_SCAN_HOOK"`, and add
  `ENV_LEGACY_PRE_SCAN_COMMAND: &str = "BOB_HIGHLIGHTS_PRE_SCAN_COMMAND"`. Keep the
  constant block sorted.
- Rename the `PreScanCommand` struct to `PreScanHook` and the helpers
  `configured_pre_scan_command`, `run_pre_scan_command`, `check_pre_scan_command`, and
  `pre_scan_command_from_os` to the `pre_scan_hook` spelling.
- `configured_pre_scan_hook(no_hooks: bool) -> Result<Option<PreScanHook>>`:
  - return `Ok(None)` immediately when `no_hooks`;
  - otherwise, if the environment defines `ENV_LEGACY_PRE_SCAN_COMMAND` at all (even
    empty), return an error naming `BOB_HIGHLIGHTS_PRE_SCAN_HOOK`;
  - otherwise keep today's precedence: `ENV_PRE_SCAN_HOOK` (empty disables) over the
    config file value.
- `run_pre_scan_hook`: keep `sh -c`, `current_dir(&config.bob_dir)`, null stdin,
  inherited stdout/stderr, and the dry-run `would-run` preview. Add
  `.env("BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK", "1")` to the child command.
- Change every `pre_scan_command:` stdout label (`run`, `would-run`, `none`, `ok`,
  `fail`) to `pre_scan_hook:`. Error text such as `"pre-scan command failed with ..."`
  may stay in prose form but should read "pre-scan hook" for consistency.
- New arg builder, matching the style of the neighboring builders:

  ```rust
  fn no_hooks_arg() -> Arg {
      Arg::new("no-hooks")
          .long("no-hooks")
          .short('n')
          .action(ArgAction::SetTrue)
          .help("Ignore the configured highlights.pre_scan_hook")
  }
  ```

  Register it on the root command in `build_cli()`, inside `with_scan_args` (declaration
  order stays alphabetical: `bob-dir`, `dry-run`, `jobs`, `lib-dir`, `no-hooks`,
  `ref-dir`, `verbose`, `write-pdfs`, `xlib-dir`), and directly on the `doctor`
  subcommand builder — not on `with_config_args`, which `marker` also uses.

- `run()`: read the flag from the root matches and OR it with the subcommand's own flag,
  then thread it through: `run_scan(sub_matches, no_hooks)` and
  `run_doctor(sub_matches, no_hooks)`, which pass it to `scan_library(..., no_hooks)`
  and `doctor_vault(config, no_hooks)`.
- `doctor_vault`: when `no_hooks`, print `pre_scan_hook: skipped (--no-hooks)` and
  record no failure; otherwise behave as today.
- `scan_library`: unchanged apart from calling `configured_pre_scan_hook(no_hooks)`.

### `tests/cli.rs`

Update the existing pre-scan tests (config YAML keys, `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND`
env names, and the `pre_scan_command:` label assertions), then add:

- `--no-hooks` before the subcommand skips a configured hook: sentinel script is never
  executed, no `pre_scan_hook:` line is printed, and the scan still writes notes for
  PDFs already in `lib/`.
- `--no-hooks` after the subcommand (`highlights scan --no-hooks`) behaves identically.
- `--no-hooks` also overrides `BOB_HIGHLIGHTS_PRE_SCAN_HOOK` when that env var is set.
- the hook child sees `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1` (hook script writes the value
  to a log file that the test reads).
- a config file containing `highlights.pre_scan_command` fails `scan` with exit 1 and a
  message naming `pre_scan_hook`.
- `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND` in the environment fails `scan` the same way.
- `highlights --no-hooks doctor` reports `pre_scan_hook: skipped (--no-hooks)` and still
  ends in `result: ok`.

### Docs

- `README.md`: the `bob highlights` usage block gains `[-n|--no-hooks]` on the `scan`
  and `doctor` lines; rename the `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND` entry in the
  Environment section to `BOB_HIGHLIGHTS_PRE_SCAN_HOOK` (alphabetical position is
  unchanged) and note that the legacy variable is now an error; document that bob
  exports `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1` to the hook; update the `pre-scan hook`
  prose near the config-file note and in the `scan` bullet.
- `docs/highlights-ref-sync.md`: update the YAML sample and hook prose, the env override
  paragraph, the `doctor` checklist line, and the dry-run/scan safety paragraphs;
  document `--no-hooks` where the pre-scan hook is described.
- `docs/vault-git-sync.md`: update the `highlights: pre_scan_command:` sample in the
  Highlights bridge section to `pre_scan_hook:`. (The behavioral description of
  `bob_xlib_pull` in that section is updated in the `dotfiles` phase.)

### Verification

`just all` (fmt check, clippy over all targets, `cargo test`). While iterating,
`cargo test highlights` is the fast subset.

---

## Phase `dotfiles`: Auto-scan from `bob_xlib_pull` and follow the rename in chezmoi

Open the chezmoi repo with the `/sase_repo` skill (`sase repo open chezmoi`) and use the
printed path for every chezmoi read and write. The bob-cli doc edit at the end of this
phase happens in this agent's own bob-cli checkout.

### chezmoi `home/dot_config/bob/config.yml`

Rename the key under `highlights:` from `pre_scan_command` to `pre_scan_hook`, keeping
the value `PATH="$HOME/bin:$PATH" bob_xlib_pull`.

### chezmoi `home/bin/executable_maybe_bob_highlights_sync`

This script decides whether to run a scan at all, and it reads the hook configuration
itself:

- rename `pre_scan_command_configured` to `pre_scan_hook_configured`;
- check `BOB_HIGHLIGHTS_PRE_SCAN_HOOK` instead of `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND`
  (same "set but empty means disabled" semantics);
- change the awk key pattern from `pre_scan_command:` to `pre_scan_hook:`.

Everything else (lock directory, bob lookup, output capture) stays as-is.

### chezmoi `home/bin/executable_bob_xlib_pull`

Add a scan step that runs after both `handle_host` calls and before the final
`exit "$result"`. Keep it POSIX `sh` and reuse the script's existing helpers:

- **Skip silently when invoked as the pre-scan hook**: `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK`
  non-empty means bob is about to scan anyway. Check this first.
- **Skip silently when the cron sync lock is held**: try
  `mkdir "${TMPDIR:-/tmp}/maybe_bob_highlights_sync.lock"`; on failure, another scan is
  already running. On success, remember that _this_ process created it so `cleanup()`
  removes it (and only then) alongside the existing `bob_xlib_pull.lock` teardown. Add a
  short comment in both scripts noting the shared lock name.
- **Resolve `bob` the same way `maybe_bob_highlights_sync` does**: `command -v bob`,
  else `$HOME/.cargo/bin/bob` if executable, else print
  `bob_xlib_pull: bob command not found: <path>` to stderr and set `result=1` without
  scanning (cron's minimal PATH is why the fallback exists).
- **Run the scan through the existing `run_waited` helper** so the signal-handling and
  child-tracking behavior already in the script applies:
  `run_waited "$bob_bin" highlights --no-hooks scan -w`. Let stdout/stderr inherit so
  cron captures them.
- **On non-zero status**, print `bob_xlib_pull: highlights scan failed (exit N).` to
  stderr and set `result=1`. A failed probe or transfer earlier in the run must not skip
  the scan; statuses combine into the single `result` the script already exits with.

### chezmoi `tests/bash/bob_xlib_pull_test.sh`

The suite stubs `uname`, `ssh`, and `rsync` on a fake `PATH`; `bob` must be stubbed the
same way or the real binary would be invoked by every existing test.

- Add `write_bob_stub` to `set_up` (alongside the other stub writers). The stub logs
  `bob-scan|<all args>` to `EVENT_LOG` and exits with `${BOB_SCAN_EXIT:-0}`.
- Thread `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK` and `BOB_SCAN_EXIT` through the `env` lists
  in `run_xlib_pull` and `start_xlib_pull`, defaulted in `set_up`.
- Confirm the existing tests still pass — in particular those asserting
  `assert_same "0" "${RUN_RC}"` and `assert_empty "$(stderr_text)"`.
- New tests:
  - a successful pull logs exactly one `bob-scan|highlights --no-hooks scan -w`;
  - an empty-queue run still scans;
  - `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1` produces no `bob-scan` event, with rc 0;
  - a pre-existing `maybe_bob_highlights_sync.lock` directory under `TMPDIR` suppresses
    the scan and is still present afterwards;
  - `BOB_SCAN_EXIT=7` yields rc 1 and a `highlights scan failed` diagnostic on stderr;
  - a non-Darwin `FAKE_UNAME` run logs no `bob-scan` event.

### Verification

In the chezmoi checkout: `just test-bash` (bashunit) plus `sh -n` on both edited
scripts. Follow the chezmoi repo's own agent instructions for applying the source tree
after its commit.

### Docs

Update the Highlights bridge section of bob-cli `docs/vault-git-sync.md`: after draining
athena and apollo, `bob_xlib_pull` now runs `bob highlights --no-hooks scan -w` itself
on the MacBook, so a manual pull syncs immediately; it skips that scan when bob invoked
it as the `pre_scan_hook` (bob is already scanning) and when the
`maybe_bob_highlights_sync` lock is held; the 15-minute cron job remains the periodic
fallback.
