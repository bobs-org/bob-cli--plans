---
tier: tale
size: medium
title: Restore archive pair-index read performance and land bob-cli-j5.4.1
goal:
  Archive pair resolution costs no more than max(25 ms, 15%) over pre-epic read latency
  on every reader, with unchanged semantics, and epic bob-cli-j5.4.1 and its parent
  phase bob-cli-j5.4 are closed with verified evidence.
proposed_by: bbugyi200.athena.bob-cli-j5.4.1.land
proposed_with: claude/opus
bead: bob-cli-j5.4.1
status: done
---

- **PARENT:**
  [202611/archive_pairing_and_rename.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/archive_pairing_and_rename.md)
- **BEAD:**
  [bob-cli-j5.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j5/bob-cli-j5.4.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-j5.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-j5.4.1.land.md)
- **COMMITS:**
  - [d65be1c](https://github.com/bobs-org/bob-cli/commit/d65be1ce02b66f9a08c6a31049990aa74cccca65)
    — perf(archive): cache pair facts for read-only consumers

# Restore archive pair-index read performance, then land bob-cli-j5.4.1

This tale is the last piece of epic `bob-cli-j5.4.1` ("Complete archive pairing and
guarded rename-following"). Its lander checked all five closed phases against the source
and the epic's commits (`b97d8d4c`, `0ed552a5`, `0996111c`, `2327590f`, `03dfe6de`). On
the integrated tree, `just check` and `just docs-check` pass, there are no
`--epic-symbol` entries, and the live read-only preview matches the plan. Read the
lander's note on the epic with
`sase bead read bob-cli-j5.4.1 -r "Need the land review note and plan path"` before
starting.

One issue caused by the epic is still open, and it must be fixed before the epic closes.
The epic plan required the reader migration to keep existing read performance. It did
not. Release-profile (`--profile local-install`) medians on the live vault (about 6.7k
Markdown notes, athena), installed pre-epic `bob` against HEAD `03dfe6de`:

| Command (`BOB_DIR=~/bob`)                                       | Before | After  |
| --------------------------------------------------------------- | ------ | ------ |
| `capture-related -f json -- "fix the archive rename flow"`      | 59 ms  | 224 ms |
| `capture-related -d 0 -f json -- "fix the archive rename flow"` | 39 ms  | 201 ms |
| `ref list -f json`                                              | 165 ms | 330 ms |
| `stats -f json`                                                 | 171 ms | 247 ms |

Under `strace`, `capture-related` opens 7,579 `.md` files after the change, against 856
before. `capture-related` drives the Bob Mac Capture "Similar tasks" band, which runs
after each settled draft, so this latency is user-facing.

## Root causes

All causes are in code this epic added.

1. `collect_done/pair.rs`: `PairCensus::read_vault` (behind
   `ArchivePairIndex::read_vault`) reads the full contents of every note, one at a time,
   and keeps every body in `contents: BTreeMap<PathBuf, String>`. Resolution only needs
   three things: the set of note paths (for existence checks and the `NoteIndex`), each
   note's raw frontmatter `done_tasks:` value (non-archives) or `parent:` value
   (archives, via `link_repair::archive_parent_target`), and the set of paths that could
   not be read.
2. `capture_related/corpus.rs::collect_from_archives` calls `read_vault` whenever any
   archive falls inside the mtime window. After each nightly archive, that is almost
   every call.
3. `ref_tasks/walk.rs::RefTaskIndex::build` already reads every note in parallel
   (`read_prefiltered_parallel`), then calls `read_vault` and reads the whole vault a
   second time. The epic plan said to keep the frontmatter facts from the existing read,
   before the `#ref` prefilter. This implementation does not do that.
4. `stats/index.rs` calls `ArchivePairIndex::from_buffers`, which clones every note
   buffer (about 14 MiB) into the census. `collect_done/plan.rs` calls `from_contents`,
   which clones the captured contents map the same way.

## Fixed constraints

- Pair resolution semantics must not change. Every existing truth-table test in
  `collect_done/pair.rs`, and all stats, brief, ref-task, highlights, migrate-tasks,
  capture-related, and collect_done tests, must pass. Only constructor plumbing may
  change in those tests; no assertion may change. The retained read-failure semantics
  stay exactly as they are: a census failure, plus refusal only for lookups it could
  plausibly affect, as in the canary CLI test.
- `read_vault` keeps exactly its current path universe (`collect_markdown_files`: skip
  dot directories and symlinks, case-insensitive `.md`). Any shared or reused read must
  produce lookups identical to `read_vault` for every path.
- Lookups do no disk I/O. Mutating consumers (`ref_tasks/insert.rs`, Highlights sync
  execution, reference-task migration) must never use an incomplete or stale census as
  proof that no conflict exists.
- No CLI, JSON, or docs contract changes, no version or `CHANGELOG.md` edits, and no
  memory edits. No live vault mutation. The only live command allowed is the read-only
  `task archive -n -f json` preview, plus the read-only timing commands above.

## Steps

1. **Store facts, not bodies, in the census.** In `collect_done/pair.rs`, replace
   `PairCensus.contents` with a per-path facts map. Each entry holds the raw frontmatter
   target the resolver needs and is computed once per note. Existence checks
   (`contents.contains_key`) use the key set. `resolve_pairs`, `resolve_parent_claim`,
   and `resolve_done_tasks_claim` consume the facts. `from_buffers`, `from_contents`,
   and `with_contents` extract facts from borrowed text and clone no bodies. Keep
   frontmatter-only parsing: a `done_tasks:` in the body or inside a fence must stay
   invisible.
2. **Read the census in parallel.** Make `PairCensus::read_vault` extract facts on
   scoped worker threads. Follow the `std::thread::scope` and `AtomicUsize` pattern
   already in `ref_tasks/walk.rs::read_prefiltered_parallel`, or factor out a small
   shared helper. Keep path order deterministic and retain read failures as before. Only
   the frontmatter is needed, so reading less than the whole file is optional, but a
   fact parsed from a bounded read must be identical to one parsed from the full text.
3. **Read the vault once in `RefTaskIndex`.** `RefTaskIndex::build` must take the pair
   facts from the same parallel read that feeds its `#ref` prefilter, not from a second
   `read_vault`. Its own path list (`capture_dependency_tasks::vault_note_paths`, minus
   `lib/`, `xlib/`, and conflict copies) differs from the `read_vault` universe, so the
   pair census must still cover the full `read_vault` universe. For example, walk the
   `read_vault` universe once, read each file once, and filter the `#ref` subset from
   those same bytes. Keep the no-archive fast path lazy, and keep
   `RefTaskIndex::archive_pairs()` working for preview reuse.
4. **Stop cloning in stats and collect_done.** `stats/index.rs` and
   `collect_done/plan.rs` build the index through the borrowing constructors from
   step 1. `StatsIndex` keeps its single-read contract.
5. **Tests.** Add a parity test showing that `read_vault`, `from_contents`, and the
   `RefTaskIndex`-derived index return identical `for_source`/`for_archive` results on
   one fixture. The fixture includes reciprocal noncanonical pairs, nested sources,
   `lib/` and `xlib/` notes, a conflict-copy name, a dot directory, false body or fenced
   `done_tasks:` text, and an unreadable note. Run the focused suites for every module
   touched, then `cargo fmt --all`, `just lint`, and `just check`. Do not run
   `just check-full`.
6. **Measure.** Build HEAD with `cargo build --profile local-install --bin bob`. Build
   the baseline from commit `b97d8d4c` (the last commit before the pair index): export
   it with `git archive b97d8d4c | tar -x -C <new empty temp dir>` and build it there
   with the same profile. Put the baseline in its own temporary `CARGO_TARGET_DIR`, and
   delete the temp dir afterwards. For each command in the table above, take the median
   of at least seven runs against the live vault, alternating binaries. **Target:** each
   HEAD median is no more than `max(25 ms, 15%)` above its baseline.
7. **Only if step 6 misses the target:** add a fingerprinted census cache for the
   read-only surfaces (`capture_related` and `RefTaskIndex` residence and preview).
   Model it on `capture_drawers::load_drawer_catalog`: put it under
   `bob_env::bob_cli_cache_dir()`, version and vault-key it, and invalidate each file on
   its `(mtime_ns, size)` fingerprint. Store only the per-note facts. Mutating consumers
   keep a fresh read. Add tests for cache hits, per-file invalidation, and a version
   mismatch. Then measure again. If the target is still missed, record the numbers and
   the reason in the close note instead of closing silently, then file a follow-up
   through `/sase_new_task` that names `bob-cli-j5.4.1` as the proposing bead.
8. **Live preview.** Run the freshly built binary with `BOB_DIR` set to the live vault:
   `task archive -n -f json`. Resolve the binary path from Cargo's real target directory
   (`cargo metadata --format-version 1 --no-deps`, `target_directory`). Do not install
   the binary or apply the plan. Confirm the result still matches the lander's record:
   exit 0 preview, one rename (`done/2026/20260528_day_done.md` to
   `done/2026/20260528_done.md` for `2026/20260528.md`), and all six diagnostics counts
   at 0. If the vault has changed since then (for example, new task moves after a
   nightly run), explain each difference, and never mutate the vault.

## Closeout (final step; do it in this same turn as the code)

9. Run `sase bead epic-symbols bob-cli-j5.4.1`. For every entry listed, either resolve
   the symbol (wire it up, privatize it, add a non-test pragma, or delete it, per the
   Symvision epic-whitelist policy) or re-key it to a still-open bead that needs it.
   Close the epic with `sase bead close bob-cli-j5.4.1 --note "<verification>"`. The
   note must cover: the lander's verification (five phases checked against source and
   commits; no `PROPOSED FOLLOW-UP` entries, so none were filed; intervening commits
   `3f8c49d4`, `b5196709`, `076fdc48`, `8728fe28`, `8d098e55` needed no integration; the
   lander added a stats-after-rename test, a CLI Git rename test, and a `pair.rs` doc
   fix) and this tale's results (before and after medians, the parity test, gate
   results, and the live preview result with its SHA). Never pass `--force` just to make
   the close succeed. Then run `just symvision` and confirm the whitelist is clean. Then
   set `status: done` in the frontmatter of the epic's plan file, using the PLAN path
   that `sase bead read bob-cli-j5.4.1` shows.
10. The epic's `parent_bead` is phase bead `bob-cli-j5.4` ("One archive pairing
    resolver, rename-following, and health diagnostics"), inside outer epic
    `bob-cli-j5`. Check that the closed epic delivered everything that phase requires:
    - the metadata-first resolver (`collect_done/pair.rs`);
    - every listed caller migrated (archiver destination and `link_repair.rs` helpers,
      `stats/index.rs` and `periods brief`, `capture_related/corpus.rs`,
      `ref_tasks/walk.rs`, `ref_tasks/insert.rs`, `highlights_ref/sync.rs`,
      `ref_library/migrate_tasks/plan.rs`);
    - `rename_following = rename` with guarded recoverable removal;
    - all diagnostics in human and JSON output, including dry runs;
    - tests (one archive per renamed source, contradictory binding refused, each
      diagnostic, stats and periods attribution after a rename);
    - the live check;
    - docs in `docs/task-archive.md`.

    Run `sase bead epic-symbols bob-cli-j5.4` and resolve or re-key any entries. Then
    close only that phase with
    `sase bead close bob-cli-j5.4 --note "<what you verified, including the live preview SHA, rename list, task move count, and all health-code counts>"`.
    Do not close the outer epic `bob-cli-j5` or any of its other phases; its own land
    agent handles them.
