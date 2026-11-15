---
tier: tale
title: Repair subscription core parity and finish its landing
goal:
  Subscription history, row projections, and import previews agree on guarded inputs and
  exact money, and bob-cli-ls.2.1 plus its parent core phase close normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ls.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ls.2.1
status: done
---

- **PARENT:**
  [202611/subscription_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/subscription_core.md)
- **BEAD:**
  [bob-cli-ls.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ls/bob-cli-ls.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ls.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ls.2.1.land.md)
- **COMMITS:**
  - [b8aed5a](https://github.com/bobs-org/bob-cli/commit/b8aed5a943242a5517a6f536de9914432193ed90)
    — fix(bank): repair subscription core ledger, import after-state, and guards

# Finish subscription core and close its landing

## Outcome and boundaries

Repair the existing subscription core, prove its read/import agreement and guarded
inputs, and finish the landing of `bob-cli-ls.2.1` and its parent **phase**
`bob-cli-ls.2` in this coding turn. This is one bounded Rust repair and regression-test
unit; do not create another child epic or defer closeout to another land agent.

Read `sase bead read bob-cli-ls.2.1 -r "Need landing audit and follow-up outcomes"`, its
three closed children, and `bob-cli-ls.2` before editing. Audited context:

- `plan:202611/subscription_core.md` is the current epic's authoritative plan.
- `plan:202611/bank_subscriptions_album.md` supplies the parent core contract.
- `research:202611/bank_subscriptions_feature_design/bank_subscriptions_feature_design.md`
  supplies the match, import-loader, and attribution rationale; only those sections are
  needed. Use `sase artifact read` for sidecar artifacts.

The final decisions remain `payee_source = record_title`, `memory_decision_record = no`,
and `memory_glossary_term = no`. Edit no memory or generated AGENTS files, Cargo
versions, or CHANGELOG. Use fictional Tunely/Cloudbox fixtures only; do not inspect or
mutate the real bank vault. Preserve bank totals, write-once rows, typed-match ID-only
changes, and replay idempotence. Do not implement the outer phases' new Markdown
naming/link insertion, business findings, subscription report command, seeding, or
plugin features.

## Audited baseline and drift

At audit time HEAD and freshly fetched origin/master were `055f9b6e`. Reviewed epic
commits are `c59b5b80` (registry/matching), `96b35f41` (attribution/ledger and
completion test helper), and `055f9b6e` (loader/projection/docs). All three child beads
are closed, but their source and tests leave the gaps below. Existing matching, cents
parsing, calendar helpers, coverage merging, and suggestion tiers are useful
implementation building blocks; repair them rather than replace the feature.

Interleaved commits `ee6d6503`, `476c5440`, `d794e0ba`, and `b638fb0d` change Google
Calendar behavior/docs and require no subscription integration. `248befdd` adds the bank
config report, fingerprints, and located diagnostics. The subscription integration
already contains those changes: preserve `config_fingerprint`, all located config
findings, config-report usage counts, hidden-category rules, and shared rule matching.
The preceding `bd9c4c13` YAML-span extraction is already present and is not a competing
subscription implementation. Recheck commits since `055f9b6e` and integrate any new
consumers or conflicting edits before verification.

The epic's old unused-registry clippy note is addressed in source: `RecordField` was
removed and status/cadence/pricing now have consumers. Verify this with the normal gate;
do not declare the source review a passing check. No `--epic-symbol` entries existed for
`bob-cli-ls.2.1`; this checkout currently has no `symvision` recipe. Recheck both facts
during closeout.

## 1. Separate historical transactions from row projections

Repair `subscriptions/attribution.rs`, `ledger.rs`, and `projection.rs` together.

- `collect_universe` currently pushes unique orphan-ID rows through `typed_txn`, then
  `build_ledger` counts them like journal history. Historical ledger membership must be
  exactly folded journal transactions plus ID-less rows outside that account's merged
  coverage. Orphan-ID rows can still receive row projections. Separate the projection
  inputs/results from ledger-eligible transactions explicitly; a renamed flag or
  `journal_only=false` is not sufficient.
- Preserve row-level explicit-link precedence and safe ambiguity handling while
  excluding covered ID-less rows from historical totals. Include a covered typed row
  with an explicit subscription link in projection tests; ledger exclusion must not
  erase the row's own explicit relationship. Duplicate IDs must not copy an arbitrary
  current row's fields into history.
- Journal-only entries currently get `category: None` and the resolver has no config
  argument. Provide the shared resolver the effective config needed to classify
  journal-only evidence through existing read-time rule and built-in semantics. A
  fictional `TUNELY MOVE` rule categorizing an outflow as `transfer` must prevent
  pattern attribution even if a Tunely needle matches. Keep unique explicit row links
  authoritative; never infer income/refund from sign alone. Do not invent a second
  matcher or a learned row for absent journal-only evidence.
- Ensure every command can reuse the same typed attribution/ledger snapshot. Keep
  diagnostics deterministic and free of raw private descriptions.

Add behavioral tests proving orphan IDs do not affect gross/net/latest/price/timeline,
their row projections still work, covered explicit typed rows remain visible, duplicate
IDs remain deterministic, and journal-only rule exclusions work. Update the existing
`orphan_id_rows_are_row_only_and_not_extra_journal_entries` test: it currently checks
only an attribution flag, never whether the record ledger incorrectly counts the row.

## 2. Enrich the complete import after-state, including restore

`import_plan.rs::affected_rows` currently filters to wanted IDs first, passes the old
`input.journal`, and supplies `self.txn_events` only as a description map. That omits
new `before_start` transactions, new coverage, and current authored links/categories on
unaffected historical rows. A new row may also be matched on its cleaned payee rather
than its statement evidence. Repair the data flow:

1. Form a pure projected journal from the folded existing journal and this batch's new
   transaction/import events, preserving transaction-key/ID deduplication and
   `before_start` history. Use existing journal fold/identity semantics.
2. Form all projected bank rows from original scanned notes overlaid with planned
   after-images and new notes, preserving unaffected rows' current explicit links and
   categories. Compute one shared enrichment snapshot before selecting output IDs.
3. Select the affected/restored output rows only after enrichment. Ensure raw import
   descriptors reach the intended matching seam, while stored/read provenance follows
   the existing masking contract. The raw-description override must not be used only for
   recurring flags while attribution silently reads cleaned text.
4. In `import_run.rs`, source-free `bob bank import -F` currently bypasses
   `load_record_links` and supplies a missing registry. Load and guard the subscription
   registry for restored rows too. Missing/invalid album behavior must remain the same
   as other import paths, and no-op restore/replay must remain write-free.

Exercise a batch with before-start and written charges; an unaffected old row explicitly
linked to a different record than its descriptor matches; changed descriptors; a covered
typed row; a forced restore without input files; and replay. Assert the complete
subscription object, candidate/recurring flags, and relevant suggestions agree across
preview, apply, subsequent list, and report. Do not merely compare two direct calls to
`enrich_rows` with the same preconstructed journal.

## 3. Bind registry decisions to their guarded inputs

`subscriptions/loader.rs` uses one context and guarded record bytes, but its
`registry_dependencies` loop protects only entries already identified as albums.
Discovery also depends on parent/ancestor and other root-note contents. Root filename
membership alone cannot notice an existing non-album note becoming a conflicting album,
or an existing parent changing type/parent without a filename change.

Use the existing snapshot/registry facilities to capture the consulted semantic inputs
and bind the context to those exact bytes. Add the necessary parent/ancestor and root
claim preimages, including absent dependencies, to `ReadDependencies`; do not implement
an unguarded second discovery. Preserve record-folder and root membership guards,
including absent folders, and the restaurant exact-title path. Distinguish malformed
subscription declaration (one warning, import can proceed without facts) from unreadable
or racing membership (refuse).

Add plan-then-apply tests that change a subscription record field, its declaration, its
parent, an existing root note into a conflicting declaration, or record-folder
membership (including creating an initially missing folder). Each must refuse before
writing either bank rows or journal bytes. Retain the restaurant stale-input tests. The
current subscription test only asserts a dependency path was placed in a vector; that is
not apply-time proof.

## 4. Make money and timeline boundaries truthful

Repair the concrete arithmetic gaps in `ledger.rs`, `cadence.rs`, and row drift:

- Charge sums, negation, net subtraction, variable averages, and fixed four-weekly
  conversion currently saturate; refund totals use an unchecked `sum()`. Four-weekly
  `amount_cents = 1_000_000_000_000_000_000` has a representable monthly equivalent
  `1_083_333_333_333_333_333`, but the current multiply saturates, the half-up addition
  overflows, and `unwrap_or(0)` produces zero. Use checked widened intermediates and
  exact half-up rounding; no floats, wrapping, silent saturation, or fallback zero on
  arithmetic failure. Represent unrepresentable derived facts explicitly through the
  shared typed result/diagnostic path and suppress unavailable facts rather than emit
  invented money. Preserve independent attribution and ordinary bank totals.
- `current_step` only searches nonnegative steps and stops at 240. A monthly
  `renews: 2028-01-31` with today `2026-11-15` currently places all 12 buckets in the
  future. Locate today's cadence interval using a signed offset from the original
  anchor, with no arbitrary 240-step truncation. Preserve month-end/leap clamping,
  inclusive coverage, current upcoming interval, and strict missed-after-due behavior.
- Keep row states tied to qualifying charge rows: `to_cancel` only on the latest
  qualifying charge; `after_cancel` on charges satisfying that record's ended rule,
  while the record-level finding reference retains the newest offending charge. Refund
  rows do not become charge sentries or variable usage charges.

Test exact representable large calculations, overflowing totals and refund sums,
`i64::MIN` magnitude handling, rounding near the integer boundary, future renews more
than one cadence away, old anchors beyond 240 periods, month-end/leap dates, multiple
after-cancel rows, and refund exclusions. Keep checked failures observable,
deterministic, and isolated instead of panicking the whole command.

## 5. Verify the assembled core and documentation

Add the missing synthetic command-level subscription fixtures in `tests/cli/bank.rs` (or
a focused sibling module) and focused loader/apply tests. At the audited HEAD that CLI
file contains no subscription tests. Cover the requirements in steps 1–4 plus renamed
album noun binding, invalid/missing album warning bounds and null/false fields, rule >
subscription > learned/bank suggestions, hidden/unknown categories, selected month using
out-of-month history, unchanged authored categories, read-only byte preservation,
typed-row ID-only matching, restaurant compatibility, and byte-idempotent replay. Reuse
existing bank fixtures and freeze `BOB_NOW`.

Read the `docs.md` reference memory before touching docs. Keep `docs/bank.md` and
`docs/reference/json-outputs.md` consistent with the repaired behavior; do not add outer
phase commands prematurely. Run focused bank tests as needed, then `just check` (the
required file-change gate, equivalent here to fmt/lint/test) and `just docs-check`. Do
not run `just check-full`. Use `/sase_monitor` for a command that needs handoff,
including a `sase tool run check` escalation; the resumed coding turn must still perform
the closeout below. Never arrange closeout after this work's own commit SHA, push, or
CI.

The following unrelated follow-ups were fully triaged by the lander; do not create
duplicates or repeat votes:

- `bob-cli-ls.2.1.2` note #2 matches `bob-cli-g8`. The phase already corroborated it;
  `96b35f41` changes its test helper to `bob_completion_command`. The lander added a
  source-fix note and declined another task/vote. Preserve that change.
- `bob-cli-ls.2.1.3` note #1: canonical docs word counting reproduced 6055 words at
  bob-plugins `63ff165`, versus 6000 at `f72bd4e`. The Setup header phase `bob-cli-lt.3`
  caused the +55; its own note #1 acknowledges it. The lander routed a
  `DISCOVERED ISSUE` to active epic `bob-cli-lt` and corroborated `bob-cli-il` as a
  post-close recurrence (its earlier fixed source was 5546 at `61c506d`). No new task.
  If docs-check still fails only for this independent page, record the unchanged
  evidence and proceed with completed subscription scope. Do not edit the mounted plugin
  doc, raise its cap, or expand this tale into Bank Setup work.

## 6. Finish this epic and its parent phase in the coding turn

This is the tale's final work step. There is no later land agent to resume this landing.

1. Re-read `bob-cli-ls.2.1`, every descendant and note, and its linked plan; confirm all
   required repairs above and every original note are addressed. Recheck post-audit
   commits and descendant readiness, not just old closed statuses. Record exact gates
   and follow-up outcomes in the verification note.
2. Run `sase bead epic-symbols bob-cli-ls.2.1`. Resolve every listed exemption by
   wiring, privatizing, a justified non-test pragma, or deletion, or re-key it only to a
   still-open later bead which genuinely needs it. Do not leave entries keyed to this
   epic or a closing phase. Check temporary unused re-exports as well.
3. Close normally:
   `sase bead close bob-cli-ls.2.1 --note "<source/commit/note audit, repaired parity and guards, exact verification results, integration review, and both follow-up outcomes>"`.
   If rejected for symbols, fix them and retry. If rejected for incomplete phases,
   finish/reopen them; never use `--force` merely to make a close succeed or to advance
   this successful nested landing. Deliberate cancellation/supersession, if actually
   warranted, requires its separate explicit reason and non-done resolution.
4. Run `just symvision` if available after closing; otherwise record that this checkout
   has no such recipe. Open the plans sidecar through `/sase_repo` and set
   `status: done` in the frontmatter of **`202611/subscription_core.md`**, the epic PLAN
   artifact `plan:202611/subscription_core.md`. Use the path printed by
   `sase repo open plans`; never copy a numbered workspace path into commands or plans.
5. Run `sase bead read bob-cli-ls.2.1 -r "Need the parent link"`. Its audited parent is
   phase **`bob-cli-ls.2`**. Re-read that phase, verify this completed child fulfills
   the core scope, retire any entries from `sase bead epic-symbols bob-cli-ls.2`, and
   close only that parent phase normally with
   `sase bead close bob-cli-ls.2 --note "<core contracts verified, shared APIs, gate results, follow-up outcomes>"`.
   Leave containing epic **`bob-cli-ls`**, its plan file, and **`bob-cli-kt`** open for
   their assigned work/lander. If the phase is genuinely incomplete, note the concrete
   blocker on it instead of forcing it closed.
6. Finish through `/sase_final` with all changed repositories declared, including the
   plans sidecar. Closeout happens in the same turn as the final code, before the host
   creates that turn's commit. Report what closed and any genuine remaining blocker.
