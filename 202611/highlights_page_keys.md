---
tier: tale
title: Vim-style page keys for Highlights
goal:
  Add gg, G, <N>g, and ⌃O page-jump keys to Highlights through the chezmoi Hammerspoon
  config.
size: medium
decisions:
  ctrl_o_history:
    ask: Should repeated ⌃O keep walking back through older jumps, like Vim's jump list?
    choices:
      stack:
        A second ⌃O after 2g-then-⌃O goes to an older jump, or says "No earlier page"
      toggle: A second ⌃O returns to page 2, so repeated ⌃O flips between two pages
    default: stack
    why: It matches Vim's ⌃O, which this keymap borrows.
    answer: stack
  success_alert:
    ask: Should each successful jump also flash a "Page N of M" alert?
    choices:
      quiet:
        Successful jumps stay silent; only out-of-range, empty-history, and failed jumps
        alert
      alert: Every successful jump also shows "Page N of M" briefly
    default: quiet
    why: Highlights' window title already shows the page, and Vim jumps are silent.
    answer: quiet
proposed_by: bbugyi200.athena.17h
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 12:51:12
status: wip
---

# Plan: Vim-style page keys for Highlights (`gg`, `G`, `<N>g`, `⌃O`)

## Goal

While Highlights is frontmost and its PDF view has focus:

| Keys            | Action                                                 |
| --------------- | ------------------------------------------------------ |
| `gg`            | Jump to the first page                                 |
| `G`             | Jump to the last page                                  |
| `<N>g` / `<N>G` | Jump to page N (1 ≤ N ≤ page count)                    |
| `⌃O`            | Jump back to the page you were on before the last jump |

Example: on page 30, `2g` goes to page 2 and `⌃O` returns to page 30. `<N>G` is accepted
too. In Vim, `<N>G` means "go to line N", and without it a typed count followed by `G`
would be silently thrown away. N is the physical page number Highlights shows in its
title (`Page N of M`), not a printed page label such as `xii`.

All edits go in the linked **`chezmoi`** repo, next to the existing `\` tab-close
leader. Open it with `/sase_repo` (`sase repo open chezmoi -r "…"`) and read its
`AGENTS.md` first. Do not change bob-cli or bob-mac-capture. Read them only for
reference, if at all.

## Findings

These are read-only checks on the Mac (`ssh mac`, System Events/JXA, the Highlights app
bundle) made on 2026-11-03 against Highlights 2026.1.2 (`net.highlightsapp.universal`).

- **Current page and page count are in the window title.** The focused document window's
  `AXTitle` is `the_ai_native_sdlc_playbook.pdf – Page 10 of 44`, and an `AXStaticText`
  child carries the same `Page 10 of 44`. bob-mac-capture already parses this format
  (`HighlightsWindowTitle` in `Sources/RefsCore/RefsHighlightsTabs.swift`): en dash,
  hyphen, or em dash separator, case-insensitive `page N of M` at the end.
- **The PDF view is PDFKit, and every page is an AX element.** Tree: `AXWindow` →
  `AXSplitGroup` → `AXScrollArea` → `AXGroup` (subrole `AXSharedDocumentContainer`,
  description `document`) → 44 `AXPage` children in page order. Each child has a 0-based
  `AXPageIndex` (child 1 → 0, child 44 → 43), an `AXPosition`/`AXSize`/`AXFrame`, and
  one action, `AXScrollToVisible`. The sidebar's scroll area sits inside a separate
  `AXGroup` and has no `AXSharedDocumentContainer`.
- **The PDF scroll area's vertical `AXScrollBar` has a settable `AXValue`** (settable =
  true; it read 0.206 on page 10 of 44). The value is the scroll fraction
  `offset / (documentHeight − viewportHeight)`, with 0 at the top. The scroll area
  offers only `AXScrollUp/Down/Left/RightByPage` actions.
- **Highlights has no "Go to Page…" command and no AppleScript dictionary.** The Go menu
  has Previous Page (⌥↑), Next Page (⌥↓), Back (⌘[), and Forward (⌘]). Back and Forward
  walk PDFView's own history, which a scroll-bar jump does not record. So `⌃O` needs its
  own jump history.
- **No Highlights menu item uses a bare `g`, `G`, a digit, or `⌃O`.** Every menu
  shortcut uses ⌘ except ⌥↑/⌥↓, ⌃⇥/⌃⇧⇥, and fn chords. View › Single Page / Two Pages /
  Two Pages – First Single are all _continuous_ layouts, so there is no page-at-a-time
  mode.
- **`highlights://<name>#page=N` links exist but are a poor fit.** The binary looks
  bookmarks up by file name ("Get FileBookmark with name"), so PDFs with the same name
  are ambiguous and a jump could open the wrong document. Do not use them.
- **Possible `⌃O` collision.** Bob Mac Capture's setting "In Highlights, open Bob Refs
  with" can be Command-O (default), Control-O, or Off. Bryan's
  `org.bobs.bob-mac-capture` domain does not set `refsHighlightsOpenKey`, so it is at
  the Command-O default and `⌃O` is free today. The README must warn that the Control-O
  choice would collide.
- `hs.eventtap.event:getCharacters(true)` returns the key's characters with every
  modifier except Shift removed (`o` for ⌃O, `G` for ⇧G). The current `build_key` only
  reads `getCharacters()`, which gives `\x0f` for ⌃O.

What could not be confirmed without scrolling Bryan's live document: whether setting the
scroll bar's `AXValue` actually scrolls the PDF, how `AXScrollToVisible` aligns a page,
and how the title reads in two-page layouts. The design below checks every jump against
the title and the page geometry, and it falls back to a second mechanism when the first
misses.

## Design

### Module layout

- New **pure** module `home/dot_hammerspoon/highlights_pages.lua` (no `hs`; busted
  `loadfile`s it, like `highlights_tabs.lua`). It holds the key state machine, the title
  parser, jump-history helpers, the async jump runner, and alert copy.
- `home/dot_hammerspoon/highlights_tab_keys.lua` stays the **single** runtime module and
  keeps its **one** event tap and app watcher. It routes each keyDown to the existing
  `\` leader or to the new page keys, and its real adapter gains the page AX functions.
  One tap avoids ordering races between two taps over the same keys. The tap also
  already owns the frontmost check, `can_arm`, the deactivate cancel, and `stop()`. Keep
  the file name so `init.lua`, its spec, and the restart hook do not churn. Update the
  module's header comment to say it wires both key sets.

### Key state machine (`HighlightsPages.new_keys({ timeout })`)

Keys arrive as the table `build_key` returns: `chars`, `escape`, `cmd`, `alt`, `ctrl`,
`shift`, and `autorepeat`, plus a new `plain` field (characters without non-Shift
modifiers; see the runtime changes). Call `on_key(key, now, can_arm)`. It returns
`{ swallow = bool, command = nil | { kind = "goto", target = "first" | "last" | <int> } | { kind = "back" } }`.
Also expose `cancel()` and `is_pending()`. `KEY_TIMEOUT_SECONDS = 1.0`, measured from
the last swallowed key, so a slowly typed `123g` still works. `MAX_COUNT_DIGITS = 6`.

**Idle state.** Call `can_arm()` only for the four arming keys below, inside `pcall`. If
it returns false or errors, pass the key through. An arming key must also be
non-autorepeat with no ⌘ or ⌥.

- `g` (no Shift, no ⌃) → pending-`g`, swallow.
- `G` (no ⌃) → `goto last`, swallow.
- `1`–`9` (no Shift, no ⌃) → pending count, swallow. A bare `0` is not an arming key and
  passes through.
- ⌃O (`ctrl` and `plain == "o"`, no Shift) → `back`, swallow.
- Any other key passes through, including `\`.

**Pending state** (pending-`g` or a count):

- If the key arrives after the timeout, reset to idle and handle the key as idle.
- An autorepeat key is swallowed and changes nothing.
- Escape cancels the pending sequence and is swallowed.
- In pending-`g`, `g` → `goto first`.
- In a count, `0`–`9` appends a digit and restarts the timer. A digit past
  `MAX_COUNT_DIGITS` cancels the count and is swallowed.
- In a count, `g` or `G` → `goto <count>`.
- Any other key, including ⌘/⌥/⌃ chords and `\`, cancels the pending sequence and is
  then handled as idle. So `\` reaches the tab leader, ⌃O still works, a digit after `g`
  starts a count, and unrelated keys pass through.

### Routing in `handle_key`

The frontmost check and the deactivate/terminate watcher now cancel **both** machines.
Each Highlights keyDown is routed like this:

1. If the tab leader is pending, give the key to it. If the leader swallows it or
   returns a command, use that outcome.
2. Otherwise, or if the pending leader declined, give the key to the page keys. Their
   `can_arm` is `can_arm_pages` (below). If they swallow the key or return a command,
   use that outcome.
3. If the tab leader was not offered the key in step 1, give it to the leader now; this
   is how a bare `\` arms. Otherwise pass the key through.

Commands are dispatched through `after_fn(0, …)`, as today.

`can_arm_pages = can_arm() and adapter.has_document()`. The existing `can_arm` handles
the focused pid and the text-field roles. It still passes when the focused element
cannot be read, which is fine for `\`. `has_document()` adds a cheap fail-closed guard:
the focused Highlights window must report an `AXDocument`. Letter and digit keys
therefore type normally in the search field, in notes, in the Bob Refs panel (another
pid), in Settings or About windows, and whenever AX can't read a document.

### Jump runner (`HighlightsPages.run(command, jumps, deps, on_done)`)

`deps` provides:

- `page_info() → { page, count, document } | nil, reason`
- `scroll_to(n) → bool` (primary mechanism)
- `scroll_to_visible(n) → bool` (fallback mechanism)
- `landed(n) → bool`
- `after(seconds, fn)` and `now()`
- `log(message)` (optional)

Wrap every dep call in `pcall`. `on_done` receives
`{ status, kind, page, count, from, via?, reason? }`, where `status` is
`done | out_of_range | empty | failed`.

1. Call `page_info()`. On failure, finish `failed` with reason
   `couldn't read Highlights' page number`.
2. Resolve the target: `first` → 1, `last` → `count`, an integer → itself. If it is
   outside `1..count`, finish `out_of_range` with no history change and no scroll.
3. **goto:** if target ≠ current page, `push_jump(jumps, document, current)`. Still jump
   when target == current, so `gg` while halfway down page 1 snaps back to the top of
   page 1. **back:** `pop_jump(jumps, document, current)`. It discards entries equal to
   the current page. If nothing is left, finish `empty`.

   > [!decision] ctrl_o_history = toggle After `pop_jump` returns page P, push the
   > current page before jumping to P. A second ⌃O then pops it and returns, so ⌃O
   > alternates between two pages. Update the README sentence and the history specs to
   > match.

4. Land the jump with up to two attempts, `scroll_to(target)` and then
   `scroll_to_visible(target)`. An attempt whose call returns false is skipped. After a
   call, poll every `POLL_SECONDS = 0.05` for up to `LAND_TIMEOUT_SECONDS = 0.75`. The
   jump has landed when `page_info().page == target` or `landed(target)`. If the
   fallback attempt is the one that lands, call `log` with a fixed line saying the
   scroll-bar jump missed and `AXScrollToVisible` landed. If neither lands, finish
   `failed` with the reason `couldn't find the PDF view` (both calls returned false) or
   `didn't reach page N`. If a **back** jump fails, push its entry back so it isn't
   lost.
5. Finish `done` with `via = "scroll_bar" | "scroll_to_visible"`.

### Jump history (plain data, reload-safe)

`runtime.jumps` is a plain table: `{ [document] = { page, page, … } }`. Like
`runtime.lastBatch`, it is kept across `start()` (`hs.reload`) but forgotten by a full
Hammerspoon restart. Pure helpers:

- `push_jump(jumps, doc, page)` skips a duplicate of the newest entry and keeps the
  newest `MAX_JUMPS = 100` entries per document.
- `pop_jump(jumps, doc, current) → page | nil` drops entries equal to `current` before
  returning one.

The key is the decoded `AXDocument` path, so every PDF or tab has its own history. Only
these keys record jumps. Clicking the table of contents, a link, or a thumbnail, and
plain scrolling, do not.

### Alert copy (`HighlightsPages.message(result)`)

- `out_of_range`: `No page 50 — this PDF has 44 pages` (`has 1 page` when the count is
  1).
- `empty`: `No earlier page`.
- Read failure: `Couldn't read Highlights' page number`.
- goto `failed`: `Couldn't jump to page N: <reason>`.
- back `failed`: `Couldn't go back to page N: <reason>`.
- Busy: `Still jumping pages`.
- `done`: returns nil, so no alert is shown.

> [!decision] success_alert = alert `done` returns `Page N of M`, and the runtime shows
> it through the existing `alert_fn`. Add it to the README and specs.

On `failed`, the runtime logs
`H.printf("Bob Highlights page keys %s failed: %s", kind, reason)` inside `pcall`.
Reasons are fixed strings. Never log PDF titles or paths; this follows the tab-keys
convention.

## Changes

1. **`home/dot_hammerspoon/highlights_pages.lua` (new, pure).** Contains everything in
   the Design section. `parse_title(title) → page, count | nil` mirrors
   bob-mac-capture's rule (a separator, then case-insensitive `page N of M` at the end).
   It also requires `1 ≤ page ≤ count`. A title such as `Page 3 of 5.pdf` with no
   caption must not match.
2. **`home/dot_hammerspoon/highlights_tab_keys.lua` (runtime).**
   - `require("highlights_pages")`. In `start()`, create `runtime.page_keys` and keep
     `runtime.jumps` if it is already a table. Cancel the page keys in `stop()`, in the
     frontmost miss, and on deactivate/terminate.
   - `build_key`: add `plain`, taken from `event:getCharacters(true)` when
     `getCharacters` is a function, else `event.plain`, else `chars`.
   - Real adapter additions. Reuse `ax_attr`, `focused_window()`, and `document()`.
     - `has_document()`: true when `document()` returns a path.
     - `page_info()`: parse the focused window's `AXTitle`; `document` is `document()`.
     - `find_pdf_view(window)`: search `AXChildren` to depth ≤ 3. Never descend into
       `AXScrollArea`, `AXWebArea`, `AXTextArea`, `AXTable`, or `AXOutline`. Return the
       first `AXScrollArea` that has a direct child with
       `AXSubrole == "AXSharedDocumentContainer"` whose first child's role is `AXPage`.
       Return `area, doc`.
     - `page_element(doc, n)`: fast path is `AXChildren[n]` when its
       `AXPageIndex == n − 1`; otherwise scan the children for that index.
     - Geometry: use `AXFrame` (`{x, y, w, h}`), or fall back to `AXPosition` +
       `AXSize`. Coordinates are top-left screen coordinates.
     - `scroll_to(n)`:
       - Find the bar through the area's `AXVerticalScrollBar`, or else a child
         `AXScrollBar` whose `AXOrientation` is `AXVerticalOrientation`.
       - Compute `max = doc.h − area.h`.
       - Compute `v`: 0 when `n == 1` or `max ≤ 0`; otherwise
         `clamp((page.y − doc.y) / max, 0, 1)`.
       - Call `bar:setAttributeValue("AXValue", v)` in `pcall`. Return true only when it
         neither errors nor returns nil.
     - `scroll_to_visible(n)`: call `page:performAction("AXScrollToVisible")` in
       `pcall`.
     - `landed(n)`: true when the page's frame covers the scroll area's vertical
       midpoint, or when the page lies entirely inside the scroll area. This covers
       two-page spreads and a short last page at the end of the scroll range.
   - `handle_key` routing and `can_arm_pages`, as designed above.
   - `dispatch_page(command)`:
     - Refuse with the tab `BUSY_MESSAGE` while a tab close runs. Refuse with
       `Still jumping pages` while a page jump runs. The tab `dispatch` also refuses
       with `Still jumping pages` while a jump runs.
     - Track the page run with its own `runtime.page_running` flag.
     - Build page `deps` from the adapter, `after_fn`, `now_fn`, and a `log` that calls
       `H.printf`.
     - Call `HighlightsPages.run`, then show `message(result)` when it is non-nil.
     - Expose `runtime._page_deps_for_tests`.
3. **`home/dot_hammerspoon/init.lua`.** Change only the comment above
   `HighlightsTabKeys.start()` so it mentions the page keys. Keep the log string
   `Bob Highlights tab keys start failed`, which `tests/hammerspoon/init_spec.lua`
   asserts.
4. **`home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`.** Add the
   `# highlights_pages.lua: {{ include "dot_hammerspoon/highlights_pages.lua" | sha256sum }}`
   hash line in sorted order. `tests/bash/hammerspoon_restart_hook_test.sh` requires a
   hash line for every Lua file.
5. **`tests/hammerspoon/highlights_pages_spec.lua` (new, pure).** Use table-driven cases
   where they fit.
   - **Keys:**
     - `gg`, `G`, `2g`, `12g`, `12G`.
     - A bare `0` passes through; `10g` works.
     - Escape cancels and is swallowed.
     - Timeout expiry re-handles the key as idle: `g`, wait over 1 s, then `g` stays
       pending.
     - Autorepeat never arms and is swallowed while pending.
     - ⌘/⌥ chords pass through and cancel a pending sequence.
     - `\` cancels a pending sequence and passes through.
     - `g` then `5` starts a count.
     - A 7th digit cancels the count.
     - ⌃O fires `back`; ⇧⌃O does not.
     - `can_arm` false or throwing → pass.
     - `can_arm` is not called for non-arming keys.
   - **`parse_title`:** en dash, hyphen, and em dash; mixed case; `Page 3 of 5.pdf`;
     page > count; empty or nil.
   - **History:**
     - push/pop order, per-document isolation, and duplicate skip.
     - The `MAX_JUMPS` cap.
     - `pop_jump` discards entries equal to the current page.
     - The chosen `ctrl_o_history` behaviour.
   - **Runner, with fake deps and a fake clock/queue:**
     - The primary attempt lands.
     - The primary returns false → the fallback lands and the log line is written.
     - The primary misses the deadline → the fallback lands.
     - Both miss → `failed` with `didn't reach page N`.
     - Both return false → `couldn't find the PDF view`.
     - `out_of_range` makes no `scroll_to` call and records no history.
     - `page_info` failure.
     - `gg` on page 1 still scrolls and records no history.
     - `back` when history is empty → `empty`.
     - A failed `back` restores its history entry.
   - **`message`:** every status, including the 1-page plural.
6. **`tests/hammerspoon/highlights_tab_keys_spec.lua`.**
   - Add `setAttributeValue` to `make_ax_element`. A per-element setter can override it.
   - Add `plain` to fake key payloads.
   - Add a `make_pdf_view(env, opts)` fake, alongside `make_native_tabs`, that models
     the live tree from the Findings:
     - Window → split group → [sidebar `AXGroup` → `AXScrollArea` without a document
       container, PDF `AXScrollArea`].
     - The PDF scroll area holds the document container with `AXPage` children
       (`AXPageIndex`, frames derived from a scroll offset, page height, and gap) and a
       vertical scroll bar exposed both as a child and as `AXVerticalScrollBar`.
     - Setting the bar's `AXValue` moves the offset. `AXScrollToVisible` scrolls a
       page's top into view.
     - The window's `AXTitle` is `x.pdf – Page P of N`, where P is the page at the
       viewport's top edge. `AXDocument` is a `file://` URL.
     - Options can make the bar or `AXScrollToVisible` ignore requests.

   End-to-end specs use the **real adapter** (`start_with_real_adapter`) and drain the
   queue:
   - On page 10 of 44, `2g` sets the bar to page 2's top and the title reads page 2.
     `⌃O` then returns to page 10. With `stack`, a second `⌃O` alerts `No earlier page`.
   - `gg` lands on page 1 with the bar at 0. `G` lands on page 44.
   - `50g` alerts `No page 50 — this PDF has 44 pages`. Every key is swallowed and the
     bar is untouched.
   - When the bar ignores sets, `AXScrollToVisible` lands, and a log line without the
     title or path is written.
   - When both ignore requests, the alert is
     `Couldn't jump to page 2: didn't reach page 2`, plus a log line.
   - `g`, `5`, and ⌃O pass through (return false) when the focused element is an
     `AXTextField` or `AXTextArea`, when it belongs to another pid, and when the window
     has no `AXDocument`.
   - History is per document: after switching the window's `AXDocument`, `⌃O` alerts
     `No earlier page`.
   - Deactivating Highlights cancels a pending `g`.
   - The `\` leader still works:
     - Existing `\>` specs pass unchanged.
     - `g` then `\` then `>` closes the tabs to the right.
     - `\` then `g` within 1 s cancels the leader, and `g` arms the page keys.
   - A page command while a tab close is running shows the tab busy message.

7. **`README.md`.** Add a sibling `## Highlights page keys` section right after
   `## Highlights tab keys` (leave that section's text as is). Cover:
   - The key table above.
   - Each jump lands at the top of the page, and N is the page number in Highlights'
     title.
   - A count outside 1–M alerts and doesn't move.
   - Finish a sequence within 1 s of its previous key.
   - The keys type normally in text fields, the Bob Refs panel, and windows without a
     PDF.
   - `⌃O` history is per PDF, records only these keys' jumps, and lives in memory (a
     reload or restart may forget it).
   - Keep Bob Mac Capture's "In Highlights, open Bob Refs with" at Command-O or Off;
     Control-O would collide with `⌃O`.

   Wrap the prose at 88 columns (prettier `--prose-wrap=always --print-width=88`).

## Verification

From the chezmoi repo root:

- `just test-hammerspoon`, `bashunit tests/bash/hammerspoon_restart_hook_test.sh`,
  `stylua --check home/dot_hammerspoon tests/hammerspoon`, and
  `prettier --check --prose-wrap=always --print-width=88 README.md` must all pass.
- Prove the end-to-end specs exercise the real adapter. Temporarily make the real
  `scroll_to` return false, confirm the scroll-bar spec fails (or reports the fallback),
  then restore it.
- Optional read-only live check. It is best-effort, because `ssh mac` times out when the
  lid is closed. Using `osascript` against System Events on the focused Highlights
  window, confirm:
  - The PDF scroll area exposes `AXVerticalScrollBar`.
  - The document container's `AXPage` child count equals the title's page count.

  Do **not** set values, press buttons, perform actions, or send keys on the Mac.
  Bryan's real documents are open there.

- In the final report, tell Bryan to run `chezmoi update -a --force` on the Mac after
  the commit lands; the restart hook reloads Hammerspoon. Then he should try these in
  Highlights:
  - `2g` then `⌃O`.
  - `gg`, `G`, and `999g` (expect the out-of-range alert).
  - Typing `g` and digits in the search field.

  If a Hammerspoon console line says the scroll-bar jump missed, the fallback is
  carrying the jumps. Report that, so a follow-up can drop or fix the primary.

## Out of scope

- `⌃I` (jump forward again), counts on `⌃O`, and other Vim motions. With the default
  `stack` history, `⌃I` would need a jump list with a cursor; it can be a follow-up.
- Changes to Bob Mac Capture or its Control-O takeover option. Only the README warns
  about it.
- Persisting jump history across Hammerspoon restarts.
