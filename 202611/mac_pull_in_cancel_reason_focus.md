---
tier: tale
title: Make ⌃⇧X Cancel work on the Bob Mac Capture Pull-In card
goal:
  Pressing ⌃⇧X on any Pull-In (successor) row visibly opens a focused Cancel reason
  field, and refused chords show Bob's why.
size: medium
decisions:
  keycast_shift:
    ask:
      Should this tale also make the screencast keycast overlay show ⇧ on chorded
      letters, so ⌃⇧X stops recording as ⌃X?
    choices:
      include:
        Fix the Hammerspoon keycast label in chezmoi plus its busted spec; future
        screencasts show ⌃⇧X
      skip:
        Leave the overlay as is; ⌃⇧X and ⌃X keep rendering the same in screencasts and
        keystroke tracks
    default: include
    why:
      The ambiguity hid which chord was pressed in this bug's screencast; the fix is
      tiny
    answer: include
proposed_by: bbugyi200.athena.1e4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 14:07:03
status: wip
---

# Plan: Make ⌃⇧X (Cancel) work on the Bob Mac Capture Pull-In (successor) card

## Summary

The successor panel in the 2026-11-18 screencast is **Bob Mac Capture's Pull-In card**,
the native window that floats over Obsidian. Its footer shows
`⌃N ⌃P Move · space Mark · ⌘A All · ⌃X Done · ⌃⇧X Cancel · esc Back`. The Obsidian
plugin's successor picker (`bob-navigation-hotkeys`) is separate code with different
hints. It already focuses its reason input and is out of scope.

⌃⇧X reaches the app and the router maps it correctly. The **Cancel reason editor it
opens never works**:

1. **The reason field is never focused.** `CapturePullInReasonField` is a plain SwiftUI
   `TextField` with no focus binding. `requestFocus(.pullInReasonField)` reaches
   `CapturePanelView.applyFocusRequest`, which only resolves `.editor`,
   `.pullInOfferList`, `.inboxRouteRows`, and `.moveChoiceRows`. It sets SwiftUI focus
   to nil, and no AppKit field claims first responder. The controller's
   `pullInReasonFocused` detection looks for a `CapturePickerFilterNSTextField` with
   `pullInReasonFieldAccessibilityIdentifier`, and a SwiftUI `TextField` never matches
   it. So the router's reason-mode branch is dead code. Return in the field confirms and
   also **commits the whole card** through `confirmPullInChoice`, and Esc is not the
   reason-restore path.
2. **The field is clipped under the last row.** `CapturePullInChoiceHeightPolicy` sizes
   the list viewport from row and section counts only. The inline field sits under the
   highlighted row and pushes content past the fixed viewport. Nothing scrolls to it,
   because `scrollTo` fires only when `highlightedKey` changes. In the screencast the
   highlighted row was the last of two ("#ref Paste-ready captures…", an open `[*]`
   task, so Cancel-eligible). ⌃⇧X opened an invisible, unfocused editor, and nothing
   visibly happened. The existing design fixture opens the editor on row 1, so it never
   showed the clipping.
3. **Refusals are invisible (same chords, ineligible rows).** On refusal,
   `stagePullInDoneForHighlight` / `beginPullInCancelForHighlight` (and the `…ForRow`
   variants) only beep and post a VoiceOver announcement. The strip shows
   `verdictRefusalMessage` only when `hasVerdicts || activeReasonKey != nil`, and these
   early returns never set it. The original feature plan required that "an ineligible
   row also displays and announces Bob's `why`".

The original feature plan (`202611/mac_pull_in_verdict_ui.md`, commit `6f4e2799`)
specified the behaviour that is missing: "Opening Cancel focuses an inline single-line
field… Blur confirms the text once", plus "actual first-responder detection" and
"Account for the inline field/status strip in panel sizing". This tale implements that
contract.

**Keys:** keep **⌃X = Done / ⌃⇧X = Cancel**. The chord is delivered: nothing on the Mac
binds ⌃⇧X globally (checked: Hammerspoon binds, Amethyst uses ⌃⌥⇧X, no Services,
symbolic hotkeys, or app key equivalents).
`testRouterMapsOfferCloseChordsAndReasonFieldKeys` already proves the routing. The chord
also mirrors the Obsidian picker's Ctrl+Shift+X and the documented contract
(`docs/capture-previews.md`, `docs/keyboard.md`). Do not rebind it.

## Where the work happens

All product changes are in the linked **`bob-mac-capture`** repo. Open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "…"`) and read its `AGENTS.md` first.
The `bob-cli` checkout itself needs no changes. If `keycast_shift = include`, the linked
**`chezmoi`** repo also changes (§6).

Key files (bob-mac-capture):

- `Sources/BobMacCapture/CapturePullInChoiceView.swift`: layout constants
  (`extension CapturePanelLayout`), `CapturePullInChoiceHeightPolicy`,
  `CapturePullInChoiceCard` (`ScrollViewReader` list), `CapturePullInReasonField`,
  footer `verdictStripText`.
- `Sources/BobMacCapture/CapturePickerFilterField.swift`: AppKit-owned
  `CapturePickerFilterField` / `CapturePickerFilterNSTextField` (already used for the
  Pull-In filter with `focusTarget: .pullInFilter`).
- `Sources/BobMacCapture/CapturePanelModel.swift`: `stagePullInDoneFor{Highlight,Row}`,
  `beginPullInCancelFor{Highlight,Row}`, `confirmPullInReason`, `restorePullInReason`,
  `updatePullInReasonText`, `movePullInChoiceHighlight`, `focusPullInChoiceFilter`,
  `requestFocus`.
- `Sources/BobMacCapture/CapturePullInChoice.swift`: `CapturePullInChoiceState`
  (`activeReasonKey`, `verdictRefusalMessage`, `moveHighlight`, `beginOfferCancel`,
  `confirmOfferCancel`, `restoreOfferCancel`).
- `Sources/BobMacCapture/CapturePanelController.swift`:
  `repairPickerFilterFocusIfNeeded`, the `CaptureKeyRoutingContext` construction
  (`pullInReasonFocused`), and the `.beginPullInCancel` / `.confirmPullInReason` /
  `.restorePullInReason` dispatch.
- `Sources/BobMacCapture/CapturePanelView.swift`: `applyFocusRequest`,
  `currentAuxiliaryHeight`.
- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`: `pullInChoiceCommand`. Expected
  to stay unchanged.

## Changes

### 1. Make the reason field AppKit-owned so ⌃⇧X focuses it

- Replace the SwiftUI `TextField` inside `CapturePullInReasonField` with
  `CapturePickerFilterField`:
  - `text: choice.activeReasonText`
  - `placeholder: "Why cancel? (optional)"`
  - `accessibilityLabel: "Why cancel? optional reason"`
  - `focusRequest: model.focusRequest`
  - `onTextChange: { model.updatePullInReasonText($0) }`
  - `fieldIdentifier: pullInReasonFieldAccessibilityIdentifier`
  - `focusTarget: .pullInReasonField`
  - `onFocusChange:` see §3
- Keep the `⊘ Why cancel?` caption, the `↵ · Esc back` hint, background, padding, and
  the field's placement outside the row's toggle `Button`.
- `CapturePickerFilterField` hard-codes a 15 pt font. Add an optional font-size
  parameter (default 15, so the existing filters render unchanged) and pass the body
  size the reason field used before.
- Return and Esc now go through the existing router reason branch (`pullInReasonFocused`
  becomes true): Return → `.confirmPullInReason`, Esc / ⌃[ → `.restorePullInReason`.
  Drop the SwiftUI `.onSubmit` path. Return in the reason field must confirm the reason
  only, never commit the card.
- Confirm that `updateNSView`'s existing behaviour works for both fields: it claims
  focus when `focusRequest.target == focusTarget` with a new sequence, resigns when the
  target moves elsewhere, and re-claims on the next runloop turn. The filter resigns
  when the request moves to `.pullInReasonField`, and the reason field resigns when
  `confirmPullInReason` / `restorePullInReason` request `.pullInOfferList`.
- A mouse click into the reason field must not be undone by the next SwiftUI update.
  Today `updateNSView` would resign it because the focus request still targets the list.
  Mirror `focusPullInChoiceFilter`: `onFocusChange(true)` calls a new
  `model.focusPullInReasonField()` that does `requestFocus(.pullInReasonField)` (only
  while `activeReasonKey != nil`).

### 2. Focus repair in the controller

In `CapturePanelController.repairPickerFilterFocusIfNeeded`, add a branch like the
`.pullInFilter` one. When `model.pullInChoiceVisible`,
`model.focusRequest.target == .pullInReasonField`, `activeReasonKey != nil`, and the
field with `pullInReasonFieldAccessibilityIdentifier` exists but does not hold first
responder, call `requestFirstResponder()` and emit a signpost such as
`pull-in-reason-focus-repaired`. This covers the field being created a frame after the
request, or being recycled by the `LazyVStack`.

### 3. Blur confirms once

The original contract says "Blur confirms the text once". Add
`model.pullInReasonFieldDidBlur()`. Wire it to `onFocusChange(false)` (the field's
`controlTextDidEndEditing`). It confirms the reason the same way `confirmPullInReason`
does, but only when `activeReasonKey` is still set. The Return path and the Esc path
already clear `activeReasonKey` before focus moves, so the blur that follows them must
be a no-op: no double-apply and no second advance. Focus moving inside the card counts
as a blur, for example clicking another row or ⌃N/⌃P after a click moved focus out.

### 4. Size and scroll so the field is always visible

- Add `CapturePanelLayout.pullInReasonFieldHeight`. Give `CapturePullInReasonField` that
  fixed frame height, so the policy and the view agree. Include the `VStack(spacing: 2)`
  gap or fold it into the constant.
- Extend `CapturePullInChoiceHeightPolicy` with a reason-editor flag (for example
  `hasActiveReasonEditor`). Set it in `init(choice:)` from
  `choice.activeReasonKey != nil` and add `pullInReasonFieldHeight` to
  `listContentHeight` while it is set. Leave `maxListHeight` / `minListHeight`
  unchanged.
  - With fewer than `pullInMaxVisibleRows` rows the viewport, and through
    `currentAuxiliaryHeight` the panel, grows by one field height while editing and
    shrinks back afterwards.
  - With more rows the viewport stays at max and the list scrolls.
  - Update the policy's doc comment: the budget is still fixed against
    filtering/collapsing, and only the transient reason editor adds height.
- In `CapturePullInChoiceCard`'s `ScrollViewReader`, give the reason field a stable
  `.id` (for example `"reason:" + key`). Add
  `.onChange(of: choice.activeReasonKey) { … proxy.scrollTo(reasonID, anchor: nil) }`.
  Defer one main-queue turn if needed so the scroll runs after the viewport resize.
  Respect Reduce Motion as the card already does.
- `previewMode` (`previewList` with
  `.frame(height: policy.listViewportHeight).clipped()`) picks up the new height
  automatically. Check that the render fixture shows the field under the last row.

### 5. Show refusals for ⌃X / ⌃⇧X on ineligible rows

- In the model's four entry points (`stagePullInDoneForHighlight`,
  `stagePullInDoneForRow`, `beginPullInCancelForHighlight`, `beginPullInCancelForRow`),
  the ineligible branch keeps the beep and announcement. It also stores
  `choice.verdictRefusalMessage = choice.refusalReason(for: key) ?? "This offer cannot close here"`
  and writes the choice back. Keep the existing "Closing offers needs a newer bob"
  capability branch, and give it the same strip treatment if that falls out naturally.
- In `verdictStripText`, show `verdictRefusalMessage` whenever it is non-empty. Drop the
  `hasVerdicts || activeReasonKey != nil` gate for it. The Esc-armed text still wins,
  and `errorMessage` keeps today's gate.
- Clear `verdictRefusalMessage` when the highlight moves (`moveHighlight` /
  `movePullInChoiceHighlight`), on mark toggles, and on filter edits. Successful staging
  already clears it. This stops a stale refusal from describing another row.

### 6. Keycast overlay shows ⇧ on chorded letters

> [!decision] keycast_shift = include In the linked `chezmoi` repo (open with
> `/sase_repo`; read its `AGENTS.md`), edit `home/dot_hammerspoon/keycast_model.lua`
> `format_label`. Today a printable letter always drops the ⇧ prefix ("Shift is already
> visible in the case of a printable letter"), but letters are also upper-cased whenever
> ⌃/⌥/⌘ is held, so ⌃⇧X and ⌃X both render `⌃X`. Drop ⇧ only when **no** chord modifier
> is held: plain Shift+x stays `X`, ⌃⇧X becomes `⌃⇧X`, and ⌃X stays `⌃X`. Add spec cases
> to `tests/hammerspoon/keycast_model_spec.lua`, and update any `screen_recorder` /
> keystroke-track spec that asserts the old label. Run the busted specs the way CI does
> (`.github/workflows/ci.yml` installs `busted`). After the change lands, run
> `chezmoi update -a --force` per chezmoi's instructions. The Mac picks the change up on
> its next chezmoi apply, and the Hammerspoon restart hook reloads it.

> [!decision] keycast_shift = skip Make no chezmoi changes.

### 7. Docs

In `docs/capture-previews.md` ("Successor offers"), replace "Control-Shift-X opens the
Cancel reason" with wording that says the reason field takes focus and scrolls into
view: Return confirms it, Esc restores the prior outcome, and a refused Done/Cancel
shows Bob's reason in the strip. Make the same small fix in `docs/keyboard.md` if it
needs one. Run `just docs-check`.

## Tests (bob-mac-capture)

Extend the existing suites:

- **`CapturePullInChoiceDesignTests`**
  - Add a fixture that opens the reason editor on the **last** row of the two-row close
    card (`beginPullInCancelForRow` on the second offer). Assert that the card height /
    `CapturePullInChoiceHeightPolicy(choice:)` grows by `pullInReasonFieldHeight` and
    that the field lies within the list viewport.
  - Add a refusal-strip fixture for an ineligible row with **no** staged verdicts.
- **Height policy unit tests** (where the pull-in policy is tested, or new ones next to
  the design tests)
  - Reason editor adds exactly the field height below the max.
  - The viewport is clamped at max with many rows.
  - There is no change when no editor is active.
- **`CapturePullInChoiceModelTests`**
  - `beginPullInCancelForHighlight` on an eligible last row sets `activeReasonKey` and
    `focusRequest.target == .pullInReasonField`.
  - `pullInReasonFieldDidBlur` confirms once (a second call or a Return-then-blur
    sequence does not double-apply or double-advance), and Esc-then-blur restores
    without staging.
  - Refusal from both chords sets `verdictRefusalMessage` with no verdicts staged, and
    moving the highlight clears it.
- **Controller / AppKit level**, where feasible with the existing patterns in
  `CaptureEditorTextConfigurationTests`, which use real `NSPanel` and
  `CapturePickerFilterNSTextField`:
  - A `CapturePickerFilterNSTextField` with `pullInReasonFieldAccessibilityIdentifier`
    is found by
    `CapturePanelController.pickerFilterField(fromFirstResponder:identifier:)` and
    `findPickerFilterField`, so `pullInReasonFocused` is real.
  - If a controller harness can route an `NSEvent` through `routeKeyDown`, assert that
    ⌃⇧X leaves the reason field as first responder and that Return then dispatches
    `.confirmPullInReason`, not `.confirmPullInChoice`.
- Keep `testRouterMapsOfferCloseChordsAndReasonFieldKeys` green. The router is expected
  to stay unchanged.

## Verification

Follow bob-mac-capture's `AGENTS.md` exactly:

1. `just portable-check` on athena.
2. Compile a scratch copy on the Mac, best effort: rsync without `.build`/`.git`, run
   `bundle.sh`, then `just typecheck-tests`, then delete the copy. Never use Bryan's
   checkout or `just install`. If the Mac is offline, say in the final report that the
   macOS targets are unverified.
3. `just docs-check` (docs changed).
4. Run `Scripts/macos-ci-check` before landing. After the push, watch the exact-SHA
   macOS CI run to green with `/sase_monitor`. A red lint step can hide later steps, so
   check all of them.
5. Format only changed lines (`.swift-format`: 4-space indent, 100 columns).
6. If `keycast_shift = include`: run the chezmoi busted specs, then
   `chezmoi update -a --force` after the chezmoi commit lands.

Manual acceptance, for Bryan once the updated app is installed, reproducing the
screencast:

- Open a capture whose Pull-In card has two close-eligible offers and highlight the
  **last** row. ⌃⇧X shows the `⊘ Why cancel?` field fully visible under that row (the
  card grows) with the caret in it.
- Typing a reason and pressing Return stages `⊘ Cancel` with the quoted reason and
  advances. The card does not commit.
- ⌃⇧X then Esc restores the prior outcome (including a staged Done) without arming the
  discard warning.
- ⌃⇧X on a staged Cancel row clears it to Leave.
- On an ineligible row (for example a recurring task), ⌃X and ⌃⇧X show Bob's `why` in
  the strip even with nothing staged.
- ⌃X Done behaviour is unchanged.

## Out of scope

- The Obsidian `bob-navigation-hotkeys` successor picker (its own Ctrl+Shift+X path
  already focuses its reason input).
- Changing the ⌃X/⌃⇧X chords or any other Pull-In key.
- Bob CLI (`offer_close.rs`) eligibility or copy.
