---
tier: tale
size: medium
title:
  "Finish and land bob-cli-52: URL routing correctness, hermetic tests, and the ref clip
  fold"
goal:
  "The URL-routing epic bob-cli-52 meets its plan contract: ref jobs recover and never
  spin, capture honors inline @@ and prints clean warnings, Keep pull needs the inbox
  note only for task writes, ingest classifies and stays silent, the tests are hermetic,
  the build is warning-free, and integration with the ref clip fold is green. The epic
  is then closed and its plan marked done."
proposed_by: bbugyi200.athena.bob-cli-52.land
bead: bob-cli-52
create_time: 2026-10-07 12:20:00
status: wip
---

- **PARENT:**
  [202610/url_capture_ref_routing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/url_capture_ref_routing.md)
- **BEAD:**
  [bob-cli-52](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-52/README.md)

# Finish and land bob-cli-52 (links go to the reading queue)

## Context

Epic **bob-cli-52** (plan `plan:202610/url_capture_ref_routing.md`; read it with
`sase bead read bob-cli-52 -r "<why>"` and open the PLAN path it prints) shipped all
nine phases:

- ingest `2eafe60`
- hardening `3232214`
- intent `df9d504`
- grammar `98fd8ae`
- gkeep `0a8c879`
- jobs `f4fb812`
- capture `c9b361b`
- mac (bob-mac-capture `bf43ab2` + `b19c913`, macOS CI green)
- live verification (bob-cli-52.9)

The land agent's audit of master `2d568fa` found that the work is real and broadly
matches the plan. Several gaps remain, though, all caused by the epic or by its
collision with unrelated commit `84a8a31` ("fold bob ref clip into bob ref create as
hidden alias", which landed mid-epic). This tale fixes those gaps and then closes the
epic.

**Follow-up triage is already done and recorded** on bob-cli-52 (the "LANDING FOLLOW-UP
TRIAGE" note). Do not file or re-triage follow-ups. The pre-existing failures stay out
of scope:

- `native::completion::kinds::tests::every_value_arg_has_a_decision` (bob-cli-4j);
- `native::highlights_ref::create::tests::listen_filter_renders_card_and_encoded_play_link`
  (bob-cli-4u);
- the parallel flakes
  `capture_pomodoros::...missing_note_and_missing_section_are_warning_successes`
  (bob-cli-40) and `note_ready::tests::scan_excludes_r3_and_r7_paths` (bob-cli-5c).

**Verification gate.** This repo has no `just check` recipe, so the gate is `just all`
(fmt, clippy, `cargo test`). `cargo test` stops at the first failing target, and the lib
target fails on the known items above. So also run `cargo test --no-fail-fast --tests`
and confirm that the only failures are those known items. Every other integration target
must be fully green, including `tests/cli` (today it has 2 failures that this tale
fixes) and `gkeep_pull`. Also run `just check-adapter`.

Shared testing rules from the epic still apply. No test may reach the network, the real
vault, the real `~/.local/state`, or a real browser. Use the fake curl, `FakeClip`,
`BOB_HIGHLIGHTS_RESOLVE`, an isolated `XDG_STATE_HOME`, and `BOB_REF_JOBS_KICK=off`. New
JSON fields stay additive.

## Work

### 1. Integrate with `84a8a31` (the `bob ref clip` fold)

1. `tests/fixtures/help/ref-short.txt` still lists a `clip` row under "Highlights
   pipeline". `84a8a31` made `clip` a hidden alias and removed it from `HELP_GROUPS`
   (`src/native/highlights_ref/cli.rs`), but the snapshot was last written by the jobs
   phase. Regenerate the snapshot so the group reads
   `create, doctor, jobs, marker, scan, sync`.
   `cargo test --test cli -- --exact help::ref_help_matches_grouped_snapshot` must pass.
2. In `src/native/highlights_ref/clip_adapter.rs` (`resolve_with`, about line 115), the
   missing-uv hint still says "bob ref clip runs its pinned web capture adapter". Change
   it to name `bob ref create`, and say that uv is also searched in `~/.local/bin`,
   `~/.cargo/bin`, `/opt/homebrew/bin`, and `/usr/local/bin`.
   - Keep the `uv was not found` substring: `ingest.rs::classify_unprefixed` maps it to
     `dependency`.
   - Then grep `src/`, `docs/`, and `README.md` for any other user-facing `bob ref clip`
     / `bob highlights clip` advice that epic code added, and point it at
     `bob ref create`. The hidden alias itself stays.

### 2. Ingest (`src/native/highlights_ref/ingest.rs` and its callees)

1. **Unmapped curl exits are `network`.** The plan's error table says curl exits "6, 7,
   35/60, and others" are `network` (retryable).
   - Today `fetch.rs` (about lines 339-345) reports other exits as
     `curl failed (exit N) ...`. `classify_unprefixed` doesn't match that text, so it
     falls through to `internal`, which is not retryable. That means a transient curl
     failure during `bob gkeep pull` writes the note as a ⚠️ task instead of leaving it
     in Keep.
   - Map `curl failed (exit` to `IngestErrorKind::Network`, and add a unit test.
2. **Split hints out of `fetch_and_route` errors.** `ingest_url` builds that error with
   `ingest_error(error.message.clone(), None)` (about line 446), so a `\nhint: …` line
   (since `84a8a31`, create's "pass --html" hint for HTTP failures) stays inside
   `IngestError.message` and lands in `done.jsonl`.
   - Run the message through `split_hint`, as `command_error` does.
   - Add a test that `message` has one line and `hint` carries the hint.
3. **Ingest prints nothing itself.** The ingest contract says it writes nothing to
   stdout or stderr. Two places break that:
   - `fetch.rs::fetch_with_curl` prints `fetching <host>…` to stderr on a TTY (about
     line 135). This garbles `bob gkeep pull`'s `Clipping … (i/N)` spinner on a
     terminal.
   - `pdf_target.rs` prints the ≥50 MiB `warning: PDF is … bytes` with `eprintln!`
     (about line 324).

   Thread a minimal quiet/progress option through the functions that ingest calls
   (`target::fetch_and_route`, `fetch::fetch_url`, and the PDF size check).
   - Ingest sends those lines through `IngestRequest::progress` (or drops the TTY line).
   - `bob ref create` keeps today's exact output: the characterization tests in
     `tests/cli/highlights/create.rs` must pass unchanged.

4. **Make the docs match the code.**
   - `ingest_url` composes the same building blocks as create: `resolve_url_syntactic`,
     `fetch_and_route`, `sources` dedupe, `pdf_target` planning/stamp/install, and
     `ClipAdapterClient`.
   - But `bob ref create` keeps its own printing routes and never calls `ingest_url`. It
     also never takes `ingest.lock`; design decision 13 says that lock serializes the
     capture worker and Keep pull.
   - Fix the three places that claim otherwise:
     - the module doc at the top of `ingest.rs` (lines 1-9);
     - the "Ingest boundary" section of `docs/highlights-create.md` (about lines
       130-140);
     - the `ref/ingest.lock` row in `docs/ref-jobs.md` (about line 29: "(shared with
       create)" → "(capture worker and bob gkeep pull)").
5. **Hermetic ingest unit tests.**
   - `second_ingest_waits_for_the_lock` takes the real
     `~/.local/state/bob-cli/ref/ingest.lock`. Split `lock_ingest` into a
     `lock_ingest_at(dir, progress)` core, with `lock_ingest` passing
     `bob_cli_state_dir().join("ref")`, and test against a temp dir.
   - `scratch_bob_dir_never_touches_bob_dir_env` mutates process-wide `BOB_DIR` inside
     the parallel unit suite, and it uses `"not a url"`, which fails before any vault
     read, so it proves nothing. Delete it and replace it with a CLI test in
     `tests/cli/highlights/jobs.rs`:
     - seed one ref job whose `bob_dir` is a scratch vault already holding a PDF-backed
       ref note for the URL;
     - run `bob ref jobs run` with `BOB_DIR` pointing at a sentinel path;
     - assert the outcome is `in_library` / `already_in_library` and the sentinel was
       never created.
   - In the `fallback_note` test (about lines 885-889), the `long`/`note` values are
     built and never asserted. Assert truncation to 120 characters plus `…` on the first
     line.
6. **Hermetic missing-uv case** in
   `tests/cli/highlights/create.rs::ingest_characterizes_url_failure_modes` (about lines
   2328-2350). It empties `PATH` but leaves `HOME` real, so `env::resolve_uv()` finds
   `~/.local/bin/uv` and runs the real web-clip adapter. The case fails on athena (it
   gets a 404 / private-address error instead of `uv was not found`).
   - Set `HOME` to an empty temp dir for that command.
   - If `/opt/homebrew/bin/uv` or `/usr/local/bin/uv` exists on the host, skip only that
     sub-case with an `eprintln!` note (those fallbacks are absolute paths).

### 3. Ref jobs (`src/native/ref_jobs/`)

1. **Stale `running/` recovery is off by one** (`worker.rs::recover_running`, about
   lines 105-130).
   - The plan says: increment `attempts`, then fail as `internal` at 2 or more. The code
     checks `job.attempts >= 2` before the increment, so the fallback fires on the third
     stale recovery while the message says "stopped twice".
   - Fix it so the first stale recovery goes back to `pending/` with `attempts: 1`, and
     the second fails as `internal` and writes the fallback.
   - Update the tests in `tests/cli/highlights/jobs.rs` (they seed attempts 0 and 2) to
     cover attempts 0 → pending and 1 → fallback.
2. **The worker can spin forever.** In `drain_pending` (about lines 180-205), a failing
   `move_job` into `running/` hits `continue` and re-reads the same first pending file
   forever. `park_unreadable` also ignores a failed rename the same way.
   - Remember the paths that failed in this pass and skip them (or end the drain). Leave
     them in `pending/`, set `stuck_now`, and exit 1.
   - Add a unit or CLI test that makes `running/` unwritable (mode 0500), then asserts
     that `bob ref jobs run` returns promptly with exit 1 and the job is still pending.
     Restore permissions in the test.
3. **The re-check pass loses the report.** In `run_jobs` (about lines 23-45), when the
   lost-wakeup re-check cannot take `worker.lock`, the function returns `0` straight
   away.
   - That drops the earlier passes' outcomes (never printed) and their exit code (1 when
     stuck).
   - Break out instead: print the accumulated report and return the max code so far.
     Only the very first lock failure should print `another clip worker is running` and
     exit 0.
4. **Modes and atomic writes** (State layout: dirs 0700, files 0600).
   - `ensure_spool_dirs` sets 0700 on `jobs/` and its subdirectories but not on the
     `ref/` parent.
   - `done.jsonl` (`spool.rs`, about line 472) and `worker.log` (`kick.rs`, about
     line 65) are created with the default umask mode. Create them 0600 with
     `OpenOptionsExt::mode`.
   - Stuck jobs are written with a plain `fs::write` (`worker.rs`, about line 322).
     Route them through the spool's existing atomic install (temp file, fsync, rename,
     0600).
   - Extend the unit tests to assert the modes.
5. **`bob ref -n jobs`** (the `bob ref` group flag before the subcommand) prints the
   `jobs` help instead of listing. `bob ref jobs` correctly treats the bare form as
   `list`. Make the bare-`jobs` rewrite also apply when `bob ref`'s own flags come
   before `jobs`, and add a CLI test.

### 4. Capture (`src/native/capture*`, `src/native/capture_language/`)

1. **An inline `@@` declaration must block the claim.** R1 says a draft that declares an
   `@@` global destination never claims a reference item. Today only standalone `@@…`
   lines set `has_global_destination` (`draft.rs` about lines 600-640, and the editor's
   `editor_parse.rs` about line 237). Inline tokens are collected only after items are
   claimed.
   - So `Buy milk @@groceries⏎⏎https://example.com/post` queues the link instead of
     sending it to `groceries.md` as a task.
   - Fix capture and capture-parse the same way: after all declarations are known, if a
     global destination exists and routing claimed any item, re-parse those items with
     `has_global_destination = true` so they become ordinary tasks that inherit the
     global.
   - Add CLI tests for `bob capture -d -f json` and `bob capture-parse -f json`: the URL
     item is a task routed to the `@@` target, and its mode is not `ref`.
2. **Show the config-error message, not its debug form.**
   `capture/mod.rs::load_capture_routing` (about line 154) prints `{error:?}`, so users
   see `URL routing is off: Invalid("…")`.
   - Give `ConfigError` a `Display` impl (or use its message), and print exactly
     `bob capture: warning: URL routing is off: <message>`.
   - Make the same fix where `bob gkeep pull` warns about a routing config error (it
     also uses `{error:?}`).
   - Tighten `tests/cli/capture/ref.rs` (about line 308) to assert the full line.
3. **Explicit nulls in `ref.library`.** The plan's capture JSON contract shows
   `"library": {"verdict": …, "path": null, "title": null, "reading_state": null, "message": null}`.
   `RefLibraryJson` (`capture/output.rs`, about lines 37-48) skips absent fields.
   - Drop those four `skip_serializing_if` attributes so the keys are always present.
     Bob Mac Capture decodes them with `decodeIfPresent`, so both forms decode.
   - Update affected assertions, and add one that a `not_found` item carries all four
     keys as `null`.
4. **Remove dead routing-off scaffolding.**
   - The planner arms that still return the usage error "reference items are not
     enabled" (`capture/plan.rs` about lines 801-803, `capture/batch.rs` about lines
     230-232) can no longer run, because `plan_capture_batch` routes Ref items first.
     Turn them into the same invariant/internal errors their neighbouring arms use.
   - Update the stale comments in `capture_language/model.rs` (about lines 9-11 and
     66-67) that say production callers keep routing off.
5. **Missing contract tests** in `tests/cli/capture/ref.rs`:
   - Human wording for the four cases not yet covered (`legacy`, `unknown`, `clipping`,
     `duplicate`), using the exact headline and dim detail text from the plan's Human
     wording table, both dry and real.
   - An `unknown` verdict test (make the ref dir unreadable, or use another seam that
     makes the library unreadable).
   - The forced flags `-s`, `-t`, `-S`, and `--task-ref` each keep a bare URL a task
     (only `-r` and `-c` are covered today).

### 5. Keep pull (`src/native/gkeep/pull.rs`, docs)

1. **Require the target note only for task writes.**
   - `needs_target` correctly covers task writes and permanent clip fallbacks. But
     `all_clipped` (about lines 312-335) also requires at least one successful clip. So
     two kinds of pull still demand `gkeep_inbox.md`, although neither writes tasks:
     - a pull whose only URL note fails retryably reports `target note … does not exist`
       instead of the clip-failure row (exit 1, note left in Keep);
     - a crash-recovery pull with only `ref_created` archive-only notes.
   - Require the target only when the run writes tasks. Keep today's check unchanged for
     runs that have no URL-only, `create_ref`, or `ref_created` archive-only notes at
     all, so the existing missing-target tests stay green.
   - Add `tests/gkeep_pull.rs` cases for both scenarios without `gkeep_inbox.md`.
   - Extend the existing crash-between-clip-and-archive test (about line 1528) to
     re-pull and assert the note is archived as `ArchiveOnly` with no second clip.
2. **Count failed fallback writes.** `count_failed` (about line 1412) checks only
   `Write`/`WriteRevision`.
   - A `CreateRef` permanent fallback whose task write fails verification therefore
     reports `ok: true` and `summary.failed: 0` (the exit code is already 1), and the
     human row still says "written as a task with a ⚠️ note".
   - Count it as failed, and make the row say it failed.
   - Add a test using the same verify-failure seam the existing `Write` tests use.
3. **Docs and help.**
   - Add `[-R|--no-ref]` to the `bob gkeep pull` usage line in `docs/gkeep.md` (about
     line 33) and `README.md` (about line 813).
   - Add one after-help example to `pull_after_help` (`gkeep/cli.rs`, about line 125)
     that mentions URL-only notes being clipped and `-R`.

### 6. Warning-free build

`cargo build` now prints 12 warnings, all from epic code. Fix each one:

- the unused re-exports in `ref_jobs/mod.rs` (lines 19-26), `url_routing/mod.rs` (lines
  12-15), and `capture_language/mod.rs` (line 64);
- `EditorItemOutcome` being more private than `parse_editor_item_with`
  (`editor_parse.rs`);
- the never-used `CaptureParseOptions::routing_off`, `IngestRoute::as_str`,
  `ref_jobs::kick::kick_with`, and `url_routing::policy::normalize_exclude_host`.

Remove the dead re-exports. Mark test-only helpers `#[cfg(test)]` or make production
code use them: `kick()` can call `kick_with(&current_exe)`, and `config/mod.rs` can
reuse `normalize_exclude_host` instead of its own copy (about line 599). Delete what is
truly unused. Then `cargo build` and `cargo clippy --all-targets --all-features` must
print no warnings from `ref_jobs`, `url_routing`, `capture*`, `highlights_ref/ingest`,
or `gkeep`.

### 7. Verify

1. `just all`, then `cargo test --no-fail-fast --tests`. The only failures may be the
   four known items in Context. `tests/cli`, `gkeep_pull`, and `gkeep_list` must be
   fully green, and `just check-adapter` must print `ok`.
2. Build `bob` and run a scratch-vault smoke with `-b <tmp vault>` and an isolated
   `XDG_STATE_HOME`:
   - `bob capture -d -f json 'https://example.com/post'`: library keys appear as `null`;
   - the inline-`@@` draft from 4.1 gives a task;
   - `bob ref -n jobs` lists;
   - `bob ref -h` shows no `clip` row.

### 8. Close out epic bob-cli-52 (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-52`. At planning time it reported "No
   --epic-symbol entries for bob-cli-52". If any entry appears now, resolve it (wire up,
   privatize, add a non-test pragma, or delete it per the Symvision epic-whitelist
   policy). Re-key a Justfile line to an open bead only if a still-open later bead
   really needs the exemption.
2. Close the epic with a note summarizing the verification:
   `sase bead close bob-cli-52 --note "<what was verified: all 9 phases audited against plan:202610/url_capture_ref_routing.md; bob-cli-4v closed; follow-up triage recorded in the LANDING FOLLOW-UP TRIAGE note (+1s on 4j/4u/40, new tasks 5b-5h, declined items with reasons); landing fixes from this tale (84a8a31 help-snapshot/hint integration, ingest kinds/hints/silence/docs, hermetic lock and missing-uv tests, worker recovery/spin/report/modes, ref -n jobs, inline @@ claim, config warning text, explicit library nulls, gkeep target check and failed-fallback count, docs, warning-free build); test results with the known failures named>"`.
   - Never use `--force` to make the close succeed.
   - If the close is refused because of leftover `--epic-symbol` entries, clean them up
     and close again.
3. Run `just symvision` if that recipe exists. At planning time this repo's justfile had
   none; if it is still absent, say so in your final response.
4. Set `status: done` (from `status: wip`) in the YAML frontmatter of the epic's plan
   file, `plan:202610/url_capture_ref_routing.md`. Use the PLAN path that
   `sase bead read bob-cli-52 -r "Need the plan path"` prints.
5. bob-cli-52 has no `parent_bead`, so nothing further needs closing.
