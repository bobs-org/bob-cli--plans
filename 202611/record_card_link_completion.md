---
tier: tale
title: Fix link and task completion on record drafts
goal:
  Typing [[ or (( while adding or editing a record shows bob's live note list or Task
  Wikilink picker instead of hiding it behind the Record Card, with keys routed to that
  assist.
size: medium
decisions:
  field_value_links:
    ask:
      Should free-text record field values (like a zettel's claim::) also complete [[
      note links and (( task links?
    choices:
      prose_links:
        bob hands open [[/(( tokens in non-suggest Text values to link completion; docs
        updated
      values_own:
        Values keep owning [[/(( as documented; only title lines and note bullets
        complete (Mac-only)
    default: prose_links
    why:
      Claim-style fields are prose, and links typed into them already capture and store
      correctly
    answer: prose_links
proposed_by: bbugyi200.athena.1dt
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 07:07:38
status: wip
---

# Fix link and task completion on record drafts (Record Card)

## Problem

In the capture panel, typing `[[` or `((` while adding a record (`@album/`) or editing
one (`@album/<stem>`) never gives usable completion. The screencast
(`~/tmp/screencasts/20261117_215727.mp4`) shows this draft on the `sase_blog_zettel`
album:

```text
AI Slop @sase_blog_zettel/
- status:: draft
- claim:: I have noticed several different types of AI slop. ...
- section:: not_ready
- See the [[sa
```

- `[[`, `[[s`, `[[sa`: the "New zettel" Record Card stays up. A note list flashes for
  one frame only, and it shows the _previous_ keystroke's rows (`12qs`, `20230719`, …
  for `[[s`). Then it disappears.
- `((`: the editor dims and the footer switches to the Task Wikilink picker hints (Move
  · Insert link · Insert & Capture · Capture & Open · esc). No picker rows ever show,
  because the Record Card is still drawn in the slot.

## What was verified

bob already answers correctly for title lines and note bullets on both create and update
drafts (checked with
`bob capture-complete --all-tasks --cursor N --format json -- "<draft>"`):

| Caret position                            | bob context                | `record` key |
| ----------------------------------------- | -------------------------- | ------------ |
| `- See the [[sa` (create or update draft) | `wikilink_note`, 20 rows   | absent       |
| `- See the ((` (create or update draft)   | `wikilink_task` + `picker` | absent       |
| `AI Slop [[sa @sase_blog_zettel/` (title) | `wikilink_note`            | absent       |
| `- claim:: see [[sa`                      | `record_value`, **0 rows** | present      |
| `- claim:: see ((`                        | `record_value`, **0 rows** | present      |

So the screencast bug is entirely in Bob Mac Capture. The field-value rows are a
separate bob-cli limitation, covered by DECISION `field_value_links`.

## Root causes

1. **Mac: the slot order hides every editor assist.** In
   `Sources/BobMacCapture/CapturePanelView.swift`, `auxiliaryRegion` checks
   `else if let presentation = model.recordPresentation { RecordCardView … }` _before_
   `model.pickerVisible` (`CapturePickerCard`) and before `auxiliaryScrollRegion`. The
   scroll region holds the ordinary `CompletionList`, `CapturePickerChip`,
   `TaskIDPromptCard` (Add block ID for an ID-less `((` pick),
   `RelationLabelPromptCard`, and `PomodoroNamePromptCard`. On a record draft none of
   those can show. The one-frame flash happens because the analysis start in
   `CapturePanelModel.swift` sets `recordPresentation = nil` while
   `recordInteractionPending` stays true. The stale `completionResponse` then renders
   through the scroll region until the parse lands and the card reclaims the slot. The
   auxiliary height policy (the `recordShowsGenericCompletionFallback` →
   `recordPresentation` → `pickerVisible` chain near the `RecordCardHeightPolicy` call)
   has the same order.
2. **Mac: record keys bypass a visible link list.** In
   `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, when `recordCardActive` is
   true, Return routes to `.recordReturn` unless a card list or the `record_target`
   generic fallback is visible. Tab routes to `.recordTab` unless a card list is
   visible. Even a rendered `[[` list would be skipped: Return would try to create the
   record with `[[sa` in it, and Tab would jump to a field.
   `CaptureRecordCompletionSurface` in
   `Sources/CaptureCore/CaptureRecordPresentation.swift` only recognizes record contexts
   (`isRecordCompletion` in the model), so an ordinary completion is `.none` for the
   card.
3. **bob-cli (decision-dependent): text values swallow link tokens.** In
   `src/native/capture_complete/records.rs`, `Session::complete` runs `value_probe`
   first. That probe claims any caret inside a `key:: value` and returns `record_value`
   (for a plain non-suggest Text field: zero rows), so `engine::build_result_inner`
   never reaches its `capture_links` (`[[`) or `capture_task_wikilinks` (`((`) branches.
   This is documented as "Typed record field values own `%`, `#`, `@`, `~`, URLs, and
   `((` before task pickers" in `docs/capture.md` (Guided fields area) and the
   `capture-complete` long help in `src/native/capture_complete/cli.rs`.

## Rule to implement

> On a record draft, a live editor assist owns the Record Card's slot. This covers the
> Task Wikilink picker (and any other picker), the reopen chip, the Add block ID /
> relation-label / Pomodoro-name prompts, and any current non-record completion list
> (`wikilink_note`, `wikilink_heading`, `wikilink_block`, `task_type`, routes, …). Keys
> go to that assist. The card comes back unchanged once the assist closes (accept,
> Escape, cancel, or caret moves away). Record lists (`record_field` / `record_value` on
> the card, `record_target` generic fallback) keep today's behavior.

This keeps the Mac a thin client (`decisions:mac-capture-is-a-thin-client`). It decides
placement only from bob's `context`, `record` metadata, and picker descriptor, and never
parses link grammar.

## Part A — bob-cli (only when `field_value_links = prose_links`)

> [!decision] field_value_links = values_own Skip Part A entirely. Leave bob-cli code
> and docs unchanged, and leave out the field-value checks in Part B's tests.

1. `src/native/capture_complete/records.rs`, `Session::complete`: after `value_probe`
   returns `Some(probe)`, check two things:
   - the probed field is free text: `FieldKind::Text` with `suggest: false`. Unknown
     keys already default to that.
   - the caret sits inside an open link token, found with the same detectors
     `engine::build_result_inner` uses:
     `capture_links::completion_field_at(raw, cursor, <route label>)` for
     `[[`/heading/block, and
     `capture_task_wikilinks::completion_field_at(raw, cursor, <route label>)` for `((`.
     Respect the same `task_type_completion_at` guard the engine applies to `((`.

   If both hold, record `set_files_opened(...)` as the other exits do and return
   `Ok(None)`, so the ordinary engine serves the `wikilink_*` response with no `record`
   metadata. Extract one small helper, for example `link_token_owns_value_caret`, rather
   than duplicating the detection.

2. Keep everything else as it is:
   - Choices, Date, Number, URL, and Link fields keep their value menus. Link fields
     already offer their own link rows.
   - Suggest-enabled Text fields keep their folder-value menu, which already matches
     link-shaped values such as `[[nj_towns#^town|westfield]]`.
   - A caret in a text value but outside an open link token (including just after a
     closed `]]` or `))`) still gets `record_value`.
   - `-n/--next-missing-field` and `-F/--record-field` stay explicit record requests and
     are unaffected.
   - Parse-side ownership of `%`, `#`, `@`, `~`, and URLs inside values does not change.
     This is a completion-only change.
3. Tests in `src/native/capture_complete/tests/records.rs`. Add a plain non-suggest text
   field to a test album; prefer a new small album over touching the shared restaurant
   schema. Cover:
   - `- claim:: see [[No` → `wikilink_note` with rows and no `record`.
   - `[[note#` → `wikilink_heading`.
   - `- claim:: see ((` → `wikilink_task` with a `picker`.
   - Each of the above on an `@album/<stem>` update draft as well.
   - Unchanged `record_value` for: a suggest field with `[[`, a choices field with `[[`,
     a caret after a closed link, and a plain caret in text.
4. Docs:
   - Reword the ownership sentence in `docs/capture.md` and the matching long-help text
     in `src/native/capture_complete/cli.rs`. Free-text (non-suggest) values hand an
     open `[[`/`((` token to ordinary note/heading/block/Task Wikilink completion; other
     kinds still own their values.
   - Run `just docs-gen` so `docs/reference/cli/capture-complete.md` regenerates, then
     run `just docs-check`.
   - Do not edit `Cargo.toml` versions or `CHANGELOG.md`. Use a non-breaking
     `feat(capture): …` subject, since clients already handle `wikilink_*` contexts.
5. Gate: `just fmt`, `just lint`, `just test`. Re-run the probe table above against the
   built binary: the field-value rows should now report `wikilink_note` /
   `wikilink_task`.

## Part B — Bob Mac Capture (linked `bob-mac-capture` repo)

Open it with `/sase_repo` and read its `AGENTS.md` first: compile rules, formatting, CI
watching.

1. **CaptureCore surface** (`Sources/CaptureCore/CaptureRecordPresentation.swift`):
   - Add an `editorList` case to `CaptureRecordCompletionSurface`: a current, visible,
     non-record completion list on a record draft.
   - Extend `make(...)` with an `editorListVisible: Bool = false` parameter, so existing
     callers and tests keep compiling.
   - Return `.editorList` when `!isRecordCompletion && editorListVisible`.
   - Make `showsGenericFallback` true for `.editorList` too, so the existing card-slot
     `CompletionList` branch, its height policy, alias-tip suppression, and the Return
     acceptance in both the router and `recordReturn` / `finishRecordReturn` cover it.
   - Add a separate predicate for Tab acceptance (for example `tabAcceptsVisibleList`:
     card lists plus `.editorList`, not `record_target`).
   - Update the doc comment on the enum.
2. **Model** (`Sources/BobMacCapture/CapturePanelModel.swift`):
   - `recordCompletionSurface` passes
     `editorListVisible: recordCardActive && completionVisible && !pickerVisible`.
     Ordinary drafts, which have no active card, never take the record branch.
   - Expose `recordEditorListVisible`.
   - Expose `recordCardYieldsToEditorAssist`, defined as `recordCardActive` and any of:
     `pickerVisible`, `taskIDPromptVisible`, `relationLabelPromptVisible`,
     `pomodoroNamePromptVisible`, `pickerChipVisible`.
   - The editor list has to stay mounted through the pending-analysis window
     (`recordPresentation == nil`, `recordInteractionPending == true`). It must not
     flash or jump between the 430-pt scroll-region list and the full-width slot list on
     each keystroke.
3. **View** (`Sources/BobMacCapture/CapturePanelView.swift`):
   - In `auxiliaryRegion`, rank `model.pickerVisible` (`CapturePickerCard`) above both
     record branches.
   - Gate the `RecordCardView` branch on `!model.recordCardYieldsToEditorAssist`, so
     prompts and the chip fall through to `auxiliaryScrollRegion`.
   - Mirror the same order in the auxiliary height policy.
   - Leave `hasAuxiliaryContent` and the footer chain alone. The footer already shows
     picker hints while a picker is up.
4. **Keys** (`CaptureKeyCommandRouter.swift`, plus the `CaptureKeyRoutingContext` built
   in `CapturePanelController.swift`):
   - Add `recordEditorListVisible` to the context.
   - On an active card with assist available, plain Tab accepts when
     `recordCompletionListVisible || recordEditorListVisible`. It opens the picker from
     the chip when `pickerChipVisible`. Otherwise it stays `.recordTab`.
   - Shift-Tab, the `record_target` fallback's Tab (first missing field), and Return
     without a selected row (`.recordReturn`) stay unchanged.
   - Arrow and Control-N/P keys already follow `completionVisible`.
   - In the controller's `.escape` chain, dismiss a visible picker chip before
     `dismissRecordCard()`. The order becomes completion → chip → card → close.
5. **Fixtures**: extend `Scripts/generate-record-fixtures.sh`, which builds real-bob
   fixtures in an isolated vault with an explicit `BOB_BIN`. Add:
   - a note-bullet `[[` on a create draft
   - a note-bullet `[[` on an `@eat/<stem>` update draft
   - a note-bullet `((` on a create draft, with at least one linkable task in the
     scratch vault if the existing notes have none

   Then regenerate. If any existing fixture's diff changes, review it and keep it
   truthful. Update `Tests/Fixtures/record-fixtures-README.md`. Never hand-write bob
   JSON, apart from the existing documented older-Bob case. Part A needs no Mac fixture:
   a field-value link reaches the Mac as the same `wikilink_*` response.

6. **Tests**:
   - `Tests/CaptureCoreTests/CaptureRecordTests.swift`: surface cases for `.editorList`
     and its predicates. Record contexts are unchanged.
   - `Tests/BobMacCaptureTests/CaptureRecordInteractionTests.swift` (router):
     - Tab with an editor list → `.acceptCompletion`.
     - Return with editor list + selection → `.acceptCompletion`; without a selection →
       `.recordReturn`.
     - Tab with the chip on a card → `.openPickerFromChip`.
     - The existing `record_target` fallback and non-record Tab tests stay green.
   - `Tests/BobMacCaptureTests/CaptureRecordPanelTests.swift` (fake-bob panel), on both
     create and update drafts:
     - A `[[` note bullet shows the list in the card slot.
     - Return inserts bob's replacement and runs no mutating `bob capture`.
     - The card returns after accept and after Escape (Escape closes the list first,
       then the card).
     - `((` mounts the picker with `recordCardYieldsToEditorAssist` true; cancelling
       restores the card.
     - The list stays visible across a keystroke that restarts analysis.
7. **Docs** (`docs/capture-previews.md` Record Card section, `docs/keyboard.md` "Record
   cards"):
   - State the yield rule and the key changes.
   - Note that `((` uses the normal Task Wikilink picker on record drafts.
   - Run `just docs-check`.
8. **Gate**, per that repo's `AGENTS.md`:
   - Run `just portable-check` on a host with Swift.
   - Do the best-effort mac compile of a scratch copy (`bundle.sh`, then
     `just typecheck-tests`; never Bryan's checkout or `just install`).
   - Run `Scripts/macos-ci-check` before landing.
   - After pushing, watch the exact-SHA macOS run to green with `/sase_monitor`.
   - Format only changed lines.
   - If the mac is offline, say the macOS targets are unverified.

## Out of scope

- The card-to-spinner flicker on keystrokes that do not involve an assist (the `(` frame
  in the screencast). Only the assist path must not flicker.
- The Obsidian Record Card's Text stage in bob-plugins (a plain textarea with no `[[`
  suggester).
- Escaped brackets in the album index callout for linked field values.

## Acceptance

- On the screencast draft, `- See the [[sa` shows a live note list in the Record Card's
  slot. Return or Tab inserts bob's link, and the card returns with the record still
  unsaved.
- `- See the ((` shows the Task Wikilink picker. An ID-less pick shows Add block ID.
  Escape or cancel returns to the card.
- The same holds on `@sase_blog_zettel/ai_slop` (edit) drafts.
- With `prose_links`: `- claim:: see [[sa` and `- claim:: see ((` complete the same way.
  Choices/suggest/date fields keep their value menus.
- Record field/value lists, `record_target` rows, Tab-to-next-field, and
  Return-to-create behave exactly as before when no editor assist is visible.
