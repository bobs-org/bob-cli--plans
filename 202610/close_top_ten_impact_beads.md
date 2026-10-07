---
tier: epic
title: Close the ten highest-impact bob-cli task beads
goal: "Every bead in the 48-hour impact ranking (bob-cli-4j, 2e, 21, 33, 59, 4m, 4x, 4r,
  4u, 3c) is implemented, verified on the final master tree, and closed before this epic
  lands. As a result, master's test gate is honest and green: `just check` runs every
  test binary and passes twice in a row on athena. The artifact-link store accepts
  writes again. Native Tasks queries no longer fail on the 2 s sandbox deadline. A bare
  plugin sync can no longer roll back the vault. The zorg-era reading records are in the
  reference library.

  "
phases:
  - id: red-tests
    title: Fix the deterministic red tests
    depends_on: []
    size: small
    description:
      "red-tests: give `ref create --audio` a completion decision (bob-cli-4j), confirm
      the stale `clip` help snapshot is fixed (bob-cli-5i), and make the listen-card
      test accept both Pandoc ampersand escapings without losing URI coverage
      (bob-cli-4u); close all three."
  - id: env-isolation
    title: Stop lib tests from racing on process environment
    depends_on: []
    size: medium
    description:
      "env-isolation: replace every module-private env-mutating test helper with one
      shared isolation mechanism that never lets a test see another test's BOB_DAY_FILE
      or BOB_NOW, enforce it with clippy, stress-test it, and close bob-cli-2e,
      bob-cli-40 (superseded) and bob-cli-5c."
  - id: check-gate
    title: Add the canonical just check gate
    depends_on:
      - red-tests
      - env-isolation
    size: small
    description:
      "check-gate: add `just check` (fmt, clippy, and every test binary with
      --no-fail-fast), make `just test` stop masking binaries, clear any remaining
      clippy deny or environment-dependent CLI test, document the gate, prove it green
      twice, and close bob-cli-3c."
  - id: link-store
    title: Repair the artifact-link event store
    depends_on: []
    size: large
    description:
      "link-store: plan and run a backed-up data repair of the colliding operation_id
      events in the plans sidecar so `sase artifact doctor` is healthy and typed links
      write again, backfill the relations kept as free text, propose the sase hardening
      as follow-ups, and close bob-cli-21."
  - id: tasks-sandbox
    title: Build the Tasks JS sandbox only when a query needs it
    depends_on:
      - check-gate
    size: medium
    description:
      "tasks-sandbox: construct the Tasks JavaScript sandbox lazily and give its
      initialization its own budget apart from the 2 s per-expression deadline, add
      regression tests, measure before/after latency on the live read path, and close
      bob-cli-33."
  - id: plugins-sync-guard
    title: Refuse bare plugin syncs from a different bob-plugins checkout
    depends_on:
      - check-gate
    size: small
    description:
      "plugins-sync-guard: make `bob plugins sync` without `--repo` abort before any
      pull or copy when run from inside a bob-plugins checkout other than the resolved
      repo, test it, document the rule in bob-cli and bob-plugins AGENTS.md, and close
      bob-cli-59."
  - id: ref-migration
    title: Migrate zorg-era reading records into the reference library
    depends_on:
      - check-gate
    size: xlarge
    description:
      "ref-migration: settle the migration design with Bryan, author and drive a nested
      epic that moves the ~424 unindexed zorg-era reading records into `ref/` through a
      dry-run-first, idempotent, reversible vault write, and close bob-cli-4x once `bob
      ref doctor` coverage confirms it."
proposed_by: bbugyi200.athena.0y2
create_time: 2026-10-07 14:38:39
status: wip
---

- **PROMPT:**
  [prompts/202610/close_top_ten_impact_beads.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/close_top_ten_impact_beads.md)

# Plan: Close the ten highest-impact bob-cli task beads

## 1. Why this epic exists

The research report
`research:202610/task_bead_48h_impact_ranking/task_bead_48h_impact_ranking.md` ranked
the ten most impactful task beads from the 48 hours before 2026-10-07. Read it with
`sase artifact read` before starting any phase. This epic implements and closes every
bead in that ranking. Its central finding: one deterministic lib-test failure
(bob-cli-4j) makes plain `cargo test` stop before 14 of its 15 test binaries run, so
1,306 integration tests silently never run. That has already let a real regression
(bob-cli-5i) land unseen. Making the gate honest comes first, and most other phases
depend on it.

### 1.1 The ten beads and who closes them

| Rank | Bead       | Status at planning                    | Closed by            | Close criterion (summary)                                           |
| ---- | ---------- | ------------------------------------- | -------------------- | ------------------------------------------------------------------- |
| 1    | bob-cli-4j | ready, ci, small                      | `red-tests`          | kinds coverage test passes; `--audio` completes files               |
| 2    | bob-cli-2e | ready, bug, small                     | `env-isolation`      | no lib test mutates shared process env unguarded; stress runs clean |
| 3    | bob-cli-21 | ready, bug, large                     | `link-store`         | `sase artifact doctor` healthy; `sase artifact link add` works      |
| 4    | bob-cli-33 | ready, bug, medium                    | `tasks-sandbox`      | no-by-function queries build no sandbox; init has its own budget    |
| 5    | bob-cli-59 | ready, bug, small                     | `plugins-sync-guard` | bare sync from a foreign plugins checkout refuses before pull/copy  |
| 6    | bob-cli-4m | **already closed** (done, 2026-10-06) | lander verifies      | still closed; walk-identity tests pass                              |
| 7    | bob-cli-4x | ready, feature, xlarge                | `ref-migration`      | nested epic landed; `bob ref doctor` coverage confirms              |
| 8    | bob-cli-4r | **already closed** (done, 2026-10-07) | lander verifies      | still closed; `bob ref doctor` reports `annotations: ok`            |
| 9    | bob-cli-4u | ready, ci, small                      | `red-tests`          | listen test passes on Pandoc 3.1.3 and 3.1.11.1                     |
| 10   | bob-cli-3c | ready, bug, small                     | `check-gate`         | `just check` exists, runs every binary, passes twice on athena      |

### 1.2 Companion beads closed with them

These beads are not in the ten, but they share a root cause with ranked beads, or they
block the ranked beads' acceptance. The research's recommended sequencing folds them in:

- **bob-cli-5i** (ci, small): the `bob ref -h` snapshot still listed `clip`. Commit
  6244ddd (the bob-cli-52 landing) appears to have already removed the line from
  `tests/fixtures/help/ref-short.txt`. `red-tests` verifies this and closes the bead.
- **bob-cli-40** (flake): the same test and cause as 2e. `env-isolation` closes it with
  resolution `superseded`.
- **bob-cli-5c** (flake): the `BOB_NOW` variant of the same race class. Its root cause
  is not yet confirmed. `env-isolation` must confirm the cause before closing it.

## 2. Rules for every phase

- **Read first.** Read your bead(s) with `sase bead read <id> -r "<why>"` before you
  change anything. The bead descriptions and +1 evidence give exact file paths, line
  numbers, and reproduction commands. Line numbers drift, so re-locate them in current
  source.
- **Closing.** When a phase's change is committed and its verification passes, the phase
  closes the beads it owns with
  `sase bead close <id> --note "<what you verified, on which commit and host>"`. Use
  `-R superseded --reason "..."` only where this plan says so. If you cannot meet a
  bead's close criterion, leave the bead open. Then append a note to your phase bead
  that explains exactly what remains; the lander must finish it (see §10).
- **No new beads.** Phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead.
- **Run every test binary.** Until `check-gate` lands, plain `cargo test` hides
  failures. Always verify with `cargo test --no-fail-fast` (or explicit `--lib` plus
  `--test cli` runs). Two known failures belong to sibling phases until those phases
  land: bob-cli-4j and bob-cli-4u fail deterministically, and the env race fails
  intermittently. Skip them by name with `--skip` rather than treating them as yours. A
  failure that is not on that list is a finding: report it.
- **Active epics touching nearby code.** Rebase onto the latest master before you
  commit, keep your diff confined to your phase, and do not take over another epic's
  scope.
  - bob-cli-5j (paired return links) edits the `bob ref create` Pandoc path in
    `src/native/highlights_ref/create.rs`.
  - bob-cli-3j (shell completion) owns the completion stack. Its phases are all closed,
    and its land agent is finishing.
  - bob-cli-28 owns the clippy deny in `tests/cli/capture/pomodoro_name.rs`. §5 covers
    what to do if it is still present.
- **Other repositories.** The linked bob-plugins repo, the plans sidecar, and the `sase`
  project must be opened with `/sase_repo`. Read research and plan artifacts with
  `sase artifact read`.

## 3. Phase `red-tests`: Fix the deterministic red tests

Owns: bob-cli-4j, bob-cli-5i, bob-cli-4u.

### 3.1 bob-cli-4j: `ref create --audio` has no completion-kinds decision

- `native::completion::kinds::tests::every_value_arg_has_a_decision` fails on every run
  with `value-taking args without a kinds decision: ["ref create:audio"]`. The arg is
  `Arg::new("audio")` in `src/native/highlights_ref/create.rs` (`--audio PATH`, `-a`).
  It is shared by `bob ref create` and the `bob highlights create` path.
- Fix: give it `.value_hint(clap::ValueHint::FilePath)`, following how sibling path args
  declare their decision. If the codebase convention for this command family is a
  kinds-table entry in `src/native/completion/kinds.rs`, use that instead. Do not add an
  exemption.
- Verify:
  - `cargo test --lib native::completion::kinds::tests::every_value_arg_has_a_decision -- --exact`
    passes.
  - The completion endpoint offers files after `bob ref create --audio `, for example
    `bob __complete zsh --protocol 1 -- bob ref create --audio ''` from a directory
    containing a file. Use the protocol the completion tests use.

### 3.2 bob-cli-5i: `bob ref` help snapshot listed `clip`

- Run `cargo test --test cli help::ref_help_matches_grouped_snapshot -- --exact`.
- If it passes, which is expected after 6244ddd, grep the help fixtures and `docs/` for
  `clip` listed as a visible `bob ref` subcommand. Fix any remaining occurrence.
- If it fails, delete the stale `clip` line from `tests/fixtures/help/ref-short.txt`.
- Either way, the hidden alias must keep working: `bob ref clip --help` (also
  smoke-tested by `just install-smoke`).

### 3.3 bob-cli-4u: the listen-card Pandoc test pins ampersand escaping

- `native::highlights_ref::create::tests::listen_filter_renders_card_and_encoded_play_link`
  asserts the exact LaTeX string
  `obsidian://open?vault=Research\%20Notes\&file=lib\%2Fchat\%2Freport.mp3`. Pandoc
  3.1.11.1 (athena, the landing host) emits a bare `&` there. Pandoc 3.1.3 (apollo)
  emits `\&`.
- **First confirm correctness.** Check that both forms yield the same PDF link target
  (hyperref's `\href` accepts both). If the bare `&` actually breaks compilation or the
  link, fix the renderer instead of the test.
- If both forms are correct, change the assertion to accept either `\&` or `&` at that
  position. Still assert all of the following:
  - the percent-encoded segments (`Research\%20Notes`, `lib\%2Fchat\%2Freport.mp3`);
  - the `\BobListenCard{` / `\BobListenPlay{}` structure;
  - narration-link removal.
- Do not merely drop URI coverage.
- Do not fix bob-cli-58 (Markdown PDF headings) here, even though it shares this Pandoc
  path.
- Record `pandoc --version` in the close note. Verify on 3.1.11.1 when the phase runs on
  athena. Otherwise say which version you tested; the lander re-verifies on athena.

## 4. Phase `env-isolation`: Stop lib tests from racing on process environment

Owns: bob-cli-2e, bob-cli-40, bob-cli-5c.

### 4.1 Problem

Lib tests share one process. Eleven modules mutate process environment with
`std::env::set_var` / `remove_var`, each behind its own private lock or none:

- `capture_pomodoros.rs`
- `capture_complete/tests/mod.rs` (`DAY_FILE_LOCK`)
- `capture_project_note.rs`
- `completion/{mod,verify}.rs`
- `highlights_ref/{fetch,listen,arxiv}.rs` and `highlights_ref/tests/audio.rs`
- `ref_jobs/kick.rs`
- `ob.rs`

Meanwhile, other tests read the same variables ambiently. Two known victims:

- `capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
  first calls `list_capture_pomodoros` with the ambient `BOB_DAY_FILE`.
- `note_ready::tests::scan_excludes_r3_and_r7_paths` reads `BOB_NOW` twice through
  `bob_env::current_datetime()`, while capture_complete tests set it.

The race is load-dependent. It reproduced on busy landing hosts (up to 5 of 5 runs) and
0 of 5 times on a lightly loaded apollo.

### 4.2 Fix

The preferred fix is test-scoped overrides that never touch process env. Any design is
acceptable if it removes the race class, not just these two tests.

- Add one crate-wide, `#[cfg(test)]` env-override facility. Recommended: a thread-local
  override map plus a panic-safe scoped guard, so a test's overrides are invisible to
  other test threads. It must support both setting and unsetting a variable.
- Route production env reads that tests override through `bob_env` accessors that
  consult the override in test builds. At minimum: `BOB_DAY_FILE`, `BOB_NOW`, `DATE`,
  `BOB_DIR`, `HOME`, `XDG_*`, and every variable the listed modules set.
- Where the code under test passes environment to a child process or another thread,
  pass the value explicitly (for example with `Command::env`). Only if that is
  impossible, use the one shared crate-wide process-env mutex, and have readers of those
  variables take it too.
- Delete every module-private `with_env` / `ENV_LOCK` / `DAY_FILE_LOCK` in favor of the
  shared facility.
- Enforce it: add `clippy.toml` `disallowed-methods` entries for `std::env::set_var` and
  `std::env::remove_var`, with a reason that points at the shared facility. The only
  `#[allow]` may be inside that facility, if a process-env fallback remains.
- A crate-wide `RwLock` (mutators write-lock, env-reading tests read-lock) is an
  acceptable alternative only if you can show that every env-reading test takes it.

### 4.3 Verify, then close

- `rg 'env::(set_var|remove_var)' src` matches only the shared facility, or nothing.
- Run `cargo test --lib` at default parallelism at least 10 times under load. For
  example, run a concurrent `cargo build --release` in another target dir, or pass
  `-- --test-threads=64`. Both victim tests must pass every run. Also loop the two
  victim tests plus `capture_complete` and `note_ready` tests 20 times. Record the
  counts.
- bob-cli-5c: confirm that `BOB_NOW` interference is the cause, for example by forcing
  the interleaving before the fix. If the shared facility removes it, close 5c `done`.
  If it turns out to be different shared state, fix it here if it is small. Otherwise
  leave 5c open, with a phase-bead note for the lander.
- Close bob-cli-2e `done`. Close bob-cli-40 with
  `-R superseded --reason "fixed by bob-cli-2e's shared test env isolation"`.

## 5. Phase `check-gate`: Add the canonical just check gate

Owns: bob-cli-3c. Runs after `red-tests` and `env-isolation`, so the suite should be
green.

### 5.1 justfile changes

- Add `check` as the canonical gate:
  - `cargo fmt --check`;
  - `cargo clippy --all-targets --all-features`, where any deny-level error fails;
  - `cargo test --no-fail-fast`, so the lib, every `tests/*` binary, and doctests all
    run even when one binary fails.
- Change `test` to `cargo test --no-fail-fast` too, and keep `all` working (same steps),
  so `just all` / `just test` can never again stop after the lib binary.
- Use the existing `_banner` style.
- Do not add `check-full`.

### 5.2 Other failures that would keep the gate red

- **Clippy deny.** Check whether `tests/cli/capture/pomodoro_name.rs` (~line 808) still
  has the tautological `|| true` assertion (`clippy::overly_complex_bool_expr`). It is
  owned by in-progress epic bob-cli-28. If it is still there:
  - Replace it with a meaningful assertion of the intended invariant. Find that
    invariant from bob-cli-28.1's commit 22abed4 and its plan.
  - If there is no real invariant, remove the tautological `assert!`.
  - Then append a note to bob-cli-28 (`sase bead note bob-cli-28 "..."`) saying it was
    fixed here, with the commit.
- **Environment-dependent CLI test.**
  `capture::r#ref::capture_url_with_markers_or_flags_stays_a_task` failed under a stale
  SSH-forwarded `DISPLAY` (xclip). If the CLI test harness still lets spawned `bob` see
  the ambient `DISPLAY` / `WAYLAND_DISPLAY`, scrub them in the shared test command
  builder so the gate does not depend on the session. Verify with
  `DISPLAY=:99 cargo test --test cli capture_url_with_markers`.
- **Anything else.** Any other failing test owned by an active epic (bob-cli-5j or
  another) is not absorbed here. Record it as a PROPOSED FOLLOW-UP naming the owner.

### 5.3 Docs, proof, and close

- Document the gate in the README's development section, where `just all` is described:
  what `just check` runs, and that it runs every test binary.
- Proof:
  - `just check` passes twice in a row at default parallelism. Prefer athena; record the
    host and test counts.
  - Demonstrate that masking is gone. Temporarily break one lib test locally, without
    committing it. `just check` must still run and report the CLI binary.
- Close bob-cli-3c.

## 6. Phase `link-store`: Repair the artifact-link event store

Owns: bob-cli-21. This is a `large` phase. Plan first; the plan gate is where Bryan
approves the repair, because the plans sidecar is shared by every workspace and machine.

### 6.1 Known facts

- Since about 2026-09-07, every `sase artifact link add` in this project fails with:
  `artifact-link event store is invalid: validation: operation_id de29d2e25c1cfb4381f223c44d576f8c was reused for different artifact link events`.
- `sase plan propose` with a `links:` inlet archives the plan, then crashes before the
  approval gate.
- `sase artifact doctor` exits 1 and reports
  `Link event objects 977 durable / 10482 pending (delta +9505)`.
- The operation_id appears in two plans-sidecar objects:
  - `link-events/v1/6e/6efae635a11ed1b059bcf2570f6dfcb6bbf69a96f4b7d2e92d5df125f60fe26c.json`
  - `link-events/v1/51/513bdb2010562a67007a473f61a187bcb41705f6c0e99eb130631f767620c233.json`

### 6.2 The plan must cover

1. Open the plans sidecar and the `sase` project with `/sase_repo`. Read the link-event
   store code: validation, reduction, operation_id semantics, and what "pending" vs
   "durable" means. Find out how the duplicate pair arose and which event is canonical.
   Also find out whether the 9,505-event pending backlog is a symptom of the same
   blockage.
2. Take a restorable backup of every object you will touch. Prefer sase's own repair
   path, such as `sase artifact doctor --fix`, if it can resolve the collision.
   Otherwise hand-repair the minimum set of event objects in a single sidecar commit,
   with a documented rollback.
3. Never run `prune --apply`, `reclaim --apply`, or trash purge.
4. Verify:
   - `sase artifact doctor` exits 0.
   - A real typed link writes and reads back, for example
     `sase artifact link add bead:bob-cli-5i related bead:bob-cli-4j "4j's lib failure masked 5i"`.
   - The pending/durable gap is reduced or explained.
5. Backfill the typed `related` links that beads recorded as free text because of this
   defect. Find them with `sase bead search` for notes saying typed links were rejected
   (known: 4t, 4u, 58, 5c, 5g, 5h, 5i, 24, 2t, 2u). List what you backfilled in the
   close note.
6. Hardening in sase is out of scope for this epic. Record both of these as PROPOSED
   FOLLOW-UPs for the lander:
   - one bad historical event pair must not block every future write;
   - `sase plan propose` must validate or publish the `links:` inlet before consuming
     the scratch plan, or roll the archive back on failure.
7. Close bob-cli-21.

## 7. Phase `tasks-sandbox`: Build the Tasks JS sandbox only when a query needs it

Owns: bob-cli-33.

### 7.1 Problem

`src/native/dataview/tasks/mod.rs` builds `js::JsSandbox::new(&index.tasks, ...)`
eagerly in four places: `query_matching_descriptions`, `query_rich_tasks`, `run`, and
`run_note`. Sandbox initialization JSON-parses the whole vault and hydrates every task
through Moment. It does so inside the 2 s `EXPRESSION_TIMEOUT` (`js.rs`) that is meant
for one user expression. On a busy host, `bob plan`, `bob freshness`, task-status-hooks
plan budgets, the tmux segment, and `bob query` then fail with a JavaScript
`interrupted` error that reads like a bad query. Even successful runs take about 5.5 s.

### 7.2 Fix

- Construct the sandbox lazily, only when the parsed query actually needs JavaScript (a
  `by function` filter, sort, or group, or any other JS-evaluated clause; audit every
  sandbox use).
- Give the initialization and hydration unit its own deadline, separate from the
  per-expression budget: either generous and fixed, or scaled with task count. That way
  by-function queries on a large vault on a busy host do not fail at init either.
- Keep the 2 s per-expression budget for user expressions.
- The error for a genuine init timeout must say it timed out initializing, not look like
  a bad query.

### 7.3 Tests

- A query without by-function constructs zero sandboxes. Use a construction counter or
  test hook.
- A by-function query on a fixture still works.
- Initialization succeeds when the expression budget is injected tiny. This proves the
  budgets are separate.

### 7.4 Measure, then close

Run each of these 3 times against the live vault before and after the fix. Both are read
paths.

- `bob freshness list -f json -l 1`
- `bob plan -f json`

Record the wall times in the close note; the research measured about 5.5 s. Then close
bob-cli-33.

## 8. Phase `plugins-sync-guard`: Refuse bare plugin syncs from a different bob-plugins checkout

Owns: bob-cli-59.

### 8.1 Problem

`bob plugins sync` without `--repo` resolves the plugins repo through
`bob_env::plugins_dir()` (`BOB_PLUGINS_DIR`, else
`~/projects/github/bobs-org/bob-plugins`; see `src/native/plugins/cli.rs` around the
`unwrap_or_else(bob_env::plugins_dir)`). It then git-pulls that checkout and copies its
files into the vault. Run from a SASE linked bob-plugins worktree, this silently deploys
the canonical checkout instead. On 2026-10-07 it rolled `ledger-tools` back from 1.34.0
to 1.33.0.

### 8.2 Fix

- When `--repo` is absent and the cwd is inside a git checkout that is a bob-plugins
  checkout whose canonical toplevel differs from the resolved repo, abort before any
  pull or copy, with a nonzero exit.
- Identify a bob-plugins checkout robustly: by its origin remote `bobs-org/bob-plugins`,
  or by a stable repo-root marker. Pick whichever the repo reliably has.
- The error names both paths and gives the exact command to run instead, for example
  `bob plugins sync --repo <toplevel> [-p <id>]`.
- Keep today's behavior in all other cases:
  - explicit `--repo`;
  - running from inside the resolved checkout itself;
  - running from outside any bob-plugins checkout;
  - `scripts/install_all`, which already passes `--repo`.
- Honor the same rule in `--dry-run`, so the preview refuses too. No new flags are
  needed. If you add one, read `cli_rules.md` with `/sase_memory_read` first.

### 8.3 Tests

Add CLI tests with temp git repos and a temp vault:

- a foreign checkout is refused, and the vault is untouched with no pull attempted;
- the resolved checkout is allowed;
- an unrelated cwd is allowed;
- explicit `--repo` is allowed.

Never run a bare sync against the live vault to verify this.

### 8.4 Docs, then close

- Update `docs/plugins.md`.
- Update bob-plugins `AGENTS.md`, which currently says only to run `bob plugins sync`.
  Open the linked repo with `/sase_repo` and commit there. It should say that from a
  SASE worktree you pass `--repo <worktree>` and `-p <id>` to move one plugin.
- Close bob-cli-59.

## 9. Phase `ref-migration`: Migrate zorg-era reading records into the reference library

Owns: bob-cli-4x. This is an `xlarge` phase. Its worker authors a nested epic plan with
`/sase_plan`, and this phase is complete only when that nested epic has landed and
bob-cli-4x is closed.

### 9.1 Background to read first

- bead bob-cli-4x;
- the bob-cli-4w (reference library) epic plan and its research, whose "Phase 5" scoped
  this migration out;
- `src/native/ref_library/coverage.rs`, whose counting rules already define the
  inventory;
- live `bob ref doctor` output: about 424 unindexed `status::` records, led by
  `work_ref.md` (75) and `nvim_ref.md` (69);
- the home project's `obsidian` reference memory (vault conventions, git sync, recovery
  runbooks), via `/sase_memory_read`;
- relevant `decisions` strands and `glossary` terms (Reference Note, Reference Task);
- the `/bob_ref` skill.

### 9.2 Settle the design with Bryan first

Before writing the nested plan, ask Bryan with `/sase_questions`. Offer concrete
options, each with a recommendation:

1. Destination layout: `ref/ai/` like the 282 existing legacy migrations, per-topic
   folders, or something else.
2. The legacy status → library status mapping for `UNREAD`, `COLLECT_FLEETING_NOTES`,
   `REVIEW_FLEETING_NOTES`, `REVIEW_LIT_NOTES`, `READ`, `ABANDONED`, and `BOOK`.
3. The dedupe rule against existing ref notes, for example the same URL, or the same
   `source_block` + `source_path`.
4. What happens to the source hub lines: removed, replaced by a link to the new ref
   note, or left with a marker.
5. Whether the migrator is a reusable `bob ref` subcommand or a one-off tool. If it is a
   subcommand, read `cli_rules.md`.

### 9.3 Requirements for the nested epic

- **Dry run first.** It produces a reviewable report: counts by source file and status,
  dedupe hits, and collisions. The real run is idempotent, so a rerun is a no-op.
- **Reversible.** The vault write is one reversible batch: a clean vault git state
  before, one sync/commit after, and a documented rollback per the obsidian runbooks.
- **Done when:**
  - `bob ref doctor` coverage reports 0 unindexed records, or only a residue Bryan
    explicitly accepted and the report lists;
  - `bob ref find` / `bob ref list` return migrated records;
  - the JSON envelope `coverage.scope` caveat is updated or removed.

## 10. Landing checklist (explicit instructions for the epic land agent)

The epic must not land while any of the ten ranked beads is open. Before declaring the
epic landed or closing the epic bead, do all of the following.

### 10.1 Audit every bead

Run `sase bead read <id> -r "landing audit"` for:

- the ten ranked beads: bob-cli-4j, 2e, 21, 33, 59, 4m, 4x, 4r, 4u, 3c;
- the companions: bob-cli-5i, 40, 5c.

Each must be `CLOSED`. Resolution must be `done`, except bob-cli-40 (`superseded`). Each
close note must state its verification.

### 10.2 Re-verify on the final master tree, on athena

| Bead                  | Verification                                                                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3c (+ the whole gate) | `just check` passes twice in a row at default parallelism.                                                                                                                |
| 4j                    | `cargo test --lib native::completion::kinds::tests::every_value_arg_has_a_decision -- --exact` passes.                                                                    |
| 4u                    | `cargo test --lib native::highlights_ref::create::tests::listen_filter_renders_card_and_encoded_play_link -- --exact` passes with athena's Pandoc (3.1.11.1 at planning). |
| 5i                    | `cargo test --test cli help::ref_help_matches_grouped_snapshot -- --exact` passes, and `bob ref clip --help` still works.                                                 |
| 2e / 40 / 5c          | `cargo test --lib` passes 5 runs in a row at default parallelism, and `rg 'env::(set_var\|remove_var)' src` matches only the shared facility.                             |
| 21                    | `sase artifact doctor` exits 0, and one real `sase artifact link add` succeeds.                                                                                           |
| 33                    | The no-sandbox regression test exists and passes; `bob freshness list -f json -l 1` and `bob plan -f json` each succeed 3 times against the live vault (record times).    |
| 59                    | The plugins-sync guard CLI tests pass. Never run a bare sync against the live vault to check this.                                                                        |
| 4x                    | The nested epic is closed, and `bob ref doctor`'s coverage row matches the accepted outcome.                                                                              |
| 4m                    | Still closed. The walk-identity tests pass in the linked bob-plugins checkout (`npm test`; open the repo with `/sase_repo`).                                              |
| 4r                    | Still closed, and `bob ref doctor` reports `annotations: ok`.                                                                                                             |

### 10.3 Finish anything still open or failing

- If any of the ten (or a companion) is not closed, or its verification fails, the
  lander owns it. Finish the work in the landing tale or a child plan, verify it, and
  close the bead. This includes 4m or 4r if a +1 has reopened them since planning.
- Do not close a ranked bead as `canceled` or `superseded` on your own authority. If one
  genuinely cannot be finished (for example, Bryan rejects 4x's migration design), stop
  and ask Bryan with `/sase_questions`.

### 10.4 Follow-ups, then close the epic

- Turn phase PROPOSED FOLLOW-UPs into task beads with `/sase_new_task`. This includes
  `link-store`'s two sase hardening proposals.
- Only then close the epic bead.

## 11. Out of scope

- bob-cli-58 (Markdown PDF headings) and bob-cli-3w (plugin perf flake): the report's
  "open-only" substitutes.
- bob-cli-5f / 5g (ref-job retries/drain).
- bob-cli-4z, 50, and 5h (they reverse recorded decisions or are blocked).
- The memory-bead batch (51, 5d, 5e, 4t, 4n, 5a, 4o, 57).
- Code changes inside sase itself.
