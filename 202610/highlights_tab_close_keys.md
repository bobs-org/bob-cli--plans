---
tier: tale
title: Highlights tab-close leader keys in Hammerspoon
goal:
  While Highlights is frontmost, \\, \<, and \> close the other, left, or right tabs of
  its focused window.
size: medium
decisions:
  scope:
    ask: Should `\\` also close the tabs in Highlights' other windows?
    choices:
      window:
        Close only the focused window's other tabs; other Highlights windows stay open
      app:
        Also close every tab in Highlights' other windows, leaving only the current tab
        open
    default: window
    why: matches macOS's own Close Other Tabs, and `\<` / `\>` can only mean one tab bar
    answer: window
  reopen:
    ask: Should a fourth key, `\u`, reopen the last batch these keys closed?
    choices:
      none:
        Ship only the three requested keys; recover a mistaken close with Highlights'
        File › Open Recent
      backslash_u:
        Add `\u` to reopen the last closed batch's PDFs (in memory only; a reload
        forgets it)
    default: none
    why:
      keeps to the three requested keys, and Open Recent already recovers a mistaken
      close
    answer: backslash_u
proposed_by: bbugyi200.athena.146
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 12:12:02
status: wip
---

# Highlights tab-close leader keys in Hammerspoon

## Goal

While Highlights (bundle id `net.highlightsapp.universal`) is the frontmost app, add
three vim-style leader sequences. `\` is the leader. Press it, then the second key
within 1 s:

| Keys | Action                                                                        |
| ---- | ----------------------------------------------------------------------------- |
| `\\` | Close every other tab in the focused Highlights window; keep the selected tab |
| `\<` | Close every tab to the left of the selected tab                               |
| `\>` | Close every tab to the right of the selected tab                              |

Outside Highlights, and while typing in any text field, these keys behave exactly as
they do today.

## Why Hammerspoon (chezmoi), not Bob Mac Capture

Bryan suggested both. This plan picks Hammerspoon, for these reasons:

- **Leader sequences need an active event tap.** The tap must swallow `\`, wait for the
  next key, let a non-matching key through untouched, and leave `\` alone in text
  fields. Hammerspoon's `hs.eventtap` does exactly that. The chezmoi config already uses
  one in `home/dot_hammerspoon/screenshot_region.lua`, so Hammerspoon already holds
  Accessibility. Bob Mac Capture has two kinds of key handling. Its Carbon chord hotkeys
  (`HotKeyRegistry`, driven by `HighlightsTakeover`) cannot see a second key that is not
  registered, so they cannot release it either. Its `NSEvent` global monitor
  (`HighlightsSaveKeyMonitor`) is passive. The app's `docs/permissions.md` also promises
  that it "never consumes or re-posts the key." Adding an active tap to that signed app
  would widen its privileged surface. It would also mean rewriting that promise, plus
  the Settings and docs pages, for what is a personal key binding.
- **Faster iteration.** A full `chezmoi apply` restarts Hammerspoon by itself. Bob Mac
  Capture needs a signed rebuild and reinstall, and an ad-hoc reinstall silently drops
  its Accessibility grant.
- **Accepted costs.** This plan re-implements in Lua a small part of Bob Mac Capture's
  Accessibility (AX) close ladder; it does not reuse `AXHighlightsTabsReader`. Tabs
  closed by these keys also do not go onto Bob Refs' ⇧⌘T closed-tab stack.

## Facts carried over from Bob Mac Capture

Read `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift` in the linked
`bob-mac-capture` repo (open it with `/sase_repo`; read only, do not edit). Mirror these
proven behaviours:

- Highlights uses native macOS window tabs. From a window, search `AXChildren` to a
  depth of at most 3 for the `AXTabGroup` role. Do not descend into `AXScrollArea`,
  `AXWebArea`, `AXTextArea`, `AXTable`, or `AXOutline`. The group's `AXTabs` attribute
  gives the tab buttons in tab-bar order. A button's `AXTitle` is the PDF's file name,
  and `AXValue == 1` marks the selected tab.
- **Highlights' per-tab close control accepts `AXPress` but does not close the tab.**
  The close that works has three steps: `AXPress` the tab button to select it, confirm a
  fresh read shows it selected, then `AXPress` the focused window's `AXCloseButton`.
  That closes only the visible tab. Re-read the focused window after selecting, because
  each native tab is its own `NSWindow`.
- A close may raise a save sheet (an `AXSheet` child of a window). When it does, stop.
  Never press close again.
- When only one tab is left, the tab bar (and its `AXTabGroup`) can disappear.
- Budgets: 0.4 s AX messaging timeout, 50 ms poll interval, 2 s per close.

## Files

All changes go in the linked **`chezmoi`** repo (open it with `/sase_repo`). There are
no bob-cli or bob-mac-capture changes. Read the chezmoi repo's `AGENTS.md` first. Follow
its pattern of a pure module plus a runtime module, like `ping_window.lua` and
`ping_indicator.lua`.

1. `home/dot_hammerspoon/highlights_tabs.lua` (new, **pure**): no `hs` references at
   all, so specs can `loadfile` it directly. It holds the constants, the key classifier,
   the leader state machine, the close runner (adapter and scheduler are injected), and
   the alert copy.
2. `home/dot_hammerspoon/highlights_tab_keys.lua` (new, **runtime**): the `hs` glue. It
   holds the event tap, the application watcher, the real AX adapter, `hs.alert`, and
   reload-safe global state.
3. `home/dot_hammerspoon/init.lua`: `require("highlights_tab_keys")` and start it inside
   `xpcall`, exactly like `PingIndicator.start()`. Log failures as
   `Bob Highlights tab keys start failed: …`. A throwing start must never break the
   other hotkeys or the Pomodoro item.
4. `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`: add a hash line
   for each new module. `tests/bash/hammerspoon_restart_hook_test.sh` enforces this.
5. Tests: add `tests/hammerspoon/highlights_tabs_spec.lua` and
   `tests/hammerspoon/highlights_tab_keys_spec.lua`, and update
   `tests/hammerspoon/init_spec.lua`.
6. `README.md`: add a `## Highlights tab keys` section (see Docs).

## Pure module: `highlights_tabs.lua`

Constants: `BUNDLE_ID = "net.highlightsapp.universal"`, `LEADER_TIMEOUT_SECONDS = 1.0`,
`POLL_SECONDS = 0.05`, `SELECT_TIMEOUT_SECONDS = 1.0`, `CLOSE_TIMEOUT_SECONDS = 2.0`.

### Key model and leader state machine

The runtime turns each keyDown into a plain table:
`{ chars = event:getCharacters(), escape = <keycode is hs.keycodes.map.escape>, cmd, alt, ctrl, shift, autorepeat }`.
Matching uses characters, so it follows the keyboard layout.

`M.new_leader({ timeout = LEADER_TIMEOUT_SECONDS })` returns an object with
`:on_key(key, now, can_arm)` →
`{ swallow = bool, command = nil | "others" | "left" | "right" }` and `:cancel()`.
Rules:

- **Idle.** The leader arms only on `chars == "\\"` with none of cmd/alt/ctrl/shift
  held, with `autorepeat` false, and when `can_arm()` returns true. Call `can_arm`
  lazily, only for that key, because it costs an AX call. When it arms, record
  `armed_at = now` and swallow the key. Every other key passes through
  (`swallow = false`).
- **Pending.** First, if `now - armed_at > timeout`, drop back to idle and handle the
  key as idle. That way `\`, a pause of more than 1 s, then `\` re-arms instead of
  firing. Otherwise:
  - An autorepeat key is swallowed and the leader stays pending, so holding `\` never
    fires `\\`.
  - Any key with cmd, alt, or ctrl held cancels the leader and passes through.
  - `"\\"` with no shift fires `others`, `"<"` fires `left`, and `">"` fires `right`.
    Each is swallowed and the leader returns to idle.
  - Escape cancels the leader and is swallowed.
  - Any other key cancels the leader and passes through untouched. The swallowed `\` is
    **not** replayed: outside text fields it has no use in Highlights, and replaying it
    would beep.
- The state machine never touches keyUp events.

### Close runner

`M.run_close(mode, deps, on_done)`. `deps` is injected:

- `read()`: reads the focused Highlights window and returns
  `{ tabs = { {title, selected}, … } }`. It returns `{ tabs = nil }` when the window has
  no tab group (a single tab), or `nil, reason` when Highlights or its window cannot be
  read.
- `select(index)` returns `ok`.
- `close_selected()` presses the focused window's `AXCloseButton` and returns `ok`.
- `has_sheet()` returns a boolean.
- `after(seconds, fn)` and `now()` are the scheduler and clock.

`on_done(result)` receives
`{ status = "done" | "nothing" | "needs_attention" | "failed", closed = n, reason = string? }`.

The algorithm is asynchronous, with each poll scheduled through `after`. It never blocks
Hammerspoon's main thread.

1. Make an initial `read()`. Fewer than two tabs gives `nothing`. Otherwise let `k` be
   the selected index (1-based), `kept_title = tabs[k].title`, and `n = #tabs`. Set
   `left_remaining` to `k-1` for left/others (0 for right) and `right_remaining` to
   `n-k` for right/others (0 for left). If both are 0, the result is `nothing`. Cap the
   number of iterations at the initial planned count.
2. Each step starts with a fresh `read()`. Check that
   `tabs[kept_index].title == kept_title`. If it does not match, the tabs changed under
   you: stop with `failed`. Matching by position keeps duplicate titles safe. Pick the
   target: index `1` while `left_remaining > 0` (do the left side first), else
   `kept_index + 1` while `right_remaining > 0`, else finish.
3. **Select.** If the target is not already selected, call `select(target)`. Then poll
   `read()` until the target shows as selected, within `SELECT_TIMEOUT_SECONDS`. If it
   never does, stop with `failed`.
4. **Close.** Call `close_selected()` once. Poll until `read()` shows the tab count
   dropped by exactly one; a `tabs = nil` read counts as one tab. Then decrement the
   matching counter and `kept_index` (for a left close), increment `closed`, and go back
   to step 2. If `has_sheet()` becomes true, stop with `needs_attention` and leave the
   sheet up. If `CLOSE_TIMEOUT_SECONDS` passes, stop with `failed`, and never press
   close a second time for that tab.
5. **Finish.** Unless the result is `needs_attention`, reselect the kept tab with
   `select(kept_index)` if a tab group still exists and the kept tab is not selected.
   Then call `on_done`. A `tabs = nil` read once every planned close is done counts as
   success, not as drift.

### Alert copy

`M.message(mode, result)` returns one line for `hs.alert`:

- `done`: `Closed 1 tab to the right` / `Closed 3 tabs to the left` /
  `Closed 4 other tabs`.
- `nothing`: `No tabs to the left` / `No tabs to the right` / `No other tabs`.
- `needs_attention`: `Highlights needs you — answer the save dialog (closed N)`.
- `failed`: `Stopped after closing N tab(s): <reason>`.
- A separate busy line: `Still closing Highlights tabs`.

> [!decision] scope = app `run_close` also takes `scope`. Once the focused window's
> `others` run finishes, record the kept PDF's path from the focused window's
> `AXDocument`, read as in the Swift reader's `documentPath(from:)`. Then loop over a
> fresh app `AXWindows` list each time, skipping the window whose `AXDocument` is the
> kept path. On another window, press its `AXCloseButton` and poll until the total tab
> count across windows drops. This closes that window's visible tab with no select step,
> and closing its last tab closes the window. Apply the same sheet, timeout, and
> iteration-cap rules. Finally, raise the kept window (`AXRaise`) and count every close
> in the message (`Closed 6 other tabs`). The adapter gains `windows()` and
> `close_window(window)`, and the specs cover a two-window run.

> [!decision] reopen = backslash_u A pending leader plus `"u"` with no modifiers fires
> `reopen`. During a run, after step 3 confirms the target is selected, read the focused
> window's `AXDocument` path and add it to the run's list. When a run closes at least
> one tab, those paths replace `runtime.lastBatch` (in memory only, never logged). `\u`
> with a batch runs `hs.task.new("/usr/bin/open", nil, { "-b", BUNDLE_ID, path… })`,
> clears the batch, and shows `Reopened N tabs`; with no batch it shows
> `Nothing to reopen`. Add `\u` to the key table and the specs.

## Runtime module: `highlights_tab_keys.lua`

- Keep reload-safe state in a global table, `BobHighlightsTabKeys`, the same way
  `BobPingIndicator` does. `M.start(options)` first stops any previous tap, application
  watcher, and pending run timer. It also takes overrides (`hs`-free fakes) for tests.
- **Scoping.** Use `hs.application.watcher`. On `activated` for `BUNDLE_ID`, start the
  tap; if `tap:isEnabled()` is false, restart it, because macOS can disable taps. On
  `deactivated` or `terminated` for `BUNDLE_ID`, stop the tap and call
  `leader:cancel()`. On start, enable the tap right away if Highlights is already
  frontmost. Inside the callback, also confirm the frontmost bundle id; on a mismatch,
  cancel and pass the key through.
- **Tap.** `hs.eventtap.new({ keyDown }, cb)`. Wrap the body in `pcall`. Any internal
  error is logged with `hs.printf("Bob Highlights tab keys …")` and returns `false`; a
  bug must never eat a keystroke. A fired command is dispatched with `after(0, …)`, so
  the callback returns immediately. While a run is active, a fired command only shows
  the busy alert.
- **`can_arm`.** Read `hs.axuielement.systemWideElement()`'s `AXFocusedUIElement` with a
  0.25 s timeout (`:setTimeout`).
  - If the element's `:pid()` is not Highlights' pid, return false. This is the Bob Refs
    panel case: that panel is non-activating, so Highlights stays frontmost while you
    type in Bob Refs' search field.
  - If its `AXRole` is `AXTextField`, `AXTextArea`, or `AXComboBox`, return false.
  - If the element is unreadable, fall back to the Highlights application element's
    `AXFocusedUIElement` and apply the same role check. If that is unreadable too,
    return true (the PDF view).
- **AX adapter.** Implement `read`, `select`, `close_selected`, and `has_sheet` on the
  Highlights app element's `AXFocusedWindow`, re-resolving it on every call, with the
  tab group search, close ladder, and timeouts above. `has_sheet` checks the focused
  window and then each of the app's `AXWindows` for an `AXSheet` child.
- Results go to `hs.alert.show(M.message(...), 1.5)`. Never log PDF titles or paths.

## Tests

- `tests/hammerspoon/highlights_tabs_spec.lua` loads the pure module with `loadfile`.
  - **Leader.** Arming only on a bare `\` when `can_arm` is true, and `can_arm` never
    called for other keys. `\\`, `\<`, `\>` (shift set) fire the right commands. Escape
    cancels and is swallowed. Another key cancels and passes through. A cmd/ctrl/alt
    chord cancels and passes through. An autorepeat while pending is swallowed and fires
    nothing. An autorepeat `\` while idle does not arm. Expiry after 1 s: the second `\`
    re-arms instead of firing. `cancel()` works.
  - **Runner**, with a fake tab model, a fake clock, and a queue-driven `after`:
    - Each mode closes the right tabs in the right order and reselects the kept tab.
    - `nothing` for a single tab, a `tabs = nil` read, the leftmost tab with left, and
      the rightmost tab with right.
    - Duplicate titles.
    - A select timeout, and a close timeout that never presses close twice.
    - A sheet gives `needs_attention` and no reselect.
    - Drift gives `failed`.
    - The tab group vanishing after the last close gives `done`.
    - The iteration cap.
  - `M.message` copy for every status and mode.
- `tests/hammerspoon/highlights_tab_keys_spec.lua` uses a fake `hs` (follow the env
  style of `ping_indicator_spec.lua`):
  - The tap is enabled only while Highlights is active and stopped on deactivate or
    terminate; another app's activation leaves it off.
  - `\` is swallowed when the focus is Highlights' PDF view.
  - `\` passes through for an `AXTextField` or `AXTextArea` focus, and for a focus owned
    by another pid.
  - A full `\>` sequence presses the expected fake AX tab buttons and close button.
  - A callback error returns `false` and is logged.
  - Calling `start()` twice stops the previous tap, watcher, and timers.
  - The tab-group search respects depth 3 and skips the excluded roles.
- `tests/hammerspoon/init_spec.lua`:
  - Stub `package.loaded.highlights_tab_keys` in `setup_hammerspoon_init_environment`,
    and restore it afterwards.
  - Assert that init calls `start` exactly once.
  - Assert that a throwing start is logged and the other hotkeys still install,
    mirroring the ping tests.

## Docs

Add `## Highlights tab keys` to the chezmoi `README.md`, after the Pomodoro section. It
covers:

- The key table.
- Active only while Highlights is frontmost.
- The focused window only.
- The 1 s leader timeout.
- `\` types normally in text fields and in the Bob Refs panel.
- A save dialog stops the run.
- These closes skip Bob Refs' ⇧⌘T stack.
- A two-line "why Hammerspoon, not Bob Mac Capture" note.

Keep it prettier-clean (`--prose-wrap=always --print-width=88`).

## Verification

In the chezmoi checkout:

- `just test-hammerspoon`
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh`
- `stylua --check home/dot_hammerspoon tests/hammerspoon`
- `just lint-md`

All must pass. After the commit lands, run `chezmoi update -a --force`, as the chezmoi
`AGENTS.md` requires.

The Mac (`ssh mac`) is best effort. If it is online, run `chezmoi update -a --force`
there, which restarts Hammerspoon, and confirm in the Hammerspoon console that the
module started without errors. Pressing the keys needs Bryan. If the Mac is offline, or
the manual check could not run, say in the final report that the live Highlights
behaviour is unverified, and list this check for Bryan:

- With 5 tabs, `\>` from tab 2, `\<` from tab 3, and `\\` from the middle each close the
  right tabs and leave the kept tab selected.
- `\` types normally in a Highlights note or search field and in Bob Refs' search field.
- `\` then `j` scrolls normally.
- A tab with unsaved highlights stops the run at its save sheet.

## Out of scope

- No bob-mac-capture changes.
- No ⇧⌘T integration.
- No configurable keys.
- No other apps.
