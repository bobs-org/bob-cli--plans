---
tier: epic
title: Relation = alias, direction flips, and the == escape
goal: 'In Bob Mac Capture, `=` is the only relation alias for `~`, and it opens Links
  To only where Bob''s gate allows. A `~` or `=` glued to a complete `~note:id` /
  `~~note:id` flips its direction. The relation label card flips direction with `~`
  / `=` and re-asks Bob for the new direction. A fast `==` types one literal `=` on
  every surface where `=` stands for `~`. Bob stays the only grammar authority, and
  `bob capture` / `capture-parse` stay literal.

  '
phases:
- id: card-flip
  title: Mac relation label card flips direction
  depends_on: []
  size: medium
  description: 'card-flip: in the Mac relation label card, `~`/`=` flip the relation
    in choose mode, edit the token, re-fetch catalog and verdicts, and support a fast
    `==` literal escape. Also adds teaching keycaps and fixes Escape from a Shift-Return
    card. Mac only.'
- id: bob-grammar
  title: Bob capture-rewrite relation grammar
  depends_on: []
  size: medium
  description: 'bob-grammar: in bob-cli capture-rewrite, an `=` opener replaces the
    relation `-` alias, `~~=` gives `~`, and a new `toggle_relation_direction` rule
    flips a complete token on a glued `~`/`=`. Ships with docs, help, and tests as
    `feat(capture)!`.'
- id: mac-equals-opener
  title: Mac adopts the = opener and post-token flip
  depends_on:
  - card-flip
  - bob-grammar
  size: medium
  description: 'mac-equals-opener: the Mac forwards `=` and token-glued `~` to Bob
    and replaces the dash arm with an equals opener arm that keeps the Space/Escape
    restores. It applies `toggle_relation_direction` natively without a picker and
    deletes every relation `-` path.'
- id: equals-escape
  title: Fast == escape in the editor and picker
  depends_on:
  - mac-equals-opener
  size: medium
  description: 'equals-escape: one shared equals-gesture record with a ~350 ms key-to-key
    window, so a fast second `=` undoes whatever the first `=` did and types one `=`.
    Covers the editor before and after Bob replies, every flip, and the picker; the
    card reuses it.'
- id: records
  title: Decision record, relation docs, and parity follow-up
  depends_on:
  - bob-grammar
  - equals-escape
  size: small
  description: 'records: author the decision record for the new gestures and partly
    supersede the card''s catalog-fetched-once claim. Add the docs/task-relations.md
    sentence, and file the Obsidian label-stage parity bead.'
decisions:
  memory_relation_flip_record:
    ask: Add the = alias / direction-flip decision record and mark the card's fetch-once
      claim superseded?
    memory:
    - decisions:relation-card-tab-accepts-ctrl-d-hides
    - decisions
    corrects: the catalog fetched once on open
    default: true
    answer: true
proposed_by: bbugyi200.athena.198
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-06 12:50:19
status: wip
bead_id: bob-cli-hz
---

- **PROMPT:** [prompts/202611/relation_equals_alias_and_direction_flips.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/relation_equals_alias_and_direction_flips.md)
- **BEAD:** [bob-cli-hz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hz/README.md)

# Plan: Relation `=` alias, direction flips, and the `==` escape

## Context

Bryan wants four changes to how Bob Mac Capture handles task relations (`~note:id`,
`~~note:id`, `|label`):

1. `=` becomes the only relation alias for `~`, used where it is appropriate.
2. A `~` or `=` typed right after `~foo:bar` turns it into `~~foo:bar`, and the reverse.
3. A fast `==` escapes the `=` alias everywhere it acts as `~`.
4. `~` / `=` flip "links to" ↔ "linked from" in the relation **label** card, as they
   already do in the relation picker.

The consolidated research report
`research:202611/relation_equals_alias_and_direction_flips/relation_equals_alias_and_direction_flips.md`
designs all four. Bryan accepted **every** requirement in it, including the
[ADJUSTED]/[ADDED] rows A1–A10. Each phase worker should read it with
`sase artifact read` before starting, focusing on "Recommended behavior contract",
"Facts I verified", and "Implementation design". This plan restates the binding
contract, so workers do not need to re-derive it.

Governing records:

- `decisions:mac-capture-is-a-thin-client`: Bob decides syntax. The Mac only routes
  keys, keeps one-shot arms, and renders. The Mac may use Bob's `capture-parse` spans as
  a _send filter_ but must never lex tokens itself.
- `decisions:bob-cli-releases-ride-a-daily-train`: mark the removed `-` alias with `!`
  in the commit subject, and never edit the version or CHANGELOG.
- `decisions:relation-card-tab-accepts-ctrl-d-hides`: navigation stays local. A
  direction flip is an _edit_, not navigation, and it re-fetches (see the `records`
  phase).

### Binding behavior contract

`D` is an eligible task draft. "Fast" means two non-repeat `=` key-downs within the
escape window with nothing else in between. The window is a named constant of 350 ms,
measured from `NSEvent.timestamp`.

| Surface and state                           | Key               | Result                                                               |
| ------------------------------------------- | ----------------- | -------------------------------------------------------------------- |
| Editor, `D `                                | `=`               | `D ~`, opens Links To                                                |
| Editor, `D `                                | fast `==`         | `D =`, picker dismissed (also when the second key beats Bob's reply) |
| Editor, `D `                                | `=` then slow `=` | `D ~~`, Linked From                                                  |
| Editor, `D `                                | `=` then `~`      | `D ~~` at any speed                                                  |
| Alias-opened Links To, empty filter         | Space / Escape    | `D = ` / `D =`, at any speed                                         |
| Editor, `D `                                | `-`               | Literal `-`, no picker (**behavior removed**)                        |
| Editor, `x=2`                               | `-`               | `x=2~` (unchanged past-segment alias)                                |
| Editor, item start, `x `, `=bugs`, …        | `=`               | Native meaning (unchanged; Bob's gate refuses)                       |
| Editor, `D ~` / `D ~~`                      | `=`               | `D ~~` / `D ~`                                                       |
| Editor, `D ~foo:bar` / `D ~~foo:bar`        | `~` or `=`        | `D ~~foo:bar` / `D ~foo:bar`, key consumed, no picker pops           |
| Editor, `D ~foo:bar\|see`                   | `~` / `=`         | Flips / label becomes `see=`                                         |
| Editor, `D ~foo:bar ` (spaced)              | `~` or `=`        | Starts another relation (unchanged)                                  |
| Editor, after any flip above                | fast `==`         | Flip undone, one `=` typed at the original caret                     |
| Task picker, any filter                     | `~` or `=`        | Flips and keeps the filter (unchanged)                               |
| Task picker                                 | fast `==`         | Original direction restored, one `=` inserted into the filter        |
| Label card, choose mode                     | `~` or `=`        | Flips token and sentence, re-asks Bob, keeps query and selection     |
| Label card, choose mode                     | fast `==`         | Flip undone, one `=` typed into the label field                      |
| Label card, rename mode                     | `~` / `=`         | Text, as today                                                       |
| Pasted text, `bob capture`, `capture-parse` | any               | Literal; no timing-based syntax reaches the grammar                  |

Settled points (do not reopen):

- The auto-inserted trailing space after a picker accept stays. At `~foo:bar ␣`, `=`
  starts a second relation.
- A card flip keeps the label phrase. It never auto-inverts `supersedes` ↔
  `superseded by`.
- `<` / `>` direction markers and a modifier chord are rejected.
- The Obsidian Task Card label stage is out of scope. The `records` phase files a
  follow-up bead for it.

### Phase graph

`card-flip` (Mac) and `bob-grammar` (bob-cli) are independent and may run in parallel.
The Mac phases are serialized (`card-flip` → `mac-equals-opener` → `equals-escape`)
because they all edit `CapturePanelModel.swift`, `CaptureKeyCommandRouter.swift`, and
`CapturePanelController.swift`. `records` runs last.

### Repositories and verification (every phase)

- **bob-cli** (this repo): `just check`. For help and doc changes, also run
  `just docs-gen` and `just docs-check`, after reading the `docs.md` and `cli_rules.md`
  reference memory.
- **bob-mac-capture** (linked repo): open it with `/sase_repo` and read its `AGENTS.md`.
  Never land Swift you have not compiled:
  - Run `just portable-check` (on a host with `swift`).
  - Compile a scratch copy on the mac: rsync without `.build`/`.git`, run `bundle.sh`
    and `just typecheck-tests`, then delete the copy. Read the `tailnet.md` reference
    memory for access.
  - Run `Scripts/macos-ci-check` before landing. After every push, watch the exact-SHA
    macOS run to green with `/sase_monitor`.
  - If the mac is offline, say in the final report that the macOS targets are
    unverified.
  - Format only the lines you change (`.swift-format`).
  - Docs changes need `just docs-check`.
- Mac tests use fake-bob replies, injected `NSEvent` timestamps, and delayed replies.
  **Never use wall-clock sleeps.** Async tests must never await an untimed continuation.

## Phase `card-flip`: Mac relation label card flips direction

Mac-only. This phase needs no Bob release and carries the Escape fix. Main files:
`CaptureKeyCommandRouter.swift` (`relationLabelPromptCommand`),
`CapturePanelModel.swift` (`CaptureRelationLabelPromptState` near the top,
`openRelationLabelPromptFromPickerRow`, `cancelRelationLabelPrompt`,
`fetchRelationLabelCatalog` / `fetchRelationLabelSuggestions`,
`relationLabelFooterItems`), `CapturePanelController.swift` (command dispatch), and
`CapturePanelView.swift` (card header).

1. **Routing.**
   - In choose mode (`renameActive == false`), an unmodified (Shift allowed), non-repeat
     `~` or `=` with no marked text in the card field returns a new
     `.flipRelationLabelDirection` command. This check runs before
     `isolatedPromptCommand`.
   - Auto-repeat is consumed.
   - Rename mode keeps `~` / `=` as ordinary text (A10).
2. **Guards.** Flip only when `!isSaving`, `rename == nil`,
   `draftSnapshot == plainDraft`, and the byte at `tokenRange.start` is `~`. Otherwise
   consume the key without effect.
3. **Draft edit.**
   - Make `direction` and `tokenRange` mutable.
   - Insert or remove one `~` at `tokenRange.start + 1`.
   - Shift `tokenRange.end`, `labelRange`, and `restoreCursor` by ±1, following the
     existing rename draft fix-up, which already rebases these fields.
   - Apply the edit through a focus-preserving native edit so the card keeps focus.
   - Update `draftSnapshot`.
   - The sentence follows `direction` automatically (`relationLabelSentenceSource` /
     `Target`).
4. **Invalidation (A6).**
   - Old catalog candidates carry replacement ranges into the _old_ draft, and their
     `row_disable` reasons and cancel/complete/successor verdicts are computed for the
     source task, which changes with direction. They must never commit after a flip.
   - Clear `relationCancel`, `relationComplete`, `relationSuccessor`, `suggestions`,
     `suggestionAuthoredLabel`, `errorMessage`, and `pendingSubmit`.
   - Bump both label generations, drop the in-flight markers, and re-fetch the catalog.
     Also re-fetch the typed-row probe when the query is non-empty.
   - Return before the fresh catalog/probe arrives goes through the pending-submit path,
     and Tab does nothing until the catalog arrives. Neither ever commits a stale
     candidate.
   - Show the list in a way that does not flash empty, for example by keeping the rows
     dimmed with the header spinner.
   - Keep the query, `selectedKey` (if the key survives the refresh, show its new
     disabled reason; existing key-survival rules apply), `oneOff`, `showHidden`, hidden
     rows, and ghosts.
   - A late reply for the old direction is dropped by the existing `draftSnapshot`
     guards. Add a test for this.
   - The flip writes no label preference and runs no effect.
5. **`returnPicker` and Escape (A7).**
   - A flip also flips `returnPicker`: its source (`.linksTo` ↔ `.linkedFrom`), its
     pre-insert `draftSnapshot` (insert or remove the second `~`), and its
     `replacementRange.end` / `restoreCursor`.
   - Fix the existing bug in `cancelRelationLabelPrompt`. Today, Escape from a card
     opened by Shift-Return sets `picker = returnPicker` but leaves the inserted token
     in the draft and `pickerIndex` nil (`closePickerForAccept` cleared it), so the
     picker's snapshot guards fail.
   - Escape must restore `returnPicker.draftSnapshot` undoably and rebuild `pickerIndex`
     from `returnPicker.candidates` under the (possibly flipped) source, as
     `cancelTaskIDPrompt` does. Then recompute `pickerPresentation`, keep the filter,
     re-highlight the marker, and refocus the filter.
   - First write a test that fails today.
     `testRelationCardEscapeReturnsToPickerWhenStaged` asserts only `picker != nil`;
     strengthen it to check the restored draft, a non-nil presentation, and a working
     accept and flip after Escape.
   - In the `|` flow (`returnPicker == nil`), Escape keeps the flipped token and
     restores the shifted caret.
6. **Card `==` escape.**
   - Add a small reusable equals-gesture primitive: the window constant (350 ms), the
     first key's `NSEvent.timestamp`, and a non-repeat check. Synthesized events with
     `timestamp: 0` (see `CapturePanelController`) must never satisfy the window. The
     `equals-escape` phase generalizes this primitive, so give it a neutral name and
     home.
   - A card `=` flip arms it. A second fast `=`, with nothing else in between and the
     same prompt still open, undoes the flip (same edit and invalidation as above) and
     inserts one `=` into the label field at the field caret through the normal
     label-update path, so the probe runs.
   - Any other key, a click, a selection change, a rename, or dismissal clears the arm.
     `~` never inserts text, because labels can never contain `~`.
7. **Teaching (A9).**
   - Show a muted `~ flips` keycap in the card header (choose mode only).
   - When the selected row is disabled for `new_task`, or Bob sent an inverse hint (copy
     that says "use ~~"), add a flip keycap to the footer. Decide from structured fields
     (`disabledReason` code, the hint's presence), never by parsing copy.
   - Do not spend a sixth footer pair otherwise.
   - Announce "Links To" / "Linked From" for VoiceOver on each flip.
8. **Docs** (`just docs-check`):
   - `docs/relationship-pickers.md`: add the card keys, the flip semantics (re-asks Bob,
     keeps query and selection), and the fixed Escape.
   - `docs/keyboard.md`: add the relation-card key rows.
9. **Tests** (`CaptureRelationPanelTests` and siblings):
   - flips in both directions from both the `|` and Shift-Return flows;
   - catalog and verdicts re-fetched, and a late old-direction reply dropped;
   - Return during the re-fetch commits only after the new reply;
   - query, selection, `oneOff`, and Hidden state preserved;
   - rename passthrough and the `isSaving` guard;
   - Escape after a flip from both flows;
   - key repeat consumed;
   - fast `==` (and the threshold boundary) versus slow `= =` (flip twice).

## Phase `bob-grammar`: Bob capture-rewrite relation grammar

Main file: `src/native/capture_language/rewrite.rs`. The JSON shape and `schema_version`
of `capture-rewrite` stay the same. Only a new `rule` value is added.

1. **`=` opener replaces the relation `-` alias (A1, A2).**
   - Rename `expand_relation_dash_alias` → `expand_relation_equals_alias` and
     `relation_dash_position_ok` → `relation_alias_position_ok`. Keep the gate
     unchanged: space or tab before the key, a non-empty item prefix after bullet
     stripping, balanced backticks, `wikilink_claims` false, and `completion_field_at`
     on the swapped draft giving `RelationTo` with replacement exactly
     `(start, cursor)`.
   - Summary: `Typed ~ for =`.
   - In `expand_sigil_alias`, `-` now tries only `expand_past_segment_dash_alias`, so
     `x=2-` → `x=2~` and `x=~-` → `x=~~` stay.
   - Delete the `-=` branch of `expand_relation_flip_alias` and
     `finish_relation_flip_dash_alias`.
   - For `=`, try the bare flips first (byte before is `~`), then the opener.
2. **Bare flips.** Keep `~=` → `~~`. Add `~~=` → `~`: delete one `~` and the trigger,
   but only where the result opens an empty `RelationTo` field ending at the caret
   (checked through `completion_field_at`, same wikilink guard). Past-segment `x=~~=`
   therefore stays literal. Update the doc comments that say `~~=` stays literal.
3. **Post-token flip: new `RewriteRule::ToggleRelationDirection`
   (`toggle_relation_direction`) (A3).**
   - It runs in `rewrite_draft` before `expand_sigil_alias` and requires `--cursor`.
   - **Trigger.** The byte before the cursor is `~` or `=`. With that byte removed, a
     `ScannedRelation::Complete` from `scan_line_relations` on the caret's physical line
     ends exactly at the trigger's start. The trigger must therefore be glued: a spaced
     key starts another relation, and text glued after the caret changes the token and
     disqualifies it.
   - **Exclusions.** Escaped, protected (code span, wikilink), partial, invalid, and
     strikethrough tokens never qualify. After a `|label`, only `~` qualifies, because
     `see=` is a legal label and `~` never is.
   - **Eligibility.** Also delegate it to the completion grammar, as the other aliases
     do. On the stripped draft, `completion_field_at` at the token end must report
     `RelationTo` / `RelationFrom` covering the token, or `RelationLabel` for a labelled
     token, so contexts that never parse relations (Pomodoro commands, `x=` tokens) stay
     literal.
     - Probe with `bob capture-complete` / `capture-parse` first. The research verified
       that `Fix it ~foo:bar` with the caret at the end yields `relation_to` with
       replacement `7..15`.
     - If a legitimate case (quoted note, labelled token) is not covered, fall back to
       the item parser's relation spans, never to weaker lexing.
   - **Edits.** Emit two sorted edits: insert `~` at `start + 1` (To → From) or delete
     the byte at `start + 1` (From → To), and delete the trigger. Every other byte
     (quoted note names, label, `||`) is preserved. The cursor maps to the token's new
     end.
   - **Validation.** Re-scan the result and require the same note, block ID, and label
     in the opposite direction; otherwise decline.
   - Summary: `Flipped to Linked From` / `Flipped to Links To`.
4. **Idempotence.** Feeding any output back with its cursor changes nothing.
5. **Docs and help.**
   - `src/native/capture_rewrite.rs` long help: replace the `-` relation sentences, add
     `~~=` and the toggle rule, and fix the rule count ("four rules").
   - `docs/capture.md`: the "Sigil alias" relation paragraphs (`-` / `-=` text near the
     past-segment paragraph), a new "Relation direction toggle" paragraph, and the
     `rule` value list.
   - Regenerate `docs/reference/cli/capture-rewrite.md` with `just docs-gen`, then run
     `just docs-check`.
   - Do not edit the mounted `docs/reference/mac-capture/*` copies; they are owned by
     bob-mac-capture and git-ignored here.
6. **Tests.**
   - Unit tests in `src/native/capture_language/tests/rewrite.rs`: retarget
     `rewrite_draft_expands_relation_dash_alias` / `…_stays_literal` to `=`, and replace
     `…_relation_dash_flip_alias`.
   - CLI tests in `tests/cli/capture/rewrite.rs`: the `-` → `~` and `-=` cases become
     `=` cases, plus `-` is now literal.
   - Cover every gate exclusion: `=`, `=3`, `x =`, `x=2 =`, `=/ =`, `Fix it ==`, line
     start, bullet start, open backtick, and wikilink.
   - `-` after a space is now literal, while `x=2-` and `x=~-` still rewrite.
   - `~=` → `~~`, `~~=` → `~`, and `x=~~=` literal.
   - The post-token matrix: both directions × `~`/`=`, quoted and Unicode notes,
     labelled (`|see=` literal, `|see~` flips, `||` preserved), spaced key, a token not
     at line end, an escaped token, a token in a code span, and a strike token.
   - Idempotence for every rule.
7. **Commit** as `feat(capture)!: …`, because the documented `-` → `~` relation alias
   goes away. A new Mac with an old Bob degrades to a literal `=`. An old Mac with a new
   Bob degrades to a literal `-`, and it never forwards the new triggers.

## Phase `mac-equals-opener`: Mac adopts the `=` opener and post-token flip

Requires the `bob-grammar` behavior. Build bob-cli at that commit to record fixtures
with real `bob capture-rewrite` output; do not hand-write expected JSON.

1. **Forwarding** (`CapturePanelController.noteSeparatorToggleIntentIfEligible`). These
   are send filters only; Bob decides.
   - Forward `=` when the previous character is a space, tab, or `~`, or when the latest
     `capture-parse` spans put the end of a relation token (`relation_block_id`, or
     `relation_label` including a quoted label's closing quote) at the caret.
   - Forward `~` only in that last (relation-span) case, so an ordinary `~` never waits
     on a rewrite round trip before its picker opens.
   - Keep forwarding `-` for the past-segment alias.
   - A line-start `=` (Pomodoro commands) is never forwarded, so its completion is not
     delayed.
2. **Equals opener arm.** Replace `CaptureRelationDashArm` / `relationDashLiteralArm`
   with an equals opener arm.
   - Arm it after an `expand_sigil_alias` reply whose single edit replaced `=` with `~`
     and whose byte before the edit is not `~`, because that is the opener rather than
     `~=` (A1; cld F11).
   - While the arm is live on an alias-opened Links To with an empty filter, Space
     restores `= ` and Escape restores `=`, at any speed, through the native undoable
     edit, and closes the picker.
   - Delete the `-` → `--` restore, `restoreRelationDashLiteral(_:key:)`'s dash
     spelling, `.restoreRelationDashLiteral`, and `relationDashLiteralArmed`. Rename
     them for `=` where they survive.
   - A second `=` in this phase still flips through the picker (slow behavior); the fast
     escape arrives in `equals-escape`.
   - Shape the arm so `equals-escape` can fold it into the shared equals-gesture record
     from `card-flip`.
3. **Flip replies.**
   - `CaptureRelationFlipAliasOpen` keeps covering `~=` → `~~` and now `~~=` → `~`. Drop
     `-=`.
   - Neither flip arms Space/Escape.
4. **`toggle_relation_direction` replies.** Add the rule name to the native single-edit
   path (`applyCaptureRewrite`; `CaptureRewriteResponse.coveringEdit` merges the two
   edits), so Undo is one step.
   - Do **not** request completion, so no picker pops.
   - Do not arm anything.
   - Announce Bob's summary.
   - An unknown rule must never fall through to `setPlainDraft` for this case.
5. **Picker.**
   - The flip behavior is unchanged.
   - While the opener arm is live, the subtitle may read "`==` types `=` · `~` flips"
     (keep it if it fits; the `equals-escape` phase makes the `==` part true, so add the
     `==` text there if not here).
6. **Docs** (`just docs-check`):
   - `docs/relationship-pickers.md`: remove the `-` paragraph, and add the `=` opener,
     the Space/Escape restores, and the post-token flip.
   - `docs/keyboard.md`: the `-` and `=` editor rows, and the picker paragraph that
     mentions the `-` restores.
   - `docs/capture-panel.md`: the rewrite-lane paragraph.
7. **Tests.**
   - Retarget `CaptureRelationDashAliasTests` to the `=` opener, keeping its
     past-segment cases, and update `CaptureRelationFlipAliasTests`.
   - Replace the `capture-rewrite-sigil-alias-relation*.json` fixtures with recorded
     `=`, `~=`, `~~=`, and toggle fixtures.
   - Add cases:
     - the post-token flip applied natively with no completion request and no arm;
     - labelled `~` flips and labelled `=` stays literal;
     - an old-Bob `changed: false` leaves the literal key;
     - `-` after a space stays literal;
     - `~` that is not at a relation end is not forwarded.

## Phase `equals-escape`: fast `==` escape in the editor and picker

Generalize the `card-flip` primitive into one equals-gesture record (A4, A5), and fold
the opener arm from `mac-equals-opener` into it.

1. **Record.** The record holds:
   - the surface (editor, picker, or card);
   - the prompt or picker identity;
   - the pre-gesture draft, selection, filter, and direction;
   - the rewrite generation;
   - the first key's `NSEvent.timestamp`;
   - whether it is an opener (only the opener arms the Space/Escape restores).

   Check it before ordinary relation routing in the editor, picker, and card.

2. **Arming.** Arm the record after:
   - an `=` opener reply;
   - a `~=` / `~~=` flip;
   - a `toggle_relation_direction` reply whose trigger was `=`;
   - a picker `=` flip;
   - a card `=` flip (already done in `card-flip`; move it onto the shared record).
3. **Second `=` after Bob's reply,** fast and with nothing in between: restore the saved
   state and type one `=` at the original caret (or into the filter or field). Do not
   "flip again".
   - The editor opener gives `D =` and the picker is dismissed.
   - `~=` → `~~` then fast `=` gives `~=`; `~==` → `~=`.
   - A post-token flip undone gives `~foo:bar=`, which Bob reports as
     `invalid_relation`; that is expected.
   - A picker flip undone restores the direction and inserts `=` into the filter.
   - `===` gives `==`, because the third `=` follows `=` and no alias fires.
4. **Second key before Bob's reply.** Reuse the `HashAliasProbe` pattern: re-ask Bob
   about the first draft, and act only if the first key really was an alias.
   - For `==`, collapse to one `=`.
   - For `=~`, produce `D ~~` (the reliable Linked From spelling at any speed).
   - Otherwise leave the text.
   - A stale first reply is still dropped.
5. **Clearing and guards.**
   - Ignore auto-repeat and marked text (IME).
   - Clear the record on any other key, a click or caret move, a paste, Undo, a
     dismissal, or a window timeout.
   - Synthesized `timestamp: 0` events never satisfy the window.
   - A slow `= =` keeps meaning flip-twice / Linked From.
6. **Picker copy.** While an opener record is live, the picker subtitle reads "`==`
   types `=` · `~` flips", unless `mac-equals-opener` already added it.
7. **Docs.**
   - `docs/keyboard.md` and `docs/relationship-pickers.md`: the fast-`==` rule on every
     surface.
   - `docs/capture-panel.md`: the timing note, next to the `\` / `/` double-tap
     paragraph. That rule is a stateless arm; this one is timed, so say so.
8. **Tests** (`CaptureRelationFlipAliasTests`, `CaptureRelationDashAliasTests`
   successor, and card tests):
   - existing "two `=` presses flip there and back" cases now carry slow timestamps;
   - fast `==` before, during, and after Bob's reply;
   - the threshold boundary;
   - `=~`, `~==` → `~=`, and `===`;
   - `~=` not arming Space/Escape;
   - the post-token undo and the picker filter insert;
   - the card now uses the shared record;
   - synthesized-event and key-repeat guards.

   Use delayed fake-bob replies and injected timestamps.

## Phase `records`: decision record, relation docs, and parity follow-up

> [!decision] memory_relation_flip_record

1. **Decision record.** Use `/sase_memory_write`; this plan's accepted memory decision
   authorizes it. Add a new `decisions` strand, for example
   `relation-equals-alias-and-flips`.
   - **Claim:** `=` is the only relation alias for `~` and fires only under Bob's Links
     To gate. The past-segment `-` stays. A glued `~`/`=` flips a complete relation
     token (only `~` after a label). `~`/`=` flip direction in the editor, picker, and
     label card. A card flip re-fetches the catalog and verdicts. A fast `==` (350 ms
     key-to-key) types one literal `=`, and Space/Escape restore on an alias-opened
     empty Links To.
   - **Applies to** bob-cli and Bob Mac Capture.
   - **Why:** the research consolidation and its rejected-alternatives table.
   - **Cost:** the removed `-` alias, the timing window, and the trailing-space
     trade-off.
   - **Reopens when:** Bryan wants the "just accepted" arm, or a different `==` result.
   - **Evidence:** the research ref plus the landed commits.
2. **Supersede in part.** Mark `decisions:relation-card-tab-accepts-ctrl-d-hides` with
   `metadata.status: superseded-in-part` and `superseded_by` pointing at the new strand.
   - Add a back-link sentence: a direction flip is an edit that re-fetches the catalog;
     navigation still never does.
   - Do not otherwise edit its accepted body.
   - Run `sase memory init`.
3. **`docs/task-relations.md`** (bob-cli): extend the intro sentence about the picker
   flip. In Bob Mac Capture, `~`/`=` also flip in the label card, and a glued `~`/`=`
   flips a completed token. Run `just docs-check`.
4. **Follow-up bead.** Use `/sase_new_task` to file a bob-plugins task bead for parity:
   the Obsidian Task Card relation-label stage should flip direction on `~` the same
   way.
5. **End-to-end check.**
   - Run the behavior-contract table against the landed bob-cli
     (`bob capture-rewrite --cursor …` for the Bob rows).
   - Confirm the exact-SHA macOS CI runs of the three Mac phases are green.
   - Report any row that could only be checked by reading code.
