---
tier: tale
size: medium
title: Land starts-after relations — fixture re-sync, relation overhead cut, epic
  closeout
goal: Finish the last epic-caused work for bob-cli-fs and close it. Re-sync the plugin
  fixture copies. Bring relation-enabled capture overhead down to the shared byte
  pass the design priced. Then close the epic in the same turn.
proposed_by: bbugyi200.athena.bob-cli-fs.land
proposed_with: claude/opus
bead: bob-cli-fs
status: done
---

- **PARENT:**
  [202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/starts_after_relation.md)
- **BEAD:**
  [bob-cli-fs](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/README.md)

# Land starts-after relations (epic `bob-cli-fs`)

## Context

Epic `bob-cli-fs` ("Starts-after relations offer the next task when one is parked or
done") has all 11 phases closed. Its land audit verified the feature end to end:

- Rust policy and engine.
- Capture `x*` / `!` / `@route+id!` events.
- Authoring indications.
- The Obsidian picker, notice, cycler, and unlink.
- Bob Mac Capture.
- The chezmoi enable.

A release-build smoke on a scratch vault showed `x*` raising `event: parked` and `!`
raising `event: done` with the `Starts after “…”` group. Relations off gives no group.

The audit is recorded as a note on `bob-cli-fs`. Read it with
`sase bead read bob-cli-fs -r "<why>"`. It also holds the follow-up triage, which is
complete; do not redo it.

Two things the epic caused are still unfinished. This tale does both, then closes the
epic.

1. **Plugin fixture drift (epic note #3).** bob-cli `71907dab` (phase `bob-cli-fs.5`)
   changed `tests/fixtures/relation_successor/vectors.json`: SA31/SA32 left `ui_only`,
   and their `coverage` values became `"capture unlink"` / `"capture unlink silence"`.
   Both bob-plugins copies still carry the old bytes (sha256 `08ffcced…`; canonical is
   `b992209a…`):
   - `plugins/task-status-cycler/scripts/fixtures/relation_successor/vectors.json`
   - `plugins/bob-navigation-hotkeys/scripts/fixtures/relation_successor/vectors.json`

   As a result, `node --test scripts/test-task-status-cycler-relation-successors.cjs`
   fails `cycler relation successor vectors.json is byte-identical to bob-cli` (21/22
   pass). No plugin runner reads `ui_only` or `coverage`, and the `config`/`key`/`rule`/
   `copy` sections and counts (12/4/25/31) are unchanged. A byte copy is therefore the
   whole fix.

2. **Relation-enabled capture overhead has no owner.** `docs/task-dependencies.md` §12.7
   records apollo deltas of `x*` +83.83 ms and `!` +67.32 ms. Its notes concede that
   "`!` with relations enabled is not within a few ms of relations disabled, and `x*`
   still adds well beyond the shared byte pass". Bead `bob-cli-gc` explicitly excludes
   relation-specific overhead; it owns only the pre-feature `!` path. Chezmoi now
   enables `starts after`, so Bryan pays this on every close.

   The land audit measured on athena: 64 cores, release build of master `a414c171`,
   **same binary**, only `relations.successor_labels` toggled, 30 runs after 5 warmups.

   | Case                                                                         | off median | on median | Δ         | CPU median off → on |
   | ---------------------------------------------------------------------------- | ---------- | --------- | --------- | ------------------- |
   | `hello`                                                                      | 13.61      | 13.63     | +0.02     | 13.5 → 13.5         |
   | `x*` park, one matched source                                                | 58.09      | 97.72     | **+39.6** | 58 → 355            |
   | `!` on a task without `[id::]` (no dependents pass; zero matched sources)    | 58.35      | 86.35     | **+28.0** | 58 → 329            |
   | `!capture-measure:implement` (dependents pass also runs; one matched source) | 75.31      | 96.47     | **+21.2** | 362 → 357           |

   What these numbers say:
   - The relations-off dependents pass costs `!` off − `!noid` off ≈ **17 ms**. A
     relation-only pass costs 28–40 ms.
   - When both passes run, the byte pass is shared: CPU is unchanged. Yet wall time
     still grows by 21 ms. That is serial or post-pass work, not reads; `strace -c`
     shows the same ~6,000 open/read/close calls with relations on and off.
   - The plan's own target (an enabled `x*` that parks a linked task stays ≤ 70 ms) is
     met with relations off on athena (58 ms) and missed only because of this feature
     (98 ms).

The binding design is in the epic plan `plan:202611/starts_after_relation.md`, section
"Gates and performance". Relation sources come from one batch-scoped parallel byte pass,
shared with the dependents pass when both run. Released checks read only the T′ notes
involved. There is no persistent index or 60 s cache.

## Constraints

- Do not change behavior. Every `rule`/`copy`/`config`/`key` vector and every SA test
  must still pass. JSON stays additive at `schema_version` 1.
- Keep the zero-read guarantees for `hello` and plain `x`, and the existing read-count
  tests in `src/native/capture/relation_sources.rs` (for example
  `either_consumer_order_walks_and_reads_the_catalog_once` and
  `cheap_gate_rejects_without_catalog_or_source_reads`).
- No persistent cache, no relation-label scan cache, no LINKED FROM reads (design rule).
- Never time or write against the live `~/bob`; use a scratch copy.
- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`. Commits are `perf(capture): …` /
  `test(…)` / `docs(…)` with no `!`.
- Do not edit memory: the epic's DECISIONS allow glossary edits only, and those are
  done.
- bob-plugins is a linked repo. Open it with `sase repo open bob-plugins -r "<why>"`,
  use the printed path, and read its `AGENTS.md`/`CLAUDE.md` first. Never edit `~/bob`
  plugin copies directly.
- Do not run `just check-full`. The file-change gate is `just check`.

## Step 1 — Re-sync the plugin fixture copies

1. Copy bob-cli `tests/fixtures/relation_successor/vectors.json` byte-for-byte over both
   bob-plugins copies listed above. Check that all three sha256 sums match.
2. In bob-plugins, run:
   - `TZ=America/New_York node --test scripts/test-task-status-cycler-relation-successors.cjs scripts/test-navigation-relation-successor-vectors.cjs scripts/test-task-status-cycler-parked-successors.cjs`
     (all pass; the byte-identity test now passes);
   - `npm run build:check`.
3. Commit in bob-plugins, for example:
   `test(relation-successors): re-sync vectors after capture unlink moved SA31/SA32`.
   Reference bead `bob-cli-fs` the way that repo's commits do.
4. Run `bob plugins sync` from the opened checkout, as the repo rule requires. The
   fixtures are not deployed. If sync refuses because other checkouts hold uncommitted
   vault deploys, do not pass `-F`; record the refusal in the close note.

## Step 2 — Cut the relation-enabled overhead

### 2a. Reproduce the measurement first (athena)

1. Build a release binary: `cargo build --release --bin bob`. The target dir may be
   redirected, so find it with `cargo metadata --format-version 1 --no-deps` →
   `target_directory`. Copy the binary into a scratch dir.
2. Create a scratch vault from the Markdown only:
   `rsync -a --prune-empty-dirs --exclude='.git/' --include='*/' --include='*.md' --exclude='*' ~/bob/ "$S/vault/"`
   (about 6,850 notes).
3. Write the fixtures into the scratch vault (never `~/bob`):
   - `$S/vault/capture-measure.md`:
     ```
     - [ ] #task Prepare evaluation prompt [id:: evaluate] ^evaluate
       - 🪢 **LINKS:** starts after [[capture-measure#^implement]]
     - [*] #task Send implementation prompt [id:: implement] ^implement
     - [*] #task Unrelated chore ^chore
     ```
   - `$S/vault/2026/20261102.md`:
     ```
     ## Pomodoros
     - [ ] (**0900-0930**) — BUILD
       - [[capture-measure#^implement]]
     ```
   - `$S/off.yml` is empty. `$S/on.yml` is
     `relations:\n  successor_labels: [starts after]\n`.
4. Create a wrapper `run.sh <off|on> <capture text>` that executes:
   `env BOB_DIR=$S/vault BOB_DAY_FILE=$S/vault/2026/20261102.md BOB_CONFIG_FILE=$S/<cfg>.yml BOB_NOW="2026-11-02 09:30:00" XDG_CONFIG_HOME=… XDG_DATA_HOME=… XDG_STATE_HOME=… XDG_CACHE_HOME=… (all under $S) $S/bob capture -b $S/vault --dry-run --no-clip --no-ref -f json -- "<text>"`.
   Use a bash script file: zsh aliases break inline function definitions.
5. Smoke the fixtures first:
   - `run.sh on 'x*'` has `"event":"parked"` and
     `Starts after “Send implementation prompt”`;
   - `run.sh on '!capture-measure:implement'` has `"event":"done"`;
   - `run.sh off 'x*'` has no `Starts after`.
6. Time the cases with `hyperfine -N -w 5 -r 30 --export-json …`, both configs:
   - `hello`;
   - `x`;
   - `x*`;
   - `!capture-measure:chore` (call it `!noid`);
   - `!capture-measure:implement`.

   Read the medians from `results[].measurements[].time_wall_clock.value`; this
   hyperfine's JSON has no top-level `median`. Confirm the table in Context reproduces
   within noise.

### 2b. Attribute before changing anything

Add temporary `std::time::Instant` timers, printed to stderr behind a local env check
and removed before commit. Cover each stage of the relation path:

- the catalog walk;
- `SharedNoteBytes::build` / `read_snapshot_parallel`
  (`src/native/capture/dependencies.rs`), split into worker time and join/merge time;
- `collect_relation_sources` (`src/native/capture/relation_sources.rs`): overlay,
  `overlaid_matching`, `source_targets_for_events`,
  `dependency_snapshot.overlaid(staged.clone())`, `current_contents_from_snapshot`, and
  `load_target_note`;
- engine classify/plan (`src/native/task_complete/relation_successors/`,
  `src/native/task_complete/successors.rs`);
- the parked path in `src/native/capture/pomodoro_close.rs`;
- the done path in `src/native/capture/task_complete.rs`;
- output and copy building.

Compare the same stages with relations off.

Leads to check. They are hypotheses, not findings:

- When any source matches, `collect_relation_sources` passes the **whole** dependency
  snapshot (`dependency_snapshot.overlaid(staged.clone())`) as `dependency_notes`. The
  engine may then parse every `dependsOn`/`id::` note. The dependency proof only needs
  the notes that can satisfy the matched sources' own prerequisites, project-blocking,
  and recovery inputs (SA8, SA11, SA14, SA15).
- In the `!` both-passes case, CPU is equal but wall time is +21 ms. Look for
  main-thread work after the join: snapshot text clones, `staged.clone()`, per-note
  `String` copies, re-running `parse_tasks`/`logical_lines` on large inputs, a second
  `NoteIndex`/catalog build, or worker chunking that is unbalanced by size once
  relations retain extra text.
- In the relation-only cases (`x*`, `!noid`), the pass costs 28–40 ms against 17 ms for
  the dependents pass. Check whether `relations_enabled` makes workers classify and
  retain texts for both predicates when only one consumer will read them. Also check
  whether a parked-only batch forces the dependency side.

### 2c. Fix, then re-measure

Remove the attributed waste without changing results. Add or extend unit tests that pin
the narrowed reads: matched-source dependency proof, plus no full-snapshot clone when
not needed. The SA vectors must still pass; that includes
`src/native/task_complete/tests/relation_successor_vectors.rs` and the capture CLI SA
tests.

**Targets.** Athena, same binary, on − off medians from the same hyperfine run. Let D =
`!implement` off − `!noid` off, measured in that same run; it is about 17 ms today.

| #       | Case                                               | Must meet                                                 |
| ------- | -------------------------------------------------- | --------------------------------------------------------- |
| A       | `!capture-measure:implement` (shared pass)         | Δ ≤ +5 ms                                                 |
| B       | `!noid` (relation-only pass, zero matched sources) | Δ ≤ D + 5 ms                                              |
| C       | `x*` park (relation-only pass, one matched source) | Δ ≤ D + 5 ms                                              |
| —       | `hello`, plain `x`                                 | unchanged within noise; zero added reads (existing tests) |
| stretch | `x*` with relations on                             | ≤ 70 ms median on athena                                  |

A must be met. If B or C cannot be met after the attributed non-pass work is gone, the
timers must show that the residual above D is the design-accepted byte pass itself
(catalog walk plus parallel read/classify). Record that attribution. No other residual
stage may exceed 5 ms.

### 2d. Record in §12.7

Add a dated "Re-measured" block to `docs/task-dependencies.md` §12.7, after the existing
apollo blocks. Do not rewrite history. Include:

- The athena same-binary on/off table, before and after, with the method (fixtures, env,
  hyperfine settings, commit SHAs, binary sha256).
- An apollo re-measure using the existing §12.7 method: `ssh apollo` works from athena,
  and `cp -a ~/bob` goes into a `mktemp -d` scratch there. Compare the pre-change master
  binary with the candidate, 25 interleaved runs after 5 warmups, the same five cases
  plus `!` with dependents. If apollo is unreachable, say so and record athena only.
- Honest budget notes: which targets above are met, and the remaining gap to the ≤ 70 ms
  `x*`/`!` budget. Keep the `!` baseline gap attributed to `bob-cli-gc`.

Run `just docs-check` after editing docs.

## Step 3 — Verify

- `just fmt`
- `cargo test --lib task_complete::` and `cargo test --lib capture::relation_sources`
  (or the module paths that hold the touched tests).
- `just check` (must exit 0).
- `just docs-check` (must pass).
- Re-run the step 2a smokes on the final binary.
- Remove the temporary timers; `git diff` must show no timer code.

## Step 4 — Close the epic (same turn; final step)

This tale has no land agent of its own, so this step is the landing. Do not wait for
this tale's own commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-fs`. At the audit there were no entries. If any
   appear, resolve each one: wire it up, privatize it, add a non-test pragma, or delete
   it per the Symvision epic-whitelist policy. Re-key a Justfile line only to a
   still-open bead that genuinely needs it.
2. Close the epic normally, never with `--force`:
   `sase bead close bob-cli-fs --note "<verification>"`. The note must state:
   - the land audit scope (all 11 phases and the nested epics fs.2.1/fs.4.1/fs.8.1/
     fs.10.1, DECISIONS honored, source and docs verified, and post-start drift in
     bob-cli and bob-plugins needing no relation wiring; see the audit note on the
     epic);
   - the follow-up triage dispositions: fs.6 #1, fs.7 #1, and fs.9 #1 declined as
     duplicates of open bob-cli-et/f7/eh/ff/f8, already corroborated by
     bob-cli-fs.8.1.land; grandchild proposals dispositioned by their landers; epic note
     #2 resolved by `8c140e43`;
   - the fixture re-sync (commit SHA and the three matching hashes, plus the
     `bob plugins sync` outcome);
   - the overhead fix: attribution, before/after athena and apollo numbers, and which
     targets were met;
   - the gates run and their results.
3. Run `just symvision` if the recipe exists. At the audit this repo had none; say so in
   the close note if that is still true.
4. Set `status: done` in the epic's plan file frontmatter: the PLAN path printed by
   `sase bead read bob-cli-fs -r "<why>"`, which is
   `plan:202611/starts_after_relation.md` in the plans sidecar. Change only
   `status: wip` → `status: done`.
5. `bob-cli-fs` has no `parent_bead`, so the landing ends here.
