---
tier: tale
size: small
title: Highlights ⌃I jumps forward through page history
goal:
  In Highlights, ⌃I walks forward again through the pages ⌃O walked back over, per PDF.
decisions:
  new_jump_forward:
    ask:
      After going back with ⌃O, should a new gg/G/<N>g jump discard the pages ahead of
      you?
    choices:
      discard: Clear the pages ahead, like browser history and Vim's jumpoptions=stack
      keep: Move them behind you, like Vim's default jump list, so ⌃O revisits them
    default: discard
    why:
      ⌃O/⌃I stay a plain back/forward stack; ⌃O never revisits pages you backed away
      from
    answer: discard
proposed_by: bbugyi200.athena.17h.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 13:31:59
status: wip
---

# Plan: `⌃I` jumps forward through Highlights page history

## Goal

Add `⌃I` (Control-I) to the Highlights page keys. It walks forward again through the
pages that `⌃O` walked back over, like Vim's `⌃I` and a browser's Forward button.

Example: on page 30, `2g` goes to page 2, `⌃O` returns to page 30, and `⌃I` goes back to
page 2. After `30 → 2g → 10g`, two `⌃O` presses go 10 → 2 → 30. Two `⌃I` presses then go
30 → 2 → 10, and a third `⌃I` alerts `No later page` and doesn't move.

All edits go in the linked **`chezmoi`** repo. Open it with `/sase_repo`
(`sase repo open chezmoi -r "…"`) and read its `AGENTS.md` first. This builds on the
page keys from `plan:202611/highlights_page_keys.md` (commit `846e1165`,
`feat(hammerspoon): add Highlights page navigation keys`). Do not change bob-cli or
bob-mac-capture.

## Findings

- **What ships today.** `home/dot_hammerspoon/highlights_pages.lua` is the pure (no
  `hs`) module with the key state machine (`new_keys`), `parse_title`, `push_jump` /
  `pop_jump`, the async runner `run(command, jumps, deps, on_done)`, and `message`.
  `home/dot_hammerspoon/highlights_tab_keys.lua` is the runtime: one event tap, the
  `build_key` key table (it already has `plain` from `event:getCharacters(true)`), the
  real AX adapter, `dispatch_page`, and `runtime.jumps`. `runtime.jumps` is
  `{ [document path] = { page, page, … } }`, a single back stack per PDF.
- **How `⌃O` works now.** A goto pushes the current page onto the back stack _before_
  scrolling. A back pops the newest entry that isn't the current page, and a failed back
  pushes that entry back. Nothing records where `⌃O` came from, so `⌃I` needs a second,
  forward stack.
- **`⌃I` vs Tab.** macOS sends ⌃I as keycode 34 (`kVK_ANSI_I`) with characters `"\t"`.
  `getCharacters(true)` ignores Control, so `plain == "i"`. The Tab key (keycode 48),
  including Highlights' own ⌃⇥ / ⌃⇧⇥ tab switching, has `plain == "\t"`. Matching `ctrl`
  plus `plain == "i"` therefore never catches Tab or ⌃⇥. This is the same rule `⌃O` uses
  with `plain == "o"`.
- **No collisions.**
  - Highlights' menu shortcuts (checked live for the page-keys plan) all use ⌘, except
    ⌥↑/⌥↓, ⌃⇥/⌃⇧⇥, and fn chords. None is ⌃I.
  - Bob Mac Capture's only `I` hotkey is the capture hotkey Control-Shift-Command-I
    (`HotKeyRegistry.production`). Its ⌘ makes the page keys pass it through. Its
    Highlights key choices (Command-O/E/T/Y/B/L, Control-O, Shift-Command-O) have no
    Control-I.
  - The chezmoi Hammerspoon `hs.hotkey.bind` calls use ⌘⌥⌃V and ⌃⌥⇧S. There's no
    Karabiner config, and the nvim config doesn't map `<C-i>`.

## Design

### Keys (`highlights_pages.lua`, `new_keys`)

- In idle, ⌃I is an arming key: `ctrl`, `plain == "i"`, no Shift, no ⌘ or ⌥, and not
  autorepeat. It returns `{ swallow = true, command = { kind = "forward" } }` after
  `can_arm` passes, exactly like ⌃O does for `back`. If `can_arm` is false or throws,
  the key passes through.
- In a pending `g` or count sequence, ⌃I cancels the sequence and fires `forward`.
  Today's pending-state chord rule already sends Control chords other than `o` through
  `cancel()` and `handle_idle`, so this needs no new branch. Cover it with a spec.
- `⇧⌃I`, `⌘⌃I`, `⌥⌃I`, an autorepeat ⌃I, Tab, and ⌃⇥ all pass through unchanged.

### History (two stacks per PDF)

Replace the single `jumps` table with a history object:

```lua
history = {
	back = { [document] = { page, … } },
	forward = { [document] = { page, … } },
}
```

Add `M.new_history()` to return `{ back = {}, forward = {} }`. Both sides reuse
`push_jump` / `pop_jump` unchanged: newest last, newest-duplicate skip, `MAX_JUMPS` cap,
and drop-equal-to-current on pop. Change `M.run` to
`M.run(command, history, deps, on_done)`. If `history`, `history.back`, or
`history.forward` is not a table, use a fresh empty table in its place, so a bad
argument can't crash the runner.

**A command changes history only once its jump lands (`done`).** The one exception is
the up-front pop of a back/forward target, which is restored on failure. This moves
goto's push from before the scroll to the `done` path. Today a failed goto still pushes
the current page; after this change it doesn't, and a failed goto must not discard the
forward stack.

| Command        | Target                                           | On `done` (landed)                                                                    | On `failed`                                  |
| -------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------- | -------------------------------------------- |
| goto           | `first` / `last` / N (range-checked as today)    | If target ≠ from page: push from page onto `back[doc]`, then apply `new_jump_forward` | No history change                            |
| back (`⌃O`)    | `pop_jump(back, doc, current)`; nil → `empty`    | Push from page onto `forward[doc]`                                                    | Push target back onto `back[doc]` (as today) |
| forward (`⌃I`) | `pop_jump(forward, doc, current)`; nil → `empty` | Push from page onto `back[doc]`                                                       | Push target back onto `forward[doc]`         |

A goto whose target is the current page (`gg` while partway down page 1) still scrolls.
It changes neither stack, so it does not clear the forward pages. An `out_of_range` goto
changes nothing either.

> [!decision] new_jump_forward = discard A landed goto with target ≠ from page sets
> `forward[doc] = nil`, after pushing the from page onto `back[doc]`.

> [!decision] new_jump_forward = keep A landed goto with target ≠ from page first moves
> the forward pages behind you, as Vim's default jump list does. Push each
> `forward[doc]` entry onto `back[doc]` with `push_jump`, nearest first (iterate the
> forward array from its last element to its first). Then push the from page and set
> `forward[doc] = nil`. Example: after `A → B → C → D` (back `{A, B, C}`), two `⌃O`
> presses leave you on B with back `{A}` and forward `{D, C}`. A new jump from B to E
> gives back `{A, C, D, B}`, so `⌃O` from E walks B, D, C, A. Add a pure spec for
> exactly this order.

Pages you reach by other means (scrolling, the table of contents, links, thumbnails)
don't touch either stack. `⌃I` after a manual scroll still pops the next forward page
and pushes the page you scrolled to onto `back`, so `⌃O` returns there. That matches
Vim, where cursor motion after `⌃O` doesn't drop the forward entries.

### Alert copy (`M.message`)

- `empty` with `kind == "forward"`: `No later page`. `empty` with any other kind stays
  `No earlier page`.
- `failed` with `kind == "forward"`: `Couldn't go forward to page N: <reason>`.
- Successful forward jumps stay quiet (`done` → nil), as decided for the page keys
  (`success_alert = quiet`).

## Changes

1. **`home/dot_hammerspoon/highlights_pages.lua`.**
   - Add the ⌃I idle command (`{ kind = "forward" }`) next to ⌃O in `idle_command`.
   - Add `M.new_history()`.
   - Change `M.run` to take `history` and implement the table above, including the
     `new_jump_forward` branch you chose. Add a small local helper for the "landed"
     history update so the `done` path stays readable.
   - Update `M.message` for the forward `empty` and `failed` copy.
   - Update the header and doc comments that say `Ctrl-O` only, e.g. "Create the `gg` /
     `G` / count / Ctrl-O key state machine", to mention Ctrl-I.
2. **`home/dot_hammerspoon/highlights_tab_keys.lua`.**
   - Replace `runtime.jumps` with `runtime.history`. In `start()`, keep the existing
     object only when it's a table whose `back` and `forward` are both tables. Otherwise
     use `HighlightsPages.new_history()`. This keeps the existing reload-survival
     behavior.
   - Pass `runtime.history` to `HighlightsPages.run` in `dispatch_page`.
   - Update the `stop()` doc comment ("Keeps `lastBatch` and jumps in memory") to say
     "jump history".
   - No adapter, routing, `build_key`, or `can_arm_pages` changes. `forward` goes
     through the same `dispatch_page`, busy checks, failure logging
     (`Bob Highlights page keys forward failed: …`), and quiet success as `back`.
3. **`tests/hammerspoon/highlights_pages_spec.lua`.** Update `run_fake` and every
   existing runner assertion from `state.jumps[...]` to `state.history.back[...]`; build
   state with `M.new_history()`. Then add:
   - **Keys:**
     - ⌃I (`chars = "\t"`, `plain = "i"`, `ctrl`) fires `forward`.
     - `⇧⌃I`, `⌘⌃I`, `⌥⌃I`, and an autorepeat ⌃I pass through.
     - Tab and ⌃⇥ (`plain = "\t"`) pass through, and `can_arm` is not called for them.
     - `g` then ⌃I, and `1` `2` then ⌃I, cancel the pending sequence and fire `forward`.
     - ⌃I passes when `can_arm` is false or throws.
   - **Runner:**
     - The walk `30 → 2g → 10g`, `⌃O`, `⌃O`, `⌃I`, `⌃I`, `⌃I` visits 10, 2, 30, 2, 10,
       and the last `⌃I` is `empty` / `forward`. Assert both stacks after each step.
     - A landed back pushes the from page onto `forward`. A landed forward pushes the
       from page onto `back`.
     - A new goto after `⌃O` applies the chosen `new_jump_forward` branch. For `keep`,
       include the `A, C, D, B` order spec.
     - `gg` on page 1 and an out-of-range goto leave `forward` intact.
     - A failed forward restores its entry and leaves `back` unchanged.
     - A failed back leaves `forward` unchanged.
     - A failed goto leaves both stacks unchanged.
     - The forward stack is per document.
   - **`message`:** `No later page`, and
     `Couldn't go forward to page 2: didn't reach page 2`.
4. **`tests/hammerspoon/highlights_tab_keys_spec.lua`.** These end-to-end specs use the
   real adapter (`start_with_real_adapter`, `make_pdf_view`). Add a `tap_ctrl_i(env)`
   helper next to `tap_ctrl_o`:
   `{ chars = "\t", plain = "i", flags = { ctrl = true }, keycode = 34 }`.
   - **Forward walk.** On page 10: `2g`, `⌃O` (back to 10), `⌃I` (page 2, no alert),
     then `⌃I` alerts `No later page` and the offset doesn't change.
   - **`new_jump_forward`.** On page 10: `2g`, `⌃O`, `gg`. Then `⌃I` alerts
     `No later page`.
     - With `discard`, `⌃O` returns to 10 and the next `⌃O` alerts `No earlier page`.
     - With `keep`, `⌃O` returns to 10 and the next `⌃O` goes to 2.
   - **Pass-through.** Extend the table-driven text-field / other-pid / no-`AXDocument`
     spec with `assert.is_false(tap_ctrl_i(env))`.
   - **⌃⇥ still switches tabs.** In the PDF view, ⌃⇥ (`chars = "\t"`, `plain = "\t"`,
     ctrl, keycode 48) returns false.
   - **Per-document isolation.** In the existing isolation spec, `⌃I` after switching
     `AXDocument` alerts `No later page`.
   - Existing `⌃O`, `\` leader, and busy specs must pass unchanged, apart from the
     `jumps` → `history` rename where they read runtime state.
5. **`README.md`, `## Highlights page keys`.**
   - Add a `⌃I` row to the key table: "Jump forward again after `⌃O`".
   - Extend the example paragraph: `⌃I` goes forward again through the pages `⌃O` walked
     back over, and shows `No later page` at the end.
   - Add one sentence for the chosen `new_jump_forward` branch:
     - `discard`: a new `gg`, `G`, or `<N>g` jump clears the pages ahead.
     - `keep`: they move behind you, so `⌃O` revisits them.
   - Add one short sentence that `⌃I` is Control-I, so Highlights' ⌃⇥ tab switching
     still works.
   - Leave the Bob Mac Capture Control-O warning as is.
   - Wrap at 88 columns (`prettier --prose-wrap=always --print-width=88`).

No changes to `init.lua` or the restart hook. The hook already hashes both Lua files, so
editing them is enough to restart Hammerspoon on apply.

## Verification

From the chezmoi repo root, all of these must pass:

- `just test-hammerspoon`
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh`
- `stylua --check home/dot_hammerspoon tests/hammerspoon`
- `prettier --check --prose-wrap=always --print-width=88 README.md`

Prove the new end-to-end specs exercise ⌃I. Temporarily change the idle ⌃I match to
`plain == "x"`, confirm the forward-walk spec fails, then restore it.

Do not send keys, set AX values, or perform actions on the Mac. Bryan's real documents
are open there.

In the final report, tell Bryan to run `chezmoi update -a --force` on the Mac after the
commit lands (the restart hook reloads Hammerspoon). Then he should try these in
Highlights:

- `2g`, `⌃O`, `⌃I` (back on page 2), then `⌃I` again (`No later page`).
- ⌃⇥ still switches tabs.
- If ⌃I does nothing or types into the PDF, report it. That would mean `plain` isn't
  `"i"` on his keyboard layout, and `⌃O` would be affected the same way.

## Out of scope

- Counts on `⌃O` / `⌃I` (e.g. `3⌃O`), and walking the history while ⌃O/⌃I auto-repeats.
- Persisting history across Hammerspoon restarts.
- Highlights' own Back/Forward (⌘[ / ⌘]), which follow PDFView's separate history.
