---
tier: tale
size: medium
title: Land bob-cli-9c — fix ingest-at-birth defects, tighten tests, refresh docs,
  close the epic
goal: Every defect the bob-cli-9c landing review found in the ingest-at-birth code
  is fixed and tested, stale "after bob ref scan" docs match the new behavior, and
  epic bob-cli-9c is closed with its plan file marked done.
proposed_by: bbugyi200.athena.bob-cli-9c.land
proposed_with: claude/opus
bead: bob-cli-9c
status: done
---

- **PARENT:**
  [202610/ref_ingest_at_birth.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_ingest_at_birth.md)
- **BEAD:**
  [bob-cli-9c](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9c/README.md)

# Land bob-cli-9c: finish ingest at birth

## Context

Epic **bob-cli-9c** ("bob ref create files its own references", plan
`plan:202610/ref_ingest_at_birth.md`) has all six phases closed. Its commits are
`85eac25a`, `c4ac06fb`, `fa7745fc`, `5848c39d`, `47036193`, and `2a11d051`.

The rollout is verified live:

- athena and apollo run `bob-ref-intake.timer`, and their recent runs print
  `intake: nothing queued`.
- The Mac runs the new `bob` (`-k`, `scan -i`, and the doctor `intake_last` row are
  present), has chezmoi applied at `9ee5fd2a`, and has no `pre_scan_hook` or
  `bob_xlib_pull`.
- Vault commit `6f3c4d02` (`bob ref create: ingest chat/…`) is a real end-to-end ingest:
  the PDF, ref note, and `sase.md` landed in one commit.

The landing review compared the code against the plan and found the defects below. The
epic caused all of them, so they are epic work. Commits that landed after the epic
started need no integration work:

- `de78e982` (`added_at`) reads the `created` stamp that intake writes.
- `82413421` (pasted-screenshot image notes) writes through `plan.image_assets`, which
  the intake commit path set already includes.

The only phase follow-up (an ETXTBSY fetch flake) was triaged as a +1 on `bob-cli-5l`.

Read `sase memory read cli_rules.md -r "<why>"` before touching CLI help or synopses.
All file references below are repo-relative. Line numbers are approximate; find the code
by the quoted text.

## Part A — Code fixes

Work in `src/native/highlights_ref/` unless a step says otherwise.

1. **Make `needs_write_pdfs` reachable.**
   - The bug: `intake.rs` (in `IntakeSession::ingest`) classifies a planning refusal
     with `error.message.contains("requires --write-pdf")`. The real refusals in
     `sync.rs` (two sites, both
     `"reference note changed but --write-pdf was not supplied; refusing to update the PDF marker"`)
     never match, so they always queue as `refused`.
   - Fix: give both `CommandError`s `.with_code("needs_write_pdfs")`, and match on the
     error's code in intake rather than on its text.
   - Check that adding a code changes no existing human or JSON scan output. If a JSON
     error code already surfaces those refusals, keep it and match on it instead.
   - Add a test. If one is reachable, use a late-audio `Audio` unit drained by
     `bob ref scan -i` whose existing ref note had a synced frontmatter field (for
     example `status`) edited, so the marker would need a PDF write. Assert the item
     queues with reason `needs_write_pdfs`, the mp3 is back in `xlib/`, and the tree is
     clean.
   - If no CLI route can reach it, add a unit test of the classification helper instead.

2. **Release locks in the documented order.**
   - The bug: `IntakeSession` declares `_maintenance_lock` before `_scan_lock`. Rust
     drops fields in declaration order, so `bob_sync.lock` is released before
     `scan.lock`.
   - Fix: declare `_scan_lock` first, with a one-line comment that field order is the
     release order (scan.lock, then bob_sync.lock).

3. **Run the write guard during planning, before any write.**
   - The bug: `write_single_pdf_sync` (`sync.rs`) runs `ensure_safe_to_write` inside the
     write step. A guard refusal therefore goes through the
     `git_changes_only_moved_files` fallback. For a `LibraryPdf` unit (`-o lib/…`), the
     untracked lib PDF is not among the moves, so the refusal becomes `Incomplete` (exit
     1, "ingest stopped after moving the PDF into lib/") even though nothing was moved
     or written.
   - Fix: in `IntakeSession::ingest`, call
     `ensure_safe_to_write(config, iter::once(&plan))` right after
     `plan_single_pdf_sync` succeeds. A refusal goes through
     `rollback_or_queue(… QueueReason::Refused …)`, like a planning refusal.
   - Then write with `execute_pdf_sync`, or a guard-free sibling of
     `write_single_pdf_sync`, so the guard does not run twice. Keep
     `write_single_pdf_sync` unchanged for its other callers.

4. **Compute the commit path set before writing.**
   - The bug: `scoped_write_path_set` (`guard.rs`) includes a routed intent only while
     `intent_write_planned` sees the candidate ids missing from the file. Intake calls
     it _after_ `write_single_pdf_sync`, so an explicitly routed intent file that was
     just written drops out of the scoped commit.
   - Fix: compute `scoped_write_path_set(config, &plan).write_paths` before the write
     and reuse it for the commit.
   - Add a unit or integration test where a drained or attached PDF has an annotation
     routed to another note. Assert that note is in the ingest commit and that
     `git status --porcelain` is empty afterwards. If no fixture can route an annotation
     cheaply, at minimum add a unit test proving the path set is captured before the
     write.

5. **`finish_intake` never prints.** `ingest_url` promises silence on stdout and stderr,
   and two code paths break that promise:
   - **Ignored paths.** `IntakeSession::ingest` `eprintln!`s
     `warning: skipped ignored intake write path …`. Instead, carry the ignored paths in
     `Ingested` (for example `ignored_paths: Vec<PathBuf>`) and report them from the
     callers:
     - `file_unit`/`finish_intake` sends each warning line through `request.progress`
       when present and drops it otherwise. Create's progress callback already
       `eprintln!`s, so create keeps showing it.
     - The `scan -i` human drain prints it to stderr.
     - The `scan -i` JSON drain adds no new keys; it prints to stderr as it does today.
   - **Status record.** `intake_status::update` `eprintln!`s
     `warning: could not record intake status: …`. Make `record_success` and
     `record_failure` return the warning (for example `Option<String>`) instead of
     printing:
     - `finish_intake` routes it through `request.progress`.
     - The scan callers (`mod.rs` writing full scans, human and JSON;
       `record_drain_status`) keep printing it to stderr.
   - Grep `intake.rs` and `intake_status.rs` for any remaining `eprintln!` or `println!`
     reachable from `finish_intake`, and route those the same way.

6. **Check the kill switch before clipping in `ingest_url`.**
   - The bug: `apply_intake` (`ingest.rs`) calls `intake::ingest_on_create()` only after
     the clip is installed in `xlib/`. An invalid `BOB_HIGHLIGHTS_INGEST_ON_CREATE` then
     returns an `IngestError`, the ref-jobs worker writes its fallback inbox task, and
     an orphan PDF is left in `xlib/`.
   - Fix: resolve `ingest_on_create()` at the top of `ingest_url`, before `lock_ingest`
     and any fetch, and pass the resulting keep reason into `apply_intake`.
   - Add a test: a ref-jobs run (or a direct `ingest_url` unit test) with
     `BOB_HIGHLIGHTS_INGEST_ON_CREATE=maybe` fails before anything is written to
     `xlib/`. Also add a create test: `bob ref create` with an invalid value fails, its
     error names the variable and the accepted values, and nothing is written.

7. **Stop labelling an incomplete ingest as `refused`.**
   - The bug: `map_created_intake` (`ingest.rs`) maps `IntakeOutcome::Incomplete` to
     `CreatedIntake::Queued { reason: "refused", … }` with the lib PDF path. `refused`
     means "rolled back to `xlib/`", and the ref-jobs footer points at
     `bob ref scan -i`, which deliberately skips a lib PDF that has no note.
   - Fix: use reason slug `incomplete`, matching the slug `intake_status` already
     records, and keep the `Queued` shape so `DoneRecord.intake` stays
     `ingested|queued`.
   - In the ref-jobs worker line, `bob ref jobs` output, and the gkeep human line, an
     `incomplete` row says that it stopped after moving into `lib/` and that a full
     `bob ref scan` finishes it, not `bob ref scan -i`.
   - The `ref_jobs/output.rs` footer names `bob ref scan -i` only when an ordinary
     queued row exists, and names `bob ref scan` for incomplete rows.
   - Update `docs/ref-jobs.md` and `docs/gkeep.md` where they list queue reasons.

8. **Fix the commit-failure wording.**
   - The bug: both create printers in `intake.rs` print
     `sync: committed with warning (…); background vault-sync will commit it` when
     `ob::commit_paths` failed, but nothing was committed.
   - Fix: print the plan's shape:
     `sync: commit failed (<warning>); background vault-sync will commit it`. Then
     append the post-sync status the same way the success line does.
   - Update any test or doc that quotes the old text.

9. **Make the `scan -i` JSON dry run exit 0 like human mode.**
   - The bug: in `scan_json.rs` (intake-only JSON path),
     `let ok = failures.is_empty() && queued.is_empty();` also applies to `--dry-run`. A
     would-queue item (for example a local collision) therefore makes
     `scan -i -d -f json` exit 1, while the human dry run and the docs say exit 0.
   - Fix: for dry runs, `ok` ignores `queued`.
   - Add a test: a dry run with one item that would queue gives JSON `ok: true`, exit 0,
     and lists the item in `queued`.

10. **Keep the doctor `intake_last` row on one line.**
    - The bug: `intake_last_summary` (`doctor.rs`) prints `fail.message` verbatim, and a
      multi-line git error (a `pre_sync_failed` detail) spills across lines.
    - Fix: render only the first non-empty line, trimmed. Optionally cap it at about 120
      characters with `…`.
    - Add a doctor test with a multi-line recorded message.

## Part B — Test tightening (`tests/cli/…`)

1. `tests/cli/highlights/ingest.rs`, default-create test:
   - The follow-up dry-run scan assertion is vacuous. It checks for
     `note_action: would-create`, which scan never prints. Assert instead that the
     dry-run scan output names no note create for `ref/chat/report.md`: neither
     `would create` (the wording from `report.rs`) nor `note_action: create`. Confirm
     against real output.
   - Also assert that the peer's `sase.md` contains the `[[ref/chat/report|Report]]`
     link after its pull.
   - Also assert that origin's `HEAD` equals the vault's `HEAD` (for example
     `git ls-remote origin HEAD` or `git rev-parse origin/master` after a fetch).
2. `ingest.rs`, lock-busy queue case: assert that `HEAD` did not move (no commit).
3. Kill switch tests:
   - A config test that the `ingest_on_create` key is absent → `None` (default on).
   - The invalid-env test from A6.
4. `tests/cli/help_options.rs`: add `-k, --keep-intake` to the `bob ref create`
   option-order list, between `-i, --include-id` and `-L, --listen`. Confirm against the
   real help order.
5. `tests/cli/highlights/listen.rs` `listen_attach_files_audio_for_a_library_capture`:
   - Replace the `audio:` OR mp3-path check with explicit checks that the note has an
     `audio:` frontmatter key and the player embed.
   - Assert that the attach commit's `diff-tree` names both `lib/papers/ea_graph.mp3`
     and `ref/papers/ea_graph.md`.
6. `tests/cli/highlights/jobs.rs` `ref_jobs_run_ingests_into_a_synced_vault`:
   - Assert that exactly one new commit landed, with subject
     `bob ref jobs run: ingest <type>/<stem>`.
   - Assert that it contains the lib PDF, the note, and the parent.
7. `tests/gkeep_pull.rs` has no filed-path test.
   - Add one in a git vault with a bare origin (port a minimal `init_vault_sync_pair`
     into `tests/gkeep_support`, or build the origin with `git init --bare` and
     `clone`), using the existing fake-curl PDF clip.
   - Assert the human `filed →` line and the JSON `clip.intake == "ingested"` with a
     `clip.note` path.
   - Assert the journal `pdf` is the `lib/` path.

## Part C — Docs and comments

1. `README.md`:
   - **Synopses.** In the `bob ref create` synopsis (Reference library section), add
     `[-k|--keep-intake]` after `[-i|--include-id]`. In the `bob ref scan` synopsis, add
     `[-i|--intake-only]` after `[-f|--format human|json]`. Keep each synopsis in the
     same order as `--help`.
   - **Capture workflow** step ("`bob ref list` shows the note after `bob ref scan`"):
     say the background job files the note when it runs, and that a queued clip appears
     after the host's intake drain (`bob ref scan -i`).
   - **Vault layout table:**
     - `xlib/` is the host-local intake queue: PDFs kept with `-k` or queued when inline
       intake could not run.
     - `lib/` holds library PDFs, filed by `bob ref create` and its producers, or by
       `bob ref scan` for queued items.
   - **Capture section** ("the note appears in `bob ref list` after `bob ref scan`"):
     same correction as the capture workflow step.
2. `docs/ref.md` "Choose a command" capture row: the same correction.
3. `docs/capture.md`: the intro ("note after `bob ref scan`") and the URL capture
   passage ("appears after `bob ref scan`") get the same correction.
4. `docs/highlights-create.md` "Several URLs": the collection gets its reading task when
   `create` files it, not "after `bob ref scan`".
5. `docs/highlights-ref-sync.md`:
   - The JSON `ok` field row: with `--intake-only`, `ok` is also false when a writing
     drain leaves anything in `queued`.
   - The pre-scan hook paragraph claims `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1` means a hook
     "never deadlocks on the scan lock", but bob does not read that variable. Replace
     the claim:
     - Bob exports the variable so a hook can detect it and avoid shelling back into
       `bob ref scan`.
     - A nested writing scan would wait on the held locks and fail with `lock_busy`.
     - Mirror this in the matching `README.md` sentence if it makes the same claim.
6. `src/native/highlights_ref/clip.rs` module doc: replace "`scan` writes the ref note"
   with create-time intake (the note is written at create unless kept or queued).
7. Final grep of `README.md`, `docs/`, and `src/`:
   - `after \`bob ref scan\``, `bob_xlib_pull`, `pre_scan_hook: PATH`, and `Mac runs`.
   - Fix any remaining passage that presents a later scan as the normal path. Leave two
     kinds of passage alone: ones that describe genuinely queued or kept items, and the
     capture `in_intake` verdict wording ("waiting for bob ref scan"), which Bob Mac
     Capture mirrors verbatim.

## Part D — Verify

- Run the focused test binaries first: highlights ingest, scan_intake, listen, jobs,
  doctor_library, help_options, and gkeep_pull. Then run `just check` (not
  `just check-full`).
- The ETXTBSY flake in `native::highlights_ref::fetch::tests` is a known flake tracked
  by `bob-cli-5l`. If it fails, rerun it once; do not fix it here.

## Part E — Close out epic bob-cli-9c (final step)

1. Run `sase bead epic-symbols bob-cli-9c`. At planning time it printed
   `No --epic-symbol entries for bob-cli-9c.` If any entries are listed now, resolve
   each one (wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy) before closing. No other bead is open under this
   epic, so re-keying is not an option.
2. Close the epic, with a note that summarizes the verification:

   ```bash
   sase bead close bob-cli-9c --note "<verification>"
   ```

   The note must cover:
   - all six phases were reviewed against the plan and their commits;
   - live rollout on athena, apollo, and the Mac (timers active, new bob, chezmoi
     `9ee5fd2a`, bridge retired), plus end-to-end vault commit `6f3c4d02`;
   - post-epic commits `de78e982` and `82413421` need no integration;
   - the follow-up triage (bob-cli-9c.4's ETXTBSY flake → +1 on bob-cli-5l);
   - each Part A–C fix listed briefly;
   - the `just check` result.

   Never use `--force` to make the close succeed. If the close is rejected, fix the
   cause (for example leftover `--epic-symbol` entries) and close again.

3. Run `just symvision` if the recipe exists, and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/ref_ingest_at_birth.md` (the PLAN path printed by
   `sase bead read bob-cli-9c -r "Need the plan path for closeout"`). It currently says
   `status: wip`.
5. The epic has no `parent_bead`, so no ancestor needs handling.
