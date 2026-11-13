---
tier: epic
title: Bank, weekly — file every row with one key
goal: 'Bank tracking becomes one Monday chore: a Wells Fargo download link, one import
  command, then Ctrl+Enter on each waiting bank row opens a Category card where a
  single key ticks the row, files its category from Bryan''s own list, and lands the
  cursor on the next waiting row, so a week of transactions takes about two minutes.

  '
decisions:
  split_subscriptions:
    ask: Split the subscriptions category into Tech & AI and Media?
    default: true
    why: Categories say what you bought; the subscriptions album (bob-cli-kt) owns
      recurrence.
    answer: true
  prefill_seed_rows:
    ask: Should the rollout pre-file the seven already-acknowledged November rows
      with categories?
    default: true
    why: They hold the most frequent merchants, so the first Monday walk starts with
      warm suggestions.
    answer: true
  kv_snooze:
    ask: When should the bob-cli-kv bank-feed spike wake up?
    choices:
      hold: Keep its 2026-12-14 snooze; the walk trial decides whether a feed is still
        needed
      now: Unsnooze it when this epic lands so a feed spike can run beside the trial
    default: hold
    why: The trial ends on the snooze date, and the walk may drop the download toil
      enough.
    answer: hold
  memory_bank_row:
    ask: Update glossary:bank-row to say Ctrl+Enter files a waiting row with a category?
    memory:
    - glossary:bank-row
    default: false
    answer: true
  memory_walk_record:
    ask: Add a decisions record for how a bank close files a category and advances?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: weekly-chore
  title: One weekly Bank chore with the Wells Fargo link
  depends_on: []
  size: small
  description: 'weekly-chore: in the vault, turn the Monday Bank import chore into
    the single weekly Bank chore (verified Wells Fargo download link, #browser, one-line
    import command, the walk), and cancel the daily Acknowledge bank occurrence with
    history intact.'
- id: qfx-cleaning
  title: Clean Wells Fargo QFX payees and merchant keys
  depends_on: []
  size: medium
  description: 'qfx-cleaning: teach naming.rs the real QFX description shapes so payees
    lose dates, places, card digits, order codes, and the account holder''s name;
    merchant keys collapse repeat merchants; learned payees are re-cleaned; the default
    account stops getting [acct::].'
- id: category-contract
  title: Category hotkeys, groups, hidden, and row suggestions
  depends_on:
  - qfx-cleaning
  size: medium
  description: 'category-contract: add the mapping form for bank.categories (label,
    hotkey, group, hidden) with derivation and validation, report hotkey/group/hidden
    in categories[], add a read-time suggestion {category, source} to every row JSON
    from one function shared with import, and point import''s next step at the walk.'
- id: vault-rollout
  title: Bryan's taxonomy in bank.md and clean seed rows
  depends_on:
  - category-contract
  size: small
  description: 'vault-rollout: after every host runs the new bob, write the journal-grounded
    category list with hotkeys and groups into bank.md, drop the dead VENMO rule,
    re-clean the seven seed rows (and pre-file them per the decision), and update
    the hub prose to the weekly ritual.'
- id: card-model
  title: Pure Category card, write, and advance models
  depends_on:
  - category-contract
  size: medium
  description: 'card-model: in bob-ledger-tools, add pure, unit-tested models for
    decoding the new JSON, the card view model and key table, the single-transaction
    row write plan, the next-waiting-row finder, the walk session counter, and the
    end-of-run summary.'
- id: card-ui
  title: The Category card, the walk, and Bank set category
  depends_on:
  - card-model
  size: medium
  description: 'card-ui: render the Category card in the key-card family, wire api.bank
    v2 claimRowClose and setCategory, commit, advance, refresh suggestions after each
    filing, show the end-of-run notice, add Bank: set category, hide hidden categories
    from [cat:: suggestions, and land waiting-pill jumps at the payee.'
- id: ctrl-enter-walk
  title: Ctrl+Enter hand-off, jump history, real-Obsidian proof, deploy
  depends_on:
  - card-ui
  - vault-rollout
  size: medium
  description: 'ctrl-enter-walk: make task-status-cycler delegate Ctrl+Enter on waiting
    bank rows to ledger-tools with today''s flip as fallback, add nav''s recordJump
    member for the Ctrl+O landing, extend the real-Obsidian bank e2e, document the
    plugins, and deploy with bob plugins sync.'
- id: wrapup
  title: Weekly guide, contract docs, memory, and the trial
  depends_on:
  - weekly-chore
  - ctrl-enter-walk
  size: medium
  description: 'wrapup: rewrite the bank guide for the weekly walk, note the card
    in docs/bank.md ownership, prove the real vault end to end with counts only, apply
    the memory and kv decisions, and record the 2–4 week trial as a proposed follow-up.'
proposed_by: bbugyi200.athena.1c7
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 10:55:23
status: done
bead_id: bob-cli-lc
---

- **PROMPT:** [prompts/202611/bank_weekly_category_walk.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_weekly_category_walk.md)
- **BEAD:** [bob-cli-lc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lc/README.md)

# Plan: Bank, weekly — file every row with one key

## Why

The `bob-cli-kd` epic restarted Bryan's bank habit: a `bank` sub-area, monthly bank
month notes of checkbox bank rows, a QFX import that writes only what he missed, and a
balance proof. Two mornings in, the daily ritual had already turned into a daily QFX
download, which is too much toil to justify. Bryan asked for:

- Ctrl+Enter on a bank row should jump to the next unticked bank row.
- The bank chore in `~/bob/gtd_daily.md` should carry a good Wells Fargo link and recur
  weekly instead of daily.
- A user-configured category list (categories exist only partly today), grounded in the
  transactions he imported on 2026-11-12, without exposing his personal life.
- Closing a bank row with Ctrl+Enter prompts for a category that takes a single key.

The consolidated research report
`research:202611/bank_weekly_category_walk/bank_weekly_category_walk.md` reviews this
request, and Bryan agreed with every requirement it recommends. Every phase worker reads
the sections relevant to its phase with
`sase artifact read research:202611/bank_weekly_category_walk/bank_weekly_category_walk.md "<why>"`.
For the bank contract, read `docs/bank.md` and `bead:bob-cli-kd` (with its plan,
`plan:202611/bank_daily_acknowledgment.md`). Re-verify any number your phase acts on,
because the vault moves daily.

Evidence, re-checked while planning (aggregate only):

- **Two bank chores, not one.** `gtd_daily.md` holds a daily **Acknowledge bank** chore
  (`[repeat:: every day when done]`; its completed occurrences carry `^acknowledge-bank`
  and a LINKED FROM line) and a Monday **Bank import** chore
  (`[repeat:: every week on Monday when done]`, scheduled 2026-11-16).
- **Categories are mostly built.** `bank.md` declares 14 scalar categories plus the four
  built-ins, and has one `VENMO → transfer` rule. The grammar, `check`, totals,
  breakdowns, import-time rules, and the learned tier all ship. What is missing is the
  moment of choice: Ctrl+Enter on a bank row flips one status byte (`task-status-cycler`
  `src/192-bank-row-guard.js`, `cyclerBankLocalFlip`) from two entry points
  (`handleToggleOpenDoneCommand` and `handleVimTaskToggleOpenDone`).
- **The journal.** 850 `bank.txn` events from 2026-05-01 to 2026-11-12: 843
  `before_start` and 7 `written`. About 31 rows a week. The top 10 merchants cover over
  half the rows. Inflows are rare (8 in 6.5 months).
- **The cleaner misses the real QFX shape.** About 86% of descriptions carry Wells
  Fargo's `PURCHASE MM/DD … CARD dddd` tail, and none carry the `AUTHORIZED ON` phrase
  `naming.rs` was built for. Because the purchase date pollutes the merchant key, the
  learned tier recognizes a repeat merchant on roughly one row in five.
- **A privacy leak.** Six of the seven rows in the git-synced vault carry the card's
  last four digits in the payee. One carries the account holder's name.
- **Noise.** Import writes `[acct:: checking]` on every row even though `checking` is
  the default account, contradicting `docs/bank.md` ("written only for a non-default
  account"). `grammar::render_row` is called with `Some(&pending.account)`.
- **No VENMO rows.** The 6.5-month journal has zero VENMO descriptions; person-to-person
  payments use another rail. The one rule is dead weight and would misfile shared
  household payments as transfers.
- **The seed rows.** `bank/2026-11.md` holds seven rows under Thu Nov 12: all ticked,
  all with `^bank-` IDs, none with `[cat::]`.

## The Monday this builds

The weekly **Bank** chore appears in Monday's PRE list. Its link opens Wells Fargo's
Download Account Activity sign-on. Bryan picks Checking and Quicken (QFX), keeps the
default dates, and runs one remembered command on the Mac:
`bob bank import ~/Downloads/*.qfx && rm ~/Downloads/*.qfx`. Import ends with
`Next: file the 31 waiting rows — Ctrl+Enter on each in bank/2026-11.md`.

In Obsidian he clicks the month card's `31 to acknowledge` pill. The cursor lands on the
first waiting row's payee. Ctrl+Enter opens the Category card with ☕ Coffee ringed
(`learned`), and ↵ (or a second Ctrl+Enter) files it. The row ticks, gains
`[cat:: coffee]`, flashes once, and the cursor lands on the next waiting row. A letter
picks any other category, and Space files a row with no category. After the last row a
notice closes the loop with bob's numbers: `🏦 November filed — 31 rows in 1m 48s` and
`WF Checking ✓ balanced · Nov 13`. Then comes the one line in last week's note: what
surprised me?

## Settled requirements (research, accepted by Bryan)

- **A1. One weekly chore.** Merge the two chores. The Monday import chore keeps its
  identity and recurrence and becomes **Bank**. Cancel the daily chore's active
  occurrence, and leave its completed occurrences, `^acknowledge-bank`, and the LINKED
  FROM line untouched.
- **A2. The verified link.** Use Wells Fargo's published Download Account Activity
  sign-on deep link
  `https://connect.secure.wellsfargo.com/auth/login/present?origin=cob&loginMode=jukePassword&serviceType=accountActivityDownload&LOB=CONS`
  with `#browser`, and name the click path in the text as a fallback. QFX only: never a
  CSV or PDF substitute.
- **A3. The card always opens on a waiting row.** bob's suggestion is pre-highlighted. ↵
  accepts, a letter overrides, Space files with no category, and Esc cancels.
- **A4. Advance within the note.** Move forward to the next waiting row, wrap once to
  the first remaining one, and never auto-open another file. The end of the run shows a
  notice with bob's numbers and names other months that still have waiting rows.
- **A5. A journal-grounded list.** Build the category list from the full journal, not
  just the seven new rows. Every rename is free while no row carries a category.
- **A6. Cleaning first.** Fix WF QFX payee cleaning and the merchant key, and clean
  learned payees before reuse.
- **A7. Undo and correction.** Ctrl+Enter on a ticked row stays today's in-place untick:
  no card, no jump. A separate **Bank: set category** command opens the same card on any
  row without touching its checkbox.
- **A8. Bulk paths stay category-free.** `bob bank ack`, `ack -D`, the queue card's day
  ack, and checkbox clicks only tick.
- **Answered research questions.** `household` (person-to-person payments that are
  really shared costs) counts as spending, never `transfer`. The chore stays on Monday.
  `kids` is hidden, because nothing in 6.5 months used it (one line restores it).
  `personal` keeps its neutral name, and Bryan may rename it freely. Splitting
  subscriptions follows `split_subscriptions`, and the feed spike follows `kv_snooze`.
- **Out of scope.** Splits, budgets, per-row due dates, pending transactions, cloud/ML
  categorization, a category manager, category hierarchies, backfilling the 843
  `before_start` transactions, rules created automatically from a choice, an `ack --cat`
  flag, a new `bob bank` subcommand or option, and the subscriptions album
  (`bob-cli-kt`).

## Design calls this plan makes beyond the research

1. **Suggestions are computed by bob at read time, not just stamped at import.** Every
   row JSON gains `suggestion {category, source}`. Import's category tier calls the same
   function. This lets the walk **learn inside one session**: after Bryan files the
   first coffee row, bob suggests Coffee for the next one. This matters most in the
   first weeks, when no history exists.
2. **The walk refreshes bob's view after each filing.** After each filing, the plugin
   saves the note and re-fetches that month's `list`. The next card waits up to 800 ms
   for that fetch before it opens. A highlight never changes under Bryan's finger, and ↵
   only accepts a highlight that is on screen.
3. **A key never changes meaning.** Space always means "no category" (it removes an
   existing `[cat::]`). Esc, `q`, and `Ctrl+[` always cancel. ↵ and a second Ctrl+Enter
   accept what is highlighted. An inflow reorders the groups visually, but no key moves.
4. **The card degrades and never blocks acknowledging.** If ledger-tools is missing or
   older than bank API v2, Ctrl+Enter keeps today's flip. If bob or the configuration is
   unreachable, a restricted card offers Space and Esc and upgrades in place when
   categories arrive before the first keypress. If bob reports no hotkeys (an old bob),
   tiles are reachable by arrows and ↵. The plugin never derives a hotkey.
5. **A walk session counts progress and pays it back.** Progress reads `N of M` (filed
   this session + 1, of filed + still waiting). The end-of-run notice reports rows,
   elapsed time, and accepted / picked / no-category counts. Bryan sees the trial's
   numbers without telemetry, and nothing is persisted.
6. **The chore removes toil, not just cadence.** Bryan keeps Wells Fargo's default date
   range (it must cover at least 14 days; the overlap heals a skipped week). He runs one
   history-recallable command that imports and then removes the download, because the
   QFX holds the full account number. `&&` never deletes after a refused import.
7. **The rollout is host-gated.** The mapping form is a configuration error for an older
   bob, so `bank.md` changes only after athena, apollo, and the Mac all run a bob that
   accepts it.
8. **Set category is a correction, not a walk.** It never advances.
9. **The waiting pill and queue-row jumps land at the payee start,** the same landing as
   the walk, so a Ctrl+Enter right after a jump starts filing.

## Contract — bob-cli

### Category configuration

`bank.categories` stays an ordered map. A value is either the existing scalar label or a
mapping:

```yaml
bank:
  categories:
    coffee: { label: ☕ Coffee, hotkey: c, group: Everyday }
    kids: { label: 🧸 Kids, hidden: true }
    transit: 🚆 Transit # scalar form stays legal
```

| Key      | Rule                                                                                                                                                                              |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `label`  | Required in the mapping form. Same emoji/label split as today.                                                                                                                    |
| `hotkey` | Optional. One character in `[a-z0-9]`. A YAML integer 0–9 reads as that digit; docs show it quoted. `q` is reserved (it closes the card).                                         |
| `group`  | Optional, non-empty text. It names the card's section.                                                                                                                            |
| `hidden` | Optional boolean, default false. A hidden category stays valid on rows, in `check`, and in totals. It is never offered in the card, in `[cat:: ` suggestions, or as a suggestion. |

- **Built-ins** keep their meaning. Undeclared built-ins get hotkeys `income i`,
  `refund r`, `transfer t`, `cash a`, group `Money moves`, and come after the declared
  categories. Declaring a built-in may override its label, hotkey, group, or hidden
  flag.
- **Assignment order:**
  1. Every explicit hotkey is claimed, on hidden categories too, so unhiding never
     reshuffles.
  2. Built-in defaults are claimed unless an explicit hotkey took them. A built-in whose
     default was taken derives one instead.
  3. Visible categories without a hotkey derive one in declared order. Derivation tries
     the first free character of the key, then the key's remaining characters, then the
     label's ASCII letters lowercased, then `1`–`9` and `0`.
  4. A visible category that finds no free key reports `hotkey: null`.
  5. Hidden categories without an explicit hotkey report `null`.
- **Errors** are `bank_config_invalid`, with file, line, and key path, like every other
  configuration error. They cover:
  - a duplicate explicit hotkey, reported at the later one;
  - a reserved or invalid hotkey;
  - a missing `label`;
  - a non-boolean `hidden`;
  - an empty `group`;
  - unknown mapping keys, reported with a nearest-key suggestion.

  The indentation-aware line walk gives block-form subkeys their own line. Flow-form
  subkeys report the category's line.

- **JSON** (`schema_version` stays 1, additive): each `categories[]` entry gains
  `hotkey` (string or null), `group` (string or null), and `hidden` (bool). Order is
  declared order, then undeclared built-ins. Clients render what bob reports.

### Row suggestions

Every row JSON (`list`, `report`, `import` `rows[]`) gains
`suggestion: {category, source} | null`, where `source` is `rule`, `learned`, or `bank`.

- **Identities of a row:**
  - the journal merchant key, when its `^bank-` ID has journal evidence (recomputed from
    the masked `desc` with the current cleaner, as `learn()` does today);
  - `merchant_key(payee)`, so typed rows and renamed imported rows participate.
- **Tiers, highest first** (the same order as import naming):
  1. **rule:** the first `bank.rules` entry with a `cat` whose `match` is a
     case-insensitive substring of the row's journal description, or of its payee when
     it has none. The read side only has the masked description, so a match text that
     contains a run of four or more digits cannot match at read time. Document this.
  2. **learned:** the most recent acknowledged row that carries a declared, non-hidden
     `[cat::]` and shares an identity with this row, never the row itself. Ties break by
     (date, note, line), as import does.
  3. **bank:** the Wells Fargo built-ins classified from the journal description, with
     no transaction type at read time.
- Undeclared and hidden categories are dropped, as is a suggestion for a row with no
  identity and no rule hit. A positive amount alone never implies income or refund.
- **One function.** Import's category tier calls the same suggestion function with the
  transaction's raw description, type, and merchant identity. An imported row's written
  category therefore equals what bob suggested at import time. The payee and link tiers
  of naming are unchanged, apart from cleaning.
- The index is built once per read over the scanned rows. Reads stay O(rows), and there
  is no full-vault scan.
- `import` keeps writing a category when a tier finds one. Waiting rows never teach.

### QFX cleaning

This work is in `naming.rs`, `identity.rs`, and `import_plan.rs`. Built-ins still
classify on the raw text and type before cleaning.

1. Strip the QFX tails `PURCHASE[ RETURN] [AUTHORIZED ON ]MM/DD … [CARD dddd|••••]` and
   `RECURRING PAYMENT [AUTHORIZED ON ]MM/DD …`, including the place and state between
   the date and the card. Keep the `AUTHORIZED ON` and CSV shapes working.
2. Strip processor order codes: a `*` followed by a short token that contains a digit.
   Keep `*` followed by words (a real sub-merchant behind a delivery app or processor
   stays distinct).
3. Bank-derived payees never keep a standalone run of four or more digits, a
   phone-number token, or an ACH/P2P account-holder name tail (`Surname, Given M.` after
   the reference digits). A counterparty name stays.
4. `merchant_key` inherits all of this through `clean_description`. Old journal events
   become comparable with no migration.
5. **Learned payees.** A learned payee is re-cleaned only when it still carries noise
   the new cleaner removes (a QFX tail, a card tail, a 4+ digit run). A name Bryan
   edited into a clean form passes through byte-identical.
6. **Default account.** Import passes no `[acct::]` when the row's account is the
   configured default, as the contract already says.
7. **Synthetic fixtures** in the real QFX shape only:
   - the same merchant on different days yields one key;
   - delivery sub-merchants stay distinct;
   - order codes collapse;
   - card digits and holder names never reach a written row;
   - an edited learned payee survives byte-identical;
   - the CSV and `AUTHORIZED ON` shapes keep their current results.

### Import's next step

When import writes waiting rows, the human `Next:` line points at the walk:
`Next: file the N waiting rows — Ctrl+Enter on each in bank/YYYY-MM.md (or bob bank to list them).`
Name the oldest month that has waiting rows, plus `and K more months` when several do.
JSON is unchanged.

## Contract — the Category card (bob-plugins)

### Ownership and APIs

```text
Ctrl+Enter ─► task-status-cycler bank guard (192)
               │ waiting row and ledger-tools api.bank v2 claimRowClose → true?
               ├─ yes ─► Category card (ledger-tools) → one editor transaction
               │          → advance + Ctrl+O jump → refresh → end-of-run notice
               └─ no  ─► today's plain [ ]↔[x] flip (ticked rows always go here)
```

- **bob-cli** owns categories, hotkeys, groups, hidden, suggestions, cleaning, learning,
  and every total.
- **bob-ledger-tools** owns the card, the write, the advance, the session, the notice,
  and **Bank: set category** (`bank-set-category`, no default hotkey). `api.bank`
  becomes version 2 and adds two members:
  - `claimRowClose(editor, {path, line, lineText}) → boolean` is synchronous and never
    throws. It returns true when ledger-tools takes the gesture: it opens the card,
    focuses an already-open card, or swallows the key inside the post-landing gesture
    lock. It returns false for anything that is not a waiting bank row.
  - `setCategory(editor)` opens the card in set mode.

  All v1 members stay.

- **task-status-cycler** keeps its standalone guard and recognizer and gains one
  feature-detected delegation used by both Ctrl+Enter entry points. Delegation applies
  only when the guard sees `kind: "row"` with status space. Ticked rows, stubs, invalid
  rows, and every other path behave exactly as today. A false or throwing claim falls
  back to `cyclerBankLocalFlip`. The card is never task completion.
- **bob-navigation-hotkeys** gains an additive, feature-detected `recordJump(from, to)`
  api member on the same jump history the review walk uses. Ledger-tools calls it when
  present, so Ctrl+O returns to the filed row.
- The walk is bank-local. It never calls `claimReviewWalkCtrlEnter` and never touches
  the `]s` walker, successors, freshness, recurrence, or the bank journal. It never
  spawns `bob bank ack`. The category is Bryan's edit in his editor, so the
  `docs/bank.md` ownership rule (bob's only row edits are `ack` and the ID append) still
  holds.

### Look

A compact card in the `.bob-key-card` family (decay card, Cmd+Y copy card, album schema
card), wearing the bank's magenta owner hue through the same `bankCardRootHue` token the
bank cards use. It renders well in light and dark themes, and state is never carried by
color alone.

```text
╭──────────────────────────────────────────────────────────────────╮
│ 🏦  $6.45  Corner Cafe                                Thu Nov 12 │
│     3 of 31 · November  ▰▰▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱▱           │
├──────────────────────────────────────────────────────────────────┤
│ EVERYDAY                                                         │
│ ┃C┃ ☕ Coffee      ↵   [V] 🏪 Convenience    [S] 🛍️ Shopping       │
│ [D] 🛵 Delivery        [E] 🍽️ Eating out     [G] 🛒 Groceries      │
│ [P] 👤 Personal        [H] 💊 Health         [O] 🐶 Dog            │
│ [F] 🎟️ Fun             [N] 🚆 Transit        [K] 🚗 Car            │
│ BILLS                                                            │
│ [1] 🏠 Housing   [2] 💡 Utilities   [3] 💻 Tech & AI  [4] 📺 Media │
│ [5] 📆 Loans     [6] 🤝 Household   [7] 🏛️ Taxes      [8] 🧾 Fees  │
│ MONEY MOVES                                                      │
│ [I] 💵 Income    [R] ↩️ Refund      [T] ↔️ Transfer   [A] 🏧 Cash out│
├──────────────────────────────────────────────────────────────────┤
│ ↵ ☕ Coffee · learned      ␣ no category      esc cancel          │
╰──────────────────────────────────────────────────────────────────╯
```

- **Header.** The amount comes first in tabular numerals: outflows neutral, inflows
  green with `+`. Then the payee and the right-aligned muted day. A flagged row shows
  its ⚑ chip and note. A non-default account shows a muted account chip. Set mode
  replaces the progress line with the row's state (`waiting` or `✓ acknowledged`) and
  shows no meter.
- **Tiles** sit in a 3-column grid (4 on wide panes) of monospace uppercase keycaps,
  emoji, and label, under quiet small-caps group headings. Groups appear in
  first-appearance order. Ungrouped categories form an untitled section at their first
  position. Hidden categories never render. The highlight is an accent ring with a soft
  fill and a trailing `↵`. Clicking a tile equals pressing its key.
- **Inflow.** For an inflow, the Money moves group renders first and spending groups dim
  (still selectable, same keys).
- **Footer** (`.bob-key-card-footer`). The ↵ hint names the highlight and its origin:
  - row category X, and the suggestion is also X from source s: `↵ ☕ Coffee · s`;
  - row category X, and the suggestion differs or is missing:
    `↵ ☕ Coffee · on the row`;
  - no row category, and the suggestion is Y: `↵ 🛒 Groceries · s`;
  - neither: `pick a category`.
- **Restricted mode** (no category data): the header, one muted line
  `Categories unavailable — <reason>`, and the footer `␣ acknowledge · esc cancel`.
- **Motion and access.**
  - On commit the card closes at once, the filed row gets a brief settle flash, and the
    landing row gets a soft focus flash (the yank-flash idiom). Both are off under
    `prefers-reduced-motion`.
  - The modal is `role="dialog"`. Tiles are options with `aria-keyshortcuts`.
  - A live region announces `Filed $6.45 as Coffee · next $12.90 waiting`.
  - There are no charts, budgets, or celebration animations.

### Keys

| Key                           | Close mode (Ctrl+Enter on a waiting row)                                            | Set mode (Bank: set category)         |
| ----------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------- |
| category hotkey (either case) | Tick + write or replace `[cat:: key]`, close, advance                               | Write or replace `[cat:: key]`, close |
| ↵ or a second Ctrl+Enter      | Accept the visible highlight; with none, a dim "pick a category" nudge and no write | Same, without ticking                 |
| Space                         | Tick with no category (removes an existing `[cat::]`), advance                      | Remove `[cat::]`                      |
| ← → ↑ ↓, Ctrl+N / Ctrl+P      | Move the highlight through visible tiles                                            | Same                                  |
| Esc, `q`, Ctrl+[              | Close; nothing written; cursor stays                                                | Same                                  |
| anything else                 | Ignored, with the copy card's dim nudge                                             | Same                                  |

- **Opening chord.** Copy the decay card's `openingChordKeys` swallow, so the held or
  repeating Enter that opened the card never accepts. A second Ctrl+Enter counts only
  after the opening Enter's keyup.
- **Gesture lock.** For about 300 ms after a landing, `claimRowClose` swallows
  Ctrl+Enter. A double press never opens a card on the landing row and never falls back
  to a flip.
- **One card at a time.** A claim while a card is open focuses that card.

### Write rules

- **At open,** capture:
  - the view, file path, and line;
  - the exact line text;
  - the `^bank-` ID, if any.
- **At commit,** re-resolve the row in that same editor:
  - by its ID (it must be unique);
  - otherwise by the captured line, whose text must be byte-identical.

  Refuse with `Bank: the row changed — nothing written` (no write, no advance) when:
  - the text changed;
  - the file changed;
  - the row is no longer a row (or, in close mode, no longer waiting);
  - the line holds two or more `[cat::` fields.

- **One editor transaction** (`editor.transaction` with every change), so one undo
  reverts tick and category together:
  - The status byte goes from ` ` to `x`, in close mode only.
  - An existing single `[cat:: …]` value is replaced in place.
  - A missing one is inserted as ` [cat:: key]` right after the payee span, the
    canonical first tail slot, before any `[acct::]`, link, flag, note, or ID.
  - Space removes ` [cat:: …]` with its one leading space.
- **Reuse, don't re-parse.** Payee and field spans come from the existing
  `bankScanSource` / `bankTokenizeLine` recognizers. There is no new grammar. Every byte
  the plugin does not own is preserved.
- **Never block acknowledging.** Space in restricted mode still ticks.

### Advance, session, and refresh

- **Target.** Only after a successful commit, find the next waiting row in document
  order after the filed row:
  - a valid top-level checkbox row under a day heading, outside fences;
  - flagged waiting rows count;
  - ticked rows, stubs, invalid rows, children, prose, and plain checklist lines are
    skipped.

  If none is below, wrap once to the first waiting row above. If none exists at all,
  stay and show the end-of-run notice. Never open another file.

- **Landing.**
  - The cursor goes to the payee start (Vim `cw` renames at once).
  - The row scrolls into view, centered when it was off-screen.
  - `recordJump(filed, landing)` is called when nav provides it.
- **Session** (in memory, per note path).
  - It starts at the first commit in a note with no active session.
  - It ends at the end-of-run notice, after 15 idle minutes, or when the walk moves to
    another note.
  - It counts filed, accepted (↵), picked (key or click), and no-category (Space), plus
    the elapsed time.
  - Progress is `N of M`, where N is filed + 1 and M is filed + still waiting in the
    live note.
- **Refresh.**
  - After each commit, `await view.save()`, then force one single-flight fetch of
    `bob bank list -a -m <month> -f json`.
  - Opening the next card awaits that flight for at most 800 ms, then uses the freshest
    data.
  - Match the card's row to bob's JSON row by `^bank-` ID, else by path and line with
    equal cents and payee. With no match, the card shows no suggestion but still works.

### End of run

Show a notice of about 8 seconds, built from bob's JSON. bob owns every number:

```text
🏦 November filed — 31 rows in 1m 48s
WF Checking ✓ balanced · Nov 13   ·   2 waiting in October →
24 accepted · 5 picked · 2 without a category
```

- Line 2 uses each importable account's `status` and hint. It never says caught up while
  an account is `stale`, `gap`, or `delta`; it shows `⚠ import 12d ago` or `Δ $12.40`
  instead.
- `N waiting in <Month> →` comes from `bob bank list -f json` across months. A click
  opens that month at its first waiting row.
- When bob is unreachable, only line 1 shows.

### Elsewhere in the plugins

- `[cat:: ` typing suggestions skip hidden categories.
- The month card's waiting pill and the queue card's row links land at the payee start.
- Old bob: when `categories[]` has no `hotkey` field, tiles are reachable by arrows and
  ↵, and the footer adds a muted `update bob for one-key filing`.

## Contract — the vault

### The weekly chore (`gtd_daily.md`)

Edit the Monday **Bank import** chore in place, keeping its status, `[created::]`,
`[scheduled::]`, and recurrence. The shape is below; adapt the wording through house
style.

```markdown
- [?] #task #gtd #pre #browser **Bank**:
  [Wells Fargo › Download Account Activity](https://connect.secure.wellsfargo.com/auth/login/present?origin=cob&loginMode=jukePassword&serviceType=accountActivityDownload&LOB=CONS)
  → Checking · Quicken (QFX) · default dates (≥ 14 days) · on the Mac:
  `bob bank import ~/Downloads/*.qfx && rm ~/Downloads/*.qfx` · in [[bank]], Ctrl+Enter
  each waiting row · ✓ balanced? · one line in last week's note: what surprised me?
  [repeat:: every week on Monday when done] [created::2026-11-11]
  [scheduled::2026-11-16]
```

The walk step reads "Ctrl+Enter each waiting row" on purpose. It is true before the card
ships (Ctrl+Enter ticks) and after (Ctrl+Enter opens the card), and it never tells Bryan
to press a letter that Vim's normal mode would treat as an edit before the card exists.

Cancel exactly the active daily **Acknowledge bank** occurrence, using bob's own cancel
gesture where one exists. Otherwise make a guarded edit of only its status byte plus
`[cancelled:: <today>]`. Confirm that the cancel spawns no new occurrence.

### Bryan's categories (`bank.md`)

The list is grounded in the full journal, with coarse bands only. The layout puts
letters on what Bryan decides, digits on bills that bob usually already knows, and
`i r t a` on the built-ins.

| Key           | Label          | Hotkey | Group    | Band                                                  |
| ------------- | -------------- | ------ | -------- | ----------------------------------------------------- |
| `coffee`      | ☕ Coffee      | c      | Everyday | several a week                                        |
| `convenience` | 🏪 Convenience | v      | Everyday | near daily (convenience-store and fuel-stop runs)     |
| `shopping`    | 🛍️ Shopping    | s      | Everyday | several a week                                        |
| `delivery`    | 🛵 Delivery    | d      | Everyday | about weekly (kept apart from eating out)             |
| `dining`      | 🍽️ Eating out  | e      | Everyday | about weekly                                          |
| `groceries`   | 🛒 Groceries   | g      | Everyday | about weekly                                          |
| `personal`    | 👤 Personal    | p      | Everyday | about weekly (a recurring cluster with no other home) |
| `health`      | 💊 Health      | h      | Everyday | about weekly                                          |
| `dog`         | 🐶 Dog         | o      | Everyday | occasional                                            |
| `fun`         | 🎟️ Fun         | f      | Everyday | occasional                                            |
| `transit`     | 🚆 Transit     | n      | Everyday | about weekly                                          |
| `car`         | 🚗 Car         | k      | Everyday | monthly                                               |
| `home`        | 🏠 Housing     | 1      | Bills    | monthly                                               |
| `utilities`   | 💡 Utilities   | 2      | Bills    | monthly                                               |
| `tech`        | 💻 Tech & AI   | 3      | Bills    | monthly and more                                      |
| `media`       | 📺 Media       | 4      | Bills    | monthly                                               |
| `loans`       | 📆 Loans       | 5      | Bills    | monthly                                               |
| `household`   | 🤝 Household   | 6      | Bills    | monthly (spending, never `transfer`)                  |
| `taxes`       | 🏛️ Taxes       | 7      | Bills    | rare                                                  |
| `fees`        | 🧾 Fees        | 8      | Bills    | about weekly                                          |
| `kids`        | 🧸 Kids        | (none) |          | hidden                                                |

> [!decision] split_subscriptions = no Keep `subscriptions: 🔁 Subscriptions` (hotkey 3,
> Bills) in place of `tech` and `media`, and renumber the remaining Bills digits 4–8.

Built-ins stay undeclared (`i` Income, `r` Refund, `t` Transfer, `a` Cash out, under
Money moves). Remove the `VENMO → transfer` rule, which has zero journal matches and
would misfile shared payments. Add no other rules: learning covers repeats after one
filing.

## Privacy (every phase)

- No merchant names, amounts, people, places, account digits, or raw descriptions in any
  plan, bead note, phase report, commit message, test fixture, or research file. Report
  counts and percentages only.
- Fixtures are synthetic and shaped like real QFX.
- Reading the vault and journal for aggregate analysis, and editing Bryan's own bank
  notes, is fine. Never echo their contents outward. Bank data still reaches no
  agent-facing surface: period briefs, Stats, and the Mac app.

## Phases

```text
weekly-chore ─────────────────────────────────────────────────────────────┐
qfx-cleaning ─► category-contract ─┬─► vault-rollout ──────────┐          │
                                   └─► card-model ─► card-ui ──┴─► ctrl-enter-walk ─► wrapup
```

Every bob-cli phase:

- runs `just check` and `just docs-check` (plus `just docs-gen` when help or generated
  docs move);
- uses Conventional Commits and never edits the `Cargo.toml` version or `CHANGELOG.md`;
- updates `docs/bank.md` and `docs/reference/json-outputs.md` for what it changes;
- adds no new subcommand or option.

Every bob-plugins phase:

- opens the repo with `/sase_repo` and reads its `AGENTS.md`;
- edits `src/` fragments only, registers new fragments in `src/fragments.json`, and
  keeps each hand-edited fragment at or below 1000 lines (`241k-plugin-bank-actions.js`
  is near its cap, so new code goes in new fragments);
- runs `npm run build` and `npm test`, then `bob plugins sync`.

### weekly-chore

- Run `bob vault-sync` first so the vault is current. Re-fetch the Wells Fargo link
  without credentials and confirm it still resolves to the sign-on page for the
  account-activity download.
- Edit the Bank import chore per [the weekly chore](#the-weekly-chore-gtd_dailymd), and
  cancel the active daily occurrence. Touch no completed occurrence, block ID, or LINKED
  FROM line.
- **Gate:**
  - exactly one open bank chore remains in `gtd_daily.md`;
  - its scheduled date stays the Monday it already carries;
  - `bob plan` with `BOB_NOW` set to that Monday morning (or the PRE view the morning
    review uses) lists it;
  - the daily chore appears in no future PRE list;
  - the open/done task counts move by exactly the expected amounts.
- Let vault-sync carry the change, and report counts only.

### qfx-cleaning

- Implement [QFX cleaning](#qfx-cleaning) and the default-account `[acct::]` fix, with
  the synthetic fixture matrix.
- Measure the learned tier on the real journal in memory, then report it as percentages
  only:
  - **before and after:** the share of transactions whose merchant key matches an
    earlier transaction's key;
  - **after only:** the share of card-tail, order-code, and holder-name noise left in
    cleaned payees (it should be 0).
- Update `docs/bank.md` "Naming" and "Hashed IDs" for the new stages, the learned-payee
  rule, and the `[acct::]` behavior.

### category-contract

- Implement [category configuration](#category-configuration),
  [row suggestions](#row-suggestions), and [import's next step](#imports-next-step).
  Import's category tier must call the shared suggestion function.
- **Tests:**
  - scalar and mapping forms, block and flow;
  - derivation order and stability;
  - built-in overrides;
  - each error with its line and key path;
  - hidden behavior in `check`, totals, and suggestions;
  - each suggestion tier and its precedence;
  - in-session learning (acknowledging a categorized row flips the next same-merchant
    row's suggestion to `learned`);
  - a typed row learning from a renamed imported row;
  - import-written category = read-time suggestion;
  - the JSON fields on `list`, `report`, and `import`.
- Docs: the `bank.md` key table, a mapping-form example, hotkey assignment, the JSON
  fields, and the rule limitation for 4+ digit match text.

### vault-rollout

- **Host gate first.**
  - Install the landed bob on athena (`just install` from an up-to-date bob-cli
    checkout) and on apollo.
  - Check the Mac over the tailnet (`ssh mac`, best effort; it is offline unless its lid
    is open) and update it the same way.
  - On each host, prove acceptance by running `bob bank check -b <scratch copy>` against
    a scratch vault that holds the new `bank.md`.
  - If the Mac cannot be reached or updated, ask Bryan with `/sase_questions` before
    changing `bank.md`.
- Write [Bryan's categories](#bryans-categories-bankmd) in the mapping form, remove the
  VENMO rule, and update the hub prose: weekly, the Category card, and that typed rows
  still start ticked.
- **Seed rows.**
  - Replace each of the seven rows' payee with the cleaned merchant name the new cleaner
    yields for its journaled description.
  - Drop `[acct:: checking]`.
  - Keep each amount, checkbox, and `^bank-` ID byte-identical.

> [!decision] prefill_seed_rows Also add each seed row's `[cat:: key]` from the
> taxonomy, so the learned tier starts warm for the first Monday walk.

- **Gate:**
  - `bob bank check` reports 0 errors and no new warnings;
  - the account status is unchanged;
  - `bob bank list -a -f json` reports a hotkey on every visible category, `kids`
    hidden, and the expected group order;
  - no row carries a retired key (if one does, hide that key instead of deleting it).

### card-model

- **New pure fragments** in bob-ledger-tools, each exported for tests through
  `350-exports.js`:
  - decoding of `hotkey`, `group`, `hidden`, and `suggestion`;
  - the card view model: groups, tiles, inflow ordering, highlight and footer rules,
    restricted and old-bob modes;
  - the key table, including the opening-chord and lock rules as pure state;
  - the write planner (close and set modes, replace/insert/remove, every refusal
    reason);
  - the next-waiting-row finder;
  - the session counter;
  - the end-of-run summary model.
- **Fixture vectors** cover:
  - typed and imported rows;
  - notes, flags, links, `[acct::]`, other block IDs, and CRLF;
  - fenced lookalikes;
  - wrap-around;
  - flagged waiting rows;
  - duplicate `[cat::` fields;
  - an inflow;
  - each footer variant.
- Regenerate `plugins/bob-ledger-tools/scripts/fixtures/bank/` JSON from the new bob
  against a synthetic vault.

### card-ui

- **The card.** Build the modal from the card-model view model, with CSS in the key-card
  family and the bank magenta hue.
- **API.** Wire `api.bank` v2 (`claimRowClose`, `setCategory`) and the
  `bank-set-category` command.
- **The walk.**
  - commit through one editor transaction;
  - the landing, flashes, and live region;
  - the gesture lock;
  - the save → refresh flight and the ≤ 800 ms await;
  - the session;
  - the end-of-run notice and its month link;
  - the feature-detected `recordJump` call.
- **Smaller changes.** `[cat:: ` suggestions hide hidden categories, and the pill and
  queue-row landings go to the payee start.
- **Tests** run through the modal and ledger-tools harnesses and cover:
  - each key in each mode;
  - opening-chord and lock swallowing;
  - refusal on a changed row;
  - restricted-mode upgrade before the first key;
  - no advance on Esc, refusal, or set mode;
  - one undo step reverting tick and category.

### ctrl-enter-walk

- **task-status-cycler.** Add the delegation to both Ctrl+Enter entry points, with the
  fallback and safety rules above. Extend
  `scripts/test-task-status-cycler-bank-guard.cjs`:
  - ledger-tools absent, v1, or v2;
  - claim true, false, or throwing;
  - a ticked row never delegates;
  - stubs and invalid rows are unchanged;
  - no task metadata, ever.
- **bob-navigation-hotkeys.** Add the additive `recordJump(from, to)` api member, with a
  test that Ctrl+O returns to the filed row.
- **Real Obsidian.** Extend `scripts/obsidian-bank-ui-e2e.cjs` with this scenario on
  synthetic data:
  1. import a fixture;
  2. `Ctrl+Enter`, `c` files the row and lands on the next;
  3. `Ctrl+Enter`, `Ctrl+Enter` accepts a learned suggestion;
  4. Space;
  5. Esc writes nothing;
  6. Ctrl+Enter on a ticked row unticks;
  7. the end-of-run notice;
  8. Ctrl+Z reverts one filing.
- **Docs.** Update `docs/bob-ledger-tools.md`, `docs/task-status-cycler.md`, and
  `docs/develop/plugin-apis.md` (bank v2 and nav `recordJump`), then run
  `npm run docs:check`.
- **Deploy.** Run `bob plugins sync` on athena, and on the Mac over the tailnet when it
  is reachable. Confirm in the real GUI on the Mac when possible: the card renders in
  both themes and a walk over the real November note behaves.

### wrapup

- **Guide.** Rewrite `docs/guides/acknowledge-bank-transactions.md` for the weekly walk:
  - Monday's download and import;
  - filing with the card;
  - the optional mid-week glance (typing rows still works, and the daily chip stays);
  - catching up after a lapse (bulk ack stays tick-only, and set category fixes rows
    later);
  - editing categories (hotkeys, groups, hidden);
  - the status chips;
  - if it goes wrong.

  Keep the URL. Add the card to `docs/bank.md` "Ownership" (the category is Bryan's
  editor edit). Run `just docs-check`.

- **Real-vault proof, counts only:**
  - `bob bank check` is clean;
  - `list -a -f json` reports hotkeys and how many waiting rows carry a suggestion;
  - the chore is in place;
  - the deployed plugins report bank v2.

> [!decision] memory_bank_row Add one sentence to `glossary:bank-row`: Ctrl+Enter on a
> waiting row opens the Category card, which ticks it and files `[cat:: key]` in one
> gesture, then lands on the next waiting row. Use `/sase_memory_write` and
> `sase memory init`.

> [!decision] memory_walk_record Add `decisions:bank-close-files-a-category` through
> `/sase_memory_write`. It records: the card always opens on waiting rows; ticked rows
> untick; keys never change meaning; bulk paths stay tick-only; the walk is note-local
> and never opens another file; suggestions are bob's at read time. Cite this plan and
> the research report.

> [!decision] kv_snooze = now Unsnooze `bob-cli-kv` when the epic lands and add a bead
> note citing this plan. With `hold`, add only a note that the walk trial informs it on
> 2026-12-14.

- Record a `PROPOSED FOLLOW-UP:` for the 2–4 week trial review. It covers:
  - the share of acknowledged rows with a category;
  - the no-category rate and the correction rate (set-category use, from Bryan);
  - weekly import continuity from the journal's `bank.import` events;
  - whether most filings were ↵ (if so, consider skip-when-suggested);
  - whether to show suggestion chips on waiting rows.
