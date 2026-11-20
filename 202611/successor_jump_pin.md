---
tier: epic
title: Ctrl+J jump pin on the successor card
goal: "On the successor card in Obsidian and in Mac Capture, Ctrl+J pins one offer as
  the jump successor. The pinned offer can still be left open, pulled in, marked Done,
  or canceled. Once every write of the triggering close or capture has landed, Obsidian
  opens that task's note and focuses its line. The pin never writes to the vault, wins
  over the Cmd+Return open-notes landing, parks a ]s walk instead of advancing it, and
  never fails the close if the target can't be found.

  "
decisions:
  walk_park_record:
    ask:
      Record a decision that a pinned close lands on the pin and parks the walk, marking
      two walk records superseded in part?
    memory:
      - decisions
      - decisions:answering-advances-the-walk
      - decisions:ctrl-enter-leaves-pre
    default: true
    corrects:
      takes that row out of today's walk advances once to the next remaining item.
    answer: true
  jump_glossary_term:
    ask: Add a Jump Successor glossary term and regenerate the docs glossary from it?
    memory:
      - glossary:jump-successor
    default: false
    answer: false
phases:
  - id: cli-jump-offer
    title: bob capture --jump-offer and the jump JSON contract
    depends_on: []
    size: medium
    description:
      "cli-jump-offer: add `-j, --jump-offer <N|KEY>` to `bob capture`, the
      `pull_in.jump.supported` capability, the write-free `offer_jump_stale` refusal,
      and a top-level `jump` object resolved from final staged text (routed moves, block
      IDs, cleaned description), with dry-run parity, contract vectors, and docs §12.24
      plus the picker v5 rows."
  - id: nav-pin-card
    title: Obsidian successor card pin (picker API v5)
    depends_on: []
    size: medium
    description:
      "nav-pin-card: in bob-navigation-hotkeys, extract code from fragment 206 to make
      room, then add the Ctrl+J pin: radio toggle by key, a solid accent `↗ Jump`
      capsule, hover button, footer chip, summary, and status copy. Enter's highlight
      fallback is suppressed while a pin is staged, and a jump-only commit returns
      `{picks, closes, jump}`. Picker API goes to v5."
  - id: nav-jump-landing
    title: Obsidian landing API, walk follow, and receipt line
    depends_on:
      - nav-pin-card
    size: medium
    description:
      "nav-jump-landing: add nav `api.landOnTask(target)`, which re-finds the task by
      identity (never by a clamped line), focuses it, records a Vim jump, and flashes
      the line. Add reviewWalk v2 `continue(origin, {follow})`, which parks the walk
      before landing, and a notice v5 `↗ “S”` receipt line."
  - id: cycler-jump-close
    title: task-status-cycler honors the pin after every managed Done
    depends_on:
      - nav-jump-landing
    size: medium
    description:
      "cycler-jump-close: offer the pin only when the caller can honor it; keep `jump`
      through normalization; resolve the target's post-write identity from picks,
      closes, or recovery edits; land after retirement and the notice, or return it to
      walk-owned callers. Covers the Vim Ctrl+Enter walk path, parked successors, and
      cycler API v6."
  - id: nav-jump-callers
    title: Wire the remaining Obsidian picker callers
    depends_on:
      - cycler-jump-close
    size: medium
    description:
      "nav-jump-callers: opt the PRE/POST checklist claim and the relation-complete
      compose path into the jump; on project promotion, land on the pin instead of the
      new ^prj line; leave the pin withheld on the relation-merge card. Tests for each."
  - id: mac-jump-pin
    title: Mac Capture Pull-In card pin
    depends_on:
      - cli-jump-offer
    size: medium
    description:
      "mac-jump-pin: add `jumpKey` to CapturePullInOutcomes. Ctrl+J is consumed by the
      card's router. Add the capsule with a glide animation, plus footer, button,
      context-menu, and VoiceOver copy. Gate `--jump-offer` on the capability, reopen
      after `offer_jump_stale`, and on success open the jump landing as a one-entry
      `bob-open-notes` URL in place of the open-notes intent."
  - id: jump-docs-memory
    title: Walk docs, decision record, and glossary
    depends_on:
      - nav-jump-callers
      - mac-jump-pin
    size: small
    description:
      "jump-docs-memory: update bob-cli `docs/freshness.md` §6 for the pinned-close
      park, cross-check every surface's Ctrl+J docs, and (per the memory decisions)
      write the walk-park decision record, the supersession marks, and the glossary term
      with `just docs-gen`."
proposed_by: bbugyi200.apollo.research.5m.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 17:03:12
status: wip
---

- **PROMPT:**
  [prompts/202611/successor_jump_pin.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/successor_jump_pin.md)

# Plan: Ctrl+J jump pin on the successor card

Source research (Bryan agreed with its requirements; memory changes only as the
decisions above allow): `research:202611/successor_jump_pin/successor_jump_pin.md`. This
plan is the contract every phase implements; where it is more specific than the
research, this plan wins.

## The design in one sentence

**Jump is where you land, never what happens.** Ctrl+J stages one _jump pin_ on one
offer of the successor card. The pin sits alongside the four verdicts (Leave, Pull in,
Done, Cancel), is applied after every write lands, finds the task again by identity, and
never touches the vault.

Each combination reads as a sentence. _Leave it, take me there_ lets Bryan prioritize,
schedule, or split the task in place. _Pull it in, take me there_ lets him fix it before
starting. _Close it, take me there_ lets him write the skipped Work Log. _Cancel it,
take me there_ lets him salvage its notes.

## Shared semantics (all surfaces)

1. **One pin per card, zero by default.**
   - Ctrl+J toggles the pin on the highlighted row.
   - On another row, Ctrl+J moves the pin; on the pinned row it clears the pin.
   - The pin is keyed by offer **key**, never by list index.
   - Ctrl+J never moves the highlight. To pin a row and also close it, press Ctrl+J
     first, then Ctrl+X (which still advances, as it does today).
2. **Independent of verdicts.**
   - Verdict, mark, filter, fold, and group-expand changes never clear or move the pin.
   - Marks and verdicts still exclude each other on a row, as today.
3. **Navigation only.**
   - The pin writes nothing: no field, tag, lane change, freshness stamp, block ID,
     timer, or reopen.
   - A pinned row that is left open recovers exactly like an unpinned one.
   - Focusing the pinned task never stamps it and never uses up a review landing.
4. **Pinnable rows.**
   - v1 pins keyed, selectable offer rows only.
   - Ctrl+J on a muted row, a header, a `+N more` fold, or an empty list does nothing
     except show a refusal status line.
   - Muted-row pins (for example, jumping to a still-blocked dependent) are out of scope
     for this epic.
5. **Lifetime.**
   - The pin lives exactly as long as the card's marks do: it survives Mac
     Back-from-inbox-route and refusal reopens, and it dies when the card closes without
     committing.
   - If a refresh loses the pinned key, the pin is dropped with a visible status line
     (fail closed).
   - A pin alone never arms the two-step Esc: that guard protects staged _writes_, and
     the pin is not a write.
6. **Commit rules.** A staged pin fires on **every** commit key:

   | Staged                        | Enter / Return                                                                         |
   | ----------------------------- | -------------------------------------------------------------------------------------- |
   | nothing                       | pulls in the highlighted row (today)                                                   |
   | marks and/or verdicts, no pin | applies them (today)                                                                   |
   | **pin only**                  | **commits with no picks or closes, then jumps** (the highlight fallback is suppressed) |
   | **pin + marks/verdicts**      | applies them, then jumps                                                               |
   | Mac ⇧↩                       | applies verdicts only, keeps the pin, and still jumps                                  |
   | Esc that closes the card      | discards everything, including the pin; no navigation                                  |

   The fallback is suppressed only while a pin (or a verdict) is staged. Clearing the
   pin returns the card to its untouched behavior.

7. **Precedence after commit.** Exactly one landing per gesture, in this order:
   - A staged pin **replaces** Mac Capture's open-notes landing, whichever key asked for
     it: ↩, ⇧↩, ⌃↩, or an earlier draft ⌘↩/⌃↩.
   - A staged pin also replaces the `]s` walk's auto-advance: the walk **parks**, and
     the next `]s`/`[s` resumes at the closed row's neighbour. This is the Ctrl+Shift+M
     rule.
   - It replaces project promotion's landing on the new `^prj` line.
   - Without a pin, today's behavior is unchanged everywhere.
8. **Reliability.**
   - Writes land first.
   - The target is found again against post-write text: by routed move, then block ID,
     then exact planned post-write line, then a unique cleaned-description match.
   - Never land on a pre-write line number, and never clamp onto a nearby line.
   - If the target can't be found after commit, every write stays. The jump is skipped
     and one short notice names the lost task. Navigation failures never replay a write.
   - A stale jump selector at Mac submit refuses with no writes, like the other offer
     selectors.

## Look (both surfaces)

The card already uses its colors this way:

- **Obsidian:** `--interactive-accent` for the highlighted row (a 3 px inset leading
  rule plus an accent border) and a soft accent tint for marked rows.
- **Mac:** green for Pull in/Done, red for Cancel, and amber for caveats.

The pin therefore gets the one treatment the card does not use yet: a **solid filled
accent capsule**. Highlight is an outline, a mark is a tint, and the pin is a solid
capsule, so all three stay distinct in the same hue. That solid accent also matches the
accent landing flash, so pinning a task and arriving at it read as one story.

- **Capsule:** an icon plus a word, never an icon alone.
  - Obsidian: Lucide `arrow-up-right` plus "Jump", in `--font-ui-smaller` at weight 600,
    colored `--text-on-accent` on `--interactive-accent`.
  - Mac: SF Symbol `arrow.up.forward` plus "Jump", on `Color.accentColor` with white
    text. The Mac already uses this glyph for "open in Obsidian".
  - The capsule is a pill sitting at the row's trailing edge, aligned with the title
    line. Other row styling stays unchanged: no extra tint and no leading rule.
- **Motion:**
  - Obsidian fades and scales the capsule in over 120 ms (0.92 → 1).
  - On the Mac, the capsule glides between rows with `matchedGeometryEffect`.
  - Under reduced motion, both surfaces cross-fade instead. Arrival reuses the existing
    `.bob-landing-flash` (1.2 s, or 600 ms with reduced motion).
- **Footer, the safety net:**
  - The summary line gets ` · ↗ “S”` appended, with S truncated to 40 characters plus
    `…`. A pin hidden by the filter is therefore never a surprise.
  - The Ctrl+J hint chip sits last before Esc.
- **Copy table:**

  | Moment                  | Obsidian                                                | Mac                                                 |
  | ----------------------- | ------------------------------------------------------- | --------------------------------------------------- |
  | hint chip               | `Ctrl+J` jump                                           | `⌃J` Jump                                           |
  | Enter, pin only         | `↵` leave open & jump                                   | button "Leave Open & Jump ↩"                       |
  | Enter, pin + marks only | `↵` pull in & jump                                      | "Pull In N & Jump ↩"                               |
  | Enter, pin + verdicts   | `↵` apply & jump                                        | "Apply & Jump ↩"                                   |
  | set (status/VO)         | `↗ Jumps to “S” after the close`                       | "Jump set to S"                                     |
  | cleared                 | `Jump cleared`                                          | "Jump cleared"                                      |
  | not pinnable            | `Jump works on offer rows only` (refusal + shake)       | same text in the refusal strip                      |
  | refresh lost the key    | `Jump target is no longer offered — cleared`            | same                                                |
  | row a11y suffix         | `jump target`                                           | ", jumps here after capture"                        |
  | landed receipt line     | `↗ “S”`                                                | notification body line `↗ S`                       |
  | lost after commit       | `Couldn't find “S” to jump to — your changes are saved` | notification line `↗ Couldn't find “S” to jump to` |

  On the Mac, the ⇧↩ skip button's existing label gets " & Jump" appended while a pin
  is staged, and the `⌃↩ Pull In & Open` hint is hidden (⌃↩ then behaves like ↩).

- **Pointer parity:**
  - Obsidian adds a third hover button `↗` after ✓ and ⊘, with aria-label "Jump to S
    after the close".
  - The Mac adds "Jump Here After Capture" / "Clear Jump" to the row context menu, plus
    a matching `.accessibilityAction(named:)`, never a Button nested in the row Button.

## cli-jump-offer: bob capture --jump-offer and the jump JSON contract

This phase runs in bob-cli. Contract first: this phase writes the cross-surface docs
that the other phases implement.

**CLI.** Add `jump_offer_arg` in `src/native/capture/cli.rs`:

- Spelling: `-j, --jump-offer <N|KEY>`. `-j` is free.
- Arity: one value, not repeatable.
- Help text: "Focus this offer after the capture (any verdict); one offer".
- Placement: alphabetical, following `cli_rules.md` (read it).

Register the flag everywhere a capture flag is listed:

- `CaptureRequest` gets a `jump_offer: Option<String>` field. Update every struct
  literal (`project_note.rs`, `task_blocks.rs`, `inbox_route.rs`, `budget.rs`,
  `tests/template.rs`, `tests/album_index.rs`).
- `capture_flag_guard.rs`: the `LONGS` list and `short_values`. This stops an un-probed
  old binary's flag guard from eating the flag as capture text.
- `completion/kinds.rs`: a `FreeText` entry like `pull-in`.
- The `capture_help_lists_options_alphabetically` order test.

**Selection.**

- Resolve the selector in the same probe that resolves `-p`/`-x`/`-X`: the `pull_in.rs`
  `plan_with_pull_in` probe, `key_for_number`, and `surviving_keys`, so numbers match
  the card. All digits means a number; anything else is a key.
- An unknown or stale selector, or `-j` when there are no offers, refuses with **no
  writes** using a new `offer_jump_stale` error built in `output.rs` beside
  `offer_close_stale`. Copy that error's shape:
  - kind: Usage, exit 2;
  - message: first line plus `offer_list_lines`;
  - `extra = {pull_in, offers}`.
- `-j` is **not** a pull-in choice. Without `-p`, a capture that requires a choice still
  refuses `pull_in_choice_required`; `-p none -j N` is the "leave open and jump" form.
- Use the same `-p`/`-x`/`-X` conflict rules. The jump's outcome is derived:
  `"pulled_in"` if picked, `"done"`/`"cancelled"` if closed, else `"left"`.

**Capability.**

- Add `jump: PullInJumpJson { supported: true }` to `PullInJson` (`output.rs`), set in
  both `PullInJson::empty()` and `assemble_pull_in`.
- Gating is by field presence, the same pattern as `close`.

**Result JSON.**

- Add top-level `jump: Option<CaptureJumpJson>` to `CaptureResult` (skipped when None),
  built in a new `src/native/capture/jump.rs` beside `build_open_notes`, from the
  **final staged text**.
- Call it after the plan freezes, the same place `build_open_notes` is called in
  `mod.rs`.
- Shape:

```json
"jump": {
  "key": "dev.md#^fixmig",
  "n": 2,
  "label": "Fix the migration",
  "outcome": "left | pulled_in | done | cancelled",
  "landing": { "path": "/abs/dev.md", "relative_path": "dev.md", "role": "note",
               "label": "dev", "line": 42, "ref": "42:35a733bd", "block_id": "fixmig" }
}
```

`landing` uses the same struct as `OpenNoteJson`. It is `null` when the target can't be
found again; this is a lost target, and the capture still succeeds.

Re-find the landing in this order:

1. A `batch.routed_landings` move for the row (an inbox-routed pull-in follows its move,
   as `routed_landing` does).
2. The row's block ID, including one minted by the pull-in, via `line_for_block_id` on
   staged text.
3. Otherwise, a match on cleaned description plus settled ordinal among ID-less tasks.
   Reuse `successor_task_lookup`'s rule; do not fork it.

Then fill `line`, `ref` (`TaskRef` line plus digest of the physical line), and
`block_id` exactly as `landed_note` does. Never use `OfferCloseJson.line` or a
pre-post-pass `row.line`: Cancel Logs, Task Link inserts, mention mirrors, and type
agreement all shift lines after those were recorded.

**Human output.**

- One receipt line after the existing ones: `↗ jump  “S”  dev.md:42`.
- When lost: `↗ jump target “S” not found after writes`.
- Use the existing receipt colors and helpers.

**Dry-run.** `--dry-run` emits the same `jump` object. That is parity by construction,
since it is built pre-commit from staged text.

**Vectors.** Add these as CLI tests in `tests/cli/capture/` (new `successor_jump.rs`),
documented as **§12.24 JS vectors**:

| #    | Case                                                          | Expect                                                                      |
| ---- | ------------------------------------------------------------- | --------------------------------------------------------------------------- |
| JS1  | `-p none -j 2` on a recovering dependent                      | no pick; outcome `left`; landing on the `[unblocked::]`-stamped line        |
| JS2  | `-p 2 -j 2`                                                   | `pulled_in`; landing `block_id` is the minted ID                            |
| JS3  | `-x 2 -j 2`                                                   | `done`; landing on the `[x]` line                                           |
| JS4  | `-X 1=why -X 2 -j 2`, both in one note, offer 1 above offer 2 | `cancelled`; landing is still exact after offer 1's Cancel Log shifts lines |
| JS5  | pulled-in inbox row routed by `-i`                            | landing follows the routed move to the destination note                     |
| JS6  | `-j 9` (no such offer), and `-j` by a stale KEY               | `offer_jump_stale`, exit 2, `extra.offers`, vault byte-identical            |
| JS7  | JS2 with `--dry-run`                                          | same `jump` as the real run                                                 |
| JS8  | `-j 2` without `-p` when the choice is required               | `pull_in_choice_required`                                                   |
| JS9  | `-j` by KEY for an ID-less keep-moving/note-pull row          | landing found by cleaned description                                        |
| JS10 | any preview                                                   | `pull_in.jump.supported == true`                                            |

**Docs (bob-cli).**

- `docs/task-dependencies.md`: a new **§12.24 "Jump successor"**. Cover the shared
  semantics, the commit table, precedence, the re-find order, the CLI and JSON contract,
  `offer_jump_stale`, the capability, and the JS vectors.
- In §12.13 (Obsidian picker API), add a **v5** row:
  - model `jump: {supported: true}` opts the card in;
  - `pick()` resolves `{picks, closes?, jump?: {key}}`;
  - jump-only is a commit, not `null`.
- In §12.14 (CLI selection), add `-j`.
- `docs/capture.md`: the stable JSON field list gets `jump` and `pull_in.jump`.
- Run `just docs-gen` (CLI reference) and `just docs-check`. Read `docs.md` memory
  first.
- Do not edit `Cargo.toml`'s version or `CHANGELOG.md` (release train). Use a
  `feat(capture):` commit subject.

**Verification.** Run `cargo test` (the new file plus the existing `successor_verdicts`,
`open_notes`, `inbox_route`, `flag_guard`, and `help_options` suites),
`just docs-check`, and clippy as the repo's CI runs it.

## nav-pin-card: Obsidian successor card pin (picker API v5)

This phase runs in bob-plugins, plugin `bob-navigation-hotkeys`. Read bob-plugins
`AGENTS.md`.

Fragments must stay at or below 1000 lines. `206-successor-picker-modal.js` is at 998,
so:

1. **First, extract with no behavior change.** Move `createSuccessorPickerApi` and
   `BobNavigationHotkeysSuccessorPickerMixin` out of 206 into a new
   `206f-successor-picker-api.js`, registered in `src/fragments.json` after the 206
   group. `690-install-plugin-mixins.js` already references the mixin by name. Run
   `npm run build` and the picker tests; the build output must be behaviorally
   identical.
2. **Then add the pin.** Put it in a new `206e-successor-picker-jump.js`, following
   206c's prototype-patch pattern (`successorInstallJumpModalPatches`, installed from
   `onOpen` next to the verdict patches). 206 itself gains only one-line hooks.

The pin's behavior:

- **State:**
  - `this.jumpKey = null` on the modal.
  - Enabled only when the normalized model has `jump.supported === true`
    (`normalizeSuccessorPickerModel` in 206a passes `jump` through).
  - Callers that don't opt in see no chip, and Ctrl+J falls through to today's behavior.
- **Key handling** (`handleJumpKey`, called from `handleKeydown` before
  `handleVerdictKey`):
  - Matches literal Ctrl (no Shift, Meta, or Alt) with `key` "j" or `code` "KeyJ".
  - Ignores repeat, IME composition, and keyCode 229, using the existing
    `successorVerdictShouldIgnoreEvent`.
  - When a reason edit is active (`reasonEditKey != null`), it consumes the key and does
    nothing.
  - Otherwise it calls `preventDefault`/`stopPropagation`, so it also works while the
    filter input is focused.
  - It toggles or moves the pin per the shared semantics, sets the status line (copy
    table), resets `escArmed` only if verdicts are staged (same as the other keys), and
    re-renders.
  - On a non-pinnable row it shows the refusal status and calls `successorShakeRow`.
- **Enter and commit:**
  - The Enter branch and `applySelection` treat a staged pin like staged verdicts:
    commit, with no highlight fallback.
  - `prospectiveKeys()` returns `[]` when a pin is staged with no marks, so the footer
    count and plan meter don't show a phantom pull-in.
  - `applySelection` settles `{picks, closes?, jump: {key}}`.
  - `settle()` keeps `jump` on the result.
  - The mixin's `pickSuccessorOffers` settle validates `jump.key` against the model's
    selectable keys (drop it if invalid) and resolves non-null when picks, closes,
    **or** jump exist.
  - `createSuccessorPickerApi` goes to **version 5**, and its `pick()` filter no longer
    drops a jump-only result.
- **Esc:**
  - Unchanged. With verdicts staged, the first Esc still arms and the second discards
    the verdicts; the pin stays, like marks do.
  - The Esc that closes the card settles `null`, so the pin is discarded.
- **Render:**
  - Rows render with the `is-jump-pin` class and a trailing
    `span.bob-successor-jump-pin` capsule (icon via `applyIcon(…, "arrow-up-right")`
    plus the text "Jump").
  - The hover action group gets a `↗` button: render the actions container on
    jump-enabled cards even when verdicts are not active. Its click mirrors Ctrl+J for
    that row.
  - The aria-label gets the `jump target` suffix.
  - The footer hint list (`successorVerdictFooterHints` in 206d, or a wrapper in 206e)
    inserts `Ctrl+J jump` before `Esc`, and the Enter chip label follows the copy table.
  - The footer summary appends a `span.bob-successor-jump-summary` (accent colored,
    ellipsized).
- **CSS** (`styles.css`):
  - The capsule pill, with `margin-left: auto`, `align-self: flex-start`, and
    `white-space: nowrap`.
  - The `bob-successor-jump-in` keyframes (120 ms opacity plus scale), disabled under
    `prefers-reduced-motion`.
  - The summary span and the hover button tint.
- **Tests:**
  - Extend `scripts/test-navigation-successor-picker.cjs` and `-successor-verdicts.cjs`:
    - toggle, move, and clear;
    - the pin survives verdicts, marks, and filter;
    - Ctrl+J is inert in the reason editor and refused on muted rows;
    - pin-only Enter resolves `{picks:[], jump}` while plain Enter still pulls in;
    - pin plus closes;
    - Esc discards;
    - repeat and IME are ignored;
    - the v5 `pick()` passes jump-only results;
    - a model without `jump.supported` shows no chip, and Ctrl+J changes no state.
  - Run `npm test`.
- **Docs and release:**
  - `docs/bob-navigation-hotkeys.md` successor picker keys: add Ctrl+J.
  - `docs/develop/plugin-apis.md`: successors v5. Also fix the stale `notice` v3 row to
    say v4.
  - Run `npm run docs:check`, bump the plugin's manifest version, then
    `bob plugins sync`.

## nav-jump-landing: Obsidian landing API, walk follow, and receipt line

This phase also runs in `bob-navigation-hotkeys`. Fragments 480 (996 lines), 285 (996),
and 536 (977) are near the cap, so put new logic in new fragments (for example
`676a-task-landing.js` and `536b-review-walk-follow.js`, with any new mixin registered
in `690-install-plugin-mixins.js`). Add only one-line hooks to the full fragments.

**`api.landOnTask(target, options)`.** An additive, frozen nav API member; the root API
stays v3.

- **Signature:**
  - `target = {path, blockId?, text?, line?, description?, label?}`, where `path` is
    vault-relative.
  - It resolves `{ok, reason?, line?}` and never throws.
- **Steps:**
  1. Resolve the file. If it is missing, the result is `reason: "lost"`.
  2. Re-find the line with an anchor `resolve(text)`, which
     `resolveTaskMoveDestinationLine` already supports. Try, in order:
     - planned `line` when that line equals `text`;
     - `blockId`;
     - a unique exact `text` line;
     - a unique cleaned-description match, ignoring status char, inline `[k:: v]`
       fields, emoji dates, and trailing `^id`. Reuse an existing nav task-text
       normalizer if one exists.
     - Otherwise the target is **lost**: no clamping.
  3. Call `focusTaskAnchor` with `ensureEditing: true` and a Vim jump context created
     from the active cursor, so `<C-o>` returns to the closed row.
  4. Flash the line with `flashBobOpenNotesLandingLine`.
- **When lost:** show one Notice using the copy table's lost line, and resolve
  `{ok: false, reason: "lost"}`.
- **Never touches the walk.** It must never set `reviewLanding`, stamp anything, or
  consume a review landing.

**`reviewWalk` v2.** In `createReviewWalkApi`, set `version: 2`. When
`continue(origin, outcome)` receives `outcome.follow` (a `landOnTask` target), it routes
to a new `followReviewWalkAfter(origin, outcome)`:

- **Fresh origin on a landing:**
  1. Park first: generalize `parkReviewWalkAfterMove` (537) so it accepts the answered
     row plus `outcome.excludeRefs` as handled refs. Reuse it; don't copy it. Parking
     must come before focusing, because opening another note clears the landing.
  2. Then `landOnTask(follow)` with a Vim jump from the answered row.
  3. Then show **one** toast: `outcome.notice` plus the `↗ “S”` line.
  4. Settle the gesture lock **after** the landing resolves.
  5. Resolve `{ok, advanced: false, parked: true, landed}`.
- **Stale origin or off-landing:** show the notice and still land on `follow` (it is the
  user's explicit intent). Settle only this origin's own lock.
- **Without `follow`:** v1 behavior, byte for byte.

**Notice v5.** `showUnblocked(model)` renders one trailing `↗ “S”` line when
`model.jump && model.jump.label` exist. Put the rendering in a new fragment with a
one-line hook in 285, and bump `notice` to version 5.

**Tests:**

- New `scripts/test-navigation-task-landing.cjs`:
  - each re-find path;
  - a lost target notices and does not clamp;
  - the flash fires;
  - the Vim jump is recorded;
  - the review landing is untouched.
- Extend `test-navigation-review-advance-gestures.cjs`:
  - follow parks, and the next `]s` resumes at the closed row's neighbour;
  - the lock is held until landing;
  - one toast;
  - a stale origin still lands;
  - no `follow` means v1 behavior.
- Extend `test-navigation-hotkeys-unblocked-notice.cjs` with the jump line.

Update `docs/develop/plugin-apis.md` (landOnTask, reviewWalk v2, notice v5), then run
`npm test`, `npm run docs:check`, the version bump, and `bob plugins sync`.

## cycler-jump-close: task-status-cycler honors the pin after every managed Done

This phase runs in bob-plugins, plugin `task-status-cycler` (fragments in
`plugins/task-status-cycler/src`).

**Opt-in gate.** Set `jump: {supported: true}` on the picker model
(`buildSuccessorPickerModel` in 085b and `pickSuccessorOfferChoices` in 137a) only when
all of these hold:

- nav `successors.version >= 5`;
- `typeof nav.landOnTask === "function"`;
- either the context is **not** walk-composing (`successorNoticeTarget !== "return"`),
  or the caller opted in with `successorJumpTarget: "return"` **and** nav
  `reviewWalk.version >= 2`.

A walk-composing caller that has not opted in therefore never shows Ctrl+J. That avoids
two landings while the callers ship one phase at a time.

**Result handling.**

- `normalizeSuccessorChoices` (089g) keeps `jump: {key}` when `key` is a non-empty
  string. It returns non-null when picks, closes, or jump exist.
- `planAndApplySuccessorsNow` (137) reads `choices.jump` and runs the existing write
  path unchanged. A jump-only result still runs automatic recovery exactly as "no picks"
  does today.

**Post-write identity.** After the close pass and `removeResolvedOfferCloseRows`, build
`model.jump = {key, outcome, label, path, blockId?, text?, line?, description}`:

- **Picked:** the plan row (`unblocked`, `keep_moving`, or `relation_successors`) whose
  `offer.key === key`. Use its `note_path` and `block_id`, which includes an ID minted
  for the pick.
- **Closed:** the `closeResult.resolved` / `offer_closes` entry for the key. Use the
  post-write raw line the close writer produced. Verify it is post-write; if not, use
  the edited text the writer staged.
- **Left:** the original offer (`originalOfferByKey`) for path and block ID. For text,
  use the `after` text of the `plan.edits` entry for that path/line (the recovery
  `?`→` ` change or the `[unblocked::]` stamp) if one exists, else the offer's original
  line.
- **Refused stale pick:** still jump, with the "Left" identity.
- Always include the original line as `line` (a hint only) and the cleaned description
  for the fallback.

**Landing.** Add a `landSuccessorJump(model, context)` method, called once after
`presentSuccessorModel`:

- Call sites: `finalizeClosedTasks` (130, after reference retirement and the notice) and
  `offerParkedSuccessors` (139).
- If `context.successorJumpTarget === "return"`, it only stores the jump in
  `context.successorJumpBox.jump`.
- Otherwise it awaits `nav.landOnTask(model.jump)`.
- `finalizeClosedTasks` resolves with `jump` as well.
- When the receipt is shown, pass `jump: {label}` into the notice model so nav v5
  renders the line. Walk-return text gets the `↗ “S”` line appended.

**Callers and API.**

- **Vim Ctrl+Enter walk path** (140): pass `successorJumpTarget: "return"` and a
  `successorJumpBox`. In the settle, add `follow: box.jump` to the `complete` outcome,
  so the walk parks and lands.
- **`completeTaskAtCursor(editor, options)`** (160): accept an optional
  `options.successorJumpTarget === "return"` and return `jump` in its result.
- Bump the cycler API from v5 to **v6**. Feature-detected; documented in bob-plugins
  `docs/develop/plugin-apis.md`.
- **Parked successors:** confirm that block-id-prompt's caller
  (`130-plugin-task-link-open-and-notices.js`) does not move focus after
  `offerParkedSuccessors` resolves. If it does, make the jump win.
- **Coverage audit:** list every other `finalizeClosedTasks(` and
  `successorNoticeTarget` caller. Non-walk callers get the self-landing for free;
  walk-composing callers stay jump-less until opted in (the nav-jump-callers phase opts
  in the two in nav).

**Tests:**

- Extend `test-task-status-cycler-successor-wiring.cjs`, `-successor-apply.cjs`,
  `-successor-verdicts.cjs`, `-parked-successors.cjs`, and `-ctrl-enter.cjs`:
  - each outcome's identity;
  - jump-only runs recovery and lands;
  - the jump lands after retirement and after the notice;
  - the walk path passes `follow` and does not land itself;
  - the gate hides the pin from a non-opted walk-composing caller and from an old nav;
  - the parked path lands.
- Update `docs/task-status-cycler.md` ("Successor offers and take-backs"). Then run
  `npm test`, `npm run docs:check`, the version bump, and `bob plugins sync`.

## nav-jump-callers: Wire the remaining Obsidian picker callers

This phase runs in `bob-navigation-hotkeys`.

- **PRE/POST checklist claim** (535 → cycler `completeTaskAtCursor`):
  - When cycler `api.version >= 6`, pass `{successorJumpTarget: "return"}`.
  - When the result carries `jump`, continue with `follow` (park plus land) instead of
    the `ctrl-enter-leaves-pre` advance.
- **Relation-complete compose** (597):
  - Pass `successorJumpTarget: "return"` with a box to `api.finalizeClosedTasks`.
  - If that flow continues a walk, pass `follow`; otherwise call
    `landOnTask(actual.jump)` once after its own notice.
- **Project promotion** (667/667a, 122):
  - Set `jump: {supported: true}` on the promotion picker model.
  - Keep a by-key map of the plan rows' `notePath`, `line`, `rawLine`, and `blockId`
    (122:226-247 currently drops them from model rows). After the picks and closes
    apply, build the identity: `blockId` is the row's ID or `mintedByKey[key]`, and
    `text` is the post-close text when closed.
  - Call `landOnTask` **instead of** the final `^prj` landing. With no pin, keep landing
    on `^prj` as today.
- **Relation merge** (429a): leave the pin withheld. Its rows are hand-off slots for one
  shared merge target. Add a test that Ctrl+J shows no chip and changes nothing.
- **Tests:** extend `test-navigation-checklist-ctrl-enter.cjs`,
  `test-navigation-hotkeys-forward-promotion.cjs`, `-promotion-verdicts.cjs`, and
  `test-navigation-relation-merge.cjs`.
- **Docs and release:** document the promotion behavior in
  `docs/bob-navigation-hotkeys.md`, then run `npm test`, `npm run docs:check`, the
  version bump, and `bob plugins sync`.

## mac-jump-pin: Mac Capture Pull-In card pin

This phase runs in bob-mac-capture. Read its `AGENTS.md`: never land Swift you have not
compiled, and the macOS CI gate applies.

- **Core state** (`Sources/CaptureCore/CapturePullInOutcomes.swift`):
  - Add `jumpKey: String?` with `toggleJump(key:)` (radio toggle and move).
  - `refresh(rows:)` drops a lost `jumpKey` and reports it.
  - `effectivePickKeys` uses the highlight fallback only when marks, verdicts, **and**
    the jump are all empty.
  - Labels and footer summary follow the copy table.
  - Esc arming ignores the jump.
  - The `CapturePullInChoiceState` wrappers (`effectiveSelectionKeys`,
    `primaryCommitLabel`, `footerSummaryText`) follow the same rules.
- **Router** (`CaptureKeyCommandRouter.pullInChoiceCommand`):
  - `KeyCode.j` with `modifiers == .control` maps to a new `.togglePullInJump`.
  - While the reason field is focused, Ctrl+J maps to `.consumeKey`.
  - When `pullIn.jump?.supported != true`, `.togglePullInJump` shows the refusal strip
    "Jump needs a newer bob" and the hint chip is hidden.
  - Pin `testRouterOwnsChoiceKeys…`, and check that the editor's ⌃J bullet-newline is
    unaffected outside the card.
- **Commit** (`CapturePanelModel`):
  - `confirmPullInChoice` accepts a jump-only selection: the "No visible successor offer
    is selected" guard passes when `jumpKey != nil`, and the request sends
    `--pull-in none`.
  - `chooseNoPullInOffers` keeps the jump.
  - `BobProcessClient.capture` appends `--jump-offer KEY` only when
    `pull_in.jump.supported` is advertised; otherwise it throws `unsupportedOption`,
    like the verdict flags. Pin the argv order test.
  - `offer_jump_stale` joins the `offer_close_stale` family in
    `reopenPullInChoiceAfterOfferRefusal`. The stored choice (with `jumpKey`) reopens,
    and the refresh clears the stale pin visibly.
- **Success** (`completeSubmit`):
  - Decode `jump` with `decodeIfPresent`
    (`CaptureJump {key, n, label, outcome, landing: CaptureOpenNote?}`).
  - If the commit carried a jump and `landing` exists, open
    `ObsidianOpenURL.openNotesURL(for:)` built from a **one-entry** plan with that
    landing, through `noteOpener`, and skip the `openAfterCapture` open-notes open. This
    reuses the plugin's existing 2 s write-freshness wait, leaf reuse, Vim jump, and
    landing flash.
  - If `landing` is null (lost), fall back to today's behavior: open notes only if
    `openAfterCapture`. The notification adds the lost line.
  - `NotificationService.successContent`: with a landed jump, the default click target
    is the jump URL and the body adds `↗ S`. The "Open N Notes" action still opens the
    captured notes.
- **View** (`CapturePullInChoiceView.swift`):
  - The jump capsule goes after `trailingCapsule` in `CapturePullInOfferRow`, with
    `matchedGeometryEffect(id: "jump-pin", in:)` from the card's `@Namespace`, and a
    cross-fade under Reduce Motion.
  - Context-menu and accessibility actions.
  - The row `accessibilityLabel` gets its suffix, and toggles post VoiceOver
    announcements.
  - Footer: `hintItems` gets `⌃J Jump` before esc and drops `⌃↩ Pull In & Open` while
    pinned. Primary and skip labels follow the copy table, using the existing
    `arrow.up.forward` glyph.
- **Tests:**
  - `CapturePullInOutcomesTests` (toggle, move, clear; fallback suppression; refresh
    loss).
  - `CapturePullInClientTests` (argv and capability gate).
  - `CapturePullInDecodingTests` (a `jump` fixture plus a lost landing).
  - `CapturePullInChoiceTests` (router; footer contract
    `testFooterContractPinsHintsAndAccessibilityText`).
  - `CapturePullInChoiceModelTests`, against recorded fake bob:
    - pin-only Return sends `--pull-in none --jump-offer`, and the injected `noteOpener`
      receives a one-entry `bob-open-notes` URL;
    - ⌘↩-submitted draft plus pin opens only the jump;
    - stale reopen keeps the card state;
    - a lost landing falls back.
  - `CaptureInboxRouteChoiceModelTests`: Back restores the pin.
  - `CapturePullInChoiceDesignTests`: a pinned-row snapshot.
- **Docs:** `docs/keyboard.md` (Successor choices), `docs/capture-panel.md`
  (Control-Return section: the pin wins), `docs/protocol.md` (the `--jump-offer` gate
  and `jump` decoding), and `docs/capture-previews.md` (Successor offers). Run
  `just docs-check`.
- **Gates:**
  - Run `just portable-check`.
  - Compile a scratch copy on the mac (best effort: `bundle.sh` plus
    `just typecheck-tests`); say so if the mac is offline.
  - Run `Scripts/macos-ci-check` before landing.
  - After the push, watch the exact-SHA macOS run to green with `/sase_monitor`.

## jump-docs-memory: Walk docs, decision record, and glossary

This phase runs in bob-cli, with read-only checks of the two linked repos' docs.

- **`docs/freshness.md` §6 (review ritual):**
  - A close answered from a `]s` landing whose successor card committed with a jump pin
    lands on the pinned task and parks the walk. The next `]s`/`[s` resumes at the
    closed row's neighbour.
  - Focusing the pinned task never stamps it.
  - Add a §13 rollout-log row naming the plugin versions shipped by the epic.
- **Cross-check:** `docs/task-dependencies.md` §12.24, bob-plugins'
  `docs/bob-navigation-hotkeys.md` and `docs/develop/plugin-apis.md`, and
  bob-mac-capture's `docs/keyboard.md` must agree on keys, copy, and API versions. Fix
  drift in bob-cli directly. For a sibling repo, record `PROPOSED FOLLOW-UP:` on the
  phase bead rather than editing it.
- Run `just docs-check`.

> [!decision] walk_park_record Write a new decisions strand,
> `sase/memory/decisions/successor-jump-parks-the-walk.md`, titled **"A Jump Successor
> Lands Last And Parks The Walk"**. Follow the decisions descriptor's record format.
>
> - **Applies to:** bob-plugins (nav, cycler), Bob Mac Capture, bob-cli docs, and the
>   review ritual.
> - **Claim:** Ctrl+J pins at most one successor offer per card. The pin is navigation
>   only. After every write lands, Bob focuses that task. A pinned close that started on
>   a `]s` landing parks the walk at the closed row's neighbour instead of advancing.
>   The pin wins over Mac Capture's open-notes landing.
> - **Why:** the research bottom line, plus the precedent in
>   [[decisions/task-move-never-advances-the-walk]].
> - **Rejected:** keying the pin off the ⌘↩ submit modifier; landing after the walk
>   advance (two landings); persisting the pin in the vault.
> - **Evidence:** the research ref and this plan.
> - **Cost:** one `]s` to resume.
> - **Reopens when:** Bryan presses `]s` after nearly every pinned close.
>
> Then mark `decisions/answering-advances-the-walk` and
> `decisions/ctrl-enter-leaves-pre`:
>
> - add the new strand to `superseded_by`;
> - keep `metadata.status: superseded-in-part` (set it on `ctrl-enter-leaves-pre`);
> - add a one-line `[[...]]` back-link stating that pinned closes park instead.
>
> Use `/sase_memory_write`, then run `sase memory init`. When this decision is off,
> record `PROPOSED FOLLOW-UP:` on the phase bead instead.

> [!decision] jump_glossary_term Add a glossary strand (slug `jump-successor`): **Jump
> Successor** (aka jump pin, jump target). It is the one successor offer pinned with
> Ctrl+J on the successor card. After the close or capture commits, Bob focuses it in
> Obsidian, whatever its verdict. It is never a vault state.
>
> - Link [[successor-offer]] and [[task-successor]].
> - Keep "mark" (pull in) and "pin" (jump) distinct.
>
> Then run `sase memory init` and `just docs-gen` (which regenerates
> `docs/reference/glossary.md`) and `just docs-check`.

## Out of scope

- Muted-row pins: a navigation-only key for still-blocked or future-scheduled rows,
  which needs a bob contract change.
- Removing Ctrl+J's next-row alias in the Task Wikilink picker. That collision is
  tolerable: visible, reversible, and nothing is written until Enter.
- Any vault-persisted "jump" state.
