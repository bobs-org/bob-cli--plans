---
tier: epic
title: Weekly bank import with deterministic matching and one guarded commit
goal: 'Complete bob-cli-kd.4: import posted Wells Fargo QFX/OFX and CSV transactions
  once, preserve typed rows and edits, journal private identities, and install the
  Monday PRE chore after the command is verified.'
phases:
- id: sources-journal
  title: Parse bank exports and establish the shared journal
  depends_on: []
  size: medium
  description: 'sources-journal: parse both OFX dialects and the Wells Fargo CSV profile,
    resolve account aliases, hash identities, and expose a validated per-host bank
    journal with bounded diagnostics and balance-phase input types.'
- id: match-name-plan
  title: Plan only missing rows and preserve existing writing
  depends_on:
  - sources-journal
  size: medium
  description: 'match-name-plan: build a pure whole-batch planner for seen identities,
    typed-row matching, learned and configured naming, restaurant links, month edits,
    and journal events, with exact deterministic previews.'
- id: import-command
  title: Ship the guarded import command and recovery tests
  depends_on:
  - match-name-plan
  size: medium
  description: 'import-command: integrate bob bank import, ordered guarded journal
    appends inside the shared transaction, scoped commit and sync, human and JSON
    output, and end-to-end refusal, replay, and recovery tests.'
- id: docs-ritual
  title: Document the import and install the Monday ritual
  depends_on:
  - import-command
  size: small
  description: 'docs-ritual: finish the import and sync documentation, regenerate
    CLI references, verify the installed command, and idempotently capture the weekly
    Bank import PRE chore; leave closure evidence for the land agent.'
proposed_by: bbugyi200.athena.bob-cli-kd.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kd.4
create_time: 2026-11-11 14:24:03
status: done
bead_id: bob-cli-kd.4.1
---

- **PROMPT:** [prompts/202611/bank_import.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_import.md)
- **PARENT:** [202611/bank_daily_acknowledgment.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_daily_acknowledgment.md)
- **BEAD:** [bob-cli-kd.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kd/bob-cli-kd.4.1.md)

# Plan: Weekly bank import

This child epic implements the complete `bank-import` phase, `bob-cli-kd.4`, of
`plan:202611/bank_daily_acknowledgment.md`. Its land agent owns completion of that
phase. Child phase workers close only their own beads; nobody in this child epic closes
`bob-cli-kd` or another ancestor plan.

## Evidence and scope

The assigned phase is in progress and has no earlier-work notes. The checkout already
contains the grammar and CLI foundations:

- `07f1c744`: bank grammar, month notes, and read-only check.
- `49218b38`: list, report, ensure, and ack.
- `757e36bb`: date-heading, BOM, and planner-anchor fixes.

There is no import command, source parser, or bank journal module. There are no
`--epic-symbol` entries for `bob-cli-kd.4` at planning time. Recheck the tree and phase
notes before implementation, preserving work that lands later.

The audited design and research reads are `plan:202611/bank_daily_acknowledgment.md` and
`research:202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md`, notably the
Import contract, bank-import/bank-balance boundaries, and research sections "Getting
transactions in" and "Payees, categories, and links". The research read initially
failed; opening `bob-cli--research` through `/sase_repo` refreshed its checkout and the
same audited reference then succeeded.

The parent decisions remain final: initial live setup is checking only; subscriptions
are discretionary and outside this phase. Only the outer wrap-up owns the authorized
glossary changes. This child epic changes no SASE memory. Read CLI/docs memory before
modifying those surfaces, and the theme-journal and gcal write-once decisions before
working on journal or ownership behavior.

Use an epic because parsing/journal validation, matching/naming, and the transaction/CLI
each have independent acceptance tests and together exceed a single bounded coding task.
Every phase above is direct implementation work; the final small phase owns
documentation and the live chore. Dependencies are serial to keep shared bank APIs and
transaction changes coherent.

Excluded: balance anchors and verdicts, coverage/presence findings, twins, `import -F`,
plugin UI, subscriptions, historical `@BANK` migration, feeds, budgets, pending
transactions, and multi-currency accounting. `bob-cli-kd.5` owns the balance work and
force restoration. Supply its inputs and extension points; do not advertise `-F` until
it works. Print the bank's ledger balance and date without a fabricated balanced
verdict.

## Existing implementation seams

- `src/native/bank/{config,grammar,scan,model,month}.rs`: account/category/rule models,
  integer cents, bounded bank scan, row collection, and byte edits. Reuse
  `canonical_template`, `plan_insert_day`, `plan_append_row`, and `plan_append_id`,
  preserving BOM, newline style, children, and free prose.
- `bank::{cli,ack,ensure,output,check,status}`: established CLI and JSON conventions.
  `status::accounts` intentionally remains unverified until the next outer phase.
  `model::collect_rows` currently calls every ID-bearing row imported; the journal
  permits distinguishing matched origins.
- `src/native/pref_journal/mod.rs`: namespace, host label, timestamp, raw event reader,
  ordered fold, bounded diagnostics, and append mechanics. `attachments/journal.rs`
  demonstrates typed event validation.
- `src/native/host_catalog/txn.rs`: `PlannedTxn`, `ReadDependencies`,
  `apply_and_commit`, and `run_locked`; `notebooks/create_run.rs` demonstrates pure
  preview and guarded commit. The current transaction accepts staged text rewrites, but
  no ordered journal append effect. Address this explicitly.
- `records::context::RecordContext` and `records::catalog::RecordCatalog` are the same
  registry/catalog used by `bob records list`. Resolve restaurant records there, rather
  than reading a hard-coded legacy schema.
- Command registration touches `src/native.rs`, `src/runner.rs`,
  `src/native/catalog/table.rs`, and `src/native/completion/tree.rs` as well as the bank
  module. Catalog/help tests and generated references must agree.

## Shared contracts

Use small bank-local modules such as `source`, `identity`, `journal`, `naming`,
`import_plan`, and `import_run`; names may follow nearby conventions. Agree on these
typed boundaries in the first phase, then consume them rather than repeatedly parsing
JSON in later phases:

- A parsed statement carries source kind, file digest, source order, inferred account
  kind, inclusive date coverage, optional ledger balance/as-of, and ordered posted
  transactions. Transaction money is checked signed `i64` cents; dates are validated
  `NaiveDate`s. Raw FITID lives only in transient parse/identity state and never in
  output, errors, or journal payloads.
- An account-resolved statement adds the configured alias and start date.
- A journal snapshot exposes imports, transactions keyed by full digest, row-ID
  ownership, import provenance/order, malformed findings, and exact read preimages. Keep
  enough original event metadata for the balance phase to select anchors, union windows,
  count each transaction once, and prove presence. Repeated observations never multiply
  transaction amounts.
- A batch plan owns per-file summaries, affected row results, month before/ after
  images, ordered serialized journal events, guarded read dependencies, and findings.
  The same planner feeds preview and apply. Capture one clock instant per plan; freeze
  `BOB_NOW` in byte-comparison tests.

### sources-journal

Implement the source and journal foundation with synthetic fixtures only.

**Parsing.** Accept OFX 1.x SGML leaf tags without closing tags and OFX 2.x XML,
including preambles, CRLF/BOM, ordinary entities, and Wells Fargo header/ `CHARSET`
noise. Reuse the existing `encoding_rs`, hashing, date, and regex dependencies where
appropriate. Parse transaction fields `TRNTYPE`, `DTPOSTED`, `TRNAMT`, `FITID`, `NAME`,
and `MEMO`; coverage from `BANKTRANLIST/DTSTART` and `DTEND`; ledger balance from
`LEDGERBAL/BALAMT` and `DTASOF`; type from `BANKACCTFROM/ACCTTYPE` or `CCACCTFROM`.
Never use available balance. Preserve the statement's posted calendar day instead of
moving it across days by converting its timestamp to UTC.

Fail the entire batch on malformed required fields, invalid dates, fractional cents,
overflow, unsupported currency, conflicting repeats of a bank ID, or an account
structure that cannot be resolved safely. Do not silently skip a malformed `STMTTRN`. A
valid empty statement still carries coverage/balance evidence. If multiple statement
blocks cannot be resolved separately without conflating accounts, refuse and teach
importing separate account exports.

CSV is the single headerless five-column Wells Fargo profile:
`Date, Amount, *, (blank), Description`. Parse quoted commas, escaped quotes, CRLF, and
signed decimal money correctly; validate the profile instead of guessing column
mappings. Use `csv:<date>:<cents>:<occurrence>` as the bank ID, with occurrence counted
per date/amount in file order, independently for each file. Identical purchases remain
distinct, and whole-date overlapping exports reuse the same occurrence identities. CSV
has no balance or account kind; require `-a` when type-based resolution cannot decide.
Document that this is a fallback profile with no cross-format fuzzy identity inference.

`-a ALIAS` selects a configured importable account. Otherwise choose the unique account
with the file's kind; refuse ambiguity with aliases and a `-a` hint. Reject unknown/cash
aliases. Ignore account/routing numbers for selection and discard them; never store or
echo them. The final checking-only decision governs live configuration, while synthetic
fixtures still cover savings, credit, ambiguity, and explicit selection.

**Identity and journal.** Add the `.bob/bank` namespace, noun `bank`, to `pref_journal`.
Keys are exactly SHA-256 of `alias + "\n" + bank_id`. Reserve a row ID of the first
eight hexadecimal characters; extend to twelve when a different known key occupies the
eight-character ID. Reuse existing assignments from the folded journal, reserve IDs
within the batch, and refuse an unresolved twelve-character collision rather than
conflate rows.

Use the existing `v: 1`, timestamp, host, and kind event envelope:

- `bank.import`:
  `{account, source, file_sha256, start, end, balance_cents?, balance_asof?, counts}`.
  `source` identifies the export format; raw paths/files and bank identifiers are not
  journal payloads.
- `bank.txn`: `{account, key, id, posted, cents, desc, merchant, outcome, note}`.
  Outcomes are `written`, `matched`, or `before_start`; `note` is the vault-relative
  target note, null for before-start transactions.

Mask every run of at least four digits in `desc` as `••••`; derive the merchant key from
cleaned, masked text so identifiers cannot leak through that second field. Do not
persist raw FITID. Validate dates, digest lengths, aliases, integer cents, outcomes, and
relative note paths on journal reads. Read every host in deterministic writer order,
union identities and imports, and expose contradictions rather than silently overwrite
immutable money. Malformed JSON/envelopes/payloads produce bounded
`bank_journal_malformed` warnings without printing the raw line. Integrate that finding
into `check`; subsequent commands can reuse it.

Tests cover equivalent SGML/XML statements, header noise/encoding, CSV quoting and
repeated occurrences, invalid money/date/truncated input, empty statements,
inferred/explicit/ambiguous accounts, exact hashing, forced prefix collisions, no raw
identifiers in serialized data, multi-host replay, and malformed-line bounds. Publish
stable journal APIs for `bob-cli-kd.5` in the phase completion note. Keep unwired parser
APIs test-scoped until their consumer lands rather than adding broad unused-code
suppressions.

### match-name-plan

Implement the entire batch as an in-memory plan, with no lock, Git, sync, journal
append, or filesystem creation in the planner.

1. Load a complete bank snapshot and all journal hosts. Refuse incomplete reads and
   errors that make a target write unsafe. Account configuration, every month/journal
   used for dedupe or learning, and restaurant inputs are read dependencies. Preserve
   duplicate-ID diagnostics rather than choosing an arbitrary row.
2. Resolve all input files and validate every transaction before producing mutations.
   Process transactions by `(posted, input-file order, record order)`, maintaining batch
   reservations for IDs and typed-row candidates.
3. A key journaled by any host, or its row ID present in the bank notes, means seen. Do
   not rewrite, tick, or resurrect it. A row installed before a crash but not yet
   journaled must be recognized on retry; fill the missing journal evidence without
   another row. Record before-start transactions without creating or matching a row,
   preserving their posted money for audit while the balance phase applies its start
   filter.
4. Match only rows without a bank ID, on the same non-cash account, with exactly equal
   signed cents, and date in `[posted - 1 day, posted + 3 days]`. Use the uniquely
   nearest date and consume each candidate at most once. An equal-distance tie writes a
   new waiting row and increments ambiguous; never choose by payee or file ordering to
   break that tie. A match changes only the trailing bank ID: checkbox, wording, fields,
   links, notes, existing non-bank IDs, and child bullets remain byte-identical.
5. Otherwise render an unchecked canonical row under its posted day, using the shared
   month/day helpers. Dates can cross months; append after the day's existing row blocks
   without separating a row from its children. Recompute locations or apply descending
   original-coordinate edits so earlier inserts cannot shift later typed-row matches.

Naming uses the first matching case-insensitive substring rule, then the most recent
acknowledged ID-bearing row with the same journal merchant, then Wells Fargo built-ins,
then cleaned title-cased description. Resolve payee/category/links by this precedence,
filling unspecified rule fields from lower tiers. Preserve an explicitly selected
higher-priority link. Learn from current row text, not a stale journal copy; waiting
rows never teach a name. Break equal-date learning ties deterministically by note and
line. Existing written/matched rows are never renamed or recategorized.

Clean authorization/date prefixes, `SQ *`, `TST*`, `PAYPAL *`, `#1234` store numbers,
trailing `S<digits> CARD <dddd>`, and recognizable trailing city/ state spans. Make
cleaning conservative and fixture-driven so it does not erase an ordinary multiword
merchant. Built-ins cover ATM withdrawal (`ATM withdrawal`, `cash`), ATM/overdraft fees
(`fees` when configured), and card-payment/online-transfer descriptions (`transfer`).
Preserve the transaction type/raw transient description for classification before
cleaning can remove a useful discriminator. A positive amount alone never implies income
or refund. Never emit an undeclared optional category.

With no selected link, an exact case-insensitive restaurant title match through the
records registry supplies its note link and kind emoji (normally
`[[eat/<stem>|<emoji>]]`). No fuzzy link guessing, record mutation, or full vault task
scan. If restaurant titles are ambiguous, omit the link. Ensure untrusted source
descriptions remain one grammar-safe payee line and cannot inject headings, block IDs,
fields, or links into canonical output.

Create an import event for new evidence (a new file digest/window/balance), even if all
its transactions were seen. A fully recorded identical replay has no new events and no
commit. A new empty statement may change only the journal. Counts distinguish new,
matched, seen, ambiguous, and before-start; ambiguous is a subset of new. Journal txn
events are emitted once per new identity, including recoveries, not once per overlapping
file. Update row origin reporting to use matched journal outcomes with the existing
imported fallback for an ID without journal evidence.

Tests cover repeated/overlapping files, same-day identical purchases with different
FITIDs, one-use matching across files, both window edges, equidistant ambiguity, $20
versus $24 staying separate, before-start rows, deletion remaining deleted, collision
extension, row-without-journal crash recovery, late months, BOM/CRLF/children/prose
preservation, every naming tier, current edited learning, restaurant registry cutover,
and deterministic after-images under fixed `BOB_NOW`.

### import-command

Add `bob bank import FILE... [-a ACCOUNT] [-b DIR] [-d] [-f human|json]`. Long options
are `--account`, `--bob-dir`, `--dry-run`, `--format`, and `--help`, each with the
indicated short alias (help `-h`) and alphabetical help order. Preserve bare `bob bank`
as list. Wire command dispatch, completion, catalog membership, and help/catalog tests.
Refused input or an unsafe/ambiguous batch exits 2; runtime failures exit 1. Preserve
established bank teaching-empty behavior and `BOB_NOW` handling.

**Transaction.** Pre-parse every source before taking the maintenance path, so a
malformed input causes no vault, journal, Git, or sync change. Keep the parsed
bytes/digest fixed for this invocation. Real apply uses `host_catalog::txn::run_locked`
for identity, bounded lock, worktree check, pre-sync, fresh vault plan, guarded install,
one scoped commit, and post-sync. Never plan from a pre-sync vault snapshot and later
apply it unchecked.

Extend the shared `PlannedTxn` with a narrow ordered append effect (for example
`appends: Vec<PlannedAppend>` holding relative path, exact optional byte preimage, and
serialized suffix). Existing notebook callers pass no appends and retain their behavior.
Keep journal bytes separate from text rewrites so malformed historical bytes remain
intact. The lifecycle must:

1. Validate row/read/journal preimages and the snapshot membership needed for dedupe;
   detect newly appeared month/host journal files that invalidate it.
2. Install guarded month files first.
3. Revalidate append preimages and append prepared events to this host's journal using
   shared append mechanics, with bounded failure handling.
4. Verify resulting rows and journal, then include both paths in one scoped commit:
   `bank: import N rows (<alias>)`. For multi-account batches, use a stable
   comma-separated alias list inside the same parentheses. `N` counts new waiting rows,
   not matches or journal events.
5. Roll back owned row writes and append effects on pre-commit failures only when the
   current bytes still match the bytes this transaction installed; preserve any later
   external edit and report incomplete rollback. Handle partial append failures
   explicitly. A commit that succeeded despite an error stays committed. Post-sync
   failure reports the existing local commit and recovery hint rather than rolling it
   back.

Do not append inside an untracked verification callback: that would leave journal paths
out of scoped commits and rollback. Do not fork the maintenance lock/sync
implementation. Test the added append effect directly with the existing transaction
tests, then through bank import. The journal is append-only on success; rollback is a
guarded failed-transaction recovery.

Dry run invokes only the shared pure planner and shows exact month and journal
after-images/diffs. It creates no directories, appends nothing, runs no Git or sync, and
acquires no write lock. On the same snapshot and clock, its bytes equal a real run.
Identical replay produces no scoped import commit and no new journal events; shared
pre-sync behavior remains unchanged.

Human output follows the parent sample: file/account/date range; new, matched, seen,
ambiguous, and before-start counts; in/out cents; optional ledger balance and date;
affected paths; commit/sync result; next action. Respect `NO_COLOR` and pipes. Do not
claim balanced or all caught up.

JSON is additive `schema_version: 1`, with existing common `ok`, `today`, `categories`,
`accounts`, and `issues`, plus the notebook-style `dry_run`, `changed`,
`changes[] {path, diff?}`, `findings`, `commit`, `sync`, and `warnings`. Add
parent-contract
`files[] {file, account, source, start, end, new, matched, seen, ambiguous, before_start, out_cents, in_cents, balance_cents, balance_asof, status}`
and `rows[]` using existing row JSON. Amounts are signed integer cents. Balance status
remains unverified here; keep the verdict seam usable by `bob-cli-kd.5`. Diagnostics
identify a file and field/record position without echoing private identifiers or raw
data.

End-to-end tests use temporary Git vaults and stubbed sync, never the live vault. Prove
one scoped commit contains both rows and journal, exact dry-run parity, refusal leaves
the entire batch unchanged, replay is a no-op, overlapping files and multiple accounts
dedupe, malformed journal warnings surface, stale row/journal/registry reads refuse, a
lock conflict refuses, append/verify/commit failures recover correctly, and a post-sync
failure retains the committed result. Include interrupted state with a row but no txn
event. Run notebook/host-catalog regression tests for the shared change.

### docs-ritual

Update `docs/bank.md` with source formats, account resolution, matching, write-once
ownership, naming, hashed IDs, journal schema, preview, replay, refusal behavior, and
current limits. Update `docs/vault-git-sync.md` with the `bank: import` subject and
`.bob/bank` paths. Update `docs/reference/json-outputs.md`; run `just docs-gen` and
`just docs-check`. Do not edit the release version or CHANGELOG. Documentation must
accurately leave force restoration and balanced/gap/stale verdicts to the next phase.

After tests pass, follow the repo's supported local installation path and verify the
command that the weekly chore will call. Before vault work, read `obsidian.md` through
`/sase_memory_read`, and use `/sase_repo` if accessing another repository. Recheck the
narrow `gtd_daily` target for an existing equivalent open chore. Preserve it if present;
otherwise use `bob capture` and its preview/managed-write path to add exactly one PRE
task with `[repeat:: every week on Monday when done]`:

```text
#gtd #pre **Bank import**: download the last 14 days as Quicken (QFX) from Wells Fargo · run `bob bank import ~/Downloads/*.qfx` · tick what it wrote (bob bank) · write one line in last week's note: what surprised me?
```

Inspect current capture help for the exact route/recurrence syntax. Verify the resulting
target, repeat field, and `#gtd #pre` tags. Do not recreate the daily bank chore, change
checking configuration, or touch unrelated tasks. This live chore is expressly in the
approved parent phase scope.

A real-file dry run is optional, only if a local export is readily available. Use a
temporary vault copy, keep previews out of agent-visible output, and report counts only.
Do not import real transactions, copy raw exports into the repo/vault, or treat absence
of an export as a blocker.

## Verification and child land duties

Each coding phase runs its focused tests and formatting/lint checks; the command and
final verification cover `cargo test --lib bank`, relevant host-catalog/notebook and CLI
integration tests, and `just check`. Generate docs before `just docs-check`. Use
`/sase_monitor` for lengthy required checks, waiting for the staging command to exit and
finalizing correctly.

The land agent rechecks the full import acceptance table against the final tree,
verifies the chore evidence, and records the public parser/journal/ planner interfaces
that the outer balance phase consumes. A failing check that reproduces identically on a
clean base is recorded with exact evidence as `PROPOSED FOLLOW-UP:` on `bob-cli-kd.4`,
citing an existing tracking bead when known; it does not leave a completed phase open.
No phase worker creates new task beads. Use such notes for any genuinely out-of-scope
discovery.

Before any phase closes, run `sase bead epic-symbols <its-id>` and resolve or re-key
remaining entries to a still-open owner. The child land agent also runs
`sase bead epic-symbols bob-cli-kd.4`, verifies all child work has settled through the
normal child-epic lifecycle, and closes `bob-cli-kd.4` with a note describing
implementation and verification. This is the delegated closure requested by the phase
prompt. Never close `bob-cli-kd` or any other ancestor.
