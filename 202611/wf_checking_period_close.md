---
tier: epic
title: Period closing balance for WF Checking on period notes
goal: 'Every week, quarter, ISO-year, and all-time note shows the checking account''s
  exact posted balance at the end of that period (or, honestly labeled, as of the
  last imported day). bob computes the balance live from bank evidence, and it is
  never written into notes. Daily notes stay untouched.

  '
phases:
- id: resolver
  title: Historical balance resolver and check findings
  depends_on: []
  size: medium
  description: 'resolver: add the pure bank/balance.rs resolver (coverage spans, final
    days, batch attribution, anchor segments, cross-checks, and period resolution)
    plus the two new `bob bank check` balance findings and their docs rows, with the
    invariant test suite.'
- id: report
  title: balances[] in bob bank report, human line, and docs
  depends_on:
  - resolver
  size: small
  description: 'report: wire the resolver into `bob bank report` for every period
    selector as an additive `balances[]` array and a human Balance line, then document
    the Closing balances contract and verify read-only against the live vault.'
- id: readout
  title: Bank Charts header balance readout in bob-ledger-tools
  depends_on:
  - report
  size: medium
  description: 'readout: decode `balances[]` and paint the hero''s resting BALANCE
    readout in the Bank Period Charts header, including the header-only quiet-period
    state, tooltip, aria, and activation. Update both repos'' docs, review preview
    screenshots, and sync the plugins.'
- id: vocabulary
  title: Bank Closing vocabulary and decision record
  depends_on:
  - readout
  size: small
  description: 'vocabulary: apply the accepted memory decisions (Bank Closing glossary
    strand, Stats Strip cross-reference, decision record) and republish memory.'
decisions:
  memory_bank_closing:
    ask: Add a "Bank Closing" glossary strand defining the period closing balance?
    memory:
    - glossary:bank-closing
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
  memory_stats_strip_link:
    ask: Add one Stats Strip glossary sentence pointing at the Bank Closing in the
      charts header?
    memory:
    - glossary:stats-strip
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
  memory_close_decision:
    ask: Record a decision that period closes are posted balances drawn at read time,
      never written?
    memory:
    - decisions:period-closes-are-drawn-posted-balances
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
proposed_by: bbugyi200.apollo.research.50.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 15:40:30
status: wip
bead_id: bob-cli-mg
---

- **PROMPT:** [prompts/202611/wf_checking_period_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/wf_checking_period_close.md)
- **BEAD:** [bob-cli-mg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mg/README.md)

# Plan: Period closing balance for WF Checking on period notes

## Context

Bryan wants to see his Wells Fargo checking balance at the end of each period on that
period's note. For example, the 2026-11-15 balance should appear on `2026/2026-W46.md`.
It must never appear on daily notes. The design brief is intuitive, reliable, and
beautiful. Bryan accepted every recommendation in the consolidated research report. Read
it first:

```bash
sase artifact read "research:202611/wf_checking_period_close/wf_checking_period_close.md" "Closing-balance design context"
```

This plan adopts that report's core and tightens it in a few places, named in
[Design choices beyond the research](#design-choices-beyond-the-research):

- **The number.** It is the posted (ledger) balance at the end of a day. It is never the
  app's "available" balance, never a sum of accounts, and never the latest balance
  pasted onto an old week.
- **Drawn, never written.** bob computes every cent from `.bob/bank/<host>.jsonl` at
  read time. Notes, frontmatter, Review drafts, `bob periods brief`, `bob stats`, and
  Mac surfaces never hold it.
- **The home.** The number goes in the existing right-aligned header slot of the Bank
  Period Charts panel. That host mounts only on week, quarter, ISO-year, and all-time
  `bob-stats-strip` fences, so "not on daily notes" holds by construction.
- **The surfaces.** bob owns the math. The plugin only formats and paints. The CLI
  answers any single day (`bob bank report -D 2026-11-15`).

### What Bryan sees

The W46 note on Monday morning, when the latest import ends Friday:

```text
│ ▾ BANK CHARTS  ▪▪                      BALANCE  $1,234.56 · as of Fri Nov 13
│ IN / OUT  …                                    OUT BY CATEGORY  …
```

The same note a day later, once an import whose window passes Sunday proves Sat–Sun
empty. The amount never changes; only the claim gets stronger:

```text
│ ▾ BANK CHARTS  ▪▪                                         BALANCE  $1,234.56
```

A quiet week (no counted money and no waiting rows, such as a transfer-only week or
pre-tracking backfill) shows only the header line. Today these weeks render nothing:

```text
│ BANK                                                      BALANCE  $2,345.67
```

All amounts in this plan are synthetic. See [Privacy](#privacy).

## Design choices beyond the research

The phase workers should treat these as decided:

1. **The balance is its own element, not text in the hover readout.** A new
   `.bob-bank-period-balance` node follows `.bob-bank-period-readout` in the header.
   While the hover or focus readout has text, CSS hides the balance
   (`.bob-bank-period-readout:not(:empty) ~ .bob-bank-period-balance { display: none }`).
   Mouse-leave and Escape bring it back with no JS restore logic. The aria-live readout
   stays the only live region, so the balance never re-announces. The balance can also
   be its own focus stop and link.
2. **It has a label.** A faint small-caps `BALANCE` precedes the amount. A bare dollar
   figure next to `BANK CHARTS` could be misread as Net or Out.
3. **Dates read in full: `as of Fri Nov 13`, not `as of Fri 13`.** Add the year only
   when it differs from the wanted date's year.
4. **No Δ chip on the readout.** The close comes from bank evidence, and ledger-row
   drift does not change it. An account `delta` status appears as one tooltip line only,
   which keeps the readout calm.
5. **Anchor cross-checks degrade by segment, not by span.** A disagreement between two
   bank-reported balances hides only the days strictly between them. One bad statement
   can no longer blank 29 weeks of history. With consistent anchors this gives the same
   result as reconstructing from the latest anchor.
6. **Same-day snapshots are provisional.** A window day on or after its import's local
   date may still get postings, so it can never settle a closed period. A provisional
   anchor is never cross-checked once a later import supersedes it. This prevents false
   `unverified` blackouts from intraday downloads.
7. **CSV handling is precise.** Import and `bank.txn` events from one `bob bank import`
   run share a single frozen `ts` (`import_plan.rs` `now_stamp`). That makes a
   transaction's source attributable:
   - CSV-only batches never enter balance arithmetic. OFX windows already list every
     posting.
   - Only a batch that mixed CSV and OFX for the account can taint a span.
8. **Activation goes only where it is useful.** It opens the as-of day's heading in the
   bank month note, or that note's first line when the heading is missing. When no month
   note exists, as on pre-tracking backfill weeks, the balance is inert rather than
   opening the unrelated `bank.md`.
9. **Settling is strict, with no weekend heuristic.** The research's guarded weekend
   rule stays an opt-in follow-up.

## Resolver rules (phase `resolver`)

The resolver applies these rules to each importable, non-cash account, using the
`BankConfig`, the `JournalSnapshot`, `today`, and the configured timezone.

1. **Health gate.** Journal `io_errors` or `malformed` lines make every account
   `unverified` (`journal_unreadable`). A journal contradiction on one of the account's
   keys makes that account `unverified` (`journal_contradiction`). This mirrors
   `status.rs`.
2. **Batch attribution.** A batch is `(host, ts)`. A transaction's sources are the
   `source` values of this account's import events in its batch:
   - CSV only: the transaction never counts.
   - CSV and OFX: the transaction is ambiguous.
   - OFX only, or no import event (a recovery): the transaction counts.
3. **Coverage.**
   - `Covered(d)`: some OFX import window `[start, end]` contains `d`. Windows are not
     clipped to the configured `start`, so `before_start` history counts.
   - `Final(d)`: some OFX window containing `d` was imported on a local date after `d`.
     Factor one `import_local_date(&ImportEvent)` helper and use it here and for
     `status.rs`'s `last_import`, so both use the same configured-timezone logic.
   - A span is a maximal run of consecutive covered days.
   - An ambiguous transaction posted inside a span makes that span `unverified`
     (`csv_overlap`).
4. **Anchors.**
   - An anchor is an OFX import with `balance_cents` and `balance_asof` where the as-of
     date lies inside a span.
   - An anchor is provisional when its as-of date is on or after its import's local
     date.
   - For each as-of date, the latest-`ts` observation is the effective anchor.
   - Among earlier same-date observations, ignore provisional ones. A non-provisional
     one that disagrees is a dispute.
   - A span with no anchor is `unverified` (`no_anchor`).
5. **`B(d)`, the posted balance at the end of day `d`.** It is defined for `d` in
   `[span.start − 1, span.end]`.
   - Take the nearest effective anchor on or after `d` in the span and subtract the
     counted cents posted in `(d, anchor.as_of]`.
   - After the span's last anchor, add the counted cents posted in `(anchor.as_of, d]`.
   - Use journal cents only: never vault-row cents, categories, or transfer filters. A
     transfer out of checking moves checking. `before_start` transactions count.
6. **Cross-checks.**
   - For consecutive effective anchors `a < b`, `B` computed from `b` at `a.as_of` must
     equal `a`'s balance. A mismatch makes the days strictly between them unverified
     (`anchor_mismatch`). The anchor days stay exact.
   - A dispute makes its date unverified, along with the days strictly between its
     neighbouring anchors.
   - Each mismatch or dispute emits one finding.

**Period resolution** for a report period `P`:

| Condition (checked in order)                                 | `state`                         | `cents`    |
| ------------------------------------------------------------ | ------------------------------- | ---------- |
| `P.start > today`                                            | `none` (`reason: future`)       | null       |
| no `as_of` found (below)                                     | `none` (`reason: pre_coverage`) | null       |
| `as_of` in an unverified span or range                       | `unverified` (with reason)      | null       |
| closed `P`, `as_of < P.start`, no covered day after `wanted` | `awaiting_import`               | null       |
| closed `P`, `as_of < P.start`, coverage resumes later        | `gap`                           | null       |
| open `P`                                                     | `so_far`                        | `B(as_of)` |
| closed `P`, `as_of == wanted`                                | `settled`                       | `B(as_of)` |
| closed `P`, `as_of < wanted`                                 | `as_of`                         | `B(as_of)` |

The inputs to that table:

- **`wanted`** is `today` for an open period (`so_far`) and `P.end` otherwise.
- **`as_of`** is the latest day on or before `wanted` that is `Final` (closed `P`) or
  `Covered` (open `P`).
- **`stale`** is true only for `so_far` when `today − as_of > bank.stale_after` days.
- **`open`** is `{date: P.start − 1, cents: B(P.start − 1)}`, and `change_cents` is
  `cents − open.cents`. Both are null unless that date is verified, `as_of ≥ P.start`,
  and `P` is not all-time.
- **`hero`** is the first `kind: checking` account in config order. Never sum accounts.

**Invariants** (each one a test):

- When `as_of` is the latest anchor date and the status is `balanced`, `cents` equals
  `accounts[].balance_cents`.
- `B(d) − B(d−1)` equals the counted cents posted on `d`.
- A closed period never takes a balance just because it is the latest one.
- `as_of ≤ min(wanted, today)`.
- Every non-number state has `cents: null`, never 0. A real `$0.00` renders.

## Contracts

**`bob bank report` JSON** is additive. `schema_version` stays 1, with no `!` break.
Every period selector (`-D/-w/-m/-y/-p`, with or without `-s`/`-c`) gains a top-level
array. It has one entry per importable account in config order. Cash accounts are
omitted. When config cannot load, the array is empty.

```json
"balances": [{
  "alias": "checking", "label": "Checking", "kind": "checking", "hero": true,
  "state": "as_of", "reason": null,
  "cents": 123456, "as_of": "2026-11-13", "wanted": "2026-11-15", "stale": false,
  "open": {"date": "2026-11-08", "cents": 130000}, "change_cents": -6544,
  "coverage": {"since": "2026-05-01", "through": "2026-11-13"},
  "anchors": {"count": 5, "agree": true},
  "target": {"path": "bank/2026-11.md", "line": 26, "date": "2026-11-13"}
}]
```

- `reason` is one of `future`, `pre_coverage`, `journal_unreadable`,
  `journal_contradiction`, `csv_overlap`, `no_anchor`, `anchor_mismatch`, or `null`.
- `coverage` is the span that holds `as_of`, or null.
- `anchors.count` counts that span's bank-reported balance observations. `anchors.agree`
  is false when any cross-check in the span failed.
- `target` is null when no month note exists.

**Human output.** One line per entry whose state is not `none`, placed after the
`Out · In · Net` line. The amounts are synthetic:

```text
Balance · Checking $1,234.56 as of Fri Nov 13 · −$65.44 since Sun Nov 8
Balance · Checking $1,234.56 at close Sun Nov 15 · −$65.44 since Sun Nov 8
Balance · Checking $1,234.56 so far, as of Fri Nov 13 · stale
Balance · Checking not imported yet
Balance · Checking unverified (bank balances disagree — see bob bank check)
```

**`bob bank check` findings.** These flow through the existing balance-findings path in
`check.rs`:

- `bank_balance_anchor_mismatch` is a warning. It names the account, the anchor date,
  the bank's amount, the reconstructed amount, and the hidden date range. Hint: usually
  a bank correction or a re-keyed transaction.
- `bank_balance_csv_overlap` is info. It names the batch date and the hidden span. Hint:
  import CSV and QFX files in separate runs.

No new subcommand, flag, or config key. `stats.bank_period` stays the one plugin gate.

## Phase: resolver — Historical balance resolver and check findings

- Add `src/native/bank/balance.rs` and register it in `mod.rs`. Put tests in
  `balance_tests.rs` if they are large, mirroring `charts.rs`/`charts_tests.rs`. Keep it
  pure, over config, journal, `today`, and timezone. Expose:
  - an account-history builder (spans, final days, effective anchors, unverified ranges,
    findings);
  - `resolve(&history, &Period, today) -> PeriodBalance`;
  - a `B(d)` accessor.

  Add per-date prefix sums only if profiling shows a need. About 850 transactions makes
  a linear scan fine.

- Share the import-local-date helper with `status.rs`. Do not change any `status.rs`
  status, delta, or gap semantics.
- Append the resolver's findings in `check.rs` beside the proof findings, and add both
  codes to the "Balance findings" table in `docs/bank.md`.
- Tests: the invariant suite, plus each of these cases:
  - an anchor before, inside, and after the period;
  - gap isolation, plus `awaiting_import` vs `gap`;
  - `before_start` reconstruction;
  - a CSV-only batch excluded exactly, and a mixed batch giving `csv_overlap`;
  - a mismatch hiding only its segment, and a dispute;
  - a provisional anchor ignored once superseded;
  - a provisional window day never settling a closed period;
  - open, all-time, and future periods;
  - the W53 boundary (Q4 2026 ends 2027-01-03);
  - zero and negative balances;
  - stale only on `so_far`;
  - journal malformed or contradiction giving `unverified`.
- Gates: `just fmt`, `just lint`, `just test`, and `just docs-check`.

## Phase: report — balances[] in bob bank report, human line, and docs

- `report.rs`: `BankReport` gains `balances`. `build()` resolves them for the period
  using the journal, config, `today`, and `heading_lines` for targets, and `to_json`
  emits `balances[]`. All-time periods resolve after `all_time_period` finalizes.
- `output.rs`: add the Balance line(s). Reuse the existing day-label helpers.
- Docs:
  - In `docs/bank.md`, add a `## Closing balances` section after "Account status". It
    covers the definition, what a posted balance is versus an available one, the
    resolution and state table, the as-of honesty rule, CSV and anchor safeguards, the
    read-time, never-written rule, and the CLI day lookup.
  - Add a `report` bullet for `balances[]` in that page's JSON section.
  - Extend the Bank reports line in `docs/reference/json-outputs.md`.
  - Use synthetic amounts in every example.
- Tests:
  - `src/native/bank/tests.rs` report JSON shapes for week, quarter, day, and all-time.
  - A `tests/cli/bank.rs` end-to-end test with a synthetic WF-shaped journal: a Monday
    import ending Friday gives W-week `as_of`; adding a Tuesday import settles it; `-D`
    on the Sunday answers `as_of` Friday; a quarter settles; an old binary-free JSON
    stays `schema_version: 1`.
- Live check, read-only. Run `cargo run -q -- bank report -p <id> -s -c -f json` for
  2026-W43 through W46, Q2, Q3, and Q4 2026. Compare against the research report's
  "Verified ground truth" table:
  - W46 is `as_of` Fri Nov 13.
  - W45 and earlier weeks are `settled`.
  - Q4 is `so_far`.

  Note any discrepancy in the bead and do not paper over it.

## Phase: readout — Bank Charts header balance readout in bob-ledger-tools

Work in the `bob-plugins` linked repo, opened through `/sase_repo`, and read its
`AGENTS.md`. The code lives in `plugins/bob-ledger-tools`, a fragment build.

- **Decode and model.** Add a new fragment `src/241nca-plugin-bank-period-balance.js`,
  registered in `src/fragments.json` between `241nc` and `241nd`. It holds:
  - `bankNormalizeBalances(value)`, which uses the safe-cents path.
  - The pure `bankPeriodBalanceModel(report, ctx)`.
  - The DOM builder for the balance node.

  `bankDecodeReport` attaches `report.balances`. A malformed `balances` value becomes
  `null` with one `console.warn` per session. It never fails the report, badge, or
  charts.

- **Charts model.** `bankPeriodChartsModel` carries `balance`, which is null unless a
  hero entry has a non-`none` state, and includes it in the digest.
  - When the panel would otherwise be hidden as `empty` or `waiting-badge`, a renderable
    balance yields a new `balance` state: header only.
  - The `waiting` state (badge off) appends the balance to the right of the waiting
    line.
  - With no `balances[]` (older bob), no hero, or `none`, the model and digest are
    byte-identical to today's.
- **Paint.** Inside the header, after the readout:
  - `.bob-bank-period-balance` holds a label (`BALANCE`, faint small caps,
    letter-spacing matching the disclosure), an amount, and an as-of or words part.
  - The amount uses `--text-normal`, weight 600, and tabular numerals. It is
    `flex: none` and never truncates.
  - The as-of suffix (`· as of Fri Nov 13`, or `· as of today`) is faint and ellipsizes
    first.
  - The `balance` state uses the same `.bob-bank-period` root with a static,
    non-interactive `BANK` label: no chevron and no switch.
  - Collapsed panels keep the balance.
  - The CSS sibling rule gives the readout precedence. Follow the family's forced-colors
    and reduced-motion rules.

| Hero state                                    | Readout                                                                    |
| --------------------------------------------- | -------------------------------------------------------------------------- |
| `settled`                                     | `BALANCE $1,234.56`                                                        |
| `as_of`                                       | `BALANCE $1,234.56 · as of Fri Nov 13`                                     |
| `so_far`                                      | `BALANCE $1,234.56 · as of Fri Nov 13` (as-of in warning ink when `stale`) |
| `awaiting_import` / `gap`                     | `BALANCE not imported yet` / `BALANCE not imported` (muted)                |
| `unverified`                                  | `BALANCE unverified` (muted)                                               |
| `none`, no hero, no or malformed `balances[]` | nothing                                                                    |

Ink rules:

- A negative amount renders `−$45.10` in warning ink. It is the only colored state, and
  never red.
- `$0.00` renders normally.
- A falling balance is never colored.

- **Tooltip and aria.** The tooltip is a multi-line `title`. For `as_of`:

  ```text
  Checking · closing balance (posted)
  $1,234.56 at the end of Fri Nov 13
  Sat Nov 14 – Sun Nov 15 not imported yet
  −$65.44 since Sun Nov 8, transfers included
  ✓ Matches all 5 bank-reported balances
  Posted balance: your bank app's "available" balance also subtracts pending holds
  Click to open Fri Nov 13 in the bank month note
  ```

  Variants:
  - **`settled`** names the period end and drops the not-imported line.
  - **`so_far`** says "so far" and adds a stale line when stale: "Last bank balance is N
    days old — import a current statement".
  - **Account `delta` status** (read from `accounts[]` by alias) adds "Ledger rows
    differ from the bank by $X — see bob bank check".
  - **`anchors.count == 1`** reads "From 1 bank-reported balance".
  - **`agree: false`** adds a see-check line.
  - **Words states** map each `reason` to one sentence and a remedy.

  The `aria-label` is one sentence that spells out the account, state, amount, full
  as-of date, and activation.

- **Activation.** When `target` exists, the balance gets `role="link"`, `tabindex="0"`,
  and click, Enter, or Space through `bankBadgeNavigate(target, newLeaf)`. Ctrl/Cmd and
  middle click open a new leaf. Without a target it is inert, with no pointer. It is
  read-only and never creates a note.
- **Tests.**
  - Model goldens for every matrix row, negative and zero amounts, stale, collapsed, the
    header-only and waiting-line states, older-bob digest parity, and malformed decode.
  - Paint tests for DOM order and roles, inert without a target, and the header-only
    root.
  - A hosts assertion that day strips never mount the panel.
  - Add the new test files, if any, to `test:unit`.
  - Run `npm run build`, `npm test`, and `npm run docs:check`.
- **Look at it.** Extend `scripts/preview-bank-period.cjs` with balance fixtures (active
  week `as_of`, settled, quiet header-only, collapsed, negative, stale, unverified).
  Render them, then read the PNGs. Iterate until alignment, weight, and ink feel calm
  and of a piece with the panel.
- **Docs.**
  - In bob-plugins `docs/bob-ledger-tools.md`, update the "Bank Period Charts"
    paragraph.
  - In bob-cli `docs/bank.md`, update the "Bank period charts" section: the header
    balance and the quiet-period row of its state table. Today that row reads "No
    counted money and no waiting rows → Hidden".
  - Run `just docs-check`.
- **Deploy.** Make sure the installed `bob` emits `balances[]` (run
  `bob bank report -p 2026-W46 -s -c -f json`). If it does not, run `just install` from
  an up-to-date bob-cli master checkout. Then run `bob plugins sync`, following the
  repo's sync-refusal rules.

## Phase: vocabulary — Bank Closing vocabulary and decision record

Use `/sase_memory_write` and apply only the accepted memory decisions. Run
`sase memory init` afterwards.

> [!decision] memory_bank_closing Create the new `glossary:bank-closing` strand (keyword
> `Bank Closing`; aliases `closing balance`, `period close`). It is one paragraph:
>
> - the posted (ledger) balance of one account at the end of a period's last covered
>   day;
> - drawn by bob from the bank journal at read time and never written;
> - shown in the Bank Period Charts header on [[period-note]]s, never daily notes;
> - labeled `as of` until coverage reaches period end;
> - never the app's available balance, and never summed across accounts.

> [!decision] memory_stats_strip_link In `glossary/stats-strip.md`, extend "A
> [[bank-badge]] follows each Stats Strip for the same period." with one sentence: on
> periods, the Bank Period Charts header also shows the period's [[bank-closing]].

> [!decision] memory_close_decision Create the new
> `decisions:period-closes-are-drawn-posted-balances` record
> (`metadata.status: accepted`, `decided:` the landing date). Follow the existing record
> shape:
>
> - **Applies to:** bob-cli and bob-plugins.
> - **Claim:** period closes are posted balances, drawn at read time, labeled as-of
>   until proven, never written, and never summed.
> - **Why:** cite the research artifact ref and this epic's commits. Name the rejected
>   alternatives: writing bytes into notes, the available balance, the weekend
>   heuristic, the badge as home, and whole-dollar rounding.
> - **Cost:** absent on mobile and in plain Markdown; an as-of suffix for a few days
>   each week; differs from the app's headline number.
> - **Reopens when:** a bank feed with intraday or available balances, plugin spawning
>   on mobile, a second importable account, or the suffix proving annoying.

## Privacy

bob-cli docs publish to a public site, and the repos are shared. Real vault balances
must never appear in tests, fixtures, docs, screenshots that get committed, commit
messages, or bead notes. Use synthetic amounts everywhere. Live-vault checks are
read-only, and their numbers stay in the terminal.

## Out of scope

- Daily notes and `bob-day-stats`.
- Writing cents anywhere.
- Available balance and pending transactions.
- Net worth or account sums.
- The balance lane or sparkline and a per-bucket `charts.balances[]` series (v2).
- A masked screen-share mode.
- The weekend settling rule.
- New config keys and bank feeds.
- Retiring the `Update [[cash#^balance]]` 30-day chore in `recur.md`. That stays Bryan's
  own gesture once he has seen the closes render. It next comes due 2026-12-01.
