---
tier: tale
title: Move the Bob Refs Copy card shortcut from ⌃Y to ⌘Y
goal:
  In Bob Refs, ⌘Y opens the Copy card and copies its selected row, ⌘K shows "Copy… ⌘Y",
  and ⌃Y no longer belongs to the card.
size: small
proposed_by: bbugyi200.apollo.7m
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.7m](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.7m.md)
- **COMMITS:**
  - [734a5b0](https://github.com/bobs-org/bob-mac-capture/commit/734a5b02d7099e74fcbc67ffbba8689a22eb2310)
    — feat(refs): move Copy card shortcut to Command-Y

# Plan: move the Bob Refs Copy card shortcut from ⌃Y to ⌘Y

## Context

Epic **bob-cli-7k** added the Bob Refs **Copy card** to Bob Mac Capture. Its contract
(`plan:202610/refs_ctrl_y_copy_card.md`; read it with
`sase artifact read plan:202610/refs_ctrl_y_copy_card.md "<why>"` if you need more than
this plan gives) binds the card to **⌃Y** in three places:

- the Refs panel opens the card;
- inside the card and its highlight picker, a second press copies the selected row;
- the ⌘K Actions menu shows "Copy…" with that shortcut.

Bryan wants **⌘Y** instead, everywhere. This tale is a pure rebind. Every other rule in
the epic contract stays as written, with ⌘Y in place of ⌃Y:

- With no available row selected, ⌘Y is still consumed and does nothing (no beep).
- ⌘Y still passes through while the search field holds IME marked text.
- ⌘Y with Caps Lock on still opens the card.
- **⌃Y is freed.** With the card closed, the router no longer claims it, so it reaches
  the search field like any other unbound Control key (Cocoa's standard `yank:`). With
  the card open, ⌃Y is just an unmapped key, so the card consumes it and does nothing.
  Do not keep ⌃Y as an alias.

All the code changes are in the linked repo **bob-mac-capture**. bob-cli, its docs, and
SASE memory never mention the shortcut, so they do not change. Open the repo with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), use only the path it
prints, and read its `AGENTS.md` before editing.

Nothing else claims ⌘Y. The app's main menu binds only ⌘H, ⌥⌘H, ⌘Q, and the Edit items.
The Highlights takeover hotkey is ⌘O or ⌃O. Bryan's Hammerspoon config binds only ⌃⌥⌘V
and ⌃⌥⇧S. The Refs local key monitor also runs before menu key equivalents, the same way
⌘K already works.

### Baseline and the in-flight landing fixes

At planning time, bob-mac-capture `origin/master` was `2b7b330`. The epic's landing-fix
tale (`plan:202610/refs_copy_card_landing_fixes.md`) had just closed, but its commit was
not yet on `origin/master`. Two of its fixes touch this area:

- **Fix 10** changes the router's `case KeyCode.y` to compare
  `flags.subtracting(.capsLock) == .control`, and adds a router test asserting that ⌃Y
  with Caps Lock routes to `.showCopyCard`.
- **Fix 4** changes how `RefsCopyKeyMap.command` handles keyCode 33 and private-use
  characters, near the ⌃Y line.

Fetch and start from the latest `master`. This plan describes behaviour, so it applies
whichever shape the code is in:

- If the landing fixes have landed, rebind their Caps Lock comparison and retarget their
  ⌃Y Caps Lock test to ⌘Y.
- If they have not, add the Caps Lock tolerance for ⌘Y yourself. Do not implement any
  other landing fix.

## Changes in bob-mac-capture

### 1. Panel router (`Sources/BobMacCapture/Refs/RefsKeyRouter.swift`)

- In `case KeyCode.y`, return `.showCopyCard` when the flags without Caps Lock are
  exactly `.command`. While `context.markedTextPresent` is true, return nil instead.
- Every other flag set returns nil. That includes ⌃Y, ⌘⇧Y, and ⌥⌘Y.
- `RefsPanelModel.perform(.showCopyCard)` already returns true even with no selection.
  Leave it alone, so ⌘Y is always consumed.

### 2. Card key map (`Sources/RefsCore/RefsCopy.swift`, `RefsCopyKeyMap.command`)

- Replace the keyCode 16 rule.
  - **Today:** normalized flags `.control` or `[.control, .option]` give
    `.copySelected`.
  - **Now:** normalized flags `.command` or `[.command, .option]` give `.copySelected`.
    The ⌥ variant keeps copying the displayed alternate, exactly as ⌥↵ does.
- ⌃Y needs no special case. It falls through to the existing modifier guard and returns
  `.consume`.
- This one key map serves both the card and the highlight picker, so the picker's "↵ and
  ⌘Y copy the selected row" follows automatically.

### 3. ⌘K Actions menu (`Sources/BobMacCapture/Refs/RefsActionsMenu.swift`)

- `RefsAction.copy.keyEquivalentModifierMask` becomes `.command`. The key equivalent
  stays `"y"`, so the menu shows "Copy… ⌘Y".
- Update the doc comment above `keyEquivalent` if it lists the shortcuts.

### 4. Footer (`Sources/BobMacCapture/Refs/RefsFooter.swift`)

- In the idle hint core, `⌃Y Copy` becomes `⌘Y Copy`.
- The `ViewThatFits` variants and their order are unchanged.

### 5. README (`README.md`)

- **Bob Refs key table:** the `Control-Y` row becomes
  `| Command-Y | Open the Copy card for the selected row |`. Keep it right after the
  `Command-K` row.
- **"### Copy card":**
  - "Control-Y opens an in-panel card…" becomes "Command-Y opens…".
  - "Return or Control-Y copies the selected row" becomes "Return or Command-Y copies…".
- **The paragraph after the key table:** add Command-Y to the keys that pass through
  while IME-composing, next to Return, Ctrl-D, and Ctrl-U.
- **⌘K paragraph:** "Copy… (⌃Y)" becomes "Copy… (⌘Y)".

### 6. Tests

- **`Tests/BobMacCaptureTests/RefsKeyRouterTests.swift`.** Rename
  `testControlYOpensCopyCardAndPassesThroughWhileComposing` to
  `testCommandYOpensCopyCardAndPassesThroughWhileComposing`. It asserts that:
  - ⌘Y gives `.showCopyCard`;
  - ⌘Y with Caps Lock gives `.showCopyCard`. If the landing-fix test for ⌃Y with Caps
    Lock exists, convert it into this assertion rather than keeping both.
  - plain `y`, ⌃Y, ⌘⇧Y, and ⌥⌘Y give nil;
  - ⌘Y with `markedTextPresent` gives nil.
- **`Tests/RefsCoreTests/RefsCopyTests.swift`,
  `testKeyMapNormalizesLettersAndRoutesControlKeys`.** Assert that:
  - `"y"`, keyCode 16, `[.command]` gives `.copySelected`;
  - `[.command, .capsLock]` gives `.copySelected`;
  - `[.control]` now gives `.consume`.
- **`Tests/RefsCoreTests/RefsCopyAlternatesTests.swift`.** The keyCode 16
  `[.control, .option]` assertion becomes `[.command, .option]`, which gives
  `.copySelected`.
- **`Tests/BobMacCaptureTests/RefsInspectorTests.swift`.** `RefsAction.copy`'s mask
  assertion becomes `.command`.
- **`Tests/BobMacCaptureTests/RefsFooterTests.swift`.** Both expected hint strings read
  `⌘Y Copy`.
- Leave every other test alone. `RefsPanelModelTests` drives `perform(.showCopyCard)`
  directly, so it is key-agnostic.

### 7. Acceptance sweep

When you are done, this search over `Sources`, `Tests`, and `README.md` finds no
remaining Copy card binding to Control-Y:

```bash
grep -rn -E '⌃Y|Control-Y|Ctrl-Y' Sources Tests README.md
```

Also check that no keyCode 16 rule still pairs with `.control`.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

- **RefsCore** (`RefsCopy.swift` and its tests).
  - Run `just portable-check` (swift-format lint, then build and test the portable
    targets) on athena, the tailnet host with Swift. Read
    `sase memory read tailnet.md -r "<why>"` for host access first.
  - Run it on a scratch copy, then delete the copy.
- **App and app tests** (`Sources/BobMacCapture*`, `Tests/BobMacCapture*`).
  - Rsync a scratch copy to the Mac without `.build` or `.git`. Run `Scripts/bundle.sh`,
    then `just typecheck-tests`.
  - If `typecheck-tests` fails with "unable to load standard library for target
    arm64-apple-macosx26.0" (task bead bob-cli-7o), type-check the test targets in the
    scratch copy only, passing `-sdk "$(/usr/bin/xcrun --sdk macosx --show-sdk-path)"`,
    as the epic's phases did. Do not commit any change to `Scripts/typecheck-tests.sh`;
    that is bob-cli-7o's scope.
  - Never use Bryan's checkout or `just install`. Delete the scratch copy afterwards.
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- Keep swift-format lint clean. A lint parse error skips every later CI step.
- `git diff --check` is clean.
- The four `RefsInspectorScrollTests` CI failures predate this work. Task bead
  bob-cli-7h tracks them, so do not touch them.

Commit the bob-mac-capture change with a message such as
`feat(refs): move the Copy card shortcut from ⌃Y to ⌘Y`.

## Bryan's Mac checks (manual, after install)

1. In Refs, ⌘Y opens the Copy card, and a second ⌘Y copies the default row.
2. ⌘K shows "Copy… ⌘Y".
3. ⌃Y in the search field no longer opens the card.

## Out of scope

- Editing the epic plan, the research report, or bob-cli-7k's title. They are historical
  records of the ⌃Y design.
- Any other Copy card behaviour, and the remaining landing fixes.
