---
tier: tale
title: Restart Hammerspoon from chezmoi when its Lua config changes
goal:
  A full chezmoi apply on the Mac that changes any Hammerspoon Lua file reliably
  restarts Hammerspoon so the new config is live, even when the running config is
  broken.
size: medium
decisions:
  config_watcher:
    ask: What should happen to the broken ~/.hammerspoon pathwatcher in init.lua?
    choices:
      remove:
        Delete it; the chezmoi hook is the only restart path (one restart per apply)
      repair:
        Keep a fixed copy (global, armed first, .lua-only, debounced) as a second path
    default: remove
    why:
      Every deploy goes through chezmoi apply; a second watcher only adds a racing
      reload
    answer: remove
proposed_by: bbugyi200.athena.0yd
decided_by: auto
create_time: 2026-10-08 12:17:13
status: wip
---

# Restart Hammerspoon From chezmoi When Its Lua Config Changes

## Goal

After `chezmoi apply` / `chezmoi update` on the Mac changes any Hammerspoon Lua file,
Hammerspoon restarts and loads the new config. It must work every time, including when
the running config is broken. Today this works at best intermittently.

All work happens in the `chezmoi` linked repository (Bryan's dotfiles). Open it first
with `sase repo open chezmoi -r "<reason>"`, read its `AGENTS.md`, and treat every path
below as relative to that checkout (`.chezmoiroot` is `home`, so `include` paths inside
templates are relative to `home/`).

## Diagnosis (verified while planning)

There is **no** chezmoi hook for Hammerspoon. `home/.chezmoiscripts/` has
`run_onchange_after_*` reload hooks for gpg-agent, kitty, and tmux, but nothing for
Hammerspoon. What Bryan remembers is an in-process watcher at the bottom of
`home/dot_hammerspoon/init.lua`, added in `b682f1a8` ("feat: auto-reload Hammerspoon
config on file changes"):

```lua
local configWatcher = hs.pathwatcher.new(os.getenv("HOME") .. "/.hammerspoon/", function()
	hs.reload()
end)
configWatcher:start()
```

It fails for these reasons:

1. **Garbage collection (root cause).** Hammerspoon runs `init.lua` via `loadfile` +
   `xpcall` (Hammerspoon `extensions/_coresetup/_coresetup.lua`). When the chunk
   returns, its top-level locals become unreachable, and no closure captures
   `configWatcher` as an upvalue. `hs.pathwatcher`'s `__gc` (`watcher_path_gc` in
   `extensions/pathwatcher/libpathwatcher.m`) stops and invalidates the FSEvents stream.
   The 0.5 s Pomodoro tick allocates constantly, so a collection runs soon after every
   load and auto-reload goes quiet. The commit's claim that "a module-level local keeps
   it from being garbage collected" is wrong. The rest of the config already retains its
   runtime objects in globals for this reason (`BobPomodoroCountdown`,
   `BobPingIndicator`).
2. **Armed last.** The watcher is the last statement in `init.lua`. Any runtime error
   earlier in the file or in a required module (see `742692c6` "repair Pomodoro startup
   bootstrap") stops it from ever starting, so the fix for a broken config is never
   picked up. A syntax error in `init.lua` disables everything.
3. **Unfiltered and undebounced.** Any file event under `~/.hammerspoon` triggers a
   reload, so it can fire mid-apply after only the first file has landed.
4. **It cannot deploy its own fix.** If the watcher in the running Hammerspoon is dead,
   it never notices the new `init.lua`.

## Decisions

- **chezmoi becomes the single owner of Hammerspoon restarts.** Add a darwin-only
  `run_onchange_after_` hook that hashes every Hammerspoon Lua file and restarts
  Hammerspoon after apply. This follows the existing kitty/tmux pattern.
- **The in-config `~/.hammerspoon` watcher (reviewer decision `config_watcher`).** The
  default is to delete it. Every deploy of `~/.hammerspoon` goes through `chezmoi apply`
  on the Mac, so a repaired watcher would only add a second, racing reload on every
  apply. It still could not recover from an `init.lua` that fails to parse, and it could
  not replace the dead watcher in the currently running config. The `repair` choice
  keeps a fixed copy for the rare targeted apply or hand edit, at the cost of a double
  reload on most applies. Either way, the Pomodoro vault watcher on `~/bob` is unrelated
  and stays.
- **Restart the process instead of calling `hs.reload()`.** `hs.ipc`/`hs` CLI and
  `hs.urlevent` reloads both need the running config to be healthy and to have
  registered a handler. A restart always loads the fresh config. Use signals (`pkill`)
  rather than AppleScript `quit`, so a non-interactive apply (agent, tmux, SSH) never
  blocks on a macOS Automation (TCC) consent prompt. Relaunch with
  `open -g -a Hammerspoon` so the app does not steal focus.
- **If Hammerspoon is not running, do nothing.** It loads the new files on its next
  launch. This matches the kitty and tmux hooks.
- **Hardcoded hash list plus a drift-guard test, not `glob`.** athena runs chezmoi
  **v2.13.1** (nixpkgs); CI installs the latest. `glob` does not exist in v2.13.1
  (verified: `function "glob" not defined`). Go templates parse the whole file even
  inside a false `{{ if }}`, so an unknown function would break `chezmoi apply` on
  athena. Use only `if`, `eq`, `include`, and `sha256sum`, the same functions the
  kitty/tmux hooks use. A bash test makes the list impossible to forget.
- **Known limitation, to document:** a targeted `chezmoi apply ~/.hammerspoon` does not
  run scripts (verified with v2.13.1). The next full apply restarts Hammerspoon, because
  `run_onchange_` compares against the last successful run. Until then, use
  Hammerspoon's menu-bar **Reload Config**.

## Implementation

### 1. New hook: `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`

Wrap it in the darwin guard used by `run_onchange_after_disable_dictation_hotkey.tmpl`,
so it renders to whitespace and chezmoi skips it on Linux. Put one hash comment per Lua
file, as the kitty/tmux hooks do. Intended shape (adjust wording freely, keep the
behavior):

```bash
{{- if eq .chezmoi.os "darwin" -}}
#!/bin/bash

# init.lua: {{ include "dot_hammerspoon/init.lua" | sha256sum }}
# ping_indicator.lua: {{ include "dot_hammerspoon/ping_indicator.lua" | sha256sum }}
# ping_window.lua: {{ include "dot_hammerspoon/ping_window.lua" | sha256sum }}
# pomodoro_countdown.lua: {{ include "dot_hammerspoon/pomodoro_countdown.lua" | sha256sum }}
# screenshot_region.lua: {{ include "dot_hammerspoon/screenshot_region.lua" | sha256sum }}
#
# Restart Hammerspoon every time any ~/.hammerspoon Lua file changes! Every Lua
# file under home/dot_hammerspoon/ needs a hash line above (enforced by
# tests/bash/hammerspoon_restart_hook_test.sh). A full restart, not hs.reload(),
# loads the new config even when the running one failed to load; signals avoid
# AppleScript Automation prompts during non-interactive applies.
set -uo pipefail

app="Hammerspoon"

function hammerspoon_running() {
  pgrep -x "${app}" >/dev/null
}

# Wait up to $1 tenths of a second for Hammerspoon to exit.
function wait_for_exit() {
  local tries="$1"
  while ((tries-- > 0)); do
    hammerspoon_running || return 0
    sleep 0.1
  done
  ! hammerspoon_running
}

if ! hammerspoon_running; then
  echo "Hammerspoon is not running; it will load the new config on its next launch."
  exit 0
fi

pkill -x "${app}"
if ! wait_for_exit 50; then
  pkill -KILL -x "${app}"
  if ! wait_for_exit 20; then
    echo "Hammerspoon did not exit; restart it manually to load the new config." >&2
    exit 1
  fi
fi

if ! open -g -a "${app}"; then
  echo "Hammerspoon was stopped but could not be relaunched; start it manually." >&2
  exit 1
fi
echo "Hammerspoon restarted to load the updated config."

# vim: ft=bash
{{- end -}}
```

Required behavior:

- Not running: print a message and exit 0. Never launch it.
- Running: SIGTERM (`pkill -x Hammerspoon`), wait up to about 5 s. If it is still alive,
  `pkill -KILL -x Hammerspoon` and wait up to about 2 s. If it is still alive after
  that, exit non-zero with a stderr message and do **not** launch a second copy.
- Relaunch with `open -g -a Hammerspoon`. If that fails, exit non-zero with a stderr
  message. chezmoi re-runs a failed `run_onchange_` script on the next apply (verified),
  and the failure is reported loudly.
- No `set -e`: a non-zero `pgrep` is normal control flow.

### 2. The `~/.hammerspoon` watcher in `home/dot_hammerspoon/init.lua`

Either way, leave the `~/bob` vault `hs.pathwatcher` and everything else untouched, and
format with `stylua` (`just fmt-lua`, or
`stylua home/dot_hammerspoon tests/hammerspoon`).

> [!decision] config_watcher = remove

- Delete the trailing block: the "Auto-reload the config…" comment,
  `local configWatcher = …`, and `configWatcher:start()`.
- In the ping-start comment just above it ("A ping failure must never break the hotkeys,
  the Pomodoro item, or auto-reload…"), drop the "auto-reload" mention.

> [!decision] config_watcher = repair

- Move the block to the very top of `init.lua`, before the `require` calls, so a runtime
  error later in the file or in a required module cannot stop it from arming.
- Retain the watcher and its debounce timer in a global runtime table (for example
  `BobConfigReload`), following the `BobPomodoroCountdown` pattern, so `__gc` can never
  collect them. Stop any previous watcher or timer found there before creating new ones.
- Reload only when a changed path ends in `.lua`. Debounce with
  `hs.timer.delayed.new(1, hs.reload)`, so one apply causes one reload after the whole
  burst of writes lands.
- Create it inside `xpcall` and log with `hs.printf` on failure, so a watcher failure
  never blocks the rest of the config.
- Replace the false "module-level local" comment with one that explains the global
  retention and calls the chezmoi hook the primary restart path.

### 3. Update `tests/hammerspoon/init_spec.lua`

Grep the spec for `path_watchers`, `delayed_timers`, and `.hammerspoon/` to catch every
affected assertion.

> [!decision] config_watcher = remove

- `assert.equals(2, #env.path_watchers)` becomes `1` in "loads in a fresh Lua state…",
  "logs a throwing ping start…", and "installs a vault watcher on $HOME/bob…".
- In "installs a vault watcher on $HOME/bob…", drop the `config_watcher` lookup and its
  `assert.is_not_nil`.
- In "falls back to the 15s poll when the vault watcher cannot start", expect `0` path
  watchers. Remove the assertion that the remaining watcher is `~/.hammerspoon/`.
- Add one regression test: loading `init.lua` registers no path watcher on
  `$HOME/.hammerspoon/` and never calls `hs.reload` (`env.reload_calls == 0`). Add a
  short comment saying that chezmoi's `run_onchange_after_restart_hammerspoon` hook owns
  config restarts.

> [!decision] config_watcher = repair

- Path-watcher counts stay as they are. The config debounce is a second delayed timer,
  so "installs a vault watcher on $HOME/bob…" must expect two delayed timers and find
  the vault debounce by identity rather than as `env.delayed_timers[1]`.
- Add tests showing that:
  - the config watcher lives in the global table and watches `$HOME/.hammerspoon/`;
  - it is already started when a later part of `init.lua` throws (add a `make_hs` option
    that makes, for example, `hs.menubar.new` throw);
  - a `.lua` path event starts the debounce, and a non-Lua path such as `.DS_Store` does
    not;
  - firing the debounce calls `hs.reload` exactly once;
  - a second load stops the previous watcher and timer.

### 4. New bashunit test: `tests/bash/hammerspoon_restart_hook_test.sh`

Follow the conventions in `tests/bash/sase_completion_test.sh` (renders a hook with
`chezmoi … execute-template < template`) and `tests/bash/install_luarocks_test.sh`
(fake-binary stubs on `PATH`, call log file, `set_up`/`tear_down` with `mktemp -d`). Use
`REPO_ROOT="${PWD}"`, since `just test-bash` runs from the repo root. Pass an empty temp
`--config` file to chezmoi so the user's config cannot leak in. Skip chezmoi- dependent
tests with `bashunit::skip` when `chezmoi` is missing.

Tests:

1. **Drift guard (no chezmoi needed).** Every file from
   `find home/dot_hammerspoon -type f -name '*.lua'` (path relative to `home/`) has
   exactly one `include "dot_hammerspoon/<rel>" | sha256sum` line in the hook. Every
   `include` in the hook names an existing file. On failure, print the missing or extra
   paths.
2. **Skipped on Linux.** Render the unmodified template with
   `chezmoi --config <empty> --source "${REPO_ROOT}" execute-template` and assert that
   the output is empty or whitespace-only. Skip when `uname -s` is `Darwin`. Running
   this under athena's chezmoi v2.13.1 also proves the template parses there.
3. **Behavior.** Render the darwin body by piping the template through
   `sed 's/eq \.chezmoi\.os "darwin"/true/'` into `execute-template`. Assert that
   `bash -n` accepts it, then run it with `bash` and stub `pgrep`, `pkill`, `open`, and
   `sleep` first on `PATH`. The `sleep` stub is a no-op so the test stays fast. A state
   file holds `running`/`stopped`, and an env var picks the stub's reaction to TERM and
   KILL. Every call is logged. Cases:
   - not running → exit 0, "not running" output, no `pkill` or `open` calls;
   - running, exits on TERM → one `pkill -x Hammerspoon`, no `-KILL`,
     `open -g -a Hammerspoon` called, exit 0, "restarted" output;
   - ignores TERM, dies on KILL → `pkill -KILL -x Hammerspoon` is called, then `open`,
     exit 0;
   - survives KILL → no `open` call, non-zero exit, error on stderr;
   - `open` fails → non-zero exit, error on stderr.

### 5. README

Add a short `## Hammerspoon config restarts` section to `README.md`, for example just
before `## Pomodoro menu bar`, covering:

- A full `chezmoi apply`/`chezmoi update` on macOS restarts Hammerspoon when any Lua
  file under `home/dot_hammerspoon/` changed. The hook is
  `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`.
- It does nothing when Hammerspoon is not running.
- A new Hammerspoon Lua module must be added to the hook's hash list.
  `tests/bash/hammerspoon_restart_hook_test.sh` fails until it is.
- A targeted `chezmoi apply ~/.hammerspoon` does not run scripts. Under
  `config_watcher = remove`, use the menu-bar **Reload Config** or run a full apply.
  Under `repair`, say that the in-config watcher picks it up. Add one sentence to that
  effect after the rollback snippet in "Bob Mac Capture cutover", which uses a targeted
  apply.

Format with `prettier --write --prose-wrap=always --print-width=88 README.md`.

## Verification

Run these from the chezmoi checkout:

- `just test-hammerspoon` passes (142 specs before this change, plus the new ones).
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh` passes, then `just test-bash`
  passes.
- `stylua --check home/dot_hammerspoon tests/hammerspoon` and
  `prettier --check --prose-wrap=always --print-width=88 README.md` are clean.
- Temporarily add a scratch `home/dot_hammerspoon/scratch.lua` (do not commit it),
  confirm the drift-guard test fails and names it, then delete it.
- After committing, `AGENTS.md` requires `chezmoi update -a --force`. On athena (Linux,
  chezmoi v2.13.1) this must finish without template errors, and the hook must not run.
  This is the real-world parse check.

Manual check on the Mac (Bryan, or an agent that happens to run on darwin):

1. `pgrep -x Hammerspoon`, and note the PID.
2. `chezmoi update`. The first apply after this change runs the new hook once. Expect
   "Hammerspoon restarted to load the updated config." and a new PID. This also replaces
   the running config whose watcher was garbage collected.
3. A second `chezmoi apply` with no Lua changes must not restart it (same PID).
4. Edit any Hammerspoon Lua file in the source, apply, and confirm it restarts again.

## Out Of Scope

- `hs.ipc`/`hs` CLI installation or a `hammerspoon://` URL handler.
- Restarting on non-Lua assets under `home/dot_hammerspoon/` (none exist today).
- Any change to the `~/bob` vault watcher, the ping indicator, or Pomodoro behavior.
