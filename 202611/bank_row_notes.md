---
tier: epic
title: 'Bank row notes: Space opens a note stage on the Category card'
goal: 'During the ]b bank walk, Space on the Category card writes a short note into
  the row''s inline `— note` tail and stays on the same row (no tick, no advance),
  and the note renders everywhere as a quiet muted-italic dash-led caption.

  '
phases:
- id: note-grammar-parity
  title: Shared note cleaner, disjoint note spans, and prerequisite fixes
  depends_on: []
  size: medium
  description: 'note-grammar-parity: land the byte-identical shared note-cleaner vectors
    and the five-step cleaner in bob''s Rust parser and the plugin tokenizer, add
    disjoint noteSpan/noteMarkerSpan/noteEditable, move 🔁 link insertion before flag/note,
    protect every `$` on a row, and document the tail order in docs/bank.md.'
- id: note-caption
  title: Quiet prose caption on every row surface
  depends_on:
  - note-grammar-parity
  size: small
  description: 'note-caption: render notes as a muted italic dash-led caption in Live
    Preview, Reading view, and the queue card, with contrast-safe CSS and tests.'
- id: note-writer
  title: Pure note normalizer, write planner, and stage preview model
  depends_on:
  - note-caption
  size: small
  description: 'note-writer: add the pure bankNoteNormalize, bankNoteWritePlan (with
    round-trip self-check), and bankNoteStagePreview helpers plus their test matrix;
    ordered after note-caption only to serialize bob-plugins builds and vault syncs.'
- id: note-stage
  title: Space keymap, in-card note stage, stay-on-row commit, command, and docs
  depends_on:
  - note-writer
  size: medium
  description: 'note-stage: reassign Space to the note stage and `-` to no-category,
    build the in-card stage and bankNoteCommit that never advances, add `Bank: note
    this row`, update e2e, plugin and bob-cli docs, and apply memory only per the
    decisions.'
decisions:
  memory_keymap_decision:
    ask: Add a decisions record for the new card keymap (Space writes a row note;
      `-` files without a category)?
    memory:
    - decisions:space-writes-a-bank-row-note
    default: false
    answer: false
  memory_bank_row_glossary:
    ask: Update the glossary Bank Row strand to mention the inline note and the Space
      gesture?
    memory:
    - glossary:bank-row
    default: false
    answer: true
proposed_by: bbugyi200.apollo.research.5q.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 12:07:04
status: done
bead_id: bob-cli-oo
---

- **PROMPT:** [prompts/202611/bank_row_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_row_notes.md)
- **BEAD:** [bob-cli-oo](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oo/README.md)

# Bank row notes: Space opens a note stage on the Category card

## Goal

During the morning `]b` walk, Bryan wants to leave a short note on a bank row ("why was
this bought?") without leaving the walk. Space on the bank Category card opens an
in-card note stage. Enter writes the note into the row's existing inline `— note` tail
as its own undo step and returns to the same card on the same row: no tick, no advance,
no session filing. A muted italic caption then shows the note on the row everywhere it
appears.

This plan follows the consolidated research report
`research:202611/bank_row_note_stage/bank_row_note_stage.md`. Bryan agreed with every
requirement it recommends, including the Space ⇄ `-` reassignment. Memory edits are the
exception: the two memory decisions decide them.

## Design (owned by this plan; phases implement it as written)

### Interaction

```text
]b → Ctrl+Enter → Category card → Space → type → Enter → (same card, note visible) → d → next row
```

**Card keys (close mode, set mode, and the restricted card):**

| Key                                                       | Close mode                              | Set mode            | Restricted card                                                                               |
| --------------------------------------------------------- | --------------------------------------- | ------------------- | --------------------------------------------------------------------------------------------- |
| `Space`                                                   | Open the note stage                     | Open the note stage | Open the note stage (works offline)                                                           |
| `-`                                                       | File with no category (ticks, advances) | Remove category     | Acknowledge (no category)                                                                     |
| `Delete`                                                  | —                                       | Alias of `-`        | —                                                                                             |
| `Enter` / `Ctrl+Enter`                                    | Accept highlight (unchanged)            | Accept (unchanged)  | Acknowledge, but only after the report request has settled. While loading, Enter does nothing |
| letters, `+`/`=`, `/`, arrows, Ctrl+N/P, Esc, `q`, Ctrl+[ | Unchanged                               | Unchanged           | Esc/`q`/Ctrl+[ unchanged                                                                      |

Space never types, never ticks, and never flips `didInteract`. Repeated (held) Space is
ignored. When a native `button` inside the card has focus, Space keeps its native
activation. Space is ignored while `busy`, `waitingRefresh`, or on the summary screen.

**Footers** (built by `bankCategoryCardKeysText`, which gains a `hasNote` input):

- close:
  `↵ accept · letter choose · ␣ note · - no category · + new category · esc cancel`
- set:
  `↵ accept · letter choose · ␣ note · - remove category · + new category · esc cancel`
- restricted: `↵ acknowledge · ␣ note · esc cancel`. While loading:
  `loading… · ␣ note · esc cancel`.
- With an existing note, `␣ note` reads `␣ edit note`.

**Inside the note stage:**

| Key                                                              | Action                                                                                                                                                 |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| printable keys (letters, digits, Space, `q`, `/`, `+`, `=`, `-`) | Type into the input. Category hotkeys never fire while staged. If focus is outside the input (editor focus), the key focuses the input and is appended |
| `Enter`, `Ctrl+Enter`                                            | Save, then return to the tiles. Ctrl+Enter must never file from inside the stage                                                                       |
| `Esc`, `Ctrl+[`                                                  | Discard the draft and back out one layer to the tiles. A second Esc closes the card (`decisions:q-and-ctrl-bracket-mirror-esc`)                        |
| `q`                                                              | Types in the input. Outside a text control it backs out like Esc (same decision)                                                                       |
| IME composition                                                  | Never saves mid-composition (`isComposing` / `keyCode === 229`)                                                                                        |

After a save, re-arm the existing opening-chord guard (`bankCategoryChordInit(true)`).
Until the save Enter's keyup arrives, a held or rapid Enter cannot accept a category.

**Lifecycle on one key:** add when the row has no note; edit when it has one (input
prefilled from the raw note bytes, cursor at the end); remove by saving an empty input
over an existing note (the consequence line says "Removes the note" first). An empty
save with no existing note is a no-op that simply returns to the tiles.

**Structured legacy notes.** A hand-typed note may have prose split by links or fields
(for example `— with [[kelly]] at lunch`). Space then does not open the stage. The
card's hint line shows `This note has links or fields — edit it on the row` and
announces it.

### The write (stay is structural)

- A dedicated `bankNoteCommit` runs beside `bankCategoryCommit`, never inside it. It
  never calls the session recorder, the next-waiting finder, `bankCategoryRecordJump`,
  or the completion summary, and it never flips `didInteract`. An in-flight suggestion
  refresh still lands, and the highlight stays where it was.
- Plan purely, then apply one whole-line `replaceRange`, so the note is one undo step.
  Note and filing stay separate undo steps even when a filing key follows within CM6's
  500 ms grouping window. If the history merges them, isolate the note transaction with
  CM6 `isolateHistory`, reached through `editor.cm`.
- After applying, save the note file and report persistence errors (do not copy the
  swallow-all `bankCategorySaveNote` style for the note's outcome). Keep the cursor on
  the row at its prior column, clamped. Rebuild `modal.context` from the post-write
  buffer with `bankCategoryContextAt` (capture line text, token, card row, scan). Then
  call `bankWalkHandOffOpenedRow` with the new line text, so ID-less, text-keyed walk
  entries still resolve. Without this, the next category key refuses with "the row
  changed — nothing written".
- Session: in close mode, `notes += 1` only (never `filed`). The end-of-walk detail line
  appends `· N notes` when N > 0.
- On refusal (row moved, concurrent edit, structured note, unsafe round trip, `#task`),
  keep the stage open with the draft intact, show the reason inline, and announce it.
- Live region: `Note saved` / `Note removed` / the refusal text.

### Storage: the existing inline `— note` tail

Canonical tail order (documented in `docs/bank.md`):

```text
- [ ] $AMOUNT Payee [cat:: k] [acct:: a] [[links]] ⚑ — note ^bank-id
```

- **Insert** (no note yet): ` — {text}` goes immediately before the trailing
  `^bank-…`/custom block ID (after any flag or links), or at the trimmed line end on
  ID-less rows.
- **Replace**: only the note's own prose bytes (`noteSpan`). The existing marker
  spelling (`—` or `--`) is kept.
- **Remove**: removes the marker (when adjacent), the prose, and the whitespace before
  them. A flag always survives.
- **Round-trip self-check** (a runtime invariant, not only a test): tokenize the planned
  line. Require `note` to equal the normalized text (or null after a remove). Require
  status, cents, payee, category, account, flag, links, and IDs to be unchanged.
  Otherwise refuse with `Bank: couldn't place the note safely — nothing written`.

**Normalizer** (`bankNoteNormalize`). It is visible, never silent: every change is
listed on the stage's warning line before saving.

| Input                                               | Becomes                | Why (shown to Bryan)                                |
| --------------------------------------------------- | ---------------------- | --------------------------------------------------- |
| newlines, tabs, whitespace runs                     | single spaces; trimmed | one line per row                                    |
| leading `—` or `--` marker                          | dropped                | the caption adds the dash                           |
| `!!`                                                | `‼`                   | `!!` would flag the row                             |
| `⚑`                                                 | `⚐`                    | `⚑` would flag the row                              |
| `[[target\|alias]]` / `[[target]]`                  | `alias` / `target`     | links in notes would be learned into future imports |
| `[k:: v]`                                           | `(k: v)`               | fields belong to the row                            |
| `#tag` (Obsidian tag shape, at least one non-digit) | `tag`                  | tags in notes are not supported yet                 |
| `#task` token                                       | **refused**            | `bank_row_task` is a hard error                     |
| `%%`                                                | `%`                    | `%%` opens a comment that hides text                |
| `<!--`                                              | `< !--`                | an HTML comment would hide text                     |
| trailing `^word` (block-ID shape)                   | `^ word`               | it would become the row's block ID                  |

`$` is left alone; the rendering fixes below protect it.

### Beautiful means quiet prose

The note is the row's one piece of authored prose, so it never wears a pill.

- **Live Preview and Reading view:** one caption replaces the span from the marker (`—`
  or `--`) through the note text. It reads as muted italic text after a faint upright em
  dash: `— LEGO set for Max's birthday`. `--` hides behind the same widget. A
  flag-opened note (`!! with Kelly`) gets the same synthesized dash. The caption yields
  to raw source when the cursor enters the line, like every bank mark. Clicking it
  places the cursor (`ignoreEvent` false). It carries `title` (full text) and
  `aria-label="Note: …"`.
- **Contrast guard:** acknowledged rows already render at `opacity: 0.72`. Their caption
  must still reach WCAG 4.5:1 in the default light and dark themes, so a compounded
  faint ink must not drop it below that. Use a stronger ink under
  `.bob-bank-mark-acked`.
- **Card header:** the same caption sits under the payee line,
  `.bob-bank-category-note`, clamped to two lines with an ellipsis and a full-text
  `title`. It is a button: clicking it opens the stage.
- **Note stage:** the input takes the caption's exact place, with the same x-position,
  the same faint em dash hanging before it, and the same italic. Opening and saving
  never move the eye. The input has no box: a 1px hairline underline
  (`--background-modifier-border`) turns `--interactive-accent` on focus. The
  placeholder reads `What was this for?`, and the input has
  `aria-label="Note for <payee>"`. Below it sits one small consequence line:
  `Adds a note · stays on this row` / `Updates the note · stays on this row` /
  `Removes the note` (warning ink) / `No change`, plus up to two normalizer change chips
  (`[[max]] → max`, `+1 more`). The tiles stay painted but dimmed in place: `.is-staged`
  at opacity ≈ 0.38, `inert`, no pointer events, with a 120 ms fade that
  `prefers-reduced-motion` turns off. The keys line reads `↵ save · esc back`, or
  `↵ remove note · esc back` / `↵ back · esc back` to match the consequence.
- **Saved glow:** after a save, the header caption gets `.is-fresh`, a soft accent
  background wash that fades over ~600 ms (no animation under reduced motion). It
  confirms the stay without a toast.
- **Queue card** (`view: queue` on `bank.md`): a muted italic `— note` after the payee,
  single line with an ellipsis and a full-text `title`, from `row.note` in bob JSON.

### Everywhere it works

- Close mode (the headline `]b` flow), set mode (`Bank: set category`), and the
  restricted card. Notes are local editor edits that need no bob spawn.
- **`Bank: note this row`** (`bank-note-row`, command palette, no default hotkey). It
  opens the card in set mode, directly in the note stage. The title reads `Note`, the
  header shows, and no tiles or progress are painted. Enter saves and closes with a
  `Bank: note saved` / `note removed` / `no change` Notice. Esc closes the card, since
  there is no tile layer beneath. It never ticks and skips the report fetch. Typing
  `— text` by hand on the row stays fully supported. (Command Mode lists only bob
  catalog commands, so this is a palette command; Vim users can bind it with
  `obcommand`.)

### Scope fences (v1 excludes)

Links or tags inside notes (normalized away, visibly), multiline notes (folded), note
autocomplete, a save-and-accept chord, a `bob bank note` CLI writer, child-bullet
overflow, and Bank badge tooltip notes.

## Shared note-cleaner contract (both parsers)

Both bob's Rust parser (`src/native/bank/grammar.rs`) and the plugin tokenizer
(`241o-plugin-bank-marks-model.js`) compute a row's `note` like this:

1. Take the tail text after the note opener: the first `⚑`, `!!`, `—`, or `--` (not
   followed by `-`) that is not inside a field or link. The opener rules stay as they
   are today.
2. Remove every whole inline field `[k:: v]`, every whole wikilink `[[…]]`, every `⚑`,
   and every `!!`. Do **not** remove single `!`, internal `—`, or internal `--`.
3. Collapse whitespace runs to one space; trim.
4. Drop one leading marker, `—` or `--` (not followed by `-`), then trim again.
5. Empty → no note.

Both repos carry this vector table **byte-identically**, at bob-cli
`tests/fixtures/bank_notes/note_cleaner.json` and bob-plugins
`plugins/bob-ledger-tools/scripts/fixtures/bank/note-cleaner.json`:

```json
{
  "about": "Shared bank row note-cleaner vectors. Keep byte-identical: bob-cli tests/fixtures/bank_notes/note_cleaner.json and bob-plugins plugins/bob-ledger-tools/scripts/fixtures/bank/note-cleaner.json.",
  "cases": [
    {
      "line": "- [x] $13.71 Smashburger !! -- with Kelly",
      "payee": "Smashburger",
      "flagged": true,
      "note": "with Kelly"
    },
    {
      "line": "- [ ] $23.21 Amazon Reta* 125dt [cat:: kids] — LEGO set for Max's birthday ^bank-8f3a2c1d",
      "payee": "Amazon Reta* 125dt",
      "flagged": false,
      "note": "LEGO set for Max's birthday"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — wow! great",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "wow! great"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — LEGO — for Max",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "LEGO — for Max"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — split -- three ways",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "split -- three ways"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — -5 off coupon",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "-5 off coupon"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe -- ascii marker note",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "ascii marker note"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe [cat:: coffee] ⚑ — ask Kelly ^bank-ab12",
      "payee": "Corner Cafe",
      "flagged": true,
      "note": "ask Kelly"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe [[eat/cafe|☕]] ⚑ — ask Kelly ^bank-ab12",
      "payee": "Corner Cafe",
      "flagged": true,
      "note": "ask Kelly"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe ⚑ [[eat/cafe|☕]] — ask Kelly",
      "payee": "Corner Cafe",
      "flagged": true,
      "note": "ask Kelly"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — with [[people/kelly|Kelly]] at lunch",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "with at lunch"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — note [x:: y] tail",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "note tail"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — spaced    out\twords",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "spaced out words"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe ⚑ wow!! nice",
      "payee": "Corner Cafe",
      "flagged": true,
      "note": "wow nice"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe — — double marker",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": "double marker"
    },
    {
      "line": "- [ ] $5.00 Corner Cafe ⚑",
      "payee": "Corner Cafe",
      "flagged": true,
      "note": null
    },
    {
      "line": "- [ ] $5.00 Corner Cafe —",
      "payee": "Corner Cafe",
      "flagged": false,
      "note": null
    },
    {
      "line": "- [ ] $12.00 Lunch [cat:: dining] — split $25 w/ Alex ‼ (cat: x) ⚐ ^ word ^bank-1a2b",
      "payee": "Lunch",
      "flagged": false,
      "note": "split $25 w/ Alex ‼ (cat: x) ⚐ ^ word"
    },
    {
      "line": "- [x] $3.00 Tea — see ^ ref",
      "payee": "Tea",
      "flagged": false,
      "note": "see ^ ref"
    }
  ]
}
```

If either parser disagrees with a vector because of a pre-existing payee or opener rule
(not the cleaner), fix the parser, not the vector. If a vector itself is wrong against
the five rules above, correct it in **both** copies in the same phase and say so in the
phase's bead notes.

## Repositories and gates

- **bob-cli** (this repo): `just check` (or at least `cargo test` for the bank module
  plus `just lint`), and `just docs-check` after docs edits. Releases ride the daily
  train (`decisions:bob-cli-releases-ride-a-daily-train`). Never edit the Cargo version
  or CHANGELOG. This is not a CLI/JSON contract break: JSON `note` keeps its type and
  only becomes more faithful.
- **bob-plugins** (linked repo; open it with `/sase_repo` before reading or editing;
  read its `AGENTS.md`): edit `plugins/bob-ledger-tools/src/` fragments, keep each
  hand-edited fragment ≤ 1000 lines, and register new fragments in `src/fragments.json`.
  Then run `npm run build`, `npm test`, and `npm run docs:check`, and finish with
  `bob plugins sync`, following that repo's sync-refusal rules (never `-F` on your own).
  Add new `scripts/test-*.cjs` files to `test:unit` in `package.json`.
- **Serialization:** every bob-plugins phase regenerates the committed `main.js` and
  deploys to the vault, so the bob-plugins phases run strictly one after another.

## Phases

### note-grammar-parity: Shared note cleaner, disjoint note spans, and prerequisite fixes

**Size:** medium · **Depends on:** none · **Repos:** bob-cli and bob-plugins

1. **Fixture.** Write the shared vector table above, byte-identical, in both repos.
2. **Rust cleaner** (`src/native/bank/grammar.rs`). Replace the
   `strip_markers_fields_links` note cleaning with the five-step contract.
   `parse_row_tail`'s opener logic is unchanged. Add a unit test that loads
   `tests/fixtures/bank_notes/note_cleaner.json` (for example via `include_str!`) and
   asserts payee, flagged, and note for every case. Keep
   `flags_notes_and_ascii_spellings` green. Add a regression test that payee learning
   (`naming.rs`) ignores the note: a noted row learns the same payee as the same row
   without a note.
3. **JS cleaner** (`241o-plugin-bank-marks-model.js` `bankParseRowTail`). Implement the
   same five steps and add a parity test in `scripts/test-ledger-tools-bank-marks.cjs`
   (or a new `test-ledger-tools-bank-note-grammar.cjs`) over the plugin copy of the
   fixture, using `bankTokenizeLine`.
4. **Disjoint note spans** on every tokenized row (JS only):
   - `noteSpan`: the content of the first non-empty _prose run_ after the opener. A
     prose run is a maximal stretch of tail text that contains no field, wikilink, `⚑`,
     or `!!`. Its leading whitespace and one optional leading `—`/`--` marker are
     excluded, and so is its trailing whitespace. It never overlaps another token's
     span.
   - `noteMarkerSpan`: the `—`/`--` immediately before `noteSpan` (only whitespace
     between), or null (for example a flag-opened note `!! with Kelly`).
   - `noteEditable`: true when exactly one non-empty prose run exists.
   - Keep `note` (the cleaned value) as is. Add span tests for marker, ASCII `--`,
     flag-opened, ID and ID-less, links before and after the note, and multi-run legacy
     notes. Also add a test that `RangeSetBuilder` input stays sorted and
     non-overlapping for noted rows.
5. **Subscription link order** (`241hc-plugin-bank-subscription.js`
   `bankSubscriptionLinkWritePlan` and `bankSubscriptionGhostOffset`; check `248h`
   `bankCategorySubscriptionWritePlan`, which builds on them). Insert the `🔁` link, and
   place its ghost-chip offset, at the earliest of flag start, note marker start, note
   start, block-ID start, and line end. This keeps canonical order
   `[[links]] ⚑ — note ^id`. Update tests that pinned the old after-flag position.
6. **Whole-row `$` protection.** `token.dollars` covers every `$` on a row line, not
   only the amount's. In Live Preview, every `$` not already inside another replace
   decoration gets `BankDollarWidget` in both the decorated and undecorated branches, so
   `split $25 w/ Alex, $10 tip` can never pair into MathJax. In Reading view,
   `bankProtectMathInElement` also protects a math node whose source text occurs inside
   a bank row's line text. Add tests using `math-rows.md`-style fixtures with `$` in
   notes.
7. **Docs** (bob-cli `docs/bank.md` §Row grammar): add the canonical tail-order line,
   and state the note value rule (opener; fields, links, `⚑`, and `!!` removed;
   whitespace collapsed; one leading marker dropped).

Acceptance: both parsers pass every shared vector; noted rows produce disjoint spans; a
🔁 link added to a noted or flagged row lands before the flag or note; `$` inside notes
never renders as math. Run `just check` and `just docs-check` in bob-cli, then
`npm test` and `bob plugins sync` in bob-plugins.

### note-caption: Quiet prose caption on every row surface

**Size:** small · **Depends on:** `note-grammar-parity` · **Repos:** bob-plugins

1. **Live Preview** (`241p-plugin-bank-marks.js`). Add `BankNoteWidget`, whose `eq` keys
   on the text and whose `ignoreEvent()` returns false. In the decorated branch it
   replaces `[noteMarkerSpan?.start ?? noteSpan.start, noteSpan.end]` and renders
   `<span class="bob-bank-mark-note" title aria-label><span class="bob-bank-mark-note-dash" aria-hidden="true">—</span><span class="bob-bank-mark-note-text">…</span></span>`.
   The text is the `noteSpan` bytes with whitespace collapsed. Do not add
   `BankDollarWidget`s inside that range. Multi-run legacy notes caption only their
   first run; the rest stays raw.
2. **Reading view** (`241q-plugin-bank-marks-reading.js`). Wrap the marker-through-note
   needle with the same caption. If that needle is not one text node (for example the
   note renders markdown), fall back to the text-only needle with a dash-less caption
   variant. If both fail, leave the row untouched and never double the dash.
3. **Queue card** (`241m-plugin-bank-cards.js` `bankCardRenderQueueRow`). After the
   payee, add `<span class="bob-bank-queue-note">` with `— {row.note}`, a full-text
   title, and one ellipsized line.
4. **CSS** (`plugins/bob-ledger-tools/styles.css`, near the other `.bob-bank-mark-*`
   rules): `.bob-bank-mark-note` is italic in `--text-muted` with
   `margin-inline-start: 0.35em`, and the dash is upright in `--text-faint` with a small
   end margin. Add the acknowledged-row contrast override, the queue-note style, and a
   mobile `@media` rule that wraps rather than overflows. Follow the
   `test-ledger-tools-block-marks-contrast.cjs` pattern to assert ≥ 4.5:1 for the
   caption on waiting and acknowledged rows.
5. **Tests** in `scripts/test-ledger-tools-bank-marks.cjs` and
   `test-ledger-tools-bank-cards.cjs`. Caption present when the cursor is off the line
   and absent when it is on; `--` and flag-opened rows; no overlap with chips, flag, ID,
   or dollar widgets; Reading view wrap and fallback; queue note present or absent.
6. bob-plugins docs (`docs/bob-ledger-tools.md`): one sentence on how notes render. Then
   run `npm test`, `npm run docs:check`, and `bob plugins sync`.

Acceptance: a row like `- [ ] $23.21 Amazon Reta* 125dt [cat:: kids] — LEGO set ^bank-…`
reads as amount, payee, category pill, then a muted italic `— LEGO set` caption and a
faint `bank ✓`, in both Live Preview and Reading view.

### note-writer: Pure note normalizer, write planner, and stage preview model

**Size:** small · **Depends on:** `note-caption` (bob-plugins serialization; logically
only `note-grammar-parity`) · **Repos:** bob-plugins

1. New pure fragment `246a-plugin-bank-note-write.js` (no Obsidian globals; register it
   after `246` in `fragments.json`; export helpers through
   `356-exports-bank-category.js` or a new sibling export fragment if that file would
   pass 1000 lines):
   - `bankNoteNormalize(text)` returns
     `{ ok, text, changes: [{ from, to, why }], refusal }` per the normalizer table.
   - `bankNoteWritePlan({ capture, currentPath, currentSource, text })` resolves through
     `bankCategoryResolveForCommit`. It refuses non-editable (multi-run) notes with
     `Bank: this note has links or fields — edit it on the row`, refuses `#task`, and
     returns
     `{ ok, line, action: "insert"|"replace"|"remove"|"noop", edits, note, changes }`
     using the storage rules. It enforces the round-trip self-check before returning ok.
   - `bankNoteStagePreview({ draft, existingNote, editable })` returns
     `{ action, text, changes, refusal, consequence, keys }`, with the exact consequence
     and keys copy from the design, so the stage only renders.
2. Tests (new `scripts/test-ledger-tools-bank-note-write.cjs`, added to `test:unit`):
   - The normalizer table, row by row.
   - Insert, replace, and remove × {`^bank-` ID, ID-less, custom block ID, `⚑`-flagged,
     `!!`-flagged, links before the flag, `--` marker, CRLF source}.
   - Flag always survives a remove; replace keeps the `--` spelling; refusals for
     multi-run notes, `#task`, a moved row, and an edited ID-less row.
   - A property-style loop: every normalizer output inserted into every row shape
     tokenizes back to exactly that note, with all other tokens unchanged.
   - Preview copy for every action.
3. `npm test`, then `bob plugins sync`.

### note-stage: Space keymap, in-card note stage, stay-on-row commit, command, and docs

**Size:** medium · **Depends on:** `note-writer` · **Repos:** bob-plugins and bob-cli

1. **Key model** (`245-plugin-bank-category-card-model.js`):
   - `" "` → `{ action: "note" }`; `"-"` → `no-category`; `Delete` → `no-category` in
     set mode only.
   - Restricted Enter → acknowledge (`no-category`), except while loading. Track an
     explicit `loading` flag on the modal; do not string-compare the reason.
   - `header.note` and `header.noteEditable` come from the card row. Session gains
     `notes` via a new `bankCategorySessionNote` that never touches `filed`.
     `bankCategoryEndOfRunSummary` accepts `notes` and appends `· N notes` when N > 0.
   - Update `bankCategoryCardKeysText` (`248f`) to the footers above.
   - Update `test-ledger-tools-bank-category-card.cjs` for the new key matrix (Space,
     `-`, and `Delete` across close, set, restricted, and restricted-loading) and for
     everything else unchanged.
2. **Context** (`248` `bankCategoryContextAt`): add `note` and `noteEditable` to the
   card row from the token.
3. **Stage** (new `248n-plugin-bank-note-stage.js`, prototype methods on
   `BankCategoryCardModal` in the `248g` style):
   - `openNoteStage({ direct })`, `closeNoteStage()`, `shouldPassNoteStageKey(event)`,
     `routeNoteStageKey(event)`, `paintNoteStage(root)`, `saveNoteStage()`.
   - Keep the input element across repaints, like `newStageHost`, so focus and caret
     survive.
   - Wire `247` `routeCardKey` and the scope listener so the note stage routes before
     category keys, mirroring the new-category and subscription stages, and add the
     stage to `onEscapeKey` layering and `setNext` and `onClose` resets.
   - Paint the header caption button and the stage exactly as the design describes: same
     position, dimmed `inert` tiles, consequence and change chips, keys line, and
     `.is-fresh` glow.
4. **Commit** (`bankNoteCommit` on the Category card action mixin in `248`, or a sibling
   fragment if `248` would pass 1000 lines). Follow the write rules above: apply, save
   with an accurate outcome, keep the cursor, re-capture, re-anchor the walk, count
   `notes`, re-arm the chord guard, announce, and never advance. In direct mode, save
   then close with a Notice.
5. **Command** `bank-note-row` / `Bank: note this row`, registered beside
   `bank-set-category` (`241k` is near 1000 lines; register it from the new fragment if
   needed). It opens set mode in direct note mode and skips the report fetch. With no
   bank row under the cursor, it shows `Bank: put the cursor on a bank row first`.
6. **CSS** for `.bob-bank-category-note` (two-line clamp, button reset, hover
   underline), `.bob-bank-category-note-stage`, `-dash`, and `-input` (hairline
   underline and accent focus), `.is-staged` tiles, `.is-fresh` (600 ms wash), the
   consequence and change chips, and `prefers-reduced-motion` overrides.
7. **UI tests** (`test-ledger-tools-bank-category-card-ui.cjs`, plus a new
   `test-ledger-tools-bank-note-stage.cjs` if that file would get crowded):
   - The opening Space does not type, and a repeated Space is ignored. `q`, `-`, `+`,
     and letters type in the stage. Esc, `q` outside the input, and Ctrl+[ back out one
     layer, and a second Esc closes the card.
   - Enter and Ctrl+Enter save and stay on the same row: no tick, no advance, no jump,
     `didInteract` unchanged, and the highlight kept.
   - The next hotkey then files the _same_ row and advances. ID-less rows re-capture,
     and the walk anchor follows the new text.
   - The restricted card writes notes, and Enter does nothing while loading.
   - The structured-note hint appears; refusals keep the draft; IME Enter does not save;
     the summary counts notes.
   - Direct mode saves and closes; the command refuses off-row. Update existing tests
     that pinned Space = no-category (`-` now).
8. **E2E** (`scripts/obsidian-bank-ui-e2e.cjs` and its offline wrapper). Change the
   `walk-space` step to press `-`. Add a `walk-note` step: Space, type `for the team`,
   Enter. The row gains ` — for the team`, stays `[ ]`, and the card stays on it. Then
   Enter accepts and advances, and one Ctrl+Z undoes only the filing while the note
   remains. Run it with `xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs` when the
   environment allows, and say so in the bead notes if it cannot run.
9. **Docs:**
   - bob-plugins `docs/bob-ledger-tools.md`: rewrite the Category card paragraph's key
     sentences (Space note stage, `-` no category or remove, restricted Enter) and
     regenerate the commands table (`npm run docs:gen`).
   - bob-cli `docs/guides/acknowledge-bank-transactions.md` step 2: Space leaves a note
     and stays, `-` files with no category, and the summary counts notes. Its "If it
     goes wrong" section now reads "restricted (Enter, Space, and Esc only)". Its row
     basics stay.
   - bob-cli `docs/bank.md` §Ownership: the card's category _and note_ are Bryan's edits
     in his editor, applied through single editor transactions.
   - Run `npm run docs:check` and `just docs-check`.
10. **Memory**, only as the decisions direct:

    > [!decision] memory_keymap_decision Use `/sase_memory_write`. Add the decisions web
    > strand `decisions:space-writes-a-bank-row-note` in the existing record format
    > (origin, scope, keyword, summary, aliases, `metadata.status: accepted`, decided
    > date). Claim: Space on the bank Category card opens the note stage and never
    > answers or advances; `-` files with no category (close), removes it (set), or
    > acknowledges (restricted); restricted Enter acknowledges once loaded. Rejected
    > alternatives: keep Space and bind `-` or `.` to the note; Backspace/Ctrl+D;
    > Shift+Space. Evidence: this plan and
    > `research:202611/bank_row_note_stage/bank_row_note_stage.md`. Reopens when: Bryan
    > files many no-category rows, mis-presses in the first week, or wants links in
    > notes. Then run `sase memory init`.

    > [!decision] memory_bank_row_glossary Use `/sase_memory_write`. Add one sentence to
    > the `glossary:bank-row` strand: an optional ` — note` tail carries Bryan's note,
    > and Space on the Category card writes it without leaving the row. Then run
    > `sase memory init`.

    When a decision is false and no human reviewed the plan, record the skipped change
    as a `PROPOSED FOLLOW-UP:` note on this phase's bead. If a human turned it off,
    record nothing.

11. Run `npm test`, `npm run docs:check`, `bob plugins sync`, `just check`, and
    `just docs-check`.

Acceptance: during a `]b` walk, `Ctrl+Enter → Space → "LEGO set for Max" → Enter` leaves
the row `[ ]` with ` — LEGO set for Max` before its ID. The card stays on that row with
the suggestion highlighted and the caption glowing once. `d` then files that row and
advances, and Ctrl+Z unwinds the filing first and the note second.

## Risks and mitigations

- **Muscle memory for Space = no category.** Pressing Space by reflex opens a harmless
  empty stage; Esc backs out, and nothing is written.
- **Text-keyed (ID-less) rows going stale after a note.** The commit re-captures and
  re-anchors, and a UI test pins that the next key files the same row.
- **Parser drift between bob and the plugin.** The byte-identical shared vector table is
  tested in both repos.
- **Undo merging.** `isolateHistory`, verified by the e2e Ctrl+Z step.
- **Vault deploy conflicts.** The bob-plugins phases are serialized, and each follows
  the sync-refusal runbook.
