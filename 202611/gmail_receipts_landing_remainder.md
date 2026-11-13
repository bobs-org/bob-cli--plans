---
tier: tale
title: Finish landing Gmail archive receipts and explicit undo
goal:
  bob gmail unarchive has the plan's fault coverage and complete-flow tests, the reopen
  hint names the undo command, an acceptance matrix maps receipts behavior to real
  tests, and bob-cli-kx.3.1 plus its parent phase bob-cli-kx.3 are closed.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kx.3.1.land
proposed_with: claude/opus
bead: bob-cli-kx.3.1
create_time: 2026-11-13 14:50:22
status: wip
---

- **PARENT:**
  [202611/gmail_archive_receipts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_receipts.md)
- **BEAD:**
  [bob-cli-kx.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kx/bob-cli-kx.3.1.md)

# Finish landing bob-cli-kx.3.1: unarchive fault coverage, reopen hint, acceptance matrix, closeout

## Context

Epic `bob-cli-kx.3.1` (Gmail archive receipts and explicit undo, plan
`plan:202611/gmail_archive_receipts.md`) has all six phases closed. Its land agent
verified the landed code in bob-cli (`c994cfbd`, `59b6149a`, `2ddfc944`, `cd6aefae`,
`01ab0767`) and bob-plugins (`b519508`, `e102fe2`). It already committed these landing
fixes, so do not redo them:

- Clippy deny-errors from the epic: `src/native/gmail/unarchive.rs` (needless_borrow,
  useless_conversion) and `tests/gmail_support/mod.rs` (collapsible_if). `just lint` is
  green.
- `src/native/catalog/table.rs`: an `Excluded` row for `gmail unarchive`. This fixes
  `native::catalog::tests::public_leaves_are_entries_or_excluded`.
- bob-plugins `plugins/bob-mail/src/046-archive-banner.js`: the Move-back argv is now
  `["gmail", "unarchive", "-f", "json", "--", note]`. The old `-- note -f json` made
  clap treat `-f`/`json` as extra note operands and print human text. The fix is
  rebuilt, tested, and deployed with `bob plugins sync`, and `src/native/gmail/cli.rs`
  pins it with `unarchive_banner_argv_keeps_json_and_one_exact_operand`.

The audit found three gaps against the epic plan. Close them, then land the epic. Read
the epic plan's "Undo contract" and "Gates and landing evidence" sections first:
`sase artifact read plan:202611/gmail_archive_receipts.md "<why>"`.

## 1. Reopen restore hint names the undo command

Sync's reopen hint still says the command does not exist.
`src/native/gmail/sync/report.rs` renders `archive_hints` as
`restore stays manual: Move to Inbox in Gmail`, in both the human line (near the
`↩ reopened` print) and the JSON `message`. The epic plan requires updating restore
hints that claim the command is absent.

- Keep `kind: "restore_to_inbox"`, the `thread_id` and `note` fields, quiet behavior,
  and `inert_single_line` handling.
- The human line names `bob gmail unarchive -- <thread_id>` with the row's exact thread
  ID. The JSON `message` names `bob gmail unarchive`. Reopening still never moves mail.
- Update the asserting tests: the `report.rs` unit tests, and
  `reopening_an_archived_task_reports_a_restore_hint_without_writing` plus any
  `gmail_sync` assertion on the text. Then update `docs/gmail.md` and
  `docs/reference/json-outputs.md` if they quote the wording.
- Leave the persisted note banner text in `src/native/gmail/note/receipt.rs` unchanged:
  bob-mail's trusted-banner recognizer parses it.

## 2. Unarchive fault and fence coverage

`bob gmail unarchive` has only six integration tests (search
`tests/gmail_archive_apply.rs` for `fn unarchive_`) and no unit tests. Add fake-gog
integration tests for each case below. You may move the unarchive tests and their
helpers (`create_owned_receipt`, `seed_*`, `note_text`, `journal_has`,
`remote_write_calls`) into a new `tests/gmail_unarchive.rs` that uses
`mod gmail_support;`. Use only the existing fake-gog controls in
`tests/gmail_support/{mod.rs,fake_gog.py}`, such as `set_failure`, `set_fail_after`,
`make_archive_journal_readonly_after_batch`, `set_batch_race_before_modify`,
`write_vault_before_metadata_read`, and the `modify_grant` scenario. Add a fake hook
only if none fits.

Every test asserts that task notes and unrelated mail notes stay byte-identical. The
target note may change only by its receipt projection, and only on success.

1. **Wrong writer (live):** the configured writer is another host. Expect exit 2, a
   typed setup error, and no guard, journal, note, or remote writes. Dry-run on the same
   host still works (already covered).
2. **Missing modify scope (live):** fake gog without the `modify_grant` scenario. Expect
   exit 2, kind `modify_grant` with the login hint, and no writes.
3. **Partial multi-input failure:** two owned receipts on two threads, where one
   thread's recorded-message metadata read fails. `results[]` keeps both per-input rows
   (`restored` and `failed`), with `ok: false` and exit 1. The restored thread is
   settled (`gmail.restored`, note cleared). The failed thread keeps its receipt,
   banner, and labels.
4. **Journal append failure after remote success:** use
   `make_archive_journal_readonly_after_batch`. Expect a `failed` row of kind `journal`,
   INBOX added remotely, and the receipt still owned. After the journal is writable
   again, a retry settles through `already_in_inbox` with no second batch modify. It
   appends exactly one `gmail.restored` and clears the note.
5. **Ownership changed concurrently with sync:** release the receipt between the remote
   metadata read and the mutation, for example by a hook that appends a matching let-go
   or runs a sync that lets go. Expect a typed `stale_ownership` failure, no batch
   modify, and no `gmail.restored`. Also cover the sequential case: after sync lets go
   on a reply, unarchive returns `unchanged`/`no_owned_receipt` with zero remote writes.
6. **invalid_grant stops remaining mutations:** with two inputs, the first restore fails
   `invalid_grant`. Expect exit 2 and no mutation for the second input. Its row is still
   reported.
7. **Policy exclusion:** a recorded message carries SPAM or TRASH. Expect `failed` with
   `policy_excluded` and no mutation.
8. **Identity rules:** an ambiguous stem that matches two notes, and a note for another
   account, both fail without mutation. A raw thread ID with no receipt returns a typed
   `unchanged`. Duplicate inputs collapse to one row (already covered; keep it).
9. **Unsafe journal:** each input gets a `failed` row with `unsafe_journal`, exit 1, and
   no mutation.

If a test exposes a defect in `src/native/gmail/unarchive.rs`, fix it inside the epic
plan's undo contract. Never broaden the message set, use `threads.modify`, remove the
enrollment label, or hold the vault lock while waiting on Gmail.

## 3. Complete synthetic flows

Add these integration tests:

- **Archive → visible receipts → explicit restore → no rearchive.** After a live
  archive, `gmail list -a -f json` shows the row's `archive_receipts` with the exact
  `message_count` and label, and sync JSON has `recent_archive.count == 1` with
  `state: "archived"`. After `gmail unarchive`, the list row's `archive_receipts` is
  `[]` and the note's `gmail_archived` and banner are gone. The next live sync makes no
  batch modify, reports no new `archived` row, and reports the `recent_archive` item
  with `state: "restored"`.
- **Reopen → later close → new archive generation.** Archive, then reopen the task. The
  receipt and banner stay and Gmail is untouched. Run `gmail unarchive`, then close the
  task again. The next live sync archives under a new close generation: a distinct
  `archive_event_id`, an updated `gmail_archived`, a banner again, and `recent_archive`
  counting the thread once with its latest event.
- Archive → reply/manual Inbox → let-go/clearing is already covered by
  `receipt_lifecycle_materializes_banner_and_clears_on_let_go` and
  `manual_move_to_inbox_lets_go_without_removing_inbox`. Cite those; don't duplicate
  them.

## 4. Acceptance matrix

Extend the conformance doc comment at the top of `tests/gmail_archive_apply.rs`, or add
one to the new unarchive test file and cross-reference it. Add a "Receipts
(bob-cli-kx.3.1)" section that maps each item below to real test names. Grep every name.
The items are:

- C1/C2/C5 receipt bytes and notification
- C9/C10 clearing
- C11 no auto-restore
- C14 recovered receipt visibility
- C16/C19 read-only preview
- C20 inert output, argv, and journals
- `recent_archive` timezone/DST and generation cases (`archive_projection` and
  `archive_state` units)
- each undo fault case from section 2
- the three complete flows
- the plugin suites `scripts/test-bob-mail-archive-banner.cjs`,
  `scripts/test-ledger-tools-mail-link-marks.cjs`, and
  `scripts/test-ledger-tools-gcal-events.cjs` in bob-plugins, cited by file

## 5. Verification

- `cargo fmt --all -- --check`, then focused runs of the touched test binaries and
  `report.rs` unit tests, then `just check`. Known unrelated failures, already tracked:
  `native::gcal::cli::tests::sync_args_read_last_n_days_aliases_and_default_to_none`
  (bob-cli-l2), and the load-only flake
  `native::subprocess_runner::tests::*_bounds_pipe_drain_*` (bob-cli-kl; rerun focused
  to confirm).
- `just docs-gen` (commit anything it regenerates), then `just docs-check`. Known
  unrelated failures: the word budgets for `docs/reference/plugins/bob-ledger-tools.md`
  (bob-cli-il) and `bob-navigation-hotkeys.md` (bob-cli-jz). If lint passes, the only
  expected lychee error is bob-cli-l7. Your edits must add no new docs-check failures.
- No bob-plugins change is expected. If you make one, open bob-plugins with `/sase_repo`
  and follow its `AGENTS.md`: `npm run build`, `npm test`, `npm run validate`,
  `npm run docs:check`, then `bob plugins sync` without `-F`.
- Do not edit Cargo versions, `CHANGELOG.md`, or memory files.

## 6. Closeout (final step, same turn as the code)

1. Run `sase bead epic-symbols bob-cli-kx.3.1` (currently empty). Resolve or re-key any
   entry before closing.
2. Run
   `sase bead close bob-cli-kx.3.1 --note "<what you verified: integrated behavior, new tests, gates, deployment, baseline failures and their tracking IDs>"`.
   Never use `--force` to make it succeed.
3. Run `just symvision` if the recipe exists. bob-cli's justfile currently has none; say
   so in the note.
4. Set `status: done` in the frontmatter of the epic plan file
   `202611/gmail_archive_receipts.md` in the plans repo (the EPIC PLAN path printed by
   `sase bead read bob-cli-kx.3.1`).
5. The epic's `parent_bead` is phase `bob-cli-kx.3`, "Undo command and receipt
   surfaces". Check that its receipts scope is complete:
   - `bob gmail unarchive`
   - the `gmail_archived` and banner lifecycle
   - the desktop notification
   - `recent_archive` JSON
   - the closed-task chip
   - the Move-back button
   - the Archived-by-Bob `mail.base` view, which lives in bob-cli `note/hub.rs` per I3
   - the morning notice

   Run `sase bead epic-symbols bob-cli-kx.3`, then
   `sase bead close bob-cli-kx.3 --note "<what you verified>"`. Do not close
   `bob-cli-kx`, `bob-cli-kx.4`, or any other ancestor. The containing epic's land agent
   owns those.
