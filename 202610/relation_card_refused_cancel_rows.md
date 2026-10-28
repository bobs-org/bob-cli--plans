---
tier: tale
title: Relation card renders refused cancel-label rows cleanly
goal:
  On a new-task `~` (or recurring `~~`) relation, the Mac relation card shows every row,
  keeps refused cancel rows compact, and states bob's refusal once in full.
size: medium
decisions:
  refusal_reason_visibility:
    ask:
      Should the card show bob's refusal reason whenever a refused cancel row is
      visible, not just when highlighted?
    choices:
      highlighted:
        Reason shows only while a refused row is highlighted or typed; others show it on
        hover
      always:
        Reason also shows muted under the list whenever refused rows are visible (one
        extra line)
    default: highlighted
    why: Most new-task ~ cards pick an ordinary label, so an always-on line is noise
    answer: highlighted
proposed_by: bbugyi200.apollo.az
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-28 14:31:20
status: wip
---

# Relation card: render refused cancel-label rows cleanly (Bob Mac Capture)

## Problem

The cancel-labels epic (`bob-cli-dl`, plan `plan:202610/relation_cancel_labels.md`)
shipped the Mac `⊘ cancels` cue. When the source is the **new task being captured** (a
forward `~` relation on a capture item), bob's verdict is `new_task`. Every configured
cancel label (`duplicate of` and `superseded by` in Bryan's config) then comes back
**refused**: `disabled_reason` is set to bob's CL6 sentence, `replacement` is `""`, and
`cancels_source` is `true`. A `~~` relation whose source repeats gives the same shape
with verdict `recurring`.

bob's output is correct. Real output for draft
``Add `[w` and `]w` keymaps! @bob:wip-nav-keys ~bob:link-navigation-modes|``:

```text
implements     count 14  replacement |implements
inspired by    count 8   replacement |"inspired by"
superseded by  count 3   replacement ""  cancels_source  disabled_reason "“Add …” is new — it would be born cancelled · to retire “…”, use ~~"
duplicate of   count 1   replacement ""  cancels_source  disabled_reason (same)
polishes       count 1   replacement |polishes
verified by    count 1   replacement |"verified by"
relation_cancel.verdict = new_task, message = the same sentence
```

Bryan's screenshot of the Mac card for that draft (`superseded by` highlighted) shows
four defects:

1. **A blank row, and the highlighted row is gone.** The list reads `implements`,
   `inspired by`, `duplicate of`, _(blank slot)_, `polishes`. `superseded by` is missing
   even though it is highlighted: the sentence pill and the `2 links` field badge both
   name it.
   - **Root cause:** `CapturePanelView.swift` builds the relation card list with
     `ForEach(rows, id: \.id)`, and the Hidden section with
     `ForEach(hiddenRows, id: \.id)`.
   - `CaptureCompletionCandidate.id` (`Sources/CaptureCore/CaptureModels.swift`) joins
     `replacement` with route, path, block, and text fields. All of those are nil on
     `relation_label` rows, so the id is just `replacement`.
   - Both refused rows therefore have id `""`. SwiftUI's duplicate `ForEach` identity
     draws `duplicate of` in the first slot and nothing in the second.
   - Before this epic, the only relation-label row with an empty replacement was the
     typed invalid row, and it is filtered out of the list. So the collision is new with
     cancel labels, and it happens every time both configured labels are refused.
2. **A truncated reason repeated on each refused row.**
   - `relationRowContent` appends `Text(reason).lineLimit(1)` to the trailing edge of
     the row, which cuts bob's long sentence down to ``“Add `[w` and `]…``.
   - The same unreadable fragment repeats on every refused row.
   - `CaptureRelationCancelPresentation.slot` returns nil for `recurring` and
     `new_task`. So the card never shows the full reason anywhere unless Return is
     pressed, which sets the red error line.
3. **The footer shows `↩ inserts` with nothing after it.**
   - `relationLabelFooterItems` always adds `↩ inserts \(replacement)` for the
     highlighted row, and the replacement is empty here.
   - In one-off (`||`) mode it advertises
     `↩ inserts ||"superseded by" · won't be suggested again`, even though Return
     refuses.
4. **A typed refused label is marked invalid.**
   - Typing `duplicate of` on a new task gets bob's typed row with `disabled_reason`,
     and `relationLabelPromptBadgeText` turns any typed `disabled_reason` into the red
     `invalid` badge.
   - The label is valid; only this relation's source refuses it.
   - Related: the badge says `1 links` for a count of 1, while the row says `1 link`.

The epic did not fix this. The Mac phase (`bob-cli-dl.7`) could not reach the mac, so
`BobMacCapture` (the target that holds the view) was neither type-checked nor
screenshotted there. Its fixtures cover only the `cancels`, `already_closed`, and
inverse-hint verdicts. The one later Mac commit (caret underline) does not touch this
code.

## Scope

- **Repo:** `bob-mac-capture` only. Open it with `/sase_repo`
  (`sase repo open bob-mac-capture -r "<why>"`) and read its `AGENTS.md` first.
- **No changes** to bob-cli, bob-plugins, or chezmoi. Nav's Links stage only ever acts
  on existing tasks, so it never sees the `new_task` verdict.
- **Thin client** (`decisions:mac-capture-is-a-thin-client`): Swift must not author
  verdict text or decide a verdict. It renders bob's `disabled_reason`,
  `relation_cancel.verdict` and `.message`, and `cancels_source`.

## Changes

All paths are relative to the bob-mac-capture repo root.

### 1. Identify relation card rows by label key

- `Sources/BobMacCapture/CapturePanelModel.swift`:
  - Add a small `Identifiable` entry type, for example
    `struct CaptureRelationLabelRowEntry: Identifiable { let id: String; let candidate: CaptureCompletionCandidate }`.
  - Its `id` is `relationLabelSelectionKey(for: candidate)`: bob's `key`, falling back
    to the folded label.
  - Expose `relationLabelPromptRowEntries` and `relationLabelHiddenPromptRowEntries`,
    built from the existing `relationLabelPromptRows` and
    `relationLabelHiddenPromptRows`, in the same order.
  - Bob's catalog keys are unique. The visible list excludes hidden keys, and the Hidden
    section skips ghost-hide keys, so ids are unique across both loops. The `.id(key)`
    used by `ScrollViewReader` already uses the key.
- `Sources/BobMacCapture/CapturePanelView.swift`: change both relation-card loops
  (around the `ForEach(rows, id: \.id)` and `ForEach(hiddenRows, id: \.id)` sites) to
  iterate the entries keyed by `id`. Keep `topKey`, selection, and the Button actions
  keyed exactly as they are today.
- **Leave `CaptureCompletionCandidate.id` unchanged.** Other pickers use it for row
  identity, and widening it is out of scope. Add a one-line comment at the relation-card
  `ForEach` explaining that refused rows share an empty `replacement`, so rows are keyed
  by label.

### 2. Stop squeezing the refusal sentence into refused cancel rows

In `relationRowContent` (`CapturePanelView.swift`):

- When `candidate.cancelsSource` is true and the row is disabled, do not render the
  inline `Text(reason)`.
- Keep the existing dimming (`opacity(0.55)`, secondary foreground) and the
  `circle.slash cancels` capsule.
- Add `.help(reason)` on disabled rows so hovering shows bob's full sentence.
- Disabled rows that are not cancel rows keep today's inline reason. No such catalog row
  exists today, but the guard keeps that path unchanged.
- The VoiceOver label (`relationRowAccessibilityLabel`) keeps appending the reason, as
  it does today.

### 3. Show bob's refusal message once, in the cancel slot

In `Sources/CaptureCore/CaptureRelationCancelPresentation.swift`, change
`slot(candidate:cancel:)`:

- When the active candidate has `cancelsSource` and the verdict is `recurring` or
  `new_task`, return `CaptureRelationCancelSlot(text: cancel.message, style: .muted)`.
- Update the doc comments on `slot` and on `Style.muted`, which say these verdicts "stay
  on the existing `disabled_reason` error path".
- The active candidate is the highlighted row, else bob's fresh typed row
  (`relationLabelCancelCandidate`). This already covers both "highlighted" and "typed".

In `CapturePanelView.swift`:

- The slot `Text` must wrap and never truncate. Add
  `.fixedSize(horizontal: false, vertical: true)` and no `lineLimit`.
- Muted means secondary color, with no ⊘ glyph and no red. ⊘ means "this will cancel"
  (CL6), and red stays on the existing Return-refusal error line
  (`prompt.errorMessage`).

> [!decision] refusal_reason_visibility = always Also pass the visible rows' state into
> the slot logic. When the active candidate is not a cancel label, bob's verdict is
> `recurring` or `new_task`, and at least one visible row (`relationLabelPromptRows`)
> has `cancelsSource` with a `disabled_reason`, show `cancel.message` muted in the same
> slot. An inverse `hint` keeps precedence over it. Keep the logic in
> `CaptureRelationCancelPresentation`, for example with a `refusedRowsVisible: Bool`
> parameter, so CaptureCore tests pin it.

> [!decision] refusal_reason_visibility = highlighted No extra state. The slot shows the
> refusal only through the active-candidate rule above.

### 4. Footer: no `↩ inserts` for a refused row

In `relationLabelFooterItems` (`CapturePanelModel.swift`):

- When the highlighted row has a non-empty `disabledReason`, omit the `↩` item in both
  the plain branch and the one-off branch.
- Keep `⌃D hide` / `unhide` / `undo` (hiding a refused label still works through
  `hide_replacement`), `⌃H`, and `esc back`.
- Do not invent new footer copy.

### 5. Field badge for a typed refused cancel label

In `relationLabelPromptBadgeText`:

- Return `invalid` only when the fresh typed row has a non-empty `disabledReason`
  **and** `cancelsSource` is false.
- A typed cancel label that bob refuses by verdict falls through to the normal typed
  badge (`N links` / `new` / `hidden`).
- Use singular `1 link` for a count of 1 in all three badge branches (highlighted,
  typed, and catalog fallback), matching the row text.
- `relationLabelPromptCanSubmit` already returns false for a disabled typed row. Keep
  that.

### 6. Leave alone

- The sentence pill's cancel tint (it marks a cancel label).
- The undimmed source when the verdict is not `cancels`.
- Ctrl-N/P landing on refused rows. They stay selectable, so the reason can be read and
  ⌃D still hides them.

## Tests and fixtures

**Fixtures from real bob output.** Record them the way the `bob-cli-dl.7` fixtures were
recorded:

- Use a scratch vault and `BOB_CONFIG_FILE` with
  `relations.cancel_labels: [duplicate of, superseded by]`.
- Include at least one existing labeled LINKS line per cancel label, so the rows carry
  counts.
- Include one ordinary target task with a block ID, and one recurring task with a block
  ID.

Add to `Tests/Fixtures/`:

- `relation-cancel-complete-new-task.json`: `capture-complete --format json` for
  `New idea @inbox ~<note>:<target>|` with the cursor at the end. It has verdict
  `new_task` and two refused rows with `replacement: ""`.
- `relation-cancel-complete-new-task-typed.json`: the same draft ending
  `|"duplicate of"`, which gives bob's typed refused row.
- `relation-cancel-complete-recurring.json`: an existing-task `~~<note>:<recurring>|`
  relation, which gives verdict `recurring`.

**`Tests/CaptureCoreTests/CaptureRelationCancelTests.swift`:**

- Extend `testRealBobFixturesDecode` to cover the three fixtures. Assert that the
  refused rows decode `disabledReason`, an empty `replacement`, and `cancelsSource`.
- In `testSlotAndDimmedSourcePerVerdict`, replace the `XCTAssertNil` for the recurring
  verdict with the `.muted` slot carrying bob's message. Add the same check for
  `new_task`.
- Assert that an ordinary candidate with a `new_task` verdict and no hint still returns
  nil. Under `always`, it returns nil only when no refused row is visible; add the
  positive case too.
- Assert that `dimsSource` stays false for both verdicts.

**`Tests/BobMacCaptureTests/CaptureRelationPanelTests.swift`.** Follow the existing
card-setup helpers. With the `new_task` fixture loaded into the card:

- `relationLabelPromptRowEntries` ids are unique, contain both `superseded by` and
  `duplicate of`, and keep bob's order.
- Highlight `superseded by` with Ctrl-N. Then:
  - `relationLabelFooterItems` has no `↩` item, in both plain and one-off mode;
  - `relationLabelCancelSlot` is bob's message with `.muted` style;
  - `relationLabelPromptCanSubmit` is false;
  - submitting sets `errorMessage` to bob's reason and leaves the draft unchanged;
  - the badge reads `3 links` (or whatever count the fixture records).
- With the typed fixture: the badge is not `invalid`, the slot shows bob's message, and
  submit is refused.
- For a count-1 label, the badge reads `1 link`.

Existing invalid-label tests (`testRelationCardInvalidLabelBlocksSubmit`,
`testRelationLabelDisabledEmptyReplacementRefuses`,
`testRelationLabelTooLongUsesBobDisabledReason`, and similar) must still pass unchanged.
Those rows have no `cancels_source`.

## Docs

- In `docs/relationship-pickers.md`, replace "Disabled recurring and new-task rows keep
  Bob's `disabled_reason` and stay dimmed." with the new behavior:
  - refused cancel rows stay dimmed with the `cancels` capsule, and the full reason
    shows on hover;
  - highlighting or typing one puts bob's `relation_cancel.message` in the slot, muted
    (plus the `always` sentence if that branch is chosen);
  - the footer drops `↩`;
  - the badge keeps the link count;
  - Return shows the reason as the error.
- Run `just docs-check`.

## Verification

Follow bob-mac-capture's `AGENTS.md`:

- Run `just portable-check` on a host with Swift (athena; apollo has no `swift`) for the
  CaptureCore changes and tests.
- `BobMacCapture` and its tests changed, so also compile a scratch copy on the mac (best
  effort): rsync without `.build` and `.git`, run `bundle.sh` and
  `just typecheck-tests`, then delete the copy. Never use Bryan's checkout, and never
  run `just install`.
- Land only on green macOS CI (Build and Test, including `CaptureRelationPanelTests` and
  `CaptureRelationCancelTests`). The two unconfigured-theme caret failures tracked by
  `bob-cli-dr` are pre-existing; call them out rather than counting them as this
  change's.
- If the mac is reachable, take light and dark screenshots of the card for a new-task
  `~` draft with `superseded by` highlighted. They should show every row present, no
  blank slot, no inline fragment, and bob's full sentence in the slot. Attach them to
  the final report. If the mac is unreachable, say that the macOS targets and visuals
  are unverified.
- Commit in bob-mac-capture as one conventional commit, for example
  `fix(relation-picker): key relation card rows by label and explain refused cancel labels`.

## Out of scope

- Hidden-section rows are built from `relations list` info (`CaptureRelationLabelInfo`).
  They carry neither `cancels_source` nor a refusal, so a hidden cancel label looks
  insertable on a new task, and bob refuses it at commit. Note this as a possible
  follow-up; do not fix it here.
- Widening `CaptureCompletionCandidate.id` for other pickers.
- Any bob-cli contract change.
