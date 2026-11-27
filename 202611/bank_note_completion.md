---
tier: tale
title: Past-note completion in the Bank Category card
goal:
  When Bryan presses Space to note a bank row, the note field already offers the notes
  he has written on past bank rows, this payee's first, so a repeated note such as "Ask
  Kelly?" takes Space, Tab, Enter instead of retyping it.
size: medium
decisions:
  decision_record:
    ask:
      Add a decisions record saying past-note suggestions are recalled live from bank
      rows and never stored?
    memory:
      - decisions:bank-note-suggestions-recall-past-rows
    default: false
    answer: false
proposed_by: bbugyi200.athena.1gz
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 08:00:03
status: wip
---

# Plan: Past-note completion in the Bank Category card

## Why

Task `^cmpl-past-bank-notes` ("Add completion for past bank notes!") carries a
screenshot of the Bank Category card's note stage on a `$41.94 Amazon Reta* Ue4v4` row
with "Ask Kelly?" typed into the field ("Adds a note · stays on this row"). The vault
shows why: `bank/2026-11.md` has `— Ask Kelly?` typed by hand on four different
`Amazon Reta* …` rows. Bank notes repeat by merchant: "ask Kelly", "for Abby",
"reimbursable". Today every repeat is retyped.

## What exists (bob-plugins, `plugins/bob-ledger-tools/src/`)

- `246a-plugin-bank-note-write.js`: the pure note rules. `bankNoteNormalize` applies the
  safety rewrites, `bankNoteWritePlan` handles the row edit and its round-trip check,
  and `bankNoteStagePreview` produces the consequence copy.
- `248n-plugin-bank-note-stage.js`: the note stage on `BankCategoryCardModal`. Space (or
  the `Bank: note this row` command in "direct" mode) opens an in-card
  `<input data-bank-note-input>` in the note caption's exact slot. Enter writes the
  row's inline `— note` as its own undo step and stays on the row. Esc, Ctrl+[, and `q`
  outside the input back out. Typing updates the status and keys lines in place and
  never rebuilds or blurs the input. Tab, ↑/↓, and Ctrl+N/P are currently consumed and
  do nothing.
- `249a-plugin-bank-walk-index.js`: the ledger-owned bank-row index. It holds disk
  contents for every `bank/YYYY-MM.md` plus open-editor overlays
  (`bankWalkCollectOverlays`). `bankWalkSnapshot()` merges them, memoized by generation
  plus overlay sequence. Its `entries` are walk-scoped (waiting rows only while anything
  waits), so they cannot serve as a note source. The merged contents can.
- `bankTokenizeLine(raw)` already yields `payee`, `note`, `noteEditable`, and `noteSpan`
  for any row line.
- `247-plugin-bank-category-card-ui.js` already paints the subscription picker's option
  list, a precedent for in-card rows with Ctrl+N/P. It also defines
  `bankCategoryUiEventKey`, which names `Ctrl+N` and `Ctrl+P`.

## The experience

The flow Bryan should get: on the first Amazon row he types "Ask Kelly?" once (Space,
type, Enter, then a category key). On the next Amazon row he presses Space and sees:

```text
Category
$27.01  Amazon Reta* Vr389                                     Nov 25
—  Ask Kelly?                      ← faint ghost text inside the empty field
   ⇥  Ask Kelly?                                          4× · Nov 25
      Kelly sent me an email about monsters and groceries.     Nov 18
   Adds a note · stays on this row
4 of 7 · November
… dimmed tiles …
⇥ complete · ⌃N/⌃P choose · ↵ save · esc back
```

He presses Tab and the field reads "Ask Kelly?". He presses Enter and the note is
written. Typing anything else narrows or dismisses the suggestions. Nothing is ever
written that he did not see in the field.

### Interaction rules

Stage state gains `query` (what Bryan typed) and `selectedKey` (null, or a suggestion's
key; never an index). The field shows the selected suggestion's text when one is
selected and `query` otherwise. The list is ranked from `query` only, so navigation
never re-ranks or collapses it. This follows the precedent of
`decisions:relation-card-tab-accepts-ctrl-d-hides`: selection by key, and Tab accepts
the selected row or completes the top match.

| Key while the note stage is open                | Effect                                                                                                                                                                                                                  |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Typing, Backspace, paste                        | Edits the field. The field becomes `query`, `selectedKey` clears, and the list re-ranks.                                                                                                                                |
| Tab                                             | Takes the ⇥ row into the field and makes it the new `query`: the selected row, or the top row when none is selected. The caret goes to the end. Does nothing when the list is empty. Never saves and never moves focus. |
| → with the caret at the end and a ghost showing | Same as Tab. Otherwise → behaves natively.                                                                                                                                                                              |
| Ctrl+N or ↓                                     | Selects the next row and puts its text in the field. Past the last row it wraps back to the typed `query`.                                                                                                              |
| Ctrl+P or ↑                                     | Selects the previous row. From the typed `query` it goes to the last row.                                                                                                                                               |
| Enter, Ctrl+Enter                               | Unchanged: saves exactly what the field shows.                                                                                                                                                                          |
| Esc, Ctrl+[, `q` outside the input              | Unchanged: backs out one layer. `q` still types inside the field (`decisions:q-and-ctrl-bracket-mirror-esc`).                                                                                                           |
| Shift+Tab                                       | Consumed, does nothing (focus never leaves the stage).                                                                                                                                                                  |
| Click a row                                     | Same as Tab on that row. Mousedown is prevented so the input keeps focus. Never saves.                                                                                                                                  |

These keys work with editor focus too, matching the stage's existing routing. IME
composition and `stage.saving` pass or ignore keys exactly as today. A key the
suggestion layer consumes is never also passed to the input natively.
`shouldPassNoteStageKey` must return false for it, including → when it takes the ghost.

Enter deliberately never takes the ghost. The ghost is a preview of Tab's target, and
the field is the only thing Enter writes. Navigating puts a suggestion into the field
(omnibox style), so "Enter saves what the field shows" stays true without exception, and
the existing consequence line ("Adds a note · stays on this row") stays accurate.

### Where suggestions come from (the note library)

- **Source.** Every row in every `bank/YYYY-MM.md` held by the bank walk index: disk
  content, with the open editor's buffer winning for its note. A note saved seconds ago
  on the previous row is therefore suggested on the next row without waiting for a disk
  write or a bob call. Nothing is stored, learned, or cached outside the index's memo.
  The library is a pure view of notes already in the vault, so editing a note out of
  past rows removes it from suggestions.
- **Kept notes.** A row contributes its note only when `token.note` is non-empty,
  `token.noteEditable === true`, and `bankNoteNormalize(note)` is `ok` with
  `text === note` and no changes. Every suggestion can then be written byte-for-byte as
  shown and passes `bankNoteWritePlan`'s round-trip. Structured or legacy multi-run
  notes never appear.
- **Identity.** Suggestions are deduplicated by a folded key: NFKD with combining marks
  removed, ’‘ → ', “” → ", lowercase, whitespace collapsed, trimmed. A suggestion's
  display and insert text is the spelling of its most recent use (date desc, then path
  and line desc).
- **This row never suggests itself.** Uses at the card row's own `(path, line)` are
  dropped at rank time. An entry left with no uses disappears.

### Merchant family (so all `Amazon Reta* …` rows are one merchant)

The importer keeps Amazon's spaced order code (`AMAZON RETA* BV5NH …` becomes payee
`Amazon Reta* Bv5nh`), so every Amazon row has a unique payee.
`bankNoteMerchantKey(payee)` groups them:

1. Fold as above.
2. Drop a `*`-introduced order code, whether glued (`reta*ue4v4`) or the one token after
   `* ` (`reta* ue4v4`), when it is 1–12 alphanumerics with at least one digit.
   Word-only sub-merchants stay (`delivery *sushi` ≠ `delivery *pizza`).
3. Drop `#\d+` store numbers and mask digit runs of 4 or more as `#`.
4. Strip stray `*`, collapse whitespace, and trim.

Required examples: `Amazon Reta* Ue4v4`, `Amazon Reta* Pd4tc`, and `AMAZON RETA*BV5NH` →
`amazon reta`. `Amazon Mktpl` → `amazon mktpl`. `Delivery *Sushi` and `Delivery *Pizza`
stay distinct. `Track 5 Coffee` keeps its `5`.

`bankNoteMerchantLabel(payee)` gives the same stripping with the original case
(`Amazon Reta`). It is used only in row meta.

### Matching and ranking (pure, deterministic)

The reference date is the card row's `date`, or local today when it is absent. A use's
weight is `0.5 ^ (max(0, refDays − useDays) / 30)`, which is frecency with a 30-day
half-life. A suggestion is **affine** when it has at least one use with this row's
merchant key. Its affinity score sums only those uses. Its global score sums all uses.

- **Empty query** (the field was just opened or cleared): only affine suggestions,
  ordered by affinity score desc, then last affine use desc, then folded text asc. Other
  merchants' notes never appear on an empty field.
- **Non-empty query**: fold the query, trim its leading whitespace, and keep one
  trailing space when present, so `Ask ` matches `Ask Kelly?` but not `Asking`. A
  suggestion matches in class 0 when its key starts with the query, or in class 1 when
  the query occurs at a word boundary (the preceding character is not `\p{L}`/`\p{N}`).
  Keys equal to the query are excluded. Non-affine suggestions need a query of at least
  2 characters. Order: class asc, affine first, then score desc (affinity score for
  affine rows, global score otherwise), then last use desc, then folded text asc.
- At most **4** rows. More is reached by typing more.
- Constants live beside the model: `BANK_NOTE_SUGGEST_LIMIT = 4`,
  `BANK_NOTE_SUGGEST_HALF_LIFE_DAYS = 30`, and `BANK_NOTE_SUGGEST_GLOBAL_MIN = 2`.

**Ghost.** The ghost shows only when nothing is selected, the top row is class 0,
`top.text.slice(0, field.length).toLowerCase() === field.toLowerCase()`, the caret is at
the end with no selection range, and the text does not overflow the input
(`scrollWidth <= clientWidth`; treat missing metrics as fitting). Its text is
`top.text.slice(field.length)`. With an empty field, the ghost replaces the placeholder:
clear `placeholder` while a ghost shows and restore "What was this for?" after.

**Row meta** is right-aligned and quiet:

- Affine rows show `N× · Mon D`, using same-merchant uses.
- Other rows show `Merchant Label · N× · Mon D`, using all uses.
- `N×` appears only when N ≥ 2. The date reads `Mon D` in the reference year and
  `Mon YYYY` otherwise.

**Match highlight.** Best-effort and never wrong. Find the query's span in the display
text by comparing folded slices at word starts (class 1) or at 0 (class 0). If no exact
span is found, render without emphasis.

### Visual design (styles.css, beside the note-stage block)

The suggestions should read as alternative note lines, not as a widget:

- **Alignment.** Each row is a grid `[marker][text 1fr][meta auto]`. The marker column
  is exactly as wide as the `—` dash column (dash plus its 0.3em margin), so suggestion
  text starts on the same x as the field's first character.
- **Rows.** The note line's size (0.92em). Text is italic like every `— note` caption,
  in `--text-muted`, with the matched span in `--text-normal`. Single line with
  ellipsis, and a `title` attribute carrying the full note. Padding `0.1rem 0.3rem`,
  radius `0.3rem`, hover `--background-modifier-hover`.
- **Marker.** A small keycap in the card's keycap language (no uppercase, ~0.68em). `⇥`
  sits on Tab's target row. On a selected row it becomes `↵` (Enter saves this, since
  the field shows it), mirroring the `↵` on the highlighted category tile.
- **Selected row.** `color-mix(in srgb, var(--bob-bank-accent) 12%, transparent)`
  background, `--text-normal` text, and an accent marker border.
- **Meta.** Upright, `--text-faint`, ~0.78em, `tabular-nums`, nowrap. The merchant label
  is ellipsized at ~10em.
- **Ghost.** A `position: relative` wrapper around the input holds an `aria-hidden`
  overlay at the input's exact padding, font, italic, and letter-spacing. The typed part
  is `color: transparent` and the remainder is `--text-faint`, with `white-space: pre`
  and `pointer-events: none`.
- **Order.** Input line, then the list, then the existing status (consequence, chips,
  refusal). The list appears and disappears instantly: no animation, nothing to reduce.
  Under `forced-colors: active` the selected row outlines in `Highlight` and the ghost
  uses `GrayText`.
- **Keys line.** While the list is visible:
  `⇥ complete · ⌃N/⌃P choose · ↵ save · esc back`. Direct mode keeps its `esc close`
  ending. With no list, the line is exactly today's copy.

### Accessibility

- **Input.** `role="combobox"`, `aria-autocomplete="list"`, `aria-expanded`,
  `aria-controls` pointing to the list, and `aria-activedescendant` set to the selected
  option's id (removed when none is selected).
- **List.** `role="listbox"`, `aria-label="Past notes"`. Rows are `role="option"` with
  stable ids and `aria-selected`.
- **Announcements.** Tab and click acceptance announce `Completed: <note>` through the
  card's existing live region.

## Architecture and files (all in bob-plugins; open it with `/sase_repo`)

Suggestion ranking stays in the plugin, not in bob. bob keeps owning categories,
hotkeys, and totals. The note library is Bryan's own text read from the same index the
`]b` walk uses. Ranking runs per keystroke and must see an editor buffer that is not
saved yet, so a `bob` subprocess or a month-report refresh would be both slower and
staler. No JSON contract, `api.bankWalk` v1 field, or bob-cli code changes.

1. **New `src/246b-plugin-bank-note-suggest-model.js`** (pure, no Obsidian; listed right
   after `246a` in `src/fragments.json`). It holds `bankNoteFold`,
   `bankNoteMerchantKey`, `bankNoteMerchantLabel`,
   `bankNoteLibraryFromContents(fileContents, cache)` (a per-file memo keyed by exact
   content string, so unchanged months are never re-tokenized; returns a frozen
   `{ available, entries }`), `bankNoteSuggest({ library, query, row, limit })` (returns
   frozen view rows `{ key, text, matchClass, affine, meta, highlight }`),
   `bankNoteSuggestGhost(...)`, and `bankNoteSuggestStep(rows, selectedKey, direction)`.
2. **`src/249a-plugin-bank-walk-index.js`**: extract the disk-plus-overlay merge in
   `bankWalkSnapshot()` into `bankWalkMergedContents()` returning `{ sig, merged }`. Use
   it in the snapshot with no behavior change, and add `bankNoteLibrary()`. When the
   walk is unloaded or indexing, `bankNoteLibrary()` returns an empty library with
   `available: false`. Otherwise it is memoized on the same signature and reuses a
   dedicated per-file note cache. Keep the fragment under 1000 lines. If it would go
   over, put `bankNoteLibrary()` in a new `249b` fragment that patches
   `BobLedgerToolsBankWalkMixin.prototype`, as `248n` does.
3. **New `src/248o-plugin-bank-note-suggest-stage.js`** (after `248n`): the UI layer
   installed on `BankCategoryCardModal.prototype`. It builds and refreshes the ghost and
   list in place, routes the keys in the table above, handles row clicks, sets the ARIA
   attributes, and gets the library through `this.pluginRef.bankNoteLibrary?.()`. It
   re-ranks only when `query` changes. When `selectedKey` leaves the list after a
   re-rank, it resets to null and the field shows `query`.
4. **`src/248n-plugin-bank-note-stage.js`**: minimal hooks only. Initialize
   `query`/`selectedKey` in `openNoteStage`. Set `query = draft` and clear `selectedKey`
   in `onNoteInput` and `setNoteDraft`. Offer keys to `routeNoteSuggestKey` first in
   `routeNoteStageKey`, and add the `shouldPassNoteStageKey` carve-outs. Add the field
   wrapper and list slot in `buildNoteStageHost`, a suggestion refresh call in
   `refreshNoteStage`, and the keys-line prefix in `noteStageKeysText`. The input node
   must keep its identity across typing, navigation, and acceptance.
5. **`src/356-exports-bank-category.js`**: export the new pure helpers on
   `module.exports.helpers` for tests.
6. **`styles.css`**: the rules above, next to the existing `.bob-bank-category-note-*`
   block.
7. Run `npm run build`. Never hand-edit `main.js`.

## Tests

- **New `scripts/test-ledger-tools-bank-note-suggest.cjs`**, registered in
  `package.json` `test:unit`. It covers:
  - Merchant key and label examples.
  - Folding, including curly quotes and accents.
  - Library exclusions: structured notes, notes the normalizer would change, empty
    notes, and the card row itself.
  - Dedupe with the most-recent spelling.
  - The per-file memo: an unchanged file returns the identical frozen records.
  - Empty-query affinity-only ranking.
  - Class 0 before class 1, affine before global, and frecency over raw recency ("Ask
    Kelly?" ×4 beats a newer one-off).
  - The 2-character global minimum, exact-match exclusion, the trailing-space rule, and
    the limit of 4.
  - Ghost rules: case-insensitive prefix, no ghost for a class-1 top, no ghost when
    selected.
  - Key-based stepping with wrap through the typed query, and a selection that
    disappears resetting to null.
  - Meta and date labels.
- **Extend `scripts/test-ledger-tools-bank-note-stage.cjs`** (FakeNode harness) with a
  stub `pluginRef.bankNoteLibrary()`:
  - Space on an Amazon row shows the list, the ghost, and a cleared placeholder. Tab
    fills "Ask Kelly?". Enter calls `onNote` with exactly "Ask Kelly?".
  - Ctrl+N and ↓ put row 2 in the field, Ctrl+P wraps back to the typed text, and typing
    after navigating clears the selection.
  - → at the end accepts the ghost. → mid-field moves natively.
  - A row click fills without saving.
  - An empty library shows no list and today's keys copy.
  - Direct mode works the same.
  - The input node identity holds across all of the above.
  - The ARIA attributes are correct.
  - Esc, Ctrl+[, and `q` behavior is unchanged.
- **Extend `scripts/test-ledger-tools-bank-walk-index.cjs`**: a note in an open editor
  overlay appears in `bankNoteLibrary()` before any disk write. An unloaded or indexing
  walk yields an unavailable, empty library. `bankWalkSnapshot()` output is unchanged by
  the refactor.
- **Real Obsidian E2E (`scripts/obsidian-bank-ui-e2e.cjs` and its lib)**:
  - Give the fixture an acknowledged past row whose `— note` shares a merchant family
    with a waiting row in the walk.
  - Add a `walk-note-suggest` step: Space on that waiting row, then assert the option
    row and ghost, then Tab and Enter.
  - Assert the row's bytes end with that note, and capture `walk-note-suggest.png`.
  - Keep the offline wrapper `scripts/test-obsidian-bank-ui-e2e.cjs` token list in sync.
  - Run it with `xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs` and look at the
    screenshot. The ghost must sit exactly on the field's baseline, and suggestion text
    must align with the field's first character.

## Docs

- **bob-plugins `docs/bob-ledger-tools.md`**, in the Category card paragraph. After
  "Space notes the row and stays", add one compact clause: the note field completes from
  notes already on bank rows (this payee's first, even on an empty field; others from
  two typed letters). Tab completes the ⇥ row, → at the end takes the ghost, Ctrl+N/P or
  ↓/↑ put another suggestion in the field, and Enter saves exactly the field.
- Run `npm run docs:check` and `npm run docs:inventory:check`. bob-cli mounts this page
  at build time. No bob-cli doc changes.

## Verify and deploy

Run `npm run build`, then `npm test` (or at least `npm run build:check` plus the touched
`node --test` files and the docs checks), then the E2E above. Then run
`bob plugins sync`, following bob-plugins `AGENTS.md` if sync refuses an overwrite.

## Out of scope

- **Completion while typing `— ` directly in the month-note editor.** An Obsidian
  suggest popup claims Enter, which month-note Enter uses to continue rows. The card is
  where notes are written during the walk.
- **A hide or forget gesture, or any stored suggestion state.** To drop a suggestion,
  edit the note on its rows.
- **bob's import cleaning keeping Amazon's spaced `RETA* CODE` order code.** Filed
  separately as a bug. This plan's merchant key handles both old and fixed payees.

> [!decision] decision_record Add the strand
> `decisions:bank-note-suggestions-recall-past-rows` to the bob-cli decisions web using
> `/sase_memory_write`, then run `sase memory init`.
>
> - **Applies to:** the Bank Category card note stage.
> - **Claim:** suggestions are recalled live from notes already on bank rows through the
>   ledger index (disk plus open editors), ranked in the plugin, and never stored,
>   hidden, or learned elsewhere. Tab completes, navigation fills the field, and Enter
>   saves exactly the field.
> - **Rejected alternatives:**
>   - bob JSON suggestions: a subprocess per keystroke, and blind to the editor's
>     unsaved note.
>   - A stored catalog or hide journal: hidden state that drifts from rows.
>   - Enter taking the ghost: it writes text Bryan never saw in the field.
> - **Cost:**
>   - Typos persist until edited out of rows.
>   - No other surface reuses the ranking.
> - **Reopens when:**
>   - A second surface (Mac or terminal) needs note suggestions, so ranking moves into
>     bob.
>   - Stale or typo suggestions call for a hide gesture.
> - **Evidence:** this plan, the bob-plugins commit, and the four hand-typed
>   `— Ask Kelly?` rows in `bank/2026-11.md`.
