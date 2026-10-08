---
tier: tale
title: Make tmux.conf's extended-keys-format line safe on tmux < 3.5
goal:
  New tmux servers started via tm no longer show "invalid option extended-keys-format"
  on tmux 3.4 hosts, while tmux 3.5+ keeps csi-u.
size: small
decisions:
  add_regression_test:
    ask:
      Add a static bashunit regression test guarding the -q flag on that tmux.conf line?
    default: true
    why: Cheap guard; a live tmux test cannot catch this on athena's tmux 3.5a.
    answer: true
proposed_by: bbugyi200.athena.0y4
decided_by: auto
create_time: 2026-10-08 06:52:33
status: wip
---

# Plan: Make tmux.conf's `extended-keys-format` line safe on tmux < 3.5

## Problem

Starting a new tmux server with the `tm` script (tmuxinator wrapper in the chezmoi repo,
`home/bin/executable_tm`) shows a config-load error on some machines:

```
invalid option: extended-keys-format
```

## Root cause (verified)

- Commit `2ce5ba03` ("feat(tmux): update extended-keys config for ctrl+shift chords",
  2026-10-03) added `set -s extended-keys-format csi-u` to the shared
  `home/dot_config/tmux/tmux.conf` (currently line 52) so Ctrl+Shift chords reach
  Textual apps in the CSI-u format.
- `extended-keys-format` first appeared in tmux **3.5**. tmux 3.4 and earlier do not
  know the option, so `set-option` fails while the config is being parsed, and tmux
  shows the error when the first client attaches.
- `tmux.conf` is not a chezmoi template, so every tailnet machine gets the same file,
  but the machines run different tmux versions:
  - `athena`: Debian tmux 3.5a. Accepts the line (no error).
  - `apollo`: Ubuntu tmux 3.4. `invalid option: extended-keys-format`.
  - `mac`: Homebrew tmux 3.4. `invalid option: extended-keys-format`.
- Reproduced by sourcing the deployed config into an isolated server
  (`tmux -L <probe> -f /dev/null new-session -d; tmux -L <probe> source-file ~/.config/tmux/tmux.conf`).
  The 3.4 hosts print exactly the error above, and that is the only error the config
  produces on 3.4.
- `tm` only triggers the error indirectly. It runs `tmuxinator start`, which starts a
  tmux server that loads `tmux.conf`. The `tm` script itself needs no change.

## Fix

Use tmux's standard `-q` flag on that one line. The `-q` flag of `set-option`
"suppresses errors about unknown or ambiguous options." The flag exists in every tmux
version in use here. Verified results:

- tmux 3.5a sets the option exactly as before (`extended-keys-format csi-u`).
- tmux 3.4 silently skips the line (exit 0, no message).
- Errors about invalid values are still reported, so a mistyped value is not hidden.

On 3.4 this changes nothing about how the system behaves: tmux was already rejecting the
line there, so the option has never taken effect on those hosts.

Alternatives considered and rejected:

- A `%if "#{>=:#{version},3.5}"` guard. tmux compares format strings lexicographically,
  and version strings like `3.5a`/`next-3.6` make the comparison fragile. It also adds
  more syntax for no gain over `-q`.
- Turning `tmux.conf` into a chezmoi template keyed on the host. That is heavier, and it
  keys on the hostname when the real dependency is the tmux version.
- Upgrading tmux on apollo and the mac only. That fixes these two hosts but leaves the
  shared config incompatible with any tmux 3.4 install. It is still a reasonable
  optional follow-up for Bryan (`brew upgrade tmux` on the mac), but it is out of scope
  here.

## Implementation steps

All edits are in the **chezmoi** repo. If you are not already running in a chezmoi
workspace, open it first with `/sase_repo` (`sase repo open chezmoi -r "..."`), use the
path it prints, and read its `AGENTS.md`.

1. **`home/dot_config/tmux/tmux.conf`** — in the "Forward Kitty keyboard protocol"
   block:
   - Change `set -s extended-keys-format csi-u` to `set -sq extended-keys-format csi-u`.
   - Add one or two lines to the existing comment above it, explaining that
     `extended-keys-format` requires tmux 3.5+, and that `-q` makes tmux 3.4 (apollo,
     the mac) skip the line instead of showing "invalid option" at startup. Match the
     style of the surrounding comments.
   - Do not touch `set -s extended-keys on` or the `terminal-features` line. Both are
     valid on 3.4.

> [!decision] add_regression_test

2. **New `tests/bash/tmux_conf_portability_test.sh`** — a static bashunit regression
   test that follows the style of `tests/bash/zshrc_portability_test.sh` (header banner,
   `TMUX_CONF="${PWD}/home/dot_config/tmux/tmux.conf"`, `test_*` functions using
   `assert_contains`/`assert_not_contains`):
   - `test_extended_keys_format_is_quiet_on_older_tmux`: assert the file contains
     `set -sq extended-keys-format csi-u`, and does not contain the unguarded
     `set -s extended-keys-format` form.
   - Keep the test static (no live tmux server). The test host has tmux 3.5a, so a live
     sourcing test would pass even with the bug present.

3. **Verify**
   - `bashunit tests/bash/tmux_conf_portability_test.sh` passes. Then run
     `just test-bash` (or at least confirm no other bash test regresses).
   - Local tmux 3.5a: source the edited source-tree file into an isolated server and
     confirm there is no error and that the option still applies:
     ```bash
     S=extkeys_verify_$$
     tmux -L "$S" -f /dev/null new-session -d 'sleep 30'
     tmux -L "$S" source-file home/dot_config/tmux/tmux.conf   # expect no output
     tmux -L "$S" show -s extended-keys-format                   # expect: csi-u
     tmux -L "$S" kill-server
     ```
     (Sourcing may print TPM-related noise from the trailing
     `run '~/.tmux/plugins/tpm/tpm'` line. Only an `extended-keys-format` error counts
     as a failure.)
   - tmux 3.4 on apollo, which is reachable over Tailscale SSH: pipe the edited file
     over SSH and source it into an isolated server. Expect no
     `invalid option: extended-keys-format` output:
     ```bash
     ssh -o BatchMode=yes apollo 'f=$(mktemp); cat > "$f"; S=extkeys_verify_$$;
       tmux -L "$S" -f /dev/null new-session -d "sleep 30";
       tmux -L "$S" source-file "$f"; echo rc=$?;
       tmux -L "$S" kill-server; rm -f "$f"' < home/dot_config/tmux/tmux.conf
     ```
     The mac (`ssh mac`, tmux at `/opt/homebrew/bin/tmux`) is best-effort: it is offline
     unless its lid is open, so skip it if it does not respond.
   - Run `shellcheck` (if available) on the new test file.

4. **Deploy** — per the chezmoi `AGENTS.md`, run `chezmoi update -a --force` on athena
   after the commit lands. Apollo and the mac pick up the change on their next
   `chezmoi update`. The error only appears when a tmux server starts, so tmux servers
   that are already running do not need to be restarted.

## Out of scope

- The unrelated `unknown variable: TMUX_PLUGIN_MANAGER_PATH` message in athena's tmux
  message log (TPM bootstrap). Do not change it here.
- Upgrading tmux on apollo or the mac.

## Acceptance criteria

- On tmux 3.4 hosts, a new tmux server started via `tm` no longer shows
  `invalid option: extended-keys-format`.
- On tmux 3.5+ hosts (athena), `tmux show -s extended-keys-format` still reports
  `csi-u`, so the Ctrl+Shift chord forwarding for Textual apps is unchanged.
- The new bashunit test passes, and `just test-bash` has no new failures.
