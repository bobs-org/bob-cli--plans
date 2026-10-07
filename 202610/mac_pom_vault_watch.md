---
tier: tale
title: Mac pom re-syncs on daily-note change
goal:
  The Hammerspoon Pomodoro menu item reflects a changed current Pomodoro within about a
  second of the daily note being written, while polling bob less often than today.
size: medium
proposed_by: bbugyi200.apollo.5k
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5k](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5k.md)
- **COMMITS:**
  - [b0b4dcb](https://github.com/bbugyi200/dotfiles/commit/b0b4dcba1f5d93ad26d61e2848c6a5d1e5ca6b90)
    — feat(hammerspoon): re-sync pomodoro menu on daily-note change

# Mac Pom: Re-sync On Daily-Note Change Instead Of Waiting For The 15s Poll

## Problem

The Hammerspoon Pomodoro menu-bar item (the "mac pom") learns about a changed Pomodoro
only when its 15-second `hs.timer` poll next runs `bob pomodoro --show-stale`. So the
lag after an edit is anywhere from 0 to 15 seconds. `bob pomodoro` itself returns almost
instantly; the wait is the poll interval.

The Mac reads its local vault (`$HOME/bob`, kept in sync by the
`com.bbugyi.bob-vault-sync` LaunchAgent). `bob pomodoro` reads only today's daily note,
`$HOME/bob/<YYYY>/<YYYYMMDD>.md`. So a file-system watch on that note can trigger a
re-sync within about a second of any write: an Obsidian edit, a `bob` command, Bob Mac
Capture, or a vault-sync pull of an edit made on another machine. FSEvents is a kernel
facility, so the watch costs essentially nothing. With the watcher handling edits, the
poll becomes a safety net and can slow from 15s to 60s. The menu then reacts faster
**and** spawns `bob` (via a login zsh) about 4x less often.

There is one existing gap to close at the same time. `syncBobPomodoro` returns early
when a task is already running. A file change that lands while a sync is running would
be dropped until the next poll, because that running task may have read the note before
the write.

## Scope

All code changes are in the linked **`chezmoi`** repo. Open it with `/sase_repo`
(`sase repo open chezmoi -r "..."`), work only in the path it prints, and read its
`AGENTS.md` first. bob-cli code does not change: the `bob pomodoro` output contract
stays the same.

Files:

- `home/dot_hammerspoon/init.lua`: the runtime changes below.
- `tests/hammerspoon/init_spec.lua`: mock additions and new specs.
- `README.md`: the "Pomodoro menu bar" section is the spec. Update its polling sentence.

## Changes

### 1. Vault watcher (`init.lua`)

- Add a constant for the vault root, `os.getenv("HOME") .. "/bob"`. This matches the
  vault-sync LaunchAgent's hard-coded `~/bob`. Add a short comment that Hammerspoon does
  not inherit shell `BOB_DIR`.
- Create `bobPomodoroRuntime.vaultWatcher = hs.pathwatcher.new(vaultRoot, callback)` and
  start it. Watch the vault root rather than the year directory, so a new year directory
  and a day file that does not exist yet need no watcher rebuild.
- In the callback `(paths, flagTables)`, schedule a re-sync only if some path ends with
  today's day-file suffix, `"/" .. os.date("%Y") .. "/" .. os.date("%Y%m%d") .. ".md"`.
  Compute the suffix at callback time so midnight is handled. Matching on the suffix,
  rather than the full path, keeps the check correct if FSEvents reports a resolved or
  symlinked prefix. Ignore everything else: vault-sync touches `.git/` about every 15s,
  Obsidian rewrites `.obsidian/workspace.json`, and other notes also change. Those
  events must cost only the string check. Wrap the callback with the existing
  `guardedBobPomodoroCallback("vault watcher", ...)`.
- Debounce with `hs.timer.delayed.new(0.25, fn)`, stored as
  `bobPomodoroRuntime.vaultChangeDebounce`. Each matching event calls `:start()`, which
  restarts the countdown, so a burst of saves or a multi-file git pull produces one
  sync. When the timer fires, it calls a new `requestBobPomodoroResync()` (see 2).
- Make start-up failure safe, the same way `PingIndicator.start()` is wrapped. If
  `hs.pathwatcher.new` errors or returns nil, or `:start()` throws, log it with
  `hs.printf("Bob Pomodoro vault watcher unavailable: %s", ...)`, leave
  `vaultWatcher = nil`, and fall back to the 15s poll (see 3). A watcher failure must
  never break the menu item, hotkeys, ping, or config auto-reload.

### 2. Queue one follow-up sync for changes that land mid-sync (`init.lua`)

- Add `requestBobPomodoroResync()`. If `bobPomodoroRuntime.task` is nil, it calls
  `syncBobPomodoro()`. Otherwise it sets `bobPomodoroRuntime.resyncRequested = true`.
- In the task completion callback, right after the existing stale-task guard and the
  `bobPomodoroRuntime.task = nil` line, take and clear the flag. Then, after the outcome
  is handled (nonzero exit, empty output, parse failure, or success), call
  `syncBobPomodoro()` once if the flag was set. The stale-task early return must not
  re-run. Several requests made during one in-flight task collapse into a single
  follow-up.
- Only watcher-driven requests set the flag. Poll, wake/unlock, manual Refresh, and the
  zero-crossing sync still go straight to `syncBobPomodoro()` and keep their current
  "drop if in flight" behavior, because the running task already answers them. That
  keeps existing specs and behavior stable.
- If `task:start()` fails, clear `resyncRequested`.

### 3. Poll interval (`init.lua`)

- Add constants `BOB_POMODORO_POLL_INTERVAL = 60`, the safety net while the watcher
  runs, and `BOB_POMODORO_FALLBACK_POLL_INTERVAL = 15`, used when the watcher is
  unavailable. Create the watcher before `syncTimer`. Then build `syncTimer` with 60 if
  `vaultWatcher` started and 15 otherwise.
- The poll still covers what the watcher cannot see: midnight rollover to a new day
  file, and a missed event. The countdown is ticked locally and the zero-crossing sync
  stays, so a slower poll does not affect what is shown during a session.

### 4. Reload lifecycle (`init.lua`)

- Stop the retained `vaultWatcher` and `vaultChangeDebounce` with the existing
  `stopBobPomodoroRuntimeObject` calls. Reset them, and `resyncRequested`, to nil/false
  next to the other runtime resets, so `hs.reload()` never leaves a second watcher
  running.

### 5. Tests (`tests/hammerspoon/init_spec.lua`)

- Mock: add `hs.timer.delayed.new(delay, fn)`. It returns a started-object-style stub
  with `start` (counts calls, marks running), `stop`, `running`, and the stored `delay`
  and `fn`, recorded in a new `env.delayed_timers` list. Keep it out of `env.timers`, so
  the existing `#env.timers == 2` assertions still hold. Let the `hs.pathwatcher` mock
  optionally error or return nil via an `options` flag, for the fallback spec.
- Update the existing `#env.path_watchers == 1` assertions to 2: the config watcher plus
  the vault watcher. Identify each by `path` (`~/.hammerspoon/` vs `$HOME/bob`).
- New specs. Use the file's existing `os.time` / `os.date` stub helpers to pin "today":
  1. Installs a vault watcher on `$HOME/bob`, retained as `runtime.vaultWatcher`, and
     the sync timer interval is 60.
  2. A callback carrying today's day-file path starts the debounce. Firing the debounce
     `fn` starts exactly one new `bob pomodoro --show-stale` task.
  3. Unrelated paths start no debounce: `.git/FETCH_HEAD`, `.obsidian/workspace.json`,
     yesterday's note, another note in the year directory, and a path that only contains
     the date string elsewhere.
  4. Three matching events before the debounce fires yield one sync.
  5. A matching change while a task is in flight starts no second task. Completing the
     first task starts exactly one follow-up. Two changes during one in-flight task
     still yield exactly one follow-up. Completing the follow-up starts nothing further.
  6. A sync-timer tick while a task is in flight does not queue a follow-up. This pins
     the unchanged poll behavior.
  7. If `hs.pathwatcher.new` throws or returns nil for the vault root, a message is
     logged, the menu runtime still installs, the config watcher still exists, and the
     sync timer interval is 15.
  8. Reload stops the old vault watcher and the debounce timer, and installs fresh ones.
- Run `just test-hammerspoon`, `just fmt-lua`, and `just lint` in the chezmoi repo. All
  must pass.

### 6. Spec (`README.md`, "Pomodoro menu bar")

Replace the sentence that begins "The menu polls `bob pomodoro --show-stale` every 15
seconds…" so it states:

- the menu re-syncs within about a second when today's daily note changes on disk (an
  Obsidian edit, a `bob` command, or a vault-sync pull);
- it polls every 60 seconds as a safety net, or every 15 seconds if the file watcher
  cannot start;
- it refreshes on wake and unlock, offers a manual Refresh item, and re-syncs once when
  crossing zero.

Leave the rest of the section unchanged.

## Out Of Scope / Follow-Ups

- **Do not edit SASE memory.** bob-cli's `glossary:mac pom` strand
  (`sase/memory/glossary/mac-menu-bar-pomodoro-indicator.md`) still says the item "polls
  every 15 seconds". After this change lands, file one `memory` task bead through
  `/sase_new_task` in the bob-cli project. It should name that strand and the proposed
  rewording: re-syncs on a change to today's daily note, with a 60s safety poll and a
  15s fallback.
- No push-trigger (for example a `hammerspoon://` URL fired by bob or bob-plugins). The
  watcher already covers every writer without coupling the repos.
- Lag for edits made on apollo/athena is still bounded by vault-sync (the Linux
  watcher's 5s settle plus the Mac LaunchAgent's 15s pull interval). This change removes
  the extra 0–15s menu poll on top of that, but does not change sync cadence.

## Deploy / Verify

- After the chezmoi commit lands, run `chezmoi update -a --force` as that repo's
  `AGENTS.md` requires.
- On the MacBook, `chezmoi update` deploys `~/.hammerspoon/`. The existing config
  watcher then reloads Hammerspoon automatically.
- Manual check on the Mac: rename or retime the current Pomodoro in Obsidian. The menu
  title should change within about 1–3 seconds (Obsidian's own autosave delay plus the
  0.25s debounce), and the dropdown's "Last sync" should jump to that moment.
