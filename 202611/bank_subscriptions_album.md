---
tier: epic
title: 'Subscriptions album: records that know how they show up on the statement'
goal: 'A 🔁 Subscriptions album under bank holds every discretionary recurring charge,
  seeded from a reviewed bank history. bob bank import links each new matching row
  to its record, and the link renders as a quiet 🔁 chip. bob bank check and bob bank
  subscriptions keep the album honest: they flag price changes, charges after a cancel,
  and possible new subscriptions. The old Zorg recurring table retires without losing
  a link, and bob-cli-kt closes.

  '
decisions:
  payee_source:
    ask: When import links a new row to a subscription, should the row's payee be
      the record's title?
    choices:
      record_title: The row reads as the record title; rename a service once, on its
        record
      learned_first: Your last hand-edited payee for that merchant wins; the title
        fills only new merchants
    default: record_title
    why: It is deterministic, and the record is the one place where a service is named
    answer: record_title
  memory_decision_record:
    ask: Add a decisions strand recording that subscription records own their bank
      match?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_term:
    ask: Add a Subscription Record glossary term (aka "Shows up as")?
    memory:
    - glossary
    default: false
    answer: false
phases:
- id: seed
  title: Seed the album from a reviewed bank history
  depends_on: []
  size: medium
  description: 'seed: create the subscriptions album with the full schema, regenerate
    the evidence from the bank journal, ask Bryan one question round, write the reviewed
    records, add obligation bank rules, and hand-link the live rows.'
- id: core
  title: Subscription registry, match resolver, and ledger
  depends_on: []
  size: large
  description: 'core: load subscription records, implement the Shows-up-as match contract
    and the single attribution resolver over rows and journal history, derive each
    record''s ledger, and add subscription fields to bank row JSON.'
- id: legacy
  title: Retire old_recur.md into a legacy_recur notebook
  depends_on:
  - seed
  size: medium
  description: 'legacy: move the Zorg table into a legacy_recur notebook under cash,
    keep every block ID, point historical links for discretionary rows at their records
    and every other link at the notebook, and point the cash audit task at the album.'
- id: import
  title: Import links new rows to their subscription
  depends_on:
  - core
  size: medium
  description: 'import: insert the subscription tier into import naming (rule, subscription,
    learned without subscription links, restaurant), fill in payee and category, and
    add the import summary line.'
- id: check
  title: Subscription findings in bob bank check
  depends_on:
  - core
  size: medium
  description: 'check: report charged-after-cancel, still-charging, price-change,
    candidate, invalid-needle, overlap, and unknown-category findings, with paste-ready
    fixes. A bank rule doubles as a candidate''s dismissal.'
- id: report
  title: The bob bank subscriptions command
  depends_on:
  - core
  - check
  size: medium
  description: 'report: add the read-only bob bank subscriptions report with beautiful
    human output and the JSON contract every plugin surface reads, plus the audit
    guide.'
- id: chips
  title: 🔁 chips on bank rows
  depends_on:
  - core
  size: medium
  description: 'chips: render subscription links as state-aware 🔁 pills in Live Preview,
    Reading view, and the queue card, with tooltips, ghost chips, and click-to-link.'
- id: linker
  title: Category card subscription linker
  depends_on:
  - chips
  - report
  size: medium
  description: 'linker: add the slash picker, the context strip, and the New subscription
    sheet to the Category card, and connect ghost-chip clicks to them.'
- id: surfaces
  title: Record header strip and album summary card
  depends_on:
  - linker
  - report
  size: medium
  description: 'surfaces: draw the per-record header strip with its charge timeline,
    the album summary card, and the formatted index rows on the subscriptions album.'
- id: acceptance
  title: Vault acceptance, memory, and closing bob-cli-kt
  depends_on:
  - legacy
  - import
  - surfaces
  size: medium
  description: 'acceptance: prove the delivered system on the real vault, keep the
    vault candidate-clean, link rows written since the seed, land the accepted memory,
    and close bob-cli-kt.'
proposed_by: bbugyi200.athena.1cp
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 13:11:17
status: done
bead_id: bob-cli-ls
---

- **PROMPT:** [prompts/202611/bank_subscriptions_album.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_subscriptions_album.md)
- **BEAD:** [bob-cli-ls](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ls/README.md)

# Plan: Subscriptions album — records that know how they show up on the statement

## Why

Bead `bob-cli-kt` asks for a subscriptions album under `bank`. It should be seeded with
discretionary keep-or-cancel charges and linked from bank rows, and `bob bank check`
should propose price changes and new candidates. Bryan's request adds three things:

1. Review his recent bank history so the album is pre-filled with the right records.
2. Have `bob bank import` link each new row to its subscription record automatically,
   rendered concisely and beautifully.
3. Lead the design so it is intuitive, reliable, and beautiful.

The vault already asks for this. In `bob.md`, `^bank-subscriptions` lists "Create
[[subscriptions]] album!" and "Add ability to link to subscription from category
picker!", and `cash.md ^audit-monthly-subscriptions` has waited since June.

The consolidated research report
`research:202611/bank_subscriptions_feature_design/bank_subscriptions_feature_design.md`
is the design basis. **Bryan agreed with every requirement it recommends**, including
its fourteen adjustments to the bead (ADJ-1 … ADJ-14). Every phase worker reads the
sections relevant to its phase with
`sase artifact read research:202611/bank_subscriptions_feature_design/bank_subscriptions_feature_design.md "<why>"`.
This plan settles the details the report left open, splits the work, and adds the design
calls listed under
[Design calls beyond the research](#design-calls-beyond-the-research).

The bead's dependency `bob-cli-k3` (albums) is closed, so its "still in progress" note
is stale. `bob albums create`, the registry, and the records writer all ship today.

### Privacy rule (every phase)

Bank data stays out of beads, plans, commit messages, docs, test fixtures, Stats, period
briefs, and the Mac app. This covers service rosters with amounts, raw descriptors,
account details, and per-transaction dates. It may live only in the vault (records,
rules, rows) and in `/sase_questions` rounds with Bryan. Docs and tests use **fictional
services and amounts** (for example "Tunely $9.99", "Cloudbox $19.99", "Daily Ledger
every 4 weeks"), with synthetic descriptors in Wells Fargo's shape. Phase notes describe
outcomes in counts ("14 records seeded, 2 rows linked"), never as tables of charges.

## Design at a glance

```text
 bank journal (.bob/bank/*.jsonl)        subscriptions/<stem>.md records
 every transaction since May, incl.      status · amount · cadence · pricing
 before_start history (masked desc)      match ("Shows up as") · category · url
              │                                       │
              └──────────────┬────────────────────────┘
                             ▼
              one resolver + one subscription ledger (bob-cli)
              link wins → else match → live beats cancelled →
              longest needle → tie = no link + overlap finding
                             │
     ┌──────────────┬────────┴───────┬───────────────┬───────────────────┐
     ▼              ▼                ▼               ▼                   ▼
 bank import    row JSON         bank check    bank subscriptions   plugin surfaces
 writes         subscription +   findings      report (JSON for     🔁 chips · card linker
 [[subscriptions/<stem>|🔁]]  bank_recurring  (propose, never create)  plugins)  · header strip · album card
```

- **A subscription is a record and a transaction is a bank row.** Records never hold
  charges, and import never creates records.
- **Each record declares how it shows up on the statement.** Its `match` list ("Shows up
  as") holds case-insensitive substrings of the bank description. This is the
  match-strings idea from the request, made optional rather than required (ADJ-2):
  records that cannot be matched safely, such as two products behind one masked
  descriptor, an aggregator like Apple, or a card-billed service, stay matchless and get
  linked by hand.
- **One resolver answers "which subscription is this charge?" for every view.** Import,
  row JSON, findings, the report, and every plugin chip use it. It covers imported rows
  and the journal-only history from before tracking started, so the full journal history
  is attributed as soon as records exist. No backfill rows and no migration are needed.
- **Canonical Markdown stays one token**, `[[subscriptions/<stem>|🔁]]`, the same shape
  as `eat`'s `[[eat/<stem>|🍽️]]`. It works without the plugin, hover-previews, and makes
  each record's backlinks its charge history. The plugin turns it into a pill that is
  quiet when all is well and speaks up only when something needs attention.

### Design calls beyond the research

1. **The resolver also owns a per-record "ledger"** (charges, latest charge, next
   expected, monthly equivalent, price-change and still-charging facts) in the core
   phase. Findings, the report, and row chips only _present_ those facts, so one rule
   never ends up implemented three ways.
2. **A bank rule is a candidate's dismissal.** "A rule matches" already means "known,
   not a subscription", so dismissing a false candidate means adding a `bank.rules`
   entry (`match` plus `payee`/`cat`). That needs no new ignore store and no fake
   cancelled records. Each candidate finding's hint shows both fixes.
3. **Paste-ready fixes are one line.** Candidate and price hints are
   `bob records new|update subscription …` commands that work in a terminal. JSON adds a
   structured `proposal` object so the plugin can prefill its New subscription sheet.
4. **Invalid needles never match.** A needle that fails validation is ignored for
   matching and reported, so a too-broad pattern cannot create false links.
5. **Cancelled records are hidden only while they are quiet.** `bob bank subscriptions`
   hides 🚫 records unless `-a` is given or the record has a finding. A charge after a
   cancel always shows.
6. **Plugin work is serialized** (chips → linker → surfaces) so the generated
   `bob-ledger-tools/main.js` and `styles.css` never need a three-way merge. The bob-cli
   phases fan out in parallel.
7. **Index prettiness lives in the plugin.** Record-index subgroup headings and facets
   are bob-owned Markdown that holds raw bank keys (`tech`) and plain numbers. The
   surfaces phase draws them as `💻 Tech & AI` and `$9.99 a month`, which keeps one
   category taxonomy (`bank.categories`) instead of copying it into the album schema.

## The album

Create it with
`bob albums create bank subscriptions -n subscription -e 🔁 -t Subscriptions -s "What we pay for on repeat, and whether it still earns its keep."`.
Then patch the frontmatter to this schema and run `bob albums sync`. `render_new_album`
deliberately omits `record.fields`, and `albums update` ops cannot author
`index_facets`.

```yaml
parent: "[[bank]]"
type: "[[album]]"
summary: What we pay for on repeat, and whether it still earns its keep.
emoji: 🔁
index_by: category
index_facets: [amount, cadence]
record:
  noun: subscription
  label: Subscription
  plural: subscriptions
  dated_notes: true
  legacy: [recur_id]
  fields:
    status:
      required: true
      values:
        to_cancel: ✂️ To cancel
        to_decide: 🤔 To decide
        active: 💚 Active
        cancelled: 🚫 Cancelled
    amount: { required: true, values: number, min: 0, label: Charged }
    cadence:
      required: true
      values:
        monthly: Monthly
        four_weekly: Every 4 weeks
        quarterly: Quarterly
        yearly: Yearly
    pricing:
      default: fixed
      values: { fixed: 📌 Fixed, variable: 📈 Usage }
    match: { list: true, label: Shows up as }
    category: { suggest: true }
    url: { values: url, label: Manage }
    renews: { values: date }
    ended: { values: date }
```

- `amount` is the **posted charge, tax included**. bob compares it against the bank,
  never against a list price. Bank code parses the decimal text into integer cents. A
  fractional cent makes the amount invalid (reported) and never becomes a float
  comparison.
- `category` holds a **`bank.categories` key**, validated by `bob bank check`, so there
  is one taxonomy. Recurrence is a relationship, not a spending category. There is no
  blanket `subscriptions` category.
- `renews` is an override for what the bank cannot see (trials, card-billed services,
  cancel at period end). Anything with bank evidence derives its next charge from the
  ledger. `ended` drives the charged-after-cancel sentry.
- `recur_id` (legacy) holds the old Zorg `ID::recur_*`, so the legacy phase can map old
  block links to records.
- Status order is the action order: ✂️ act now, 🤔 decide, 💚 fine, 🚫 history.

## The match contract ("Shows up as")

- **Semantics.** A needle matches when it is a case-insensitive substring
  (`contains_folded`, the same as `bank.rules[].match`) of the raw description at import
  time. At read time it matches the masked journal description, or the payee for a typed
  row without journal evidence. A needle with a run of 4+ digits can never match at read
  time.
- **Binding.** Bank linking binds to the album whose record noun is `subscription`,
  resolved through the records registry (`RecordContext::lookup_noun`), never through a
  hard-coded stem. The link text is the album's `emoji` (🔁). `eat` keeps its
  exact-title behavior and tests.
- **Validation** (`bank_subscription_match_invalid`, warning; invalid needles never
  match). A needle is invalid when, after trimming, it is shorter than 3 characters,
  holds a 4+ digit run, or case-insensitively equals a noise phrase. The noise phrases
  are `PURCHASE`, `RECURRING`, `PAYMENT`, `RECURRING PAYMENT`, `AUTHORIZED`,
  `AUTHORIZED ON`, `CARD`, `POS`, `ACH`, `DEBIT`, or a bare aggregator (`GOOGLE`,
  `APPLE`, `PAYPAL`, `AMAZON`, `SQ`, `TST`). Matching is case-insensitive, so needles
  that differ only in case are deduplicated within a record.
- **Resolving one transaction:**
  1. Collect every record with a matching valid needle, across all statuses. Split them
     into live (✂️ 🤔 💚) and cancelled (🚫). Live beats cancelled, but a match against
     cancelled records alone still attributes, because a charge on a cancelled record is
     the most useful alert in the system.
  2. One record in the winning group: it wins.
  3. Several: the one record whose matched needle is **strictly longest** wins.
  4. Still tied: **no attribution**. The provenance lists the tied records, and `check`
     reports the overlap. Never break a tie by file order, price, or history.

## Attribution and the subscription ledger (core facts)

**Attribution of one charge.** Charge X belongs to record R when:

1. its row text links `subscriptions/R` (source `link`; an explicit link always wins,
   whatever the description says), or
2. R wins the resolution above over its description (source `match`).

The universe is every journal transaction, read through its row when the row exists,
plus ID-less typed rows dated **outside** every import-coverage window. Typed rows
inside covered windows are left out because they may duplicate unmatched journal
transactions. Pattern inference never applies to inflows (unless the row carries
`[cat:: refund]`), `transfer`/`income`/`cash` rows, Wells Fargo built-in
transfers/cash/fees, or zero amounts. Refund inflows net against charges in totals, but
they never count as a "charge" for latest, price, or interval facts. The resolver
returns provenance: `{record, source: link|match, needle, tied: [records]}`.

**Per-record ledger** (computed once and read by every consumer):

| Fact             | Rule                                                                                                                                                                                                                                                          |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `charges`        | Attributed outflows, oldest first, with `{date, cents, path?, line?, id?, journal_only}`                                                                                                                                                                      |
| `latest`         | The newest charge, or null                                                                                                                                                                                                                                    |
| `next_expected`  | `renews` when set; otherwise `latest` + one cadence step (monthly and quarterly by calendar month with month-end clamping, four-weekly +28 days, yearly +1 year); otherwise null                                                                              |
| `monthly_cents`  | Fixed: monthly ×1, four-weekly ×13/12, quarterly ÷3, yearly ÷12, using integer math and rounding half-up. Variable: the trailing 3-month average of attributed charges, or `amount` when there are none. Cancelled records count 0.                           |
| `price_change`   | Fixed monthly or four-weekly: the last two charges are equal to each other and differ from `amount`. Fixed quarterly or yearly: the latest charge differs from `amount`. Variable: never. Gives `{from_cents, to_cents, since}`. Updating `amount` clears it. |
| `still_charging` | Status ✂️, and `latest` falls within 1.5 cadence steps of today                                                                                                                                                                                               |
| `after_cancel`   | Status 🚫, with charges dated after `ended` (any charge when `ended` is empty; the hint teaches setting `ended`)                                                                                                                                              |
| `timeline`       | Up to 12 cadence buckets ending today, each `charged`, `missed` (import coverage spans the bucket with no charge), `uncovered`, or `upcoming`                                                                                                                 |

**Row JSON (additive, in `list`, `report`, and `import` rows):**

- `subscription`: `null` or
  `{stem, path, title, status, status_label, cadence, pricing, amount_cents, source, needle, drift_cents, price_change, state}`.
  `drift_cents` is the row's outflow minus `amount` for fixed records (null for
  variable). `state` is `ok`, `drift`, `usage`, `to_cancel` (this row is the latest
  charge and `still_charging` holds), or `after_cancel`. `source: "match"` on a written
  row means "matches but carries no link". The chip draws that as a ghost.
- `bank_recurring`: true when the journal description carries Wells Fargo's
  `RECURRING PAYMENT` marker. The cleaner keeps stripping it from payees, and bob now
  reads it before cleaning.
- `subscription_candidate`: true for an outflow with `bank_recurring` that has no
  attribution and no rule match.
- `suggestion.source` gains `subscription`, used when the attributed record's category
  fills an uncategorized row. Older plugins already decode an unknown source as "no
  suggestion".

A missing album means every field is null or false and nothing else changes. An invalid
or unreadable album adds one `bank_subscription_album_invalid` warning (naming
`bob albums check`) and disables subscription facts for that read.

## Import

> [!decision] payee_source = record_title
>
> Fields fill in per field with this precedence: rule → subscription → learned →
> built-in → cleaned. A unique subscription match sets the payee to the record
> **title**, so a truncated descriptor becomes the service's name with no teaching, and
> renaming the record renames future rows.

> [!decision] payee_source = learned_first
>
> Payee precedence is rule → learned → subscription title → built-in → cleaned. A payee
> Bryan hand-edited on an earlier acknowledged row wins, and the title fills only
> merchants with no history. Category and link precedence are unchanged.

- **Link precedence:** rule link → subscription link → learned link (skipping any link
  into the subscription album) → restaurant title. `learn()` copies the first link of a
  past row as-is, so without this skip a stale copy would outrank a live record edit
  ("delete a needle and the old link keeps coming back"). Non-subscription learned links
  keep today's behavior byte for byte.
- **Category fill-down:** with no rule `cat`, a uniquely matched record's `category`
  fills the row when it names a declared bank key (hidden keys count as declared).
- **Ownership is unchanged.** Import writes a row once. Matching a typed row still
  appends only `^bank-<hex>`. A row written before its record existed stays unlinked in
  text: read-time attribution still counts it, and the plugin offers a one-click link.
  Replay stays byte-idempotent, and import never creates records.
- **Guarded inputs.** Generalize `load_restaurants` (`src/native/bank/import_run.rs`)
  into one guarded record-link registry read. It returns restaurants and subscriptions,
  with album declaration and record bytes and folder membership as `ReadDep` preimages.
  An invalid subscriptions album degrades to "no subscription links this run" with one
  bounded `bank_import_subscriptions_unavailable` warning. Unreadable membership
  refuses, as `bank_import_registry` does today.
- **Summary**, printed only when there is news:
  `🔁 2 rows linked to subscriptions · 1 possible new subscription · 1 price change — bob bank check`.
  JSON adds `subscriptions {linked, candidates, price_changes}`, counted over this
  batch's new rows.

## Findings (`bob bank check`; propose, never create)

| Code                                  | Severity | Trigger and hint                                                                                                                                                                                                                                  |
| ------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bank_subscription_cancelled_charge`  | warning  | Ledger `after_cancel`. One finding per record names the newest offending charge. Hint: the Manage `url`, or `-s ended=YYYY-MM-DD` when `ended` is empty.                                                                                          |
| `bank_subscription_to_cancel_charged` | info     | Ledger `still_charging`: "still paying $X on Mon D", plus the Manage `url`                                                                                                                                                                        |
| `bank_subscription_price`             | info     | Ledger `price_change`. Hint: `bob records update subscription <stem> -s amount=<new> -n '<Mon D>: price changed to $<new>'`                                                                                                                       |
| `bank_subscription_candidate`         | info     | See below. Hint: a `bob records new subscription '<Title>' -s status=to_decide -s amount=… -s cadence=… -s match=… -s category=…` line, or a `bank.rules` entry to dismiss. JSON adds `proposal {title, amount_cents, cadence, match, category}`. |
| `bank_subscription_match_invalid`     | warning  | A needle fails validation; names the record, the needle, and the reason                                                                                                                                                                           |
| `bank_subscription_overlap`           | info     | Two live records share a folded needle, or an observed transaction stayed tied                                                                                                                                                                    |
| `bank_subscription_category_unknown`  | warning  | `category` is not a declared bank key; suggests the nearest (`suggest_name`)                                                                                                                                                                      |
| `bank_subscription_amount_invalid`    | warning  | `amount` is missing, negative, or has fractional cents                                                                                                                                                                                            |
| `bank_subscription_album_invalid`     | warning  | The album cannot be read; points at `bob albums check`                                                                                                                                                                                            |

**Candidates.** Group outflows by `merchant_key` over the masked description. Exclude
every transaction that has an attribution (any status), matches a bank rule, is a
built-in transfer/cash/fee, or is not an outflow. A group is a candidate when it has the
`RECURRING PAYMENT` marker on any charge (this catches the first charge, including
yearly), **or** when its consecutive charges form a run of at least 3 at 26–33 days, at
least 2 at 340–390 days, or at least 3 at 85–95 days. A qualifying run also needs an
amount spread of at most 25% of its median. Rank by annualized cost and show the top 10,
followed by one `… and N more` info line. Proposals take the title from the cleaned
merchant, the cadence from the interval (monthly when only the marker is known), the
category from the latest row's `cat` or its suggestion, and a needle that passes
validation. Required negative fixtures (synthetic): a daily coffee shop with identical
amounts, identical ATM fees, $25 pharmacy charges at irregular gaps, a barber every ~80
days, and every rule-matched obligation. Required positive fixtures: a 28-day biller, a
marker-only first yearly charge, and a descriptor that changes partway through the
history.

## `bob bank subscriptions`

`bob bank subscriptions [-a] [-b DIR] [-f human|json]` is read-only. Help, completion,
and docs follow `cli_rules` (options alphabetical, short aliases, subcommands
alphabetical). The `bob bank` membership line becomes "commands over bank rows, bank
month notes, their import journal, and the subscription records rows link to".

Human output (fictional data). Groups follow status order, the `💚 Active` group splits
by bank category when it is large, and money is right-aligned and tabular. Each status
keeps its emoji and word, so color is never the only signal:

```text
🔁 Subscriptions   ≈ $136.39 a month · ≈ $1,636.68 a year
   1 to cancel · 1 to decide · 2 findings — bob bank check

✂️  To cancel
    Daily Ledger     $6.00  every 4 weeks  last Nov 10   next ≈ Dec 8    still charging
🤔  To decide
    Cloudbox        $19.99  monthly        last Nov 3    next ≈ Dec 3
💚  Active
    Codecraft      $100.00  yearly         last Feb 14   next ≈ Feb 14
    Tunely           $9.99  monthly        last Nov 21   next ≈ Dec 21   ↑ $10.99 since Oct
```

JSON (`schema_version: 1`) holds `album {stem, path, title, emoji} | null`,
`totals {monthly_cents, yearly_cents, by_status {<key>: {count, monthly_cents}}}`, and
`subscriptions[]`. Each entry has the record identity, status and label, cadence,
pricing, amount, category with emoji and label, `match`, `url`, `renews`, `ended`,
`monthly_cents`, `charges {count, total_cents, latest}`, `next_expected`,
`price_change`, `still_charging`, `after_cancel`, `timeline[]`, and `findings[]` codes.
The envelope also carries the subscription `findings[]` in check's shape, plus `issues`.
This is the one data source for the linker, the header strip, and the album card.

## Rendering (plugins)

**Row chip** (Live Preview, Reading view, queue card). It has the same geometry as the
existing bank chips: at most one extra glyph and about 14 characters after the 🔁, state
carried by words and signs (never by color alone), raw syntax under the cursor, and
source bytes untouched.

| State (from row JSON)    | Chip                                                                                                                          |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `ok`                     | `🔁`, muted. It grows to `🔁 Cloudbox` only when the payee does not name the record (case-insensitive containment either way) |
| `drift`                  | `🔁 +$1.00`, muted; amber once `price_change` fires                                                                           |
| `usage`                  | `🔁 usage`, muted                                                                                                             |
| `to_cancel`              | `🔁 ✂️ still charging`, amber; click opens the Manage `url` and modifier-click opens the record                               |
| `after_cancel`           | `🔁 🚫 after cancel`, warning tone, never red-for-spending                                                                    |
| matched, unlinked        | dashed ghost `🔁`; click inserts `[[subscriptions/<stem>\|🔁]]` as one editor transaction                                     |
| `subscription_candidate` | ghost `⟳ subscription?`; click opens the linker on that row (wired in `linker`)                                               |

Tooltip: `Tunely · 💚 Active · $9.99 a month · latest charge Nov 21`. Accessible name:
`Open Tunely subscription`, with a visible focus ring. Without the plugin, the wikilink
still shows 🔁 and navigates.

**Category card.** `/` (it collides with no category hotkey) opens an inline
subscription picker in the card. The query filters by title and needle. The ranking puts
records whose needles match this row first, then sorts by amount proximity, then live
before cancelled. Enter commits `[[subscriptions/<stem>|🔁]]`, plus `[cat::]` from the
record when the row has none, as one atomic undo step through the card's existing
writer. Escape returns to the tiles. A bottom action, **+ New subscription…**, opens a
compact sheet prefilled from the row and any candidate `proposal`. The sheet holds
title, status 🤔, amount, cadence, needle, and category. It runs
`bob records new subscription …` and then links the row. A context strip above the tiles
shows the row's subscription
(`🔁 Tunely · 💚 Active · $9.99 a month · ↑ $1.00 vs record`).

**Album surfaces.** On `subscriptions/*.md`, a header strip reads
`$9.99 a month · ≈ $120 a year · last Nov 21 · next ≈ Dec 21`, followed by a 12-dot
timeline (filled = charged, ring = missed, faint = uncovered or upcoming, amber dot =
price changed) and any finding as a sentence. On `subscriptions.md`, a `bob-bank` fence
with `view: subscriptions` draws the summary: the monthly and yearly estimate, labeled
as **an estimate of commitments, not bank spend**, status counts, the top 5 by monthly
equivalent as bars, and the findings count. The records index shows category subgroup
headings with bank emoji and labels, amounts as `$9.99 a month`, and a quiet
`next ≈ Dec 21` chip. Every surface is read-only and computed from
`bob bank subscriptions -f json`. Nothing is written to notes.

## Phase `seed` — Seed the album from a reviewed bank history

Vault work only, with Bryan in the loop. Read the report sections "What the transactions
say", "Recommended design", "Seed roster", "Open questions for Bryan", and "Delivery
plan" (P0). Also read `obsidian.md` reference memory and `docs/records.md`.

1. **Evidence.** Write a throwaway, read-only script **outside every repository** (for
   example under `/tmp`). It groups `~/bob/.bob/bank/*.jsonl` transactions (every
   outcome, including `before_start`) by merchant key and reports count, amount series,
   intervals, the `RECURRING PAYMENT` marker, and descriptor variants. Use it to refresh
   the report's seed roster. The roster is a hypothesis, the journal wins on amounts,
   and anything that moved since 2026-11-15 gets updated. Also cross-check
   `old_recur.md`'s active rows, because annual and card-billed charges may not appear
   in checking.
2. **One question round** through `/sase_questions`. Cover every 🤔 row and the report's
   seven open questions, refreshed against the new evidence. Phrase each question with
   the service, the observed pattern, and the default you will use. This is the only
   round, so batch everything into it.
3. **Album.** Create the album with the command in [The album](#the-album), patch the
   schema exactly as shown, then run `bob albums check` (clean) and `bob albums sync`.
4. **Records.** Write one `bob records new subscription '<Title>' -s …` per confirmed
   service. Statuses follow the answers; anything unconfirmed is `to_decide`. Needles
   must follow [the match contract](#the-match-contract-shows-up-as) (keep every
   observed descriptor variant). Use a bank category key, the Manage URL when known, a
   dated note such as "Seeded from the bank journal: N charges since May 2026", and
   `ended` on cancelled seeds. Then add `recur_id` by hand for every record that came
   from an `old_recur.md` row, inserted before `created`. Legacy-only rows with no
   checking evidence get the note "no checking charge since May 2026" so the audit
   closes each one deliberately. Finish with a clean `bob records check subscription`.
5. **Obligation rules.** Re-read `bank.md` right before editing (another task is
   improving bank frontmatter). Add `bank.rules` entries (`match`, `payee`, `cat`) for
   the non-discretionary recurring charges the report lists under "Do not seed — write
   `bank.rules` instead", adjusted by Bryan's answers. Skip match texts with a run of 4+
   digits. `bob bank check` must report no `bank_config_invalid`.
6. **Live rows.** Hand-link each existing bank row that clearly belongs to a seeded
   record by appending `[[subscriptions/<stem>|🔁]]` before its `^bank-` ID. This is a
   one-time edit on Bryan's behalf. Leave the payee wording alone.
7. Publish with `bob vault-sync`. Record counts, never data, in a phase note.

Done when: the album and its index are current, every record passes `bob records check`,
`bob albums check` is clean, Bryan's answers are reflected, and `bob bank check` shows
no new errors.

## Phase `core` — Subscription registry, match resolver, and ledger

bob-cli, `src/native/bank/`. Plan the module layout first (size `large`). A
`subscriptions/` submodule (`registry`, `matching`, `attribution`, `ledger`) is the
expected shape. Read `docs/bank.md`, `docs/records.md`, and the report's "The match
contract" and "Read-time attribution" sections.

- **Registry.** Resolve the album by noun through `RecordContext`. Parse records into
  typed `SubscriptionRecord`s (stem, path, title, status, amount cents, cadence,
  pricing, valid needles plus invalid ones with reasons, category, url, renews, ended)
  and collect per-record diagnostics. Two load paths:
  - a read-time loader for `list`/`report`/`check`/`subscriptions` that degrades as
    described above;
  - the guarded import loader, which generalizes `load_restaurants` and returns both
    restaurants and subscriptions with their `ReadDep`s and directory guards. Keep
    restaurant behavior and its tests unchanged.
- **Matching** exactly as in [the match contract](#the-match-contract-shows-up-as),
  including validation reasons and tie provenance.
- **Attribution and ledger** exactly as in
  [Attribution and the subscription ledger](#attribution-and-the-subscription-ledger-core-facts).
  Use integer cents everywhere and dates from the journal and rows. `BOB_NOW` sets
  "today".
- **Row JSON** additions in `list`, `report`, and `import` rows, plus the `subscription`
  suggestion source.
- **Tests** (synthetic fixtures): link beats match; live beats cancelled-only;
  cancelled-only still attributes; longest needle wins; an exact tie yields no
  attribution with provenance; descriptor drift where two needles both match; a 4+ digit
  needle never matches at read time; journal-only `before_start` charges attribute; a
  covered ID-less typed row is excluded; refunds net, while transfers, income, and zero
  rows never infer; each cadence's `next_expected` (month-end clamping included);
  integer `monthly_cents`; the `price_change` rule including a one-off proration that
  must stay quiet; `still_charging`, `after_cancel`, and `timeline`; a missing album is
  a no-op; an invalid album warns.
- **Docs.** Add a `## Subscriptions` section to `docs/bank.md` covering the album
  contract summary, the match contract, attribution, and the ledger. Extend the row JSON
  in `docs/bank.md` and `docs/reference/json-outputs.md#bank-reports`. Read `docs.md`
  and `cli_rules.md` reference memory first, then run `just docs-check`.
- Gate: `just all` and `just docs-check`. Use Conventional Commits under `feat(bank)`.
  Never edit the Cargo version or `CHANGELOG.md`.

## Phase `legacy` — Retire old_recur.md into a legacy_recur notebook

Vault work only. Follow the `eat.md` → `legacy_eat.md` recipe: a `type: "[[notebook]]"`
note with a one-line `summary` and Zorg rows as note bullets that keep their exact text
and `^z-` IDs.

1. **Baseline.** Count unresolved wikilinks and block references vault-wide with a
   read-only script outside every repository. Save the list of every inbound `old_recur`
   link form (`[[old_recur]]`, `[[old_recur|…]]`, `[[old_recur#^z-…|…]]`, embeds, and
   Markdown links). There are about 273 links in about 181 files today.
2. **Notebook.** Rename `old_recur.md` to `legacy_recur.md` with `git mv` so history
   follows. Rewrite it as `parent: "[[cash]]"`, `type: "[[notebook]]"`,
   `summary: "Zorg-era recurring-charge table, moved here from old_recur.md on <date>."`.
   Use an H1 and `##` sections for the active and inactive tables. Drop the Zorg
   generator frontmatter and the align-tables line. Keep **every** `^z-` block ID and
   every row's text and inner links. Run `bob notebooks sync`, then
   `bob notebooks check`.
3. **Inbound links.**
   - A link to a row whose `ID::recur_*` equals a record's `recur_id` becomes
     `[[subscriptions/<stem>|<Title>]]`, so each record's backlinks show about two years
     of `@BANK` price history.
   - A link to any other row becomes `[[legacy_recur#^z-…|<same alias>]]`.
   - A bare or aliased note link becomes `[[legacy_recur…]]`.
   - Dry-run the rewrite first and review the per-target counts.
4. **Audit task.** In `cash.md`, the `^audit-monthly-subscriptions` task text becomes
   `Audit monthly [[subscriptions]]`. Change only the words; fields and ID stay put.
   Leave `bob.md ^bank-subscriptions` for Bryan.
5. **Verify.** Unresolved links after the rewrite must not exceed the baseline. Zero
   `old_recur` references remain. Every `^z-` ID from the old file exists in
   `legacy_recur.md`. Publish with `bob vault-sync` immediately to shrink the conflict
   window.

## Phase `import` — Import links new rows to their subscription

bob-cli. Implement [Import](#import). The changes are in `naming.rs::resolve_name` (new
subscription tier and fill-down per the `payee_source` decision), in `import_plan.rs`
(the learned-link skip and the subscription lookup beside `restaurant_link`), in the
guarded loader in `import_run.rs`, and in the summary and JSON.

Tests (synthetic):

- a new row links and takes the title and category;
- a descriptor that changed partway still links through its second needle;
- a cancelled-only match links;
- an ambiguous tie writes no link;
- a rule link beats a subscription;
- a learned subscription link is never propagated, while a learned restaurant link still
  is;
- a typed-row match appends only the ID;
- replay is byte-idempotent;
- a changed record or a new record file makes the plan stale (guarded `ReadDep`);
- an invalid album warns and imports without links;
- `-d` equals apply.

Update the `docs/bank.md` Naming section, the import JSON, and "Current limits" (drop
subscriptions). Gate: `just all` and `just docs-check`.

## Phase `check` — Subscription findings in bob bank check

bob-cli. Implement [Findings](#findings-bob-bank-check-propose-never-create) by
presenting ledger facts and adding the candidate detector, with every listed positive
and negative fixture. Keep `check` exit semantics: warnings and info never fail the run.
Human output groups subscription findings under one `🔁 Subscriptions` heading after the
note groups. Docs: the findings table and JSON `proposal` in `docs/bank.md`. Gate:
`just all` and `just docs-check`.

## Phase `report` — The bob bank subscriptions command

bob-cli. Implement [`bob bank subscriptions`](#bob-bank-subscriptions). The work covers
the clap wiring, alphabetical placement, help text, completion, `just docs-gen`, human
output with color under `NO_COLOR` and pipe rules, the JSON contract, and the teaching
empty state (no album yet points at `bob albums create`). Write a short guide,
`docs/guides/audit-subscriptions.md`, for the monthly audit (open the album, run
`bob bank subscriptions`, act on ✂️ and 🤔, apply price hints) and link it from
`docs/bank.md` and the guides index. Gate: `just all`, `just docs-gen`, and
`just docs-check`.

## Phase `chips` — 🔁 chips on bank rows

bob-plugins `bob-ledger-tools`. Open the repo with `/sase_repo` and follow its
`AGENTS.md`: edit `src/` fragments, run `npm run build`, keep fragments ≤1000 lines, and
run `npm test` and `npm run docs:check`. Implement the row-chip table from
[Rendering](#rendering-plugins) in Live Preview (the `241o`/`241p` bank marks), Reading
view (`241q`), and the queue card row renderer (`241m`). The queue card currently draws
only money, payee, and category. Extend the bank decoder (`241h`) for `subscription`,
`bank_recurring`, `subscription_candidate`, and the `subscription` suggestion source.
Older bob output without these fields must render exactly as today. The ghost chip's
link insertion is one editor transaction. The candidate ghost shows a tooltip only
("Wells Fargo marks this as recurring — press / in the Category card"), and `linker`
wires its click. Check light and dark themes, narrow panes, keyboard focus, and the
plugin-disabled fallback. Document the chips in the plugin docs, then run
`bob plugins sync`.

## Phase `linker` — Category card subscription linker

bob-plugins `bob-ledger-tools` (`245`–`248` Category card). Implement the `/` picker,
the context strip, and the **+ New subscription…** sheet from
[Rendering](#rendering-plugins). The picker list comes from
`bob bank subscriptions -f json`. Creation runs `bob records new subscription` through
the plugin's existing spawn path, with arguments passed as an argv array and never
interpolated into a shell. After creation it links the row in the same card gesture.
Wire both ghost chips: a matched-unlinked ghost inserts the link, and the candidate
ghost opens the card's picker on that row. Keyboard: `/` opens, arrows and Ctrl+N/P
move, Enter commits, Escape backs out, and the card's lock and chord guards apply.
Document the hotkey in the plugin reference docs. Gate: `npm test` and
`npm run docs:check`, then `bob plugins sync`.

## Phase `surfaces` — Record header strip and album summary card

bob-plugins `bob-ledger-tools` (album header and records fragments `241r`–`241v`, plus
the `bob-bank` fence). Implement the header strip with its timeline, the
`view: subscriptions` fence card, and the index-row formatting from
[Rendering](#rendering-plugins). Add the fence to `~/bob/subscriptions.md` (below the
intro, above `## Records`) and publish with `bob vault-sync`. Unknown-view handling in
older plugins must stay quiet. Document everything in the plugin's albums and bank docs.
Gate: `npm test` and `npm run docs:check`, then `bob plugins sync`.

## Phase `acceptance` — Vault acceptance, memory, and closing bob-cli-kt

1. Confirm the bob on this host includes the bob-cli phases: master is installed (the
   hosts track master), or run `just install` from a checkout at current master. Then
   run on the real vault:
   - `bob bank check` reports **zero** `bank_subscription_candidate` findings. Clear any
     that remain with a record or a `bank.rules` entry, asking Bryan through one
     `/sase_questions` round only when the right call is unclear.
   - There are no errors and no invalid or overlap warnings.
   - `bob bank subscriptions` totals look plausible.
   - `bob albums check` and `bob records check subscription` are clean.
2. Hand-link any bank row written since the seed whose attribution is `source: match` (a
   one-time edit on Bryan's behalf).
3. Re-import only if a fresh export is available. Otherwise, rely on the import phase's
   fixture proof.
4. Memory. Edits follow `/sase_memory_write`, then `sase memory init`, only for accepted
   decisions:

   > [!decision] memory_decision_record
   >
   > Add `decisions:subscriptions-own-their-bank-match`, saying that a subscription
   > record declares how it shows up (`match`) and that one resolver attributes for
   > every view. The precedence is rule → subscription → learned (never a subscription
   > link) → restaurant. A tie writes no link, and bob proposes but never creates
   > records. It cites this plan and the research report, rejects learned-naming
   > propagation and exact aliases, and reopens on a bank feed with stable merchant IDs.

   > [!decision] memory_glossary_term
   >
   > Add `glossary:subscription-record` (aka subscription, Shows up as). The term means
   > a record in the `subscriptions` album under `bank` for one discretionary
   > keep-or-cancel recurring charge. Its `match` needles link bank rows as
   > `[[subscriptions/<stem>|🔁]]`. Obligations stay `bank.rules`. It is not a bank row,
   > task, or notebook.

5. Docs sweep. `docs/bank.md` must no longer list subscriptions under "Current limits",
   and cross-links among `docs/bank.md`, `docs/records.md`, and the guide must resolve.
   Run `just docs-check`.
6. Record the ADJ list as adopted and any deviations in a note on `bob-cli-kt`, then
   `sase bead close bob-cli-kt --note "<what you verified>"`. Never close the epic bead.

## Close criteria for bob-cli-kt

- The album and its reviewed seed exist.
- New imports link safely and fill in the title and category.
- The candidate, price, and cancelled-charge findings work, and the seeded vault is
  candidate-clean.
- Live rows are linked.
- The audit task points at the album.
- The de-zorg preserved every block ID and every inbound link.
- `bob bank subscriptions`, the chips, the linker, and the album surfaces ship.
- Docs and accepted memory have landed.
- The CLI and plugin gates pass.
