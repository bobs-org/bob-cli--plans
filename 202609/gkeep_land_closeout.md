---
tier: tale
size: medium
title: "bob gkeep: fix land-review defects and close epic bob-cli-2d"
goal:
  "Every epic-caused defect the bob-cli-2d land review found is fixed and tested: the
  adapter timeout, attachments, archive confirm, pull reporting, time zones, escaping,
  flakes, gkeep clippy warnings, and docs drift. Epic bob-cli-2d is then closed, with
  its plan file marked done."
proposed_by: bbugyi200.apollo.bob-cli-2d.land
bead: bob-cli-2d
status: done
---

- **PARENT:**
  [202609/bob_gkeep_inbox_drain.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/bob_gkeep_inbox_drain.md)
- **BEAD:**
  [bob-cli-2d](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2d/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2d.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2d.land.md)
- **COMMITS:**
  - [d0c1692](https://github.com/bobs-org/bob-cli/commit/d0c1692c111f2691d35c53a402a38af62591b918)
    — fix(gkeep): land closeout defects for bob-cli-2d

# Plan: fix the `bob gkeep` land-review defects, then close epic bob-cli-2d

## Context

Epic **bob-cli-2d** ("bob gkeep: drain the Google Keep inbox into Obsidian tasks") has
all seven phases closed (bob-cli-2d.1 – .7). Its commits are 55fdb18 … cad7c8e on
`master`. The epic's plan is `plan:202609/bob_gkeep_inbox_drain.md`; run
`sase bead read bob-cli-2d -r "<why>"` to get its path (the `PLAN` line). That plan is
the source of truth for every contract mentioned below, so read its sections on the
shared contract, `pull`, `list`, and rendering before you start.

The land agent checked the work and found:

- `cargo test` is green.
- The chezmoi `gkeep:` seed is committed and pushed.
- No other commits landed during the epic.
- `sase bead epic-symbols bob-cli-2d` has no entries.

It also found the epic-caused defects below. They are the remaining epic work. The
follow-up triage is already recorded on bob-cli-2d, so do not re-triage it.

Hard rules. They still apply from the epic plan:

- **Never contact live Google Keep.** Do not run `bob gkeep` against the real account.
  All tests use the fake adapter (`BOB_GKEEP_ADAPTER`).
- **No new crates.** There is no `libc`/`nix`. Use `std` and `std::process::Command`.
- **Follow the CLI rules.** Read them with `sase memory read cli_rules.md -r "<why>"` if
  you touch help text.
- **Verify with these commands:** `cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, and `cargo test`.
  - The repo has no `just check` recipe.
  - `just lint` and `just all` currently fail on one pre-existing deny error at
    `tests/cli.rs:31818` (`|| true`). Epic bob-cli-28 owns it. Do not fix it here.
  - That error stops clippy from compiling the `cli` test target. Judge clippy
    cleanliness by `src/native/gkeep/**` and `tests/gkeep*`.

## 1. Adapter client (`src/native/gkeep/adapter.rs`)

1. **The timeout must kill the whole adapter tree.**
   - `uv run --script` starts Python as a child process. `child.kill()` kills only `uv`,
     and the orphaned Python keeps the stdout/stderr pipes open, so the reader-thread
     `join()`s block forever.
   - Fix: in `spawn_adapter`, put the child in its own process group
     (`std::os::unix::process::CommandExt::process_group(0)`). On timeout, kill the
     group with `Command::new("kill").args(["-KILL", "--", &format!("-{pid}")])`, fall
     back to `child.kill()`, then `wait()`, then join the readers.
   - Add a unit test: a fake adapter that backgrounds a `sleep 30` grandchild holding
     stdout (e.g. `sleep 30 & wait`) must time out within a few seconds.
2. **Write the request without blocking past the deadline.**
   - Today `stdin.write_all` runs before the drain threads start and before the deadline
     is set. A request larger than the pipe buffer, sent to a child that never reads,
     blocks forever.
   - Fix: write stdin on its own thread, spawned together with the drain threads, and
     start the deadline before any blocking I/O.
3. **Fix the stdin EPIPE race.**
   - This causes the flaky `adapter::tests::crash_garbage_and_timeout`, whose "invalid
     JSON" assertion fails under load.
   - When the child exits 0, ignore a stdin write error (the child exited without
     reading) and evaluate stdout normally: garbage → "invalid JSON", a valid response →
     success.
   - Only report "write the adapter request" when the exit is non-zero **and** stdout is
     empty.
4. **Resolve order.** In `resolve`, check for `uv` on `PATH` _before_ materializing the
   script. Then `resolve_without_uv_is_a_setup_error` no longer writes into the real
   user cache.

## 2. Adapter script (`scripts/gkeep_adapter.py`) and model

1. **Unknown attachment kinds must not break a snapshot.**
   - `_attachment_kind` returns `"nonetype"` (etc.) when `blob.blob` is `None` or of an
     unknown class, and Rust's `AttachmentKind` rejects it, so the whole snapshot fails
     to parse.
   - Fix: the adapter emits only `image`, `drawing`, `audio`, or `other`.
   - `AttachmentKind` (`model.rs`) gains `Other` with `#[serde(other)]`.
   - `render.rs` renders `other` as `file(s)` in the `📎 N … stay(s) in Google Keep`
     line. The `list` 📎 hint counts it.
   - Add a render golden test and a model deserialization test for an unknown kind.
2. **Archive guard step 4 is missing.**
   - Today the confirm step reads the in-memory note right after the single sync that
     set `archived = True`.
   - Fix: after that sync, sync again and re-`get(id)` each note. Report `archived` only
     when the note reads `archived is True`, else `error` with detail.
3. **Internal errors must reach stderr.** Unexpected (`internal`) exceptions inside ops
   must print a traceback to **stderr** (`traceback.print_exc(file=sys.stderr)`) and
   still return the `ok:false` response. Keep scrubbing the token.
4. **Protocol hardening.**
   - A non-string `op` (e.g. a list) must return a `protocol` error response, not crash.
   - `ping` must report `"unknown"` for a package whose version lookup fails, not the
     pinned version.
5. Extend `--self-test` for items 1, 3 (response shape only), and 4.
   `just check-adapter` must print `ok`. It needs network the first time; report it if
   that is unavailable.

## 3. Pull (`src/native/gkeep/pull.rs`, `tests/gkeep_pull.rs`)

1. **`summary.failed` is wrong.**
   - Today it is `notes.len() - written - skipped + refused`, which counts every
     `pending` (archive-only) note as failed.
   - Fix: count notes that failed to write or verify, plus archive statuses
     `changed | missing | error`.
   - Add a test: an archive-only pull reports `"failed": 0` and `"ok": true`.
2. **An adapter failure during archive must print one report.** This covers a crash, a
   timeout, or an `ok:false` response.
   - **JSON mode:** stdout gets exactly **one** document. Today it prints the report and
     then a second `{"ok":false,…}` from `ui::report_error`.
     - Emit the report with `"ok": false`.
     - Set every note that was due for archive to `archive: "error"` with the adapter
       message as `detail`.
     - Add a top-level `"error": {kind, message, hint}` object.
   - **Human mode:**
     - Rows due for archive say `NOT archived: <reason>` in red. Today they say
       `left in Keep (--no-archive)`, or `would archive` for archive-only rows.
     - Then print the error line and hint on stderr.
   - Exit 1.
   - Test both modes by making the fake `archive` op exit non-zero.
3. **Failure output.**
   - The human summary line must use the warning/error prefix, not `ok`, whenever the
     run exits 1.
   - With `--quiet`, a failed run prints its failures to **stderr** only; stdout stays
     empty.
4. **Dry run.**
   - JSON reports `"written": false` per note and `summary.written: 0`.
   - With `-n`, rows say `would write · left in Keep (--no-archive)` and do not count as
     archived.
5. **Match the epic plan's pipeline order.**
   - Take `bob_sync.lock` (step 4) **before** reading the target and scanning the ledger
     and journal (step 5).
   - Do the compare-and-swap re-read immediately **before the rename**, after the temp
     file is written and `sync_all`ed. On a mismatch, delete the temp file, re-plan
     once, and abort on a second mismatch as today.
   - Set the temp file's permissions **before** `sync_all`.
6. **Git failures.**
   - If the vault is a Git worktree but `git` cannot be started, treat it as a commit
     failure: archive nothing and exit 1. Today it warns and archives.
   - Pull-lock errors other than contention print their real I/O error, not "already
     running".
7. **Test gaps in the failure matrix.** Add these assertions:
   - The normal pull inserts byte-exact golden Markdown, and exactly one new commit
     touches only the target (`git rev-list --count`, `git show --name-only`).
   - A second pull makes no new commit. Add the case where the snapshot no longer
     returns the pulled notes: nothing to pull, no archive call.
   - The full dry-run Markdown equals the bytes the real run inserts.
   - A `-d` run succeeds while `pull.lock` is held by another process.
   - The CRLF test asserts there is no bare `\n` in the result.
   - The double-modification abort test asserts the target equals the externally
     modified bytes.

## 4. Time zones (`list.rs`, `pull.rs`)

`bob_env::current_datetime()` returns **local** wall-clock time as a `NaiveDateTime`.
`list.rs` calls `.and_utc()` on it for ages (Keep and vault rows) and formats
`fetched_at` with a `Z` suffix. `pull.rs::current_ts` does the same for journal `ts`.
Outside UTC, every Keep AGE is off by the UTC offset (4h on US Eastern).

- Convert properly: `Local.from_local_datetime(&naive).earliest()` →
  `.with_timezone(&Utc)`, via one small shared helper in `gkeep/ui.rs`.
- Use that helper for ages, `fetched_at`, and journal `ts`.
- Add a `gkeep_list` test with `TZ=America/New_York` and a fixed `BOB_NOW` that pins a
  Keep age which would be wrong under the old code.

## 5. Rendering and planning (`render.rs`, `plan.rs`, `ledger.rs`)

1. **`^id` escape.** Also escape a block id when the caret starts the text (a child
   whose text is just `^abc`) or follows any Unicode whitespace, NBSP included. Mirror
   `collect_done::trailing_block_id_in_line`.
2. **Dataview `::` escape.** Escape **every** colon in any run of two or more colons
   (`:::` → `\:\:\:`). Today's non-overlapping replace leaves `::` in odd-length runs.
3. **Leading child markers.** Also escape:
   - `#{1,6}` + space headings, not only `# `;
   - thematic breaks: lines that are only `---`, `***`, or `___`, spaces allowed;
   - code fences: leading ` ``` ` or `~~~`.
4. **Source URL.** Percent-encode `%`, `(`, `)`, `<`, `>`, `[`, `]`, and whitespace in
   `note.url`.
   - A crafted id inside the URL can otherwise plant a parseable `%%gkeep:v1:…%%` marker
     before the real one.
   - Add a golden test with a spoof in the URL: `parse_markers` on the rendered
     `Source:` line must return only the real marker.
5. **Empty notes.** `plan::is_empty` must use the renderer's normalization. A note whose
   text is only zero-width characters is `empty`.
6. **Pluralization.** Render `1 item`, not `1 items`, in the untitled-list fallback.
7. **Journal precedence.** The journal is a _lower-priority_ ledger. Consult journal
   `written` records only for ids that have **no** ledger entry. Add a planner test:
   ledger has the id with an old fp and the journal has the current fp → `revised`.
8. **REF selection.** `resolve_ids` rejects an empty `-i ""` as unknown (exit 2).
9. **Target validation.** `gkeep.target` containing a `..` component is invalid (in
   `gkeep/config.rs`).

## 6. List (`src/native/gkeep/list.rs`, `tests/gkeep_list.rs`)

- **`↺ still in Keep`.** Show it for any vault task whose marker id is a non-archived
  note in the snapshot, whatever its state. Pinned, shared, and empty notes count too.
- **Vault rows oldest first.** Sort by `created`, then line. Rows without `created` go
  last, in file order.
- **Empty states.**
  - A missing target file shows a dim
    `  gkeep_inbox.md not found · create it or set gkeep.target` line, not "No open
    tasks".
  - With `--all`, the empty state reads `No tasks`.

## 7. File modes (`ledger.rs`, `login.rs`)

- Create the journal file and the login `master_token.recovered` file with mode `0600`
  **at open** (`std::os::unix::fs::OpenOptionsExt::mode(0o600)`), not chmod afterwards.
- Before appending, `Journal::append` writes a leading `\n` if the existing file is
  non-empty and doesn't end with one. A torn previous append then can't corrupt the next
  record.
- `Journal::read` skips lines that aren't valid UTF-8 and counts them, rather than
  failing the whole read.

## 8. Test-harness flakes

- **`tests/gkeep_support/mod.rs` and `tests/gkeep_adapter.rs`.** `run_fake` runs the
  fake adapter script straight from the multi-threaded test process. It must retry on
  ETXTBSY (`raw_os_error() == Some(26)`, 20ms sleeps, up to 100 attempts), like
  `adapter.rs::spawn_adapter`. Proposed by bob-cli-2d.4.
- **Stub scripts in `gkeep_auth.rs`, `gkeep_list.rs`, `gkeep_pull.rs`, and the support
  module.** Configure the stub `token_command` and `token_store_command` scripts as
  `sh '<path>'`, so they are read and never exec'd directly, avoiding ETXTBSY exit 126.
- **Unit tests in `gkeep/config.rs` and `gkeep/render.rs`.** Remove
  `unsafe { std::env::set_var/remove_var }` from `pin_utc` and the config tests.
  - Use the repo's existing env guard/lock pattern if there is one.
  - Or refactor the code under test to take the value as a parameter.

## 9. Clippy dead code and cleanups (gkeep only)

`cargo clippy --all-targets --all-features` must report **zero** warnings under
`src/native/gkeep/` and `tests/gkeep*`. Use these decisions:

- **Use the existing helper:**
  - `PlanAction::as_str` in pull's JSON, replacing the inline mapping.
  - `ArchiveStatus::is_success` in pull, replacing the `"archived" | "already_archived"`
    string re-matches.
  - `ListFormat::is_json` in `list.rs`, replacing `args.format == ListFormat::Json`.
- **Delete:**
  - `RenderedBlock.task_text`.
  - `Plan.duplicates` and `PlanSummary.duplicates`. `list` and `pull` already use
    `ledger.duplicates()`.
  - `ListSource::as_str`.
  - `Spinner::is_running`.
  - `DoctorArgs::error_format`.
  - `KeepNote::edited_local`.
  - `FailureResponse`, `AdapterError`, `AdapterErrorKind` in `model.rs`.
    `adapter::into_result` reads the raw JSON on purpose.
  - `ui::status_glyph` and `ui::warning_glyph`.
    - Decision: `list`/`pull`/`doctor`/`login` keep their always-on `✓ ! ✗ ·` glyphs.
    - The epic plan pins those literal outputs, and the piped-output tests expect them.
    - A doctor row without a glyph would lose its status.
    - Update the `ui.rs` doc comments so they no longer claim a color-only rule.
- **Also fix:**
  - the redundant closures in `plan.rs` and `adapter.rs`;
  - `&PathBuf` → `&Path` in `spawn_adapter`;
  - the single-element `for` loop in the `plan.rs` tests;
  - the leftover no-ops in `pull.rs` (`let _ = target_rel;` and similar);
  - the identical glyph match arms.

## 10. Help and docs

- **Top-level help.** `bob gkeep --help`'s after-help gains the sentence "Running
  `bob gkeep` with no command runs `bob gkeep list`." and the same `Environment:` block
  as the subcommands (`BOB_DIR`, `BOB_CONFIG_FILE`, `BOB_GKEEP_ADAPTER`). Extend
  `tests/gkeep_cli.rs` to assert both.
- **Runner example.** In `src/runner.rs`, the `bob gkeep` `AFTER_HELP` example text must
  describe what it does: show the Keep inbox and `gkeep_inbox.md` side by side, not
  "drain".
- **`docs/gkeep.md`.**
  - The JSON section gets full schemas that match the code exactly:
    - `list`: `keep` object / `{error}` / `null`, `account`, `fetched_at`, `notes[]`
      fields, `vault.path`, `tasks[]` fields, `summary` keys;
    - `pull`: `dry_run`, `archive_enabled`, `commit_enabled`, `target`, `commit`,
      `notes[]` including `skip_reason`/`archive`/`detail`, top-level `markdown`,
      `summary`, and the new `error` object on archive failure;
    - `doctor`: `checks[].{name,status,summary,hint}`.
  - The List section documents:
    - the vault table (AGE/STATUS/TASK);
    - the hints (`+N lines`, `☐ n ☑ m`, `📎 n`, `↺ still in Keep`);
    - the empty states;
    - the footer variants, including `✓ Keep inbox is clear · N pinned stays in Keep`.
  - Document the new `other` attachment kind and the escaping additions from section 5.
- **`README.md`.** Put the Gkeep entry in the Contents list in the same order as the
  sections (after Plugins).

## 11. Verify and commit

1. Run `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
   `cargo test`.
   - The only allowed clippy failure is the pre-existing `tests/cli.rs:31818` deny.
   - There must be no gkeep warnings.
2. Run
   `cargo test --test gkeep_adapter --test gkeep_auth --test gkeep_list --test gkeep_pull`
   five times in a row. `cargo test --lib gkeep` must also pass five times.
3. Run `just install-smoke`, and `just check-adapter` if network is available.
4. Commit the work (conventional `fix(gkeep): …` subjects that mention bob-cli-2d)
   through the normal SASE final flow.

## 12. Closeout of epic bob-cli-2d (final step; do this yourself)

1. Run `sase bead epic-symbols bob-cli-2d`. For every listed `--epic-symbol` entry,
   resolve the symbol per the Symvision epic-whitelist policy: wire it up, privatize it,
   add a non-test pragma, or delete it. There are no open later beads, so never re-key
   an entry. At planning time the list was empty.
2. Close the epic:

   ```bash
   sase bead close bob-cli-2d --note "<verification>"
   ```

   The note must summarize:
   - The land review: all seven phases verified against the source and the epic commits
     55fdb18..cad7c8e.
   - The chezmoi seed is committed (34c33aae).
   - There was no interleaved drift.
   - The follow-up triage: the tests/cli.rs:31818 clippy deny went to bob-cli-28 as a
     DISCOVERED ISSUE; the ETXTBSY flake was fixed here.
   - The fixes made in this tale.
   - The land-review nits deliberately declined. List them from the bob-cli-2d note
     titled `LAND DECLINED`.
   - The verification commands and their results.

   Never use `--force` merely to make the close succeed. If the close is rejected
   because of leftover `--epic-symbol` entries, clean them up and close again.

3. Run `just symvision` if that recipe exists. At planning time, `just --list` had no
   such recipe; if so, say so in your final response.
4. Set `status: done` in the frontmatter of the epic's plan file, the `PLAN` path from
   `sase bead read bob-cli-2d -r "<why>"` (`plan:202609/bob_gkeep_inbox_drain.md`).
   Commit that change the same way the plan file's repository is normally committed.
5. Epic bob-cli-2d has no `parent_bead`. Confirm with
   `sase bead read bob-cli-2d -r "Need the parent link"`. If it has none, finish
   normally.
