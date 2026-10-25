---
tier: tale
title: Fix the Hammerspoon restart hook's relaunch race
goal:
  A chezmoi apply on the Mac reliably relaunches Hammerspoon after stopping it, instead
  of failing with LaunchServices error -600 and leaving it stopped.
size: small
proposed_by: bbugyi200.athena.149
proposed_with: claude/opus
create_time: 2026-10-25 13:50:21
status: wip
---

# Plan: Fix the Hammerspoon restart hook's relaunch race (`open` error -600)

## Problem

On the MacBook, `chezmoi update -a` failed in the Hammerspoon restart hook:

```
_LSOpenURLsWithCompletionHandler() failed for the application /Applications/Hammerspoon.app with error -600.
Hammerspoon was stopped but could not be relaunched; start it manually.
chezmoi: .chezmoiscripts/restart_hammerspoon: exit status 1
```

Hammerspoon stayed stopped until Bryan started it by hand.

## Root cause (confirmed from the Mac's unified log)

The hook (`home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl` in the
**chezmoi** linked repo) sends SIGTERM, polls `pgrep -x Hammerspoon` until the process
is gone, then immediately runs `open -g -a Hammerspoon`. The process exiting does not
mean LaunchServices has unregistered it. `log show` on the Mac for the failed run:

- `13:41:42.845` WindowServer: process death, ASN `0x0-0x1e45e44`, pid 54642.
- `13:41:42.918` CoreServicesUIAgent:
  `-600 procNotFound ... attempting to send event 'aevt'/'rapp' [0x0,1e45e44 "Hammerspoon"]`.
  `open` still saw the old instance as running, so it sent a "reopen" Apple Event to the
  dead process instead of launching a new one.
- `13:41:42.930` launchservicesd: `QUITTING: pid=54642 asn=0x-0x1e45e44`. This is when
  LaunchServices dropped the old instance, about 12 ms too late.

So this is a timing race of about 100 ms between `pgrep` seeing the process exit and
LaunchServices forgetting it. It is not a permissions or SSH problem. The hook's design
(signals rather than AppleScript, a full restart rather than `hs.reload()`) is still
right. It just needs to wait for the right thing and retry when the relaunch fails for a
transient reason.

`lsappinfo` (always at `/usr/bin/lsappinfo` on macOS) shows what LaunchServices still
has registered. Verified on the Mac (macOS 26.5):

- `lsappinfo find bundleid=org.hammerspoon.Hammerspoon` prints
  `ASN:0x0-0x1e71e70-"Hammerspoon":` while Hammerspoon is registered. It prints an empty
  line once nothing is registered. The exit code is `0` in both cases, so callers must
  check the output, not the exit status.

## Where the work happens

All changes are in the **chezmoi** linked repo. Open it with `/sase_repo`
(`sase repo open chezmoi -r "<why>"`) and read its `AGENTS.md` first. Nothing in bob-cli
changes.

Files:

1. `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`
2. `tests/bash/hammerspoon_restart_hook_test.sh`
3. `README.md` (the "Hammerspoon config restarts" section)

## Changes

### 1. Hook: wait for LaunchServices to release the old instance, then retry the relaunch

In the darwin body of the template, keep the existing structure and messages
(`not running`, `did not exit`, `could not be relaunched`, `restarted`), and add:

- `bundle_id="org.hammerspoon.Hammerspoon"` next to `app="Hammerspoon"`.
- A `launchservices_registered` helper that returns success when
  `lsappinfo find "bundleid=${bundle_id}" 2>/dev/null` output contains `ASN:`. Use
  `grep -q 'ASN:'` on the output. A missing or failing `lsappinfo` must count as "not
  registered", so the hook never hangs or fails because of the probe.
- A `wait_for_launchservices_release <tenths>` helper, modeled on `wait_for_exit`, that
  polls `launchservices_registered` every 0.1 s for up to ~3 s (30 tries). It is
  best-effort: if the registration never clears, it does **not** exit. The hook goes on
  to the relaunch retries below.
- Call it once the process has exited, after the TERM/KILL block and before relaunching.
- Replace the single `open -g -a "${app}"` with a bounded retry loop of 5 attempts,
  sleeping 0.5 s between them. Capture each attempt's stderr (for example
  `err="$(open -g -a "${app}" 2>&1 >/dev/null)"`) so that a transient `-600` from an
  attempt that is later retried successfully does not print the alarming
  `_LSOpenURLsWithCompletionHandler() failed` line. If every attempt fails, print the
  last attempt's captured error to stderr, then the existing "Hammerspoon was stopped
  but could not be relaunched; start it manually." message, and `exit 1`.
- Add one or two sentences to the header comment explaining why: after the process
  exits, LaunchServices briefly keeps the old instance registered, and an immediate
  `open` then fails with -600 (procNotFound) because it tries to reopen the dead
  instance. Match the existing comment's tone and density.

Keep `set -uo pipefail` and bash 3.2 compatibility, since macOS `/bin/bash` is 3.2.
Avoid bash-4-only constructs.

### 2. Tests: `tests/bash/hammerspoon_restart_hook_test.sh`

The hook test runs with `PATH="${FAKE_BIN}:/usr/bin:/bin"`. On a Mac the real
`/usr/bin/lsappinfo` would be found, so the test **must** stub `lsappinfo` in
`FAKE_BIN`.

Stub changes in `write_host_stubs` and `run_hook`:

- New `lsappinfo` stub. It logs `lsappinfo $*` to `CALLS_FILE`. While the fake process
  is `running`, it prints an `ASN:0x0-0x1-"Hammerspoon":` line. After the process stops,
  it keeps printing that line for the next `FAKE_LS_LINGER` calls (default `0`) and then
  prints an empty line. Use a counter file in `TEST_TMP` to track the calls. It always
  exits 0, like the real tool.
- Change the `open` stub to support transient failures: `FAKE_OPEN_FAILS=N` (default
  `0`) makes the first N calls print an
  `_LSOpenURLsWithCompletionHandler() failed ... error -600.` line to stderr and exit 1.
  Later calls succeed and set the state to `running`. Use a counter file for this too.
  Replace `FAKE_OPEN_RC`, or express "always fails" as a large `FAKE_OPEN_FAILS`.
- Pass the new variables through `run_hook`, the same way the existing `FAKE_*`
  variables are passed.

Keep all existing tests passing, updating their stub variables as needed. Add tests:

- **`test_waits_for_launchservices_release_before_open`**: running, TERM succeeds,
  `FAKE_LS_LINGER=3`. Expect exit 0, `restarted` on stdout, exactly one `open` call, and
  in `CALLS_FILE` the `open` line comes after the last `lsappinfo` line that still
  reported the ASN. Asserting that the final `lsappinfo` call comes before `open` is
  enough.
- **`test_transient_open_failure_is_retried`**: `FAKE_OPEN_FAILS=1`. Expect exit 0,
  `restarted`, two `open` calls, and stderr that does **not** contain
  `_LSOpenURLsWithCompletionHandler`.
- **`test_persistent_open_failure_exits_nonzero`** (this can replace
  `test_open_failure_exits_nonzero`): every attempt fails. Expect exit 1, stderr with
  both `could not be relaunched` and the last `-600` error, and exactly 5 `open` calls.
- **`test_lingering_launchservices_registration_still_relaunches`**: `FAKE_LS_LINGER` is
  larger than the wait budget. Expect the hook to still call `open` and exit 0, which
  shows the wait is bounded and best-effort.
- **`test_not_running_does_nothing`**: extend this existing test to also assert that
  `lsappinfo` is never called.

The `sleep` stub is already a no-op, so the new loops stay fast.

### 3. README

In the "Hammerspoon config restarts" section, add one sentence: after stopping
Hammerspoon, the hook waits for LaunchServices to release the old instance and retries
the relaunch a few times. Only if all attempts fail does the apply report an error, in
which case you start Hammerspoon manually.

## Verification

From the chezmoi checkout:

```bash
bashunit tests/bash/hammerspoon_restart_hook_test.sh
just test-bash
```

All tests must pass on athena (Linux). On Linux the darwin hook is rendered via the
test's `sed` trick and runs entirely against stubs. Also run the repo's lint/format
checks (`just check` or the closest `just` target for shell files, for example
`shellcheck` or `shfmt`, if the Justfile defines them) and fix anything they report in
the touched files.

Optional, best-effort, only if the Mac is reachable (`ssh -o ConnectTimeout=8 mac`, see
the tailnet memory): sanity-check that
`lsappinfo find bundleid=org.hammerspoon.Hammerspoon` still prints an `ASN:` line. Do
**not** restart Hammerspoon on the Mac from the agent. Bryan will check the real
behavior on his next `chezmoi update -a` that touches a Hammerspoon Lua file.

## Commit / apply

Commit only through the normal SASE finalizer flow for the opened chezmoi repo. Per the
chezmoi repo's core memory, run `chezmoi update -a --force` after the commit lands. On
athena (Linux) the hook renders empty, so this is a no-op for Hammerspoon.

## Out of scope

- Switching to `open -n` or AppleScript-based relaunching. `-n` could start duplicate
  instances, and AppleScript would trigger Automation permission prompts during
  non-interactive applies.
- Any change to Hammerspoon's Lua config or to bob-cli.
