---
tier: epic
title: Large fuzzy Active Task Picker for `^` in Bob Mac Capture
goal: 'Typing `^` as a capture item in Bob Mac Capture opens a large, keyboard-first
  Active Task Picker. It lists every In Progress and Next task grouped by today''s
  Pomodoro plan, filters them instantly with fuzzy matching, and inserts the chosen
  `route:block-id` reliably. It never shows red incomplete-marker errors while you
  pick.

  '
phases:
- id: core
  title: Fuzzy matcher and picker presentation engine (CaptureCore)
  depends_on: []
  size: medium
  description: 'core: add a pure, Foundation-only fuzzy matcher, the task display-text
    parser for code spans and wikilinks, and the grouped/filtered picker presentation
    with navigation helpers. Ship thorough CaptureCore unit tests; the app''s behavior
    does not change.'
- id: picker-flow
  title: Picker state machine, keyboard routing, focus, and a functional picker view
  depends_on:
  - core
  size: medium
  description: 'picker-flow: route `active_task` completion into a modal picker state.
    This covers open, suppress, and reopen rules, full-snapshot fetch, accept and
    accept-and-submit, the two-stage Escape, trigger removal on Backspace, and the
    reopen chip. Add picker key routing, an AppKit-owned filter field, and live-preview
    suppression for an incomplete `^`. Ship a plain but fully working picker view,
    fake-bob fixtures, tests, and README behavior docs.'
- id: picker-design
  title: Beautiful picker card, sizing, accessibility, docs, and macOS verification
  depends_on:
  - core
  - picker-flow
  size: medium
  description: 'picker-design: replace the functional view with the final design,
    including the large card, pinned Pomodoro headers, rich rows, detail strip, empty
    states, chip, and key-hint footer. Add the fixed-height sizing policy, accessibility
    announcements, README visuals, rendered-image review, and macOS build, test, and
    lint verification.'
proposed_by: bbugyi200.apollo.bob-cli-2f.3.w1
create_time: 2026-09-28 18:28:10
status: done
bead_id: bob-cli-2g
---

- **PROMPT:** [prompts/202609/mac_active_task_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/mac_active_task_picker.md)
- **BEAD:** [bob-cli-2g](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2g/README.md)

# Plan: Large fuzzy Active Task Picker for `^` in Bob Mac Capture

## Context

Bob Mac Capture (the linked `bob-mac-capture` repo) is the macOS menu-bar front end for
`bob capture`. Typing `^` as the whole first token of a capture item asks
`bob capture-complete` for the `active_task` context: In Progress (`/`) and Next (`*`)
tasks that have block IDs, in Bob's ledger order. Queued tasks come first in
Pomodoro-entry and link order, then unqueued In Progress, then unqueued Next. Accepting
one inserts `route:block-id`, which turns the item into a `^route:block-id` Pomodoro
link. `#name`, `=<X>`, and `=x` suffixes still work after the insert.

Today the Mac app shows these candidates in the generic inline completion list (see the
user's screenshot of the current state). That list has these problems:

- It is a 430pt-wide, five-row popover. Task text truncates after about 30 characters
  ("Review and close/work all open bea…"), and the list crowds out the editor.
- Every row repeats the same "ACTIVE TASK" label and "Next & In Progress" section, and
  the monospaced text wastes width.
- Filtering is only Bob's prefix/substring ranking of the text typed after `^` in the
  draft. It is not fuzzy, and each keystroke needs a subprocess round trip.
- The live preview runs `bob capture --dry-run` on the lone `^`. It fails with
  "incomplete Pomodoro link; finish the marker `^<route>:<block-id>`", and the panel
  shows that error twice in red with Retry and Copy Diagnostic buttons while you are
  still picking. This is the ugliest part of the screenshot.

Real data profile (the user's vault on 2026-09-28,
`bob capture-complete -c 1 -f json -- '^'`):

- 72 candidates, all queued, across about 19 open Pomodoro entries: SASE, DECKS, READ,
  MISC, FINAL, RENAME, BOB, TOOL, CLEANUP, REMOTE, SERVICE, NEW FEATURES, SUDO, FAST
  TESTS, QUEUE, SCHEDULE, AUDIT MEMORY, GATES, and LATER, with 25 tasks in LATER.
- The same Pomodoro name can appear twice as separate entries: SASE at ledger lines 53
  and 68.
- No entry had a `time_range` or `is_current: true`. The design must still render those
  cleanly when they exist; `time_range` looks like `0900-0930`.
- `section` was `"Next & In Progress"` for every task, so it is noise in a row.
- Text runs 16–112 characters and often contains `` `code` `` spans, `[[ref/chat/…]]`
  wikilinks, curly quotes, and a trailing `!`.
- Routes: `sase`, `bob`, `sase_clean`, `sase_remote`, `sase_memory`, `sase_art_links`,
  `sase_pager`.
- `bob capture-complete -c 1 -- '^dee'` (cursor right after `^`) returns the full
  unfiltered list with `replacement {1,4}`. At cursor 4 it returns Bob's filtered list,
  which was zero rows here. A lone `^` parses as `mode: incomplete`,
  `needs: ["active_task"]`.

Repository access: open the linked repo with
`sase repo open bob-mac-capture -r "<reason>"`. If that fails because the linked
checkout's primary workspace directory is missing (a known condition on some hosts),
`sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` gives an audited checkout.
Use the printed path for all reads and edits. No `bob-cli` code or contract change is
needed: the existing version-1 `capture-complete` JSON already carries every field the
picker needs.

Relevant existing code in `bob-mac-capture`:

- `Sources/BobMacCapture/CapturePanelModel.swift`: the analysis pipeline. It handles
  `scheduleAnalysis`, `shouldRequestCompletion`, completion accept,
  `suppressedCompletionAcceptanceDraft`, the inline prompts that lock the editor, and
  live preview.
- `Sources/BobMacCapture/CapturePanelView.swift`: the panel layout,
  `CanceledDraftStashPicker` and its height policy (the precedent for a modal picker in
  the auxiliary region), `CompletionList`/`CompletionRow`, and `CapturePanelFooter`.
- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift` and
  `CapturePanelController.swift`: the local key monitor, command routing, and
  prompt-field focus repair.
- `Sources/BobMacCapture/BlockIDField.swift`: the AppKit-owned first-responder text
  field pattern to copy for the filter field.
- `Sources/CaptureCore/CaptureModels.swift`: the `CaptureCompletionCandidate` fields
  `route`, `blockID`, `statusSymbol`, `statusName`, `text`, `section`, `taskRef`,
  `line`, and `pomodoro` (`line`, `name`, `timeRange`, `isCurrent`).
- `Sources/BobMacCapture/CaptureEditorPalette.swift`: the single semantic color palette.
- `Tests/Fixtures/fake-bob`: the scripted `bob` used by `CapturePanelModelTests`. Its
  `^`, `^dee`, and `^sase:deep-fix` completion fixtures already exist.

## Design decisions (lead)

1. **Keep one window.** The picker is a modal mode of the existing capture panel, not a
   second `NSWindow`. When it opens, the capture panel grows (instantly, top edge
   anchored, through the existing content-metrics sizing path) into a large elevated
   picker card: up to 11 visible rows, full content width, roughly 520pt tall. That is
   about 3× the area of today's list. A second floating panel would need focus handoff
   between two non-activating panels, a second key monitor, hotkey and hide
   coordination, and child-window ordering. That is exactly where reliability bugs live.
   The single-window modal pattern already works for the stash picker and the Add block
   ID and Name Pomodoro prompts, including AppKit-owned field focus and orphaned-focus
   repair.
2. **Bob supplies the snapshot; the app ranks it.** Bob remains the only source of the
   candidate set, their order, each `replacement`, and the draft replacement range. The
   picker fetches one full snapshot when it opens and filters it locally with a fuzzy
   matcher in `CaptureCore`. Local filtering makes each keystroke instant and
   flicker-free, and filter text never touches the draft. The app already ranks the
   cached `capture-targets` route list locally, so this is precedent, not a new
   exception. The README states this explicitly.
3. **The picker owns the `^` part.** Whenever you edit the `route:block-id` part of a
   `^` item and it is not already an exact candidate, the picker opens. Escape
   suppresses auto-open for that token. A compact "Browse active tasks ⇥" chip then
   offers a one-key reopen, and caret-only moves show the same chip instead of popping
   the picker. Fast typing is safe: text typed before the picker appears seeds the
   filter.
4. **Insert, don't invent.** The picker only inserts `route:block-id`. Return inserts
   and returns to the editor, where the live preview and footer show the link and
   **Link**/**Start** action. Command-Return inserts and captures in one step. The app
   never synthesizes the `#name`, `=<X>`, or `=x` grammar; you type those after the
   insert, and a detail-strip hint teaches them.
5. **An incomplete `^` is a state, not an error.** When `capture-parse` reports `needs`
   containing `active_task`, the app skips the doomed live dry run and shows a calm
   status line instead of red errors.
6. **Out of scope:** `bob-cli` changes; changing inline completion for other contexts
   (`@route:`, `@route+`, `#name`, wikilinks); a second window; persisting the filter
   between openings; multi-select; opening tasks in Obsidian from the picker.

### UX specification (final state after `picker-design`)

```text
┌ capture panel (grows while picking) ─────────────────────────────────────────┐
│ ^                                                   (editor, dimmed, locked) │
│ ╭────────────────────────────────────────────────────────────────────────╮   │
│ │ [^ Active Tasks]  Filter by task, note, ^id, or Pomodoro…       5 of 72 │   │  filter bar
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │ 1  SASE                                                        9 tasks │   │  pinned header
│ │  ◐  Read and act on core_schema_skew_outage_recovery_ux  sase:recovery-p…│   │
│ │  ◉  Bug bash and improve sase TUI command-mode panel       sase:tui-cli │   │
│ │ 2  DECKS                                                       4 tasks │   │
│ │  ◐  Add support for “card blocks”                      sase:card-blocks │   │
│ │ …                                                                      │   │
│ ├────────────────────────────────────────────────────────────────────────┤   │
│ │ ◐ Read and act on core_schema_skew_outage_recovery_ux!                  │   │  detail strip
│ │ In Progress · ▣ sase.md › Next & In Progress · Queued in SASE (#1)      │   │
│ │                                          ↩ inserts ^sase:recovery-panel │   │
│ ╰────────────────────────────────────────────────────────────────────────╯   │
│ ↑↓ Move   ↩ Insert   ⌘↩ Insert & Capture   esc Clear / Cancel   (key hints)  │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Empty filter (grouped).** Sections follow Bob's order. There is one section per
  queued Pomodoro entry, keyed by `pomodoro.line`, in first-appearance order. Each
  header shows a 1-based ordinal, the name (or "Unnamed Pomodoro"), the formatted time
  range `HH:MM–HH:MM` when present, a pink "NOW" pill when `isCurrent`, and a task
  count. After those come "In Progress" and "Next" sections for unqueued tasks,
  subtitled "Not in a Pomodoro". Any other status goes in a final "Other" section. Rows
  in grouped mode omit the Pomodoro chip because the header already says it.
- **Non-empty filter (flat).** One ranked list with no headers, best match first; ties
  keep Bob's order. Rows show a small pink Pomodoro chip. Matched characters are
  semibold and accent-tinted in the text and the locator.
- **Row (34pt, single line).** A status glyph (In Progress: `circle.lefthalf.filled`,
  orange; Next: `circle.inset.filled`, blue; other: `circle.dashed`, secondary). Then
  the display text: proportional body font, code spans monospaced on a faint fill with
  backticks hidden, wikilinks accent-tinted with brackets hidden, tail truncation. The
  trailing locator is `route:block-id` in caption monospaced: route in the route color,
  block ID in the block-ID color, middle truncation, at most about 38% of the row width.
  The selected row gets an accent fill (0.16, or 0.30 with Increase Contrast); hovered
  rows get a faint fill; a click selects and inserts.
- **Detail strip (selected task).** The full text wrapped to two lines with highlights.
  A metadata line: status name; note kind icon plus label from the capture-targets cache
  (`sase.md`, falling back to `route.md`); ` › section` when present; Pomodoro wording
  ("Now · BUGS 09:05–09:30", "Queued in SASE (#1)", "Queued in an unnamed Pomodoro",
  "Not in a Pomodoro"). A trailing `↩ inserts ^route:block-id` in palette colors. A
  second hint appears after the first use: "Then type #name, = to start, or =x to
  close". With no selection, the strip shows the empty-state guidance.
- **Empty states.** No active tasks: "No In Progress or Next tasks — a task needs `[/]`
  or `[*]` and a `^block-id` to appear here." No matches: "No active tasks match “xyz” —
  Esc clears the filter." Bob warnings show as one orange caption line at the bottom of
  the list ("+N more" when there are several) and never block picking.
- **Height is fixed while open.** The visible-row budget is computed once at open from
  the grouped snapshot (rows plus headers, clamped to 4…11). Filtering never resizes the
  panel. On short screens the card shrinks to its minimum (filter bar, three rows,
  detail strip) and the list scrolls.

### Keyboard while the picker is open

| Key                                                        | Action                                                                                                                              |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Printable keys, Cmd-A/C/V/X/Z, Ctrl-A/E, arrows Left/Right | Native editing of the filter field                                                                                                  |
| Return / keypad Enter, Tab                                 | Insert the selected task's `route:block-id`, close the picker, and return to the editor with the caret after the insert             |
| Command-Return                                             | Insert, then capture (the same as pressing Return in the editor afterward)                                                          |
| Shift/Option-Return, Shift-Tab                             | Consumed (no newline, no outdent)                                                                                                   |
| Down / Ctrl-N / Ctrl-J                                     | Next row (wraps)                                                                                                                    |
| Up / Ctrl-P / Ctrl-K                                       | Previous row (wraps)                                                                                                                |
| Page Down / Page Up                                        | Move one page (visible budget − 1), clamped                                                                                         |
| Cmd-Up or Home / Cmd-Down or End                           | First / last row                                                                                                                    |
| Escape / Ctrl-[                                            | Filter non-empty: clear it. Filter empty: cancel (draft unchanged, caret restored, auto-open suppressed for this token, chip shown) |
| Backspace on an empty filter                               | Close and delete the `^` trigger together with any fragment it opened on; the caret lands where the `^` was                         |
| Ctrl-S                                                     | Consumed                                                                                                                            |
| Ctrl-C                                                     | Stash the draft and close the panel (unchanged global behavior)                                                                     |

While the chip is visible and nothing modal is open, Tab, Down, and Ctrl-N open the
picker, and Escape hides the chip. The next Escape closes the panel as before.

## Phase core — Fuzzy matcher and picker presentation engine (CaptureCore)

Everything in this phase lives in `Sources/CaptureCore` (Foundation only, `Sendable`,
`public` as needed by the app target) with tests in `Tests/CaptureCoreTests`. No app
behavior changes in this phase.

1. **`FuzzyMatcher.swift`**
   - `FuzzyQuery(_ raw: String)` splits on whitespace into tokens, drops empty tokens,
     folds each token, and strips one leading `^` from the first token. An empty query
     means "no filter".
   - `FuzzyField(_ text: String)` precomputes, per `Character`, a folded character
     (`String(c).folding(options: [.caseInsensitive, .diacriticInsensitive, .widthInsensitive], locale: nil)`;
     use the result when it is exactly one `Character`, else `c.lowercased()`'s first
     character, keeping indices 1:1) and a boundary bonus. Only the first 256 characters
     are searchable.
   - `FuzzyMatcher.match(token:in:) -> FuzzyFieldMatch?` returns the optimal (not
     greedy) subsequence alignment in O(m·n) with affine gaps. Every token character
     must match in order.
   - Default constants (named, tunable):
     - Per match: +16.
     - Boundary bonus: +10 at field start; +8 after whitespace or a non-alphanumeric
       separator (``-_:/.,;()[]{}|#`'"`` and friends); +6 at a lowercase→uppercase or
       letter↔digit transition in the original text.
     - The first matched character's bonus counts double.
     - Consecutive bonus: +8 when the match directly follows the previous matched
       character.
     - Gap penalty: open −3, extend −1 per additional skipped character, between matched
       characters only. Leading and trailing unmatched text is free.
     - Ties prefer the earliest end position, then the earliest start.
   - Return `score: Int` and `positions: [Int]` (ascending `Character` offsets into the
     original text).
2. **`ActiveTaskDisplayText.swift`.** `ActiveTaskDisplayText(parsing: String)` produces
   `text` (the display string) and `segments` (`kind: .plain | .code | .link`,
   `range: Range<Int>` in `Character` offsets of `text`).
   - Scan left to right.
   - A backtick opens a code span closed by the next backtick. The backticks are dropped
     from the display; an unmatched or empty pair stays literal.
   - `[[` opens a wikilink closed by the next `]]`. The display is the alias after the
     last `|` when present, else the whole inner target; brackets are dropped. An
     unmatched `[[` stays literal.
   - Whichever delimiter appears first wins. No other Markdown is interpreted.
   - All matching runs on `text`, so highlight offsets map directly onto segments.
3. **`ActiveTaskPickerPresentation.swift`.**
   - `ActiveTaskPickerIndex(candidates: [CaptureCompletionCandidate])` is built once per
     snapshot in Bob order.
     - It drops candidates whose `replacement` duplicates an earlier one, since row IDs
       must be unique.
     - It precomputes display text and weighted fields: display text 100%, block ID 90%,
       `route:block-id` 90%, Pomodoro name 70%, route 60%, section 40%.
   - `index.presentation(filter: String) -> ActiveTaskPickerPresentation`:
     - Empty query: `mode = .grouped`, sections per the UX spec, `orderedRowIDs` in
       section order.
     - Non-empty query: `mode = .filtered`, a single header-less section.
       - A candidate is included only when every token matches at least one field; each
         token uses its best weighted field (`score * weight / 100`).
       - The entry score is the sum; sort by score descending, then Bob order.
       - Highlight ranges are the coalesced positions from each token's best field,
         mapped to text, route, or block-ID ranges. Section and Pomodoro-name matches
         raise the rank without highlighting.
   - `Section`: `id`, `kind` (`.pomodoro` / `.unqueuedInProgress` / `.unqueuedNext` /
     `.other` / `.matches`), `title`, `timeRangeText` (`0900-0930` → `09:00–09:30`;
     anything not in `HHMM-HHMM` form is shown raw), `ordinal`, `isCurrent`, `rows`.
   - `Row`: `id` (= `replacement`), `bobIndex`, `status` (`.inProgress` / `.next` /
     `.other(String)`, from `statusSymbol` `/`, `*`, or anything else), `displayText`,
     `textMatchRanges`, `route`, `blockID`, `routeMatchRanges`, `blockIDMatchRanges`,
     `pomodoroChipText` (filtered mode only: "Now · NAME HH:MM–HH:MM" / "NAME" /
     "Planned" / nil), `pomodoroSummary` (the detail-strip wording in the UX spec, using
     the section ordinal), `section`, `insertionText` (`^` + replacement), and
     `accessibilityLabel` (for example "In Progress. Read and act on core schema skew
     outage recovery ux. Note sase, block recovery-panel. Queued in SASE, Pomodoro 1.").
   - Presentation-level values:
     - `totalCount`, `matchCount`, `countText` ("72 tasks", "1 task", or "5 of 72").
     - `emptyState` (`.noActiveTasks` / `.noMatches(query)`, with title and message
       strings).
     - `groupedVisibleRowBudget` (rows + headers of the grouped view, clamped 4…11; 4
       when empty).
   - `ActiveTaskPickerNavigation`: pure helpers over `orderedRowIDs`.
     - `next`/`previous` wrap.
     - `page(by:)` clamps.
     - `first`/`last`.
     - `resolvedSelection(preferred:)` keeps the preferred ID if visible, else the first
       ID, else nil.
4. **Tests.** Build a realistic 12–15 candidate fixture modeled on the real data
   profile. It needs two SASE entries on different lines, a LATER group, one current
   timed entry (`0900-0930`), one unnamed placeholder, unqueued `/` and `*` tasks, code
   spans, wikilinks with and without an alias, and a curly quote. Assert:
   - `FuzzyMatcherTests` (ordering properties, not exact scores):
     - Case and diacritic insensitivity (`cafe` matches `Café`, `TUI` = `tui`).
     - Consecutive beats scattered (`deep` ranks "Fix deep bug" above "Delete every
       empty page").
     - A boundary start beats a mid-word match (`fix` ranks `fix-claude-monitors` above
       `prefix`).
     - `sase:dee` matches `route:block-id`.
     - Multi-token AND semantics and order independence (`q weight` ≡ `weight q`).
     - A leading `^` is ignored.
     - No match returns nil.
     - Positions are correct and ascending; the 256-character cap holds.
   - `ActiveTaskDisplayTextTests`: code spans, aliased and unaliased wikilinks, a mixed
     line, unmatched delimiters stay literal, and segment ranges tile `text` exactly.
   - `ActiveTaskPickerPresentationTests`:
     - Grouped section order, titles, ordinals, and counts; duplicate names stay
       separate entries; NOW pill and time formatting.
     - Unqueued sections and the Other bucket.
     - Filtered ranking where a direct text or ID match outranks a Pomodoro-name-only
       match (`sudo`); `later` surfaces the LATER tasks; `bob` surfaces the `bob` route.
     - Highlight ranges map into display text and locator.
     - Stable tie order, count text, and both empty states.
     - Duplicate-replacement dedupe, row budget clamping, and every navigation helper
       including wrap and clamp edges.
   - Keep `CompletionRowContent` unchanged.

Verification for this phase: see "Verification (every phase)". At minimum
`./Scripts/xcode-swift.sh test --filter CaptureCoreTests` plus format lint must pass on
macOS.

## Phase picker-flow — Picker state machine, keyboard routing, focus, and a functional picker view

This phase makes the feature work end to end with a plain but fully usable view, so no
intermediate commit leaves `^` broken.

1. **State (in `CapturePanelModel.swift` or a new `ActiveTaskPickerState.swift`).**
   - `ActiveTaskPickerState: Equatable`: `draftSnapshot`,
     `replacementRange: CaptureRange`, `restoreCursor: Int`, `candidates` (Bob order),
     `warnings`, `filterText`, `selectedRowID: String?`, `visibleRowBudget` (fixed at
     open from `groupedVisibleRowBudget`), `snapshotIsPartial: Bool`.
   - `ActiveTaskChipState: Equatable`: `draftSnapshot`, `replacementRange`, `cursor`,
     `candidates`, `warnings`.
   - Published: `activeTaskPicker`, `activeTaskPickerPresentation` (recomputed only when
     the snapshot or filter changes; keep the `ActiveTaskPickerIndex` privately), and
     `activeTaskChip`.
   - Private: `activeTaskAutoOpenSuppressedStart: Int?`.
   - Computed: `activeTaskPickerVisible`, `activeTaskChipVisible` (false while any
     modal, prompt, or completion list is visible), `activeTaskFilterIsEmpty`,
     `selectedActiveTaskRow`.
   - Add `CapturePanelFocusTarget.activeTaskFilter`. The editor lock must stay set while
     the picker or either prompt is open; `clearEditorInputLock` checks all three.
2. **Trigger kind.** Add `CompletionTrigger { edit, selection }` to `scheduleAnalysis`.
   `editorTextDidChange` (which also covers the retained-draft re-show path from
   `prepareForPresentation`) and the `+` route commit pass `.edit`;
   `editorSelectionDidChange` passes `.selection`.
3. **Intercept `active_task` completions** in the `scheduleAnalysis` completion branch,
   generation-guarded like today.
   - A response with `context == "active_task"` never sets `completionResponse`, even
     when `candidates` is empty.
   - Compute `part = draft[r.start..<r.end]` and
     `query = draft[r.start..<min(cursor, r.end)]` by UTF-8 byte range.
   - If `part` equals a candidate `replacement` exactly: no picker, no chip.
   - Else if the trigger is `.edit` and `activeTaskAutoOpenSuppressedStart != r.start`:
     open the picker (step 4).
   - Otherwise show the chip.
   - Any `.edit` analysis that ends without an `active_task` result (another context, an
     empty `null`-context result, or no completion requested) clears the suppression and
     the chip. A `.selection` analysis without `active_task` clears only the chip. When
     a non-`active_task` response populates `completionResponse`, clear the chip.
   - While the picker is open, analysis results never modify picker or chip state and
     never populate the inline list. For example, a selection callback fired when the
     editor loses focus must not reset the picker.
4. **Open the picker.**
   - If `cursor != r.start`, first call `captureComplete(draft, cursor: r.start)` in the
     same analysis task. Bob returns the unfiltered snapshot with the same range.
   - Use that result only if its context is `active_task` and its `replacement == r`.
     Otherwise fall back to the first response and set `snapshotIsPartial` (the view
     shows "Showing Bob's matches only").
   - Present only if the analysis generation is still current. Any keystroke in the
     meantime abandons the open, which prevents stale pops.
   - On open:
     - `filterText = query`, `restoreCursor = cursor`, selection = first ordered row.
     - `dismissCompletion()`, clear the chip, set the editor input lock.
     - `requestFocus(.activeTaskFilter)`.
     - Apply `completion.warnings` to state instead of `statusText`.
5. **Filter and selection.**
   - `updateActiveTaskFilter(_:)` recomputes the presentation and resets the selection
     to the first row.
   - Select by ID (mouse), next/previous (wrap), page down/up (by
     `visibleRowBudget − 1`, clamped), first/last. All go through the `core` navigation
     helpers.
6. **Accept.** `acceptActiveTask(id:submitAfterInsert:)` and
   `acceptSelectedActiveTask(submitAfterInsert:)`:
   - Guard `plainDraft == draftSnapshot` and a valid byte range. On mismatch, close the
     picker, unlock and refocus the editor, and set status "Draft changed — reopen the
     task picker" without editing.
   - Otherwise replace the range with `candidate.replacement` and set the cursor to
     `r.start + replacement.utf8.count`.
   - Close the picker, unlock the editor, `requestFocus(.editor)`, set
     `suppressedCompletionAcceptanceDraft`, and
     `setPlainDraft(..., suppressSelectionCallbacks: true)`.
   - `scheduleAnalysis(requestCompletion: false)`, then post a VoiceOver announcement
     "Inserted ^route:block-id".
   - When `submitAfterInsert`, call `submit(openAfterCapture: false)` right after the
     insert. An empty-list accept is a no-op.
7. **Escape, cancel, and trigger removal.**
   - `escapeActiveTaskPicker()` clears a non-empty filter; otherwise it calls
     `cancelActiveTaskPicker()`.
   - Cancel closes the picker, unlocks the editor, restores a collapsed caret at
     `restoreCursor` (suppress the selection callback), focuses the editor, sets
     `activeTaskAutoOpenSuppressedStart = r.start`, and shows the chip from the picker's
     snapshot.
   - `removeActiveTaskTrigger()` runs on Backspace with an empty filter. With an
     unchanged draft and a `^` at byte `r.start − 1`, it deletes `[r.start − 1, r.end)`,
     closes the picker, places the caret at `r.start − 1`, and runs an `.edit` analysis.
     Otherwise it behaves like cancel.
8. **Chip.**
   - `openActiveTaskPickerFromChip()` works only when
     `chip.draftSnapshot == plainDraft`. It clears the suppression and opens with the
     chip's query, using the step-4 snapshot rule. Otherwise it reruns analysis.
   - `dismissActiveTaskChip()` hides the chip without clearing the suppression.
9. **Quiet incomplete `^`.**
   - After `applyParse`, if `parse.needs` or any decoded `items[].needs` contains
     `active_task`:
     - Skip `startLivePreview` and set `previewState = .idle`.
     - Clear the preview results and global destination, and clear `errorMessage`.
     - Set `statusText` to "Pick an active task — press Tab to browse".
     - Still request completion.
   - Suffix states such as `^route:block-id#` (`needs: ["pomodoro_name"]`) keep today's
     behavior.
10. **Lifecycle resets.**
    - `prepareForDismissal`: close the picker without suppression, restore the caret,
      clear the chip.
    - `resetAnalysisState`, `discardDraft`, a successful submit,
      `setProcessClient(nil)`, and stash restore clear the picker, chip, and
      suppression.
    - `toggleStashPicker` and `presentStashPicker` do nothing while the picker is open.
    - Footer Stash/Discard/Preview/Capture buttons are disabled while the picker is
      open.
11. **Routing (`CaptureKeyCommandRouter.swift`).**
    - Add `activeTaskPickerVisible`, `activeTaskFilterIsEmpty`, and
      `activeTaskChipVisible` to `CaptureKeyRoutingContext`.
    - Add commands `acceptActiveTask`, `acceptActiveTaskAndSubmit`, `nextActiveTask`,
      `previousActiveTask`, `pageActiveTasksDown`, `pageActiveTasksUp`,
      `firstActiveTask`, `lastActiveTask`, `escapeActiveTaskPicker`,
      `removeActiveTaskTrigger`, and `openActiveTaskPicker`.
    - Precedence: stash picker > task-ID prompt > Pomodoro-name prompt > active task
      picker > default (chip, completion, editor).
    - The picker branch implements the keyboard table exactly. Key codes: Page Up 116,
      Page Down 121, Home 115, End 119, and the existing arrow codes.
    - Return `nil` for every key the table leaves to native field editing, notably
      Ctrl-A/E, which must not fall through to the editor line-edge commands.
    - In the default branch, when `activeTaskChipVisible`: unmodified Tab, Down, and
      Ctrl-N return `.openActiveTaskPicker`.
12. **Controller (`CapturePanelController.swift`).**
    - Pass the new context fields and perform the new commands.
    - `.escape` hides a visible chip before closing the panel.
    - Before routing, if the picker is visible and the filter field does not hold first
      responder, claim it (find it by accessibility identifier, like
      `findBlockIDField`). The keystroke that raced the open then lands in the filter
      instead of the disabled editor.
13. **`ActiveTaskFilterField.swift`.**
    - An `NSViewRepresentable` over an `NSTextField` subclass copied from
      `BlockIDField`'s first-responder pattern (`requestFirstResponder`, re-claim in
      `viewDidMoveToWindow`, an async retry, release on window removal).
    - Borderless, 15pt system font, placeholder "Filter by task, note, ^id, or
      Pomodoro".
    - Accessibility label "Filter active tasks" and identifier
      `org.bobs.bob-mac-capture.active-task-filter`.
    - `controlTextDidChange` → `model.updateActiveTaskFilter`. Put the caret at the end
      when seeded.
14. **Functional view (`CapturePanelView.swift`).**
    - When `activeTaskPickerVisible`, the auxiliary region shows only a plain picker:
      the filter field, the count text, and a scrollable list rendering the presentation
      sections as simple text headers plus simple rows (status glyph, display text with
      bold matches, locator).
    - Add a detail line with `insertionText`, the empty-state text, and warnings.
    - Size it like today's completion list: fixed viewport height, full width.
    - While the chip is visible, show a small "Browse active tasks ⇥" button that opens
      the picker.
    - `hasAuxiliaryContent` includes the picker and the chip.
    - Keep the existing `CompletionList` for every other context.
15. **Fixtures and tests.**
    - Extend `Tests/Fixtures/fake-bob` so `capture-complete` for draft `^dee` at
      `--cursor 1` returns the full `^` candidate list with `replacement {1,4}`, and add
      a third `^` candidate (unqueued Next).
    - Add or rewrite `CapturePanelModelTests`:
      - Typing `^` opens the picker. No inline completion, focus target
        `.activeTaskFilter`, editor locked, no `capture --dry-run … -- ^` recorded,
        `errorMessage == nil`.
      - `^dee` records a second `capture-complete` at `--cursor 1` and seeds the filter
        with `dee`.
      - A zero-candidate caret response still opens from the full snapshot.
      - A caret-only move into `^` shows the chip, not the picker.
      - An exact part shows neither.
      - Accept inserts `sase:deep-fix`, puts the caret at 14, unlocks the editor, does
        not reopen after the delayed SwiftUI callback, and live preview becomes
        `pomodoro_link`. Update
        `testCaretActiveTaskCompletionAcceptsRouteBlockIDWithoutReopening` to this flow.
      - Command-accept submits.
      - The two-stage Escape, suppression across further edits of the same token, and
        the chip reopening the picker.
      - Backspace on an empty filter removes the `^`.
      - The stale-draft guard.
      - Hide closes the picker and re-show reopens it.
      - Leaving the token clears suppression and the chip.
    - Add router tests in `BobMacCaptureTests`: the full picker table, native
      pass-through (printable keys, Cmd-V, Ctrl-A, Ctrl-E), chip routing, and precedence
      against prompts and the stash picker.
16. **README (behavior).**
    - Rewrite the Runtime Contract `^` paragraph for the picker: when it opens, snapshot
      plus local fuzzy filtering as a deliberate presentation-only responsibility (with
      the route-cache precedent), the insert and suffix rules, the chip, and the quiet
      incomplete state.
    - Add a "While the active task picker is open" keyboard table.
    - Remove Active Task rows from the Wikilink Completion inline-row description.

## Phase picker-design — Beautiful picker card, sizing, accessibility, docs, and macOS verification

Replace the functional view with the final design from the UX specification. Keep the
model and routing contracts from `picker-flow` intact.

1. **Files.** Add `Sources/BobMacCapture/ActiveTaskPickerView.swift`:
   `ActiveTaskPickerCard`, `ActiveTaskFilterBar`, `ActiveTaskSectionHeader`,
   `ActiveTaskRow`, `ActiveTaskDetailStrip`, `ActiveTaskEmptyState`, `ActiveTaskChip`,
   and `ActiveTaskKeyHints`. Add status styles
   (`CaptureEditorPalette.activeTaskStatus(_:) -> (symbol, color)`) so colors keep one
   source of truth. The Pomodoro tint is the existing `.pomodoroStart` pink; route and
   block ID use the existing palette.
2. **Card.**
   - `.regularMaterial` background, 12pt corner radius, 0.5pt `primary.opacity(0.08)`
     stroke, shadow radius 18 y 8, spanning the full auxiliary width (no 430pt cap or
     leading indent).
   - Filter bar (46pt): a scope token capsule (`^` monospaced semibold in the route
     color plus "Active Tasks" caption semibold), the filter field, the trailing
     `countText` (monospaced digits, secondary), and a small spinner only while a
     snapshot fetch is pending.
   - Hairline dividers.
   - List: `ScrollViewReader` + `LazyVStack(pinnedViews: [.sectionHeaders])`. Headers
     use a material background so pinned headers cover the rows beneath. Scroll the
     selection into view with minimal movement (anchor `nil`) on every selection change.
   - Detail strip (80pt), then the Bob-warning line and the partial-snapshot note inside
     the list's bottom edge.
3. **Rows, headers, detail strip, empty states, and chip** exactly per the UX
   specification.
   - Build rich text with `AttributedString`: per segment kind, apply monospaced plus
     faint background for code and the accent color for links; overlay semibold plus
     accent on match ranges.
   - Hover uses `.onHover`. A tap selects and inserts via
     `model.acceptActiveTask(id:submitAfterInsert: false)`.
   - The note kind icon and label come from `targetCacheSnapshot`: `inbox` → `tray`,
     `area` → `square.stack`, `project` → `folder`.
4. **Panel integration.**
   - While the picker is visible:
     - The editor renders at 50% opacity with hit-testing off. A transparent tap target
       over it calls `cancelActiveTaskPicker()`.
     - The auxiliary region renders only the card. Preview, destination, and error stay
       hidden.
     - `CapturePanelFooter` swaps its status and buttons for `ActiveTaskKeyHints`:
       keycap-style chips for ↑↓ Move · ↩ Insert · ⌘↩ Insert & Capture · esc
       Clear/Cancel, at the same footer height path so metrics keep working.
   - The chip renders as a compact capsule button aligned with where completion lists
     appear: `list.bullet.rectangle.portrait` icon, "Browse active tasks", and a "⇥"
     keycap.
   - No animations; resizing stays instant.
5. **Sizing.**
   - Add `CapturePanelLayout` constants: filter bar 46, row 34, section header 26,
     detail strip 80, list padding 6, max rows 11, min rows 3, corner radius 12.
   - Add `ActiveTaskPickerHeightPolicy(visibleRowBudget:displayScale:)` alongside
     `CanceledDraftStashPickerHeightPolicy`:
     - `listViewport = 2·padding + row·budget`
     - `ideal = filterBar + 1 + listViewport + 1 + detail`
     - `minimumVisible` uses `min(3, budget)` rows
     - Values are pixel-rounded.
   - Wire it into `currentAuxiliaryHeight` and the `onChange` hooks that reset
     `measuredAuxiliaryContentHeight`, the same way the stash picker is wired, so the
     panel grows on open, shrinks on close, and never resizes while filtering.
6. **Accessibility.**
   - The card is a `.contain` element labelled "Active task picker" with a usage hint.
     Headers use `.isHeader`.
   - Rows use `children: .ignore` with the presentation `accessibilityLabel`, the
     `.isButton` trait (plus `.isSelected`), and a default action that inserts.
   - Post `AccessibilityNotification.Announcement` on open ("Active tasks, 72 tasks"),
     on match-count change ("5 matches"), and on keyboard selection moves (the row
     label).
   - Selection and hover fills strengthen under Increase Contrast; text contrast never
     relies on tint alone.
7. **Tests (`BobMacCaptureTests`).**
   - `ActiveTaskPickerHeightPolicy` math and clamping.
   - Controller metrics: the panel grows from compact on open, keeps its height across
     filter changes, and shrinks after cancel, mirroring
     `testReceiveStashContentMetricsGrowsFromCompactAndShrinksAfterDismissal`.
   - Content-height policy with the picker's required minimum on a short screen.
   - The footer hint mode.
8. **Rendered-image review.**
   - Add a test gated on the environment variable `BOB_MAC_CAPTURE_RENDER_DIR` (skipped
     when unset). It renders `ActiveTaskPickerCard` with `ImageRenderer` at 760pt width
     in light and dark appearance for four states: grouped, filtered `q weight`, no
     matches, and no active tasks. It writes PNGs. The AppKit filter field may render
     blank; that is acceptable.
   - Run it on the macOS host, copy the PNGs back, and inspect them with the image
     reader.
   - Iterate on spacing, contrast, truncation, and alignment until the card looks
     polished in both appearances.
9. **README (visuals).** Describe the card layout, grouping and headers, row anatomy,
   detail strip, empty and warning states, chip, footer hints, fixed-height sizing, and
   accessibility behavior. Keep the behavior text from `picker-flow` consistent.

## Verification (every phase)

- There is no Swift toolchain on Linux hosts. Verify on macOS:
  - Copy the opened checkout to the tailnet host `mac`, for example
    `rsync -a --delete --exclude .build <checkout>/ mac:/tmp/bob-mac-capture-<phase>/`.
  - Run `just format-lint build test` there over `ssh mac` (or the matching
    `./Scripts/xcode-swift.sh` commands).
  - `mac` is best-effort and offline unless its lid is open.
- If `mac` is unreachable, confirm that the `macOS 26 SwiftPM` GitHub Actions run passes
  for the pushed commit (format lint, build, full `swift test`, bundle) and record the
  run ID. If neither is possible, record the precise limitation on the phase bead and
  keep all automated assertions in place.
- `picker-design` additionally runs the rendered-image review. If a logged-in GUI
  session is available on `mac`, it also installs a bundle and smoke-tests the real
  panel with the fake-bob fixture or a real vault dry run, typing `^`: open, filter,
  arrow, Return, ⌘↩, Esc twice, Backspace, and the chip.

## Done when

- Typing `^` as a capture item opens the large Active Task Picker with all In Progress
  and Next tasks, grouped by today's Pomodoro plan in Bob's order. No red live-preview
  error appears for the incomplete `^`.
- The filter bar fuzzy-matches task text (code and link aware), block ID,
  `route:block-id`, route, section, and Pomodoro name, with multi-token AND,
  case/diacritic folding, highlighted matches, and instant local updates. The panel
  height never changes while filtering.
- Return, Tab, and a click insert exactly Bob's `replacement` into Bob's range, keeping
  typed suffixes. ⌘↩ inserts and captures. Escape clears, then cancels with suppression
  and the chip. Backspace on an empty filter removes the trigger. Stale drafts are never
  edited.
- The inline completion list is unchanged for every other context.
- Keyboard, VoiceOver, and Increase Contrast behave as specified, and README documents
  behavior and visuals.
- `just format-lint`, `just build`, and `just test` pass on macOS (or the CI run is
  green), including the new CaptureCore and app tests.
