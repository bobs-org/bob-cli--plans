---
tier: tale
title: Install and authorize gog on apollo for read-only Gmail
goal:
  gog and the sase_gmail skill can read Bryan's Gmail (read-only) on apollo, with the
  gog binary install managed so it cannot silently go missing again.
size: small
decisions:
  install:
    ask: How should the gog binary be installed?
    choices:
      pinned:
        A chezmoi installer pinned to v0.44.0 with an embedded sha256; upgrades are
        deliberate commits
      latest:
        Chezmoi installer tracks the newest GitHub release weekly, like lazygit;
        machines auto-upgrade
      manual:
        A one-off install on apollo only; no chezmoi change, so the next fresh machine
        repeats this breakage
    default: pinned
    why:
      gog holds Gmail credentials and sase_gmail relies on its safety flags; upgrades
      deserve review
    answer: pinned
  auth:
    ask: How should apollo get its Gmail authorization?
    choices:
      copy:
        Copy athena's encrypted gog state over SSH; no browser, but both machines share
        one refresh token
      fresh:
        Copy only the OAuth client; Bryan approves a new read-only consent and pastes
        back one redirect URL
    default: copy
    why:
      athena's read-only token is healthy and copying it needs no interactive OAuth
      round trip
    answer: copy
  athena:
    ask: Should the installer also replace athena's May source build now?
    choices:
      upgrade:
        Run the installer on athena now and verify it with auth doctor; both machines
        run one release
      keep:
        The installer skips source builds (git-describe versions), so athena stays on
        v0.16.0-18
    default: upgrade
    why:
      one verified release beats silent drift, and v0.44.0 reads athena's existing state
      unchanged
    answer: upgrade
proposed_by: bbugyi200.apollo.7n
decided_by: auto
create_time: 2026-10-14 11:22:13
status: wip
---

# Plan: Install and authorize `gog` on apollo for read-only Gmail

Make `gog` work on apollo so that `gog` and the `sase_gmail` skill can read Bryan's
Gmail (bryanbugyi34@gmail.com, read-only), the way they already do on athena. The fix
lives in the **chezmoi** linked repo and on the machines themselves. No bob-cli files
change.

## Findings (state when this plan was written)

### apollo (the target)

- `gog` resolves to `~/bin/gog`. That file is the chezmoi-managed wrapper
  `home/bin/executable_gog`, which runs
  `GOG_KEYRING_PASSWORD="$(pass show gogcli/keyring-password | sed -n '1p')" ~/.local/bin/gog "$@"`.
- **`~/.local/bin/gog` does not exist**, so every `gog` call fails with
  `~/.local/bin/gog: No such file or directory`. chezmoi ships the wrapper to every
  machine, but nothing installs the binary. That gap is the root cause.
- There is no `~/.config/gogcli` and no `~/.local/share/gogcli`, so apollo has no gog
  state at all.
- `pass show gogcli/keyring-password` works non-interactively from an agent shell (the
  password store is synced). `jq`, `curl`, `tar`, and `sha256sum` are available. The
  system Go is 1.22.2 with `GOTOOLCHAIN=auto`.
- **PATH pitfall:** `~/go/bin` comes _before_ `~/bin` on PATH, and `~/.local/bin` comes
  _after_ it. A `go install` into `~/go/bin` would shadow the wrapper and drop
  `GOG_KEYRING_PASSWORD`. The real binary must live at `~/.local/bin/gog` and never in
  `~/go/bin`.

### athena (the working reference)

- `~/.local/bin/gog` is a May 2026 source build of upstream main: `v0.16.0-18-g9ebdc01`,
  from a clean checkout at `~/projects/github/openclaw/gogcli`.
- Its state lives in `~/.config/gogcli/` (directories 0700, files 0600):
  - `config.json` contains `{"keyring_backend": "file"}`.
  - `credentials.json` holds the OAuth desktop client's `client_id` and `client_secret`.
  - `keyring/` holds three encrypted file-keyring entries. They are encrypted with the
    same `pass gogcli/keyring-password` that apollo can already read.
- Account: bryanbugyi34@gmail.com, client `default`, services `gmail`, scope
  `gmail.readonly`, created 2026-05-14.
- `gog --no-input --json auth doctor --check` reports `status: ok`, and the
  `refresh.default.bryanbugyi34@gmail.com` check passes. So the OAuth app is In
  production, not in 7-day Testing.

### Upstream gogcli

- The latest release is **v0.44.0** (2026-10-11). Its assets include
  `gogcli_0.44.0_linux_amd64.tar.gz` and `checksums.txt`.
- The Go module moved to `github.com/openclaw/gogcli`.
- Source checks confirm v0.44.0 is compatible with athena's state:
  - The keyring key format is unchanged: `token:<client>:<email>` plus the legacy
    `token:<email>`.
  - Credentials and the keyring dir in the legacy `~/.config/gogcli` location are still
    read (`docs/paths.md`). New writes go to the XDG data dir.
  - `gog --json auth status` still exposes `.account.email`.
  - Every command and flag the `sase_gmail` skill uses still exists: `gmail search`,
    `gmail get`, `gmail thread get`, `gmail thread attachments`, `gmail attachment`,
    `--gmail-no-send`, `--wrap-untrusted`, `--enable-commands`, and
    `--sanitize-content`.
- `auth doctor` exits 0 even when it reports failed checks. Gate on the JSON instead:
  `jq -se 'length == 1 and .[0].status == "ok"'`.

## Step 1: Add the chezmoi installer and install on apollo

> [!decision] install = manual Skip the chezmoi file. Instead, do the same download,
> sha256 check (against the release's `checksums.txt`), and `install -D -m 755` into
> `~/.local/bin/gog` by hand on apollo only. Then continue with Step 2. Skip Step 3,
> because athena is untouched.

1. Open the repo with `sase repo open chezmoi -r "<reason>"` and read its `AGENTS.md`.
   Use the printed path for all edits.
2. Add `home/.chezmoiscripts/run_onchange_install_gog.tmpl`. Model it on
   `home/.chezmoiscripts/run_onchange_install_lazygit.tmpl` and keep the same idiom:
   - Wrap the script in a `{{- if ne .chezmoi.os "darwin" -}}` … `{{- end -}}` guard.
   - Start with a `#!/bin/bash` shebang.
   - Add a header comment carrying a weekly `WEEK#{{ output "date" "+%V" | trim }}`
     marker. The rendered script then changes weekly, so chezmoi re-checks each week.
     This makes it self-heal a deleted binary, which is exactly apollo's current
     failure.
   - Source `~/.local/share/chezmoi/lib/chezmoi_utils.sh`.
   - Use `chez::log` for messages and `chez::build_dir_root` for the build dir.
   - End with `chez::exit_install "$?"` and `# vim: ft=bash`.
3. The script's behaviour:
   - Read the current version from the raw binary, not the wrapper. For example, run
     `"$HOME"/.local/bin/gog --version` and normalize it by taking the first field and
     stripping a leading `v`. Treat a missing binary as `NONE`. Confirm the exact
     release output format against the downloaded binary.
   - If the current version equals the target version, log "already installed" and
     succeed **without network access**.
   - Otherwise:
     1. Download the tarball into a freshly emptied `$(chez::build_dir_root)/gog`.
     2. Verify its sha256.
     3. Extract only the `gog` member. Check the archive layout with `tar tzf` first.
     4. Install it with `install -D -m 755 gog "$HOME"/.local/bin/gog`.
     5. Re-read the installed binary's version and fail if it does not match.
   - Support x86_64 only (`linux_amd64`), like lazygit. On any other `uname -m`, log and
     return 1.
   - If `$HOME/go/bin/gog` exists, log a warning that it shadows the `~/bin/gog` wrapper
     and drops the keyring password. This mirrors lazygit's shadowed-binary log.
4. Version source, by decision:
   - `install = pinned`: hard-code `GOG_VERSION=0.44.0` and
     `GOG_SHA256=<sha256 of gogcli_0.44.0_linux_amd64.tar.gz>`. Copy the hash from the
     v0.44.0 release's `checksums.txt` while implementing, then check the downloaded
     tarball against the embedded hash with `sha256sum -c`.
   - `install = latest`: resolve the newest tag from the GitHub releases API the way
     `latest_lazygit_version` does. Download that release's `checksums.txt` and verify
     the tarball against its line.
5. Source-build handling, by decision:
   - `athena = upgrade`: any version that differs from the target is replaced.
   - `athena = keep`: if the current version looks like a git-describe source build
     (`-g[0-9a-f]+`), log "source build detected; skipping" and succeed.
6. Check the rendered script: run `bash -n` on it, and run `shellcheck` if it is
   installed.
7. **Install on apollo now.** The commit only lands at turn finalization, so render and
   run the script directly:
   - Render it with
     `chezmoi execute-template < <repo>/home/.chezmoiscripts/run_onchange_install_gog.tmpl`
     into a scratch file, then run that file with `bash`.
   - **Caution:** without a TTY, `chez::exit_install` turns a failure into exit 0.
     Always confirm the result independently with `~/.local/bin/gog --version`.
   - Run the script a second time to show it short-circuits as already installed.

## Step 2: Authorize apollo

Look before writing. If `~/.config/gogcli` or `~/.local/share/gogcli` exists on apollo
by now, stop: list the file names only (never their contents) and report back instead of
overwriting. Never print tokens, `credentials.json`, keyring entries, or the keyring
password. Do not modify anything on athena in this step.

**`auth = copy` (default):** stream athena's state across the tailnet, preserving
permissions:

```bash
ssh athena 'tar -C ~/.config -cf - gogcli' | tar -C ~/.config -xpf -
```

Then make sure `~/.config/gogcli` and `~/.config/gogcli/keyring` are mode 0700 and every
file is 0600. apollo already has the same keyring password via `pass`, so the
file-keyring entries open as-is.

> [!decision] auth = fresh
>
> 1. Copy only `gogcli/config.json` and `gogcli/credentials.json` from athena, using the
>    same `tar` stream with explicit members. Apply the same permissions.
> 2. Run
>    `gog auth add bryanbugyi34@gmail.com --services gmail --readonly --gmail-scope readonly --remote --step 1`
>    and capture the consent URL it prints.
> 3. Use `/sase_questions` to give Bryan that URL. Ask him to approve it and paste back
>    the full redirect URL from the address bar. The localhost page failing to load is
>    expected. Warn that the code expires within minutes.
> 4. Finish with the same flags plus `--remote --step 2 --auth-url '<pasted url>'`, from
>    the same machine and gog home (the flow uses PKCE). If the exchange fails as
>    expired, restart at step 1.
> 5. Never write the redirect URL to files, commits, or logs. Afterwards apollo has its
>    own revocable refresh token.

## Step 3: Roll out to athena (only when `athena = upgrade` and `install` is not `manual`)

1. Pipe the rendered script to athena: `ssh athena bash -s < <rendered script>`. The
   script sources athena's own `~/.local/share/chezmoi/lib/chezmoi_utils.sh`.
2. Verify it independently:
   - `ssh athena '~/.local/bin/gog --version'` reports the target release.
   - `ssh athena 'PATH=$HOME/bin:$PATH gog --no-input --json auth doctor --check' | jq -se 'length == 1 and .[0].status == "ok"'`
     exits 0.
3. Rollback if the doctor check fails on athena: rebuild the old binary from athena's
   clean checkout at `~/projects/github/openclaw/gogcli` (commit `9ebdc01`, `make`),
   copy `bin/gog` to `~/.local/bin/gog`, confirm the doctor check passes again, and
   report what failed.
4. Athena's next `chezmoi update` re-runs the committed script, which is a no-op once
   the version matches.

## Step 4: Verify apollo end to end (all branches)

Run these checks both in the agent shell and through `zsh -lc '…'`. The login shell
proves interactive PATH order too.

1. `command -v gog` prints `/home/bryan/bin/gog`, the wrapper, not `~/go/bin/gog`.
2. `gog --version` reports the installed release.
3. `gog --no-input --json auth doctor --check | jq -se 'length == 1 and .[0].status == "ok"'`
   exits 0. If it reports `keyring.open` or `keyring.password` failures, compare the
   `pass` entries on the two machines without printing them, and report back. Do not
   re-authenticate unless `auth = fresh` was chosen.
4. `gog --json auth list` shows bryanbugyi34@gmail.com, client `default`, and
   `gmail.readonly` as its only Gmail scope.
5. Run a `sase_gmail` skill smoke test, following the skill exactly:
   1. Resolve the account with `gog --json auth status | jq -r '.account.email'`.
   2. Build the documented `GOG_GMAIL_FLAGS` array.
   3. Run `gmail search "in:inbox newer_than:7d" --max 3` and print only the result
      count and message IDs.
   4. Run `gmail get <first id> --sanitize-content` and report only that it succeeded.
   - Keep subjects, senders, and bodies out of the transcript.
6. Negative check: with the same flags, a command outside the `--enable-commands` list
   (for example `gmail labels list`) is rejected. This shows the skill's restriction
   still holds on this release.

## Step 5: Land and hand back

- The chezmoi linked repo is this turn's only dirty repo. Give it a `commit` decision in
  `/sase_final` with a message like
  `feat(gog): install pinned gogcli release on Linux machines`. Adjust the message to
  match the chosen `install` decision.
- The chezmoi `AGENTS.md` requires `chezmoi update -a --force` after commits. The
  finalizer commits after the turn's last action, so tell Bryan to run it on apollo (and
  on athena under `athena = upgrade`) once the commit lands. It re-runs the script as a
  no-op re-check.
- The final report should cover:
  - the gog version on each machine
  - the doctor result on each machine
  - the outcome of the skill smoke test
  - that the refresh token is shared with athena (`auth = copy`) or is separate
    (`auth = fresh`)

## Out of scope

- **macOS (`mac`).** chezmoi also deploys `~/bin/gog` there, but this installer is
  Linux-only, like lazygit. A Homebrew `openclaw/tap/gogcli` install would be separate
  work; mention it in the report as a possible follow-up.
- Changing gog scopes, the OAuth client, or the `sase_gmail` skill text.
- Deleting athena's gogcli source checkout.

## Acceptance criteria

- On apollo, a plain `gog` resolves to the `~/bin/gog` wrapper and runs an installed
  release from `~/.local/bin/gog`.
- On apollo, `auth doctor --check` passes the JSON predicate.
- On apollo, the `sase_gmail` skill's search and get commands succeed with only the
  `gmail.readonly` scope.
- Unless `install = manual`, the committed chezmoi installer is idempotent: a rerun
  makes no network calls once the version matches.
- Under `athena = upgrade`, athena runs the same release and its doctor check passes.
  Under `athena = keep`, athena's binary is unchanged.
