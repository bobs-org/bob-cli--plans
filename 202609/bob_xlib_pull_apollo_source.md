---
tier: tale
title: Pull Bob xlib intake from apollo instead of athena's LAN alias
goal:
  The MacBook's `bob_xlib_pull` pre-scan hook probes `xhome` then `apollo` for its
  Highlights intake source instead of `home` then `xhome`, and the one bob-cli doc
  sentence that names the old source host describes the new candidate list.
size: small
proposed_by: bbugyi200.apollo.d
---

# Plan: Pull Bob xlib intake from apollo instead of athena's LAN alias

## Goal

Change the SSH source-host candidate list in the chezmoi-managed `bob_xlib_pull` script
from `home xhome` to `xhome apollo`, and update the single sentence in bob-cli's
`docs/vault-git-sync.md` that still documents `home:bob/xlib/` as the pull source. No
other behavior changes.

## Background (verified at planning time, 2026-09-03)

`bob_xlib_pull` is the Highlights intake bridge. It is not a bob-cli file: it lives in
the **chezmoi linked repo** at `home/bin/executable_bob_xlib_pull`. chezmoi's
`.chezmoiroot` is `home`, so that file deploys to `~/bin/bob_xlib_pull`.

How it runs:

- The chezmoi-managed Bob config (`home/dot_config/bob/config.yml`) sets
  `highlights.pre_scan_command: PATH="$HOME/bin:$PATH" bob_xlib_pull`.
- `bob highlights scan` executes that command with `sh -c` before intake, and treats a
  nonzero exit as a scan failure (`run_pre_scan_command` in
  `src/native/highlights_ref/mod.rs`).
- On the MacBook, cron runs `~/bin/maybe_bob_highlights_sync -w` every 15 minutes,
  logging to `/var/tmp/maybe_bob_highlights_sync.log`.
- The script itself is Darwin-only: a `uname -s` gate makes it `exit 0` immediately on
  anything but macOS, so it is inert on apollo and on athena.

What the loop does today (chezmoi `home/bin/executable_bob_xlib_pull`, line 56):

```sh
for candidate in home xhome; do
  if ssh $ssh_opts "$candidate" true >/dev/null 2>&1; then
```

with `ssh_opts='-o BatchMode=yes -o ConnectTimeout=5 -o ConnectionAttempts=1'`. The
first host that answers becomes `$source_host`; if none answer the script exits 0. It
then rsyncs `"$source_host":bob/xlib/` into the Mac's `~/bob/xlib/` with
`--remove-source-files` and prunes empty remote directories under `~/bob/xlib`.

The host aliases come from the MacBook's `~/.ssh/config`:

| alias    | target              | machine                            |
| -------- | ------------------- | ---------------------------------- |
| `home`   | 192.168.1.156:34857 | athena, over the home LAN          |
| `xhome`  | 100.87.31.114:34857 | athena, over Tailscale             |
| `apollo` | 159.223.165.54      | apollo, the shared rendezvous host |

So the current list means "athena, by whichever route answers". The requested list means
"athena over Tailscale, otherwise apollo" — a different machine, not a different route.

Observed state that motivates the change and shapes the expectations below:

- From the MacBook, **both** `home` and `xhome` currently time out. athena is not
  reachable, so the bridge is a no-op today; `/var/tmp/maybe_bob_highlights_sync.log` is
  0 bytes.
- `apollo` answers from the MacBook, and it is the machine SASE agents now run on.
- apollo has **no** `~/bob` and no `~/bob/xlib` yet.
- The Mac side is live: `~/bob`, `~/bob/lib`, and `~/bob/xlib` all exist there.

## Implementation

### 1. chezmoi repo — the candidate list

Open the chezmoi repo with `/sase_repo` (`sase repo open chezmoi -r "..."`) and use the
path it prints. Do not edit `~/bin/bob_xlib_pull` directly; it is a chezmoi target and
would be overwritten.

In `home/bin/executable_bob_xlib_pull`, change the one loop header:

```sh
for candidate in home xhome; do   # before
for candidate in xhome apollo; do # after
```

Nothing else in the script changes. The remote source path (`"$source_host":bob/xlib/`),
the `find ~/bob/xlib -mindepth 1 -type d -empty -delete` cleanup, `$ssh_opts`, the
`$TMPDIR` lock directory, and the exit traps are all host-agnostic and already correct
for either candidate.

### 2. bob-cli repo — the stale doc sentence

In this repository, `docs/vault-git-sync.md` (the "Highlights bridge" section, around
line 120) currently reads:

> The pre-scan command pulls `home:bob/xlib/` into the MacBook's `~/bob/xlib/` with
> `rsync --remove-source-files`, removes empty source directories on athena, and lets
> `bob highlights scan` move the PDFs into `lib/` and write the matching `ref/` notes.

Rewrite it so it describes the candidate list rather than a single hard-coded host: the
pre-scan command picks the first reachable of `xhome` (athena over Tailscale) and
`apollo` (the rendezvous host), pulls that host's `bob/xlib/` into the MacBook's
`~/bob/xlib/` with `rsync --remove-source-files`, removes empty source directories on
that host, and lets `bob highlights scan` move the PDFs into `lib/` and write the
matching `ref/` notes. Add a sentence noting that when no candidate answers, the hook
exits 0 and the scan proceeds with whatever is already local. Keep the surrounding
prose, the 88-column wrapping, and the existing code fence untouched.

### 3. Explicitly out of scope

Do not modify any of these:

- `home/dot_config/bob/config.yml` — the `pre_scan_command` value is unaffected.
- `home/bin/executable_maybe_bob_highlights_sync` — the cron wrapper is host-agnostic.
- `home/bin/executable_get_sase_profile` and `home/bin/executable_get_sase_logs` — they
  still reference `home`/`xhome` for unrelated transfers.
- Anything under `sase/memory/` in either repo.

## Consequences to expect, not to "fix"

These follow directly from the requested host list. The implementer should verify them
rather than change the script to work around them:

1. **rsync will fail while apollo has no `~/bob/xlib`.** rsync exits 23 with "No such
   file or directory". That is not fatal: the script has no `set -e`, and its final
   command is `ssh ... >/dev/null 2>&1 || true`, so `bob_xlib_pull` still exits 0 and
   `bob highlights scan` still succeeds. `maybe_bob_highlights_sync` discards scan
   output when the scan succeeds, so nothing reaches the cron log. The bridge starts
   transferring as soon as `~/bob/xlib` exists on apollo. Do not add a remote-directory
   probe or change the exit handling as part of this plan.
2. **Each run now pays up to ~5s for the dead `xhome` probe** before falling through to
   apollo (`ConnectTimeout=5`, `ConnectionAttempts=1`, one candidate). That is the cost
   of keeping athena first, which is the requested order.
3. **Dropping `home` gives up athena's fast LAN path.** If athena returns, the pull runs
   over Tailscale instead. That is the requested trade.

## Validation

In the chezmoi repo:

1. `sh -n home/bin/executable_bob_xlib_pull` — the script's shebang is `/bin/sh`, so
   validate it as POSIX sh, not bash.
2. `grep -n 'for candidate in' home/bin/executable_bob_xlib_pull` prints exactly
   `for candidate in xhome apollo; do`, and `grep -n 'home xhome'` on that file returns
   nothing.
3. `just lint` and `just test` stay green. Neither newly covers this change — the
   bashunit suite under `tests/bash/` has no `bob_xlib_pull` test — so they are a
   regression check only. Do not add a bashunit test for this one-line host-list change.

In this (bob-cli) repo:

4. `git diff` touches only `docs/vault-git-sync.md`, and only within the "Highlights
   bridge" section. No Rust source changes, so `cargo` checks are not required for the
   doc edit; run `just all` only if some other file was touched by accident.

## Deployment

The chezmoi repo's own `CLAUDE.md`/`AGENTS.md` require `chezmoi update -a --force` after
any commit to that repo. Run it. On apollo it is effectively a no-op for this file (the
script is Darwin-gated), but the repo rule still applies.

The change only reaches the MacBook when the commit is on chezmoi's `master` **and** the
Mac applies it. Do not force-apply chezmoi on the Mac from a review branch. Once the
commit is on `master`:

```bash
ssh mac 'PATH="/opt/homebrew/bin:$PATH" chezmoi update -a --force'
ssh mac 'grep -n "for candidate in" ~/bin/bob_xlib_pull'
```

Flag to Bryan before running the first command: `chezmoi update -a --force` on the Mac
pulls all of chezmoi `master` and force-applies it, so it also deploys anything else
that landed since the Mac last updated. If that is not wanted right now, stop after the
commit and leave the Mac deployment to Bryan.

After deployment, confirm live behavior from apollo:

```bash
ssh mac 'PATH="$HOME/bin:$HOME/.cargo/bin:/opt/homebrew/bin:/usr/bin:/bin" bob_xlib_pull; echo "exit=$?"'
ssh mac 'PATH="$HOME/bin:$HOME/.cargo/bin:/opt/homebrew/bin:/usr/bin:/bin" bob highlights doctor'
```

Expect `exit=0`. While apollo has no `~/bob/xlib`, expect an rsync stderr line about the
missing source directory and zero files transferred. Confirm the Mac's `~/bob/xlib` is
unchanged and that no lock directory is left behind at
`${TMPDIR:-/tmp}/bob_xlib_pull.lock`.

## Acceptance

- `home/bin/executable_bob_xlib_pull` probes `xhome` then `apollo`, and that is the only
  change to the file.
- `docs/vault-git-sync.md` no longer claims the pull source is `home:bob/xlib/` and no
  longer hard-codes athena as the host whose empty directories get pruned.
- `sh -n` passes; chezmoi `just lint` and `just test` still pass.
- On the MacBook (once deployed), `bob_xlib_pull` exits 0 and `bob highlights scan`
  keeps succeeding, with `~/bob/xlib` left untouched while apollo has nothing to send.

If `bob_xlib_pull` exits nonzero on the Mac after deployment, that falsifies the
exit-path analysis above: capture the exact command output and stop rather than patching
the script's error handling on the fly.

## Follow-ups (not part of this plan)

- apollo has no `~/bob/xlib`, so the bridge stays a no-op until the Bob vault exists on
  apollo. That provisioning is separate work.
- `home/bin/executable_get_sase_profile` and `home/bin/executable_get_sase_logs` still
  target `home`/`xhome`. If athena is genuinely retired they need their own migration.
- The global `~/CLAUDE.md` core memory still describes athena as the home server. Any
  correction there is a sase memory change and must go through `/sase_memory_write`; do
  not edit it as part of this plan.
