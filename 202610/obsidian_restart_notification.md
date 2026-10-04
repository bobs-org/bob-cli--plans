---
tier: tale
title: Notify on the Mac when install-all needs to restart Obsidian
goal: just install-all posts one clear Notification Center banner on macOS when a
  plugin change means a running Obsidian must restart, and a second banner when that
  restart cannot finish. A stopped Obsidian, an unchanged vault, and Linux stay quiet.
size: small
proposed_by: bbugyi200.athena.0wh.f0
status: done
---

# Plan: Mac notification when `just install-all` needs to restart Obsidian

## Goal

When `scripts/install_all` is about to restart a running Obsidian, or when that restart
is owed but cannot finish, post a Notification Center banner on macOS. The banner says
what is happening and plays the default alert sound. It is best-effort: a missing
permission or a failed `osascript` never changes the restart, the marker, or the exit
status.

This builds on the restart behavior landed by
`plan:202610/install_all_restart_on_plugin_change.md` (commit `54ff70a`). Do not
re-implement that behavior. If `obsidian_step` in `scripts/install_all` does not already
restart only when `PLUGINS_CHANGED` is set or the pending marker exists, stop and report
that instead of inventing a second restart policy.

## Design call-outs

1. **Use `osascript` `display notification`, the mechanism the script already uses to
   quit Obsidian.** No new dependency and no new signed app.
   - Title, subtitle, body, and `sound name "default"`. An unrecognized sound name plays
     the system default alert, which is the sound we want.
   - Keep the `osascript` process alive with `delay 0.5` in the same script. On current
     macOS, an `osascript` that exits immediately can drop the banner before
     Notification Center accepts it.
   - Pass the title and body as `osascript` argv (`osascript - "$title" "$body"` with a
     quoted heredoc and `on run argv`). The copy is fixed, and argv keeps it out of the
     shell quoting.
   - macOS files this banner under Script Editor. `display notification` cannot set a
     custom icon or attach a button. The subtitle `bob install-all` is what makes the
     banner identifiable.
   - Do not request notification permission. A prompt mid-install is worse than a missed
     banner. If Script Editor notifications are off, or Focus hides them, the restart
     still happens and the terminal row is the record.

   Rejected alternatives:
   - **`terminal-notifier` or `alerter`.** A Homebrew tool with its own permission
     identity. The good path would be untested unless it happened to be installed.
   - **Asking Bob Mac Capture to post it.** That app's `UNUserNotificationCenter`
     banners are the house style (title, body, default sound, silent when unauthorized,
     never a permission prompt). It is the wrong sender here: `just install` leaves a
     stopped Bob Mac Capture stopped, and a missing sibling checkout skips it. Launching
     it only to notify would violate that rule. A new IPC surface in another repo is out
     of proportion to one banner.
   - **A new signed notifier applet in bob-cli.** Permission is tied to the signing
     identity, so this adds an install step and a second app for one sentence of copy.

2. **Notify only when a restart is actually owed, and say which kind.**
   - **About to quit a running Obsidian** (no earlier step failed): one banner before
     the quit, so the disappearance is explained before the Automation prompt or the
     quit. Title `Restarting Obsidian`.
   - **A step already failed:** one banner, no quit. Title `Obsidian needs a restart`.
     The pending marker stays, as it does today.
   - **The quit or relaunch then fails:** a second banner, because the first one
     promised a reopen that did not happen. Same failure title.
   - **Not running, plugins unchanged, could not check whether it is running, relaunch
     not confirmed yet, Linux, any other OS:** no banner. A stopped Obsidian loads the
     plugins on next launch. An unknown running-state is not a claim that a restart is
     needed. The unconfirmed relaunch already had the "will reopen" banner. Linux has no
     Notification Center; do not add `notify-send`.

3. **The banner never owns the outcome.** `notify_mac` ends with `|| true`. It does not
   increment `FAILURES` or `WARNINGS`, and it does not write or clear the pending
   marker. Print the banner text through `command_line` before the call so the
   transcript shows it even when `osascript` fails.

## Part 1 — `scripts/install_all`

Add `notify_mac` next to the other helpers. It takes a title and a body, both fixed
literals from the call sites below. Comment, in two lines, why the `delay` is inside
this `osascript` and why a failure is ignored.

```bash
notify_mac() {
  local title="$1" body="$2"
  command_line "osascript display notification \"$body\" with title \"$title\" subtitle \"bob install-all\" sound name \"default\""
  osascript - "$title" "$body" <<'APPLESCRIPT' || true
on run argv
  display notification (item 2 of argv) with title (item 1 of argv) subtitle "bob install-all" sound name "default"
  delay 0.5
end run
APPLESCRIPT
}
```

The displayed command line is the readable form, not a dump of the heredoc. Stay on bash
3.2. Keep mode `100755`.

### Call sites in `obsidian_step`

Darwin only. Guard with `[[ "$(uname -s)" == Darwin ]]`. Do not call `notify_mac` from
the Linux branch.

**Step already failed** (the existing `FAILURES > 0` return), before the glyph line:

- `PLUGINS_CHANGED == 1`: title `Obsidian needs a restart`, body
  `Plugins changed, but a step failed. Fix it and rerun just install-all.`
- Marker only: title `Obsidian needs a restart`, body
  `An earlier plugin update still needs a restart, but a step failed. Fix it and rerun just install-all.`

**Running, about to quit** (after `running` is `true`, before the quit `command_line`):

- `PLUGINS_CHANGED == 1`: title `Restarting Obsidian`, body
  `Plugins changed. It will reopen in a moment.`
- Marker only: title `Restarting Obsidian`, body
  `Finishing an earlier plugin update. It will reopen in a moment.`

**After that promise breaks,** post `Obsidian needs a restart` with:

- Quit `osascript` failed:
  `Quit failed. Allow Terminal in Automation, then rerun just install-all.`
- Still running after 30s:
  `Obsidian did not quit. Close any open dialog, then restart it to load the new plugins.`
- `open -b md.obsidian` failed:
  `Obsidian quit but did not reopen. Open it to load the new plugins.`

No banner on: not running; `is running` / quit-check `osascript` itself failing;
`relaunch requested; not confirmed yet`; Linux manual hint; Linux CLI restart success or
failure; unsupported OS; plugins unchanged.

### Usage text

In the `--help` prose, add one sentence after the existing restart sentence:
`On macOS it posts a Notification Center banner before that restart.` Leave the header
`then` line as it is.

## Part 2 — docs

- **`README.md`, the install-all paragraph** (the one that starts "After a clean run").
  Add: before the macOS quit, `just install-all` posts a Notification Center banner
  titled `Restarting Obsidian` with subtitle `bob install-all`. The banner titled
  `Obsidian needs a restart` is posted instead when a failed step or a failed
  quit/relaunch leaves the new plugins unloaded. macOS files the banner under Script
  Editor. The script never asks for notification permission, and a denied or dropped
  banner does not change the restart.
- **`docs/getting-started.md`:** extend the install-all sentence so it also says the
  macOS restart posts a notification first.
- **`docs/plugins.md`:** extend the install-all sentence that mentions
  `--dry-run --format json` with "and, on macOS, posts a notification before restarting
  Obsidian."

Do not edit archived plans under `sase/` or `~/.sase/plans`. Do not change the justfile.
Do not change Bob Mac Capture.

## Verification

Do not run the real `just install-all` or `scripts/install_all` against the real home.
It would replace the installed `bob`, write to `~/bob`, and could restart Bryan's
Obsidian.

1. `just check-scripts` and `shellcheck scripts/install_all` are clean. Do not run
   `just all`; this change has no Rust.
2. `scripts/install_all --help` includes the new banner sentence.

Reuse the sandboxed recipe in `plan:202610/install_all_restart_on_plugin_change.md`
(temp `HOME`, `CARGO_HOME`/`RUSTUP_HOME` captured first, fake vault, sibling
bob-plugins, stub `PATH`). Do not create a bob-mac-capture checkout, so the Darwin
`mac_capture_step` skips. Add these stubs and point `PATH` at them first:

- `uname` prints `Darwin` for the Darwin scenarios and is absent for the Linux control.
- `osascript` appends its argv and any stdin to `$T/osascript.log`. If the script text
  contains `display notification`, exit with a configurable status (default 0) and do
  not sleep. If the `-e` argument contains `is running`, print `true` or `false` from a
  state file. If it contains `to quit`, set that state and exit with a configurable
  status. Do not invoke the real `/usr/bin/osascript`.
- `open` appends its argv to `$T/open.log`, sets the running state, and exits with a
  configurable status.
- `pgrep` exits 0 while the state file says running, else 1.

Because `HOME`, `uname`, `osascript`, `open`, and `pgrep` are sandboxed, the real Mac
and the real Obsidian are never touched. This host is Linux; the Darwin branch runs here
only through those stubs.

Scenarios:

1. **Fresh vault, Obsidian "running".** One `display notification` whose stdin or argv
   carries title `Restarting Obsidian` and body
   `Plugins changed. It will reopen in a moment.`, plus subtitle `bob install-all` and
   sound `default`. That log entry is before the quit invocation. Then quit and
   `open -b md.obsidian`. The row is `↻ restarted`. The marker is gone. Exit 0.
2. **Immediate rerun.** No `display notification`. Row
   `– plugins unchanged · restart not needed`. Exit 0.
3. **Not running.** No `display notification`, no quit, no `open`. Row
   `– not running · plugins load on next launch`. Marker removed. Exit 0.
4. **Could not check.** The `is running` invocation exits 1. No `display notification`.
   Marker kept. Exit 1.
5. **Failed step.** A failing `just` stub makes bob-cli ✗. One banner titled
   `Obsidian needs a restart` whose body contains `step failed`. No quit and no `open`.
   Marker present. Exit 1.
6. **Pending recovery.** Rerun scenario 5 without the failing `just`, plugins unchanged,
   Obsidian running. Banner title `Restarting Obsidian`, body contains `earlier`. Quit
   and open run. Marker removed. Exit 0.
7. **Notification `osascript` exits 1, quit and open succeed.** Restart still completes,
   the Obsidian row is `↻ restarted`, there is no extra failure row for the banner, the
   marker is removed, and the exit status is 0.
8. **Quit fails.** First banner is `Restarting Obsidian`. Second is
   `Obsidian needs a restart` and contains `Quit failed`. `open` is not called. Marker
   kept. Exit 1.
9. **Does not quit within 30s.** Quit returns 0 but the running state stays true. After
   the real 30s wait, the second banner contains `did not quit`. `open` is not called.
   Marker kept. Exit 1. This scenario takes about 30 seconds; do not shrink the
   production timeout to speed it up.
10. **Relaunch fails.** Quit succeeds, `open` exits 1. Second banner contains
    `did not reopen`. Marker kept. Exit 1.
11. **Linux control.** Remove the `uname` stub, keep `pgrep` reporting running, and use
    the `$HOME/.local/bin/obsidian` stub from the previous plan. Output contains no
    `display notification`. The row is `↻ restart requested`. The marker is removed.
    Exit 0.

The real Mac banner cannot be exercised on this host. Say that in the final report, and
ask Bryan to run one real `just install-all` on the MacBook when plugins actually change
and confirm the banner appears before Obsidian quits.
