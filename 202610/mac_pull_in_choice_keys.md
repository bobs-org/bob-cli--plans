---
tier: tale
title: Ctrl-N/Ctrl-P navigation and dependable Space marking in the Mac successor choice
goal:
  In Bob Mac Capture's successor pull-in choice card, Ctrl-N/Ctrl-P move to the
  next/previous offer like Down/Up, and Space marks the highlighted offer from the
  moment the card opens.
size: small
decisions:
  empty_filter_space:
    ask:
      Should Space mark the highlighted offer even while the empty filter holds focus?
    choices:
      toggle: Space marks unless the focused filter already holds text
      type: Space types into a focused filter even when it is empty, as it does today
    default: toggle
    why:
      A leading space never filters usefully, and Space then works wherever focus lands
    answer: toggle
proposed_by: bbugyi200.apollo.7g
decided_by: auto
create_time: 2026-10-13 17:39:58
status: wip
---

# Plan: Ctrl-N/Ctrl-P and Space in the Mac successor pull-in choice

## Context

Epic bob-cli-7c (Successor Offers, `plan:202610/successor_offers.md`) added an in-panel
successor pull-in choice card to the linked repo **bob-mac-capture** (phase
bob-cli-7c.7, commits `395c521`, `b0e2ff2`, `6c4209b`). Return on a `!` or `=x` draft
whose preview has offers opens a grouped multi-select card. Choosing sends the picked
offer keys to `bob capture --pull-in`. Open the repo with `/sase_repo`
(`bob-mac-capture`) and read its `AGENTS.md` first.

What the code does today:

- **Ctrl-N/Ctrl-P are not handled.** `CaptureKeyCommandRouter.pullInChoiceCommand`
  (`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`) maps Return/keypad Enter,
  Escape, Ctrl-[, Up/Down, Space, and Cmd-A. Every other key returns `nil`. The pull-in
  branch runs before the picker and completion branches. So Ctrl-N/Ctrl-P fall through
  to AppKit and do nothing in the choice card. Every other list in the app supports
  them: the capture picker (`pickerCommand`: Ctrl-N/J next, Ctrl-P/K previous), the
  stash picker, and completions. README lines ~463, ~709, ~1323, and ~1602 document
  those.
- **Space works only while the pull-in filter does not hold focus.** The router returns
  `.togglePullInOffer` for unmodified, non-repeat Space only when
  `context.pullInFilterFocused` is false. Otherwise it returns `nil`, so Space types
  into the filter. The controller's local key monitor
  (`CapturePanelController.installKeyMonitorIfNeeded`) sets `pullInFilterFocused` from
  `CapturePickerFilterNSTextField.holdsFirstResponder` on the field with identifier
  `pullInFilterFieldAccessibilityIdentifier`.
  - `CapturePanelModel.openPullInChoice` requests `.pullInOfferList` focus. That is a
    SwiftUI `.focusable()` card containing the AppKit filter field, and the editor
    becomes `.disabled` at the same moment.
  - Nothing guarantees the filter field cannot end up as first responder after that
    handoff. The README (around line 1554) hedges: "Space toggles it after navigation
    moves focus to the offer list".
  - Navigation (`.nextPullInOffer` / `.previousPullInOffer`) already calls
    `resignPullInFilterIfFocused()` before it moves the highlight.
  - At open, the filter is always empty, because `filterText` starts as `""`.
- **The card's hint does not mention moving or marking.** It reads "Return choose ·
  Shift-Return choose none · Esc cancel" (`CapturePullInChoiceCard` in
  `Sources/BobMacCapture/CapturePanelView.swift`).
- No global hotkey uses Ctrl-N or Ctrl-P (`HotKeyRegistry.swift`). No memory note or
  bob-cli doc pins the choice card's keys: `docs/task-dependencies.md` covers only Esc
  and ⇧↩.

Related open beads that touch this card but do not overlap: bob-cli-7j (adds a
destination-retarget key to the same router branch) and bob-cli-76 (preview and
notification copy for project pull-ins). Keep the new bindings clear of the retarget-key
design space: use only Ctrl-N/Ctrl-P and Space.

## Changes (all in bob-mac-capture)

### 1. Router: Ctrl-N/Ctrl-P move the highlight

In `pullInChoiceCommand`, add:

- `KeyCode.n` with `modifiers == .control` → `.nextPullInOffer`.
- `KeyCode.p` with `modifiers == .control` → `.previousPullInOffer`.
- Any other modifier set on N/P → `nil`. Plain N/P must keep typing into a focused
  filter.

Do this whether or not the filter has focus, the same as Up/Down. The controller's
existing `.nextPullInOffer` / `.previousPullInOffer` handling already resigns the filter
and requests `.pullInOfferList` focus, so the controller needs no change for these keys.
Do not add a repeat guard: Up/Down allow repeats and Ctrl-N/Ctrl-P should match.
Ctrl-J/Ctrl-K are out of scope.

### 2. Router and context: Space marks whenever the filter is empty

> [!decision] empty_filter_space = toggle Add `var pullInFilterIsEmpty = true` to
> `CaptureKeyRoutingContext`. In the controller's key monitor, fill it from
> `model.pullInChoice?.filterText.isEmpty ?? true`. Pass it into `pullInChoiceCommand`
> beside `filterFocused`. For unmodified Space: return `.consumeKey` on a repeat.
> Otherwise return `.togglePullInOffer` when the filter is unfocused **or** empty, and
> `nil` (native typing) only when the filter is focused **and** non-empty. Leave first
> responder where it is when Space toggles from an empty focused filter: typed letters
> still filter, and Ctrl-N/Ctrl-P/arrows already move focus to the list. This makes
> Space work as the first key after the card opens, wherever AppKit put focus.

> [!decision] empty_filter_space = type Keep the Space routing rule as it is. Only
> reword the README and hint to say that Space marks once the offer list has focus.
> Navigation, including the new Ctrl-N/Ctrl-P, moves focus there.

### 3. Hint and accessibility copy

Update the hint line in `CapturePullInChoiceCard` to list moving and marking, for
example `Ctrl-N/P move · Space mark · Return choose · Shift-Return none · Esc cancel`,
and keep the existing caption style. Update its `accessibilityLabel` to match, for
example "Up, Down, Control-N, or Control-P moves the highlight. Space marks the
highlighted offer. Return chooses …". Keep the line readable at
`CapturePanelLayout.panelMinimumContentWidth`, where wrapping to two lines is
acceptable. `CapturePullInChoiceDesignTests` renders this card at that width.

### 4. README

Rewrite the successor-choice paragraph (around README lines 1551–1560):

- Up/Down **or Control-N/Control-P** moves the highlight, including from the filter.
- Under `toggle`: Space marks or unmarks the highlighted offer whenever the filter is
  empty or unfocused, including immediately after the card opens. The filter takes
  spaces only once it holds text.
- Under `type`: Space marks or unmarks once the offer list has focus, and navigation
  moves focus there.
- Drop the hedged "after navigation moves focus to the offer list" wording under
  `toggle`.

Keep the rest of the paragraph: Cmd-A, Shift-Return, Escape, and the content
description.

### 5. Tests

In `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift`
(`testRouterOwnsChoiceKeysAndLeavesFilterTypingNative`, or a sibling test), assert:

- Ctrl-N (key code 45) → `.nextPullInOffer` and Ctrl-P (35) → `.previousPullInOffer`,
  both with `pullInFilterFocused: false` and `true`.
- Ctrl-N with `isARepeat` still navigates.
- Ctrl-Shift-N, Cmd-N, and plain N/P → `nil` while the filter is focused.
- The pull-in branch still wins over `pickerVisible: true` and `completionVisible: true`
  for Ctrl-N/Ctrl-P. The completion branch would otherwise claim them.
- Under `toggle`, Space (49) with `pullInFilterFocused: true, pullInFilterIsEmpty: true`
  → `.togglePullInOffer`. With `pullInFilterIsEmpty: false` → `nil`. Repeat → still
  `.consumeKey`.
- The existing filter-focused Space assertion must set `pullInFilterIsEmpty: false` to
  keep its meaning.
- Under `type`: only the Ctrl-N/Ctrl-P assertions. The existing Space assertions stay.

The private `keyEvent` helper passes `""` characters for non-Space keys. Routing is by
key code, so it already works for N/P; pass `"n"`/`"p"` if you prefer realism. Extend
`testControllerDispatchMovesMarksAndCancelsOnlyTheChoice`, or add a sibling, with a
model-level sequence:

1. `perform(.previousPullInOffer)` then `perform(.nextPullInOffer)` moves and keeps
   `.pullInOfferList` focus.
2. `perform(.togglePullInOffer)` marks the highlighted key immediately after
   `installPullInChoiceForPreviews`, with no navigation first.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

1. Run `just portable-check` on a host with Swift. athena is a tailnet host with Swift;
   apollo has none. Use a scratch rsync copy, then delete it.
2. Compile on the mac, best effort: rsync a scratch copy without `.build`/`.git`, run
   `./Scripts/bundle.sh --identity -`, then
   `./Scripts/xcode-swift.sh build --build-tests`, then delete the copy. Never use
   Bryan's checkout or `just install`.
   - The mac's Command Line Tools cannot import XCTest (`no such module 'XCTest'`;
     bob-cli-7c.7 note #2), so only the app product build counts there.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified locally.
3. macOS CI (`.github/workflows/ci.yml`) is the test authority. Once the commit is
   published, monitor the run with `/sase_monitor`.
   - `CapturePullInChoiceTests`, `CapturePullInChoiceModelTests`, and
     `CapturePullInChoiceDesignTests` must pass, along with Lint Swift formatting and
     Build.
   - Known pre-existing `RefsInspectorScrollTests` failures (bob-cli-7h) skip the
     Bundle/launch/install gates. Report them, but they do not block this change.
   - After a red lint step, check the later steps too: a lint parse error skips Build
     and Test.
4. Manual check on the mac, best effort and only if Bryan's installed app is updated
   through his normal flow; do not install it yourself. On a draft such as
   `!<task with dependents>`:
   - Return opens the card.
   - Space marks the first offer immediately.
   - Ctrl-N/Ctrl-P move between offers.
   - Space unmarks.
   - Return submits only the marked offers.

## Out of scope

- Ctrl-J/Ctrl-K in the choice card.
- Destination retargeting (bob-cli-7j) and pull-in preview/notification copy
  (bob-cli-76).
- The Obsidian successor picker (`bob-navigation-hotkeys`) and bob-cli docs: no doc
  lists these Mac keys.
