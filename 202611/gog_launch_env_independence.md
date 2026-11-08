---
tier: tale
title: Make bob gcal and bob gmail independent of how bob was launched
goal:
  bob gcal sync and every bob gmail command work the same from Obsidian, Bob Mac
  Capture, launchd, SSH, and a terminal, and gog environment failures name their real
  cause.
size: medium
decisions:
  keyring_route:
    ask: How should a GUI-launched bob get gog's file-keyring password?
    choices:
      wrapper:
        New gcal.gog/gmail.gog keys point at the ~/bin/gog wrapper; bob never sees the
        secret
      command:
        New keyring_password_command keys; bob runs pass itself, like
        gkeep.token_command
    default: wrapper
    why: the wrapper already exists, agents use it, and the secret stays out of bob
    answer: wrapper
  decision_record:
    ask:
      Add a decisions record that bob runs gog against its own config root and a
      configured executable?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.cz
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 06:39:31
status: wip
---

# Make `bob gcal` / `bob gmail` independent of how bob was launched

## Problem

On Bryan's MacBook, after a reboot, the Obsidian command-mode sequence step
`bob gcal sync` fails with:

```
error: bob gcal sync: no bob-gcal OAuth client is stored
  hint: run `bob gcal login -c PATH` with a Desktop-app client JSON
```

The same command works in a terminal. Read-only checks over SSH showed the OAuth client
and tokens are intact. `zsh -lc 'bob gcal doctor'` passes the gog, client, account,
token, and calendars checks. Only the "last sync" row fails, and it records this error.

## Root cause (verified on the Mac)

Obsidian was relaunched at login, so it got launchd's minimal environment:
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`, no `XDG_CONFIG_HOME`, and no
`GOG_KEYRING_PASSWORD`. Bob Command Mode (bob-plugins
`plugins/bob-command-mode/src/030-bob-env.js`) prepends `~/.cargo/bin`, `~/.local/bin`,
`/opt/homebrew/bin`, and `/usr/local/bin` to `PATH`. It does not add `~/bin`, and it
does not set `XDG_CONFIG_HOME`. Bob then resolves the bare `/opt/homebrew/bin/gog`, and
three things are wrong at once:

1. **Wrong gog store.** gog (v0.47.1) picks its config dir from `XDG_CONFIG_HOME` and
   falls back to `~/Library/Application Support/gogcli` on macOS. Bryan's store is
   `~/.config/gogcli`. His login shell exports `XDG_CONFIG_HOME=$HOME/.config` from
   `~/.profile`. gog's verbatim stderr:
   `OAuth client credentials missing (OAuth client ID JSON). … (expected at /Users/bbugyi/Library/Application Support/gogcli/credentials-bob-gcal.json)`.
   bob classifies this as `no_client` and tells the user to re-run `login -c PATH`. That
   advice is wrong: following it would split the store.
2. **No keyring password.** Since Nov 7 the Mac's gog tokens live in gog's encrypted
   _file_ keyring (`keyring_backend: file`). With the right store but the bare gog, gog
   fails with
   `… read encoded file keyring item: no TTY available for keyring file backend password prompt; set GOG_KEYRING_PASSWORD`.
   bob maps that to the `keyring` kind with a misleading "could not save …" message.
3. **The wrapper is not robust either.** The chezmoi wrapper `~/bin/gog` supplies the
   password via `pass show gogcli/keyring-password`. Under launchd's bare `PATH` it
   prints `pass: command not found`, passes an _empty_ password, and gog fails with
   `… read encoded file keyring item: aes.KeyUnwrap(): integrity check failed.`

Only `~/bin/gog`, with `XDG_CONFIG_HOME=~/.config` and Homebrew on `PATH`, succeeded in
a scrubbed environment. That was verified with
`env -i HOME=… PATH=<command-mode PATH> XDG_CONFIG_HOME=~/.config ~/bin/gog … calendar calendars`.
"After reboot" matters because before it, Obsidian had been started from an environment
that already had those variables.

Every gog spawn goes through one choke point: `src/native/google/gog.rs`
`Gog::run_with_stdin` → `subprocess_runner::run`. `Gog::resolve` (18 callers under
`src/native/gcal/` and `src/native/gmail/`) picks the executable from `BOB_GCAL_GOG` /
`BOB_GMAIL_GOG`, else `gog` on `PATH`. The fix covers every bob caller: Obsidian command
mode, the Ledger Tools auto-sync, Bob Mac Capture's background Gmail kick, launchd, and
cron.

## Goal

`bob gcal …` and `bob gmail …` behave the same from a terminal, SSH, Obsidian, Bob Mac
Capture, and launchd. When gog fails for an environment reason, the error names the real
cause instead of sending the user to re-run login.

## Part A: bob-cli (this repository)

### A1. Pin gog's config root (unconditional)

In `Gog::run_with_stdin`, give every gog child an explicit `XDG_CONFIG_HOME` through
`RunSpec.env`. Use bob's own config-root rule:

- the bob process's non-empty `XDG_CONFIG_HOME` (tilde-expanded, matching
  `src/native/config/mod.rs::resolve_config_path`);
- otherwise `$HOME/.config`.

Put the root computation in one small helper in `src/native/google/`, read through
`bob_env` so tests can drive it. Doctor (A5) and the classifier hint (A4) reuse the same
helper.

Effects:

- On Linux this is a no-op.
- On macOS, gog always reads `~/.config/gogcli`, the same root that holds
  `bob/config.yml`, no matter who launched bob. Login and sync can never disagree about
  the store.
- A macOS store that only exists under Application Support is no longer read. Document
  that and the remedy, which is to rerun `bob gcal login` or move the `gogcli` dir.

`BOB_CONFIG_FILE` does not affect the root.

### A2. Configured gog executable

> [!decision] keyring_route = wrapper

- Add an optional string key `gog` to the `gcal:` and `gmail:` config blocks:
  `GcalSettings` / `GmailSettings` in `src/native/config/gcal.rs` and
  `src/native/config/gmail.rs`, surfaced through `src/native/gcal/config.rs` /
  `src/native/gmail/config.rs` as `gog()`.
  - Expand a leading `~/`. After expansion the value must be absolute.
  - A relative path, an empty string, or a non-string is a config error (exit 2). The
    message names the key and the config path, in the same style as the `timeout_secs`
    errors.
- Add `gog_config_key: "gcal.gog"` / `"gmail.gog"` to `GoogleService` in
  `src/native/google/profile.rs` for messages.
- Resolution order in `Gog::resolve`, which gains the configured path as an argument;
  update all 18 callers to pass `config.gog()`:
  1. `BOB_GCAL_GOG` / `BOB_GMAIL_GOG` (env stays the top override; tests rely on it);
  2. `gcal.gog` / `gmail.gog`;
  3. `gog` on `PATH`;
  4. well-known fallbacks, in order: `~/.local/bin/gog`, `/opt/homebrew/bin/gog`,
     `/usr/local/bin/gog`. This mirrors `bob_env::resolve_uv`, so a GUI-launched bob
     finds a Homebrew gog without PATH help.
- If the configured path is missing or not executable, return the existing `gog_missing`
  setup error with a hint to fix `<key>` in the config file.

> [!decision] keyring_route = command

- Instead of a `gog` key, add an optional string key `keyring_password_command` to
  `gcal:` and `gmail:`.
  - When it is set and bob's environment has no non-empty `GOG_KEYRING_PASSWORD`, bob
    runs it once per process with `sh -c`.
  - Mirror `GkeepConfig::read_token`: inherit stdin and stderr, take the first non-empty
    stdout line, and never echo the secret.
  - Run it with `PATH` extended by the existing Homebrew dirs when they are missing.
  - Pass the result to every gog child as `GOG_KEYRING_PASSWORD` in `RunSpec.env`, never
    in argv or logs.
  - A failing or empty command is a `keyring` setup error naming the key.
- Keep `BOB_*_GOG` → `PATH` resolution. Add the same well-known fallbacks
  (`~/.local/bin/gog`, `/opt/homebrew/bin/gog`, `/usr/local/bin/gog`).
- Use `keyring_password_command` instead of `gog` in A4, A6, and Part B's config.

### A3. Leave the child PATH alone

bob does not rewrite `PATH` for gog. The wrapper (Part B) owns finding `pass`/`gpg`.

### A4. Name the real cause in gog failure messages

In `classify_gog_failure` (`src/native/google/gog.rs`), keep every error **kind** and
exit code unchanged so no JSON contract changes. Refine the messages and hints:

- **Keyring branch** (`store secret` / `keyring`):
  - If the text mentions `GOG_KEYRING_PASSWORD` or `no TTY available for keyring`, use
    the message "gog's file keyring is locked: this run has no GOG_KEYRING_PASSWORD".
    - Hint for `wrapper`: point `<gog_config_key>` at a gog wrapper that exports it, for
      example `~/bin/gog`.
    - Hint for `command`: set `keyring_password_command`.
  - If the text mentions `integrity check failed`, use the message "gog could not unlock
    its file keyring (wrong or empty GOG_KEYRING_PASSWORD)". The hint says to check the
    password source.
  - Otherwise keep today's text, but say "read or save" instead of "save".
- **`no_client` branch:** when gog's text contains `expected at <path>`, put the store
  dir (the path's parent) in the message: "no bob-gcal OAuth client is stored in <dir>".
  Keep the hint.
- Gmail's `redact_gmail_setup_error` keeps the classifier hint, so the new hints flow
  through. Leave its fixed messages alone.
- Add unit tests built from the three verbatim gog strings quoted under Root cause,
  keeping the `/Users/bbugyi/...` path in the fixture. Assert kind, exit code, message,
  and hint.

### A5. Doctor

In `bob gcal doctor` and `bob gmail doctor`, the `gog` row summary becomes
`v<version> · <program> · store <root>/gogcli`:

- Strip a leading `v` from gog's reported version. Today it prints `vv0.47.1`.
- Tilde-abbreviate the home dir if a helper for that already exists.

The `client` row's "no … OAuth client is stored" summary names the same store dir.

### A6. Tests (integration)

- In `tests/gcal_support/fake_gog.py`, add an opt-in env-capture log, for example
  `BOB_GCAL_FAKE_ENV_LOG`. It records the `XDG_CONFIG_HOME` the fake received and
  whether `GOG_KEYRING_PASSWORD` was present (presence only, never the value).
- The harness clears env and sets `HOME` to a temp dir.
  - Assert that with no `XDG_CONFIG_HOME`, gog receives `<HOME>/.config`.
  - Assert that with `XDG_CONFIG_HOME=<dir>`, gog receives `<dir>`.
- `wrapper`:
  - With `BOB_GCAL_GOG` removed and `gcal.gog` set to the fake's path, commands use the
    fake; doctor's gog row shows that path.
  - A relative `gcal.gog` is a config error.
  - Add one equivalent `gmail.gog` test using the gmail harness in
    `tests/gmail_support/`.
- `command`:
  - With `keyring_password_command: printf 'pw\n'`, the fake sees the password present.
  - With an ambient `GOG_KEYRING_PASSWORD`, the command does not run.
  - A failing command is a `keyring` error.
- Add a unit test for the resolution order, including the fallbacks, using temp
  executables.

### A7. Docs and help

Read `sase memory read docs.md` first.

- **Help text:** update the `Environment:` lines for `BOB_GCAL_GOG` / `BOB_GMAIL_GOG` in
  `src/native/gcal/cli.rs` and `src/native/gmail/cli.rs` to "overrides gcal.gog
  (gmail.gog) and PATH" under `wrapper`. Then run `just docs-gen` so
  `docs/reference/cli/gcal.md` and `gmail.md` regenerate.
- **`docs/reference/configuration.md`:**
  - Add the new key rows.
  - Update the `BOB_*_GOG` env rows.
  - State the gog config-root pin next to the `gcal.timeout_secs` / `gmail.timeout_secs`
    blocks.
- **`docs/gcal.md` and `docs/gmail.md`:** add one short contract paragraph covering gog
  resolution order and the config-root pin.
- **`docs/guides/connect-google-calendar.md` and `docs/guides/connect-google-mail.md`,
  under "If it goes wrong":**
  - Add a row: "Works in a terminal, fails from Obsidian, Bob Mac Capture, or after a
    reboot". Answer: point `gcal.gog` at a gog wrapper that supplies
    `GOG_KEYRING_PASSWORD`, or set `keyring_password_command` under `command`.
  - Make the `no_client` row mention the store dir shown in the message.
- Run `just docs-check`.

### A8. Gates and commit rules

- Run `just fmt`, `just lint`, `just test`, `just docs-check`, and `just check-scripts`
  if any script changed.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`; release-plz owns both.
- Use a Conventional Commit subject without `!`, for example
  `fix(google): run gog with bob's config root and a configured executable`. New keys
  are additive and kinds and exit codes are unchanged.

## Part B: chezmoi linked repo (open with `/sase_repo`; follow its AGENTS.md)

### B1. Harden `home/bin/executable_gog`

The same file runs on Linux and macOS. Keep its real-gog selection (`~/.local/bin/gog`,
else `/opt/homebrew/bin/gog`) and add:

- **PATH:** append `/opt/homebrew/bin` and `/usr/local/bin` to `PATH` only when the dir
  exists and is not already present. This makes `pass` and `gpg` resolve from
  launchd/GUI environments.
- **Store:** `export XDG_CONFIG_HOME="${XDG_CONFIG_HOME:-$HOME/.config}"`, so direct
  wrapper calls agree with bob's pin.
- **Existing password:** if `GOG_KEYRING_PASSWORD` is already non-empty, `exec` gog
  untouched.
- **Password from pass:** otherwise read the first line of
  `pass show gogcli/keyring-password`. If it is non-empty, `exec` gog with it. If it is
  empty or pass failed, print one stderr line, for example
  `gog wrapper: could not read gogcli/keyring-password from pass; gog will ask for the keyring password`,
  and `exec` gog **without** setting the variable. Never pass an empty password: that
  produced the cryptic "integrity check failed".
- **stderr:** send errors there, for example the "gog executable not found" message.
- Run `bash -n` and shellcheck if available.

### B2. `home/dot_config/bob/config.yml`

> [!decision] keyring_route = wrapper

Add active blocks: `gcal:` with `gog: ~/bin/gog`, keeping the existing commented example
keys beneath it, and a new `gmail:` block with `gog: ~/bin/gog`. Each gets a one-line
comment saying why: GUI-launched bob has no login-shell PATH or keyring password.
Unknown keys are ignored by older bob builds (`unknown keys ignored` tests in both
config modules), so the order of landing does not matter.

> [!decision] keyring_route = command

Add `gcal:` and `gmail:` blocks with
`keyring_password_command: pass show gogcli/keyring-password`.

### B3. Apply

Follow the chezmoi repo's AGENTS.md apply rule after the commit.

## Part C: verify on the real Mac before finishing (best effort)

The Mac is SSH alias `mac` and is often offline. If it is unreachable, skip Part C and
say so in the final report. Work only under a scratch dir such as
`~/tmp/bob-gog-env-check` on the Mac. Do **not** touch the Mac's bob checkout
(`~/projects/github/bobs-org/bob-cli`), its installed `~/.cargo/bin/bob`, its
`~/bin/gog`, or its `~/.config/bob/config.yml`. Use read-only bob commands only.

1. **Build:** rsync this repo's working tree, excluding `target/` and `.git/`, into the
   scratch dir. Build `cargo build --bin bob` through `zsh -lc` with a generous
   foreground timeout.
2. **Stage:** copy the hardened wrapper to `<scratch>/gog` (mode 0755). Write
   `<scratch>/config.yml`: the Mac's current config plus the Part B2 blocks, with
   `~/bin/gog` replaced by `<scratch>/gog` under `wrapper`.
3. **Run the failing environment:** with both the launchd PATH
   (`/usr/bin:/bin:/usr/sbin:/sbin`) and the command-mode PATH (prepend
   `~/.cargo/bin:~/.local/bin:/opt/homebrew/bin:/usr/local/bin`), run
   `env -i HOME="$HOME" USER="$USER" PATH=<…> BOB_CONFIG_FILE=<scratch>/config.yml <scratch>/target/debug/bob …`:
   - `gcal doctor`: every row passes except possibly the stale "last sync" row, and the
     gog row shows `v0.47.1` (single `v`).
   - `gcal list`: read-only; it succeeds and lists today's events.
   - `gmail doctor`: passes.
4. **Sanity:** in the same scrubbed env, run the scratch bob with
   `BOB_GCAL_GOG=/opt/homebrew/bin/gog` and no wrapper. It should report the new "file
   keyring is locked … GOG_KEYRING_PASSWORD" message, not `no_client`.
5. **Clean up:** remove the scratch dir.

Never print tokens, the keyring password, or Bryan's email in commands or reports. Read
the account from gog or config on the Mac side if one is needed.

## Final report to Bryan

Say what changed in each repo and the Part C results. Then give these post-commit deploy
steps for the Mac:

1. `chezmoi update -a --force`
2. `cd ~/projects/github/bobs-org/bob-cli && git pull --ff-only && just install`
3. In Obsidian, press **Resume from 2** on the stopped sequence, or rerun it.

Until then, `bob gcal sync` works from a terminal.

## Memory

> [!decision] decision_record

Use `/sase_memory_write`. Add one decisions-web record,
`gog-runs-against-bob-config-root`, under the web's conventions:

- **Applies to:** bob-cli `bob gcal` and `bob gmail`.
- **Claim:** bob runs every gog child with `XDG_CONFIG_HOME` set to bob's config root,
  and with the executable chosen by `BOB_*_GOG`, then `gcal.gog` / `gmail.gog`, then
  `PATH`, then well-known fallbacks. Under `command`, it uses `keyring_password_command`
  instead.
- **Why:** cite this plan's verified Mac failure.
- **Cost:** a macOS store left under Application Support is not read.
- **Reopens when:** gog gains its own stable store discovery.

Run `sase memory init` afterwards.

> [!decision] decision_record = no

No memory change.

## Out of scope

- Making bob-plugins import the login-shell environment. bob-cli is the single choke
  point and also covers Bob Mac Capture and launchd.
- Changing gog itself.
