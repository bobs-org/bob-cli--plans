---
tier: tale
title: Typing ~ anywhere in a relation picker's filter flips the link direction
goal:
  In Bob Mac Capture's relation picker and Obsidian's Links / Linked-from stages, a ~
  typed at any caret position flips Links to and Linked from in place, keeping the
  filter text.
size: medium
proposed_by: bbugyi200.athena.152
proposed_with: claude/opus
create_time: 2026-10-28 08:41:26
status: wip
---

# Plan: `~` anywhere in a relation picker's filter flips the link direction

## Problem

Bob Mac Capture's relation picker (opened by `~` in a capture draft) shows the subtitle
**any task · ~ flips direction** and a `~ Flip direction` key hint, but `~` only flips
while the filter is **empty**. After typing `foot`, a `~` lands in the filter as text
(`foot~`), matches nothing, and the direction stays **Links To** (Bryan's screenshot:
chip `~ Links To`, filter `foot~`, "0 of 4119", "No tasks match “foot~”").

The Obsidian Links / Linked-from stages (Task Card `l` / `⇧L`, `Ctrl+Shift+L` /
`Ctrl+Shift+Alt+L`, nav `api.relations.openStage`) have **no** in-stage direction flip
at all; a `~` typed there is just filter text.

## Goal

In both clients, typing `~` at **any** caret position in a relation picker's task filter
flips the direction (Links to ↔ Linked from) in place:

- The typed `~` never enters the filter. The filter text and caret stay exactly as they
  were, and the list re-filters against the new direction's pool with that same text.
- Typing `~` again flips back. Repeated flips always alternate and never stack sigils.
- Nothing is written to the vault. A flip only changes which picker is showing.

Already true, and must stay true: Mac Backspace on an empty `~~` filter flips back,
Backspace on an empty `~` filter removes the token, the `~ Flip direction` hint, and the
subtitle copy.

## Non-goals

- The relation **label** card / Obsidian relation-label stage (Shift-Return or `|`) is
  out of scope. Its `~` handling stays as it is today.
- Pasted text and IME/dead-key composition that put a `~` into the filter keep today's
  behaviour (a literal filter character). Only a typed `~` keystroke flips.
- No bob CLI code or JSON contract changes. The Mac app still edits only the draft's
  sigil bytes and asks bob for the new completion, exactly as the existing empty-filter
  flip does, so `decisions:mac-capture-is-a-thin-client` still holds.

## Repositories

The work spans three repos. Open the two linked repos with the `/sase_repo` skill
(`bob-mac-capture`, `bob-plugins`) and read each repo's `AGENTS.md` before editing:

1. **bob-mac-capture**: the Swift change, its tests, and its docs.
2. **bob-plugins**: the Obsidian change, its tests, and its docs.
3. **bob-cli** (this repo): one contract sentence and one conformance vector in
   `docs/task-relations.md`.

## Part 1 — Bob Mac Capture

### Current mechanics (read these first)

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, picker routing, `default:`
  branch: a single-character key in `context.pickerOperatorContinuationKeys` returns
  `.continuePickerOperator` only when `context.pickerFilterIsEmpty`. The context already
  carries `pickerSourceIsRelation`, `keyIsRepeat`, and `editorHasMarkedText` (the first
  responder `NSTextView`'s marked text, which covers the filter field's field editor).
- `Sources/BobMacCapture/CapturePanelModel.swift`:
  - `pickerOperatorContinuationKeys` returns `["~"]` for relation sources (empty filter
    only).
  - `continuePickerOperator(_:)` returns early on a non-empty filter. For relation
    sources it calls `flipRelationDirection(insertingSecondTilde: source == .linksTo)`.
  - `flipRelationDirection(insertingSecondTilde:)` guards `filterText.isEmpty`, inserts
    the second `~` at `openPicker.restoreCursor` (or deletes the byte at
    `replacementRange.start + 1`), updates `draftSnapshot`, `replacementRange`,
    `restoreCursor`, and the marker highlight, then
    `scheduleAnalysis(... requestCompletion: true ...)`.
  - `handleCompletionResponse` already sends a `relation_to` ↔ `relation_from`
    completion at the same replacement start to
    `handleRelationLiveFlip(..., keptFilter: openPicker.filterText, ...)`, which
    re-presents the picker **keeping the filter**. That path needs no change.
  - `removePickerTrigger()` uses `flipRelationDirection(false)` for `.linkedFrom`.
- The filter text is local to the picker (`updatePickerFilter` →
  `index.presentation(filter:)`). It is never written into the draft. The draft holds
  only the `~` / `~~` token, plus any query Bob seeded before the picker opened (for
  example `New task ~tar` with replacement `9..<13`).

### Changes

1. **Router.** In the picker `default:` branch, before the continuation-key check, add:
   `context.pickerSourceIsRelation`, `event.characters == "~"`, no
   Command/Option/Control, and `!context.editorHasMarkedText` → return a new
   `CaptureKeyCommand.flipRelationDirection`. A key repeat (`context.keyIsRepeat`)
   returns `.consumeKey` instead, so holding `~` neither flips repeatedly nor types
   tildes. Filter emptiness does not matter. Add the case to the `CaptureKeyCommand`
   enum and dispatch it in `CapturePanelController.swift` next to
   `.continuePickerOperator` (`model.flipRelationDirection()`).
2. **One flip path.** Make the model's flip an internal `func flipRelationDirection()`
   with no direction argument. Remove the relation branch from
   `continuePickerOperator(_:)` and drop `"~"` from `pickerOperatorContinuationKeys`, so
   relation `~` has exactly one route. `removePickerTrigger()` keeps calling the flip
   for `.linkedFrom`.
3. **Flip semantics** in `flipRelationDirection()`:
   - Remove the `filterText.isEmpty` guard. Keep the
     `plainDraft == openPicker.draftSnapshot` guard and the `byte[r.start] == "~"`
     guard.
   - Derive the current direction from the **draft bytes**, not from `picker.source`:
     the token is Linked From when `r.start + 1 < r.end` and `byte[r.start + 1] == "~"`.
     This fixes the race where a second `~` arrives before `handleRelationLiveFlip` has
     re-presented the picker. That `~` would otherwise see the stale `.linksTo` source
     and produce `~~~`.
   - Edit the sigil, not the caret position. To Linked From, insert `~` at
     `r.start + 1`. To Links To, delete the byte at `r.start + 1`. Shift
     `replacementRange.end` and `restoreCursor` by ±1 when they sit at or after the edit
     point. Today the code inserts at `restoreCursor`, which corrupts a Bob-seeded query
     (`~tar` would become `~tar~`).
   - Leave `picker.filterText` untouched. The re-present already passes `keptFilter`.
4. **Copy.** No string changes. The subtitle, the action line
   (`↩ link · ⇧↩ link with a relation · ~ flips direction`), the key hints, and the
   accessibility labels already promise this behaviour.

### Tests (`Tests/BobMacCaptureTests/CaptureDependencyPanelTests.swift`)

Update the tests that pin the old empty-filter-only path
(`testRelationPickerInsertsTildeReplacement`'s continuation-keys assertion,
`testRelationLiveFlipKeepsPickerOpen`, and `testRouterTildeContinuesRelationPicker`),
then add:

- Router: `~` (keyCode 50, Shift) with `pickerSourceIsRelation: true` returns
  `.flipRelationDirection` for both an empty and a non-empty filter. With `keyIsRepeat`
  it returns `.consumeKey`. With Command, Option, or Control, or with
  `editorHasMarkedText`, it does not flip. On a non-relation picker with a non-empty
  filter it returns `nil` (types normally).
- Model, live fake bob: on `New task ~` with filter `foot`, a flip makes the draft
  `New task ~~`. The picker stays visible, the source becomes `.linkedFrom`, and
  `filterText == "foot"`. A second flip returns to `New task ~`, `.linksTo`, with the
  filter still `foot`.
- Model: on `New task ~tar` (replacement `9..<13`, filter `tar`), a flip makes the draft
  `New task ~~tar`, with the tilde inserted after the sigil and not at the caret.
- Model: two flips in a row, before the first re-present lands, end at `New task ~` /
  `.linksTo` (never `~~~`).
- Backspace on an empty `~~` filter still flips back. Backspace on an empty `~` filter
  still removes the token.

Add `Tests/Fixtures/fake-bob` dispatch entries for any new drafts (`New task ~~tar` and
friends), following the existing `relation-complete-*.json` fixtures.

### Docs

- `docs/relationship-pickers.md`, Relation Picker section: replace "Typing a second `~`
  at the same replacement start live-flips …" and "Empty-filter `~` is operator
  continuation, like Complete's `!`." with: typing `~` anywhere in the filter live-flips
  Links To ↔ Linked From; the filter keeps its text and the typed `~` never enters it;
  repeats are ignored.
- `docs/keyboard.md` line ~108: "treat empty-filter `~` as a live direction flip"
  becomes "treat `~` anywhere in the filter as a live direction flip".

### Verification (per `AGENTS.md`)

- `just portable-check` and `just docs-check`. Swift is available on athena.
- `Sources/BobMacCapture*` and `Tests/BobMacCapture*` changed, and portable-check cannot
  type-check them. Compile a scratch copy on the mac as `AGENTS.md` describes (rsync
  without `.build`/`.git`, `bundle.sh`, `just typecheck-tests`, then delete the copy;
  never use Bryan's checkout or `just install`). If the mac is offline, say in the final
  report that the macOS targets are unverified. CI runs the tests.
- Format only the changed lines (`.swift-format`, 100 columns).

## Part 2 — Obsidian (bob-plugins, `plugins/bob-navigation-hotkeys`)

This plugin uses the fragment build. Edit `src/` fragments only, keep each hand-edited
fragment at or below 1000 lines, run `npm run build`, and never hand-edit `main.js`.

### Current mechanics (read these first)

- `src/426-picker-relation-stage.js`: `showRelationStage(direction)` sets
  `relationStageDirection`, rebuilds the stage through
  `plugin.buildVaultRelationStage({... direction})`, resets marks (`clearPendingBatch`,
  `resetLocalTaskMarks`), applies list options (title `🪢 Links · …` /
  `🪢 Linked from · …`), and calls `renderAll({ clearQuery: true })`, which wipes the
  filter. It then kicks off `refreshVaultRelationStage` /
  `refreshVaultLinkedFromIncoming`. Both are stale-guarded by
  `modal.vaultStage === stage` plus a request id, and `applyVaultStageItems` also checks
  `seed === stage.refreshOf`, so a rebuilt stage strands in-flight refreshes for the old
  direction. `getRelationStageFooterHints()` lists the stage keys.
- `src/450-picker-keydown.js`: `handleKeydown` dispatches stage keys. The Links-stage
  `⇧↵` branch keys off `this.relationStage && this.stage === "value"`.
- `src/312-task-card-relations.js`: `taskCardLinkedFromAvailabilityReason` refuses
  non-single sessions with `"Linked from needs a single task"` (`N<Ctrl+Shift+Alt+L>` is
  refused because a reverse batch cannot pick one target).
- `src/200-filtered-picker-modals.js`: the base `input` listener re-renders results.
  `getQuery()` trims and lowercases `inputEl.value`.

### Changes

1. **Keydown.** In `handleKeydown`, while
   `this.relationStage && this.stage === "value"`, an `event.key === "~"` with no
   Ctrl/Meta/Alt, `event.isComposing !== true`, and `event.keyCode !== 229` calls
   `preventDefault()` and `stopPropagation()`. Then:
   - If `event.repeat === true`, stop there: the key is consumed and nothing flips.
   - Otherwise, unless a commit is in flight (`this.opening`), call
     `this.flipRelationStageDirection()`. The relation-label stage and the block-ID
     prompt are other stages and are not intercepted.
2. **`flipRelationStageDirection()`** (in `426-picker-relation-stage.js`):
   - Next direction is `"from"` when the current one is `"to"`, and vice versa.
   - Flipping to `"from"` in a non-single session (`this.taskSession` with
     `type !== "single"`): show `new Notice("Linked from needs a single task")` with the
     same text as `taskCardLinkedFromAvailabilityReason`, keep the Links stage, its
     marks, and its filter, and return `false`.
   - Otherwise save `inputEl.value`, `selectionStart`, and `selectionEnd`. Call
     `showRelationStage(next, { keepQuery: true })`. Add that option so the rebuild
     skips `clearQuery` and its initial items use the current query. Then restore the
     value and selection, and re-render so the new pool filters with the same text.
   - The rebuild clears Tab marks and resets the selection to the first row. Marks
     belong to the old direction's CURRENT/RESULTS pool and cannot carry over
     meaningfully. Mention this in the docs.
3. **Footer hint.** `getRelationStageFooterHints()` adds
   `{ keys: ["~"], label: "flip direction" }` after the `⇧↵` hint, only in single-task
   sessions (where the flip can succeed). Update any test that pins the exact hint list.
4. Leave the stage subtitle and consequence strip copy as they are. They already follow
   the direction.

### Tests (`scripts/test-navigation-relation-links-stage.cjs`, cite LG16)

- A `~` keydown on a Links stage with query `foot` calls `preventDefault`. Afterwards
  `relationStageDirection === "from"`, the title is `🪢 Linked from · …`,
  `inputEl.value === "foot"`, and the visible items are the Linked-from pool filtered by
  `foot`.
- With the caret mid-query (`fo|ot`), the value stays `foot` and the caret stays at 2.
- A second `~` flips back to `"to"` with the query kept.
- A counted session shows the notice `Linked from needs a single task`, stays `"to"`,
  and keeps the value and marks.
- Ctrl/Alt/Meta+`~`, `isComposing`, keyCode 229, and `this.opening` do not flip.
  `event.repeat` is consumed without flipping.
- Flipping clears marks.
- A refresh started for the old direction that resolves after the flip does not replace
  the new stage.
- The footer shows `~ flip direction` in single sessions and omits it in counted
  sessions.
- A `~` keydown in the relation-label stage is not intercepted.

### Docs

- `docs/task-relations.md`, "Linking from the Task Card": add that typing `~` anywhere
  in the Links / Linked-from filter flips the stage in place, keeping the filter text
  and clearing marks; counted sessions refuse the flip to Linked from with
  `Linked from needs a single task`.
- `docs/task-card.md`: add `~` to the key table entry for the relation stages.
- `plugins/bob-navigation-hotkeys/manifest.json` description: extend the relation clause
  ("… Shift+Enter edits a relation label …") with "`~` in the filter flips direction".
  Then run `npm run docs:gen` so the generated inventory in
  `docs/bob-navigation-hotkeys.md` matches.

### Verification (per `AGENTS.md`)

`npm run build`, `npm test` (which includes `build:check` and the docs inventory check),
`npm run docs:check`, then `bob plugins sync` to deploy to the vault.

## Part 3 — bob-cli contract (`docs/task-relations.md`)

That file is the contract every implementation cites, and new gestures get a vector in
the same change.

- Intro paragraph: after "from the Obsidian Task Card (`l` / `⇧L`)", add one sentence:
  in either client's relation picker, typing `~` anywhere in the task filter flips Links
  to ↔ Linked from in place and keeps the filter text.
- §15 "LG — Obsidian gestures": add
  `| LG16 | ~ typed anywhere in a Links / Linked-from filter | the stage flips direction in place; filter text and caret kept, the ~ never typed, marks cleared; a counted session refuses the flip to Linked from with "Linked from needs a single task" |`
  (format the tilde in code spans like the neighbouring rows).
- Run `just docs-check`. Do not touch `Cargo.toml` or `CHANGELOG.md`.

## Done when

- Mac: in the screenshot scenario (`… ~`, then filter `foot`, then `~`), the chip reads
  **Linked From**, the draft token is `~~`, the filter reads `foot`, and the rows are
  Linked From candidates matching `foot`. Another `~` returns to Links To with `foot`
  kept.
- Obsidian: the same sequence in the Links stage switches to `🪢 Linked from · …` with
  `foot` kept, and back again.
- All three repos' checks listed above pass. The plugins are synced. Any unverified
  macOS compile is reported.
