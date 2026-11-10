---
tier: tale
title: Fix Mac Record Card previews for set choices, status footer, and route emoji
goal:
  A record draft's card shows the chosen choice value, keeps its footer hint after
  informational status, and the route emoji cleanly replaces the @.
size: medium
decisions:
  set_choice_row:
    ask: How should a Record Card choice row that already has a value show it?
    choices:
      value:
        Only Bob's display string, like other set rows; unset choice rows keep capsules
      highlight:
        All choice capsules stay, with the set one emphasized; long choice lists stay
        wide
    default: value
    why: It matches other set rows, update rows, and the committed record summary
    answer: value
  route_emoji:
    ask:
      Also fix the editor route emoji so it cleanly replaces the @ instead of sitting on
      top of it?
    choices:
      conceal:
        The @ glyph hides under an emoji fitted to one cell; draft bytes and offsets
        unchanged
      defer: The overlay stays as is and a small bug task bead records the root cause
    default: conceal
    why: Same screenshot and feature; small and self-contained
    answer: conceal
proposed_by: bbugyi200.athena.1ao
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 06:16:17
status: wip
---

# Fix Mac Record Card previews: set choices, status footer, route emoji

## Diagnosis

The screenshot shows a restored canceled draft:

```text
Hercules @movies/
- status:: watched
- [[abby]] liked this!
```

The Record Card showed three defects:

- The **Status** row showed both `🔖 To watch` and `✅ Watched` as identical gray
  capsules. Nothing showed that `watched` was the chosen value.
- The card footer read `Restored canceled draft` in red monospace, the style for a
  refusal, in place of the `⇥ next field · ⏎ create · Esc back to draft` hint.
- In the editor, the 🎬 adornment sits on top of a still-visible blue `@` and covers
  part of the `m` in `movies/`.

**Bob is correct; every defect is in bob-mac-capture.**

- `bob capture-parse -f json` reports `status` as `state: "set"`, `value: "watched"`,
  `display: "✅ Watched"`, together with the full `choices` list. Bob always sends
  `choices` for choice-kind fields, set or not.
- `bob capture --dry-run -f json` plans the right Markdown.
- The committed `movies/hercules.md` has `status: watched`.

Bob's wire contract does not change.

### Root causes

1. **Bob's value for a set choice field is hidden.**
   - In `Sources/BobMacCapture/RecordCardView.swift`, `RecordCardRowView.valueContent`
     checks `row.kind == "choices" && !row.choices.isEmpty` _before_ checking
     `row.display`.
   - As a result, every choice row renders the schema capsule list, whatever its state.
     That includes set, default, and update-unchanged rows. The capsules have no
     selection styling, so Bob's `display` never appears.
   - This has been present since the card first landed (8c13ac4).
   - `Tests/Fixtures/record-parse-status-liked.json` (status set, display `Liked`, 2
     choices) already exercises this case, but no assertion covers which branch is
     drawn.
2. **Informational panel status is painted as a refusal.**
   - `RecordCardView.footerResolved` passes `model.statusText` to
     `CaptureRecordFooterMessage.copy` in
     `Sources/CaptureCore/CaptureRecordPresentation.swift`.
   - `copy` uses `isOrdinaryStatus`, an allowlist of `""`, `Ready`,
     `Target cache stale`, and anything ending in `…`. Every other status counts as a
     refusal: it replaces the footer hint and is painted red.
   - `statusText` is an untyped panel string assigned at roughly 200 sites, and most of
     them are informational. Examples: `Restored canceled draft` from
     `restoreStashEntry`, `Inserted …`, `Similar tasks changed`,
     `Saved templates opened`, and `Added ^id to …`.
   - The status in the screenshot is not `isSupersedable` either. It therefore survives
     later parses and stays red while the user keeps editing.
   - The panel's own footer line shows `statusText` in secondary color. That line
     remains the canonical place for informational status.
3. **The route emoji overlay collides with the route glyph.**
   - `RecordRouteEmojiOverlay` in `Sources/BobMacCapture/CapturePanelView.swift` draws
     the emoji at 11 pt, shifted 2 pt left, over the route-start `@`.
   - Meanwhile, `paintHighlighting` in `CapturePanelModel.swift` still paints that `@`
     in the accent color. The `@` shows around the emoji, and the emoji, being wider
     than one monospace cell, covers the next character.
   - The `capture-record-route-emoji` render fixture draws the overlay _without_ the
     underlying text, so the image review could not catch this.

## Work — bob-mac-capture only

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
before editing. bob-cli needs no source or doc changes: the mounted
`reference/mac-capture/` pages come from bob-mac-capture's `docs/`. Keep Bob as the only
source of copy, as `decisions:mac-capture-is-a-thin-client` requires:

- no locally formatted values
- no local validity judgments

### 1. A set choice row shows its value

Add a portable resolver in CaptureCore so the branch order can be unit-tested on Linux.
For example, add an enum `CaptureRecordRowValueMode` in
`CaptureRecordPresentation.swift` and a method
`CaptureRecordRow.valueMode(hasCandidates:hasDiagnostic:)`. The resolver checks, in this
order:

1. live completion candidates (unchanged)
2. changed update row: `before → after` (unchanged)
3. the decision branch below
4. Bob's diagnostic
5. `⇥ fill` hint for a missing row
6. empty

Then make `RecordCardRowView.valueContent` switch on the resolved mode instead of its
inline `if` chain. Do not change any other styling.

> [!decision] set_choice_row = value Move the display branch ahead of the choice-capsule
> branch. A row whose `display` is nonempty always shows Bob's display in `stateColor`,
> keeping the missing-row `⇥ fill` hint. Choice capsules render only for choice rows
> with no display: missing, none, or invalid. Bob reports an invalid choice such as
> `status:: nope` with `display: null` plus an error diagnostic, so that row keeps
> today's capsules. The card footer still shows the diagnostic in red.

> [!decision] set_choice_row = highlight Keep capsules for every choice row. Add
> `value: String?` to `CaptureRecordRow`, taken from `field.value?.stringValue`; the
> memberwise initializer defaults it to nil. The capsule whose `choice.value` equals the
> row value gets:
>
> - the selected background tint used by `valueScroller`
> - `stateColor` text in semibold
> - the `.isSelected` accessibility trait
>
> Default-state rows are handled the same way. Rows with no value render exactly as
> today. The resolver returns the choice-capsule mode carrying the selected value.
> Clicking a capsule still only requests the field.

### 2. Only refusals turn the footer red

- **Model flag.** In `CapturePanelModel`, add
  `@Published private(set) var statusIsRefusal = false`, plus a `didSet` on `statusText`
  that resets it to `false`. Any later status then supersedes a refusal without touching
  the ~200 assignment sites. Add a small helper, such as `refuseStatus(_:)`, that
  assigns the text and _then_ sets the flag.
- **Sites that become refusals.** Use the helper or flag at these sites:
  - `finishRecordReturn`: the `blocker.message ?? "Needs <label>"` path, the
    `blocker.message` path, and the `.failed` preview path (`errorMessage ?? message`).
  - Parse application (`statusText = diagnostic.message`), only when that diagnostic's
    severity is `error`. Warnings stay ordinary status.
  - `failPreview`, the explicit dry-run failure (`Preview failed`), the live-preview
    failure (`Preview failed`), and `showRecordRefusal` (`failure.error`).
  - The three `Capture failed` assignments.
- **Footer API.** Change `CaptureRecordFooterMessage.copy` to take
  `statusIsRefusal: Bool` instead of judging the text:
  - refusal status (nonempty) → `(statusText, true)`
  - otherwise Bob's error diagnostic → `(diagnostic, true)`
  - otherwise `(footerHint, false)`
- **Footer caller.** `RecordCardView.footerResolved` passes `model.statusIsRefusal`. The
  card footer never mirrors informational status.
- **Unchanged behavior.** Keep `isOrdinaryStatus` and `isSupersedable` only for the
  parse-apply reset to `Ready`, and keep that behavior unchanged. Update their doc
  comments so they no longer claim to decide refusal color.

### 3. The route emoji replaces the `@` cleanly

> [!decision] route_emoji = conceal
>
> - **Conceal the `@`.** In `paintHighlighting`, find each `record_route` span that has
>   an adornment, using the same `CaptureRecordRouteAdornment.all(in:)` data: the owning
>   item must report a nonempty emoji. Paint that span's route-start grapheme (the `@`)
>   with a `.clear` foreground and keep the chip wash. Never change draft bytes, copy
>   text, or selection offsets.
> - **One shared predicate.** Conceal only while the overlay is drawn. The overlay is
>   hidden while `isSubmitting || editorInputLocked`, so expose one model predicate that
>   both the overlay and the highlighter use. If highlighting is not repainted when that
>   predicate flips, repaint on the flip, or drop the gate from both. A visible `@` must
>   never lose its emoji, and the emoji must never sit over a visible `@`.
> - **Fit the emoji.** In `RecordRouteEmojiOverlay`, fit the emoji to the `@`'s cell:
>   - frame it to `monoCharWidth` × `lineHeight`, centered
>   - set `lineLimit(1)` and a `minimumScaleFactor` so it never exceeds the cell
>   - remove the −2 pt x nudge
>
>   It must not overlap the next character.

> [!decision] route_emoji = defer Skip section 3. Use `/sase_new_task` to file one `bug`
> task bead (size `small`) in the bob-cli project. It must state root cause 3 above and
> the fix shape from the `conceal` branch.

## Tests

CaptureCore tests (`Tests/CaptureCoreTests/CaptureRecordTests.swift`, portable):

- **Value mode.**
  - `record-parse-status-liked.json` status row (set, display `Liked`, 2 choices): under
    `value` it resolves to display. Under `highlight` it resolves to capsules with
    `liked` selected.
  - `record-parse-incomplete.json` status row (missing, no display): choice capsules.
  - `record-parse-invalid.json` status row (invalid, null display, error diagnostic):
    choice capsules.
  - A default link row shows its display.
  - A changed update row resolves to changed.
  - Live candidates win over every other mode.
- **Footer.** Replace the allowlist assertions in
  `testFooterMessageTreatsReadyAsOrdinaryHint` with:
  - Informational text (`Restored canceled draft`, `Inserted ✅ Watched`,
    `Similar tasks changed`) with `statusIsRefusal: false` → the footer hint, not a
    refusal.
  - `Needs Status` with `statusIsRefusal: true` → red refusal.
  - Informational status plus an error diagnostic → the diagnostic, as a refusal.
  - Keep the existing `isSupersedable("Needs Status")` coverage.

BobMacCapture tests (`Tests/BobMacCaptureTests`, macOS CI):

- **Panel tests** (`CaptureRecordPanelTests.swift` or `CapturePanelModelTests.swift`):
  - Restoring a stashed record draft leaves `statusText == "Restored canceled draft"`
    and `statusIsRefusal == false`.
  - Return on a draft missing `status` sets `Needs Status` with
    `statusIsRefusal == true`.
  - A later informational status clears the flag.
  - Follow the bounded async harness the file already uses; no untimed waits.
- **Highlighting** (if `route_emoji = conceal`, in `CaptureEditorHighlightTests.swift`):
  - The route-start `@` is clear when its item reports an emoji.
  - It stays visible when the emoji metadata is absent (`record-parse-old-bob.json`) and
    while submitting or locked.
  - With two record types, both `@`s are concealed.
  - The plain draft string and copy offsets are unchanged.
- **Render fixtures** (`CaptureRecordCardDesignTests.swift`):
  - Add a Record Card render for the set-status fixture, with an informational
    `statusText`, in light and dark.
  - If `conceal`, redraw `capture-record-route-emoji` with the draft text in the editor
    font under the overlay, so bleed-through would be visible.

## Docs

In bob-mac-capture's `docs/capture-previews.md` (Record Card section):

- State that a set choice row shows Bob's display (or, under `highlight`, emphasizes the
  set capsule).
- State that the footer turns red only for refusals and Bob's diagnostics.
- If `conceal`: state that the type emoji replaces the route's `@` glyph in place.

That page is at the reference word cap (6000), so keep the wording net-neutral or
shorter. Run `just docs-check`.

## Verification

1. **Portable gate.** `just portable-check` on athena (Swift 6 is installed). Format
   only edited lines (`.swift-format`: 4 spaces, 100 columns).
2. **Mac scratch compile.**
   - Read the tailnet memory first:
     `sase memory read tailnet.md -r "Mac scratch compile"`.
   - On the Mac, rsync a scratch copy without `.build`/`.git`.
   - Run `bundle.sh` and `just typecheck-tests`, then delete the copy.
   - Never use Bryan's checkout or `just install`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.
3. **CI gate.** Run `Scripts/macos-ci-check` before landing. After the push, use
   `/sase_monitor` to watch the exact-SHA macOS run to green.
4. **Review the CI render-fixture images.** Confirm that:
   - the Status row reads `Liked` (or the highlighted capsule) instead of two identical
     capsules
   - the footer shows the `⇥ next field …` hint after an informational status
   - under `conceal`, the emoji hides the `@` without overlapping the next letter
5. **Final report.** Tell Bryan to reinstall Mac Capture, then reproduce the
   screenshot's draft with a fresh title, because `movies/hercules.md` now exists and
   would refuse with `record_exists`.
