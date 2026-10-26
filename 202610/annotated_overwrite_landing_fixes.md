---
tier: tale
size: medium
title: Finish annotated reference overwrite - landing fixes and closeout
goal:
  Annotated reference overwrite passes on macOS and Linux, never discards late editor
  saves or edits the wrong task, reports skipped links, and epic bob-cli-c9.5.1 plus
  phase bob-cli-c9.5 are closed with verified evidence.
proposed_by: bbugyi200.athena.bob-cli-c9.5.1.land
proposed_with: claude/opus
bead: bob-cli-c9.5.1
create_time: 2026-10-26 11:13:25
status: wip
---

- **PARENT:**
  [202610/annotated_reference_overwrite.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/annotated_reference_overwrite.md)
- **BEAD:**
  [bob-cli-c9.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.5.1.md)

# Finish annotated reference overwrite: landing fixes and closeout

## Context

This tale is the landing repair for epic `bob-cli-c9.5.1` (annotated reference overwrite
on the Mac, plan `plan:202610/annotated_reference_overwrite.md`). It sits under phase
`bob-cli-c9.5` of outer epic `bob-cli-c9`. All five child phases are closed. Their
commits on master are:

- `e4825208`: archive planning.
- `564168f3`: recovery.
- `af416080`: scan guard.
- `ca6065af`: execute.
- `8a1679e5`: acceptance tests, kit, and docs.

`just check` passes on athena. Landing review and master CI found epic-caused defects
that block closing. This tale fixes them and then performs the full closeout in the same
turn. It must never wait for its own commit, push, or CI run.

The epic plan's contract still governs and its decisions remain final:

- `legacy_force = no`
- `keep_created = yes`
- `memory_decision_record = no`

No memory notes change in this tale, no decision record is written, and no new CLI flag
is added. Do not edit the `Cargo.toml` version or `CHANGELOG.md`.

The pre-existing CI failures below are not part of this tale. Leave them alone. They are
recorded as a DISCOVERED ISSUE on `bob-cli-c9` and belong to that epic.

- `highlights_create_collection_overwrites_exact_set_with_listen` (pre-sync of the
  default `~/bob`).
- `highlights_create_force_adopted_empty_target_matches_missing_target_usage`.

## Problems to fix

### 1. macOS CI is red because of this epic

The `test (macos-latest)` job of master CI run 40333521246 (`ca6065af`) fails these
tests:

- `annotated_overwrite_moves_bundle_and_merges_task_link_edit_once`: panics at
  `execute.rs` "archived asset: NotFound".
- `failed_annotated_transaction_reverses_the_pdf_move_inside_accessor`: fails with
  "ignoring malformed overwrite intent with invalid path
  //private/var/folders/.../vault/ref/papers/report.assets".
- `tests/cli/highlights/overwrite_web.rs::annotated_web_target_refuses_without_network`
  and `::every_annotation_source_refuses_with_the_contract_remedy`. They assert the
  off-Mac text, but on macOS the key-off message
  `annotated overwrite requires highlights.coordinate_pdf_writes: true` wins.

The same canonicalization root cause also breaks these older macOS-only tests:

- `crash_without_rollback_is_recovered_to_the_old_state`
- `failpoints_that_abort_before_commit_restore_the_old_state`
- `unrelated_staged_and_dirty_files_survive_abort`
- `local_pdf_overwrite_listen_failure_after_narration_keeps_scratch`

They have failed since the c9.4 overwrite work; fix them along with this epic's tests.

The cause is `/var` → `/private/var`. `overwrite.rs` `checked_vault_path` returns a
canonicalized path, which becomes `note_absolute`. Later code strips it against the
non-canonical `config.bob_dir`, so the `.assets` path and other paths are either dropped
or recorded as absolute paths in the intent.

Requirements:

- Every path that reaches `touched`, `created`, intent moves/writes, preimages, or
  `assets_relative` must be vault-relative even when the vault root is reached through a
  symlink.
  - Strip against the canonical vault root wherever a canonical path is stripped, or
    keep the non-canonical joined path for relative computation.
  - Fix the root cause. Do not loosen intent path validation.
- Reproduce on Linux by running the overwrite unit and CLI subsets with `TMPDIR`
  pointing at a symlink to a real scratch directory. `std::env::temp_dir()` honors
  `TMPDIR`. The tests above must fail before the fix and pass after.
  - Add one focused regression that runs an annotated overwrite (and its rollback)
    against a vault whose parent path is a symlink.
- `tests/cli/highlights/overwrite_annotated.rs::coordination_key_on_still_refuses_annotated_overwrite_off_macos`
  writes `coordinate_pdf_writes: true`. On a Mac that would run a real coordinated
  overwrite, so mark it `#[cfg(not(target_os = "macos"))]`.
- Make the two `overwrite_web.rs` annotated assertions platform-aware.
  - Reuse or share the `assert_annotated_host_refusal` logic in
    `overwrite_annotated.rs`: off macOS, the host text plus the
    `-f starts a new reading from any host` hint; on macOS, the key-off message or the
    host text.
- Audit every test added by the five epic commits for the same macOS assumptions.
  - That covers `tests/cli/highlights/replaced_capture.rs`, `overwrite_annotated.rs`,
    `overwrite_recovery.rs`, and the `execute.rs`, `recovery.rs`, `archive.rs`, and
    `tests/marker_write.rs` unit tests.
  - Look for host-text asserts, the coordinated path running for real, and temp-path
    canonicalization.

### 2. The reading-task lookup matches block IDs by substring

`overwrite/execute.rs` `task_line_index` (added in `ca6065af`) and the older
`find_task_line` both pick the first line that contains `^{block_id}`. Reread IDs follow
`ref-<slug>-vN`, so `^ref-paper` also matches `^ref-paper-v2` or `^ref-paper-two`. A
reread task listed above the original in the same area note then gets edited or verified
instead of the real task.

Fix:

- Match the block ID exactly, as an end-of-line Obsidian anchor (token boundary after
  the ID).
- Prefer an existing helper, for example the block-id parsing in
  `collect_done/transform.rs` or `note_tasks.rs`, over a new parser.
- In `task_line_index`, prefer the planned `plan.task.line_index` when that line carries
  the exact block ID. Link rewrites preserve line count.

Add a regression with `^ref-report-v2` (and `^ref-report-two`) above `^ref-report` in
the task file. The run must edit and verify only the real task.

### 3. Recovery's reverse rename coordinates the wrong file and can fall back

All of these are in `overwrite/recovery.rs`.

- **Wrong file.** `PlatformReverseCoordinator::reverse_file` coordinates on `to`, the
  live path. That path is absent by construction. The open Highlights document follows
  the forward rename to the archive path (`from`). Coordinate on `from`, the file being
  moved, using `file_coordination::coordinate_merging_write`.
  - That function already runs the accessor directly off macOS, so call it
    unconditionally.
  - Remove the `for_explicit_write().map(..).unwrap_or(false)` fallback. A config error,
    or the key being turned off after a crash, must never produce an uncoordinated
    rename on macOS.
- **Hash proof outside the accessor.** Re-check the ownership hash proof inside the
  granted accessor, immediately before the rename, the way `execute.rs` rechecks.
  - On a mismatch, keep files and intent and fail closed.
  - Holds and failures keep the intent and the files.
- **Non-atomic no-clobber.** `rename_no_clobber` checks that the destination is absent
  and then calls `fs::rename`, which can replace a file that appears in between. Use the
  atomic `highlights_ref::io::rename_no_replace` (`renameat2 RENAME_NOREPLACE` /
  `renamex_np RENAME_EXCL`), keeping the same-filesystem and source-gone checks.
- **No recovery hint.** Fail-closed messages from `preserve_message` say only "left the
  intent in place". The epic contract requires a recovery hint.
  - Name the intent file path and the paths whose bytes no longer match.
  - Tell the user what to do: keep or commit the late save, move the conflicting file
    aside, then rerun a writing command or remove the intent.
  - Document the same manual procedure in the recovery section of
    `docs/highlights-ref-sync.md`.
- **Stale comment.** Update the doc comment above `impl PendingMove` that still says the
  constructors are test-only.

Tests:

- The reverse coordinator is invoked with the archive path.
- A hold or failure retains files and intent.
- A hash change injected between planning and the accessor fails closed.
- A destination that appears before the rename is never replaced.

### 4. Bob-created files are not published atomically

These are in `overwrite/execute.rs`. `write_new_file` (the frozen note) and
`publish_pdf_after_archive` (the replacement PDF) use `create_new` plus `write_all`. A
short write, ENOSPC, or crash leaves partial bytes. Recovery then fails closed forever
because the bytes match neither absence nor the recorded write, which blocks every
create, scan, and vault-sync on that host.

Fix:

- Write to a uniquely named temp file in the destination directory, `sync_all`, then
  publish with `io::rename_no_replace`. Remove the temp on any error.
- The destination must still be absent before publishing (no clobber), and it is either
  absent or complete after any failure.
- Make sure recovery and the commit manifest ignore or clean a leftover temp from a
  crash, and that temps are never committed. A hidden, recognizable name in the same
  directory is fine if recovery removes it.
- Add a failpoint or unit test that simulates a failed write. Rollback must then restore
  the old bundle automatically without leaving the intent.

### 5. Older archives are restored from HEAD on rollback

`overwrite.rs` `plan_annotated_archive` puts every earlier archive's note, annotated
PDF, and assets (`history` records) into `touched`. `recovery.rs` stores all
fingerprinted preimages in `intent.touched`, and `restore_from_base` runs
`git checkout <base> --` on them, uncoordinated.

The transaction never writes these files. Highlights may legitimately save annotations
into an older archived PDF after a crash, and any rollback or crash recovery then
silently discards those saves.

Fix:

- Keep history paths in the preflight fingerprint and dirty checks.
- Exclude them from the restore set: `intent.touched` and the in-transaction rollback.
  Restore only paths the transaction writes, moves, or deletes.
- Add a regression: a second annotated overwrite of a reference that already has one
  archive fails at a failpoint after an older archived PDF was modified. The older
  archive's bytes must survive rollback and crash recovery.

### 6. Inbound highlight links: ambiguous and dangling cases

Ambiguous links:

- `skipped_ambiguous` from `rename::links` is only hashed (`overwrite.rs` integrity
  fingerprint) and never reported. The contract says ambiguous bare targets remain
  "unresolved and reported".
- Print one `warning:` line per skipped ambiguous link (file and target) in dry-run and
  success output, through the shared `overwrite/local.rs` output.

Dangling links:

- An inbound `#^h-x` link whose anchor does not exist in the old note is rewritten
  today. Verification (`execute.rs`, "rewritten highlight anchor does not resolve") then
  fails after mutating the vault and rolls back, so one stale link anywhere in the vault
  blocks annotated overwrite with an opaque `verify_failed`.
- Only rewrite `#^h-` links whose anchor exists in the old note. Leave dangling ones
  unchanged and report them with a `warning:` line.
- Verification checks exactly the links that were rewritten.

Cover both cases with unit tests and one CLI dry-run assertion.

### 7. Smaller contract and accuracy fixes

- **Hold codes in the docs.** `docs/highlights-create.md` lists overwrite holds as
  `editor_active`, timeout, and `marker_changed`. Explicit writes never emit
  `editor_active`. The executor emits `coordination_timeout`, `coordination_failed`, and
  `marker_changed`, and the preimage-changed abort asks for scan and retry. Document the
  real codes.
- **Counts in `print_archive_line`** (`overwrite/local.rs`):
  - It prints "1 highlights". Pluralize the way the host refusal does.
  - The highlight-link count includes rewrites inside the live note itself, which are
    not written as inbound edits. Count only the inbound links actually moved, and keep
    the docs example consistent.
- **Frozen `parent`.** The frozen note's `parent` is built as
  `[[ref/{ref_type}/{stem}]]` in `render_frozen_note`, but the archive reader and
  verification require it to equal the live note path. Derive it from the live note's
  actual vault path (without `.md`), so a `ref_type:` that disagrees with the directory
  cannot make every attempt fail verification or hide the archive from the scan guard.
- **Recorded digest without the archived PDF.** In the scan guard
  (`overwrite/archive.rs`, around `validated_archive_file` in the annotated-PDF record
  reader), a recorded `annotated_pdf_sha256` is ignored when the archived PDF file is
  missing. The contract says the immutable recorded digest proves byte equality.
  - Keep the path-safety validation: vault-relative `ref_history` path belonging to this
    reference.
  - Do not require the file to exist for the hold.
  - Add a test that deletes the archived PDF and still holds the write-back.
- **Verbose scan output.** `report.rs` `print_verbose_scan_write_marker_lines` returns
  early when coordination is off and there were no plan-time holds. A write-back caught
  at execution time (`replaced_capture`) then has no verbose `pdf: … held` line. Also
  print when any write outcome is held.
- **Dead check.** `overwrite.rs` checks `plan.integrity_fingerprint().len() != 64`,
  which can never be true. Remove the dead check, or use the fingerprint for a real
  comparison (for example in `same_locked_preflight`). No `#[allow(dead_code)]`.

### 8. Test coverage the epic contract required

- Run the failpoint sweep against the annotated fixture too, arming every relevant
  `OverwriteFailpoint`:
  - `AfterIntentPersistence`, `AfterFrozenNote`, `AfterSidecarRemoval`,
    `AfterArchivePdf`, `AfterAssets`, `AfterPdf`, `AfterNote`, `AfterTaskEdit`,
    `AfterSyncWrite`, `ForcedVerificationFailure`, `BeforeCommit`.
  - Use a fixture that has assets and an inbound `#^h-` link.
  - Each must restore the exact prior bundle and links and clear the intent.
  - `CommitReportedAsError` and `PostSyncFailure` must keep the committed archive.
  - `CrashWithoutRollback` must recover through the v2 move reversal.
- Tighten the "no open annotation tasks in the archive" assertion in the execute tests.
  Today it ignores any open line containing `[h::`. It must assert there are no open
  task lines in the frozen note other than the neutralized banner.

## Verification gates

Run these and record real results in the closeout note:

- `cargo fmt --check` and `cargo clippy --all-targets --all-features -- -D warnings`.
  `just check` runs both. Use `just lint` or `just check` as the repo expects.
- The overwrite, recovery, scan, and sync subsets:
  - Unit: `highlights_ref::overwrite`, `rename::links`, `marker_write`.
  - CLI: `highlights::overwrite`, `replaced_capture`, `ref_scan`, `ref_sync`.
  - Run them both normally and with `TMPDIR` set to a symlinked scratch directory.
- `just check` (not `just check-full`). Use `/sase_monitor` if it risks a tool timeout.
- `just check-scripts` (includes the acceptance kit `--self-test`).
- `just docs-gen` then `just docs-check` (docs change).

macOS cannot run here. State explicitly that the macOS fixes were verified on Linux
through symlinked-temp emulation and `cfg` review. CI confirmation happens after the
push and is not awaited. Real Highlights acceptance remains Bryan's manual Mac step, and
the kit is ready for it. Never claim it passed.

A failure that reproduces identically on a clean master checkout is recorded with
evidence and is not this tale's work. One example is the two pre-existing
`tests/cli/highlights/create.rs` tests that depend on `~/bob` existing.

## Final step: close out the epic and its parent phase (same turn)

Do this after the code and gates above, in the same turn. Do not wait for the commit,
push, or CI.

1. Run `sase bead epic-symbols bob-cli-c9.5.1`. For every listed `--epic-symbol` entry
   (none were listed at planning time):
   - Resolve the symbol: wire it, privatize it, add a non-test pragma, or delete it.
   - Only if a still-open later bead needs the exemption, re-key the justfile line to
     that bead.
2. Close the epic: `sase bead close bob-cli-c9.5.1 --note "<verification>"`.
   - The note summarizes what was verified:
     - All five phases (archive, guard, recovery, execute, acceptance) and their notes
       were reviewed.
     - The fixes in this tale.
     - The gate results with real commands and outcomes.
     - The macOS-emulation caveat and pending manual Mac acceptance.
     - Integration review of outside commits since `e4825208`: `7b8c5ddb`, `69dfd7e8`,
       `3633a10a`, `01f7ec97`, `c7ced497`, `92706cb4`, plus any newer master commits.
       The source-store overwrite integration belongs to `bob-cli-cc.5`, which was
       notified.
     - The follow-up triage already recorded in the epic's notes.
   - Never use `--force` merely to make the close succeed.
3. Run `just symvision` if the recipe exists. This justfile had none at planning time;
   if so, say so in the note.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path shown by
   `sase bead read bob-cli-c9.5.1 -r "Need the plan path for closeout"`
   (`plan:202610/annotated_reference_overwrite.md`).
5. The epic's `parent_bead` is phase `bob-cli-c9.5` of outer epic `bob-cli-c9`.
   - Re-read it with
     `sase bead read bob-cli-c9.5 -r "Verify phase scope before closing"`. Confirm the
     child plan delivered its whole description:
     - Overwrite annotated references (highlights, image selections, `.assets/`) on
       macOS only, under `highlights.coordinate_pdf_writes`.
     - Rename the PDF and assets into `ref_history/`.
     - Move inbound `#^h-` links.
     - The scan write-back guard.
     - The extended acceptance runbook.
     - The refusal-with-hint off the Mac.
   - Check for new sibling notes.
   - Run `sase bead epic-symbols bob-cli-c9.5` and resolve or re-key any entries.
   - Close only that phase with
     `sase bead close bob-cli-c9.5 --note "<what you verified>"`. In the note, mention
     that its own note #1 (the missing recapture research reference) stays for the
     `bob-cli-c9` land agent.
   - Do not close `bob-cli-c9` or touch `bob-cli-c9.6`. The outer epic's land agent owns
     them.
