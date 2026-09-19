---
tier: tale
title: Copy MacBook screenshots to Athena in parallel with Apollo
goal:
  "Ctrl-Alt-Shift-S still captures a region on the MacBook, but macscrot uploads the PNG
  to apollo and athena at the same time, reports which hosts received it, and leaves
  #sshot able to use a local copy or fetch from whichever host has the file."
size: small
proposed_by: bbugyi200.apollo.0q
create_time: 2026-09-19 09:25:12
status: wip
---

# Copy MacBook screenshots to Athena in parallel with Apollo

## 1. Scope and repository

All implementation changes belong in the linked **`chezmoi`** repository (the dotfiles
repo). Open it first and use only the path printed by:

```bash
sase repo open chezmoi -r "Implement parallel MacBook screenshot upload to apollo and athena"
```

The Hammerspoon keymap itself does not change. `Ctrl-Alt-Shift-S` (`ctrl+alt+shift+s`,
the same chord as `ctrl+shift+option+s`) already shells out to `~/bin/macscrot`; the
work is to make that script dual-write, then keep `#sshot` correct when the file may
already exist on the machine that expands the xprompt.

Do not change `bob-cli` source. Do not mention any numbered workspace directory in
commits or docs.

Expected files:

| File                             | Responsibility                                                                                      |
| -------------------------------- | --------------------------------------------------------------------------------------------------- |
| `home/bin/executable_macscrot`   | Capture locally, then upload to `apollo` and `athena` in parallel.                                  |
| `home/sase/xprompts/sshot.yml`   | Resolve the nth screenshot from the union of local/apollo/athena copies.                            |
| `home/dot_hammerspoon/init.lua`  | Comments only: the hotkey still calls macscrot; destinations are both hosts.                        |
| `tests/bash/macscrot_test.sh`    | New bashunit coverage for parallel upload, partial failure, and cancel.                             |
| `tests/bash/sshot_fetch_test.sh` | New bashunit coverage if fetch logic is extracted; otherwise keep tests next to the xprompt helper. |

`home/dot_hammerspoon/screenshot_region.lua` is unchanged. The region picker already
hands a rectangle to `runMacscrot`; that boundary stays.

## 2. Current behavior

End-to-end path today:

1. Hammerspoon binds `{ctrl, alt, shift}+s` in `home/dot_hammerspoon/init.lua` and runs
   `ScreenshotRegion.pick`, then `~/bin/macscrot 'x,y,w,h'`.
2. `macscrot` writes `~/tmp/screenshots/YYYYMMDD_HHMMSS.png` on the MacBook, then
   **sequentially** `ssh apollo 'mkdir -p ~/tmp/screenshots'` and `scp`s the PNG to
   `apollo:~/tmp/screenshots/<basename>`.
3. On success it copies `~/tmp/screenshots/<basename>` to the clipboard and shows a
   macOS notification titled "Screenshot uploaded" with subtitle **Apollo**.
4. `#sshot` / `#sshot:<n>` (`home/sase/xprompts/sshot.yml`) always `ssh apollo` to list
   `ls -1t -- *.png`, then `scp`s that basename onto the current machine.

The script's header states the original invariant: _the MacBook never SSHes into Athena
directly; it only pushes to Apollo_, and Athena later pulls via `#sshot`. That was
written when Apollo was the single rendezvous. It is now the thing to retire.

Chezmoi already gives the MacBook a Tailscale SSH alias for both hosts
(`home/private_dot_ssh/private_tailnet.conf`: `Host apollo` and `Host athena`). Athena
is reachable; the Mac simply never uses that alias in this path. `bob_xlib_pull` already
talks to both hosts in parallel from the Mac, so dual-host SSH from Darwin is an
established pattern in this repo.

There are **no** bashunit tests for `macscrot` or `#sshot` today. Hammerspoon tests only
assert that `init.lua` loads and installs two hotkeys; they do not need to change unless
comments are the only edit.

## 3. Design decisions

These are settled. Do not reopen them during implementation.

### 3.1 MacBook pushes to both hosts. Apollo does not fan out.

Do not implement "scp to apollo, then apollo copies to athena". That is sequential from
the Mac, still fails when Apollo is down, and contradicts "in parallel". The MacBook
already has `ssh athena` via chezmoi-managed `~/.ssh/tailnet.conf`. Use it.

### 3.2 Both uploads start before either is required to finish.

After a successful local capture, launch the apollo upload and the athena upload as
sibling background jobs, then `wait` for both. A slow or hung host must not delay the
other host's `mkdir`/`scp`. Prove this with a test that blocks one host and still
observes the other host's transfer start (same idea as
`tests/bash/bob_xlib_pull_test.sh`'s blocked-probe cases).

### 3.3 Fast-fail SSH. Never wait on TCP's default timeout.

Bare `ssh apollo` / `scp` have no `ConnectTimeout` today, so a down host can stall the
notification for a long time. Both uploads must use BatchMode and a short connect
timeout (on the order of 5–8 seconds, consistent with `bob_xlib_pull`'s
`ConnectTimeout=5`). Suggested options:

```text
-o BatchMode=yes -o ConnectTimeout=8 -o ConnectionAttempts=1
```

Pass the same options to `ssh` and `scp`. Do not prompt for passwords.

### 3.4 Success means at least one remote received the file.

| Local capture     | Apollo | Athena | macscrot exit | Notification                                                |
| ----------------- | ------ | ------ | ------------- | ----------------------------------------------------------- |
| cancelled / empty | —      | —      | 1             | Hammerspoon "Screenshot failed" (unchanged)                 |
| ok                | ok     | ok     | 0             | "Screenshot uploaded" / subtitle names both hosts           |
| ok                | ok     | fail   | 0             | success, subtitle names Apollo, body warns Athena failed    |
| ok                | fail   | ok     | 0             | success, subtitle names Athena, body warns Apollo failed    |
| ok                | fail   | fail   | 1             | Hammerspoon "Screenshot failed"; stderr names both failures |

Apollo is no longer a hard requirement. Both machines are first-class SASE hosts; a
home-network or droplet blip must not discard a copy that did land. Clipboard copy of
`~/tmp/screenshots/<basename>` still runs whenever at least one remote succeeded — that
path is valid on whichever host has the file.

Keep the existing clipboard and notification helpers. Change the subtitle so it is not
hardcoded to `"Apollo"`. Prefer a stable, human list such as `Apollo + Athena`,
`Apollo`, or `Athena`. Partial failure belongs in the notification message and in
stderr; it must not flip a one-host success into exit 1.

### 3.5 `#sshot` must not assume Apollo is the only store.

If macscrot dual-writes and `#sshot` still always lists Apollo, then:

- On Athena, `#sshot` pays a redundant scp even when the PNG is already local.
- If Apollo missed an upload that Athena has, `#sshot n` can pick the wrong file (local
  `ls -1t` and Apollo `ls -1t` diverge under partial replica).
- If Apollo is down, `#sshot` fails even when the file is sitting in `~/tmp/screenshots`
  on the current host.

`#sshot` therefore treats **basename** (`YYYYMMDD_HHMMSS.png`) as the identity:

1. Best-effort list `*.png` from (a) the local `~/tmp/screenshots` directory, (b)
   `apollo`, and (c) `athena`. Skip `ssh` to the current hostname; local listing already
   covers that host. Each remote listing uses the same short SSH timeouts. A failed
   listing is empty, not fatal.
2. Union by basename. Sort newest-first. The filename is already a sortable timestamp,
   so sorting by basename is equivalent to `ls -1t` when mtimes agree and is more
   correct when they do not.
3. Pick the nth entry (`n` still defaults to 1, still rejected if `< 1`).
4. If that basename is already a nonempty local file, use it. Otherwise `scp` from a
   host whose listing contained it (try the other Linux host; do not scp from `mac`).
5. Print `local_path=...` as today so the xprompt inject step stays
   `@file:{{ fetch.local_path }}`.

If no source has an nth screenshot, keep a clear error.

Extract the fetch logic into `home/bin/executable_sshot_fetch` (or an equivalent `~/bin`
helper) if the bash would otherwise be too large to keep readable inside `sshot.yml`.
The xprompt should stay a thin wrapper that calls the helper with `n`. A helper is the
preferred shape because chezmoi already tests `~/bin` scripts with bashunit and this
listing/fetch behavior is easy to get wrong.

### 3.6 Test seams.

`macscrot` currently invokes `/usr/sbin/screencapture` by absolute path, so Linux CI
cannot stub it via `PATH`. Make the capture binary overridable (`SCREENCAPTURE_BIN`,
default `/usr/sbin/screencapture`) without changing production behavior on the Mac. Keep
`ssh` and `scp` as unqualified names so tests can inject stubs via `PATH`, matching
`tests/bash/bas_test.sh`.

## 4. Implementation

### 4.1 `home/bin/executable_macscrot`

- Rewrite the header comment. The MacBook now pushes to **both** tailnet Linux hosts.
  Drop "never SSHes into Athena".
- After a nonempty local PNG exists, upload to `apollo` and `athena` in parallel as
  specified in §3.2–§3.4. Per host: `mkdir -p ~/tmp/screenshots` then `scp -p` to
  `~/tmp/screenshots/<basename>` (same remote layout as today).
- Collect each host's exit status. Do not let `set -e` kill the script at the first
  `wait` failure; both jobs must be waited on.
- Preserve cancel/empty behavior: delete the stub file, print the existing message, exit
  1, and do not start any upload.
- Preserve the `x,y,w,h` region argument and its validation. Interactive
  `screencapture -i` without a region stays as a fallback.
- Stdout should name every successful remote (`apollo: ~/tmp/screenshots/...`,
  `athena: ~/tmp/screenshots/...`) so a human or log can see where the file went.
- Warnings for clipboard or notification helper failures stay non-fatal, as today.

### 4.2 `#sshot` consumer

- Update `home/sase/xprompts/sshot.yml` so the hidden `fetch` step uses the union
  listing in §3.5 instead of hard-coding `ssh apollo` + `scp` from Apollo.
- Keep the `n` input, the `local_path` output, and the `@file:{{ fetch.local_path }}`
  inject step.
- Update comments that say Athena always fetches from Apollo.

### 4.3 Hammerspoon comments only

In `home/dot_hammerspoon/init.lua`, the comment above the screenshot hotkey currently
says it uploads to Apollo via `~/bin/macscrot`. Change that to both hosts. Do not change
modifiers, key, `runMacscrot`, or failure notification behavior. macscrot still owns the
success notification.

### 4.4 Tests

Add `tests/bash/macscrot_test.sh` (bashunit, same style as `bas_test.sh` /
`bob_xlib_pull_test.sh`):

- Stub `SCREENCAPTURE_BIN` to write a nonempty fake PNG to the requested output path
  (and a cancel case that writes nothing / exits 0 with an empty file).
- Stub `ssh` and `scp` on `PATH` so they record host, mkdir, and destination, and can
  fail or block per host. They must not touch the network.
- Point the local screenshot directory at a temp dir (override `HOME` or add a narrow
  env override if `HOME` is too blunt; prefer whatever keeps the script honest about
  `~/tmp/screenshots`).
- Stub `pbcopy` / `osascript` so clipboard and notification side effects do not require
  macOS. The script already degrades when those binaries are missing; tests may either
  stub them or assert the warning path.
- Required cases:
  - both hosts succeed → exit 0, both mkdir+scp happened, clipboard path is
    `~/tmp/screenshots/<basename>`;
  - uploads overlap in time (block one host, observe the other start);
  - Apollo fails / Athena succeeds → exit 0;
  - Athena fails / Apollo succeeds → exit 0;
  - both fail → exit 1, no clipboard-success path;
  - capture cancelled → exit 1, zero ssh/scp invocations.

If fetch is extracted to `~/bin/sshot_fetch`, add `tests/bash/sshot_fetch_test.sh`:

- union of local + one remote, nth pick;
- skip scp when the chosen basename is already local;
- Apollo listing empty/failing still returns a local or Athena file;
- `n < 1` and missing nth entry error.

Do not add Hammerspoon specs for this unless `init.lua` behavior changes (it should
not).

## 5. Out of scope

- Changing the keymap, region picker, or Hammerspoon task wiring.
- One-time backfill of screenshots that already exist only on Apollo. If the user wants
  that later, a manual `rsync -a apollo:~/tmp/screenshots/ athena:~/tmp/screenshots/` is
  enough; do not add migration code.
- Retention / cleanup of `~/tmp/screenshots`.
- Bob Mac Capture, `mac_copy_png`, or any other hotkey.
- Edits to `~/.ssh/config` or `private_tailnet.conf`.

## 6. Validation

From the chezmoi checkout:

1. `just test-bash` — new macscrot (and sshot-fetch, if extracted) cases plus the
   existing bashunit suite.
2. `just test-hammerspoon` — still loads `init.lua` and still installs two hotkeys.
3. If Lua comments in `init.lua` changed,
   `stylua --check home/dot_hammerspoon tests/hammerspoon`.
4. `just lint-keep-sorted` if `sshot.yml` was edited inside a keep-sorted block (it
   currently is not; do not invent a keep-sorted region).

Manual smoke after deploy (report explicitly if the coding agent is not on the MacBook):

1. On the MacBook, `chezmoi update` / apply so `~/bin/macscrot` is new, then reload
   Hammerspoon (path watcher should reload on apply).
2. `Ctrl-Alt-Shift-S`, capture a region.
3. Confirm `~/tmp/screenshots/<basename>.png` exists on **both** `apollo` and `athena`
   with the same basename.
4. On Apollo and on Athena, `#sshot` injects that newest image without failing if the
   other host is briefly unreachable, as long as a copy exists locally or on the
   remaining host.

## 7. Deployment

Chezmoi's post-commit hook is `chezmoi update -a --force`. That applies the tree on the
machine that committed. The MacBook is the machine that runs `macscrot`; it must receive
`~/bin/macscrot` and the updated Hammerspoon comments before the new behavior exists.
Apollo and Athena need the updated `#sshot` xprompt. State this in the implementation
notes if the coding host is not the MacBook.

## Acceptance criteria

- `Ctrl-Alt-Shift-S` still opens the existing region picker and still calls
  `~/bin/macscrot` with `x,y,w,h`.
- A successful capture copies the PNG to apollo **and** athena concurrently, not
  sequentially, with short SSH timeouts.
- One host failing does not fail the capture if the other host succeeded; both failing
  still fails the capture.
- The success notification names the host(s) that received the file.
- `#sshot` / `#sshot:<n>` picks the nth newest basename across local/apollo/athena and
  does not scp a file that is already local.
- New bashunit tests cover parallel start, both-success, each partial-failure direction,
  both-failure, and cancel-with-no-upload.
- The old "MacBook never SSHes into Athena" comment is gone.
