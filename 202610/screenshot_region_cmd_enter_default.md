---
tier: tale
title: Screenshot picker saves its default box only on ⌘Enter
goal:
  In the ⌃⌥⇧S picker, Enter captures without changing the saved default box, and ⌘Enter
  captures and saves it.
size: small
decisions:
  fallback_box:
    ask:
      Should the built-in fallback box, shown only when no saved default fits any
      screen, become near-fullscreen?
    choices:
      keep:
        Leave the centered 60% fallback; Bryan sets his near-fullscreen default once
        with ⌘Enter
      near_full:
        Make the fallback a centered 90% box, so a fresh or off-screen default also
        opens near fullscreen
    default: keep
    why:
      The fallback is rare once a default is saved, so keep this change limited to the
      Enter keys
    answer: keep
proposed_by: bbugyi200.athena.research.50.linker.w0.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 15:43:11
status: wip
---

# Plan: Screenshot picker: Enter captures, ⌘Enter captures and saves the default box

## Goal

The `⌃⌥⇧S` screenshot region picker preloads a remembered box (the "default"). Today
every Enter both captures and overwrites that default, so one capture of a small region
replaces Bryan's usual near-fullscreen default. After this change:

- **Enter** captures the current box and leaves the saved default alone.
- **⌘Enter** does what Enter does today: captures the current box and saves it as the
  new default.

The next time the picker opens, it shows the saved default, not the last captured box.
Bryan normally keeps a near-fullscreen default and uses ⌘Enter only when he wants to
take several screenshots of the same spot.

## Where the work happens

- All changes are in the **chezmoi** linked repo. Open it with your `/sase_repo` skill
  (`sase repo open chezmoi -r "<reason>"`), use only the path it prints, and read that
  repo's `AGENTS.md` before editing. Make no changes to bob-cli.
- Files:
  - `home/dot_hammerspoon/screenshot_region.lua` (modify; the only behavior change)
  - `tests/hammerspoon/screenshot_region_spec.lua` (modify)
  - `home/dot_hammerspoon/init.lua` (a comment-only edit; see step 5)
- Do not touch `home/bin/executable_macscrot` or
  `home/.chezmoiscripts/run_onchange_after_restart_hammerspoon.tmpl`. Do not add a Lua
  module. The restart hook already hashes both edited Lua files.
- Baseline: chezmoi `efada113`
  (`feat(hammerspoon): draw-first screenshot region picker with ⌘-drag move`).
  `busted ./tests/hammerspoon` passes with 176 successes.

## How it works today

In `screenshot_region.lua`, `M.pick(onConfirm)` calls `loadInitialRegion()`, which
preloads the `hs.settings` key `macscrotLastRegion` when it fits a screen and otherwise
uses `defaultRegion(frame)` (a centered box at 60% of the screen). The event tap's
`keyDown` branch handles `return` and `padenter` like this: it swallows the key while a
mouse drag is in progress, and otherwise calls
`finish(region.w >= MIN_SIZE and region.h >= MIN_SIZE)`. `finish(confirm)` computes
`finalRegion = roundedRect(clampRegionToFrame(region, screenFrame))`, cleans up, and on
confirm always calls `hs.settings.set(SETTINGS_KEY, finalRegion)` before
`hs.timer.doAfter(CAPTURE_DELAY, ...)` calls `onConfirm(finalRegion)`. Esc calls
`finish(false)`. The keyDown flags are never read today.

## Behavior contract (implement exactly this)

- **K1 Enter captures only.** `return` or `padenter` with ⌘ not held captures
  `finalRegion` exactly as today (same rounding and clamping, same 80 ms delay, same
  `onConfirm` argument). It makes **no** `hs.settings.set` call.
- **K2 ⌘Enter captures and saves.** `return` or `padenter` with `event:getFlags().cmd`
  true saves `finalRegion` to `macscrotLastRegion`. Then it captures it exactly as K1
  does. The saved value and the `onConfirm` argument are the same rect.
- **K3 Only ⌘ matters.** Other modifiers are ignored, as they are for mouse gestures.
  ⇧Enter behaves like Enter, and ⌘⇧Enter behaves like ⌘Enter.
- **K4 Mid-drag guard unchanged.** While a mouse drag is in progress, Enter and ⌘Enter
  are both swallowed and do nothing. Esc still cancels at any time and never writes
  settings.
- **K5 Preload unchanged.** The picker still opens on the saved default (or the fallback
  box when none fits a screen). A plain-Enter capture is not remembered anywhere, in
  memory or on disk, so the next open shows the saved default again.
- **K6 Key name unchanged.** Keep the `macscrotLastRegion` settings key so Bryan's
  existing saved box carries over as the initial default.
- **K7 Hint.** The second hint line becomes
  `Enter: capture · ⌘Enter: capture + save default · Esc: cancel`. The first line is
  unchanged. Widen the hint so the longer line does not clip.
- **K8 Nothing else changes.** Mouse gestures, the hotkey, `macscrot`, the upload, the
  clipboard, and the 80 ms delay stay as they are.

## Implementation steps (`home/dot_hammerspoon/screenshot_region.lua`)

Keep the existing style: tabs, small local helpers, few comments.

1. **`finish(confirm, saveDefault)`.** Add the second parameter. On confirm, call
   `hs.settings.set(SETTINGS_KEY, finalRegion)` only when `saveDefault` is true. Leave
   the rest of `finish` as it is, so the save still happens before the `doAfter`
   scheduling. Esc and cancel paths keep calling `finish(false)`.
2. **Enter branch.** In the `keyDown` branch, keep the `if drag then return true end`
   guard first. Then read `local flags = event:getFlags() or {}` and call
   `finish(region.w >= MIN_SIZE and region.h >= MIN_SIZE, flags.cmd == true)`. Do not
   change the event-tap mask.
3. **Hint.** Set `HINT_TEXT` to
   `"Drag: new box · ⌘-drag: move · Corners: resize\nEnter: capture · ⌘Enter: capture + save default · Esc: cancel"`
   (literal UTF-8 `·` and `⌘`), and raise `HINT_WIDTH` from 360 to 420. `HINT_HEIGHT`
   stays 42, and the frame stays capped at `screenFrame.w - 16`. Leave the placement and
   the text-frame insets as they are.
4. **Fallback box (decision `fallback_box`).**

   > [!decision] fallback_box = keep Leave `defaultRegion` unchanged (centered, 60% of
   > each screen dimension).

   > [!decision] fallback_box = near_full Add `local FALLBACK_FRACTION = 0.9` next to
   > the other constants. Use it in place of both `0.6` literals in `defaultRegion`, so
   > the result is centered and still at least `MIN_SIZE` on each side. On the 1000×800
   > test screen, the fallback becomes `50,40,900,720`.

5. **Stale comment in `init.lua`.** The comment above the `⌃⌥⇧S` binding says the
   selector "preloads the last confirmed rectangle", which is no longer true. Reword it
   to say that the selector preloads the saved default rectangle, that ⌘Enter saves a
   new default, and that plain Enter captures without changing it. Keep the rest of the
   comment (macscrot owns capture, upload, clipboard, and notification). This is a
   comment-only change; no code changes.

## Tests (`tests/hammerspoon/screenshot_region_spec.lua`)

Extend the existing spec and keep its `make_hs(env)` / `setup()` / `env.restore()`
structure.

**Stub and helper changes:**

- Add `settingsWrites = 0` to the `env` table in `setup`, and increment it in the
  `hs.settings.set` stub. Seeding the saved region writes `env.settings` directly, so
  seeding does not count as a write.
- Change `key(env, name)` to `key(env, name, flags)` and pass `flags or {}` to `fire`.
- Update the `HINT_TEXT` constant to the K7 text.

**Existing cases to update:**

- Cases 1 and 2 currently assert that a plain Enter saved `200,200,150,80`. Change both
  to assert that `macscrotLastRegion` is still `100,100,600,400` and that
  `env.settingsWrites == 0`. Keep their `confirmed` assertions.
- Case 22: assert a hint background width of 420, still 42 tall, and rename the case to
  match. All other existing cases stay as they are; they already pass under K1 (case
  17's second round loads the unchanged `100,100,600,400` default, and its press at
  `(400,400)` still draws).

**New cases** (append them as 23–29 in the existing `describe`, or in a second
`describe` block). Unless noted, the screen is `{x=0, y=0, w=1000, h=800}` and the saved
box is `100,100,600,400`.

23. **Enter captures without saving.** Draw `(200,200)→(350,280)`, then Enter. The
    capture is `200,200,150,80` with a 0.08 delay, `settingsWrites == 0`, and the
    setting is still `100,100,600,400`. After `env.repick()`, `rendered_region` is
    `100,100,600,400`.
24. **⌘Enter captures and saves.** Same draw, then `key(env, "return", { cmd = true })`.
    The capture is `200,200,150,80` with a 0.08 delay, `settingsWrites == 1`, and the
    setting is `200,200,150,80`. After `env.repick()`, `rendered_region` is
    `200,200,150,80`. Then a plain Enter captures `200,200,150,80` again
    (`confirmed[2]`) and `settingsWrites` stays 1.
25. **⌘ with the keypad Enter also saves.** Draw `(300,300)→(500,450)`, then
    `key(env, "padenter", { cmd = true })`. The setting is `300,300,200,150`.
26. **Only ⌘ matters.** Draw `(200,200)→(350,280)`, then press Enter with
    `{ shift = true }`. That captures, and `settingsWrites == 0`. After `env.repick()`,
    draw `(200,200)→(350,280)` again, then press Enter with
    `{ cmd = true, shift = true }`. That captures and saves `200,200,150,80`.
27. **⌘Enter mid-drag does nothing.** Press `(200,200)`, drag `(350,280)`, then press
    ⌘Enter. The picker stays open (`assert_picker_open`) and `settingsWrites == 0`.
    Release at `(350,280)`, then ⌘Enter saves `200,200,150,80`.
28. **No saved default.** Use `open({ saved = false })`. A plain Enter captures the
    fallback box, and `macscrotLastRegion` stays nil. After `env.repick()`,
    `rendered_region` is still the fallback. Then ⌘Enter saves the fallback. The
    fallback is `200,160,600,480` with `fallback_box = keep`, or `50,40,900,720` with
    `fallback_box = near_full`.
29. **The saved rect equals the captured rect.** Draw `(200.4,200.6)→(350.2,280.7)`,
    then ⌘Enter. `confirmed[1]` and the saved setting are both `200,201,150,80`. This
    shows that ⌘Enter saves the rounded `finalRegion`, not the raw region.

## Verification

From the chezmoi checkout:

- `just test-hammerspoon` (or `busted ./tests/hammerspoon`): 0 failures and 0 errors.
  The count is the 176 baseline plus the new cases.
- `just fmt-lua`, then `stylua --check home/dot_hammerspoon tests/hammerspoon` is clean.
- `bashunit tests/bash/hammerspoon_restart_hook_test.sh` still passes.
- `git diff --stat` shows only the three files listed above.

## Commit and rollout

- Commit the chezmoi changes through the normal `/sase_final` repository obligation.
  Suggested subject: `feat(hammerspoon): save screenshot default region only on ⌘Enter`.
- chezmoi's `AGENTS.md` requires `chezmoi update -a --force` after the commit lands. On
  the Mac, the restart hook's `screenshot_region.lua` hash changes, so Hammerspoon
  restarts with the new picker.
- The agent cannot test on the Mac. End the final response with this checklist for
  Bryan:
  - The first open after rollout shows the last box the old Enter saved. Draw your usual
    near-fullscreen box and press ⌘Enter once to make it the default.
  - Draw a small box and press Enter. The next open shows the near-fullscreen default.
  - Draw a box and press ⌘Enter. The next open shows that box. Check that no other app
    or global shortcut takes ⌘↩ while the picker is open.
  - ⌘ with the keypad Enter also saves (if you have a keypad).
  - Enter and ⌘Enter mid-drag do nothing, and Esc never changes the default.
  - Both hint lines render unclipped on the smallest screen in use.

## Out of scope

Leave these out of this change, and do not file beads for them:

- a key that saves the default without capturing, or a key that resets it;
- remembering a plain-Enter box for the rest of the session;
- an on-screen cue showing whether the current box matches the saved default;
- renaming the `macscrotLastRegion` key or migrating its stored value;
- per-display defaults, and the multi-display "picker opens on the saved box's screen"
  fix;
- any change to `macscrot`, the hotkey, or the mouse gestures.
