---
tier: tale
title: Highlights Ctrl-D/U/E/Y scroll keys in Hammerspoon
goal:
  In Highlights, ⌃D/⌃U scroll the PDF half a screen and ⌃E/⌃Y scroll about a line,
  holdable, via chezmoi Hammerspoon.
size: medium
decisions:
  line_step:
    ask: Should ⌃E / ⌃Y move about one line of body text per press?
    choices:
      one_line: 20 points per press, about one body-text line at fit-width zoom
      two_lines: 40 points per press, about two lines; twice as fast when held
    default: one_line
    why:
      You asked for line-by-line, and your 30 ms key repeat already makes a held
      20-point step fast
    answer: one_line
proposed_by: bbugyi200.athena.1b6
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 06:46:39
status: wip
---

# Highlights `⌃D` / `⌃U` / `⌃E` / `⌃Y` scroll keys (Hammerspoon)

## Goal

While a PDF is in front in Highlights (`net.highlightsapp.universal`), add Vim-style
scroll keys:

| Keys | Action                                         |
| ---- | ---------------------------------------------- |
| `⌃D` | Scroll down half the visible PDF height        |
| `⌃U` | Scroll up half the visible PDF height          |
| `⌃E` | Scroll down one line step (`line_step` points) |
| `⌃Y` | Scroll up one line step                        |

Holding a key keeps scrolling. The keys use the same gating as the existing page keys
(`gg`, `G`, `<N>g`, `⌃O`, `⌃I`): they type normally in text fields, in the Bob Refs
panel (another process), and in windows without a PDF.

## Where the work lives

All edits are in the **`chezmoi` linked repo**. Open it with `/sase_repo`
(`sase repo open chezmoi -r "<reason>"`) and work only in the path it prints. Read its
`AGENTS.md` first. Files:

- `home/dot_hammerspoon/highlights_pages.lua`: the pure (hs-free) page-key state
  machine, runner, and alert copy. Add the scroll key parsing and the pure scroll math.
- `home/dot_hammerspoon/highlights_tab_keys.lua`: the hs glue (event tap, watcher, real
  AX adapter). Add the scroll adapter functions and the coalescing dispatcher.
- `tests/hammerspoon/highlights_pages_spec.lua` and
  `tests/hammerspoon/highlights_tab_keys_spec.lua`: busted specs.
- `README.md`: the `## Highlights page keys` section.

Do not add a new Lua module. If you do anyway, every Lua file under
`home/dot_hammerspoon/` needs a hash line in
`home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`, and
`tests/bash/hammerspoon_restart_hook_test.sh` enforces this. Editing the two existing
modules already changes their hashes, so a Mac apply restarts Hammerspoon.

## Findings that shape the design

These come from the earlier page-keys work (plan `202611/highlights_page_keys.md`) and
from a read-only probe of the live Mac on 2026-11-11:

- The PDF view is PDFKit. AX tree: `AXWindow` → `AXSplitGroup` → `AXScrollArea` →
  `AXGroup` (subrole `AXSharedDocumentContainer`) → `AXPage` children.
  `M.find_pdf_view(window)` in `highlights_tab_keys.lua` already returns
  `(area, document)`.
- The scroll area's vertical `AXScrollBar` has a settable `AXValue`. Its value is
  `offset / (documentHeight − viewportHeight)`, with 0 at the top. The probe confirmed
  this: area height 818, document height ≈ 40043, offset 4837 → 4837 / 39225 = 0.1233,
  and the bar read 0.12334. The existing `scroll_to` page jump already sets this value.
  Its helpers `vertical_scrollbar(area)`, `element_frame(el)`, and `current_pdf_view()`
  are local to `make_real_adapter`. Reuse them.
- The scroll area offers only the `AXScroll{Up,Down,Left,Right}ByPage` actions. Those
  move a full page, so they cannot do half-page or line steps. Do not use them.
- No Highlights menu item uses `⌃D`, `⌃U`, `⌃E`, or `⌃Y`. Every menu shortcut uses ⌘,
  except ⌥↑/⌥↓, ⌃⇥/⌃⇧⇥, and fn chords. Bob Mac Capture's in-Highlights keys (open, tabs,
  copy, code, link) use ⌘ or `⌃O` only, so nothing collides.
- At the probed zoom, a Letter page is 955 points wide (≈1.56×). One line of 10-pt body
  text is about 18–19 screen points, which is where the 20-point `line_step` comes from.
- Bryan's key repeat is `KeyRepeat = 2` (30 ms, about 33 repeats per second) with
  `InitialKeyRepeat = 15`. A held key can arrive faster than one AX round trip, so
  scroll requests must be coalesced, not queued one per key.
- Through System Events/JXA, element sizes wrap at 16 bits: the document height of 40043
  reads as −25493. Hammerspoon's `hs.axuielement` returns real CGSize doubles, so the
  runtime is unaffected. Do not trust osascript size readings when probing.

## Design

### 1. Key parsing (`highlights_pages.lua`, `new_keys` / `idle_command`)

- Add a scroll table keyed by the plain character: `d` → down/half, `u` → up/half, `e` →
  down/line, `y` → up/line.
- In `idle_command`, keep the existing `has_cmd_or_alt` rejection first. Then, when
  `key.ctrl` is set and `key.shift` is not, and `key_plain(key)` is in the scroll table,
  return
  `{ kind = "scroll", direction = "down"|"up", amount = "half"|"line", autorepeat = key.autorepeat == true }`.
  This check must come **before** the existing `key.autorepeat` early return, because
  scroll keys accept autorepeat and every other page key keeps rejecting it. `key_plain`
  already handles `getCharacters(true)` returning `d` for ⌃D.
- The existing pending-state code needs no change. A ctrl chord other than `⌃O` already
  cancels a pending `g` or count and re-handles the key from idle, so `g⌃D` and `5⌃E`
  drop the pending prefix and scroll once. Counts are not supported on scroll keys.
- As with every arming key, `can_arm` gates the scroll keys. When it returns false the
  key is not swallowed and passes through, so Control-D/E/Y keep their macOS
  text-editing meaning in text fields.
- Update the `new_keys` doc comment to mention the scroll keys.

### 2. Pure scroll math (`highlights_pages.lua`)

Add constants and functions. Keep them hs-free so busted can `loadfile` the module:

- `M.SCROLL_HALF_FRACTION = 0.5`
- `M.SCROLL_LINE_POINTS = 20`.

  > [!decision] line_step = two_lines Use `M.SCROLL_LINE_POINTS = 40`, and use 40 points
  > everywhere this plan says 20 (README, specs).

- `M.add_scroll(batch, command) -> batch`: accumulate signed step counts into
  `{ half = <int>, line = <int>, fresh = <bool> }`. Down is +1 and up is −1. A nil batch
  starts at zeros. `fresh` becomes true when any added command has `autorepeat ~= true`.
- `M.scroll(batch, deps) -> result`, synchronous:
  1. `geometry, reason = deps.scroll_geometry()`. Return
     `{ status = "failed", kind = "scroll", reason = reason or "couldn't find the PDF view" }`
     when geometry is not a table with numeric `value`, `view_h`, and `document_h`.
  2. `delta = batch.half * geometry.view_h * M.SCROLL_HALF_FRACTION + batch.line * M.SCROLL_LINE_POINTS`.
  3. `maximum = geometry.document_h − geometry.view_h`. If `maximum <= 0` or
     `delta == 0`, return `{ status = "edge", kind = "scroll" }`.
  4. `target = clamp(geometry.value + delta / maximum, 0, 1)`. If
     `|target − geometry.value| < 1e-6`, return `edge` (already at the top or bottom).
     Make no write.
  5. `ok, reason = deps.set_scroll_value(geometry, target)`, wrapped in `pcall`. Return
     `done` (with `from` and `to` values) on success. Otherwise return `failed` with the
     reason, defaulting to `"the PDF didn't scroll"`.
- `M.message(result)`: for `kind == "scroll"`, return nil for `done` and `edge` (quiet
  success, as with jumps). Return `"Couldn't scroll the PDF: <reason>"` for `failed`.
  Leave the other branches unchanged.

### 3. Real adapter (`highlights_tab_keys.lua`, `make_real_adapter`)

Add two functions and export both from the adapter table:

- `scroll_geometry()`: get `area, document = current_pdf_view()`. Return
  `nil, "couldn't find the PDF view"` when either is missing. Read the frames with
  `element_frame` and the bar with `vertical_scrollbar(area)`. Read
  `value = tonumber(ax_attr(bar, "AXValue"))`. If a frame, the bar, or the value is
  missing, return `nil, "couldn't read the scroll position"`. Otherwise return
  `{ value = value, view_h = area_frame.h, document_h = document_frame.h, bar = bar }`.
  `bar` is an opaque handle that the pure code only passes back.
- `set_scroll_value(geometry, value)`: call
  `geometry.bar:setAttributeValue("AXValue", value)` in `pcall`. A failed call or a nil
  result returns `false, "couldn't set the scroll bar"`; nil is how `scroll_to` already
  detects failure. Then read the bar's `AXValue` back. If it is still within `1e-6` of
  `geometry.value`, return `false, "the scroll bar didn't move"` so that a silently
  ignored write becomes visible. Otherwise return true.

Each flush finds the view again rather than caching AX elements, because a tab switch
replaces them. Coalescing (below) absorbs the AX latency.

### 4. Coalescing dispatcher (`highlights_tab_keys.lua`, `M.start` / `handle_key` / `M.stop`)

- In `handle_key`, when `page_outcome.command.kind == "scroll"`, call
  `queue_scroll(page_outcome.command)` instead of scheduling `dispatch_page`. Still
  return `page_outcome.swallow == true`.
- `queue_scroll(command)`:
  - If `runtime.running` (tab close) or `runtime.page_running` (page jump) is set, drop
    the command silently. The key is still swallowed, and no alert shows. A busy alert
    at 33 Hz would stack, and a scroll during a jump would break the jump's landing
    check.
  - Otherwise set
    `runtime.scroll_batch = HighlightsPages.add_scroll(runtime.scroll_batch, command)`.
    If `runtime.scroll_scheduled` is false, set it to true and call
    `after_fn(0, flush_scroll)`.
- `flush_scroll()`: clear `runtime.scroll_scheduled`, then take and clear
  `runtime.scroll_batch`. Return if the batch is nil or a tab close or page jump is
  running. Otherwise `pcall(HighlightsPages.scroll, batch, scroll_deps)` with
  `scroll_deps = { scroll_geometry = adapter.scroll_geometry, set_scroll_value = adapter.set_scroll_value }`.
  A raised error becomes a `failed` result. On `failed`:
  - Always `H.printf("Bob Highlights scroll keys failed: %s", reason)`. Never log
    document paths or titles; an existing spec asserts this for page keys.
  - Show `HighlightsPages.message(result)` only when `batch.fresh` is true. A failing
    held key then alerts once per fresh press instead of 33 times a second.
- In `M.stop()`, reset `runtime.scroll_batch = nil` and
  `runtime.scroll_scheduled = false`. `stop()` already stops pending timers. If the flag
  were left true, scrolling would never schedule again after a restart.
- Export `runtime._scroll_deps_for_tests = scroll_deps` next to the other `_for_tests`
  hooks.
- Scrolls never touch `runtime.history`. They are not jumps, so `⌃O` / `⌃I` ignore them,
  as in Vim.
- Update the module header comment to say the tap also handles scroll keys.

### 5. README (`## Highlights page keys`)

- Broaden the intro sentence so it covers scrolling as well as page jumps. Keep the
  heading text, because other docs may link to its anchor.
- Add the four keys, either to the existing table or in a short `### Scroll keys`
  subsection with its own table. Say that:
  - `⌃D` / `⌃U` scroll half the visible PDF height, and `⌃E` / `⌃Y` scroll about one
    line (20 points; 40 points with `line_step = two_lines`).
  - Holding a key keeps scrolling, and the top and bottom stop it quietly.
  - Scrolls are not jumps, so `⌃O` does not return across them.
  - A count before a scroll key is ignored.
  - Scroll keys do nothing while a page jump or tab close is running.
  - They pass through in text fields, the Bob Refs panel, and windows without a PDF,
    where Control-D/E/Y keep their usual text-editing meaning.
- Format with `prettier --write --prose-wrap=always --print-width=88 README.md`.

## Tests

Use the existing fakes. In `highlights_tab_keys_spec.lua`, `make_pdf_view` has area
height 500 and page step 820, and its bar's `AXValue` getter and setter drive
`state.offset`. Its `ignore_bar` option makes the bar ignore writes. `env.tap_key`
payloads take `chars`, `plain`, `flags`, and `autorepeat`. Add a helper such as
`tap_ctrl(env, letter, opts)` that sends `chars = string.char(letter:byte() - 96)`,
`plain = letter`, `flags = { ctrl = true }`, and passes `autorepeat` through.

**`highlights_pages_spec.lua` (pure):**

- `⌃D`, `⌃U`, `⌃E`, and `⌃Y` produce the four scroll commands, with `swallow = true`,
  when `can_arm` is true.
- An autorepeat scroll key still produces a command with `autorepeat = true`, while an
  autorepeat `g`, `G`, `⌃O`, or digit still does not arm.
- Shift (⌃⇧D), cmd, or alt variants and `can_arm == false` pass through.
- A pending `g` followed by `⌃D` scrolls, and a following lone `g` does not complete
  `gg`. `5⌃E` yields one line step, and the count is gone afterward.
- `add_scroll` sums signed half and line counts, and sets `fresh` only for a
  non-autorepeat command.
- `scroll`:
  - one half step moves `view_h/2 / (document_h − view_h)`;
  - a line step moves `SCROLL_LINE_POINTS / maximum`;
  - mixed batches sum;
  - the result clamps to `[0, 1]`;
  - at the top, an up batch returns `edge` without calling `set_scroll_value`;
  - `document_h <= view_h` returns `edge`;
  - geometry failure and set failure return `failed` with their reasons;
  - a throwing dependency is handled.
- `message`: nil for scroll `done` and `edge`, and `Couldn't scroll the PDF: …` for
  `failed`.

**`highlights_tab_keys_spec.lua` (runtime, real adapter via `make_pdf_view`):**

- From page 10 (offset 7380), `⌃D` moves the offset by +250 and `⌃U` restores it. `⌃E`
  moves +20 and `⌃Y` restores it. No alerts and no logs.
- Coalescing: three `⌃E` taps (the first fresh, then two with autorepeat) before
  `env.drain()` produce exactly one bar write moving +60. Count writes by wrapping the
  bar's `setAttributeValue`. All three taps return true (swallowed).
- At offset 0, `⌃Y` and `⌃U` cause no write, no alert, and no log. At the bottom, `⌃D`
  does the same.
- These pass through: a text field focus role (`AXTextField`), another pid, and
  `no_document`. The tap returns false, and nothing scrolls.
- With `ignore_bar = true`, a fresh `⌃D` shows one
  `Couldn't scroll the PDF: the scroll bar didn't move` alert and logs once. A following
  autorepeat-only batch logs without another alert.
- `⌃D` while `runtime.page_running` is set (simulate with `_page_dispatch_for_tests` and
  an undrained queue, or set the flag directly) is swallowed and does not scroll.
- After scrolling, `⌃O` still says `No earlier page`, which shows that scrolls record no
  history.
- `M.start()` called twice, with a scroll queued but not drained in between, still
  scrolls after the restart. This shows that `stop()` reset `scroll_scheduled`.
- A pending `\` leader followed by `⌃U` cancels the leader and scrolls up.
- Ctrl-Tab stays unswallowed. The existing spec covers this; keep it passing.

## Verification

From the chezmoi repo root:

- `just test-hammerspoon` (busted). Baseline for the two Highlights specs today: 75
  successes.
- `stylua --check home/dot_hammerspoon tests/hammerspoon`
- `prettier --check --prose-wrap=always --print-width=88 README.md`
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh` (it must still pass, since no
  module is added).
- Show that the runtime specs exercise the real adapter. Temporarily make the real
  `set_scroll_value` return false, confirm the scroll specs fail, then restore it.
- Optional read-only live check (best-effort; `ssh mac` times out when the lid is
  closed): with `osascript -l JavaScript` against System Events, confirm that the
  focused Highlights window's PDF scroll area still exposes a vertical scroll bar with a
  numeric value. **Never set values, perform actions, or send keys on the Mac**, because
  Bryan's real documents are open there. Remember the 16-bit size wrap noted above.

Follow the chezmoi repo's after-commit rule (`chezmoi update -a --force` on the host
that commits). In the final report, tell Bryan to run `chezmoi update -a --force` on the
Mac; the restart hook reloads Hammerspoon. Then he should try, in Highlights:

- `⌃D` / `⌃U` / `⌃E` / `⌃Y` in a long PDF, and holding `⌃E`;
- `⌃Y` at the top of the PDF (expect nothing to happen);
- `⌃E` in the search field (expect end-of-line, not a scroll).

If an alert says `Couldn't scroll the PDF: the scroll bar didn't move`, the scroll-bar
write is being ignored. Report that so a follow-up can add a second mechanism, such as a
scroll-wheel event.

## Out of scope

- Counts on scroll keys (`5⌃E`), Vim's `'scroll'` option, and `⌃F` / `⌃B` full-page
  scrolling.
- Smooth or animated scrolling.
- Changes to Bob Mac Capture or Highlights itself.
