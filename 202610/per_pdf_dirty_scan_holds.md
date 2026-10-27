---
tier: tale
title: Hold dirty reference targets per PDF and finish the rest of the scan
goal:
  Dirty reference targets hold only their owning PDFs while clean scans finish and Bob
  Refs shows retry status.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4
create_time: 2026-10-27 06:33:24
status: wip
---

- **PARENT:**
  [202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4.md)

# Hold dirty reference targets per PDF and finish the rest of the scan

Implement phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4` of
`plan:202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md`.
The epic's final decision is **dirty_pdf = hold**. This plan does not reopen that
decision. The underlying task is `bob-cli-b1`; leave that task and every ancestor bead
open for the parent epic's land agent. Close only the assigned phase after
implementation and verification.

## Scope and evidence

At planning time, bob-cli HEAD is `c2b69089`. The phase is in progress with no previous
work notes. Its implementation has not landed: both human and JSON full scans call
`ensure_safe_to_write` over all plans and return a hard error before executing any PDF
when one protected target is dirty. The existing test
`ref_scan_json_dirty_targets_error_envelope` explicitly expects this behavior. There are
no `--epic-symbol` entries attached to the phase at planning time.

This is one bounded, medium implementation: share a scan preflight result, route dirty
PDFs through the existing hold reporting, add a small Mac presentation extension, and
verify both sides of the existing schema. No additional epic is needed. Do not expand
this into a general scan rewrite or unrelated backlog work.

Read the assigned bead and approved parent artifact through their audited commands
before implementation. Use `/sase_repo` to open `bob-mac-capture`; all paths below are
relative to their owning repository. Read its `AGENTS.md`. Use `/sase_memory_read` for
`docs.md`, `decisions:open-pdf-writes-are-coordinated`, and
`decisions:mac-capture-is-a-thin-client`. This plan authorizes no memory edits.

Relevant existing implementation:

- bob-cli `src/native/highlights_ref/guard.rs`: `scoped_write_path_set` already
  enumerates note, marker PDF, image-asset, routed-task, and reading-task writes,
  together with the existing dirty-veto exemptions. `dirty_entry_allowed_for_plans`
  permits narrowly defined unstaged edits to a reference note. The guard omits
  replaced-capture holds before querying Git.
- `sync.rs::scan_library_write` and `scan_json.rs::scan_library_json` have separate
  execution loops. `assemble_full_scan_entries` pairs one write outcome with each
  planned PDF in scan order. Skipping a plan without recording an outcome would
  misassociate later results.
- `model.rs::ScanHold`, `marker_write.rs::scan_hold_for`, the human report, JSON `held`
  array, and `summary.held` already represent non-failure whole-PDF holds.
  `scan_receipt.rs` persists the same envelope printed by JSON mode.
- Bob Mac Capture `Sources/RefsCore/RefsScan.swift` decodes hold reasons as strings, so
  another reason fits schema 1. However, `RefsPDFUpdates.swift` drops unknown reasons
  from presentation. The receipt observer uses this classifier to update the panel.
  `RefsFetching.scan()` invokes `ref scan -a -w -f json`, with the existing fallback for
  older bob versions; `RefsLibrary` refreshes after a scan.

## Required behavior

For a writing full scan, a dirty protected target holds only the PDF plans that would
write it. Those plans perform no PDF, reference-note, annotation-task, reading-task, or
image-asset writes. Clean independent plans still execute. There is no new force flag,
dirty-PDF exemption, or uncoordinated fallback. Preserve the existing reference-note
allowances and reading/default-task destination exemptions; do not widen them. "Dirty
target" below means a path rejected by that existing policy, not an already permitted
authored edit.

Use `reason: "dirty_targets"` in the existing five-field hold row
`{pdf, note, title, reason, message}`. Include the rejected vault-relative target paths
in a deterministic message, ending with `retrying next scan`, for example
`dirty vault targets: ref/chat/memo.md · retrying next scan`. Emit one hold per PDF even
when several of its paths are dirty. Preserve deterministic scan order.

A scan with only holds succeeds: exit 0, `ok: true`, `failures: []`, and an accurate
`summary.held`, even if every PDF is held. Held PDFs contribute no successful write
counts or `notes` rows. Genuine plan/write failures retain exit 1 and the
partial-failure envelope; discovery, collisions, and Git/preflight infrastructure errors
retain hard errors. Holds never mask those failures.

The wire shape stays schema 1 and preserves field order. Adding a reason is
decoder-compatible, but changing dirty scans from an error envelope/exit 1 to holds/exit
0 changes the command contract: mark the landing description with `!` and describe that
migration. Keep the old Mac hard-error fixture coverage because older bob versions can
still return `error.code: dirty_targets`.

Apply this policy to writing full scans in both human and JSON modes, including attended
scans and the `highlights scan` alias. Preserve dry-run planning and receipt rules.
Targeted `ref sync`, create-time intake, `scan --intake-only`, and overwrite retain
their existing refusal/queue contracts; do not change them as an accidental consequence
of extracting the shared guard.

## Implementation

1. Extract a data-returning preflight in `guard.rs` that associates rejected paths with
   their owning plans, while retaining `ensure_safe_to_write` as the existing
   error-producing adapter for non-scan callers. Build the relevant path sets from
   finalized plans, preserve replaced-capture precedence, and obtain the Git status
   before any per-PDF execution. Share that snapshot across the scan so a successful
   earlier PDF's write to a shared destination cannot falsely make a later plan appear
   externally dirty. Do not implement this by re-running the old guard just before each
   PDF after earlier writes have occurred.

   Association must be exact and conservative: a protected shared dirty target holds
   every plan whose policy rejects that target. An allowance for one plan must not
   accidentally authorize another plan's explicit routed write. Preserve existing veto
   exemptions per plan. A held plan never reserves execution-state dedupe entries or
   contributes another PDF's task writes.

   The current Git parser uses human `--short` text and stores the printed path
   verbatim. Do not introduce a safety hole by matching that quoted/rename text against
   filesystem paths. Use a structured NUL-delimited porcelain status read for the new
   association, handling rename/copy source records and quoted/non-ASCII filenames
   correctly. Keep doctor output meaningful if the shared `git_status` representation
   changes. An unclassifiable status must fail closed, never silently allow a write.
   Keep existing missing-Git/non-worktree behavior and other guard policy outside this
   change.

2. Add `HoldReason::DirtyTargets` and its stable slug/message support. Feed the
   preflight into both full-scan execution loops. Each planned PDF contributes exactly
   one `ScanWriteOutcome`: dirty holds bypass `execute_pdf_sync` entirely; permitted
   plans use the existing executor and coordination policy. Prefer one small shared
   helper for this branching rather than duplicating policy. Preserve replaced-capture
   holds, collision validation, preimage checks, lock order, and the coordination time
   budget. Do not consume coordination budget for dirty holds.

   Reuse the existing hold renderers, counts, and receipt assembly. Verify concise and
   verbose output both identify the held PDF and retry message. Do not replace the
   normal completed-scan receipt with a scan-wide dirty error. Intake moves that precede
   this preflight retain their existing reporting semantics.

3. In Bob Mac Capture, add dirty holds to `RefsPDFUpdates` classification and append a
   truthful footer clause such as `1 PDF held for local changes · retrying next scan`
   (pluralize for several). Do not call these reading or coordination-timeout holds.
   Reuse the receipt presentation path; no vault inspection or new command is needed.
   Dirty holds remain quiet in hidden-panel notifications and do not create the
   `Highlights needs you` dialog warning. Ensure an all-dirty result is visibly held
   rather than looking like an unexplained no-op.

4. Update bob-cli `docs/highlights-ref-sync.md`: Holds, scan dirty-preflight behavior,
   JSON reason vocabulary/error table, exit status (including all held), receipts, and
   Expected Failures. Retain the distinction from targeted sync and intake. Update Bob
   Mac Capture `docs/refs.md` at its scan/update presentation contract. Do not edit
   `AGENTS.md`, memory files, versions, or `CHANGELOG.md`.

## Verification

Use temporary fixture vaults only. No live-vault edits or deployment is required. Run
builds through the configured `$CARGO_TARGET_DIR`.

- Add scan integration coverage with one dirty PDF marker target among clean PDFs under
  `-w`, and a separate dirty note target case. Verify the clean output really changes,
  while every held PDF's owned output bytes stay unchanged and no new task/asset side
  effects appear. Test both human and JSON entrypoints, including `-a -w -f json`, and
  check concise plus verbose retry text.
- Cover all dirty (exit 0, all held, zero writes), dirty plus a genuine per-PDF failure
  (exit 1), shared explicit dirty destinations, two clean plans sharing a destination,
  and existing permitted dirty reference-note/default-task edits. Verify
  staged/untracked protected targets and filenames requiring Git quoting remain
  protected, including rename parsing. Exercise these at the appropriate guard or
  integration layer without duplicating the same assertion everywhere.
- Update `ref_scan_json_dirty_targets_error_envelope` and the JSON/human dirty receipt
  tests in `tests/cli/highlights/scan.rs` to the hold contract. Assert the schema, field
  order, relative paths, hold ordering/counts, absent error, and exact JSON
  stdout/receipt byte equality. Preserve actual hard-error receipt coverage using an
  output/intake collision instead of dirty targets. Confirm dry runs still write no
  receipt and targeted-sync dirty refusal tests still pass.
- Add Mac fixtures for mixed clean/dirty and all-held responses. Extend portable
  `RefsScanTests`, `RefsPDFUpdatesTests`, and receipt-reader tests as appropriate:
  decode schema 1, preserve created/updated rows, classify dirty holds, show retry
  wording, and avoid false busy warnings or hidden notifications. Keep old-server and
  unknown-field decoding coverage.
- Test Bob Refs' actual `.scan`/⌘S route using the existing fake-bob harness and
  `FAKE_BOB_REFS_SCAN_FIXTURE`, including post-scan refresh and hold presentation
  through the receipt observer. Assert mixed success still exposes clean results,
  all-held does not show a failure banner, and dirty retry wording reaches the footer.
  Use bounded async waits. Relevant app tests are in
  `Tests/BobMacCaptureTests/RefsPanelModelTests.swift` and
  `RefsScanReceiptObserverTests.swift`; portable tests alone do not prove that route.

Run focused Rust tests, then the full `just check` (never `just check-full`) and
`just docs-check`. In Bob Mac Capture run `just portable-check` and its docs check.
Never land Swift that has not compiled. Follow that repository's instructions for a
best-effort isolated Mac build; never use Bryan's checkout or install the app. Before
landing new macOS app tests, obtain an actual macOS CI Build/Test verdict for the
changed tree. Prefer batch 17's macOS precommit gate if it has landed. Otherwise use the
parent epic's authorized scratch-branch/draft-PR workflow and remove the temporary
branch/PR afterward. Do not mistake a lint failure followed by skipped Build/Test for
verification, or a type-check for executed tests. Use `/sase_monitor` for long checks
and CI waits; wait for the handoff command to exit.

If a full check fails, compare the same failure with the clean base before attributing
it. A failure reproduced identically on the base does not keep this phase open: append
the evidence (and any existing tracking task) as a `PROPOSED FOLLOW-UP:` note on this
phase. Do not create task beads. Fix regressions caused by this work before closure.

## Completion

Record the implementation, Rust/Swift/docs verification, exact macOS CI result, and any
limitations on the assigned phase for the parent lander. Account for both changed
repositories in the normal SASE finalization flow. Leave `bob-cli-b1` and all ancestor
beads open; the parent's lander owns their closure and aggregate release checks.

Immediately before closing, run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4
```

Resolve any remaining symbols or re-key their Justfile entries to a still-open owner,
then close only this phase with evidence:

```sh
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.4 --note "<implemented behavior and verification>"
```
