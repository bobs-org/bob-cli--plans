---
tier: epic
title: Bank — acknowledge every transaction daily, prove it weekly
goal: 'Bryan restarts his daily bank habit right away. Every posted transaction and
  deposit becomes one checkbox row in a monthly bank note under a new `bank` sub-area
  of `cash`. Typing a row acknowledges it. A weekly `bob bank import` of a Wells Fargo
  QFX file writes, unchecked, only the rows he missed, and a balance proof shows that
  nothing was skipped. Every total is computed when it is drawn (terminal, month card,
  queue card, daily chip) and never written into notes. A missed week costs one sitting
  to catch up, not a retyping spiral.

  '
decisions:
  accounts:
    ask: Which accounts should bank.md track from day one?
    choices:
      checking: Wells Fargo checking only; a card is added later with one bank.md
        entry
      checking_card: Also a credit card you spend on, with its card payments marked
        transfer
    default: checking
    why: The vault shows no card in use, and the old @BANK habit was checking-only
    answer: checking
  subscriptions_scope:
    ask: Which recurring charges should the follow-up subscriptions album hold?
    choices:
      discretionary: Only keep-or-cancel charges; mortgage, loans, and insurance stay
        categories
      all_recurring: Every recurring charge, so the album also shows a fixed monthly
        burn
    default: discretionary
    why: Obligations aren't keep-or-cancel choices, and auditing those choices is
      the album's job
    answer: discretionary
  memory_bank_glossary:
    ask: Should the wrap-up add Bank Month Note and Bank Row glossary strands?
    memory:
    - glossary:bank-month-note
    - glossary:bank-row
    default: false
    answer: true
  memory_bank_decision:
    ask: Should the wrap-up add the bank-rows-not-records decision record?
    memory:
    - decisions:bank-rows-not-records
    default: false
    answer: false
phases:
- id: bank-restart
  title: Restart the habit in the vault with no code
  depends_on: []
  size: small
  description: 'bank-restart: create `bank.md` under `cash` with its `bank:` config,
    the `bank_month` type note, and this month''s bank month note in the final template.
    Capture the daily PRE chore, retire the decaying `^daily-bank-records-again` as
    superseded, and gate on link and task counts.

    '
- id: bank-grammar
  title: The bank module, row grammar, and bob bank check
  depends_on: []
  size: medium
  description: 'bank-grammar: add `src/native/bank/` with the `bank:` config parser
    (key-path diagnostics), the row and day-heading grammar, the canonical renderers
    and insertion planners, and the month-note scanner. Ship the read-only `bob bank
    check` lint in the Integrations section, and write `docs/bank.md` as the contract.

    '
- id: bank-cli
  title: Read the bank — list, report, ensure, and ack
  depends_on:
  - bank-grammar
  size: medium
  description: 'bank-cli: add the bare `bob bank` waiting queue, `report` with the
    totals semantics, and two capture-style guarded writes: `ensure` (month note and
    day heading) and `ack` (tick shown rows). Give each beautiful human output and
    `schema_version: 1` JSON, with account status stubbed as `unverified`.

    '
- id: bank-import
  title: Weekly QFX import that writes only what you missed
  depends_on:
  - bank-cli
  size: large
  description: 'bank-import: parse Wells Fargo QFX (OFX 1.x SGML and 2.x XML) plus
    one CSV profile; resolve accounts without numbers; journal hashed identities per
    host; match typed rows; name rows from rules, learned payees, and restaurant records;
    write unchecked rows in one guarded, committed transaction with an exact dry run;
    add the weekly PRE chore.

    '
- id: bank-balance
  title: Prove every transaction — coverage, presence, balance
  depends_on:
  - bank-import
  size: medium
  description: 'bank-balance: derive each account''s anchor from the bank''s own numbers,
    prove coverage, presence, and balance, report possible twins, unmatched typed
    rows, and edited amounts, and fill account status in check, list, report, and
    import. Add `import -F` to restore deleted rows deliberately.

    '
- id: bank-obsidian
  title: Obsidian bank cards, styled rows, and the daily chip
  depends_on:
  - bank-cli
  size: large
  description: 'bank-obsidian: in bob-ledger-tools, draw the `bob-bank` month and
    queue cards, style bank rows and day headings, add the quiet daily-note chip,
    the open-today and acknowledge commands, Enter continuation with `- [x] $`, and
    category suggestions. Guard bank rows against task gestures, then `bob plugins
    sync`.

    '
- id: bank-wrapup
  title: End-to-end proof, the guide, docs audit, memory, and follow-ups
  depends_on:
  - bank-restart
  - bank-balance
  - bank-obsidian
  size: medium
  description: 'bank-wrapup: walk every bank gesture end to end in a temporary vault,
    write the how-to guide, audit every bank doc against shipped behaviour, check
    the real vault, apply only the accepted memory decisions, and propose the subscriptions
    album (blocked on bob-cli-k3) and the later views as follow-ups.'
proposed_by: bbugyi200.athena.1be
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 11:24:14
status: wip
bead_id: bob-cli-kd
---

- **PROMPT:** [prompts/202611/bank_daily_acknowledgment.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_daily_acknowledgment.md)
- **BEAD:** [bob-cli-kd](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kd/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                    | Why                                                                                   |
| ------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| derives-from | [research:202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md][1] | The consolidated bank research report; Bryan accepted every requirement it recommends |

[1]:
  https://github.com/bobs-org/bob-cli--research/blob/main/202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md

<!-- sase:links:end -->

# Plan: Bank — acknowledge every transaction daily, prove it weekly

## Why

Bryan wants to acknowledge every transaction and deposit in his own bank account. Doing
so keeps him aware of his spending, and the result should be a well-structured,
good-looking view of that data. The origin is `bob.md ^bank-records` ("Use records to
start tracking bank transactions again!") together with the decaying
`cash.md ^daily-bank-records-again`.

This epic implements the consolidated research report
`research:202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md`. Bryan agreed
with every requirement it recommends. Every phase worker reads the sections of that
report relevant to its phase with
`sase artifact read research:202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md "<why>"`.
The report's verified facts are the starting point. Re-verify any number your phase acts
on, because the vault moves daily.

Evidence that shapes the design (from the report, re-checked while planning):

- **It is a proven habit.** About 2,400 hand-typed lines ran from 2023-11 to 2025-12:
  `## expenses`, then `## @BANK`, in `_habit.md` notes. It died in a holiday "Skipping
  until …" spiral. After that, the revival task went down the P-ladder (P0 → P1 → P2),
  and it is now scheduled for 2026-12-01. The data model was never the problem. The
  cadence and the cost of catching up were.
- **Volume.** About 115 lines a month, a median of 6 a day, and a maximum of 34. That is
  about 1,400 outflows a year before deposits.
- **The album contract rejects one record per transaction.** The `bob-cli-k3` albums
  epic (`plan:202611/album_and_record_notes.md`) defines:
  - an index that lists and rewrites every record on every write;
  - filenames that come from titles, with collisions refused (`7-Eleven` appears 277
    times);
  - `record_checkbox` as an error;
  - a record as "one durable thing".

  Transactions break all four rules. Subscriptions pass them.

- **The ritual already has a home.** `gtd_daily.md` holds the `#gtd #pre` chores that
  repeat `every day when done`.

### Settled requirements (from the research, accepted by Bryan)

- **ADJ-1.** The container is a `bank` sub-area with monthly notes holding checkbox
  rows, not an album of transactions. The album goes to **subscriptions**, under `bank`.
- **ADJ-2.** Only the subscriptions album waits for `bob-cli-k3`. The ritual starts now
  with no code.
- **ADJ-3.** An acknowledged row is a ticked row, and typed rows start ticked. A weekly
  import plus a balance check proves "every". Only posted transactions count. Deposits
  are rows. Transfers are marked so they never count as spending.
- **ADJ-4.** "Every day" means a daily chance to look, not a duty. It is a `#gtd #pre`
  repeating chore, and `^daily-bank-records-again` is retired as superseded. After a
  gap, import fills in what was missed, and a whole day can be acknowledged at once.
- **ADJ-5.** Categories are optional and learned from Bryan's own edits. Only
  `transfer`, `income`, `refund`, and `cash` carry meaning for totals.
- **ADJ-6.** Coverage is explicit. v1 covers Wells Fargo checking plus any card Bryan
  actually spends on (see the `accounts` decision). An ATM withdrawal is "cash out,
  destination unknown".
- **ADJ-7.** The views are drawn, never written: the month card, the queue card, the
  daily chip, and the terminal. Stats comes later.
- **ADJ-8.** Out of v1: budgets, envelopes, splits, multiple currencies, investments,
  net worth, pending transactions, and categorization by a cloud model.

### Design calls this plan makes beyond the research

1. **Names that don't collide.** In this vault, "ledger" already means the Pomodoro
   ledger (and `bob-ledger-tools`), and "inbox" means GTD inbox files. So this plan uses
   **bank month note** and **bank row**, and calls unchecked rows **waiting** ("3 to
   acknowledge"). The waiting view is `view: queue`.
2. **The configuration lives under one nested `bank:` key** in `bank.md`, the way album
   schemas live under `record:`. The area's top-level frontmatter stays ordinary, and
   the plugin hides the single property row.
3. **The bank's own numbers set the balance anchor.** Each account configures only a
   `start:` date. Bryan never types a balance. The first import that covers `start`
   derives the opening balance, and every later import proves it.
4. **The grammar is lenient to read and canonical to write.** `$` is required, which
   rules out `- [ ] 3 things` reading as $3. Bold around the amount is accepted but
   never needed. bob writes `- [ ] $61.20 ShopRite`, and the plugin draws the emphasis
   (tabular, right-aligned). The research's bold amount stays valid input, so rows typed
   in that style keep parsing.
5. **Typing is one keystroke per row.** In bank month notes, Enter after a row continues
   with `- [x] $`, because typed rows are acknowledged as they are written. `⚑` and `—`
   have ASCII spellings (`!!` and `--`).
6. **Interactive gestures write like capture; import writes like maintenance.** `ensure`
   and `ack` are guarded writes with no commit, which vault-sync carries. `import` is
   one guarded transaction with one scoped commit, `bank: import N rows (<account>)`.
7. **"Seen" means journaled or present in the vault.** A crash between writing the rows
   and appending the journal can never duplicate a row. A deleted row stays deleted, and
   `import -F` restores it on purpose (the `gcal sync -F` precedent).
8. **The journal stores no bank identifiers.** It keeps only a sha256 of
   `(account alias, bank transaction id)`, plus masked descriptions. Accounts are
   matched by OFX account type (checking, savings, credit), never by number. `-a`
   settles ambiguity.
9. **Ack touches only what you saw.** The plugin passes the IDs it drew, so a row that
   lands after the card was painted stays waiting.
10. **The plugin draws prev/next month navigation** in the month card instead of writing
    links that would dangle until next month exists.
11. **No "typical" comparisons in v1.** With no history they would be noise. They are a
    proposed follow-up after the trial, once there are at least two months of rows.
12. **A `cash` account kind** lets typed cash purchases say where ATM cash went without
    counting the same money twice.
13. **The weekly import rides Monday's weekly review**, not a separate Sunday ritual,
    and asks the "what surprised me?" question there.

## Vocabulary

- **Bank hub:** `bank.md`, a sub-area (`type: "[[area]]"`, `parent: "[[cash]]"`). It
  holds the `bank:` configuration, the queue card, and bank tasks such as disputes and
  cancellations. `@bank` stays an ordinary task route.
- **Bank month note:** `bank/YYYY-MM.md`, with `type: "[[bank_month]]"`,
  `parent: "[[bank]]"`, and `month: YYYY-MM`. It reads like a statement: one `##`
  heading per day, oldest first, with bank rows under each.
- **Bank row:** one posted transaction or deposit, written as a top-level checkbox line
  with a `$` amount under a day heading. It never carries `#task`, so it is never a
  task.
- **Waiting:** a bank row whose box is `[ ]`. **Acknowledged:** `[x]`.
- **Typed row:** a row with no `^bank-…` ID; Bryan wrote it. **Imported row:** written
  by `bob bank import`. **Matched row:** a typed row that an import confirmed by
  appending its ID.
- **Journal:** `.bob/bank/<host>.jsonl`, the append-only per-host record of every bank
  transaction bob has seen. It exists only to prevent duplicates and to prove coverage.
- Not to be confused with the **Pomodoro ledger**, an **inbox file**, an **album**, or a
  **record note**.

## The contract

### Files and membership

| Path                | Kind            | Membership rule                                                                    |
| ------------------- | --------------- | ---------------------------------------------------------------------------------- |
| `bank.md`           | Bank hub        | Fixed vault-root stem; its frontmatter must hold a `bank:` mapping. One per vault. |
| `bank/YYYY-MM.md`   | Bank month note | The filename is authoritative. Other files in `bank/` are ignored.                 |
| `bank_month.md`     | Type note       | An ordinary `[[type]]` child, in the house style of `notebook.md`.                 |
| `.bob/bank/*.jsonl` | Journal         | A `pref_journal` namespace (`src/native/pref_journal/`). bob is the only writer.   |

- With no `bank.md`, every `bob bank` command prints a teaching empty state. It names
  `docs/bank.md` and gives a paste-ready minimal `bank:` block, then exits 0. Only
  `check` exits 1, with `bank_hub_missing`, and only when `bank/` exists without
  `bank.md`.
- The scan reads `bank.md` and `bank/*.md` only, about 12 small files a year. It needs
  no cache and never scans the whole vault.
- Bank rows must stay invisible to every task scanner: `bob plan`, `bob ready`,
  `bob task archive`, `bob task reconcile`, freshness, status hooks, and the Tasks
  plugin's `#task` global filter. `bank-grammar` pins this with one integration test.

### `bank.md` and its configuration

````markdown
---
parent: "[[cash]]"
type: "[[area]]"
created: 2026-11-12T07:30:00-05:00
bank:
  accounts:
    checking:
      label: WF Checking
      kind: checking
      start: 2026-11-12
  categories:
    groceries: 🛒 Groceries
    dining: 🍽️ Dining
    coffee: ☕ Coffee
    transit: 🚆 Transit
    car: 🚗 Car
    home: 🏠 Home
    utilities: 💡 Utilities
    dog: 🐶 Dog
    kids: 🧸 Kids
    health: 💊 Health
    shopping: 🛍️ Shopping
    subscriptions: 🔁 Subscriptions
    fun: 🎟️ Fun
    fees: 🧾 Fees
  rules:
    - match: VENMO
      cat: transfer
---

# 🏦 Bank

Every transaction and deposit in our accounts, acknowledged daily. Each month has one
note under `bank/`. Under each day, type what posted, ticked:
`- [x] $13.71 Smashburger [cat:: dining]`. Use `+$` for money in, `⚑` (or `!!`) for
anything to look into, and `—` (or `--`) for a note. Money tasks like disputes and
cancellations live here too.

```bob-bank
view: queue
```
````

| Key                 | Rule                                                                                                                                                                                                     |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bank.accounts`     | Required, non-empty, ordered map. The key is an alias matching `[a-z][a-z0-9_]*`. **The first account is the default** for rows without `[acct::]`.                                                      |
| `…<alias>.label`    | Optional display name. Defaults to the alias, humanized.                                                                                                                                                 |
| `…<alias>.kind`     | Required. `checking`, `savings`, `credit`, or `cash`. A `cash` account is never imported or balance-checked, and its rows never count in Out (the ATM withdrawal already did).                           |
| `…<alias>.start`    | Required for every kind except `cash`. `YYYY-MM-DD`. Import never writes a transaction posted before it, and the balance anchor is taken at it.                                                          |
| `bank.categories`   | Optional ordered map from `key` to `"<emoji> <Label>"`. Labels split into icon and text by the album choice-label rule. The declared order is the order for chips and suggestions.                       |
| built-in categories | `income` (💵 Income), `refund` (↩️ Refund), `transfer` (↔️ Transfer), and `cash` (🏧 Cash out) always exist. Declaring one overrides only its label; its meaning never changes.                          |
| `bank.rules`        | Optional list of `{match, payee?, cat?, link?}`. `match` is a case-insensitive substring of the cleaned bank description. The first matching rule wins, and rules apply only to rows that import writes. |
| `bank.stale_after`  | Optional, in days (default `10`). An importable account whose last import is older than this is `stale`.                                                                                                 |

- Every configuration error names the file, the 1-based line, and the key path, for
  example
  `bank.md:9 bank.accounts.checking.kind: expected checking, savings, credit, or cash`.
  Reuse the indentation-aware frontmatter line walk the albums parser uses. Unknown keys
  inside `bank:` are errors with suggestions. Unknown top-level keys are tolerated.
- No account number, routing number, credential, or balance is ever stored.

### The bank month note

The canonical template is everything through the first day heading. `bob bank ensure` or
`bob bank import` writes it once when the note is missing, and `bank-restart` writes the
same bytes. The rows below the heading illustrate the grammar:

````markdown
---
parent: "[[bank]]"
type: "[[bank_month]]"
month: 2026-11
created: 2026-11-12T07:30:00-05:00
---

# 🏦 November 2026

```bob-bank
view: month
```

## Thu Nov 12

- [x] $13.71 Smashburger [cat:: dining] [[eat/smashburger|🍽️]] ^bank-8f3a2c1d
- [x] $4.25 7-Eleven — coffee on the walk
- [ ] $61.20 ShopRite [cat:: groceries] ^bank-2b91c0e4
- [ ] +$1,250.00 Payroll [cat:: income] ^bank-1c0e77aa
- [x] $63.50 ATM withdrawal [cat:: cash] ⚑ fee again? ^bank-5e0d9a17
````

**Day headings.**

- **Canonical form:** `## <Wkd> <Mon> <D>` in English abbreviations (`## Thu Nov 12`).
- **Also read:** `## Nov 12`, `## 2026-11-12`, and `## 2026-11-12 Thu`. The note's
  `month` supplies the year and month.
- **Not a day:** any other `##` heading. Bryan's own sections, such as `## Notes`, are
  allowed.
- **Writers:**
  - insert a missing day heading in date order among the existing day headings, or after
    the fence when there are none;
  - separate sections with one blank line;
  - put rows right under the heading, after one blank line.
- **Empty day heading.** One with no rows is legal and means "I looked; nothing posted."
  The plugin draws it as a muted "nothing posted".

### The bank row grammar

A **row** is a top-level list item (no indent) with a checkbox, under a day heading,
whose body starts with an amount. Indented children of a row are free notes. A checkbox
line whose body does not start with an amount is a plain checklist line, and bank
features ignore it.

```text
- [x] $13.71 Smashburger [cat:: dining] [acct:: card] [[eat/smashburger|🍽️]] ⚑ — with Kelly ^bank-8f3a2c1d
  └┬┘ └─┬──┘ └───┬────┘ └─────────── tail, in any order ──────────────────────────┘ └──── id ────┘
   box  amount  payee
```

- **Box.**
  - `[ ]` means waiting; `[x]` or `[X]` means acknowledged.
  - Any other status is `bank_row_status`. The vault's `[?]`, `[/]`, and `[*]` mean task
    states.
- **Amount.**
  - Optional `**…**` bold, then an optional sign (`+`, `-`, or `−`), then **a required
    `$`**, then digits with optional thousands commas and an optional `.d` or `.dd`.
  - `+` means money in. No sign, `-`, or `−` means money out.
  - Parsed to signed **integer cents**, never a float. JSON uses negative for out.
- **Payee.** The text after the amount, up to the first ` [`, `[[`, `—`, `--`, `⚑`,
  `!!`, or trailing ` ^id`. It is trimmed and must not be empty.
- **Tail**, in any order:
  - `[cat:: key]` and `[acct:: alias]` Dataview inline fields;
  - wikilinks;
  - `⚑` or `!!`, which flags the row;
  - a note: the free text after the first `—`, `--`, or flag marker, with markers,
    fields, and links removed.
- **ID.** A trailing `^bank-<hex>` carries bank identity. Any other trailing block ID is
  Bryan's and is ignored.
- **Empty stub.** `- [x] $` or `- [ ] $` with nothing after the `$` is an unfinished
  line. It is ignored and yields no finding, so the plugin's Enter continuation is
  always safe.
- **Canonical write order** (import only):
  `- [ ] <±$amount> <Payee> [cat:: k] [acct:: a] [[link|icon]] ^bank-<hex>`. `[acct::]`
  is written only for a non-default account. bob never writes a note or a flag.
- **Amount display.** Thousands commas, always two decimals, and an explicit `+` on
  money in.

**Ownership** follows `decisions:bob-gcal-is-read-only-and-proposes-once`:

- bob writes a row once.
- After that, bob makes only two edits to a row: `ack` ticks `[ ]` to `[x]`, and import
  appends ` ^bank-<hex>` to a typed row it matched.
- Bryan's edits always win. A disagreement with the bank is reported by `check`, and bob
  never reverts it.

### What the totals mean

`report` computes every total from rows, at read time. Rows on `cash`-kind accounts and
rows with `[cat:: transfer]` are kept apart:

| Total             | Rule                                                                                                                  |
| ----------------- | --------------------------------------------------------------------------------------------------------------------- |
| **Out**           | The sum of outflows, excluding `transfer` and cash-account rows. It includes `cash` (ATM) and uncategorized outflows. |
| **In**            | Inflows categorized `income` or `refund`. A credit alone is never income.                                             |
| **Net**           | In − Out.                                                                                                             |
| Refunds           | `refund` inflows, also shown on their own.                                                                            |
| Transfers         | `transfer` in and out, shown, never in Out or In. A card purchase plus its payment from checking counts once.         |
| Cash out          | Outflows with `[cat:: cash]`, labelled "destination unknown". Cash-account rows sum into "cash traced".               |
| Unclassified      | Uncategorized outflows (inside Out) and uncategorized inflows (outside In). Always shown, never hidden.               |
| Breakdown         | Out by category, in declared order, then `cash`, then unclassified. Each entry has a share of Out.                    |
| Waiting / flagged | Counts of `[ ]` rows and `⚑` rows.                                                                                    |

An inflow with a spending category (for example, a restaurant refund tagged `dining`) is
counted as unclassified in, and `check` gives the info hint `bank_inflow_category`: "use
refund or income".

### Import: sources, identity, journal, matching, naming

- **Sources.**
  - **QFX/OFX** (Quicken Web Connect). Read OFX 1.x SGML (unclosed tags) and OFX 2.x
    XML. Read `STMTTRN` (`TRNTYPE`, `DTPOSTED`, `TRNAMT`, `FITID`, `NAME`, `MEMO`),
    `BANKTRANLIST` `DTSTART`/`DTEND`, `LEDGERBAL` `BALAMT`/`DTASOF`, and `BANKACCTFROM`
    `ACCTTYPE` or `CCACCTFROM`.
  - **CSV** fallback, one profile. Wells Fargo's commonly documented layout is
    `Date, Amount, *, (blank), Description`, with no header. Verify it against a real
    download when one is available. CSV has no IDs, so build a synthetic id
    `csv:<date>:<cents>:<occurrence>`, YNAB-style. CSV has no balance, so a CSV-only
    account is `unverified`.
  - Only posted transactions count. Pending transactions never reach QFX.
- **Account resolution.**
  - `-a ALIAS` wins.
  - Otherwise use the unique configured account whose `kind` matches the file's account
    type: checking, savings, or credit.
  - Otherwise refuse the file with exit 2, list the candidates, and teach `-a`.
  - Account numbers are read only for this match, and never stored or printed.
- **Identity.**
  - `key = sha256(alias + "\n" + bank_id)` as hex.
  - The row ID is `^bank-` plus the first 8 hex digits, lengthened to 12 when those 8
    already name a different key in the vault.
- **Journal** (`pref_journal` namespace `.bob/bank`, noun `bank`). Each host appends;
  readers fold every host.
  - `bank.import`:
    `{account, source, file_sha256, start, end, balance_cents?, balance_asof?, counts}`.
  - `bank.txn`: `{account, key, id, posted, cents, desc, merchant, outcome, note}`.
    - `outcome` is `written`, `matched`, or `before_start`.
    - `desc` is masked: every run of 4 or more digits becomes `••••`.
    - `merchant` is the normalized merchant key.
- **Seen.** A transaction is seen when any host journaled its key, or when a row in
  `bank/` carries its ID. A seen transaction is never written again.
- **Typed-row matching.** An unseen transaction matches a typed row when the row:
  - has no ID;
  - is on the same account (cash-account rows never match);
  - has the same signed cents;
  - sits under a day from 1 day before to 3 days after `posted`.

  Transactions are processed in `(posted, file order)` order, and each one takes the
  nearest-dated candidate. A typed row matches at most once. When two candidates are
  equally near, the match is **ambiguous**: the transaction is written as a new waiting
  row and reported, never guessed. Two identical purchases on one day both survive,
  because each has its own `FITID` (or CSV occurrence).

- **Naming written rows.** Precedence for payee, `cat`, and links:
  1. `bank.rules`, the first match;
  2. **learned**: the most recent _acknowledged_ row with an ID whose journaled
     `merchant` equals this transaction's merchant (copy its payee, `cat`, and links);
  3. built-in Wells Fargo rules:
     - `ATM WITHDRAWAL` → `ATM withdrawal` with `cash`;
     - ATM and overdraft fees → `fees`;
     - card payments (`CRD AUTOPAY`, `CREDIT CARD PAYMENT`, `PAYMENT - THANK YOU`) and
       `ONLINE TRANSFER` → `transfer`;
  4. the cleaned description, title-cased.

  **Cleaning** strips:
  - the `PURCHASE AUTHORIZED ON MM/DD`, `RECURRING PAYMENT AUTHORIZED ON MM/DD`, `SQ *`,
    `TST*`, and `PAYPAL *` prefixes;
  - `#1234` store numbers;
  - the trailing `S<digits> CARD <dddd>`;
  - a trailing city and state.

  With no learned link, a payee that equals a restaurant record's title
  (case-insensitive) gets `[[eat/<stem>|<kind emoji>]]`. Find the record through the
  same records registry `bob records list` uses, so this follows the albums cutover
  automatically.

- **Where rows land.** Each row goes under its `posted` day in that month's note, which
  is created when missing. New rows are appended after that day's last row. A late
  arrival in an earlier month is just another write there. The balance phase marks that
  month's proof as changed.
- **The transaction.**
  - Plan every file of one run together (overlapping files dedupe within the batch).
  - Install rows through `host_catalog::txn::run_locked` (vault identity, bounded lock,
    pre-sync, guarded preimages, one scoped commit `bank: import N rows (<alias>)`,
    post-sync). Then append the journal and include it in the same commit.
  - A malformed file or transaction refuses the **whole batch** (exit 2, nothing
    written).
  - A no-op run commits nothing.
  - `-d` prints the exact bytes and diffs, and appends no journal.
  - Raw files are never copied into the vault.

### Proving every transaction

For each importable account, `bank-balance` computes:

1. **Anchor.** Take the earliest `bank.import` with a balance that covers `start`. Then
   `anchor = balance − Σ journaled cents with start ≤ posted ≤ asof`. The first import
   sets the anchor, and every later import proves it.
2. **Coverage.** The union of the imports' `[start, end]` ranges is contiguous from the
   account's `start` to the latest `asof`. If the first file begins after `start`, that
   is a gap ("download a longer window").
3. **Bank balance.** For each later import, `anchor + Σ journaled cents ≤ asof` must
   equal its ledger balance (the _ledger_ balance, never the available balance). A
   mismatch means an import gap or a dedupe fault.
4. **Presence.** Every journaled `written` or `matched` transaction still has its row,
   with the journaled amount. A deleted row is `bank_row_missing`, because "deleted" is
   not the same as "never happened". An edited amount is `bank_amount_edited`.
5. **Status**, one per account:
   - `balanced`: coverage, bank balance, and presence all hold, and the account is not
     stale;
   - `delta`: shows `Δ $12.40` (the bank's balance minus anchor plus the row amounts
     with IDs) and lists the suspects;
   - `gap`;
   - `stale`: the last import is older than `stale_after`;
   - `unverified`: CSV only, or no import yet;
   - `untracked`: a `cash` account, which is never imported.

   bob never writes an adjusting row.

6. **Twins and leftovers.**
   - A typed row and an imported waiting row on the same account, dated within the match
     window, whose payees are equal or whose amounts differ by at most max($5, 25%), are
     listed as **possible twins**. Bryan deletes one; nothing merges.
   - A typed row on an importable account dated more than 3 days before coverage ends is
     `bank_row_unmatched` ("the bank never showed this").

### Findings (`bob bank check`)

| Code                                             | Severity | Phase   | Trigger                                                                                    |
| ------------------------------------------------ | -------- | ------- | ------------------------------------------------------------------------------------------ |
| `bank_hub_missing`                               | error    | grammar | `bank/` exists without `bank.md`, or `bank.md` has no `bank:` mapping                      |
| `bank_config_invalid`                            | error    | grammar | Any `bank:` problem (file, line, key path)                                                 |
| `bank_day_invalid`                               | error    | grammar | A date-shaped day heading outside the note's month, or an impossible date                  |
| `bank_row_unparsed`                              | error    | grammar | An amount-led row that doesn't parse (for example, three decimals)                         |
| `bank_row_payee_missing`                         | error    | grammar | A row with no payee                                                                        |
| `bank_row_status`                                | error    | grammar | A row whose box is not `[ ]`, `[x]`, or `[X]`                                              |
| `bank_row_task`                                  | error    | grammar | `#task` on a row                                                                           |
| `bank_row_outside_day`                           | error    | grammar | An amount-led row above the first day heading or under a heading that isn't a day          |
| `bank_category_unknown` / `bank_account_unknown` | error    | grammar | An unknown `[cat::]` or `[acct::]` value, with a suggestion                                |
| `bank_id_duplicate`                              | error    | grammar | One `^bank-` ID on two rows                                                                |
| `bank_month_mismatch`                            | warning  | grammar | `month:` or `type:` disagrees with the filename                                            |
| `bank_day_duplicate` / `_order` / `_weekday`     | warning  | grammar | Two headings for one day, headings out of order, or a weekday that disagrees with the date |
| `bank_row_no_checkbox`                           | warning  | grammar | An amount-led list item with no checkbox                                                   |
| `bank_row_zero`                                  | warning  | grammar | A `$0.00` row (no fake rows; completing the chore is the record)                           |
| `bank_field_unknown`                             | warning  | grammar | An inline field other than `cat` or `acct` (catches `[cta:: …]`)                           |
| `bank_flagged`                                   | info     | grammar | Each `⚑` row, so check lists everything to look into                                       |
| `bank_inflow_category`                           | info     | grammar | An inflow with a spending category                                                         |
| `bank_journal_malformed`                         | warning  | import  | Skipped journal lines (bounded, per `pref_journal`)                                        |
| `bank_balance_delta` / `bank_coverage_gap`       | error    | balance | The bank balance doesn't hold, or the coverage has a hole                                  |
| `bank_row_missing` / `bank_amount_edited`        | warning  | balance | A presence failure                                                                         |
| `bank_account_stale`                             | warning  | balance | The last import is older than `stale_after`                                                |
| `bank_row_unmatched`                             | warning  | balance | A typed row the bank never showed                                                          |
| `bank_twins`                                     | info     | balance | A pair of possible twins                                                                   |

### Commands

These follow `sase/memory/cli_rules.md`:

- `bank` goes in the **Integrations** section, with the membership rule "commands over
  bank rows, bank month notes, and their import journal".
- Subcommands are alphabetical, and options are sorted by long name. Every option has a
  short alias.
- The bare group runs the read-only `list`.

```text
bob bank [list] [-a] [-b DIR] [-f human|json] [-m MONTH]
bob bank ack    [ID...] [-b DIR] [-D DAY] [-d] [-f human|json]
bob bank check  [-b DIR] [-f human|json]
bob bank ensure [-b DIR] [-D DAY] [-f human|json] [-o]
bob bank import FILE... [-a ACCOUNT] [-b DIR] [-d] [-F] [-f human|json]
bob bank report [-b DIR] [-D DAY] [-f human|json] [-m MONTH] [-w WEEK] [-y YEAR]
```

- **Formats.**
  - `-D` takes `today`, `yesterday`, `tomorrow`, or `YYYY-MM-DD` (as in
    `bob gcal list`).
  - `-m` takes `YYYY-MM`, `-w` takes `YYYY-Www` (ISO, Monday first), and `-y` takes
    `YYYY`.
  - `report` takes at most one period and defaults to the current month.
  - `BOB_NOW` overrides "today" everywhere.
- **`list`** shows the waiting queue across every month, grouped by day, oldest first.
  `-a` shows all rows, and `-m` limits the output to one month.
- **`ack`** ticks the named waiting rows, every waiting row of `-D DAY`, or both. It is
  a guarded, capture-style write with no commit. A row that is already ticked is a
  no-op.
- **`ensure`** creates the month note and the day heading when they are missing. It
  prints the path and the line where the next row goes (after the day's last row, or the
  blank line under the heading). `-o` opens the note in Obsidian. It is a guarded write
  with no commit.
- **Exit codes.**
  - `list`, `report`, and `ensure` exit 0, or 1 on a runtime failure.
  - `check` exits 1 on any error.
  - `ack` exits 2 when an ID is unknown or isn't a row.
  - `import` exits 2 on a refused batch and 1 on a runtime failure.
  - `ensure` exits 2 on an invalid day or configuration.
  - Read-only commands never repair.
- **Human output** respects `NO_COLOR` and pipes, never depends on terminal width, and
  uses tabular, right-aligned amounts with explicit signs. Inflows are green and
  outflows are neutral, because spending is not an error. State is always carried by
  text and shape, never by color alone.

```text
$ bob bank
🏦 Bank · 3 to acknowledge                      WF Checking ✓ balanced · Nov 14

Tue Nov 10                                     out $275.83 · in +$1,250.00
  ☐      $61.20  ShopRite           🛒 Groceries
  ☐     $214.63  The Farmer's Dog   🐶 Dog
  ☐  +$1,250.00  Payroll            💵 Income

Tick them in bank/2026-11.md, or: bob bank ack -D 2026-11-10
```

```text
$ bob bank
🏦 Bank · nothing to acknowledge               WF Checking ⚠ last import 12 days ago
Download a QFX from Wells Fargo, then: bob bank import ~/Downloads/*.qfx
```

The output never says "all caught up" while any account is stale, has a gap, or shows a
Δ.

```text
$ bob bank report
🏦 November 2026 · so far                       WF Checking ✓ balanced · Nov 14
Out $1,904.12 · In +$2,500.00 · Net +$595.88    3 to acknowledge · 1 ⚑

🛒 Groceries      ███████████░░░░░   $512.40   27%
🍽️ Dining         ██████░░░░░░░░░░   $281.15   15%
🔁 Subscriptions  ████░░░░░░░░░░░░   $188.02   10%
🏧 Cash out       ███░░░░░░░░░░░░░   $130.00    7%  destination unknown
·  Unclassified   ██░░░░░░░░░░░░░░    $92.10    5%
Transfers ↔ $1,200.00 · Refunds +$0.00 · Cash traced $42.00

Mo Tu We Th Fr Sa Su      ░ under $25 · ▒ under $100 · ▓ $100+
                1  2      (one cell per day; today outlined; ☐ marks waiting days)
```

```text
$ bob bank import ~/Downloads/Checking1.qfx
🏦 Import · WF Checking · Nov 1 – Nov 14 · Checking1.qfx
  9 new (waiting) · 23 matched your rows · 41 already seen · 0 ambiguous · 4 before start
  out $1,204.33 · in +$2,500.00
  ledger balance $12,345.67 as of Nov 14 · ✓ balanced
  → bank/2026-11.md  +9 rows, 23 IDs
Committed bank: import 9 rows (checking). Next: bob bank to acknowledge the 9.
```

### JSON (additive; `schema_version: 1`)

Amounts are always signed integer `*_cents`, never strings or floats. Every report
shares these objects:

```json
{
  "ok": true,
  "schema_version": 1,
  "today": "2026-11-14",
  "categories": [
    { "key": "groceries", "emoji": "🛒", "label": "Groceries", "builtin": false }
  ],
  "accounts": [
    {
      "alias": "checking",
      "label": "WF Checking",
      "kind": "checking",
      "default": true,
      "start": "2026-11-12",
      "status": "unverified",
      "last_import": null,
      "balance_cents": null,
      "balance_asof": null,
      "delta_cents": null,
      "hint": "No import yet — bob bank import FILE"
    }
  ],
  "issues": { "errors": 0, "warnings": 0 }
}
```

- **Row:**
  `{path, line, date, account, cents, payee, category, links, note, flagged, acknowledged, id, origin}`.
  - `origin` is `typed` (no ID), `matched`, or `imported`. Before the journal exists, a
    row with an ID reports `imported`.
- **`list`:** adds `filter` (`waiting` or `all`), `month`,
  `days[] {date, heading, path, line, rows[]}`, and `counts {rows, waiting, flagged}`.
- **`report`:** adds:
  - `period {kind, key, start, end, label, so_far, note}`;
  - `totals {out_cents, in_cents, net_cents, refund_cents, transfer_out_cents, transfer_in_cents, cash_out_cents, cash_traced_cents, unclassified_out_cents, unclassified_in_cents}`;
  - `breakdown[] {key, emoji, label, out_cents, share, rows}`;
  - `days[] {date, heading, path, line, out_cents, in_cents, rows, waiting, flagged}`
    (every day of the period, including empty ones);
  - `waiting {period, all, oldest}` and `flagged`.
- **`check`:** adds `findings[] {code, severity, path, line, message, hint}`, with
  `issues` counting them by severity.
- **`ensure`:** adds `{path, line, day, created_note, created_heading}`.
- **`ack` and `import`** print the notebook-style envelope: `ok`, `dry_run`, `changed`,
  `changes[] {path, diff?}`, `findings`, `commit`, `sync`, and `warnings`.
  - `ack` adds `acknowledged[]`, `already[]`, and `unknown[]`.
  - `import` adds
    `files[] {file, account, source, start, end, new, matched, seen, ambiguous, before_start, out_cents, in_cents, balance_cents, balance_asof, status}`
    and `rows[]`.

### Views in Obsidian (bob-ledger-tools)

Every number comes from `bob bank list|report -f json`. Nothing is written into notes.

- **Month card** (a `bob-bank` fence with `view: month`, in each bank month note; the
  month comes from the note's `month:`):
  - **Header:** `🏦 November 2026 · so far` on the left, and the account status chip on
    the right (`✓ balanced · Nov 14`, `Δ $12.40`, `⚠ import 12d ago`, or `unverified`).
  - **Totals line:** Out · In · Net, plus a `3 to acknowledge` pill that scrolls to the
    first waiting row.
  - **Category bars** (emoji, label, bar, amount, share). Cash out carries "destination
    unknown".
  - **Heat strip:** one cell per day, Monday first. Intensity comes from the month's own
    quantiles. Today is outlined, waiting days get a dot, and a click scrolls to that
    day's heading.
  - **Navigation:** `‹ Oct · 🏦 Bank · Dec ›`. Only months that exist are links.
  - An empty month shows a teaching state.
- **Queue card** (`view: queue`, on `bank.md`):
  - The header reads `🏦 3 to acknowledge`, or `✓ nothing to acknowledge` with any
    account caveat.
  - Waiting rows are grouped by day, oldest first, each linking to its line. Each row
    has a checkbox that acks that row. Each day has `✓ Acknowledge day`, which acks
    exactly the IDs drawn.
  - The footer shows the account statuses and the last import, the weekly import
    command, and links to the six most recent months with their Out.
- **Rows** in `bank/*.md`, in Live Preview and Reading view:
  - amounts are tabular and right-aligned in a fixed-width column; inflows are green
    with `+`;
  - `[cat:: k]` shows as an `🛒 Groceries` chip, and `[acct:: a]` as a muted account
    chip;
  - `⚑`/`!!` shows as a flag chip;
  - `^bank-…` is hidden and replaced by a faint "bank ✓" glyph;
  - a waiting row gets a 2px area-hue start bar, and acknowledged rows dim slightly;
  - raw syntax reappears when the cursor is on the line (the root-marks pattern).
- **Day headings** get a right-aligned `out $357.29 · in +$1,250.00` and a
  `2 to acknowledge` badge.
- **Daily chip.** Inside the rendered `bob-plan` block, after the day strip, every daily
  note shows one quiet chip for its date, `🏦 out $41.20 · in +$0.00`. Today's note adds
  `· 3 to acknowledge`. A click runs `ensure` for that day and jumps there. The chip is
  hidden when there is no `bank.md`, and a setting can turn it off.
- **Commands.**
  - `Bank: open today` (`bank-open-today`) runs `ensure -D today` and places the cursor
    where the next row goes.
  - `Bank: acknowledge this day` (`bank-ack-day`) runs `ack` with the IDs in the day
    section under the cursor.
  - Neither gets a default hotkey; the plugin reference documents both.
- **Typing helpers** (bank month notes only):
  - Enter at the end of a row continues with `- [x] $`. Enter on an empty stub clears
    it.
  - `[cat:: ` suggests categories (emoji and label, in declared order) from the JSON
    `categories`.
- **Guards.**
  - The `bank` property row is hidden on `bank.md` only.
  - In `bank/*.md`, task-status-cycler and ledger-tools task gestures must never add
    `#task`, `[completion::]`, or a non-`[ ]`/`[x]` status to a bank row. A complete
    gesture on a row only toggles `[ ]`↔`[x]`.
  - `$` on a row must never render as inline math. Verify both views.

### Privacy and security

- The vault is a private Git repository on three hosts, and the old `@BANK` amounts are
  already in it. Even so, scope the exposure on purpose:
  - account aliases only, never account numbers;
  - masked descriptions in the journal;
  - raw QFX/CSV files never copied into the vault;
  - no credentials anywhere.
- Fixtures are synthetic. No real bank data ever enters a repository, a phase report, or
  a bead note. Reports give counts only.
- v1 adds no bank data to any agent-facing output (period briefs, Stats, the Mac app).

## Phases

```text
bank-restart ─────────────────────────────────────────────────┐
bank-grammar ─► bank-cli ─┬─► bank-import ─► bank-balance ─────┤
                          └─► bank-obsidian ───────────────────┤
                                               bank-wrapup ◄───┘
```

- The ritual starts with `bank-restart`, independent of all code.
- The code phases run one after another inside bob-cli, so they never fight over
  `src/native/bank/`. Only `bank-obsidian` (a different repository) runs in parallel
  with import and balance.

Every code phase follows the repo's habits:

- Read the `cli_rules` and `docs` reference memory first.
- Gate on `just check` and `just lint`. Run `just docs-gen` after CLI changes and
  `just docs-check` after docs changes.
- Use conventional commits (`feat(bank): …`). Never edit the `Cargo.toml` version or
  `CHANGELOG.md`.
- New JSON grows only additively.

Vault phases read the `obsidian` reference memory first, edit `~/bob` directly, and let
`bob vault-sync` carry the commits.

### bank-restart

**Goal:** Bryan can do the daily ritual tomorrow morning, in the final grammar, with
today's bob.

- **Read first:**
  - the research sections "The month ledger note" and "The daily and weekly rhythm";
  - this plan's contract;
  - `notebook.md` and `legacy_cash.md`, for the house style.
- **Baselines** (keep the scripts for the after-run):
  - the vault-wide unresolved-link count (wikilinks, embeds, heading and block refs;
    excluding `.bob/`, `_conflicts/`, and gitignored trees);
  - the vault-wide `#task` count;
  - `bob notebooks check` and `bob records check` counts.
- **`bank.md`** exactly as in the contract:
  - Keys: `parent`, `type`, `created`, then `bank:`.
  - `start` is today.
  - The starter categories: sanity-check them against the most common `@BANK`
    descriptions in the 2025 `_habit.md` notes, and adjust at most a few keys or labels.
    Keep the four built-ins undeclared unless relabelled.
  - The intro and the queue fence.
  - Do not add `color:`; cash's magenta applies through the root.
  - This resolves the dangling `Related: [[bank]]` in `cash.md`.

> [!decision] accounts = checking
>
> Seed `accounts:` with `checking` only (label `WF Checking`, kind `checking`).

> [!decision] accounts = checking_card
>
> Also seed `card` (label `Credit card`, kind `credit`, the same `start`) after
> `checking`. The built-in card-payment rules mark its payments from checking as
> `transfer`. Name the card's real issuer in `label` only if the vault states it.

- **`bank_month.md`** under `[[type]]`, in the house style of `notebook.md`:
  - `parent` and `created`;
  - two sentences: a bank month note holds one month of bank rows under day headings,
    and ticking a row acknowledges it;
  - a pointer to `bob bank` (ships with `bank-cli`).
- **`bank/<YYYY-MM>.md`** for the current month, byte-for-byte the canonical template,
  with one heading for today and no rows. Bryan types the first rows himself.
- **The PRE chore.** Capture it into `gtd_daily` with `bob capture` (not by hand), so
  the task-status hooks place it, with `[repeat:: every day when done]`, scheduled from
  tomorrow:

  ```text
  #gtd #pre **Acknowledge bank**: open the Wells Fargo app · type each new posted row under its day in this month's [[bank]] note (ticked) · tick any waiting rows · nothing new? completing this is the record
  ```

- **Retire `cash.md ^daily-bank-records-again`.** Cancel it as superseded by the new
  chore, with the relation that `docs/task-relations.md` defines. Use bob's own
  gestures; hand-edit only where none exists, and say so.
- **Leave `bob.md ^bank-records` open** for Bryan, and mention it in the report.
- **Gates.** Paste all of these into the phase report:
  - The unresolved-link count did not increase. It should drop by one: `[[bank]]`.
  - The `#task` count went up by exactly one (the chore) for open tasks, and the
    cancelled task is accounted for.
  - `bob notebooks check` and `bob records check` are unchanged.
  - Every file created, in full.

### bank-grammar

**Goal:** bob understands bank notes and tells Bryan precisely what is wrong with them,
read-only.

- **Module** `src/native/bank/`:
  - `config.rs` for the `bank:` parser and its diagnostics;
  - `grammar.rs` for row and day-heading parsing, and the canonical rendering of rows,
    headings, and amounts;
  - `month.rs` for the month-note model (frontmatter, days, rows with line numbers), the
    canonical template, and pure insertion planners: insert a day heading in order,
    append rows to a day, tick a row, append an ID to a row;
  - `scan.rs` for `bank.md` and `bank/*.md` discovery;
  - `check.rs`, `cli.rs`, and `output.rs`.

  The planners preserve BOM, CRLF, and every other byte. Later phases call them, so
  golden-test them now.

- **Every configuration key** in the contract is parsed and validated here, including
  `rules` and `stale_after` (consumed later). Reuse the albums parser's
  indentation-aware line walk for key-path diagnostics, and the album choice-label split
  for category labels.
- **`bob bank check`** with every finding marked "grammar" in the table. Human output
  groups by note, then day, and ends with a summary. `-f json` follows the contract.
  Exit 1 on errors.
- **Wiring.** Add `bank` to the Integrations section of `bob --help` and to the
  `bob <TAB>` completion groups. Add `check` and its catalog entry
  (`src/native/catalog/table.rs`). Until `bank-cli`, bare `bob bank` prints the group
  help.
- **Invisibility test.** A fixture vault holding a bank month note produces no tasks in
  `bob plan`, `bob ready`, `bob task archive -d`, or `bob task reconcile -d` (adapt this
  to each command's real dry-run flag).
- **Tests.** Golden parse cases for every grammar rule and every finding:
  - bold and plain amounts;
  - the `−` sign, commas, `$0.10 + $0.20`;
  - `!!` and `--`;
  - the empty stub;
  - every heading spelling;
  - CRLF and BOM.
- **Docs.**
  - Write `docs/bank.md` as the root contract page: files, configuration, grammar,
    ownership, totals, and findings. Add it to the `mkdocs.yml` nav next to records and
    notebooks.
  - Add a "Bank notes" section to `docs/reference/vault-format.md`.
  - Add `check` to `docs/reference/json-outputs.md`.
  - Run `just docs-gen` and `just docs-check`.
- **Real vault.** If `bank.md` exists, run `bob bank check` read-only against `~/bob`
  and paste the counts. Never edit Bryan's rows. List any finding he should fix.

### bank-cli

**Goal:** the terminal answers "what is waiting?" and "where did the money go?", and
Obsidian has the two write gestures it needs.

- **`list`** (the bare default) and **`report`** exactly per the contract, including
  every totals rule and the JSON objects.
  - Week boundaries are ISO, Monday first.
  - `report -D` yields one day: that is the daily chip's data.
  - Human output follows the samples in spirit: bars, the heat strip, and no dependence
    on terminal width.
- **`ensure`** and **`ack`** use the `bank-grammar` planners and the shared guarded
  writer (`write_staged_files_guarded`, with preimage checks), as capture does. There is
  no commit, and vault-sync carries the change.
  - `ensure` creates the folder, note, and heading on demand, and is idempotent.
  - `ack` refuses unknown IDs with exit 2 and changes nothing on a refusal.
- **Account status.**
  - Add one `bank::status::accounts(...)` function returning the contract's
    `accounts[]`. In this phase every importable account is `unverified`, with the hint
    `No import yet — bob bank import FILE`.
  - `cash` accounts report `status: "untracked"`.
  - `bank-balance` replaces only the body of that function.
- **Tests:**
  - every totals rule;
  - the acceptance arithmetic cases;
  - `ensure` on a missing note, a missing heading, and an existing heading;
  - `ack` for IDs, a day, already-ticked rows, and unknown IDs;
  - `BOB_NOW`.
- **Docs:**
  - `docs/bank.md` (commands and totals);
  - `json-outputs`;
  - `just docs-gen`;
  - `just docs-check`.

### bank-import

**Goal:** a weekly download and one command make sure no transaction is ever missed,
without retyping anything.

- **Read first:** the research sections "Getting transactions in" and "Payees,
  categories, and links", `decisions:theme-prefs-live-in-per-host-journal`, and
  `decisions:bob-gcal-is-read-only-and-proposes-once`.
- **Implement** the Import contract in full:
  - the OFX/QFX and CSV parsers (tolerate both OFX dialects, any header block, and
    `CHARSET` noise);
  - account resolution;
  - identity, the `.bob/bank` journal namespace, and "seen";
  - typed-row matching;
  - naming (rules, learned, built-ins, cleaning, restaurant links);
  - where rows land;
  - the one-commit transaction.
- **Order of effects:** install rows first, then append the journal. "Seen" includes
  vault IDs, so an interrupted run never duplicates a row.
- **Output** as in the human sample and the JSON envelope. Print the ledger balance and
  its date raw; `bank-balance` adds the verdict.
- **Fixtures.** Use synthetic Wells Fargo-shaped QFX files (SGML and XML) and a
  synthetic CSV. Never commit real data. They cover:
  - overlapping windows;
  - two identical purchases on one day;
  - a typed twin;
  - an equidistant ambiguous match;
  - a `$20` typed row against a `$24` posting;
  - an ATM withdrawal plus its fee;
  - a card payment;
  - payroll;
  - a refund;
  - a transaction before `start`;
  - a malformed amount that refuses the batch;
  - two accounts with `-a`.
- **Acceptance cases** from the table below that belong to import, each as one test.
- **Real file (optional).** If a real Wells Fargo QFX is on this host, run `-d` against
  a temporary copy of the vault and report counts only.
- **Vault, after verifying.** Capture the weekly PRE chore into `gtd_daily` with
  `[repeat:: every week on Monday when done]`:

  ```text
  #gtd #pre **Bank import**: download the last 14 days as Quicken (QFX) from Wells Fargo · run `bob bank import ~/Downloads/*.qfx` · tick what it wrote (bob bank) · write one line in last week's note: what surprised me?
  ```

- **Docs:**
  - `docs/bank.md` (import, identity, the journal, naming);
  - `docs/vault-git-sync.md` (the `bank: import` commit label and the `.bob/bank`
    journal files);
  - `json-outputs`;
  - `just docs-gen`;
  - `just docs-check`.

### bank-balance

**Goal:** "every transaction" is checkable, and the views say so honestly.

- Implement "Proving every transaction" in full:
  - the anchor;
  - coverage;
  - bank balance;
  - presence;
  - status;
  - twins and leftovers.
- **Wire the status** into `check` (every finding marked "balance" in the table), into
  the body of the `bank::status` function (so `list`, `report`, and the plugin pick it
  up with no other change), and into import's verdict line.
- **`bob bank import -F`** re-adds journaled transactions whose rows are missing, under
  their posted days. It never duplicates a row that is present, and it reports what it
  restored.
- **Tests:**
  - every acceptance case in the balance column;
  - a deleted imported row (`Δ` and `bank_row_missing`, then `-F` restores it);
  - an edited amount;
  - a skipped week (`gap`);
  - a stale account;
  - a CSV-only account (`unverified`);
  - the first file starting after `start`.
- **Docs:**
  - `docs/bank.md` ("Proving every transaction", statuses, and `-F`);
  - `json-outputs`;
  - `just docs-gen`;
  - `just docs-check`.

### bank-obsidian

**Goal:** opening a bank note feels like reading a well-designed statement, and the
daily ritual takes about two minutes.

- **Setup.**
  - Work in `bob-plugins`: open it with `/sase_repo` and read its `AGENTS.md`.
  - Edit `bob-ledger-tools` `src/` fragments only, never the generated `main.js`.
  - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.
- **Concurrency.** `bob-cli-k3`'s `plugins-albums` phase may be landing in the same
  plugin.
  - Rebase onto it.
  - Keep both sides in `fragments.json`, the lifecycle `onload`, and `styles.css`.
  - Regenerate `main.js` with `npm run build` instead of hand-merging it.
- **Data fragment**, modelled on the reading board (`241-plugin-reading-board-data.js`)
  and the notebook data fragment (`241a`):
  - stale-while-revalidate caches of `bob bank list -f json` and
    `bob bank report -f json` (keyed by args), built on `bobSpawnJson` with
    `schema_version` validation;
  - the stats error-card model for `bob-not-found`, `timeout`, and `schema-mismatch`;
  - invalidation on `metadataCache` changes to `bank.md` or `bank/*.md`, on focus and
    visibility, and on day rollover;
  - a `bank` tag in `283-plugin-command-mode-refresh.js`.
- **Views:** the month card, the queue card, the row and day-heading decorations, the
  daily chip, the commands, the typing helpers, and the guards, exactly as in "Views in
  Obsidian".
  - Model row chips on `279-plugin-root-marks.js`: `Decoration.replace`, Live Preview
    only, skipping code, revealing on the cursor line, with a Reading view
    post-processor.
  - Model the daily chip on the day strip (`239`).
  - Use the existing area-hue variables (`--bob-area-<hue>` through `data-root-hue`),
    `tabular-nums`, and the chip language of the `bob-plan` header (small-caps label,
    2px accent start border, 14% accent background).
  - Narrow panes stack the chips under the text.
- **Mark gate.** One `toggle-bank-marks` command and setting, following
  `266-plugin-mark-gate-mirror.js`, turns the row decorations off.
- **Tests** (`scripts/test-ledger-tools-bank-*.cjs`):
  - JSON decoding and errors;
  - the month model (bars, heat-strip buckets, navigation);
  - the queue model (ack payloads carry only drawn IDs);
  - row tokenization for every grammar spelling;
  - Enter continuation;
  - the task-gesture guard;
  - no math rendering.
- **Docs:** the `bob-ledger-tools` reference page (cards, chip, row styling, settings;
  the commands table is generated), and `npm run docs:check`.
- **Report:** a screenshot or text capture of each view against a fixture vault, plus
  the `bob plugins sync` output.

### bank-wrapup

**Goal:** proven end to end, documented as one story, and remembered.

- **End-to-end walkthrough** in a temporary vault (`BOB_DIR`, `BOB_NOW`), with the
  transcript pasted into the report:
  1. create `bank.md`, then `ensure`;
  2. type rows (including a stub, a flag, a note, and a cash-account row), then `check`;
  3. `list` and `report` for a month, a week, and a day;
  4. `ack` by IDs and by day;
  5. import a synthetic QFX twice;
  6. the typed twin, the $20/$24 pair, and an ambiguous tie;
  7. delete a row → `Δ` → `import -F`;
  8. skip a week → `gap`;
  9. two accounts plus a card payment that nets out;
  10. a malformed file refuses the batch.
- **Guide.** Write `docs/guides/acknowledge-bank-transactions.md` (a goal-shaped guide):
  - the two-minute daily ritual;
  - the Monday import;
  - catching up after a lapse;
  - adding an account or a rule;
  - what the statuses mean.
- **Docs audit.** Every bank mention must agree with shipped behaviour, in:
  - `docs/bank.md`;
  - the guide;
  - `vault-format.md`, `json-outputs.md`, and `vault-git-sync.md`;
  - the CLI reference;
  - the plugin page.

  Then `just docs-check`.

- **Real vault, read-only.** Paste `bob bank check`, `bob bank`, and `bob bank report`
  (amounts redacted to counts in the report) and list any findings for Bryan.
- **Memory.** Read and use `/sase_memory_write`, and apply only the accepted decisions.
  - Follow the house strand shape: frontmatter `keyword` and `aliases`, no H1, prose
    wrapped near 88 columns, links as `[[slug]]` / `[[decisions:slug]]`.
  - Then run `sase memory init`, `just docs-gen`, and `just docs-check`.

  > [!decision] memory_bank_glossary
  >
  > - **Create `glossary:bank-month-note`** (aliases: bank month, month note).
  >   - Define it: `bank/YYYY-MM.md` under the bank hub, with one `##` heading per day,
  >     oldest first, and [[bank-row]]s beneath.
  >   - Bob writes it once from the canonical template.
  >   - Totals are drawn, never written.
  >   - What it is not: a [[notebook]], an album, or a [[period-note]].
  > - **Create `glossary:bank-row`** (aliases: bank row, transaction row).
  >   - Define it: one posted transaction or deposit as a top-level checkbox line
  >     (`- [x] $13.71 Smashburger [cat:: dining]`). `[ ]` means waiting and `[x]` means
  >     acknowledged.
  >   - Typed rows start ticked. Import writes the rest unchecked, once.
  >   - It is never `#task` and never a record note.
  >   - It is checked by `bob bank check`.

  > [!decision] memory_bank_decision
  >
  > Create `decisions:bank-rows-not-records`:
  >
  > - **Claim.** Bank transactions are acknowledged checkbox rows in bank month notes
  >   under the `bank` sub-area. bob writes each imported row once, and Bryan's edits
  >   win. A per-host journal of hashed IDs prevents duplicates and proves coverage with
  >   the bank's own balance. Every total is drawn, never written. The album belongs to
  >   subscriptions.
  > - **Evidence:** the research report, this epic's commits, and `docs/bank.md`.
  > - **Rejected:**
  >   - an album with one record per transaction (it breaks the index, filename,
  >     checkbox, and durability rules);
  >   - an album of month statements;
  >   - a payee album;
  >   - `## Bank` rows in daily notes (the runner-up);
  >   - hand-kept totals;
  >   - hledger or beancount as the home.
  > - **Cost:** a weekly manual download, and no pending transactions.
  > - **Reopens when:** a read-only feed passes a Wells Fargo spike, or the goal moves
  >   from awareness to budgeting (Actual Budget).

- **Follow-ups.** Append `PROPOSED FOLLOW-UP:` notes to this phase's bead for the land
  agent:
  1. **The subscriptions album**, designed in the next section. It must depend on
     `bob-cli-k3` (`sase bead dep add <new-task> bob-cli-k3`). Carry the
     `subscriptions_scope` answer from this plan.
  2. **Later views, after a 2–4 week trial** with at least two months of rows: typical
     week and month comparisons in the month card, a year card, a `bank` Stats metric, a
     bank line in Bob Mac Capture's idle agenda (bob JSON first, thin client), and a
     week-note fence.
  3. **A read-only feed spike** (SimpleFIN or Teller), only if the trial shows the
     weekly download is the friction.
- **Report, but do not close,** `bob.md ^bank-records` for Bryan. Also note that the
  30-day `cash.md ^balance` snapshot chore becomes optional once the account shows
  `✓ balanced`.

## Follow-up: the subscriptions album

This waits for `bob-cli-k3` (`bob albums create`, the registry, and the cutover) and for
this epic's `bank-import`. It is filed as a task bead, not a phase, because an epic
phase cannot depend on another epic.

- **Create it with `bob albums create`:** parent `bank`, stem `subscriptions`, noun
  `subscription`, emoji `🔁`, title `Subscriptions`, summary "Everything that charges us
  on repeat, and whether it still earns its keep.", and
  `index_facets: [amount, cadence, category]`.
- **Fields:**
  - `status` (`to_cancel: ✂️ To cancel`, `active: 💚 Active`, `cancelled: 🚫 Cancelled`,
    with the actionable group first);
  - `amount` (number, min 0, required);
  - `cadence` (`monthly`, `yearly`, required);
  - `payee` (suggest; the bank-row payee the subscription shows up as);
  - `category` (suggest);
  - `renews` (date);
  - `url`.
- **Seed** it from `old_recur.md`'s active rows, then de-zorg `old_recur.md` into a
  `legacy_recur` notebook under `cash`, following the `eat.md` recipe (keep every block
  ID; rewrite or report every inbound link).

  > [!decision] subscriptions_scope = discretionary
  >
  > Seed only keep-or-cancel charges. Mortgage, student loans, car insurance, utilities,
  > and Kelly's salary stay as bank categories and rules.

  > [!decision] subscriptions_scope = all_recurring
  >
  > Seed every recurring charge, and add a `kind` choice (`choice`, `obligation`), so
  > the index separates what can be cancelled from the fixed monthly burn.

- **Link existing bank rows** to their records (`[[subscriptions/<stem>|🔁]]`). Learned
  naming then carries the link to every future import.
- **`bob bank check` proposes, never creates:**
  - `bank_subscription_price` when a linked row's amount differs from the record's
    `amount` ("Spotify $11.72 → $12.99");
  - `bank_subscription_candidate` when there are 3 or more charges from one merchant
    about 30 or about 365 days apart with no linked record.
- **Point** `cash.md ^audit-monthly-subscriptions` at the album.

## Out of scope (v1)

- Budgets, envelopes, split transactions, multiple currencies, investments, net worth,
  pending transactions, and cloud-model categorization (ADJ-8).
- Importing the historical `@BANK` lines. They stay as history, and a `total:` line is
  never an expense.
- A capture route for bank rows. Typing in the month note with Enter continuation is the
  v1 path, and a Mac capture form can follow the trial if typing is the friction.
- Any bank data in agent-facing outputs.

## Trial (for Bryan, after `bank-cli` ships)

Run the ritual for 2–4 weeks. These targets are product goals, not research findings:

- a median daily session under about 3 minutes;
- a missed week recovered in one sitting;
- the Monday import showing `✓ balanced`;
- at least one useful observation a week (the ATM fees are a likely first one).

If the time goes mostly to typing, move the feed spike earlier. If it goes mostly to
categorizing, drop categories from the chore.

## Acceptance cases

| Case                                                | Required outcome                                                                                 | Phase    |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | -------- |
| The same QFX imported twice, or overlapping windows | No new rows, and the review state is unchanged                                                   | import   |
| Two identical purchases on one day                  | Both survive, and overlapping imports never double them                                          | import   |
| A typed row, then the import of its twin            | One row, still ticked, now carrying `^bank-…`, with Bryan's text kept                            | import   |
| A $20 typed tab that posts at $24                   | The posted row is written waiting, and `check` lists the pair as possible twins. Nothing merges  | balance  |
| An imported row that is later deleted               | It stays deleted. `check` shows Δ and `bank_row_missing`, and `import -F` restores it on purpose | balance  |
| A card payment from checking, a salary, a refund    | Transfers net out, and a credit alone is never income                                            | cli      |
| `$0.10 + $0.20`, commas, and `−` signs              | Exact integer-cent totals                                                                        | cli      |
| A gap in imports, or a stale account                | No unqualified "all caught up", and the account is named                                         | balance  |
| A day with no transactions                          | Completing the chore is the record. No fake row is written                                       | restart  |
| A historical `@BANK` `total:` line                  | Never imported or counted                                                                        | grammar  |
| A malformed transaction in a file                   | The whole batch is refused (exit 2), and nothing is written                                      | import   |
| Enter after a row, Enter on an empty stub           | `- [x] $` continues the list, and the stub clears without a finding                              | obsidian |

## Risks

| Risk                                                         | Handling                                                                                                                                            |
| ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| The real Wells Fargo QFX/CSV shape differs from the fixtures | Both OFX dialects are tolerated. A malformed batch writes nothing. The optional real-file dry run, plus Bryan's first Monday import, are the proof. |
| The habit lapses again                                       | A daily PRE chore instead of a decaying task. Import pre-fills what was missed, and acknowledging a whole day is one click.                         |
| Task gestures corrupt bank rows                              | The guards in `bank-obsidian`, plus the `bank_row_status` and `bank_row_task` findings.                                                             |
| Two hosts import the same file before syncing                | "Seen" folds every host after sync. A same-point merge can duplicate rows, which `bank_id_duplicate` names. Bryan imports from one host.            |
| `$` renders as inline math                                   | The plugin verifies and guards both views.                                                                                                          |
| Plugin merge conflicts with `bob-cli-k3`'s plugin phase      | Rebase, keep both sides, and rebuild `main.js`.                                                                                                     |
| Bank data leaks into repositories or reports                 | Synthetic fixtures, masked journal descriptions, hashed IDs, and count-only reports.                                                                |

## Success criteria

- From tomorrow, Bryan acknowledges his bank daily through a PRE chore, typing rows in
  the final grammar. `^daily-bank-records-again` is retired as superseded.
- `bob bank` shows what is waiting, and `bob bank report` explains the month with exact
  cents. Transfers net out, and a credit alone is never income.
- One Monday command imports Wells Fargo QFX. It writes only what Bryan missed, writes
  each row once, and never writes a duplicate.
- `bob bank check` shows `✓ balanced` from the second import onward. A gap, a stale
  account, a deleted row, and an edited amount are each named.
- Bank month notes open on a month card with bars, a heat strip, and status. `bank.md`
  shows the queue, and each daily note shows one quiet chip. Every number is drawn.
- The subscriptions album, the later views, and a feed spike are proposed follow-ups.
  The album depends on `bob-cli-k3`.
