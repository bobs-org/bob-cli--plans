---
tier: tale
size: medium
title: "bob gkeep: finish the bob-cli-2d closeout and land the epic"
goal:
  "The defects and test gaps left after the bob-cli-2d closeout commit d0c1692 are
  fixed: the Ctrl-C adapter orphan, small pull contract gaps, missing regression tests,
  help duplication, and docs drift. Epic bob-cli-2d is then closed, with its plan file
  marked done."
proposed_by: bbugyi200.apollo.2w
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.2w](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.2w.md)
- **COMMITS:**
  - [c603111](https://github.com/bobs-org/bob-cli/commit/c603111d7d231e1fee673850175d8a702064d197)
    — fix(gkeep): finish bob-cli-2d closeout per 202609/gkeep_land_resume.md

# Plan: finish the `bob gkeep` closeout, then land epic bob-cli-2d

## Why the epic is still open

- Epic **bob-cli-2d** ("bob gkeep: drain the Google Keep inbox into Obsidian tasks") has
  all seven phases closed. Its code is commits 55fdb18 … cad7c8e, plus the closeout
  commit d0c1692.
- Its land agent (`bob-cli-2d.land`) reviewed the epic, recorded two notes on the bead
  (`LAND TRIAGE` and `LAND DECLINED`), and wrote the closeout tale
  `plan:202609/gkeep_land_closeout.md`. That tale's final section (§12) was the epic
  closeout.
- The tale's coder implemented §1–11 and committed them as d0c1692. It then stopped: its
  response says "Not done here: epic `bob-cli-2d` close … and plan-file `status: done`",
  and it declared `bead_action: keep`.
- Nothing resumes a lander-authored tale. So the epic has stayed `in_progress`, assigned
  to `bob-cli-2d.land`, and its plan file still says `status: wip`.

A re-audit of d0c1692 against that tale, done while planning:

- `cargo fmt --check` is clean.
- Clippy reports zero gkeep warnings (details in §7).
- `cargo test` is fully green.
- `sase bead epic-symbols bob-cli-2d` has no entries.
- Every §1–10 item is present in code.

The audit also found:

- one regression d0c1692 introduced (Ctrl-C orphans the adapter);
- small contract gaps;
- regression tests the tale required but never added, or added too weakly;
- a duplicated help sentence;
- docs that do not match the code.

This plan fixes those, then lands the epic.

## Definition of done (read this first)

This task is **not** done until §9 is complete:

- `sase bead read bob-cli-2d -r "<why>"` shows the epic `CLOSED`.
- The epic's plan file has `status: done`, and that change is committed.

Do not end your turn with the epic still `in_progress` just because the code is
finished. Skipping this step is exactly why this plan exists. If something truly blocks
the close:

1. Record a note on the epic: `sase bead note bob-cli-2d "<blocker>"`.
2. Say so plainly at the top of your final response.

## Hard rules

These still apply from the epic plan:

- **Never contact live Google Keep.** Do not run `bob gkeep` against the real account.
  All tests use the fake adapter (`BOB_GKEEP_ADAPTER`) and temp vaults.
- **No new crates.** There is no `libc`/`nix`/`signal-hook`. Use `std` and
  `std::process::Command`.
- **Follow the CLI rules** if you touch help text. Read them with
  `sase memory read cli_rules.md -r "<why>"`.
- **Do not re-triage follow-ups.** The follow-up triage is already recorded on
  bob-cli-2d.
- **Do not reopen declined findings.** Leave alone everything in the bob-cli-2d note
  titled `LAND DECLINED` (items a–k).
- **Leave `tests/cli.rs:31818` alone.** Its pre-existing `|| true` clippy deny belongs
  to epic bob-cli-28.
- **The epic plan is the contract.** It is `plan:202609/bob_gkeep_inbox_drain.md`; get
  its path from the `PLAN` line of `sase bead read bob-cli-2d -r "<why>"`. Read its
  shared-contract, `pull`, `list`, and rendering sections before changing behaviour.

## 1. Adapter lifecycle (`src/native/gkeep/adapter.rs`, `scripts/gkeep_adapter.py`)

### 1.1 Ctrl-C no longer stops the adapter (regression from d0c1692)

**Cause.** `spawn_adapter` now puts the adapter in its own process group
(`process_group(0)`), so a terminal Ctrl-C (SIGINT to the foreground group) reaches only
`bob`. bob has no signal handler, so it dies at once and releases its locks. The
orphaned `uv` and Python keep running: they keep archiving, and a stalled snapshot
lingers forever because bob's timeout died with bob.

**Fix.** Keep the process group, since the timeout kill needs it. Make the adapter exit
when bob is gone:

- **Rust.** `spawn_adapter` sets `BOB_GKEEP_PARENT_PID=<std::process::id()>` on the
  adapter command.
- **Python, at the start of `main`.** If `BOB_GKEEP_PARENT_PID` parses as a positive
  int, start a daemon thread (`threading`, stdlib only) that checks every 0.5s with
  `os.kill(pid, 0)`:
  - `ProcessLookupError` → `os._exit(1)`;
  - `PermissionError` → the process is alive, keep going.
  - Put the check in a small helper (e.g. `_parent_alive(pid) -> bool`) so it can be
    tested.
  - With the variable unset or invalid (`--self-test`, `just check-adapter`), start no
    watchdog.
  - Stopping mid-run is safe: `_save_state` already writes the state file atomically
    (temp file + `os.replace`).
- **Where to document it.** Add one sentence to the "How it works" part of
  `docs/gkeep.md`. It is an internal variable, so it does **not** go in any
  `Environment:` help block.

**Tests:**

- A Rust unit test: a fake adapter records `$BOB_GKEEP_PARENT_PID`, and the value equals
  `std::process::id()`.
- `--self-test`: `_parent_alive(os.getpid())` is true. For the dead case, spawn
  `subprocess.run(["true"])`, let it be reaped, and check that its pid reads as dead.

### 1.2 A straggler that holds the pipes after a normal exit

**Cause.** After `try_wait` returns `Some`, the reader joins have no deadline. If the
group leader exits while another group member still holds stdout or stderr (for example,
`uv` killed by the OOM killer while Python lives on, or an adapter that backgrounds a
child), bob hangs no matter what the timeout is.

**Fix.**

- After the leader exits, with any status, kill the rest of its process group before
  joining the readers. Normally the group is already empty.
- Use a helper that ignores a failed `kill` (ESRCH). It must **not** fall back to
  `child.kill()` on the already-reaped child.
- The timeout path keeps `kill_adapter_tree` as it is.

**Test.** A unit test with a fake adapter that runs `sleep 30 &`, prints a valid ping
response, and exits 0, with a 2s timeout. It must return success within a few seconds.

### 1.3 Internal errors: scrubbed tracebacks that users can actually see

**Python:**

- `_report_internal` and the catch-all in `main` both call `traceback.print_exc`
  directly. Replace that with writing `scrub(traceback.format_exc(), secrets)` to
  stderr. Tokens must never appear, even in tracebacks.
- Only format a traceback while an exception is being handled. The self-test currently
  calls `_report_internal` outside an `except`, which prints a stray `NoneType: None`.
  Raise and catch inside the self-test instead.
- The self-test's item-3 check must assert all of these (capture stderr with
  `contextlib.redirect_stderr(io.StringIO())`):
  - the response shape (`ok: false`, `error.kind == "internal"`);
  - a secret inside the exception message is redacted in both the message and the
    captured traceback;
  - a passing `--self-test` prints only `ok`, with nothing on stderr.

**Rust:**

- An adapter that exits 0 with `ok:false` currently has its stderr thrown away, so the
  traceback never reaches the user.
- For an `ok:false` response whose `error.kind` is `internal`, append the adapter's
  stderr tail (the last 20 lines, the same as `adapter_crash`) to the error message.
- Other kinds (`auth`, `network`, `rate_limit`, …) are unchanged.

**Test.** A fake adapter writes a marker line to stderr and prints an `internal`
`ok:false` response. The error message must contain the marker.

### 1.4 Deterministic stdin tests

The EPIPE test added in d0c1692 sends a ~30-byte ping, which fits in the pipe buffer, so
it almost never exercises EPIPE. Add unit tests that call `run_request` directly with a
request payload of more than 1 MiB (e.g. a `serde_json::Value` holding a long string):

- The script exits 0 at once without reading stdin, after printing a valid response.
  Expect success: the EPIPE is ignored.
- The script never reads stdin and sleeps 30s, with a 1–2s timeout. Expect a `timed out`
  error within a few seconds.

### 1.5 Small cleanup

- Delete `PINNED_GKEEPAPI` and `PINNED_GPSOAUTH` from the Python script if nothing
  references them any more. Check the script, the Rust code, and the docs.
- `just check-adapter` must still print `ok`.

## 2. Pull (`src/native/gkeep/pull.rs`, `ledger.rs`, `login.rs`, `tests/gkeep_pull.rs`)

1. **Check the target before taking the vault lock.**
   - The epic's pipeline step 3 checks that the target exists (exit 2, hint "create it
     or set `gkeep.target`") before step 4 takes `bob_sync.lock`. Today the check runs
     after the 60s lock wait, so while vault-sync holds the lock, a missing target
     reports a lock timeout instead.
   - Move only the `target_path.is_file()` check up, right after `resolve_ids`. Reading
     the target bytes and scanning the ledger and journal stay after the lock.
   - Add a test that a missing target exits 2 with the hint. If the harness can hold the
     isolated `BOB_VAULT_SYNC_LOCK_FILE` the way the existing held-`pull.lock` test
     holds `pull.lock`, run it with the lock held.
2. **Missing git blocks only vaults that really are repos.**
   - Today any failure to start `git` is a commit failure. That breaks the epic row
     "Vault not a Git worktree → writes, `commit: null`, archives" on hosts without git.
   - When `ob::detect_git_worktree` returns `Err`, check whether `bob_dir` or any
     ancestor contains a `.git` entry (directory or file):
     - if one does, keep today's path: a commit failure, archive nothing, exit 1;
     - if none does, treat the vault as not a worktree.
   - Unit-test the ancestor check.
3. **`--dry-run --no-archive`.**
   - Human rows for `archive_only` notes must read
     `already in vault · left in Keep (--no-archive)`, not `would archive · …`.
   - In JSON, archiving disabled (`-n`) reports `not_requested` for every note with no
     archive result, in dry runs too. A dry run with archiving enabled keeps
     `not_attempted`.
   - Test both.
4. **The archive-failure JSON keeps its `markdown`.**
   - When the adapter fails during archive, the block was already written and committed,
     yet the document reports `"markdown": null`.
   - Report the inserted block, the same value the success path reports.
   - Extend the existing archive-crash tests to assert:
     - `markdown` is non-null;
     - every note due for archive has a non-empty `detail`;
     - `error.kind`, `error.message`, and `error.hint` are present in JSON;
     - in human mode, the error line and hint appear on stderr, and the summary line
       uses the warning/error prefix, not `ok`.
5. **Human `--quiet` verify failure reports each note once.** Today each failure is
   printed twice on stderr, once after verify and once in the quiet report. Keep one of
   the two.
6. **Permissions are applied before writing and failures are reported.**
   - The login recovery file (`master_token.recovered`) and the journal: after opening
     the file, set mode `0600` on the open handle (`File::set_permissions`) **before**
     writing any bytes. This covers a file that already existed with looser permissions.
   - Propagate a failure (as `?` or a mapped error) instead of the current
     `let _ = fs::set_permissions(…)`.
7. **Tests the closeout tale required but did not get (or got too weakly).**
   - **Normal pull, byte-exact.** Compare the whole target file after the pull to an
     expected literal string (fixed `BOB_NOW` and `TZ=UTC`, like the existing tests).
     Today the test is labelled "Byte-exact" but only counts markers.
   - **Dry run vs real run.**
     - The dry-run JSON `markdown` must equal the real run's JSON `markdown`.
     - The target after the real run must contain that Markdown verbatim, exactly once.
     - Remove the `unwrap_or(&after)` fallback, which hides mismatches.
   - **Double-modification abort.** Assert that the target bytes **equal** the bytes the
     test hook wrote, not `contains`/`starts_with`.
   - **Empty snapshot on the second pull.** Also assert the `nothing to pull` output.
   - **Quiet failure.** With `--quiet`, a failing pull leaves stdout empty and reports
     on stderr.
   - **`ledger.rs` unit tests.**
     - `Journal::read` skips and counts non-UTF-8 lines.
     - `Journal::append` writes a leading `\n` after a torn record with no trailing
       newline.

## 3. Rendering, time zones, and list

1. **Bare list markers (`render.rs`).**
   - A child whose whole text is `-`, `*`, or `+` renders as a live nested empty list
     item.
   - Change `[+*-]\s` in `LEADING_MARKER_RE` to `[+*-](\s|$)`, matching the `#{1,6}` and
     `N.` branches.
   - Add unit tests for all three.
2. **DST gap (`ui.rs::local_naive_to_utc`).**
   - When `Local.from_local_datetime(naive).earliest()` is `None` (a wall-clock time
     that doesn't exist), retry with `naive + 1 hour` before the final `.and_utc()`
     fallback.
   - Add a `tests/gkeep_list.rs` test modelled on the existing `TZ=America/New_York`
     test:
     - `BOB_NOW` is `2026-03-08 02:30`, a time that doesn't exist in that zone;
     - the note was created `2026-03-08T06:45:00Z`;
     - the Keep AGE must read `45m` (the old fallback shows `now`).
3. **Remove the `#[cfg(test)]` date fork (`render.rs`).**
   - `created_date` and `created_datetime` currently format in UTC under `#[cfg(test)]`,
     so the render unit tests never run the production local-time code. The comment
     above that code is also wrong.
   - Refactor so the time zone is a parameter. For example, a private
     `render_note_in<Tz: chrono::TimeZone>(note, indent, revision, tz: &Tz)` that
     `render_note` calls with `&chrono::Local`, while the unit tests pass
     `&chrono::Utc`.
   - Production output must not change. Use no env mutation.
4. **List tests for closeout §6 (`tests/gkeep_list.rs`).** The code is in place but
   nothing guards it:
   - `↺ still in Keep` shows for vault tasks whose Keep note is pinned, shared, or empty
     (and not archived);
   - vault rows are sorted oldest first by `created`, then by line, with rows that have
     no `created` last, in file order;
   - a missing target prints exactly
     `  gkeep_inbox.md not found · create it or set gkeep.target`;
   - `--all` with no tasks prints `No tasks`.

## 4. Finish the §9 cleanups (pull.rs, ui.rs, adapter.rs)

1. **Use the typed status.**
   - Store the typed `ArchiveStatus` (plus detail) in pull's archive-status map instead
     of strings.
   - Delete the `unreachable!()` scaffolding around `is_success()`, and the remaining
     `"archived" | "already_archived"` and `"changed" | "missing" | "error"` string
     matches. Use `is_success()` and a typed match.
   - Turn the status into a string only when building JSON or human output (reuse or add
     `ArchiveStatus::as_str`).
2. **Remove unused parameters.**
   - d0c1692 turned leftover no-ops into unused parameters (`_bob_dir`, `_target_rel`).
     Remove them, along with the older unused `_config` and `_styler`, from the
     functions and their call sites.
   - Remove each `#[allow(clippy::too_many_arguments)]` that is no longer needed. Add no
     new `allow` attributes.
3. **Small fixes.**
   - The `ui::warn` doc comment must describe the real prefix, `bob gkeep: warning:`.
   - The test helper `assert_request_protocol` takes `&Path`, not `&PathBuf`.

## 5. Help (`src/native/gkeep/cli.rs`, `tests/gkeep_cli.rs`)

- `bob gkeep --help` prints "Running `bob gkeep` with no command runs `bob gkeep list`."
  twice, because d0c1692 added it to the after-help while `long_about` already had it.
- Remove it from `long_about` and keep it in the after-help, so `-h` and `--help` each
  print it exactly once.
- Extend `tests/gkeep_cli.rs` to assert:
  - the sentence appears exactly once in both `-h` and `--help`;
  - the top-level help has the `Environment:` header and all three variables.

## 6. Docs (`docs/gkeep.md`)

The docs must match the code exactly. Check each item below against the code that builds
the output before editing.

**pull JSON:**

- `commit` is the full 40-character SHA, or `null`; it is not a short SHA.
- `error` appears only when the adapter fails during archive.
- `skip_reason` is `empty|pinned|shared|archived|null`.
- `markdown` is the inserted block (after §2.4, also on archive failure); it is `null`
  when nothing was written.
- The `archive` values mean:
  - `not_requested`: skipped notes, and archiving disabled with `-n`;
  - `not_attempted`: a dry run, or archiving never reached;
  - the rest as listed in the docs.
- `-q -f json` prints nothing on success.

**list JSON:**

- The `keep: {error}` form appears only for snapshot failures. Config, token, and
  adapter-resolve failures print the generic error document.
- A missing target in JSON gives `tasks: []`.

**doctor JSON:**

- Add the top-level `ok` (false, with exit 1, when any check fails).
- `hint` may be `null`.
- The check names are `config|account|token|adapter|keep|target|git`.

**List section:**

- An empty Keep reads `Keep inbox is empty ✓`.
- The singular hint is `+1 line`.
- Document every part of the actionable footer, as printed by `list.rs`.
- `↺ still in Keep` never appears with `-s vault`.

**Rendering:**

- The `Google Keep image note` title fallback.
- Inner-whitespace collapsing and stripping of a leading bullet on the first line.
- The heading escape also covers a bare `#`–`######` with nothing after it.
- A bare `-`/`*`/`+` child is escaped (from §3.1).

**Failure matrix:**

- Split the row "Verify or commit failure | Exit 1; nothing is archived":
  - A verify failure leaves only the failing notes unarchived. Verified notes are still
    archived, per epic step 7.
  - A commit failure archives nothing.
- Add the missing-git rule from §2.2.

**Configuration:**

- A `gkeep.target` containing a `..` component is rejected.

**How it works:**

- The `BOB_GKEEP_PARENT_PID` watchdog (§1.1).
- `internal` adapter errors include the adapter's stderr tail (§1.3).

## 7. Verify

1. Run `cargo fmt --check`.
2. Run `cargo clippy --all-targets --all-features`.
   - The only allowed error is the pre-existing `tests/cli.rs:31818` deny.
   - That error aborts compilation of the `cli` test target, and the build may stop
     early. So also run
     `cargo clippy --all-features --lib --test gkeep_adapter --test gkeep_auth --test gkeep_cli --test gkeep_list --test gkeep_pull`.
     It must report **zero** warnings under `src/native/gkeep/` and in `tests/gkeep*`.
   - The 17 existing lib warnings are all outside gkeep; leave them.
3. Run `cargo test`. The whole suite must pass.
4. Run
   `cargo test --test gkeep_adapter --test gkeep_auth --test gkeep_list --test gkeep_pull --test gkeep_cli`
   five times in a row, and `cargo test --lib gkeep` five times. All runs must be green.
5. Run `just install-smoke`, and `just check-adapter`, which needs network the first
   time. If there is no network, say so.

## 8. Commit the code

The primary repository's commit goes through the normal SASE final flow, with a
conventional `fix(gkeep): …` subject that mentions bob-cli-2d. Do not commit by hand.
Finish §9 **before** submitting the final declaration.

## 9. Closeout of epic bob-cli-2d (mandatory final step; do it yourself)

1. **Clear epic symbols.** Run `sase bead epic-symbols bob-cli-2d`.
   - For every `--epic-symbol` entry, resolve it per the Symvision epic-whitelist
     policy: wire it up, privatize it, add a non-test pragma, or delete it.
   - There are no open later beads, so never re-key an entry.
   - At planning time the list was empty.
2. **Close the epic.** Write the close note to a temp file outside the repo, then run
   `sase bead close bob-cli-2d --note @<file>`. The note must summarize:
   - **The land review:** all seven phases were verified against the epic commits
     55fdb18..cad7c8e. The chezmoi `gkeep:` seed is committed (34c33aae). No other
     commits interleaved with the epic.
   - **The follow-up triage:** the `tests/cli.rs:31818` clippy deny went to bob-cli-28
     as a DISCOVERED ISSUE. The ETXTBSY flake and the adapter EPIPE race were fixed in
     d0c1692.
   - **Why a second closeout was needed:** the closeout tale's coder committed d0c1692
     but skipped the epic close.
   - **What d0c1692 fixed,** in one line per closeout-tale section.
   - **What this plan fixed:** §1–6 above, briefly.
   - **The land-review nits deliberately declined:** list items a–k from the bob-cli-2d
     `LAND DECLINED` note.
   - **Declined during this re-audit, with reasons:**
     - Text-line lookalikes such as `[ ] foo`, raw HTML, link reference definitions, and
       `$$` still render as Markdown. They fall outside the epic's escaping table, and a
       typed `[ ] foo` turning into a checkbox matches what the author meant.
     - `-q -f json` on an adapter failure still writes an error line to stderr as well
       as the JSON document on stdout. The JSON stays a single document on stdout.
   - **The verification commands from §7 and their results.**

   Never use `--force` merely to make the close succeed. If the close is rejected
   because of leftover `--epic-symbol` entries, clean them up and close again.

3. **Symvision.** Run `just symvision` if that recipe exists. At planning time,
   `just --list` had no such recipe; if it still doesn't, say so in your final response.
4. **Mark the plan file done.** Open the epic's plan file, the `PLAN` path shown by
   `sase bead read bob-cli-2d -r "<why>"`. It lives in the `plans` sidecar checkout
   (`sase/repos/plans/202609/bob_gkeep_inbox_drain.md`).
   - Change its frontmatter `status: wip` to `status: done`, and make no other edit.
   - In your final declaration, give that sidecar repository a `commit` decision next to
     the primary repository. Message:
     `chore(plans): mark bob gkeep epic plan done after bob-cli-2d closeout`.
5. **Parent bead.** Epic bob-cli-2d has no `parent_bead`. Confirm with
   `sase bead read bob-cli-2d -r "Need the parent link"`. If it has none, finish
   normally.
6. **Confirm.** Before the final declaration, run
   `sase bead read bob-cli-2d -r "Confirm the epic closed"` and check that it shows
   `CLOSED`. Your final response must say so.
