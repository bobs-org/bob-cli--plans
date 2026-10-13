---
tier: tale
title: Stop the macOS bob background-activity alert
goal: The Mac vault-sync LaunchAgent starts bob through Apple-signed /bin/sh, vault-sync
  still runs, and chezmoi source matches that plist so later installs do not register
  a new Background Task Management item.
size: small
decisions:
  cdhash_probe:
    ask: Re-sign the Mac's bob ad-hoc to prove a new cdhash does not raise another
      alert?
    choices:
      probe: Re-sign bob ad-hoc and require that no new bob advisory appears
      skip: Confirm program=/bin/sh and a successful sync, and leave bob's signature
        alone
    default: probe
    why: The wrapper is untested until a new bob cdhash stays quiet
    answer: probe
proposed_by: bbugyi200.apollo.7a.f0
decided_by: auto
status: done
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                           | Why                                                 |
| ------------ | ------------------------------------------------------------------ | --------------------------------------------------- |
| derives-from | [research:202610/macos_bob_background_activity_notification.md][1] | Confirmed BTM cause and the /bin/sh LaunchAgent fix |

[1]:
  https://github.com/bobs-org/bob-cli--research/blob/main/202610/macos_bob_background_activity_notification.md

<!-- sase:links:end -->

# Stop the macOS "bob can run in the background" alert

One tale, not an epic. The work is a single plist edit, one sentence in the operator
doc, and a reload of the already-identified LaunchAgent. A second phase would only exist
to wait for this turn's commit, and the reload does not need that commit.

## Cause

`com.bbugyi.bob-vault-sync` runs `~/.cargo/bin/bob` directly. `cargo install` writes a
new ad-hoc signature with no Team ID, and macOS Background Task Management identifies
that binary by its cdhash. Every `just install-all` that replaces `bob` looks like a new
background item. The alert style stays until the X is clicked. The logs tied the alert
to this agent, not to cron. Evidence is in `file:explicit:c40fb32ee051403a3c2833b8` (the
research report `research:202610/macos_bob_background_activity_notification.md`).

The tracked executable is `ProgramArguments`' first string, the program launchd is told
to start. Pointing that at `/bin/sh` makes BTM follow Apple's stable signature. `exec`
replaces the shell with `bob`, so launchd still supervises one process and the exit
status and log files stay bob's.

## Out of scope

- Do not turn the "bob" switch off in Login Items & Extensions. That stops vault sync.
- Do not run `sfltool resetbtm`.
- Do not sign `bob` with a development certificate. This Mac has no signing identity.
  That remains the fallback only if the probe below fails.
- Do not change the Linux `bob-vault-sync.service` or
  `home/bin/executable_bob_vault_sync_watch`. Those use inotify and do not cause this
  alert.
- Do not change the Mac crontab lines that call `~/.cargo/bin/bob` (`projects sync`,
  `task reconcile`, highlights). Cron is not the item the BTM logs named, and that
  crontab is not chezmoi-managed.
- Do not templatize `/Users/bbugyi`, and do not change `WorkingDirectory`,
  `StartInterval`, `RunAtLoad`, `EnvironmentVariables`, or the log paths.
- Do not edit `home/sase/memory/obsidian.md`. It still correctly says the MacBook uses
  this LaunchAgent. It does not claim the executable path.

## 1. Edit the chezmoi plist

Open the linked repo with `sase repo open chezmoi` and read its `AGENTS.md` before
editing. Change only the checkout that command prints. Do not also edit
`~/.local/share/chezmoi`; that is a different clone, and chezmoi's after-commit hook
updates it from origin.

In `home/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist`, replace the
`ProgramArguments` array with:

```xml
<key>ProgramArguments</key>
<array>
  <string>/bin/sh</string>
  <string>-c</string>
  <string>exec /Users/bbugyi/.cargo/bin/bob vault-sync -q</string>
</array>
```

Leave the rest of the plist as it is. The file is not a chezmoi template. The absolute
path matches the existing `/Users/bbugyi` entries. Do not use `~`; launchd would not
expand it, and the `exec` text is passed through literally.

On the dev host, parse the file before copying it anywhere:

```bash
python3 -c 'import plistlib,pathlib,sys; p=plistlib.loads(pathlib.Path(sys.argv[1]).read_bytes()); a=p["ProgramArguments"]; assert a==["/bin/sh","-c","exec /Users/bbugyi/.cargo/bin/bob vault-sync -q"], a'
```

Pass the edited plist path as the argument. Do not run chezmoi `just check`,
`just test`, or `just fmt-md`. Those are unrelated to this XML file and `fmt-md`
rewrites every Markdown file.

Do not commit by hand and do not invoke `/sase_git_commit`. The host commits this repo
when the turn ends. `sase/sase.yml` in that repo runs `chezmoi update -a --force` after
the commit, which pulls origin into `~/.local/share/chezmoi` on the committing host.
That hook does not reach the Mac. Do not run `chezmoi update` or `chezmoi apply`
yourself on either machine during this turn.

## 2. Record why in the operator doc

In the bob-cli checkout, edit `docs/vault-git-sync.md`. After the sentence "The
LaunchAgent runs `bob vault-sync -q` every 15 seconds and at load.", add this paragraph,
wrapped like the surrounding lines:

```markdown
launchd starts that command through `/bin/sh -c`, which `exec`s
`/Users/bbugyi/.cargo/bin/bob vault-sync -q`. Background Task Management then tracks
Apple-signed `/bin/sh` instead of the ad-hoc `bob` binary that `cargo install` replaces.
`launchctl print` shows `program = /bin/sh`.
```

Do not run bob-cli `just check`. This is a doc sentence, not a behavior change in bob.
Do not reformat the rest of the file.

## 3. Reload the live agent on the Mac

The Mac SSH alias is `mac` (user `bbugyi`). It is offline when the lid is closed. Try
`ssh mac 'id -u'` once. If it does not connect, stop this section, leave the source and
doc edits in place, and report that the agent was not reloaded. Include the commands
below so they can be run when the Mac is awake. Do not claim the notification has
stopped. Do not retry SSH in a loop.

The Mac's chezmoi source will not contain this edit until the host commit is on
`origin/master` of `git@github.com:bbugyi200/dotfiles.git`. `chezmoi apply` and
`chezmoi update` on the Mac read that source, so either command during this turn would
restore the old plist and undo the reload. Copy the edited file directly, and do not
modify the Mac's `~/.local/share/chezmoi`.

If SSH works:

1. Save the live plist:
   `ssh mac 'cp ~/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist /tmp/com.bbugyi.bob-vault-sync.plist.bak'`
2. Copy the edited plist to `mac:Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist`
   (that path is relative to the remote home). Do not use a local `~` in the remote
   path.
3. `ssh mac 'plutil -lint ~/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist'`
4. Reload, using the docs' commands. `$(id -u)` must run on the Mac:
   ```bash
   ssh mac 'launchctl bootout gui/$(id -u)/com.bbugyi.bob-vault-sync'
   ssh mac 'launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist'
   ```
   If bootout fails because the agent is not loaded, continue to bootstrap. If bootstrap
   fails, copy the backup back over the live plist, bootstrap that, and stop. Vault sync
   must not be left unloaded.
5. Confirm `ssh mac 'launchctl print gui/$(id -u)/com.bbugyi.bob-vault-sync'` prints
   `program = /bin/sh`.
6. Wait about 20 seconds, then confirm one successful run. `last exit code` in the
   `launchctl print` output is `0`, and
   `ssh mac 'tail -n 40 /var/tmp/com.bbugyi.bob-vault-sync.err'` shows no new traceback
   from this reload. Also run
   `ssh mac '/Users/bbugyi/.cargo/bin/bob vault-sync status --json'` and confirm it
   reports a recent successful sync. If the job fails, restore the backup plist,
   bootout, bootstrap, confirm `program` is `bob` again, and revert both the chezmoi
   plist and the doc paragraph. Report the stderr. Do not keep a wrapper that breaks
   sync.

Expect one new "can run in the background" alert at reload time, likely naming `sh`.
That alert is the predicted last one. Do not try to dismiss or suppress it.

## 4. Prove a new bob hash stays quiet

Use the accepted `cdhash_probe` answer on this plan. The default is `probe`.

> [!decision] cdhash_probe = probe After the reload sync has succeeded, wait a few
> seconds so the reload advisory is already in the past. Record the Mac's clock with
> `ssh mac 'date "+%Y-%m-%d %H:%M:%S"'`. Record the current cdhash with
> `ssh mac 'codesign -dv /Users/bbugyi/.cargo/bin/bob'`. Then run
> `ssh mac 'codesign --force --sign - /Users/bbugyi/.cargo/bin/bob'`. This only replaces
> the throwaway ad-hoc signature `cargo install` already writes. It does not need a
> certificate. If codesign fails because the binary is busy, wait for the current
> 15-second run to exit and try once more. If it still fails, report the error and
> follow the `skip` checks instead of inventing a certificate.
>
> Wait 25 seconds so the next interval runs the re-signed binary. Then:
>
> ```bash
> ssh mac "/usr/bin/log show --start 'TIMESTAMP' --style compact --predicate 'process == \"BackgroundTaskManagementAgent\" && eventMessage CONTAINS \"Posting advisory\"'"
> ```
>
> Replace `TIMESTAMP` with the clock reading from before `codesign`. A new advisory
> whose message names bob means the wrapper did not hide the rebuilt binary. Restore
> `/tmp/com.bbugyi.bob-vault-sync.plist.bak`, reload it, confirm `program` is the bob
> path again, and revert the chezmoi plist and the doc paragraph. Report the log lines.
> Do not implement certificate signing.
>
> No advisory in that window, or an advisory that names only `sh` and not bob, means the
> probe passed. Leave the wrapper in place. `codesign` does not need to be undone; the
> next `cargo install` replaces `bob` anyway.

> [!decision] cdhash_probe = skip Do not run `codesign`. The reload checks in section 3
> are the whole Mac verification. Say in the turn summary that a rebuilt `bob` was not
> tested, so a later `just install-all` is what shows whether the alert is gone.

## 5. Leave the Mac source alone

Do not run `chezmoi update` or `chezmoi apply` on the Mac in this turn. After the host
commit reaches `origin/master`, a later `ssh mac 'chezmoi update -a --force'` writes the
same plist the reload already installed. Until that push exists, an update on the Mac
would put the old `ProgramArguments` back. There is no scheduled chezmoi update on the
Mac, so the live file staying ahead of the Mac's chezmoi clone for the rest of this turn
is intentional.

## Done

The turn is done when the chezmoi plist and the doc paragraph match section 1 and
section 2, the plist parses, and either the Mac shows `program = /bin/sh` with a
successful vault-sync after reload (plus the accepted probe result) or SSH to `mac`
failed and that fact is reported with the reload commands still accurate. Do not claim
the alert is gone when the Mac was offline or the probe restored the old plist.
