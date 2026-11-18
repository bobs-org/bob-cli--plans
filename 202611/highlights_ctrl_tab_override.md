---
tier: tale
title: Reliable Highlights ctrl+tab tab switching via Hammerspoon
goal:
  In Highlights, ⌃⇥ and ⌃⇧⇥ switch to the next and previous native tab on every press,
  wherever focus is, while Bob Mac Capture panels keep their own ⌃⇥ panel switch.
size: medium
proposed_by: bbugyi200.apollo.ez
proposed_with: claude/opus
create_time: 2026-11-18 12:22:56
status: wip
---

# Plan: Hammerspoon owns ⌃⇥ / ⌃⇧⇥ tab switching in Highlights

## Goal

In the Highlights PDF app (`net.highlightsapp.universal`), `⌃⇥` (next tab) and `⌃⇧⇥`
(previous tab) often do nothing. Make them reliable by moving them into Bryan's existing
Hammerspoon Highlights event tap: the tap swallows the chord and selects the
neighbouring native window tab through Accessibility. AppKit focus routing then no
longer decides whether the switch happens.

All work is in the **`chezmoi`** linked repo. Open it with `/sase_repo`
(`sase repo open chezmoi -r "..."`) and read its `AGENTS.md` first. No bob-cli,
bob-plugins, or bob-mac-capture changes.

## What exists today (verified while planning)

- `home/dot_hammerspoon/highlights_tab_keys.lua` is the hs glue. It runs one `keyDown`
  event tap while Highlights is frontmost, which an `hs.application.watcher`
  starts/stops. It also holds a real AX adapter (`make_real_adapter`) whose `read()`
  returns `{ tabs = { {title, selected}, ... } }` from the focused window's
  `AXTabGroup.AXTabs`, or `{ tabs = nil }` when there is no tab group. Its `select(i)`
  AXPresses tab `i`. The `\` close leader (`highlights_tabs.lua`) and the
  `gg`/`G`/`⌃O`/`⌃I`/scroll page keys (`highlights_pages.lua`) are driven from
  `handle_key`.
- Today ⌃⇥ deliberately passes through the tap untouched. The spec test "leaves Ctrl-Tab
  alone so Highlights still switches tabs" in
  `tests/hammerspoon/highlights_tab_keys_spec.lua` asserts this, and README.md's
  "Highlights page keys" section says "`⌃I` is Control-I, so Highlights' `⌃⇥` tab
  switching still works."
- Live probe of the Mac (Highlights 2026.1.2): the window has native tabs (`AXTabGroup`
  whose `AXRadioButton` children carry `AXValue` true/false, plus a trailing `+`
  `AXButton`), and the Window menu has the native "Show Previous Tab" / "Show Next Tab"
  items. No `NSUserKeyEquivalents` override exists.
- **Bob Mac Capture uses ⌃⇥ / ⌃⇧⇥ itself.** Its Capture and Refs panels are
  `.nonactivatingPanel`s whose local monitors switch between the two panels on that
  chord (`CaptureKeyCommandRouter`). When one of those panels is key over Highlights,
  Highlights is still the frontmost app, so this tap sees the chord. It **must pass it
  through** there. The existing `\` leader already tells the two apart by the pid of the
  system-wide `AXFocusedUIElement`.
- Why the native shortcut fails is not pinned down. Likely causes: the focused PDF view,
  sidebar, or text view consumes ⌃⇥ before the menu's key equivalent sees it, or the
  menu item validates against a non-tab key window. Swallowing the chord at the session
  level and pressing the tab button makes the switch independent of the first responder,
  whatever the cause.

## Design

### Pure model, in `home/dot_hammerspoon/highlights_tabs.lua` (hs-free; busted loads it with `loadfile`)

Add:

- `M.CYCLE_CONFIRM_TIMEOUT_SECONDS = 0.5` and
  `M.CYCLE_MENU_PATHS = { [1] = { "Window", "Show Next Tab" }, [-1] = { "Window", "Show Previous Tab" } }`.
- `M.cycle_step(key)` returns `1` for ⌃⇥ and `-1` for ⌃⇧⇥, and `nil` for anything else.
  Match on `key.tab == true` (a keycode match set by the runtime, never on characters,
  because ⌃I and ⌃⇥ both produce `"\t"`) together with `key.ctrl` and with neither
  `key.cmd` nor `key.alt`. Autorepeat still returns a step.
- `M.cycle_target(count, selected, steps)` returns the 1-based wraparound index,
  `((selected - 1 + steps) % count) + 1`. It is `nil` for a bad count or selection.
- `M.cycle_message(result)` returns the alert copy:
  `"Couldn't switch Highlights tabs: <reason>"` for `failed`, and `nil` otherwise.
- `M.new_cycler(deps, on_result)` is a serialized, coalescing runner.
  `deps = { read, select, menu_step(direction) -> bool, after, now }`. Methods:
  - `push(step, fresh)`: add `step` to the pending steps and OR `fresh` (a non-repeat
    press) into the batch. If idle, schedule a flush with `deps.after(0, ...)`. **`push`
    never touches AX synchronously**, because it runs inside the tap callback.
  - The flush takes all pending steps as one batch, marks the cycler busy, and calls
    `deps.read()`:
    - `nil, err`: result `failed` with that reason.
    - `tabs == nil` (no tab bar): call `deps.menu_step(sign)` `|steps|` times. The
      status is `done` if any call returned truthy, else `nothing`. A disabled item on a
      single-tab window stays quiet.
    - No tab reads as selected: `failed` with "couldn't tell which tab is selected".
    - The target equals the selection (steps is a multiple of the count): `done`.
    - Otherwise call `deps.select(target)`, then poll `deps.read()` every
      `M.POLL_SECONDS` until `tabs[target].selected`, which is `done`. If
      `M.CYCLE_CONFIRM_TIMEOUT_SECONDS` passes first, the result is `failed` with
      "didn't switch to that tab" and any steps pushed meanwhile are dropped, so a stuck
      app cannot snowball.
    - After each batch, call `on_result({ status, reason?, fresh })`. Flush again if
      steps are pending; otherwise go idle and run the queued `when_idle` callbacks in
      order.
  - Basing each batch on a freshly confirmed selection means two quick presses move two
    tabs, and a held chord cycles at roughly key-repeat rate without flooding AX.
  - `is_busy()` is true while a batch is in flight or steps are pending.
  - `when_idle(fn)` calls `fn` right away when idle, and otherwise queues it.
  - `cancel()` drops the pending steps. An in-flight batch still finishes.
  - Guard every dep call with `pcall`, the same way `run_close` does.

### Runtime, in `home/dot_hammerspoon/highlights_tab_keys.lua`

- `build_key`: add `tab = keycode ~= nil and keycode == tab_code`, where `tab_code` is
  `H.keycodes.map.tab`, falling back to `48` (kVK_Tab).
- Add a `focus_owned_by_highlights(H)` predicate for the swallow decision. Read the pid
  of the system-wide `AXFocusedUIElement`, falling back to the pid of the system-wide
  `AXFocusedApplication`. Reuse the existing AX timeout and `pcall` style. Return true
  **only** when a pid was read and equals `highlights_pid(H)`. Unlike `can_arm`, treat
  an unreadable focus as "not ours", so the key passes through. Wrongly swallowing would
  break Bob Mac Capture's panel switch, while wrongly passing only restores today's
  native behaviour. Do **not** reject text-field roles: menu key equivalents beat text
  fields natively, so ⌃⇥ switches tabs from Highlights' search and note fields as well.
- In `make_real_adapter`, add `menu_step(direction)`: on the Highlights app object, call
  `app:selectMenuItem(HighlightsTabs.CYCLE_MENU_PATHS[direction])` under `pcall`, and
  return true only when the result is non-nil.
- In `M.start`, create
  `runtime.cycler = HighlightsTabs.new_cycler({ read, select, menu_step, after = after_fn, now = now_fn }, on_cycle_result)`.
  On a `failed` result, `on_cycle_result` always logs
  `H.printf("Bob Highlights tab cycle failed: %s", reason)`, and alerts
  `cycle_message(result)` only when `result.fresh`. Keep `options.adapter` as an
  injectable test seam (a fake adapter may lack `menu_step`, so default it to
  `function() return false end`).
- In `handle_key`, right after `build_key` and **before** the leader and page-key logic,
  when `HighlightsTabs.cycle_step(key)` is non-nil:
  1. Cancel any pending `\` leader and pending page-key sequence.
  2. If `not focus_owned_by_highlights(H)`, return `false` so the chord passes through.
  3. If `runtime.running` or `runtime.page_running` is set, return `true`. Unless the
     press is an autorepeat, also schedule the matching busy alert with
     `after_fn(0, ...)`: `HighlightsTabs.BUSY_MESSAGE` or
     `HighlightsPages.BUSY_MESSAGE`.
  4. Otherwise call `runtime.cycler:push(step, not key.autorepeat)` and return `true`.
- Ordering with the other commands. A switch must land before a close or jump reads the
  selected tab, so `\`-leader `dispatch(command)` and `dispatch_page(command)` go
  through `runtime.cycler:when_idle(...)` before their existing busy checks.
  `queue_scroll` and `flush_scroll` drop scrolls while `runtime.cycler:is_busy()`, the
  same as during a close or jump. The `bob-highlights-page` URL poll's
  `observation.busy` also counts `runtime.cycler:is_busy()`.
- `M.stop()` cancels and discards `runtime.cycler`; its timers are already stopped via
  `runtime.timers`. The watcher's deactivate/terminate branch calls
  `runtime.cycler:cancel()` next to the existing leader and page-key cancels.
- Expose `runtime._cycler_for_tests = runtime.cycler`, following the existing
  `_..._for_tests` pattern.

### README.md (repo root)

- In "## Highlights tab keys", add a short paragraph or table. `⌃⇥` selects the next tab
  and `⌃⇧⇥` the previous one, in the focused window, wrapping at the ends. Holding keeps
  cycling. Hammerspoon performs the switch, so it works wherever focus is inside
  Highlights. A Bob Capture or Refs panel keeps its own ⌃⇥ panel switch. With the tab
  bar hidden, the Window menu's Show Next/Previous Tab is used instead.
- In "## Highlights page keys", replace "`⌃I` is Control-I, so Highlights' `⌃⇥` tab
  switching still works." with a sentence saying `⌃I` is Control-I and does not
  interfere with `⌃⇥` tab switching.
- Keep prettier formatting: `--prose-wrap=always --print-width=88`.

## Tests

`tests/hammerspoon/highlights_tabs_spec.lua` (pure):

- `cycle_step`:
  - ⌃⇥ returns 1 and ⌃⇧⇥ returns -1. An autorepeat still returns a step.
  - Tab, ⇧⇥, ⌥⌃⇥, ⌘⌃⇥, and ⌃I (`tab` false, chars `"\t"`) return nil.
- `cycle_target` wraps in both directions and handles multi-step and multiple-of-count
  cases.
- Cycler, using a fake `read`/`select` whose selection lands after N polls:
  - One push selects the neighbour and confirms.
  - Two pushes before the confirm make two batches, or one batch of 2 when both arrive
    before the first flush; either way the final tab is two away.
  - With no tab group, the menu fallback gets the right direction and count. A false
    fallback gives `nothing`.
  - A confirm timeout gives `failed` and drops the steps pushed meanwhile.
  - `fresh` is false for an autorepeat-only batch.
  - `when_idle` runs immediately when idle, and otherwise after the batch.
  - `cancel` drops the pending steps.
  - `push` makes no `read` or `select` call synchronously.

`tests/hammerspoon/highlights_tab_keys_spec.lua` (runtime, existing fake-hs harness; add
`tab = 48` to the fake `hs.keycodes.map`, or rely on the 48 fallback):

- Replace "leaves Ctrl-Tab alone so Highlights still switches tabs" with the following.
  - ⌃⇥ (`keycode = 48, flags = { ctrl = true }`) is swallowed and, after `drain`,
    AXPresses the next tab through the real adapter. ⌃⇧⇥ presses the previous one. Both
    wrap at the ends.
- ⌃⇥ passes through and presses nothing when the focused element's pid is another
  process (Bob Mac Capture panel), and when focus is unreadable.
- ⌃⇥ is swallowed in a Highlights text field (`AXTextField` focus).
- ⌃I (keycode 34) still drives the page-forward jump, not a tab switch.
- ⌃⇥ cancels a pending `\` leader and a pending `g`.
- During a running close or page jump, ⌃⇥ is swallowed with the busy alert. An
  autorepeat press gets no alert.
- `\` `\` pressed right after ⌃⇥ waits for the switch to confirm, then keeps the newly
  selected tab.
- With no tab group, `selectMenuItem` is called with `{ "Window", "Show Next Tab" }`.
- A fresh failure alerts and logs, while an autorepeat failure only logs.
- Deactivation cancels pending steps, and `stop()` discards the cycler.

## Validation

Run from the chezmoi checkout (all tools are on PATH):

```bash
busted ./tests/hammerspoon
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
prettier --check --prose-wrap=always --print-width=88 README.md
bash tests/bash/hammerspoon_restart_hook_test.sh
```

No new Lua file is added, so
`home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl` needs no new hash
line. Its existing hashes of `highlights_tabs.lua` and `highlights_tab_keys.lua` already
restart Hammerspoon when the Mac next applies. Follow the chezmoi repo's `AGENTS.md`
apply rule after committing. Do not SSH into the Mac to apply or restart anything. Bryan
applies there.

## Manual check for Bryan (on the Mac, after `chezmoi update`)

1. Open four PDFs as tabs in one Highlights window. Press ⌃⇥ repeatedly with focus in
   the PDF view, then the sidebar, then the search field. Each press moves exactly one
   tab and wraps from last to first. ⌃⇧⇥ goes backwards. Holding ⌃⇥ cycles.
2. Open the Bob Capture or Refs panel over Highlights. ⌃⇥ switches between the panels
   and the Highlights tab does not change.
3. Press ⌃⇥ then quickly `\\`. The tabs other than the newly selected one close.
