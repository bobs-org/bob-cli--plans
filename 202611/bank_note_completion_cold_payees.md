---
tier: tale
title: Bank note completion shows past notes on every row
goal:
  The Bank Category note field suggests past notes even on payees with no note history.
size: medium
decisions:
  empty_field_fill:
    ask:
      When the note field is empty, should other payees' notes fill the list after this
      payee's own notes?
    choices:
      fill:
        Every row shows up to 4 notes, this payee's first and then other payees'
        most-used recent notes
      only_when_none:
        Other payees' notes appear only when this payee has none of its own
    default: fill
    why:
      one rule on every row; this payee's notes still lead, so nothing it already finds
      moves down
    answer: fill
  empty_ghost:
    ask:
      On an empty field, should the faint ghost text preview the top note even when it
      came from another payee?
    choices:
      own_payee_only:
        Only this payee's own notes ghost; new payees keep the "What was this for?"
        placeholder
      any_payee:
        The top note always ghosts, so the placeholder disappears on every row that has
        suggestions
    default: own_payee_only
    why: a ghost reads as "this is probably it", which an unrelated payee's note is not
    answer: own_payee_only
proposed_by: bbugyi200.athena.1hp
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-28 08:22:25
status: wip
---

# Plan: Make bank note completion visible on payees without history

## Why

Task `^fix-bank-note-cmpl` ("Fix bank note completion!") reports that yesterday's
`^cmpl-past-bank-notes` feature "doesn't seem to be working". Its screenshot shows the
Bank Category card's note stage on `$147.52 Total Wine 304 Dig` (2 of 5, Fri Nov 27).
The field is empty with the "What was this for?" placeholder, shows no suggestion rows,
and the keys line reads `↵ back · esc back`.

## Diagnosis (already done, do not repeat)

The feature is deployed and running, and its wiring works. Its empty-field rule hides it
on most rows:

- The Mac runs `bob-ledger-tools` 1.175.1. Obsidian started at 06:58:35, 11 s after
  `main.js` was written. A live `obsidian eval` probe on the Mac showed
  `bankNoteLibrary()` with `available: true`, the bank walk ready, and all 8 notes in
  `bank/2026-11.md`.
- The real-Obsidian e2e `walk-note-suggest` and `walk-note-suggest-save` cases pass on
  athena.
- Running `bankNoteSuggest` on the real vault month gives these results:
  - `""` on `Total Wine 304 Dig` returns `[]`.
  - `"As"` on `Total Wine 304 Dig` returns `["Ask Kelly?"]`.
  - `""` on an `Amazon Reta*` row returns `Ask Kelly?`, `Binoculars for Abby`, and
    `Kelly sent me an email…`.

The original design (plan `plan:202611/bank_note_completion.md`) has two rules that
cause this:

1. **Empty query is affine-only.** On an empty field, only notes whose uses share this
   row's merchant key appear ("Other merchants' notes never appear on an empty field").
   Total Wine has no history. `Amazon Mark` (row 1 of that session) is not the
   `amazon reta` merchant family. So both rows Bryan filed that morning opened to a
   blank list with no sign the feature exists.
2. **Two-character global gate.** `BANK_NOTE_SUGGEST_GLOBAL_MIN = 2` hides other payees'
   notes until two characters are typed. Even after rule 1 is fixed, an empty field
   would show other payees' notes, the first keystroke would hide them, and the second
   would bring them back.

The fix changes the ranking policy in the pure model and leaves the stage, keys, index,
and write path alone. All code lives in the linked **bob-plugins** repo. Open it with
`/sase_repo` (`sase repo open bob-plugins`) and read its `AGENTS.md`. bob-cli code is
untouched, and nothing changes in any CLI or JSON contract.

Governing decisions are unchanged:

- `decisions:relation-card-tab-accepts-ctrl-d-hides`: selection by key, and Tab
  completes the selected or top row.
- `decisions:q-and-ctrl-bracket-mirror-esc`: `q` types in the field, and Esc, `q`, and
  Ctrl+[ back out.

## The experience after the fix

On `Total Wine 304 Dig`, Space opens the note stage and shows:

```text
—  What was this for?
   ⇥  Ask Kelly?                                  Amazon Reta · 4× · Nov 25
      Aligator toy for Lenard!                    Amazon Mark · Nov 27
      Great Tacos!                                La Malinche Restau · Nov 23
      Binoculars for Abby                         Amazon Reta · Nov 23
   No change
⇥ complete · ⌃N/⌃P choose · ↵ back · esc back
```

Typing `g` narrows the list to `Great Tacos!` (prefix match, with its ghost), followed
by the `…groceries.` note (word-boundary match). On an `Amazon Reta*` row, this payee's
notes stay on top as they do today, and other payees' notes fill the remaining slots
below them.

## Changes (bob-plugins, `plugins/bob-ledger-tools/`)

### 1. `src/246b-plugin-bank-note-suggest-model.js` (pure model)

**Empty query.** Keep every candidate, affine or not.

> [!decision] empty_field_fill = fill Order affine candidates first, as today: affinity
> score desc, then last affine use desc, then folded key asc. Then order non-affine
> candidates by global score desc (frecency, the existing 30-day half-life), then last
> use desc, then folded key asc. Concatenate the two groups and cap at
> `BANK_NOTE_SUGGEST_LIMIT` (4).

> [!decision] empty_field_fill = only_when_none If at least one affine candidate exists,
> the empty query returns exactly today's affine-only list. Otherwise it returns the
> non-affine candidates in the global order above, capped at 4.

On an empty query, non-affine rows keep their existing meta
(`Merchant Label · N× · Mon D`, computed over all uses). Affine rows keep `N× · Mon D`.
`matchClass` stays `0` and `highlight` stays `null` for every empty-query row, as today.

**Typed query.** Remove the two-character gate. Any non-empty folded query matches
non-affine notes too. Delete `BANK_NOTE_SUGGEST_GLOBAL_MIN` and the line that uses it in
`bankNoteSuggest`. Remove the constant from `src/356-exports-bank-category.js` and from
any test that imports it. Typed ordering is unchanged: class asc, affine first, score
desc, last use desc, key asc.

**Ghost.** `bankNoteSuggestGhost` already receives the rows, so the field value tells it
the field is empty.

> [!decision] empty_ghost = own_payee_only When the field value is empty, return `null`
> unless `top.affine === true`. The stage then keeps the "What was this for?"
> placeholder, because `refreshNoteSuggestions` already restores it whenever there is no
> ghost. Typed-query ghost rules are unchanged.

> [!decision] empty_ghost = any_payee Leave `bankNoteSuggestGhost` unchanged. An empty
> field ghosts the top row whatever its payee.

Leave these unchanged: the library rules (kept notes, folding, self-row exclusion,
most-recent spelling), merchant-family keys, the 4-row limit, and the half-life.

### 2. Stage and index

No change is expected in `248n`, `248o`, or `249a`. The stage already renders
`metaMerchant` for non-affine rows, the `⇥` marker on the top row, and the keys-line
prefix whenever rows exist. Confirm this by reading, and change those files only if a
test proves a gap.

### 3. Build

Run `npm run build`. Never hand-edit `main.js`. Each hand-edited fragment must stay at
or below 1000 lines.

## Tests

- `scripts/test-ledger-tools-bank-note-suggest.cjs`:
  - Rework "empty query shows only affine notes…" into a test of the chosen
    `empty_field_fill` branch. Affine rows lead in today's order. For `fill`, the
    non-affine rows follow in global-score order, with a frecency tie broken by last
    use, and the result is capped at 4.
  - Add a cold-payee case built from the real vault's shape. On `Total Wine 304 Dig`
    (date `2026-11-27`), an empty query returns `Ask Kelly?` first, with meta
    `Amazon Reta · 4× · Nov 25`, then the other notes, capped at 4. It never includes
    the row's own note.
  - Replace the "two-character global minimum" assertions. A one-character query (`a`)
    now includes non-affine prefix matches such as `Abby gift`, and affine rows still
    lead each class. Keep the trailing-space, exact-match-excluded, and four-row-limit
    assertions.
  - Ghost: an empty value with a non-affine top row returns `null` under
    `own_payee_only`. An empty value with an affine top row still ghosts, and so does a
    typed value with a non-affine class-0 top row.
- `scripts/test-ledger-tools-bank-note-stage.cjs`: update option counts that the fill
  changes. On the Amazon row (`cardOn(2)`), the fixture library's `Corner Cafe` note
  `Ask before noon` now follows the two affine notes. Add a stage case for the
  `$2.00 Market` waiting row (`cardOn(4)`), whose payee has no notes in `NOTE_LIBRARY`.
  Space shows other payees' rows, each with a merchant label. The placeholder stays
  "What was this for?" and no ghost tail renders. The keys line gains the
  `⇥ complete · ⌃N/⌃P choose ·` prefix. Tab completes the top row without saving. Keep
  "an empty library preserves today's copy" passing unchanged.
- `scripts/obsidian-bank-ui-e2e.cjs`: after `walk-note-suggest-save`, add a
  real-Obsidian case named `walk-note-suggest-cold`. In a separate scratch month (for
  example `bank/2025-11.md`, so the walk fixtures stay undisturbed), open the card
  through `api.bank.claimRowClose` on a waiting row whose payee has no notes (for
  example `Total Wine 304 Dig`), then press Space.
  - Assert that the options include `Ask Kelly?` with its `Amazon Reta` merchant label,
    the placeholder is "What was this for?", and the ghost tail is empty.
  - Type `a` (through `setNoteDraft` or an input event) and assert `Ask Kelly?` stays
    listed.
  - Capture `walk-note-suggest-cold.png` with the existing `captureCategoryModal`
    helper, and inspect it.
- Run:
  - `npm run build:check`
  - the two focused test files with `node --test`
  - `npm test`
  - `npm run docs:check && npm run docs:inventory:check`
  - `xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs --out <tmp dir>`

  The e2e run already fails on `queue-card-live`, `queue-card-reading`, `invalid-fence`,
  `walk-learn-coffee`, `walk-learn-accept`, and `walk-real-key-swallowed`. Those
  failures are tracked (for example bob-cli-m6 and bob-cli-m7) and are out of scope. The
  pass condition is that every `walk-note-suggest*` case passes and no other case newly
  fails.

## Docs

`docs/bob-ledger-tools.md` (the Category card paragraph, around line 141) says "this
payee's notes appear first even when empty; others appear after two characters." Replace
that sentence so it describes the chosen branches, for example: "this payee's notes
lead; other payees' notes fill an empty field and match from the first character." That
page sits near its 6000-word lint cap (it was 49 words over on Nov 27), so the new
wording must not be longer than the sentence it replaces. Check whether any other doc or
help string repeats the two-character rule, and fix it too.

## Deploy

Follow bob-plugins `AGENTS.md`. After committing, run `bob plugins sync`. If sync
refuses, use the documented no-`-F` recipe. The MacBook deploys from its own bob-plugins
checkout, so the final response must tell Bryan that the Mac needs its next
`bob plugins sync` and an Obsidian plugin reload (or app restart) before the fix shows
there. Do not SSH to the Mac to deploy.

## Out of scope

- Widening merchant families, for example so `Amazon Mark`, `Amazon Mktpl`, and
  `Amazon Reta` count as one merchant. With the fill, Amazon notes already appear on
  every Amazon row.
- The six pre-existing bank e2e failures.
- Editing the vault task `^fix-bank-note-cmpl`, decision records, or other SASE memory.
  No memory note states the rules this plan changes.
