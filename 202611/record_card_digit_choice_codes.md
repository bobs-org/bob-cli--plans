---
tier: tale
title: Record Card editor digits pick choices with two-digit codes
goal:
  Typing a choice's number in the Record Card editor picks and saves it, with
  prefix-filtered codes from 10 choices.
size: medium
decisions:
  browse_digits:
    ask:
      With 10+ choices and no stage open, should the first digit open the choices stage
      filtered to it?
    choices:
      stage:
        First digit opens the choices filtered to that prefix; the second digit picks
        and saves
      status:
        Stage stays closed; the status line shows the pending prefix until the second
        digit saves
      legacy:
        Browse keeps today's 1–9 instant pick; two-digit codes work only inside the
        choices stage
    default: stage
    why:
      One key sequence everywhere, and the filtered list shows what the next digit can
      reach
    answer: stage
  badge_format:
    ask: With 10+ choices, should digit badges be zero-padded to the code width (01…17)?
    choices:
      padded: Badges show exactly what you type (01, 07, 17), so the column lines up
      plain:
        Badges show 1…17 as today; choices 1–9 still need a leading 0 that badges don't
        show
    default: padded
    why: The badge is the code you type, so nothing has to be guessed
    answer: padded
proposed_by: bbugyi200.apollo.fe
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 07:38:13
status: wip
---

# Plan: Record Card editor digits pick choices, with two-digit codes from 10 choices

## Problem

In the Record Card editor's choices stage, focus sits in the **Filter choices** input,
and today every digit goes into it as filter text. The screencast shows this: on a
`Section` field with 17 choices, typing `8` filters the list to **No matching choices.**
Backspace then has to remove it. Only the first nine choices have badges (`1`–`9`), and
positions 10+ can only be reached by navigating or clicking. The current docs say this
on purpose ("digits in the filter stay text. Positions past nine remain reachable by
navigation and click"). This plan replaces that rule.

Desired behavior:

- Typing a choice's number in the editor selects that choice and saves it.
- With **≥10 choices**, each choice has a two-digit code. The first digit is a filter:
  typing `0` shows only choices 1–9, typing `1` shows 10–19. The second digit confirms:
  `0` then `7` picks choice 7.

## Where the code lives

All work is in the **`bob-plugins`** linked repo (open it with the `/sase_repo` skill:
`sase repo open bob-plugins -r "<reason>"`). Read its `AGENTS.md` first.
`bob-ledger-tools` uses the fragment build: edit `plugins/bob-ledger-tools/src/`, run
`npm run build`, and never hand-edit `main.js`. Each hand-edited fragment must stay at
or below 1000 lines. This plan does not touch bob-cli. Its `docs/reference/plugins/`
copy is a gitignored mount of bob-plugins docs.

Relevant fragments (all under `plugins/bob-ledger-tools/src/`):

- `242n-plugin-record-card-model.js`: pure helpers. `recordCardChoiceList` assigns
  `digit: index < 9 ? index + 1 : null`. Also here: `recordCardFilterChoices`,
  `recordCardChoiceByDigit`, `recordCardHintItems` (browse hint `1–N pick` capped at 9;
  the choices-stage hints), `recordCardCaptureStage`, `recordCardLiveAnnouncement`, and
  `RECORD_CARD_DIGIT_TEACH`.
- `242o-plugin-record-card-dom.js`: `recordCardPaintChoices` paints the filter input,
  the listbox, and a `kbd.bob-record-card-choice-badge` only when `choice.digit` is set.
- `242p-plugin-record-card.js`: the `RecordCardModal` class (constructor state,
  `recordStatusModel`, `recordHandleEscape`, `recordMoveChoice`). Its own
  `recordOnKeyDown` and `recordDigit` are **shadowed**: 242r reassigns them on the
  prototype.
- `242r-plugin-record-card-typed-stages.js`: the **live** `recordOnKeyDown`,
  `recordDigit`, `recordPaintOptions`, `recordOpenStage`, and `recordCancelStage`.
  Inside the filter input, the `recordInInput` branch handles only Tab, Enter, Esc,
  arrows, and Ctrl+N/P, and lets every other key through as text. That pass-through is
  why `8` becomes filter text. The browse branch and the "stage open, focus outside the
  input" branch each test `/^[1-9]$/` and call `recordDigit`.
- `242s-plugin-record-card-typed-save.js`: the `input` listener (a `choice-filter` edit
  sets `choiceFilter`), `recordCancelStageSilent`, the Tab save-and-advance path that
  reopens a choices stage on the next field, and the `recordRefresh` keep-stage
  capture/restore.
- `372-exports-record-card.js`: test exports (`recordCardChoiceList`,
  `recordCardChoiceByDigit`, `RECORD_CARD_DIGIT_TEACH`, …).
- `plugins/bob-ledger-tools/styles.css`: `.bob-record-card-choice-badge`
  (`min-width: 1.2em`).

## Design

### Choice codes

- Each choice gets a stable **code** from its schema-order position, after empty-value
  choices are dropped so positions stay contiguous. The code width is the digit count of
  the total choice count: 1 below 10 choices, 2 for 10–99, 3 for 100–999. This
  generalizes the two-digit rule without a special case.
- Below 10 choices, codes are `1`…`9`, exactly as today.
- With 10 or more, every choice is badged; no position goes unbadged any more.

> [!decision] badge_format = padded The badge text is the code itself (`01`, `07`,
> `17`). Badges use tabular numerals so the column lines up.

> [!decision] badge_format = plain The badge text is the unpadded position (`1`…`17`).
> Typed codes are still fixed-width, so choice 7 is still `0` `7`.

- Codes never change while text filtering or prefix filtering. Filtering hides rows but
  never renumbers them, so the existing "badges stay in schema order" rule still holds.

### Digit entry (one pure helper drives every surface)

Add a pure helper to `242n`, for example
`recordCardDigitStep({ choices, visible, prefix, digit })`. `choices` is every choice;
`visible` is the text filter applied. The helper returns one of:

- `{ kind: "pick", choice }`: `prefix + digit` is exactly a full-width code of a
  **visible** choice.
- `{ kind: "prefix", prefix }`: `prefix + digit` is shorter than the code width and at
  least one visible choice's code starts with it.
- `{ kind: "none", typed }`: no visible choice matches. The caller ignores the key,
  keeps the current prefix, and teaches.

Rules:

- A choice is picked **only** when the full code is typed. Nothing auto-picks when the
  prefix leaves one row. Example: with 10 choices, `1` leaves only `10`, but you still
  type `0`. Auto-picking there would make the second digit start a new prefix.
- The pending prefix is separate state (`choiceDigitPrefix`), not filter text. Visible
  rows are the text filter intersected with the prefix. Put this in one place, either an
  optional prefix argument to `recordCardFilterChoices` or a small wrapper. Every
  visible-list consumer must use it: `recordPaintOptions`, `recordMoveChoice`, the digit
  checks, and the hints.
- When the prefix narrows the list and hides the highlighted choice, the highlight moves
  to the first visible row. Enter, arrows, and Ctrl+N/P then act on what is shown.
- The prefix has no timeout. It is cleared by any of:
  - a pick
  - Backspace (pops one digit)
  - a text edit in the filter (the `choice-filter` input event)
  - cancelling, closing, or switching the stage, including the Tab save-and-advance
    reopen
  - `recordCancelStageSilent`
- The prefix survives a background `recordRefresh` with keep-stage: add it to the
  captured and restored stage in `242s` and in `recordCardCaptureStage`. It is dropped
  on restore if it no longer matches any visible choice.
- Ignore `event.repeat` for digit keys, so a held `0` never becomes `00`.
- IME and composition guards stay as they are. Ctrl, Alt, and Meta digit chords stay
  untouched (`recordBare`).
- Replace `recordCardChoiceByDigit` with a code-based lookup, and update the exports and
  tests. Delete the shadowed `recordDigit` from `242p` so only one implementation
  exists.
- Picking applies through the existing `recordApplyChoice` → `recordApplyOps`. That is
  still one `field.set` with the current fingerprint, no preview, and the stage closes
  on success. A refusal leaves the stage open with the prefix cleared.

### Surfaces

1. **Choices stage, focus in the filter input** (the screencast case). In `242r`'s
   `recordInInput` branch, when `inputKind === "choice-filter"` and the stage is
   `choices`:
   - A bare digit calls `preventDefault()`, so the digit never becomes filter text, and
     runs the digit step.
   - Backspace with a pending prefix calls `preventDefault()` and pops one digit. With
     no prefix, Backspace keeps its native text deletion.
   - Digits pick even when filter text is present, against the visible rows. Example:
     with filter `pl`, `0` `7` picks Plans if Plans is visible. Digits never become
     filter text.
2. **Choices stage, focus outside the input** (the existing
   `this.stage.kind === "choices"` branch). Use the same digit step and the same
   Backspace pop, so behavior does not depend on focus.
3. **Browse (no stage) on a choices row.**
   - Below 10 choices: unchanged. One digit picks and saves.
   - With 10 or more choices, behavior follows the `browse_digits` decision:

> [!decision] browse_digits = stage A matching first digit opens the choices stage on
> that field, through `recordOpenStage("choices", key)`, with `choiceDigitPrefix`
> already set. The filtered list and focus on the filter input come from the existing
> stage paint. The second digit picks and saves (surface 1). A non-matching first digit
> teaches and does not open the stage. One Esc still cancels the stage and returns to
> browse. The Esc ladder does not change.

> [!decision] browse_digits = status A matching first digit stores the prefix on the
> modal with no stage open, and the status line reads `<prefix>… type the next digit`.
> The second digit picks and saves. Esc, Backspace, selecting another field, or any
> other browse key clears the prefix first. The prefix is cleared whenever the selected
> field changes.

> [!decision] browse_digits = legacy The browse branch keeps today's immediate `1`–`9`
> pick on every choices field. Two-digit codes apply only inside the stage (surfaces
> 1–2). The docs must state that browse digits address only the first nine choices.

- Targets whose stage kind is not `choices` keep today's browse behavior: non-choices
  fields still teach `RECORD_CARD_DIGIT_TEACH`. For list-of-choices fields, keep the
  existing direct path below 10 choices; with 10 or more, do nothing and do not write.

### Feedback

- **Badges:** paint `code` for every visible choice. While a prefix is pending, wrap the
  typed digits of each matching badge in a span with an emphasis class, for example
  `.bob-record-card-choice-badge-typed`. Use text nodes only (`recordCardText`), never
  `innerHTML`.
- **Pending chip:** the filter row shows a small pending-code chip, such as `0…`, only
  while a prefix is pending. The live region announces, for example,
  `Choices 01 to 09 — type the next digit` through a new `recordCardLiveAnnouncement`
  kind.
- **Teach line:** a non-matching digit sets the refusal status with a testable string.
  Suggested: `No choice <typed> · choices are <first>–<last>`, such as
  `No choice 2 · choices are 01–17` or `No choice 18 · choices are 01–17`. It is
  announced too. Nothing is written.
- **Hints:**
  - The browse hint on a choices row shows the full code range: `1–5 pick` below 10,
    `01–17 pick` (padded) at 10 or more. Compute it from `recordCardChoiceList`, not
    from `field.choices.length`, so it agrees with the report's choice list.
  - The choices-stage hint strip adds a digit item. With no prefix it shows the code
    range of the visible rows. With a prefix it shows the narrowed range, such as
    `10–17 pick`, plus `⌫ clear <prefix>`.
  - Pass the needed fields (width, prefix, visible range) through `recordStatusModel`.
- **CSS:** add `font-variant-numeric: tabular-nums` to the badge, keep the `min-width`,
  and style the typed-digit emphasis and the pending chip with existing tokens
  (`--interactive-accent`, `--background-modifier-border`).

## Tests

Use `node:test` with the existing fake-DOM harness. The unit fixture's `status` field
already has exactly 10 choices (`to_try` … `c10`), so it covers the two-digit path. The
`cuisine` field has 2 choices and covers the one-digit path.

Update the tests whose assertions the new rule breaks:

- `scripts/test-ledger-tools-record-card.cjs`, "choice digits stay in schema order while
  filtering". The 10th choice now has code `10`; nothing is null. Codes are padded and
  survive text filtering. Use the code lookup.
- The same file, "choice click, Enter, and digit emit one field.set". On 10-choice
  `status`, browse `2` alone no longer writes. Under `browse_digits = stage`, `0` then
  `2` writes `loved`.
- "IME and filter typing never navigate or apply": composition digits must still do
  nothing.

Add coverage, either in that file or in a new
`scripts/test-ledger-tools-record-card-digits.cjs` (register a new file in
`package.json` `test:unit`):

- Pure `recordCardDigitStep` vectors at widths 1, 2, and 3 (synthesize a 100+ choice
  list): pick, prefix, and none. Cover `00`, out-of-range second digits, a first digit
  with no codes, text-filter-hidden codes, and no auto-pick on a unique prefix.
- In the filter input on 10-choice `status`:
  - `0` calls `preventDefault`. The filter value stays empty, visible rows are
    `01`–`09`, and no apply happens.
  - `7` then applies exactly one `field.set` for choice 7 (`c7`).
  - Backspace pops the prefix. A filter text edit clears it.
  - `1` then `1` teaches and writes nothing.
  - Arrows, Ctrl+N/P, and Enter act on prefix-filtered rows.
- `cuisine` (2 choices): a filter-input `2` picks `pizza` immediately, and so does
  browse `2`.
- Browse entry per the chosen `browse_digits` branch, including that a non-matching
  first digit neither opens the stage nor writes.
- DOM: badges show padded codes (per `badge_format`). Typed-digit emphasis and the
  pending chip appear only while a prefix is pending. The hint strip shows the code
  ranges.
- Refresh with keep-stage preserves a still-valid prefix.

The real-Obsidian proof (`scripts/obsidian-record-card-e2e.cjs`) uses a 2-choice
`status` for its `digit-one-guarded-write` case. That path is unchanged and must stay
green. Run it only where Obsidian and `xvfb-run` exist, which is not on apollo, and
report it as skipped otherwise.

## Docs

Update `docs/record-card.md` in bob-plugins:

- **Browse keys** table: replace the `` `1`–`9` on a choices row `` row with the code
  rule. One digit picks below 10 choices. From 10 choices, the behavior follows
  `browse_digits` (for example: the first digit opens the choices filtered to that
  prefix and the second digit picks). Keep the `Digits pick a choice on choice fields`
  teaching for non-choice fields.
- **Choices** section: replace "digits in the filter stay text. Positions past nine
  remain reachable by navigation and click." with the new rules:
  - every choice has a code, padded to the count's width from 10 choices
  - digits in the filter type codes, never filter text
  - the prefix narrows the list, intersected with filter text
  - Backspace pops the prefix
  - a non-matching digit teaches and writes nothing
  - no auto-pick before the full code
- Keep the line "Digits here pick a choice; on the album card editor they jump to field
  N." `docs/albums.md` is unaffected: its keycaps are the album editor's field jumps.

Check `docs/develop/architecture.md` and other bob-plugins docs for statements about
record-card digits, and fix any that become false. No SASE memory or decision record
changes: the decisions web excludes keymaps, and no memory note describes these digits.

## Release and deploy

1. Bump `plugins/bob-ledger-tools/manifest.json` by one minor version over whatever
   master holds (1.148.0 → 1.149.0 at plan time). Then run `npm run docs:gen` so
   `docs/index.md` and `docs/bob-ledger-tools.md` follow it.
2. Run `npm run build`, then `npm run build:check`, the targeted `node --test`
   record-card files, `npm run docs:check`, and `npm test`.
3. Run `bob plugins sync` (required by the bob-plugins `AGENTS.md`). If it refuses an
   overwrite, follow the `AGENTS.md` procedure. Never pass `-F` on your own.

## Acceptance

- On a 17-choice field: open the choices stage and type `0`. Only `01`–`09` show, the
  filter stays empty, and a `0…` chip shows. Type `8`: Beads is saved (one guarded
  `field.set`), the stage closes, and the status reads `✓ Saved · u undo`.
- Typing `1` then `7` saves choice 17. Typing `2` teaches and changes nothing.
- On fields with fewer than 10 choices, one digit picks and saves exactly as today.
- `npm test` and `npm run docs:check` pass, and `bob plugins sync` reports the new
  ledger-tools version as synced.
