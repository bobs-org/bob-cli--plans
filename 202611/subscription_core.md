---
tier: epic
title: Subscription registry, attribution, and ledger core
goal: Complete bob-cli-ls.2 with one reusable subscription resolver and ledger, guarded
  registry inputs, and additive bank row JSON.
phases:
- id: registry_match
  title: Typed subscription registry and deterministic matching
  size: medium
  depends_on: []
  description: 'registry_match: load the subscription album by noun, validate typed
    records and match needles, expose snapshot-based parsing and deterministic match
    provenance, and cover failures with synthetic tests.'
- id: attribution_ledger
  title: Shared charge attribution and subscription ledger
  size: medium
  depends_on:
  - registry_match
  description: 'attribution_ledger: build the deduplicated journal-and-row universe,
    resolve explicit links and eligible pattern matches, derive coverage-aware subscription
    ledger facts, and test integer money and calendar boundaries.'
- id: integrate
  title: Guarded loaders, row JSON, suggestions, and documentation
  size: medium
  depends_on:
  - registry_match
  - attribution_ledger
  description: 'integrate: generalize the guarded restaurant loader, integrate the
    shared subscription snapshot into list/report/import row JSON and category suggestions,
    surface bounded availability warnings, document the contract, and verify the full
    core scope.'
proposed_by: bbugyi200.athena.bob-cli-ls.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ls.2
create_time: 2026-11-15 13:23:43
status: done
bead_id: bob-cli-ls.2.1
---

- **PROMPT:** [prompts/202611/subscription_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/subscription_core.md)
- **PARENT:** [202611/bank_subscriptions_album.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_subscriptions_album.md)
- **BEAD:** [bob-cli-ls.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ls/bob-cli-ls.2.1.md)

# Subscription core implementation

- Assigned phase: `bob-cli-ls.2` (`core`).
- Approved outer plan: `plan:202611/bank_subscriptions_album.md`.
- Design basis:
  `research:202611/bank_subscriptions_feature_design/bank_subscriptions_feature_design.md`.

## Scope and evidence

This child epic completes the core phase of the approved subscriptions album epic. The
assigned bead was in progress with no earlier-work notes when read. The clean checkout
at `9c2c2404` contained no subscription implementation or tests, and
`sase bead epic-symbols bob-cli-ls.2` reported no entries. Recheck the current tree
before implementing; sibling phases may have landed since this plan was written.

The work separates into three serial coding units because registry validation,
transaction identity and calendar calculations, and guarded command integration each
need substantial independent tests. Each unit is bounded medium work. The child epic's
land agent verifies the assembled result and closes the original assigned phase; child
phase workers close only their own beads. Never close `bob-cli-ls`, `bob-cli-kt`, or any
other ancestor or unrelated bead.

The following existing code supplies the integration points:

- `src/native/records/context.rs`: `RecordContext::lookup_noun`, with valid, invalid,
  unclaimed, and incomplete-discovery outcomes.
- `src/native/records/catalog.rs`: invocation-local `RecordCatalog`, exact frontmatter,
  raw field values, folder membership, and per-record diagnostics.
- `src/native/bank/import_run.rs::load_restaurants`: guarded album declarations, root
  Markdown membership, record bytes, and restaurant folder membership. Its current
  record consistency check compares titles only; subscription parsing must bind every
  semantic input to the exact guarded bytes.
- `bank/model.rs`: `collect_rows`, `BankRow`, and the shared `row_json` serializer.
  `bank/journal.rs`: folded unique transactions, IDs, import coverage, and preimages.
- `bank/suggest.rs`: shared category suggestions and read-time digit masking rules;
  `bank/naming.rs`: substring matching and Wells Fargo built-in classification.
- `bank/cli.rs`: list/report construct all rows before filtering. Import's pure
  `plan_batch` returns new/matched/restored rows, journal events, and after-images.
- `tests/cli/bank.rs`: isolated vault fixtures, frozen `BOB_NOW`, import replay,
  restaurant behavior, JSON, and file ownership checks.

Read the parent plan and the research's match, import-loader, and attribution sections
through `sase artifact read`. The research sidecar may need preparing with
`sase repo open bob-cli--research` before the audited artifact read succeeds. Read
`docs/bank.md` and `docs/records.md`; follow the `docs.md` and `cli_rules.md` reference
memories when touching docs or CLI contracts.

## Fixed decisions and boundaries

The parent decisions are final: `payee_source = record_title`,
`memory_decision_record = no`, and `memory_glossary_term = no`. Edit no memory files or
generated AGENTS instructions. Do not edit Cargo versions or `CHANGELOG.md`.

This child epic delivers reusable facts, guarded loading, row JSON, and read-time
subscription suggestions. The outer epic's import phase owns writing subscription
links/title/category into newly imported Markdown, skipping learned subscription links,
and import summary counts. The outer check phase owns business findings, candidate
detection beyond the recurring marker, and proposals. The outer report phase owns the
new `bob bank subscriptions` command. Vault seeding, migration, plugin work, and
release/install work are outside this child epic.

The shared core must expose enough typed facts for those consumers without embedding
their output rendering. Core availability warnings and structured record/match
diagnostics are included here. Do not add another matcher in a later consumer.

Use fictional services, prices, descriptors, accounts, and transaction dates in every
fixture, plan, doc, commit, and bead note. Do not read the real bank vault for this
work. Never serialize raw journal descriptions, private identifiers, or record bodies as
new public provenance or diagnostics. Preserve existing bank totals and write-once
ownership, and never create records or backfill historical rows.

## Shared internal design

Add `src/native/bank/subscriptions/` with `mod.rs`, `registry.rs`, `matching.rs`,
`attribution.rs`, and `ledger.rs`; use separate coverage/date helpers and test files
when needed for clarity. A command owns one loaded registry and one computed ledger
snapshot, reused by its consumers. Avoid global state and per-row filesystem scans.

Keep loading separate from pure calculations. Establish these capabilities, with names
adapted to local conventions:

- A registry snapshot containing album identity/emoji, typed records indexed by
  canonical path, availability (`missing`, `available`, `invalid`), and diagnostics.
  Parse from supplied record bytes as well as through a read-only vault loader.
- A pure match result with an optional record, source/needle provenance, and sorted tied
  record identities. Explicit-link resolution shares canonical identities.
- An attributed transaction collection and merged coverage windows by account.
- A ledger snapshot keyed by record identity, plus projections indexed by row
  location/identity. Business facts retain charge references so later findings need not
  recompute which charge triggered them.
- A guarded record-link input bundle supplying restaurants, subscriptions, file
  preimages, root/folder membership guards, and bounded warning information to import.

Avoid changing every `collect_rows` caller merely to add a vault path. Keep raw row
collection usable by existing tests and let a shared enrichment entry point consume the
loaded registry, config, journal, all rows, and frozen today. Compute facts before
list/report filters and attach the same typed row projection to `BankRow` for
`row_json`. Import must enrich against its projected after-state using the same entry
point, including the batch's new journal evidence and before-start history.

## Phase registry_match

### Registry contract

Resolve the album with `RecordContext::lookup_noun("subscription")`, checking the
resolved kind's noun rather than accidentally accepting an unrelated stem named
`subscription`. Use its actual folder, declaration, status labels, and emoji; do not
hard-code `subscriptions/`. A renamed album with the same noun must work.

Load `SubscriptionRecord` fields: stem, vault-relative `.md` path, title, status, amount
cents, cadence, pricing, validated needles, invalid needles with reasons, category, url,
renews, and ended. Retain a link target without `.md` separately or derive it safely.
Status is `to_cancel`, `to_decide`, `active`, or `cancelled`; cadence is `monthly`,
`four_weekly`, `quarterly`, or `yearly`; pricing defaults to `fixed` only when absent
and otherwise must be `fixed` or `variable`.

Parse the authored decimal amount exactly into nonnegative integer cents, with checked
arithmetic. Never round a fractional cent or compare money as floats. Malformed,
negative, fractional-cent, and overflowing values are diagnostics. Respect the record
schema/default rules; an explicitly empty field is not absent. Keep malformed records
isolated rather than disabling valid neighbours. Missing or invalid required
identity/status/amount/cadence/pricing makes that record unusable for matching; invalid
optional dates become diagnosed absent values, and an unknown category cannot supply a
suggestion. Invalid needles are individually excluded while the rest of a valid record
stays usable. Matchless records remain explicitly linkable.

A genuinely missing album is a quiet empty snapshot. An invalid/ambiguous album or
unreadable subscription registry disables subscription facts for that read and yields
one bounded `bank_subscription_album_invalid` warning with a `bob albums check` hint.
Preserve diagnostic paths and reasons without dumping file contents.

### Match contract

Trim needles, validate, then deduplicate case-insensitively within each record. Invalid
means fewer than three characters, an ASCII run of four or more digits, or equality
after case folding with one of:

`PURCHASE`, `RECURRING`, `PAYMENT`, `RECURRING PAYMENT`, `AUTHORIZED`, `AUTHORIZED ON`,
`CARD`, `POS`, `ACH`, `DEBIT`, `GOOGLE`, `APPLE`, `PAYPAL`, `AMAZON`, `SQ`, `TST`.

Keep structured `bank_subscription_match_invalid` diagnostics for the later check
consumer. Valid needles use the existing lowercase substring semantics on raw
descriptions at import, masked descriptions at read time, and typed payees when there is
no journal evidence. Do not clean a descriptor before matching.

Collect matched records across all statuses. If any live record matches, discard
cancelled contenders; otherwise cancelled-only attribution is valid. Compare each
record's longest matched needle. One strictly longest record wins; equal maxima across
records mean no attribution with tied identities preserved. Never use amount, file
order, or history to break a tie. Multiple matching needles in one record are not
multiple contenders. Use deterministic ordering only to present provenance.

### Verification

Cover noun resolution in a differently named album, matchless records, missing versus
invalid album, isolated malformed records, exact cents, defaults, optional values, case
folding, trimmed/deduplicated needles, every rejection class, descriptor drift through
two needles, live-over-cancelled, cancelled-only, longest-needle precedence, and exact
ties independent of file order. Export the pure snapshot parser for the guarded loader
phase; do not duplicate record parsing there.

## Phase attribution_ledger

### Transaction universe and provenance

Use every folded journal transaction exactly once, including `before_start`. Join its ID
to a unique current bank row when present; the row supplies current authored fields and
link/category decisions while the journal supplies the masked descriptor and identity.
Otherwise retain it as journal-only without writing a row. Include ID-less typed rows
only outside the union of that account's inclusive import windows. Merge
overlapping/adjacent windows; do not infer coverage from min/max dates across gaps or
from another account. ID-bearing rows without journal evidence may still receive row
projections but must not be invented as extra journal-history ledger entries.

Duplicate row IDs and conflicting explicit subscription links must not pick an arbitrary
row or record. Preserve deterministic ambiguity diagnostics; count a journal transaction
only once and avoid false explicit-link attribution. Normalize wikilink aliases and
optional `.md` when resolving the actual album's record paths; unrelated links do not
participate. A unique explicit subscription link wins over every pattern and category
inference exclusion. A malformed/unresolved link must not produce a phantom record.

For pattern inference, reject zero amounts, inflows except an explicitly categorized
`refund` row, `transfer`/`income`/`cash` rows, and Wells Fargo built-in transfers, cash
withdrawals, or fees. Reuse the built-in classifier before description cleaning. Where
journal-only evidence needs classification, use existing rules and built-in category
semantics without assuming a positive amount is income or refund. A refund can attribute
and net against spending, but is never a charge for latest, intervals, price change, or
cancellation sentries. Keep explicit-link provenance separate from match provenance,
with `needle: null` for links and tied identities for ambiguity.

### Ledger facts

All calculations are pure functions of registry, attributed transactions, coverage, and
caller-supplied today (`BOB_NOW` through the existing command clock).

| Fact               | Required behavior                                                                                                                                                                                                                                                                                                                    |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Charges            | Attributed negative transactions oldest first, preserving signed cents, date, optional path/line/id, account, and `journal_only`; deterministic ordering for same-day transactions.                                                                                                                                                  |
| Refunds and totals | Retain attributed refunds separately; compute gross charge magnitude, refund credits, and net spending with checked integer arithmetic. Refunds never enter the charge list.                                                                                                                                                         |
| Latest             | Newest charge or null.                                                                                                                                                                                                                                                                                                               |
| Next expected      | `renews` override; else latest plus one cadence step; else null. Monthly/quarterly add 1/3 calendar months with month-end clamping; four-weekly adds 28 days; yearly adds one calendar year with leap-day clamping.                                                                                                                  |
| Monthly equivalent | Fixed: amount times 1, 13/12, 1/3, or 1/12 for monthly, four-weekly, quarterly, yearly; round half-up with integer math. Variable: trailing three-calendar-month charge average, falling back to amount when that window has no charges. Cancelled records contribute zero.                                                          |
| Price change       | Fixed monthly/four-weekly needs the last two charges equal and different from configured amount; quarterly/yearly needs the latest different. Return from/to cents and the supporting date. Variable never reports a price change. A one-off monthly proration stays quiet.                                                          |
| Still charging     | `to_cancel` and latest within 1.5 cadence steps of today. Derive the cadence duration using the same calendar-step helper, compare integer day durations, and test the inclusive threshold.                                                                                                                                          |
| After cancel       | Cancelled with charges strictly after `ended`; if ended is absent, every charge qualifies. Keep the newest offending charge for the later finding.                                                                                                                                                                                   |
| Timeline           | Up to 12 cadence buckets at the current observation date; expose bounds/due date and `charged`, `missed`, `uncovered`, or `upcoming`. A missing charge is missed only after its due interval has elapsed and merged import coverage spans that interval. A still-pending due interval is upcoming; incomplete coverage is uncovered. |

Make date-window boundaries and timeline anchoring explicit in helper documentation and
tests: use the next expected schedule (renews, otherwise latest) to align cadence
intervals, and retain the current interval even when its due date is later than today.
With no schedule or historical account evidence, do not manufacture missed charges. For
records seen on several accounts, require coverage on all observed relevant accounts
before declaring an empty interval missed. Historical bucketing must use consistent
original schedule anchors so month-end clamping does not drift over time. The trailing
average uses the rolling date window from today minus three calendar months through
today, sums charge magnitudes, divides by three with half-up rounding, and keeps
refund-net totals separate from this gross charge-rate fact.

### Verification

Cover explicit-link-over-match, renamed album links, ties/conflicting links, every
inference exclusion, refund netting, journal-only before-start history, journal/row
deduplication, missing rows, account-specific coverage and gaps, exclusion of covered
typed rows, and permitted uncovered typed rows. Test every cadence, end-of-month and
leap-year dates, renews precedence, half-cent rounding, variable fallback/average,
cancelled zero commitment, fixed two-charge confirmation/proration, still-charging
boundaries, ended strictness, all timeline states, and no evidence. Keep exposed facts
sufficient for the outer epic's check/report consumers without implementing those
commands.

## Phase integrate

### Guarded record-link loader

Generalize `load_restaurants` into one guarded registry input bundle, preserving the
existing `eat` exact-title behavior. Use one `RecordContext`; return both restaurant and
subscription records. Preserve root Markdown membership and every consulted album
declaration, parent/registry dependencies, record bytes, and both record folders'
membership guards. Guard missing folders too, so a newly appearing record invalidates
the plan. Parse subscription semantics from the exact captured bytes; guard the same
bytes at transaction apply, with no unguarded second semantic read.

Distinguish an invalid subscription declaration (bounded
`bank_import_subscriptions_unavailable`, imports proceed without subscription facts)
from unreadable or racing directory membership (refuse through existing registry/read
failure handling). Preserve existing restaurant safety behavior. Do not broaden a read
failure into permission to import from an incomplete registry. Share the loader and
parser with the read path without adding lock/write behavior to read-only commands.

Pass the snapshot into `PlanInput` or an equivalent explicit pure input. Ensure every
future import naming decision can consume its raw-description resolver and all guarded
dependencies. This phase wires the data and row projection, not the outer import phase's
new Markdown naming/link behavior.

### Row JSON and suggestions

Add these fields to every list/report/import row, including restored import rows:

```text
subscription: null | {
  stem, path, title, status, status_label, cadence, pricing, amount_cents,
  source, needle, drift_cents, price_change, state
}
bank_recurring: boolean
subscription_candidate: boolean
```

Use vault-relative `.md` paths consistently, matching other note navigation objects.
`source` is `link` or `match`; matched rows without an authored link remain `match`.
Fixed outflow drift is charge magnitude minus amount; variable rows and non-charge
refund rows have null drift. Price change comes from the shared ledger. Define state
precedence as after_cancel, to_cancel, usage, drift, ok: sentries apply only to their
qualifying charge rows; `to_cancel` applies only to the latest charge when the ledger
says still_charging. Drift alone must not be promoted into a confirmed price change.

Read `bank_recurring` from the uncleaned journal/raw description's case-insensitive
`RECURRING PAYMENT` marker. Candidate is true only for an eligible outflow with that
marker, no attribution, and no matching bank rule (including a rule with only payee or
link). Use the same read-time needle masking semantics for rule matching. An absent or
disabled subscription album forces the entire new projection to null/false, as the
parent contract requires.

Add `SuggestionSource::Subscription`. For an uncategorized attributed row, use rule
category first, subscription category second, then existing learned/bank tiers. Keep
already authored categories unchanged. Preserve the existing read-time rule that
hidden/unknown categories are not offered. Expose the declared-category lookup needed by
the outer import phase, where hidden declared keys are allowed for fill-down.

List and report compute their ledger from all rows/history before filtering. Import
computes one projected journal/row after-state, deduplicating existing and newly planned
transaction events, then uses the same resolver/ledger to enrich its output. The
raw-description matching seam is available for the outer import phase; because valid
needles cannot contain masked digit runs, import/read projections must agree. Preview
and apply expose the same row facts, and import replay remains byte-idempotent.

Wire one bounded availability warning into read output/JSON and `bank check` without
implementing the outer phase's price/candidate/cancelled-charge findings. Add an
additive warnings/findings carrier only where the current envelope lacks one; keep JSON
stdout a single object and keep warning-only checks successful. Expose structured record
and ambiguity diagnostics for the later checker. Do not emit one warning per row or
silently confuse an unreadable registry with an absent one.

### Documentation and integration verification

Add `## Subscriptions` to `docs/bank.md`: album schema summary, noun binding, optional
match list, validation/precedence/ties, journal and typed-row attribution, refunds,
coverage, ledger rules, and missing/invalid behavior. Extend row JSON and suggestion
sources there and in `docs/reference/json-outputs.md#bank-reports`. Describe only
implemented behavior; leave the outer import phase's Markdown naming and command
documentation for that phase. Use fictional Tunely/Cloudbox examples throughout.

Add integration tests for matching projection equality in list/report/import, latest and
price facts drawn from history outside a selected month, recurring marker and candidate
suppression by rules, category precedence, hidden versus unknown category handling,
no-album compatibility, invalid-album warning bounds, and read-only byte preservation.
Import fixtures must also prove guarded record/declaration/membership changes invalidate
apply, restaurant output stays unchanged, typed matching appends only its ID, preview
agrees with apply, and replay makes no write. Reuse focused existing tests rather than
copying broad fixtures unnecessarily.

## Gates and landing

Each coding phase runs its focused tests and the applicable repository checks. Use
`cargo test --lib native::bank` and `cargo test --test cli bank` as targeted starting
points; choose narrower valid filters as needed. The assembled child epic must pass
`just all` and `just docs-check`. Use the SASE monitor workflow for long commands and
wait for each handoff command itself to exit before ending a turn. Do not add tests that
only mirror struct definitions; assert observable money, attribution, warning, and
ownership behavior.

Intermediate unused core APIs may follow the existing bank module convention only while
downstream wiring is pending. By landing, every temporary epic-symbol entry must be
resolved or assigned to a still-open outer phase that actually consumes the API. Run
`sase bead epic-symbols` for each phase before closing it and for `bob-cli-ls.2` before
its final close. Never leave exemptions keyed to closed phases.

The child epic's land agent reviews phase evidence, confirms the parent contract is met
across the complete core, runs the gates, and records aggregate results and API entry
points on `bob-cli-ls.2`. Discovered out-of-scope work is recorded only as
`PROPOSED FOLLOW-UP:` notes for the outer epic's land agent. A check failure reproduced
identically on a clean base is recorded with evidence and does not prevent closing
completed scope. Then close only `bob-cli-ls.2` with a verification note; never close
its parent epic or the originating feature bead. Use `feat(bank)` Conventional Commit
messages and the required `/sase_final` declaration for every turn's repository work.
