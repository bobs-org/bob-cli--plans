---
tier: epic
title: Gmail archive-on-close landing repairs
goal: 'Finish epic bob-cli-kx.2.1 by repairing the land-audit defects in the Gmail
  archive core: frozen saved IDs, journaled-before-mutation ownership, receipt label
  identity, boundary revalidation, filing-safe error reporting, preview parity, and
  accurate reports, help, and docs.

  '
parent_bead: bob-cli-kx.2.1
phases:
- id: ownership
  title: Frozen IDs, receipt identity, and journal ownership
  size: medium
  depends_on: []
  description: 'ownership: Make planning, apply, and recovery use the close''s frozen
    saved IDs; record and match the real receipt label ID; re-baseline on an archive_label
    change; accept [?] in open observations; stop reopen/re-close false recoveries;
    surface cross-host holds; add real label-diff property tests. Implement section
    1.'
- id: boundary
  title: Mutation boundary, let-go, and failure isolation
  size: medium
  depends_on:
  - ownership
  description: 'boundary: Plan every terminal journal event before its Gmail write,
    revalidate local evidence per thread under the maintenance lock, honor the planned
    decision set, fix recovery ordering, scope let-go to recorded IDs and run it independently
    of the gate, and add the missing end-to-end fault tests. Implement section 2.'
- id: integration
  title: Sync wiring, hydration capture, and preview parity
  size: medium
  depends_on:
  - boundary
  description: 'integration: Capture saved IDs before watch hydration, block archive
    writes after failed or incomplete filing, keep filing/watch report rows when the
    archive phase errors, simulate enablement in dry-run so a first preview confirms,
    and reject a supplied mismatched -c. Implement section 3.'
- id: surfaces
  title: Reports, help, docs, and final verification
  size: medium
  depends_on:
  - integration
  description: 'surfaces: Fix quiet digest leaks, hollow kept/let-go rows, and paused
    row detail; rewrite remaining read-only claims in help and docs; document the
    journal schema, vocabulary, and exit codes; refresh the C1-C20 table; run the
    docs and full gates. Implement section 4.'
proposed_by: bbugyi200.athena.bob-cli-kx.2.1.land
proposed_with: claude/opus
create_time: 2026-11-12 23:51:11
status: done
bead_id: bob-cli-kx.2.1.8
---

- **PROMPT:** [prompts/202611/gmail_archive_landing_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_archive_landing_repairs.md)
- **PARENT:** [202611/gmail_archive_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_core.md)
- **BEAD:** [bob-cli-kx.2.1.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kx/bob-cli-kx.2.1.8.md)

# Gmail archive-on-close landing repairs

Epic `bob-cli-kx.2.1` ("Gmail archive-on-close core", plan
`plan:202611/gmail_archive_core.md`) shipped seven phases: commits `677e7407`,
`92b52e4f`, `4c2cb878`, `8e78584d`, `0b806ae8`, `62ad26d7`, and `54fc8c61`. Its land
audit, at master `1863b39b`, read the source against that plan and confirmed the defects
below. All of them were caused by the epic, so they remain epic work. This child epic
fixes them. Its land agent then resumes the parent landing through `parent_bead`. No
phase closes `bob-cli-kx.2.1`, runs its closeout, or edits its plan file.

Read `plan:202611/gmail_archive_core.md` first. Its "Design shared by all phases"
section is the binding contract, and so are the reviewer decisions on `bob-cli-kx.2`:
Cancelled `[-]` archives too, the receipt label defaults to `<sync.label>/closed`,
`archive_max` defaults to 10, and `bob-cli-i4` is superseded. Do not re-ask any of them.

Those constraints still hold:

- No memory edits are authorized, including decision records and glossary strands.
- Use the fake gog and temporary synthetic vaults only. Never touch `~/bob` or the real
  account.
- No message bodies, tokens, or task text may appear in argv, journals, or warnings.
- Keep `schema_version: 1` and exit codes 0/1/2. JSON changes are additive only.
- Commit Gmail contract changes as `feat(gmail)!:` or `fix(gmail)!:`.
- Never change the Cargo.toml package version or CHANGELOG.md. Adding a dev-dependency
  is allowed.

Before changing CLI options, help, or docs, read the reference memories `cli_rules.md`
and `docs.md` with `/sase_memory_read`.

The main code lives in `src/native/gmail/sync/`:

- `archive_scan.rs` gathers evidence.
- `archive_state.rs` folds the journal.
- `archive_transport.rs` holds the pure label diff.
- `archive_preview.rs` holds `local_verdict`, `remote_verdict`, the digest, and
  `apply_gate`.
- `archive_apply.rs` holds `apply_live`, `ensure_baseline`, `persist_closes`, the
  per-thread loop, `append_*`, `reconcile_*`, `release_owned`, and `live_receipt_label`.
- `command.rs` holds the sync orchestration and `attach_archive_dry_run`.
- `report.rs` holds the human and JSON report.

Elsewhere:

- `src/native/gmail/gog.rs` holds the batch-modify and label-create adapters, and
  `src/native/gmail/doctor.rs` holds doctor.
- The tests are `tests/gmail_archive_apply.rs`, `tests/gmail_archive_preview.rs`,
  `tests/gmail_transport.rs`, `tests/gmail_sync.rs`, and `tests/gmail_shell.rs`.
- The fake transport is `tests/gmail_support/fake_gog.py` and
  `tests/gmail_support/mod.rs`. It already offers `batch_race_before_modify` and other
  fault injection.

Known unrelated failures at `1863b39b` may still be present. Record them; do not fix
them:

| Failure                                                                                                | Owner            |
| ------------------------------------------------------------------------------------------------------ | ---------------- |
| `catalog::protocol_1_parses_with_schema_version_one` expects 97 entries but gets 98                    | `bob-cli-l5.3.1` |
| `native::gcal::cli::tests::sync_args_read_last_n_days_aliases_and_default_to_none`                     | `bob-cli-l2`     |
| `subprocess_runner` pipe-drain load flake                                                              | `bob-cli-kl`     |
| `just docs-gen` / `just docs-check` stop on `sase/memory/glossary/event-ack.md` having no front matter | `bob-cli-l5`     |

A cross-cutting rule applies to every phase: **no Gmail mutation may succeed without a
journal entry describing it.** Every gap below that ends in "the remote write happened,
`event_for_terminal` returned `None`, apply returned `Ok(())`" is a violation.

## 1. ownership — frozen IDs, receipt identity, journal ownership

1. **Frozen saved IDs.**
   - The problem: `remote_verdict` (`archive_preview.rs`, the `frozen_sets` and
     `has_receipt` calls), `row_for`, the apply loop, and `reconcile_recovered` all
     compute A, U, and recovery from the _current_ scan's `evidence.saved_ids`.
     `ArchiveCloseGeneration.saved_ids` is only used in the terminal subset check.
   - Failure scenario: close G freezes `{a1}` and its batch fails transiently. Before
     the retry, `bob gmail save` adds `b2`. The retry archives `b2`, the journal append
     is rejected, the run exits 0, and the receipt ends up permanently unowned.
   - The fix: for a row with a journaled close, use `close.saved_ids` for A, U,
     `has_receipt`, recovery, and the digest. For a backlog row, use the scan's IDs. The
     digest binds them, and `event_for_confirmed_backlog` freezes them.
   - When the scan has newly saved IDs that are in INBOX, they fall in U, so the row is
     kept as `unsaved`. It never archives them.
   - Thread `close.saved_ids` (or the frozen set) through `RemoteVerdict` callers
     instead of `evidence`. The apply loop's "evidence changed" check must compare
     against the frozen close, not the row built from the same scan.
2. **Receipt label ID.**
   - The problem: `apply_live` builds `receipt_identity` from the label list _before_
     `resolve_or_create_receipt_label`. A first-use archive therefore journals
     `label.id = None`.
   - Later, `release_owned` and `reconcile_recovered` compare message `label_ids`
     against `id` _or the label name_. Messages carry only IDs, so the let-go
     crash-convergence branch records `gmail.let_go` without removing the receipt.
   - The fix: journal the resolved or created `ReceiptLabel` ID for archived and
     recovered events.
   - Match message labels by ID only. Resolve names to IDs through the live label list,
     using the existing strict resolution.
   - If a recorded receipt has no ID, resolve it by exact unique name from the live
     label list before deciding. If that fails, hold with a warning; never write a
     let-go.
   - `live_receipt_label` must validate against the real enrollment label, not the
     `Label_0` placeholder.
3. **`archive_label` change after enablement.**
   - The problem: `ensure_baseline` appends enablement only when the account is not yet
     enabled, and `compatible_label` compares names. After a config rename, every
     archive therefore journals nothing.
   - The fix: when the account is enabled and its epoch label name differs from the
     configured receipt name, append `event_for_enablement(account, true, new_label)`.
     This is the re-baseline path that `archive_state.rs`
     `disable_reenable_and_label_rename_rebaseline_without_rewriting_receipts` already
     covers.
   - After the re-baseline, closes from the old epoch project as backlog and need `-c`.
     Owned receipts keep their old identity for let-go.
   - Document this in the surfaces phase.
4. **`[?]` linkers.**
   - The problem: `[?]` is open (`scan.rs` `OPEN_STATUSES`), and `event_for_open` writes
     status `"?"`. But `parse_open_observation` accepts only `' ' '*' '/' 'x' 'X' '-'`,
     so the thread is marked malformed and held forever.
   - The fix: accept every status that the scan classifies as open, consistently in
     every task-locator parser, and add a fold round-trip test.
5. **Reopen, then re-close, of an archived thread.**
   - The problem: the messages still carry the receipt and are not in INBOX, so A is
     empty, `HOLD_UNRECONCILED` applies, and the code appends a false
     `archived{recovered:true}` for the new generation. That gives two owners for one
     set of IDs.
   - The fix: receipt-bearing messages already covered by an earlier generation's
     unreleased owned receipt are not recovery evidence for a newer close. Consume the
     newer close with a silent `kept` / `manual` outcome.
   - The earlier receipt stays owned, so a later Move to Inbox still lets go.
6. **Cross-host and invalid terminal events.**
   - The problems in `archive_state.rs`:
     - Dropping another host's close because the open observations diverge is silent.
     - `mark_invalid_scope` from an unknown-generation `gmail.archived` holds the thread
       forever with no warning.
     - A well-formed `archived` event that fails its subset or label check is dropped,
       so its generation looks unconsumed.
   - The fix: emit bounded, body-free journal warnings for each case. Treat a generation
     with a rejected terminal event as held, never as unconsumed.
7. **Real property tests for the label diff.**
   - The problem: `archive_transport.rs` only enumerates subsets of 8 fixed IDs, with
     one label set shared by every message.
   - The fix: add property tests that cover:
     - arbitrary label sets per message;
     - arbitrary literal label names and IDs, including commas and backslashes;
     - arbitrary valid ID lists;
     - idempotence;
     - that restore reverses archive;
     - that let-go never changes INBOX;
     - that only INBOX and the receipt ever change.
   - Use a dev-only `proptest` dependency if it resolves in the normal build. Otherwise
     write a seeded deterministic generator inside the test. No raw mutation argument
     may escape the type boundary.

Validation: unit and fold tests for each item, plus end-to-end fake-gog tests for:

- retry after a save between runs (frozen `{a1}` archives only `a1`, and `b2` keeps the
  row `unsaved`);
- first-use label creation followed by a reply that lets go and removes the receipt by
  ID;
- an `archive_label` rename;
- a `[?]` linker that later closes and archives;
- reopen, then re-close, with no second owner.

Then run `cargo fmt`, `just lint`, and the Gmail suites.

## 2. boundary — mutation boundary, let-go, failure isolation

1. **Journal-before-mutate.**
   - The fix: before every Gmail write, build the planned terminal event with
     `event_for_terminal`, using the same IDs and label the write will use. This covers
     archive, recovery, and let-go. If it returns `None`, hold the thread with a
     body-free warning, mark the run failed, and do not write.
   - After a successful write, append that pre-planned event.
   - Remove every silent `return Ok(())` / `continue` that follows a successful write.
     That covers `append_archived`, `append_kept`, `reconcile_recovered`, and both
     `release_owned` branches.
2. **Local revalidation at the write boundary.**
   - The problem: today the loop compares the row with the same scan it was built from,
     so a task reopened after planning still gets archived.
   - The fix: for each thread, acquire the vault maintenance lock with a bounded wait,
     following the discipline `file::apply` and the sync guard already use. Re-read that
     thread's note and linking task files through the archive scan rules, then compare
     against the frozen close. Compare the note fingerprint, task locators and statuses,
     all-closed, and the saved IDs.
   - Then issue that thread's single batch write and release the lock.
   - Never hold the lock across more than one thread's write. Never expand the frozen
     IDs. On any change, hold and warn.
3. **Honor the planned decision set.**
   - Mutate only rows whose planned decision was Archive. A row planned Kept or Held
     whose fresh verdict became Archive is held for the next run; this must never widen
     a digest-confirmed batch or exceed `archive_max`. A fresh downgrade (for example, a
     new unsaved reply) is still honored.
   - Resolve or create the receipt label only when at least one row is planned Archive.
4. **Recovery ordering.** A crash can leave frozen messages with the receipt and no
   archived event. That recovery evidence must be evaluated before the consent and
   unsaved checks. Then a reply or a removed enrollment label after the crash still
   records the recovered ownership once. It covers the current receipt-bearing frozen
   IDs and never touches unknown messages.
5. **Let-go scope and independence.**
   - Let-go removes the receipt only from recorded IDs that currently carry it. It
     verifies thread and message identity the way `remote_verdict` does.
   - `release_owned` must run on every enabled live run that has the modify grant. That
     includes runs where the gate is paused and runs where the receipt label cannot be
     resolved. `invalid_grant` still aborts.
6. **Auth and hold accounting.**
   - An `invalid_grant` raised during the never-save policy read must abort, exactly
     like the other call sites. Today `policy.rs` converts it to `awaiting_sign_in`, and
     it is held as `fetch_failed`.
   - `HOLD_UNRESOLVABLE` in the live loop marks the run failed.

Validation: end-to-end fake-gog tests, each asserting the exact write argv and the
journal:

- `invalid_grant` on the second thread stops every later write, including let-go and
  label creation, and the first thread's filing and archive are kept;
- two threads where one batch fails;
- a task reopened between planning and mutation (inject through a fake hook or an apply
  test seam);
- a reply between the last read and the batch, using `batch_race_before_modify`, where
  only frozen IDs change and the reply stays in INBOX;
- a crash between let-go's remote write and its journal append;
- recovery after a crash followed by a reply;
- let-go while a backlog gate is paused;
- a planned-Kept row that turns archivable.

Then run fmt, lint, and the Gmail suites.

## 3. integration — sync wiring, hydration capture, preview parity

1. **Saved IDs from before hydration.**
   - Snapshot each thread's validated saved IDs before `file::apply` and the run's watch
     hydration. When this run observes a close, freeze the intersection with that
     snapshot.
   - Messages hydrated in this run count as unsaved for this run. Dry-run mirrors this.
   - Keep the existing C18 suppression of lifted and bounced threads.
2. **Gate archive writes on filing health.**
   - Today `command.rs` gates live archive only on `!outcome.busy`. It must also skip
     every archive write when filing failed or recovery evidence is incomplete. That
     covers baseline, closes, label creation, archive, and let-go.
   - In that case, emit a warning and mark the run unsuccessful.
3. **Keep filing and watch rows when the archive phase errors.**
   - The problem: `apply_live(...)?` currently discards `results[]` and `watch[]` on a
     missing modify grant, `invalid_grant`, or a journal failure.
   - The fix: carry the archive error into the report as a typed error or warning, and
     still print the filing and watch rows.
   - Exit codes: 2 for setup (`modify_grant`, with its login hint) and for auth, and 1
     for runtime.
   - Keep the existing `health` success/failure semantics consistent with the filing
     outcome.
   - Tests assert that JSON `results[]` contains the filed thread. Existence of the
     inbox file is not enough.
4. **Dry-run before the first enablement.**
   - The problem: `attach_archive_dry_run` projects `Disabled`, so the preview shows
     nothing.
   - The fix: when `archive: on` but the account has no enablement record, simulate the
     enablement and open observations in memory, without persisting anything. The
     preview then shows the backlog rows and a digest.
   - The following live run writes the baseline. Given that digest with `-c`, it must
     confirm and archive. Prove this end to end from a fresh journal; existing tests
     seed enablement by hand.
5. **Supplied `-c` mismatch.**
   - The problem: `apply_gate` returns `Open` before it checks a supplied digest when
     confirmation is not required.
   - The fix: a supplied digest that does not match is always `confirm_mismatch`, with
     zero writes, in both dry-run and live. An empty plan with a supplied digest is also
     a mismatch.
6. **Guard-busy JSON.** It gains the additive `archive_enabled`, `archive_label`, and
   `archive_max` fields.

Validation: end-to-end tests for each item, the vault hash before and after dry-run, and
fmt, lint, and the Gmail suites.

## 4. surfaces — reports, help, docs, final verification

1. **Report.**
   - Keep `archive_digest` (human and JSON) only on dry-run or paused runs. Today a
     quiet live run whose rows are silent prints `archive digest <hex>` under `-q`
     (`report.rs`, around the digest line).
   - Fill live `kept` and `let_go` rows with `note`, sorted `messages`, and `tasks[]`,
     taken from the planned row or the recorded close. Human rows must not end in a
     dangling arrow.
   - Paused rows keep `reason` as the gate reason. Add an additive `decision` field (for
     example `would_archive` or `kept:<reason>`) so the confirmation shows what each row
     would do.
   - Show the decision in the human pause listing.
   - Render note paths in archive rows and hints through the same inert single-line
     helper that watch rows use.
   - Serialize the `-n` skip additively.
   - Remove the stale "dry-run only until apply lands" comments.
   - Doctor must not hide the missing-modify hint behind an early return on a journal
     warning.
2. **Help.**
   - Replace "Sync is read-only in Gmail" in the `sync` `long_about` in
     `src/native/gmail/cli.rs`. Suggested wording: "By default sync leaves Gmail
     unchanged. With gmail.sync.archive: on and the modify grant, it archives saved
     messages once every linking task is closed (adds the receipt label, removes INBOX);
     backlog or batches above archive_max need -c DIGEST from a -d preview; -n skips
     archiving for one run."
   - Change "one read-only login" in the root help to "a read-only login by default".
   - Add a live `bob gmail sync -c DIGEST` example.
   - Restore wording becomes "restore stays manual: Move to Inbox in Gmail" in `cli.rs`,
     `login.rs`, and the apply hint. Do not imply that an undo CLI exists.
   - Add `-c, --confirm` and `-n, --no-archive` to the option-order test in
     `tests/gmail_sync.rs`.
3. **Docs.** Make every Gmail doc accurate to the code:
   - `docs/gmail.md`:
     - The "without changing Gmail labels" line.
     - Policy and gone cases are a one-time `kept` that consumes the close; they are not
       held.
     - Merge holds are silent.
     - The baseline needs a live run on the writer that has the modify grant.
     - A keep consumes the close, and a reopen re-arms it.
     - `summary.archived` and the exit codes: 2 for `modify_grant` and auth; 1 for
       paused, `confirm_mismatch`, and archive failure.
     - The doctor `config` row wording.
     - The archive write fences: `--enable-commands-exact gmail.batch.modify` and
       `gmail.labels.create`, with no `--readonly`.
     - That let-go removes the receipt.
     - That an `archive_label` rename re-baselines.
   - `docs/guides/file-email-from-your-phone.md`: "Removing the Gmail label later
     changes nothing" now opts the thread out of archiving.
   - `docs/guides/connect-google-mail.md`: document `-s modify` and the `modify` scope
     rows.
   - `docs/reference/json-outputs.md`:
     - Restore the `results` / `watch` / `problems` field lists, the status vocabulary,
       "first-seen fail `ok`", and the "`-d` allowed on a non-writer host" text that
       `54fc8c61` deleted.
     - Name the exact `archive[]`, `archive_hints[]`, `archive_digest`, and `decision`
       keys.
     - Mark `restored` as reserved and never emitted yet.
   - Document the actual journal event schema (`archive_state.rs` constants) with its
     key fields, in `docs/gmail.md` and `docs/reference/vault-format.md`:
     - The events are `gmail.archive_enabled`, `gmail.archive_observed`,
       `gmail.archive_closed`, `gmail.archived` (`recovered`), `gmail.kept` (`reason`),
       `gmail.let_go`, and `gmail.restored` (reserved).
     - Document the reason, action, hint, and error vocabulary.
   - Present no deferred surface as shipped: `unarchive`, `gmail_archived`,
     `recent_archive`, chips, and notifications.
4. **Final verification.**
   - Update the C1-C20 table at the top of `tests/gmail_archive_apply.rs` so it names
     the new tests.
   - Run `just docs-gen` and `just docs-check`. If they are still blocked by the
     `event-ack.md` front-matter failure (owned by `bob-cli-l5`), run
     `BOB_UPDATE_GOLDENS=1 cargo test --test cli docs_reference` and record the blocker
     with its evidence.
   - Run every Gmail integration suite and `just check`. Never run `just check-full`.

## Closure

Each phase records its verification on its own bead. It proposes discovered work with
`sase bead note <phase-id> 'PROPOSED FOLLOW-UP: ...'` and creates no task beads. Each
phase finishes through `/sase_final`.

No intermediate symbol may need an `--epic-symbol` exemption keyed to a phase that is
about to close. Before closing, run `sase bead epic-symbols <phase-id>` and clean up
anything it lists.

This child epic's land agent verifies all four phases. Then, after closing this child
epic, it resumes the interrupted landing of parent `bob-cli-kx.2.1` as its prompt
directs.
