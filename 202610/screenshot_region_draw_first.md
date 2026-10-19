---
tier: tale
title: "Screenshot picker: plain drag draws a new box, ⌘-drag moves it"
goal:
  A plain corner-to-corner drag in the ⌃⌥⇧S screenshot picker draws a new capture box
  anywhere, ⌘-drag moves the box, corner handles still resize, and the remembered-box →
  Enter → macscrot flow is unchanged.
size: medium
proposed_by: bbugyi200.athena.research.50.linker.w0
proposed_with: claude/opus
create_time: 2026-10-19 12:53:00
status: wip
---

# Plan: Screenshot picker: plain drag draws a new box, ⌘-drag moves it

## Goal

Make the `⌃⌥⇧S` screenshot region picker easy to aim at a specific rectangle. A plain
click-drag from one corner of the target to the opposite corner draws a new capture box,
whether it starts inside or outside the remembered box and in any direction. Holding ⌘
when the mouse button goes down moves the existing box instead. Corner handles still
resize. Everything downstream of the picker (hotkey, `macscrot`, upload, clipboard,
remembered region) stays exactly as it is.

This implements the recommended solution in the research report
`research:202610/screenshot_region_draw_first_selection/screenshot_region_draw_first_selection.md`
(read it with `sase artifact read` if you want the full rationale). Bryan accepted every
requirement in it. This plan is self-contained: the behavior contract below is the
source of truth.

## Where the work happens

- All changes are in the **chezmoi** linked repo. Open it with your `/sase_repo` skill
  (`sase repo open chezmoi -r "<reason>"`), use only the printed path, and read that
  repo's `AGENTS.md` before editing. Make no changes to bob-cli.
- Files:
  - `home/dot_hammerspoon/screenshot_region.lua` (modify; the only production file)
  - `tests/hammerspoon/screenshot_region_spec.lua` (new)
- Do **not** touch `home/dot_hammerspoon/init.lua` (hotkey binding),
  `home/bin/executable_macscrot`, or
  `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`. Do not add a new
  Lua module: every Lua file under `home/dot_hammerspoon/` needs a hash line in the
  restart hook, and `tests/bash/hammerspoon_restart_hook_test.sh` enforces that.
- Baseline: chezmoi `d9084960`. The picker was last changed in `4375716e`.
  `busted ./tests/hammerspoon` passes (150 successes). `tests/hammerspoon/init_spec.lua`
  stubs out the whole picker, so nothing exercises the gestures today.

## How the picker works today

`M.pick(onConfirm)` opens a full-screen `hs.canvas` (at `screenSaver` level) on one
screen. It draws a dimmed surround, a blue box, four 10×10 corner handles, a `W x H`
label, and a one-line hint (`Drag to adjust | Enter to capture | Esc to cancel`). It
preloads the last confirmed region (`hs.settings` key `macscrotLastRegion`), or a
centered box at 60% of the screen. One `hs.eventtap` swallows `leftMouseDown`,
`leftMouseDragged`, `leftMouseUp`, and `keyDown`. The mouse-down router applies the
first rule that matches:

1. Outside the overlay's screen frame: close the picker (`finish(false)`).
2. Within ±10 pt of a corner (`handleAt`, `half = HANDLE_SIZE`): resize from that
   corner.
3. Inside the box: move the box.
4. Otherwise: draw a new box. `region` is replaced by a 16×16 stub immediately on
   mouse-down. A release with less than 3 pt of movement closes the picker.

Enter confirms (saves the region, then calls `onConfirm` after 80 ms). Esc cancels. The
pain point: the target is usually inside the remembered box, so the natural
corner-to-corner drag moves the box instead of drawing a new one.

## Behavior contract (implement exactly this)

Mouse-down routing, first match wins. The mode is chosen at mouse-down and **locked
until mouse-up**:

| #   | Press                                     | Mode                                    |
| --- | ----------------------------------------- | --------------------------------------- |
| 1   | Outside the overlay's screen frame        | Close the picker (unchanged)            |
| 2   | `event:getFlags().cmd` is true, anywhere  | **Move** (wins over handles)            |
| 3   | Within ±6 pt of a corner (`handleAt`)     | **Resize** from that corner             |
| 4   | Anywhere else (inside or outside the box) | **Draw** a new box from the press point |

Rules:

- **R1 Draw.** The new box spans the press point to the current (later the release)
  point, normalized so any drag direction works. The point is clamped to the screen
  frame, and each side is raised to the 16 pt `MIN_SIZE` as `updateNewRegion` does
  today.
- **R2 Handles.** A plain press on a corner handle resizes from that corner. The hit
  zone shrinks from ±10 pt to ±6 pt around the corner (inclusive). The visible handle
  stays 10×10 (±5 pt).
- **R3 Move.** ⌘ held at mouse-down moves the box by the drag delta from anywhere on the
  overlay's screen, including on a handle and outside the box. The box keeps its size
  and is clamped to the screen frame (`clampRegionToFrame`).
- **R4 Mode lock.** Only the mouse-down flags matter. Releasing ⌘ during a move keeps
  moving; pressing ⌘ during a draw keeps drawing. Only `cmd` is consulted; other
  modifiers are ignored.
- **R5 Clicks do nothing.** A press and release with no drag, anywhere on the overlay's
  screen, leaves the picker open and the box unchanged. Only Esc cancels. (A press on
  another screen still closes the picker, per row 1.)
- **R6 The old box survives until the drag is real.** A draw does not touch `region`
  until the pointer has moved at least 3 pt from the press point on either axis. On
  release, if the draw's extent (clamped release point minus press point) is under
  `MIN_SIZE` on **both** axes, restore the box that existed at mouse-down. A deliberate
  thin strip (long on one axis) is kept, raised to 16 pt on the short axis.
- **R7 Release never captures.** Mouse-up applies the release point (it can differ from
  the last drag event), redraws, and clears the drag. Enter (`return` or `padenter`) is
  swallowed and ignored while a drag is in progress. Otherwise Enter confirms exactly as
  today (save `macscrotLastRegion`, then `onConfirm` after `CAPTURE_DELAY`). Esc cancels
  at any time, including mid-drag, as today.
- **R8 Hint.** Replace the hint with two lines:
  `Drag: new box · ⌘-drag: move · Corners: resize` and `Enter: capture · Esc: cancel`
  (joined by `\n`). Make the hint frame taller and wider so neither line clips.
- **R9 Nothing downstream changes.** The hotkey, the `macscrot x,y,w,h` format in screen
  points, the upload, the clipboard path, saving the region only on Enter, the 80 ms
  capture delay, and preloading the remembered region all stay as they are.

## Implementation steps (`home/dot_hammerspoon/screenshot_region.lua`)

Keep the existing style: tabs, small local helpers, few comments. The routing should
change about 40 lines. Do not change the event-tap mask.

1. **Constants.** Add `HANDLE_HIT_RADIUS = 6` and `DRAG_THRESHOLD = 3` next to
   `HANDLE_SIZE`. Add `HINT_WIDTH`, `HINT_HEIGHT`, and `HINT_TEXT` for the hint (see
   step 6). `HANDLE_SIZE = 10` stays for drawing.
2. **`handleAt`.** Use `HANDLE_HIT_RADIUS` instead of `HANDLE_SIZE` for the hit square
   (`±6`, inclusive bounds as now). Iterate the corners in a fixed order (`nw`, `ne`,
   `sw`, `se`) instead of `pairs`, so results are deterministic. With ±6 zones and a 16
   pt minimum box, zones never overlap.
3. **Mouse-down.** Keep the off-screen `finish(false)` check first. Then route, without
   touching `region`:
   - `local flags = event:getFlags() or {}`
   - `flags.cmd` →
     `drag = { mode = "move", startPoint = point, startRegion = copyRect(region) }`
   - else `handleAt(point, region)` →
     `drag = { mode = "resize", handle = handle, startRegion = copyRect(region) }`
   - else →
     `drag = { mode = "new", anchor = point, moved = false, startRegion = copyRect(region) }`

   Remove the immediate `updateNewRegion(point, point)` and `redraw()` from the draw
   branch. A new mouse-down always replaces any stale `drag`.

4. **Shared drag step.** Add a local `applyDrag(point)` inside `pick` that both
   `leftMouseDragged` and `leftMouseUp` call:
   - `move`: the existing delta-plus-`clampRegionToFrame` logic.
   - `resize`: `resizeFromHandle(drag.startRegion, drag.handle, point, screenFrame)`.
   - `new`: set `drag.moved` once `|dx| >= DRAG_THRESHOLD or |dy| >= DRAG_THRESHOLD`
     (raw point vs. anchor, as today). Only when `drag.moved` is true, call
     `updateNewRegion(drag.anchor, point)`.

   `leftMouseDragged` with a `drag` calls `applyDrag(point)` and then `redraw()`.

5. **Mouse-up.** If `drag` is nil (for example a release with no press seen), swallow
   and return. Otherwise call `applyDrag(point)`. For a `new` drag, restore
   `region = copyRect(drag.startRegion)` when it never moved or when the clamped release
   extent is under `MIN_SIZE` on both axes (R6). Then `redraw()` and set `drag = nil`.
   Delete the old `finish(false)` on an unmoved draw (R5). Never call `finish` from
   mouse-up (R7).
6. **Enter guard and hint.** In the `keyDown` branch, if the key is `return` or
   `padenter` and `drag` is non-nil, return `true` without finishing. Esc is unchanged.
   In `buildElements`, use `HINT_TEXT` (the two lines from R8 joined by `\n`). Size the
   frame with `HINT_WIDTH = 360` (still capped at `screenFrame.w - 16`) and
   `HINT_HEIGHT = 42`. Place it at `bottom + 38` as now, and when it would overflow the
   bottom, fall back to `rect.y - (HINT_HEIGHT + 38)` so it clears the size label. Inset
   the text frame (for example `x + 8`, `y + 5`, `w - 16`, `h - 5`), and keep 12 pt
   `.AppleSystemUIFont`, center alignment, and `textLineBreak = "clip"`. Explicit
   newlines still break lines in clip mode. Write `·` and `⌘` as literal UTF-8 in the
   source.
7. **Comment.** One short comment above the mouse-down routing stating the precedence
   (off-screen closes; ⌘ moves from anywhere; handle resizes; otherwise draw) is enough.

## Tests (`tests/hammerspoon/screenshot_region_spec.lua`, new)

Test black-box through `M.pick` with a stubbed `hs`, following the `make_hs(env)` /
`setup()` / `env.restore()` pattern in `tests/hammerspoon/ping_indicator_spec.lua`. Load
a fresh module per test with
`assert(loadfile("home/dot_hammerspoon/screenshot_region.lua"))()` so the module-level
`activeSession` resets. Save and restore `_G.hs` around each test. The repo's `.busted`
runs specs under `nlua`.

**Stub surface that `pick` uses** (all methods are called with `:`):

- `hs.screen.allScreens()`, `hs.screen.mainScreen()`, `hs.mouse.getCurrentScreen()`:
  return a screen object whose `:frame()` returns a fresh copy of the test's frame.
- `hs.settings.get/set`: backed by a table, seeded with the test's saved region.
- `hs.canvas.new(frame)`: returns a canvas that records every `:replaceElements(...)`
  call and implements `level`, `behaviorAsLabels`, `clickActivating`, `show`,
  `bringToFront`, `hide`, and `delete` (record `deleted = true`).
- `hs.eventtap.new(types, callback)`: store the callback and return a tap whose
  `:start()` returns itself and whose `:stop()` records `stopped = true`.
  `hs.eventtap.event.types` maps `leftMouseDown`, `leftMouseDragged`, `leftMouseUp`, and
  `keyDown` to distinct values.
- `hs.keycodes.map`: `escape`, `["return"]`, and `padenter` keycodes.
- `hs.timer.doAfter(delay, fn)`: record the delay and run `fn` synchronously.
- `hs.notify.show`: record calls.
- Event factory: an object with `getType`, `location` (returns `{ x = , y = }`),
  `getFlags` (returns the flags table or `{}`), and `getKeyCode`.

**Helpers:** `press/drag/release(x, y, flags)`, `key(name)`, `rendered_region()`, and
`hint_text()`. `rendered_region()` reads the latest `replaceElements` call, finds the
element with `action == "stroke"` and `strokeWidth == 2`, and adds the screen-frame
origin back. `hint_text()` returns the multi-line text element's `text`. Confirm with
Enter and assert on the `onConfirm` argument and the saved `macscrotLastRegion` for
end-state checks. Assert that every event-tap callback returns `true`.

**Cases.** Unless noted, the screen is `{x=0, y=0, w=1000, h=800}` and the saved box is
`100,100,600,400`. "→ R" means that Enter confirms region R.

1. Plain drag inside the box `(200,200)→(350,280)` → `200,200,150,80`.
2. The same drag reversed `(350,280)→(200,200)` → `200,200,150,80`.
3. Plain drag outside the box `(750,550)→(900,700)` → `750,550,150,150`.
4. ⌘-drag inside `(200,200)→(250,240)` → `150,140,600,400`.
5. ⌘-drag starting outside the box `(800,700)→(850,740)` → `150,140,600,400`.
6. ⌘-move clamps at the edge: ⌘ `(200,200)→(900,200)` → `400,100,600,400`.
7. Tiny box `500,340,24,24`: ⌘-press on the NW corner `(500,340)→(530,360)` →
   `530,360,24,24` (moved, not collapsed). Contrast: the same press without ⌘ resizes NW
   → `508,348,16,16`.
8. Tiny box `500,340,24,24`: ⌘-press in the gap between handles `(512,352)→(532,372)` →
   `520,360,24,24`.
9. Plain SE-handle resize `(700,500)→(650,450)` → `100,100,550,350`.
10. Hit zone ±6: a press at `(104,104)→(150,160)` resizes NW → `150,160,550,340`. A
    press at `(108,108)→(150,160)` draws → `108,108,42,52`.
11. Click with no movement inside `(300,300)` and outside `(900,750)`: the picker stays
    open (tap not stopped, canvas not deleted, no `onConfirm`, no settings write) →
    Enter → `100,100,600,400`.
12. The old box stays until the threshold: press `(200,200)`, drag `(202,201)` →
    `rendered_region()` is still `100,100,600,400`. Then drag `(260,250)` → rendered
    `200,200,60,50`.
13. Accidental jitter `(200,200)→(205,206)` then release `(206,206)` → rendered and
    confirmed `100,100,600,400`.
14. Deliberate strip `(200,200)→(500,208)` → `200,200,300,16`.
15. Release point applied: press `(200,200)`, drag `(300,260)`, release `(350,280)` →
    `200,200,150,80`.
16. Mode lock: a plain press, then drag and release events carrying `{cmd=true}` → still
    draws `200,200,150,80`. A ⌘ press, then events with `{}` → still moves
    `150,140,600,400`.
17. Enter mid-drag: press `(200,200)`, drag `(350,280)`, Enter → no `onConfirm` and the
    picker stays open. Release, then Enter → `200,200,150,80`. Repeat once with
    `padenter`.
18. Esc (idle and mid-drag) → no `onConfirm`, settings unchanged, tap stopped, canvas
    deleted.
19. A press outside the overlay's screen frame `(1200,100)` still cancels (unchanged).
20. A stray mouse-up with no prior press → no error, region unchanged.
21. Negative-origin screen `{x=-1440, y=0, w=1440, h=900}`, saved box
    `-1300,100,400,300`: a draw `(-1300,200)→(-1600,400)` clamps → `-1440,200,140,200`.
    A ⌘-move `(-1100,300)→(-1600,-100)` clamps → `-1440,0,400,300`.
22. The hint text is exactly the two R8 lines joined by `\n`. The hint background (the
    rectangle element just before the hint text) is 42 pt tall and 360 pt wide on the
    1000×800 screen.

## Verification

From the chezmoi checkout:

- `just test-hammerspoon` (or `busted ./tests/hammerspoon`): all specs pass, including
  the new one and the untouched `init_spec.lua`.
- `just fmt-lua`, then `stylua --check home/dot_hammerspoon tests/hammerspoon` is clean.
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh` still passes. No new module was
  added, and the existing hash line already covers `screenshot_region.lua`.
- `git diff --stat` shows only the two files listed above.

## Commit and rollout

- Commit the chezmoi changes through the normal `/sase_final` repository obligation. A
  suggested subject is
  `feat(hammerspoon): draw-first screenshot region picker with ⌘-drag move`.
- chezmoi's `AGENTS.md` requires `chezmoi update -a --force` after commits land. On the
  Mac, a full `chezmoi update`/apply changes the restart hook's `screenshot_region.lua`
  hash, so Hammerspoon restarts with the new picker. A targeted
  `chezmoi apply ~/.hammerspoon` does not run the hook.
- The agent cannot test on the Mac. End the final response with this manual checklist
  for Bryan (mouse and trackpad):
  - draw inside the old box, and draw in reverse;
  - ⌘-drag inside and outside the box;
  - resize from a corner handle, and check that the ±6 pt zone feels right;
  - a stray click and a small jitter leave the box unchanged;
  - Enter mid-drag does nothing;
  - Esc, then hotkey → Enter recaptures the remembered box;
  - the PNG edges match the box and the overlay does not appear in the PNG;
  - both hint lines render unclipped on the smallest screen in use.

## Out of scope

Leave these out of this change. The research defers them until a trial shows a need:

- arrow-key nudging;
- edge handles;
- a ⌘-held outline color cue;
- ⌘ pressed mid-draw to reposition the box;
- ⇧ (constrain) and ⌥ (draw from center) modifiers;
- a zoom loupe, and window or accessibility snapping;
- capture on release, a second hotkey, and undo;
- capturing the menu bar or Dock (switching to `fullFrame()`);
- the multi-display "picker opens on the saved box's screen" fix;
- any change to `macscrot` or `#sshot`.

Do not file beads for these.
