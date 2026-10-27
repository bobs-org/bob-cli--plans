---
tier: tale
title: Coordinate annotated-overwrite archive moves and recovery
goal:
  Preserve the open annotated document and its save destination through archive
  replacement and both rollback paths, with native presenter evidence and explicit
  real-app acceptance limits.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3
status: done
---

- **PARENT:**
  [202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3.md)
- **COMMITS:**
  - [e557b02](https://github.com/bobs-org/bob-cli/commit/e557b021244a5aef656cbef0fcb14f6aea71aa53)
    — feat(ref): coordinate annotated-overwrite archive moves under open PDFs

# Coordinate annotated-overwrite archive moves and recovery

Complete only phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3`
(`archive-move-presenter`) from
`plan:202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md`. Its
target report is `bob-cli-d0`; leave that task and all ancestor beads open for the
batch-16 lander. Read the assigned phase and the parent plan through their audited
commands before implementing. The parent decisions `plugin_ignore = warn` and
`startup_smoke = implement` are final and concern sibling phases.

This is one medium coding unit: reuse the existing bounded move coordinator, presenter
fixture, overwrite failpoints, and recovery tests. No new platform architecture or
public interface is needed. Planning found no prior phase work notes and no completed
fix at baseline `2f004d77`. Recheck current source and notes before implementing to
avoid duplicating concurrent work.

## Findings and intended behavior

The current code still has three related paths:

- `src/native/highlights_ref/overwrite/execute.rs` wraps the whole transaction in
  `coordinate_explicit_pdf_write_with`, then `move_annotated_bundle` calls
  `rename_no_replace` on the annotated PDF without move notifications.
- Its immediate rollback calls `rollback_transaction_with_external_coordination`; the
  `ExternalCoordinationReverseCoordinator` also performs a raw inverse move. There are
  several rollback call sites, including intent, write, and commit failures.
- `overwrite/recovery.rs::PlatformReverseCoordinator::reverse_file` requests merging
  access on the archived PDF, verifies archive/replacement hashes, removes only the
  verified transaction-owned replacement, and reverse-renames without notifying the
  presenter.

`src/native/file_coordination.rs` already provides `coordinate_moving_write` and an
accessor-scoped `MoveContext` with `move_forward`/`move_back`. Its macOS backend uses
`ForMerging | ForMoving` and balanced `willMoveToURL`/`didMoveToURL` notifications; a
failed physical move reports the unchanged URL. The policy-aware
`MarkerWriteSession::coordinate_explicit_pdf_move_with` and fake notifier already exist.
The existing native tests cover moving to an absent destination and an immediate
inverse, but do not install a distinct replacement at the original path.

The user-visible destination is already specified by acceptance steps 13 and 15 in
`scripts/dev/highlights_coordination_acceptance`: the open annotated document follows
its retained PDF into `ref_history/`; later edits save into that archive, leaving the
new live capture unchanged. A successful rollback returns the document to the restored
live path. An uncertain or changed preimage is preserved and held. The original task's
speculation that following the archive could itself be wrong does not override these
existing acceptance criteria.

Read `decisions:open-pdf-writes-are-coordinated` and `docs.md` with `/sase_memory_read`.
Keep one bounded scope around the entire per-PDF transaction, save-before-write,
in-accessor revalidation, and no uncoordinated fallback. Never use `.forReplacing`,
automate Highlights UI, or change feature enablement. No memory edits are authorized. If
implementation reveals an inaccurate decision scope, record the evidence in a note on
the already-existing `bob-cli-d1`, as the parent plan directs; do not edit memory or
create a task.

## 1. Capture the missing native evidence

Extend `src/native/file_coordination/macos_tests.rs`, reusing
`tests/fixtures/file_coordination/presenter.swift` and its real AppKit save/reload
behavior. Do not override move notifications or manually change the document URL in the
fixture to manufacture success.

Add bounded characterization cases for the old behavior, and strict regression cases for
the selected safe behavior:

1. Open the original file, make an unsaved edit, obtain merging access, observe the
   saved bytes inside the accessor, raw-rename to an archive, and install different
   replacement bytes at the original path before releasing access. Record the document
   URL/content/dirty state and the bytes at both paths after a later edit and explicit
   save. The old-behavior control may expose an unsafe destination; its output must
   describe that observation rather than call it a successful fix.
2. Model crash recovery separately: a presenter holds the archived original while a
   distinct transaction-owned replacement exists at the live path. Exercise the old
   merging-only reverse sequence, record the presenter URL and later save location, then
   cover the notified reverse sequence with strict assertions.
3. Exercise a forward move, replacement installation, and immediate rollback inside one
   move accessor. After the accessor, edit/save through the still-open presenter and
   assert it writes only the restored live document.

The strict forward case must prove saved annotations reach the archive, the presenter
follows that archive, and a later save changes the archive while the replacement stays
byte-for-byte unchanged. The strict inverse cases must prove the restored URL receives
later saves and the archive path is not recreated. Use temporary fixtures only. Keep
waits bounded and preserve fixture cleanup.

Distinguish primitive characterization from production preflight: an actual overwrite
whose presenter saves new bytes after preflight must still refuse before Bob mutates
anything. It must not silently rebase the overwrite to those bytes.

Make the new cases observable on macOS CI, for example with a narrowly filtered
`cargo test --lib ... -- --nocapture` step in `.github/workflows/ci.yml`. The evidence
must identify native execution and the observed URLs/save destinations. A missing Swift
compiler, missing login session, or unregistered presenter is not a passing native
verdict; fail the required CI cases or clearly withhold the verdict. Keep ordinary
unsupported-host behavior for existing tests. Cross-compilation proves type correctness
only.

Use these observations to substantiate the coordinator choice. The implementation
default is the existing explicit move scope, consistent with the acceptance contract.
Retaining merging-only behavior is permitted only if real macOS evidence establishes the
same guarantees for forward move plus replacement, immediate rollback, and separate
recovery; document that evidence and retain regression coverage. Do not infer this from
an inode check or a Linux fake.

## 2. Wire the transaction and both inverse paths

For the explicit-move implementation:

- Select one outer `coordinate_explicit_pdf_move_with` scope for annotated overwrites,
  from the locked live PDF to its preflighted archive PDF. Keep ordinary unannotated
  overwrites on their existing merging path. Preserve the production refusal of
  annotated writes off macOS or with coordination disabled.
- Pass a narrow scoped move capability through `apply_and_verify` and
  `move_annotated_bundle`. Run the same absence, safe-parent, same-filesystem, preimage,
  hash, and inode checks; use `move_forward` for the actual PDF rename. Preserve asset
  moves, note/link/task updates, source-store handling, intent persistence,
  verification, and scoped Git commit behavior.
- Keep the outer accessor active through installation and every immediate rollback
  opportunity. Thread its inverse capability into recovery so PDF restoration calls the
  same context's `move_back`, after validating/removing only the owned replacement. Do
  not nest another coordinator or substitute a no-op notifier for the inverse. Before
  the forward move has happened, rollback must remain a valid no-move operation. Choose
  the smallest safe callback/adapter or explicit mutability change needed by the
  existing `ReverseCoordinator` seam; do not leak a move token beyond its accessor
  lifetime.
- Route independent `PlatformReverseCoordinator` recovery through a fresh bounded moving
  scope from archive to live. Recheck both recorded hashes inside that accessor before
  deleting the replacement; perform the inverse with that scope's forward operation.
  Preserve no-clobber behavior, safe paths, same-filesystem checks, recovery intent
  retention, and existing error semantics.
- Never move or delete an archive changed by a later editor save. A timeout, coordinator
  failure, changed source, changed replacement, or destination race must preserve the
  relevant user bytes and the recoverable intent. Keep the current safeguards against
  restoring older historical archives from stale snapshots. No expanded public CLI/JSON
  contract or intent-format change is expected.

This work stays within bob-cli. Do not absorb the source-pointer sibling phase or rework
ordinary ref rename/capture publication. Coordinate source overlap by preserving
neighboring changes.

## 3. Prove production routing and failure behavior

Extend existing fake-coordinator tests instead of building a second test framework.
Tests must exercise the production overwrite/recovery seams as well as the native
primitive, so correct isolated notifications cannot hide an unwired transaction.

- Annotated success: one outer coordinator call, forward notification pair, original
  inode retained in archive, and correct replacement, notes, task identity, links,
  source store, and commit scope.
- Immediate rollback after archive and after replacement: one coordinator call and
  forward plus inverse notification pairs, restored original, no archive leftovers.
  Cover the existing failpoint sweep and commit failure paths; a failure before the move
  sends no successful move notification.
- Save-before-access modifies PDF bytes: preserve the presenter save, report the
  existing preimage/marker refusal, and write no Bob transaction. After a safe
  sync/replan retry, archive those saved bytes.
- Independent crash recovery: notified inverse with replacement present and absent;
  timeout/failure does not delete the replacement; an archive changed before access or a
  changed/racing live destination refuses without overwriting user bytes.
- Preserve existing no-clobber notification behavior, repeated recovery behavior,
  non-macOS refusal, and feature-off behavior. Keep all current annotated failpoint and
  late-history-save tests passing.

Prefer focused additions to `overwrite/execute.rs` and `overwrite/recovery.rs` tests.
Extend the fake in `marker_write.rs` only as needed to observe scope and notification
order. Existing test names include
`failed_annotated_transaction_reverses_the_pdf_move_inside_accessor`,
`annotated_failpoint_sweep_restores_bundle_links_and_clears_intent`,
`reverse_hash_is_rechecked_inside_the_accessor`, and
`reverse_rename_never_replaces_a_destination_that_appears_late`.

## 4. Make the real-app acceptance evidence precise

Update `docs/highlights-ref-sync.md` and the existing guided acceptance script to
explain the archive destination and rollback behavior actually implemented. Strengthen
existing steps 12–15 instead of adding a parallel runbook:

- Record which URL the same open Highlights tab follows, whether a conflict dialog
  appears, and where a new edit/save lands.
- Snapshot the live and archive digests before prompting the user to make/save a fresh
  archive annotation, then compare them before the script deliberately plants
  `replaced_capture` bytes. Currently `live_before` is captured after the prompt, which
  cannot establish that the user's save left the replacement untouched.
- Resolve the archive belonging to this overwrite from its live note/history metadata,
  rather than relying on the last filename in a global sorted search.
- Document separate reverse/rollback acceptance with an isolated test fixture or
  test-only harness; do not add a production failure switch or damage the live vault to
  trigger recovery. State clearly when reverse behavior has only native fixture coverage
  and still lacks real-Highlights acceptance.

Preserve the original enablement verdict and current default-off setting. Bryan's real
Highlights session is a separate required acceptance limitation; do not claim it ran on
athena. No live vault or configuration edits are part of this plan.

## Verification and completion

Build using the configured `$CARGO_TARGET_DIR`. First run focused overwrite, recovery,
and file-coordination tests. Before landing any macOS-only Rust, run the parent's
cross-target type gate with the installed `aarch64-apple-darwin` target:

```text
cargo clippy --target aarch64-apple-darwin --all-targets --all-features -- -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces
```

For bundled C dependencies on athena, use temporary stubs via `CC_aarch64_apple_darwin`
and `AR_aarch64_apple_darwin`, with `CRATE_CC_NO_DEFAULTS=1`: the compiler creates its
`-o` output and the archiver creates a valid empty archive header (`!<arch>` followed by
a newline). Follow the parent's procedure; keep stubs out of source control and scope
those variables to this command. A successful stubbed check must never be described as a
native build or runtime test.

Run `just check`, `just docs-check`, shell syntax validation, and
`scripts/dev/highlights_coordination_acceptance --self-test` on the final tree. Use
`/sase_monitor` for long commands and CI waits, and wait for the handoff command itself
to exit. Do not end a turn with a command still running.

Native macOS CI must run the new cases on the actual implementation SHA and expose their
observations. If that evidence is available during implementation, inspect it and fix
any attributable failure. As explicitly assigned by the parent plan, the batch-16 lander
owns checking the landed SHA's macOS job, including steps after lint, and confirming the
native baseline/strict verdicts before resolving `bob-cli-d0`. Give that lander exact
test names, SHA/run evidence where available, and any pending native observation. Do not
label pending or skipped CI as verified. The real-Highlights guided run remains
separately unrun until Bryan performs it.

If a check fails identically on the clean base, record a `PROPOSED FOLLOW-UP:` note on
the assigned phase, citing an existing task when known, and close the phase after its
own work passes; a baseline failure does not justify leaving it open. Do not create any
beads. Other discovered work is also a `PROPOSED FOLLOW-UP:` on this phase for the
parent lander to triage.

Record baseline, changes, test results, native evidence/pending checks, and the real-app
limitation on the assigned phase. Before closing, run:

```text
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3
```

Planning found no entries; recheck at completion and resolve any new entry or re-key it
to a still-open owner. Once the implementation and phase-owned verification are
complete, close only this phase with:

```text
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.3 --note "<specific verification and native/real-app limitations>"
```

Never close `bob-cli-d0`, the batch epic, or any ancestor plan bead from this phase. Do
not change versions or `CHANGELOG.md`. Use the required `/sase_final` declaration for
normal turn completion so the host commits the completed work. The wider backlog audit
and successor-epic obligation remain with the batch-16 lander.
