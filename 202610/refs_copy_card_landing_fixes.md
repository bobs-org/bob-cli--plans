---
tier: tale
size: medium
title: Finish the Bob Refs ⌃Y Copy card (epic bob-cli-7k) and close it out
goal: "Fix the Copy card contract deviations the bob-cli-7k landing review confirmed in
  bob-mac-capture, plus one bob-cli doc sentence, verify, then close epic bob-cli-7k.

  "
proposed_by: bbugyi200.apollo.bob-cli-7k.land
bead: bob-cli-7k
create_time: 2026-10-14 13:12:32
status: wip
---

- **PARENT:**
  [202610/refs_ctrl_y_copy_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/refs_ctrl_y_copy_card.md)
- **BEAD:**
  [bob-cli-7k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7k/README.md)

# Plan: finish the Bob Refs ⌃Y Copy card and land epic bob-cli-7k

## Context

Epic **bob-cli-7k** ("Bob Refs ⌃Y Copy card", plan
`plan:202610/refs_ctrl_y_copy_card.md`) has six closed phases:

- bob-cli `e7e98d2` adds `marker_id` to the JSON.
- bob-mac-capture `060d77b`, `7e7d8d6`, `c4b1a25`, `9b70c62`, and `2b7b330` add the Copy
  card.

The epic plan is the design contract. Read it with
`sase bead read bob-cli-7k -r "<why>"` and use the PLAN path that command prints. Its
"The design contract" section is the source of truth for everything below.

The land agent reviewed every phase against that contract:

- CI run 38601917702 at `2b7b330` builds and runs all 2128 tests. The only failures are
  the four `RefsInspectorScrollTests` failures that predate the epic; task bead
  bob-cli-7h tracks them, so do not touch them.
- No `--epic-symbol` entries exist.
- The follow-ups are already triaged and recorded on the epic:
  - bob-cli-7o was filed for the `typecheck-tests` SDK defect.
  - bob-cli-7h was corroborated.
  - The bob-cli-7k.5 "Mac unverified" note was declined as resolved.

The review confirmed the defects below. This tale fixes them, verifies the fixes, and
then closes the epic. Each fix is described by its required behaviour; choose the code
shape that fits the surrounding code.

Repositories:

- **bob-mac-capture**, the linked repo, takes almost everything. Open it with your
  `/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`). Use only the path it
  prints, and read its `AGENTS.md` before editing.
- **bob-cli**, the primary repo, takes one docs sentence.

`decisions:mac-capture-is-a-thin-client` still governs. Every payload stays either a
bob-authored string or a deterministic format of bob fields.

## Fixes in bob-mac-capture: RefsCore (`Sources/RefsCore/RefsCopy.swift`)

1. **Digit rows come from `commentedHighlights`, as the contract's catalog table says.**
   - **The defect:** `RefsCopyCatalog` numbers `1`–`9` over `notesRow.yourNotes`, which
     includes standalone `kind == "note"` entries, and formats every digit with
     `RefsCopyFormat.noteBlock`. A standalone note on a digit therefore copies
     `Comment: …` instead of `Note: …`. The numbering also drifts from the inspector's
     "Your notes" list, which shows only `commentedHighlights`.
   - **The fix:** number and format the digits over `notesRow.commentedHighlights`.
     `noteCount`, `visibleNoteCount`, `noteOverflowCount`, the digit hidden reasons ("No
     note k", "Note k doesn't fit — …"), and `notesOverflowText` all follow that count.
   - **What stays:** the `A` row keeps `yourNotes`. That covers its availability, its
     `allNotes` payload, its `N notes · Markdown[ + your notes]` preview, and its count.
     Standalone notes stay reachable through `A` and `H`.
   - **The A count:** expose the `yourNotes` count on the catalog (for example
     `allNotesCount`) for the A preview, the A title, and the notes header (fix 9).

2. **The `A` confirmation title reads correctly.**
   - It is `Copied N Notes` for N ≥ 2 and `Copied 1 Note` for one.
   - When there are no notes and only `own_notes` is copied, it is `Copied Your Notes`.
     Today it reads "Copied 1 Notes" or "Copied 0 Notes".

3. **`noteBlock` edge cases.**
   - Omit the quote section when the quote is nil or blank after trimming. Today
     `quote: ""` emits a lone `>` section.
   - Treat a whitespace-only comment as absent. Today the picker can emit `Comment:   `.
   - An empty line _inside_ a multi-line quote still becomes `>`.

4. **Key map edge cases (`RefsCopyKeyMap.command`).**
   - Characters in AppKit's function-key private-use range, U+F700–U+F8FF, must return
     `.consume`. This covers Shift- or ⌥-arrows, F-keys, Home/End, PgUp/PgDn, and
     forward delete. Today they become `.unknown` and show "Nothing on
     ‘<invisible glyph>’".
   - keyCode 33 closes only when the normalized flags are exactly `.control` (`⌃[`). A
     plain or Shift `[` falls through to the printable path, so it yields
     `.unknown("[")` ("Nothing on ‘[’") instead of being swallowed.

5. **A queued copy keeps its "waiting" message.**
   - `RefsCopyCard.handle(.move)` currently clears the message even while `pending` is
     still set. A later load then copies and hides Refs with no visible warning.
   - While `pending != nil`, a move, and the clearing of a transient message, must leave
     or restore "Waiting for notes… esc cancels".

## Fixes in bob-mac-capture: app target (`Sources/BobMacCapture/Refs/`)

6. **The note capacity budgets every non-digit row and is recomputed when the notes
   change** (`RefsPanelView.copyCardNoteCapacity(for:height:)` and its callers).
   - **The defect:** `nonNoteRowCount` is `rows.filter { $0.group != .notes }.count`.
     The `A` and `H` rows live in group `.notes`, so their 24 pt each is never budgeted.
   - **The second defect:** capacity is recomputed only in `onGeometryChange`, which
     fires on height changes. Notes arrive later without changing the height, so the
     capacity measured while notes were loading goes stale. `A` and `H` then render past
     the card and push the message line over the footer.
   - **The fix:**
     - Count every row whose key is not `.note(_)` as a fixed row, including `A` and
       `H`. Their presence does not depend on capacity, so there is no feedback loop.
     - Keep the notes-header and group-separator accounting.
     - Recompute the capacity whenever the card's notes state or rows change, as well as
       on height changes. For example, keep the last measured height and recompute on
       `onChange` of the card's notes state.
   - **Tests:** extract the capacity computation into a testable static function if it
     is not already one. Add a unit test showing that a loaded card with `A` and `H` at
     a given height leaves room for them.

7. **The card transition actually animates.**
   - **The defect:**
     `.animation(.easeOut(duration: 0.14), value: model.copyCard != nil)` sits on the
     `copyCard` `Group`. The parent's `if model.copyCard != nil { copyCard } else { … }`
     inserts and removes that Group itself, so the modifier never animates its own
     insertion or the inspector's removal.
   - **The fix:**
     - Move the animation onto a container that persists across the toggle: the
       `inspector` view in the wide `HStack`, and a wrapper around the narrow-branch
       `if/else`.
     - Give the inspector and list branches an `.opacity` transition so they cross-fade.
     - Keep the specified motion: cross-fade plus a 6 pt rise, 0.14 s easeOut, and
       cross-fade only under Reduce Motion.

8. **The pending timeout is a fixed 5 s from the queued key**
   (`RefsPanelModel. updateCopyCardTimers`).
   - **The defect:** every call cancels and restarts the 5 s task while `pending` is
     set, so arrows or unknown keys extend the timeout indefinitely.
   - **The fix:** restart the timeout only when a new pending key is queued or the
     queued key changes, and cancel it when `pending` clears.
   - **What stays:** the 1.5 s transient-message timer.

9. **The notes header shows the total, not the visible slice**
   (`RefsCopyCardView.catalogRows`). It currently passes
   `group.rows.filter(\.isNoteRow).count`. Use the catalog's `yourNotes` count from
   fix 1.

10. **⌃Y never leaks with Caps Lock on** (`RefsKeyRouter`, `case KeyCode.y`).
    - Compare `flags.subtracting(.capsLock) == .control`, so the reliability contract
      ("⌃Y is always consumed") holds.
    - Leave the other router keys unchanged.

11. **The model's Esc ladder includes the card.** `RefsPanelModel.perform(.escape)`
    closes an open copy card first (`closeCopyCard()`, return true), before the banner,
    query, scope, and close steps. The controller already routes keys to the card first;
    this makes the model's ladder match the contract on its own.

12. **⌥ tracking publishes only while it matters.**
    `RefsPanelModel. updateCopyModifiers` assigns `copyAlternatesActive` only when the
    value changes. The controller's `.flagsChanged` branch skips the call when
    `model.copyCard == nil`. Today every modifier press while Refs is key re-publishes
    and re-renders the panel.

## README (bob-mac-capture)

13. Make two README edits:
    - Change the Bob Refs key table's Esc row to "Copy card, then banner, then query,
      then scope, then close". That is the contract's wording; it now reads "…, then
      banner, query, scope, then close".
    - In "### Copy card", say that `1`–`9` copy the visible **commented highlights**,
      and that `A` and `H` also reach standalone notes.

## bob-cli

14. `docs/ref.md`: add `marker_id` to the sentence near the `task` field description
    that reads "`blocked` and `task` are additive under `schema_version` 1". It becomes
    "`blocked`, `task`, and `marker_id` are additive under `schema_version` 1".
    - The epic plan required this edit; phase bob-cli-7k.1 missed it.
    - Keep `RefRow`'s separate `marker_id` field. It keeps `marker_id` next to `title`
      in JSON field order, and behaviour is identical to the plan's serde-rename idea.

## Tests

Update the tests that encode the old digit numbering. For example,
`Tests/RefsCoreTests/RefsCopyAlternatesTests.swift` has a standalone note at `.note(2)`
and unwraps `.note(3)`. Then add the following.

- **RefsCore (`Tests/RefsCoreTests/RefsCopyTests.swift`, Linux):**
  - A standalone note between commented highlights:
    - it is not a digit row;
    - the digits match `commentedHighlights` order;
    - `A` formats the note as `Note: …`.
  - The `A` titles: "Copied 1 Note", "Copied 2 Notes", and "Copied Your Notes".
  - The `noteBlock` cases: blank quote and whitespace-only comment.
  - The key map:
    - plain Return (36);
    - ⌃J and ⌃K;
    - a private-use arrow character with Shift is consumed;
    - plain `[` gives `.unknown("[")`;
    - ⌃[ closes.
  - `pendingTimedOut()` sets "Notes didn't load — nothing copied" and writes nothing.
  - A move while a copy is pending keeps "Waiting for notes… esc cancels".
  - For every value row (`I T W U L X D N O P R S`), the preview equals the text
    payload.
- **App (`Tests/BobMacCaptureTests/`):**
  - The capacity function counts `A` and `H` (fix 6).
  - A queued digit is resolved by the loader: open the card while notes are loading,
    press `1`, let the fake show fetch land, and assert exactly one pasteboard write and
    the completion.
  - ⌘K from the card closes it and calls `actionsPresenter`.
  - `perform(.escape)` with the card open closes only the card.
  - The router maps ⌃Y with Caps Lock to `.showCopyCard`.
  - Update any design fixture or test that asserted the old notes-header count or digit
    numbering.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

- **RefsCore.** Run `just portable-check` on athena, the tailnet host with Swift. Read
  `sase memory read tailnet.md -r "<why>"` for host access first. Run it on a scratch
  copy, then delete the copy.
- **App and app tests.**
  - Rsync a scratch copy to the Mac without `.build` or `.git`. Run `Scripts/bundle.sh`
    and then `just typecheck-tests`.
  - That helper currently fails on clean master with "unable to load standard library
    for target arm64-apple-macosx26.0" (task bead bob-cli-7o). If it does:
    - type-check the four test targets in the scratch copy only, passing
      `-sdk "$(/usr/bin/xcrun --sdk macosx --show-sdk-path)"`, as phase bob-cli-7k.6
      did;
    - do not commit any change to `Scripts/typecheck-tests.sh`, which is bob-cli-7o's
      scope.
  - Never use Bryan's checkout or `just install`, and delete the scratch copy
    afterwards.
  - If the Mac is offline, say in the final report and the close note that the macOS
    targets are unverified.
- Keep swift-format lint clean.
- **bob-cli:** `just check` passes. Do not run `just check-full`.
- `git diff --check` is clean in both repos.

## Closeout (final step, same turn as the code)

Do not wait for CI or for this work's own commit; the closeout happens in this turn.

1. Run `sase bead epic-symbols bob-cli-7k`. It listed none at landing. Resolve or re-key
   any entry that has appeared.
2. Close the epic: `sase bead close bob-cli-7k --note "<verification>"`. Never use
   `--force` merely to make the close succeed. The note must summarize:
   - **Steps 1–2 (land agent):**
     - all six phases were verified against the contract;
     - CI 38601917702 at `2b7b330` passed every Copy card test, and the only failures
       were the pre-existing `RefsInspectorScrollTests` (bob-cli-7h, corroborated);
     - bob-cli commits after `e7e98d2` are unrelated capture work and bob-mac-capture
       has none after `2b7b330`, so nothing needed integrating.
   - **Follow-ups:**
     - bob-cli-7o was filed for the `typecheck-tests` SDK defect, which 7k.3, 7k.4, and
       7k.6 had proposed;
     - the 7k.5 "Mac unverified" note was declined, resolved by 7k.6's type-check and by
       CI.
   - **This tale:** the fixes made and the verification actually run (portable-check
     test count, Mac bundle and type-check result, `just check`).
   - **Contract points the review kept as implemented:**
     - "Notes unavailable — nothing copied" renders red, and "No notes on this
       reference" does not. The literal contract also lists "No notes" among the red
       ones, but it is informational like the other hidden-row reasons.
     - `F` with no PDF path at all says "No PDF path for this reference".
     - A standalone-note payload has no blank line before its provenance.
     - The extra messages "No Obsidian link…", "No highlight N", and "Nothing to copy…"
       stay.
     - `RefRow` keeps its separate `marker_id` field.
3. Run `just symvision` in bob-cli if the recipe exists, and confirm the whitelist is
   clean. At landing, bob-cli's justfile had no `symvision` recipe; if that is still
   true, say so in the final report.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path that
   `sase bead read bob-cli-7k` shows). bob-cli-7k has no `parent_bead`, so nothing
   further cascades.
