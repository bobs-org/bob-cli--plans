---
tier: tale
title: Check both xlib sources concurrently and reduce pull latency
goal:
  Pull pending xlib intake from both athena and apollo while reducing empty-queue
  latency and repeated SSH setup.
size: medium
proposed_by: bbugyi200.apollo.08
create_time: 2026-09-17 09:53:43
status: wip
---

# Check both xlib sources concurrently and reduce pull latency

## Goal and scope

Make every eligible macOS invocation of `bob_xlib_pull` check both `athena` and
`apollo`, and pull pending intake from both reachable sources. Improve the common
empty-queue case and avoid paying repeated SSH connection setup costs when files exist.
Keep transfers synchronous with the pre-scan hook: all transfer work must finish before
`bob highlights scan` starts consuming local intake.

This is a `tale` with `size: medium`: one coding agent can implement the shell change,
focused regression tests, and the matching operational documentation. There is no need
for separate implementation phases or agents.

Open the linked `chezmoi` repository with `/sase_repo` before reading or editing it; use
the returned checkout and follow its `AGENTS.md`. Implementation files:

- chezmoi: `home/bin/executable_bob_xlib_pull`.
- chezmoi: new `tests/bash/bob_xlib_pull_test.sh`, following the existing bashunit
  suites; `Justfile` already discovers this directory through `just test-bash`.
- bob-cli: update the xlib bridge paragraph in `docs/vault-git-sync.md`. It still
  describes `xhome` and first-reachable-host fallback.

No changes to vault contents, memory files, global SSH configuration, host aliases, cron
schedules, or the Rust pre-scan implementation are needed. Do not introduce a new
runtime language, daemon, persistent reachability cache, or CLI interface.

## Findings and baseline

The current script is POSIX `/bin/sh` and exits immediately outside macOS. It resolves
`BOB_DIR` and `BOB_HIGHLIGHTS_XLIB_DIR`, takes the atomic
`${TMPDIR:-/tmp}/bob_xlib_pull.lock` directory lock, then tries `ssh ... true` on athena
followed by apollo and stops at the first success. It always invokes rsync and remote
empty-directory cleanup for that host, even if its queue is empty. The source remains
the remote user's `bob/xlib/`; environment overrides currently affect only the local
destination.

The SSH settings are `BatchMode=yes`, `ConnectTimeout=5`, and `ConnectionAttempts=1`.
Connection reuse is not configured on the MacBook. Thus an ordinary successful
invocation makes three independent connections: probe, rsync transport, and cleanup. The
trailing cleanup command also currently masks the rsync exit status.

Read-only measurements from the MacBook on 2026-09-17 found both source queues empty.
Three approximations of the existing path (`ssh true`, `rsync -an`, and `find` with
printing instead of deletion) took **0.84, 0.84, and 0.91 seconds**. Three pairs of
concurrent content probes took **0.92, 0.41, and 0.37 seconds**. These are small,
variable network samples, not a completed implementation benchmark. The MacBook's
observed SSH-session client is `/usr/bin/rsync` (Apple openrsync, protocol 29), with
OpenSSH 10.2p1. Its help lists `--remove-source-files`, `--ignore-existing`, and
`--timeout`. The remote rsync versions observed were 3.4.1 on athena and 3.2.7 on
apollo. Confirm the pre-scan environment's effective PATH; do not assume the Mac uses
Homebrew rsync.

`bob highlights scan` runs the hook synchronously and aborts before intake on a nonzero
hook result. Preserve quiet success when neither host is reachable, so existing local
intake can still be scanned.

## Implementation

### Concurrent, content-aware checks

1. Preserve the macOS guard, path expansion, existing invocation lock, and SSH aliases.
   Set up private per-invocation state only after acquiring the lock.
2. Start one background SSH content probe for each host before waiting for either.
   Retain both PIDs and each host's separate output/status. Do not use `wait -n`, Bash
   arrays, GNU `timeout`, or other features absent from macOS `/bin/sh`.
3. A successful probe distinguishes missing/empty source directories from pending
   non-directory entries. Use the Linux hosts' `find` with early termination after the
   first transferable entry; do not enumerate filenames or count the entire tree.
   Include symlinks and sidecars rather than checking only PDFs or regular files. Keep
   remote output machine-readable and propagate traversal errors; permission errors must
   not be mistaken for an empty queue.
4. Use `ssh -n` or equivalent stdin detachment for probes and cleanup. Do not put `-n`
   into rsync's SSH transport, which needs its protocol stdin.
5. Missing/empty queues require no rsync or cleanup connection. An arrival after the
   probe can be picked up on the next invocation; never cache an empty result between
   runs. Empty directory shells may remain until a later successful pull.

### Reuse connections and serialize destination writes

1. Use script-scoped OpenSSH multiplexing: `ControlMaster=auto`, finite `ControlPersist`
   (for example, 60 seconds), and a private per-invocation control path shared by that
   host's probe, rsync, and cleanup. This should normally mean one authenticated network
   connection per reachable host for the entire run. If the second host's idle master
   expires during a very long first transfer, permit a fresh connection for its
   transfer; do not lose queued work merely to satisfy a handshake-count target.
2. Make the control directory private and the socket path short enough for macOS Unix
   sockets. Use a short `mktemp -d /tmp/...` directory with separate host socket names;
   do not append a long hash to macOS's long `$TMPDIR`. Keep the existing invocation
   lock location unchanged. Quote every filesystem path and the rsync `-e` argument
   correctly. Do not modify `~/.ssh/config` or reuse/close unrelated user control
   sockets.
3. After launching both probes, handle athena and then apollo in a fixed order. Wait for
   athena, transfer/clean it if needed, then consume apollo's already running probe and
   transfer/clean it if needed. In particular, do not wait for apollo before beginning
   an already-ready athena transfer. Handle both hosts even if the first is unavailable
   or fails during transfer.
4. Run at most one rsync writer into the shared destination at a time. Keep archive
   transfer and sender-side removal after successful copying. Add `--ignore-existing` so
   pre-existing local files and files just received from athena are not overwritten by
   apollo; skipped copies must remain on their source. This deliberately favors
   retaining a collision over silently losing a source copy. Do not add checksum scans
   or an automatic deduplication/renaming system. Retaining identical duplicates as well
   as differing files is an intentional consequence: after local intake is consumed, a
   later pull may expose a duplicate to the scanner's existing library-collision
   handling. Document this destination-wins policy and verify it with actual rsync,
   including the Mac client, rather than relying only on mocks.
5. Reuse the same connection for best-effort removal of empty source directories after a
   successful transfer. Keep `find -mindepth 1 -type d -empty -delete`: leave the source
   root and all remaining payloads intact. Skip this cleanup on transfer failure. A
   separate multiplexed cleanup channel is simpler than a custom remote rsync protocol
   wrapper and avoids another network handshake.

### Timeouts, errors, and lifecycle

- Retain noninteractive authentication, one connection attempt, and the existing
  five-second connection timeout. Overlap both initial connection attempts; do not retry
  a failed probe through rsync or cleanup. Use script-local SSH keepalives to detect a
  lost established transport; do not impose a short total duration limit on legitimate
  large-file transfers.
- Checking an additional unavailable host cannot have zero cost in every case. When
  athena is fast and apollo is unreachable, completion may now wait for apollo's
  connection timeout. Report this explicitly rather than silently skipping apollo,
  arbitrarily shrinking the reliability window, or promising a hard five-second
  wall-clock bound (DNS/authentication can add time).
- Keep unreachable sources and absent/empty queues as successful no-work cases. Report
  genuine traversal, local setup, or transfer failures with the host where relevant,
  still attempt the other source, and return nonzero after all work is accounted for.
  Preserve a failure through cleanup. This fixes the current masked-transfer-status
  behavior; document that such failures stop the following scan, while simple offline
  hosts do not. Empty-directory cleanup alone remains best effort.
- The parent owns the invocation lock until all child activity ends. Normal exit and
  HUP/INT/QUIT/TERM must terminate/reap outstanding probe/transfer children, close only
  this invocation's masters, and remove owned temporary files before releasing the lock.
  Make cleanup idempotent and preserve signal exit statuses. Avoid deleting the lock
  while a child can still modify the destination.

## Verification and performance acceptance

Build an isolated bashunit harness with stubbed `uname`, `ssh`, and `rsync`, its own
temporary home/destination/lock paths, and per-host event logs. No test may touch the
live vault or tailnet. Cover these observable behaviors:

- Non-macOS and an existing invocation lock cause no network work.
- Both hosts are checked when athena succeeds; both nonempty hosts are pulled. All
  reachable/unreachable combinations, empty/missing queues, and traversal errors are
  distinguished correctly.
- A barrier-based SSH stub proves both probes start before either is released; do not
  rely solely on tight wall-clock thresholds. A blocked apollo probe must not prevent a
  ready athena transfer from starting. Transfers never overlap.
- Empty queues cause zero rsync and zero cleanup calls. Each successful host's probe,
  transfer, and cleanup use its same private control path and the expected SSH options.
  A failed probe receives no second connection attempt.
- A transfer failure does not prevent the second host's attempt; it survives subsequent
  successful cleanup and results in the documented exit status.
- Test relative, absolute, `~`, and `~/...` environment paths, spaces in paths,
  long/spaced `$TMPDIR`, source-path preservation, and correct rsync stdin.
- Terminate a run during a blocked probe and during transfer. Assert no surviving worker
  writes, no leaked owned sockets/state, and lock retention until workers have stopped;
  a subsequent invocation can acquire the lock.

Add a small real-rsync fixture check using disposable source/destination trees for
unique files on both sources, nested PDF/Markdown/textbundle companions,
same-relative-path files with differing contents, and an existing local file. Assert
received contents and exactly which remote fixture files remain. Include
file-versus-directory collisions and interrupted/failed transfers so retained data is
never deleted as a side effect of an optimization. Exercise the actual Mac openrsync
client against disposable Linux fixtures if available; never use real `~/bob/xlib`
payloads for destructive transfer tests.

Run from the linked chezmoi checkout:

```sh
sh -n home/bin/executable_bob_xlib_pull
bash -n tests/bash/bob_xlib_pull_test.sh
bashunit tests/bash/bob_xlib_pull_test.sh
just test-bash
git diff --check
```

Check Markdown formatting for the changed bob-cli document and run `git diff --check`
there too. The shell harness must exercise the script as `/bin/sh`; test it with macOS
`/bin/sh` as well when the Mac is available.

Use event barriers plus controlled-delay fake SSH handshakes for repeatable latency
regression checks. For two empty queues or two failed connection attempts, elapsed time
should track the slower check, not their sum. For populated sources, show that
subsequent channels reuse the connection and record handshake counts; distinguish
transfer-byte time from connection overhead. Compare a saved original script with the
candidate in isolated fixtures for both-empty, one-populated, both-populated, and
each-host-offline cases. Use several samples and medians.

Repeat read-only Mac measurements where practical and report results separately from
controlled tests. Acceptance is no systematic regression in the common both-reachable
empty case (target a lower median than the original), parallel timeout waits, and no
repeated handshake cost per source. If the implementation misses those criteria, profile
and simplify it before completion; do not claim a speedup solely from mocks or the
planning samples. Document the additional work and outage-timeout cost inherent in
checking both hosts.

Update the operational paragraph with the two-host behavior, empty fast path, collision
retention, timeout tradeoff, and failure semantics. Follow chezmoi's existing
`AGENTS.md` deployment requirement if a commit is later made; do not manually edit the
deployed `~/bin` copy.

## Technical references

OpenSSH supports sharing channels over one connection and a finite idle lifetime for a
master connection; it recommends private, uniquely identified control sockets. See the
[OpenSSH configuration manual](https://man.openbsd.org/ssh_config#ControlMaster).

Rsync documents that `--ignore-existing` leaves existing destination files alone, while
`--remove-source-files` removes successfully duplicated source entries. See the
[rsync manual](https://rsync.samba.org/ftp/rsync/rsync.1#opt--ignore-existing). Confirm
their combined behavior against the installed Mac client in fixtures.
