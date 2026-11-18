---
tier: tale
size: medium
title:
  "Land record walk: acceptance regression tests, single sync walk plan, epic closeout"
goal:
  Every acceptance criterion the record-walk epic (bob-cli-na) promised is pinned by an
  automated test, albums sync plans each album's walk once, and epic bob-cli-na is
  closed with its plan file marked done.
proposed_by: bbugyi200.apollo.bob-cli-na.land
proposed_with: claude/opus
bead: bob-cli-na
create_time: 2026-11-18 17:18:44
status: wip
---

- **PARENT:**
  [202611/record_walk_ring.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_walk_ring.md)
- **BEAD:**
  [bob-cli-na](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-na/README.md)

# Land record walk: acceptance regression tests and epic closeout

## Context

Epic `bob-cli-na` ("Record walk: ‹ › links on every record, N of M painted live") landed
all eight phases. The bob-cli commits are `41b1dab3`, `da68e6bf`, `c02d405d`,
`3b5490c1`, and `b96bbe7d`. The bob-plugins commits are `a4710c4`, `281ee8a`, and
`e7d82af`. The land agent verified the work against the epic plan. Read the plan with
`sase bead read bob-cli-na -r "<why>"` and the PLAN path it prints. On master
`b96bbe7d`, `just check` passes: fmt, clippy, 7379 library tests, and 3814 CLI tests.

The lander already made one cleanup in its own landing commit, so do not redo it. It
removed the phase-scaffolding `#[allow(dead_code)]` on `mod walk` in
`src/native/albums/mod.rs`. It also deleted the dead test-only wrapper
`insert_walk_line_for_test` in `src/native/albums/walk.rs`, so tests now call
`insert_walk_line` directly.

What remains is epic work. The plan's acceptance lists asked for automated tests that
the phases left to manual checks: the walk-ship phase's walkthrough transcript, and a
manual dry run in the walk-sync phase. Those checks are not regression-protected. Also,
`albums sync` currently plans every album's walk twice. All of this belongs to the epic:
no separate task bead.

### Gaps found (and where coverage already exists)

| Acceptance item (epic plan)                                          | Today                                                                                     |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Sync writes M = 2 / M ≥ 3 ring bytes; second sync is a no-op         | CLI tests cover only M = 1 (`tests/cli/albums_sync.rs`); unit vectors cover line bytes    |
| Hand retitle that reorders repairs ≤ 5 files; hand delete exactly 2  | Manual only (walk-ship transcript)                                                        |
| Damaged line untouched, `album_walk_damaged` warned, exit 0          | `album_walk_damaged` is asserted nowhere                                                  |
| Album held by an unreadable member is skipped; another album repairs | Untested for walks                                                                        |
| Member edited between plan and install refuses sync; nothing written | Untested for walk edits                                                                   |
| Nightly parity (nightly writes walk lines)                           | Nightly test asserts indexes only                                                         |
| Capture: neighbour preimage race refuses the whole capture           | Untested (album-note race exists: `album_edit_after_planning_refuses_and_writes_nothing`) |
| `records show` for a held album: `record.walk` is `null`             | Untested (`record_walk_value` in `src/native/records/show.rs` returns `Value::Null`)      |

## Steps

### 1. Plan each album's walk once in sync

Today `report::project_album_from_folder` (in `src/native/albums/report.rs`) calls
`walk::plan_album_walk(..., WalkScope::All)` to build the report. Then
`sync::plan_indexes` (in `src/native/albums/sync.rs`) calls `plan_album_walk` again on
the same folder with the same disk reader. That reads every record body twice on every
sync, including nightly runs and the plugin watcher's 3 s debounced sync.

- Add a `walk_edits: Vec<FileEdit>` field to `AlbumProjection`. Fill it from the
  projection's existing `walk_plan.edits`.
- In `plan_indexes`, extend `walk_edits` from `projection.walk_edits` and delete the
  second `plan_album_walk` call. Drop any import this leaves unused.
- Leave `verify_current_hosts` alone; its post-write re-plan is intentional.
- Output must stay byte-identical. All existing `albums_sync`, `albums`, and nightly
  tests must pass unchanged.

### 2. Sync acceptance tests (`tests/cli/albums_sync.rs`)

Reuse the file's helpers (`write`, `fixture`, `commit_all`, `run`, `json`, `head`,
`subject`, `changed_paths`, `diff_for`) and its `ALBUM` / `RECORD` constants. Each test
uses its own `TempDir`, with a git vault where it commits. Write records with explicit
`title:` frontmatter so the order is deterministic. Add:

1. **`walk_sync_writes_two_and_three_member_rings`.**
   - Build an `eat` album with two records (titles `Apple`, `Mango`) and run sync.
     Assert the exact canonical lines: both arms of each record name the other, and the
     album label is `↑ Restaurants`:
     `[[eat/mango|‹ Mango]] · [[eat|↑ Restaurants]] · [[eat/mango|Mango ›]]`.
   - Add a third record and sync again. Assert that all three lines form the ring with
     wraps at both ends, and that `counts.walk_lines` equals the number of lines that
     changed.
   - Run sync a third time. Assert `changed == []`, `commit == null`, and
     `sync.post == "skipped"`.
2. **`walk_sync_repairs_only_the_retitle_and_delete_radius`.** Sync and commit a
   five-record album first.
   - Hand-edit one record's `title:` so it moves to a different ring position, commit,
     and sync. Assert `counts.walk_lines <= 5` and that every record outside the old and
     new neighbourhoods is unchanged.
   - Hand-delete one record, commit, and sync. Assert `counts.walk_lines == 2`, and that
     the two changed paths are the deleted record's former prev and next.
3. **`walk_sync_leaves_a_damaged_line_and_warns`.** Give one record a walk line followed
   by extra text on the same line, e.g. `<canonical line> extra` (reason `extra_text`).
   - Run `albums check -f json`. Expect exit 0 and an `album_walk_damaged` warning whose
     message names the count and the first damaged path. Expect
     `walk.state == "damaged"` and `walk.counts.damaged == 1`.
   - Run `albums sync -f json`. Expect exit 0, the damaged file byte-identical, and the
     album's other records repaired.
4. **`walk_sync_holds_an_album_with_an_unreadable_member`.** Use two albums. In one
   album, add a member whose bytes are not valid UTF-8. Reuse the trick from
   `unreadable_member_refuses_while_malformed_member_allows_write` in
   `src/native/capture/tests/album_index.rs`: `b"---\ntitle: \xff\n---\n"`.
   - Run `albums list -f json`. Expect that album's `walk.state == "held"` and
     `walk.order == []`.
   - Run sync. Expect no walk edits for the held album's members, while the healthy
     album's records get their walk lines in the same commit.
5. **Nightly parity.** Extend
   `nightly_continues_after_album_failure_and_publishes_healthy_indexes`, or add a
   sibling nightly test, to assert that `eat/one.md` carries its walk line
   (`[[eat|↑ Restaurants]]`) after `bob nightly`.

### 3. Sync race unit test (`src/native/albums/sync.rs` tests)

Add `walk_member_body_edit_after_planning_refuses_and_writes_nothing` using the module's
`fixture()` / `planned()` helpers:

1. Plan the sync. At this point `eat/one.md` has no walk line, so the plan carries a
   `walk` edit for it.
2. Append a body line to `eat/one.md` on disk. This is a body-only edit;
   frontmatter-only `dependencies` do not catch it.
3. Drive the same guarded install `apply_and_commit` uses: build the txn with
   `planned_txn(&vault, plan)`, then call
   `crate::native::capture::write_staged_files_guarded(&txn.files, || txn.deps.validate(&vault))`.
   It must fail with `CaptureFailureKind::NoteChanged`, naming `eat/one.md`.
4. Assert that `eat.md`, `body.md`, and the edited `eat/one.md` keep their exact on-disk
   bytes.

If `planned_txn` or the writer needs a visibility tweak, keep it `pub(super)` or
test-only; do not widen public API.

### 4. Capture neighbour race (`src/native/capture/tests/album_index.rs`)

Add `walk_neighbour_edit_after_planning_refuses_and_writes_nothing`, modelled line for
line on `album_edit_after_planning_refuses_and_writes_nothing`. Plan the
`Ramen Nagi @eat/` capture; the staged set must include the neighbour
`eat/ninja_sushi.md`, as in `album_create_reads_only_its_slice_and_stages_one_edit`.
Append foreign bytes to `eat/ninja_sushi.md`, then call `commit_capture_batch`. Assert
all of:

- the error is `NoteChanged` and names the neighbour;
- `eat/ramen_nagi.md` does not exist;
- the neighbour keeps the foreign bytes;
- `eat.md` is unchanged;
- `no_capture_litter` holds.

### 5. `records show` for a held album (`tests/cli/records.rs`)

Next to `walk_show_reports_ring_positions`, add a case using the `walk_vault()` /
`write_walk_record` helpers, with one invalid-UTF-8 member added to the folder. Assert
that `records show restaurant <readable stem> -f json` succeeds with
`record.walk == null`, and that the human card prints no `N of M` walk line. If
`records show` turns out to refuse outright because of an unreadable sibling, stop. Do
not assert the refusal. Report it to Bryan, because the epic contract says `record.walk`
is `null` when held.

### 6. Verify

- Run the focused tests: `cargo test --test cli albums_sync`,
  `cargo test --test cli records`, `cargo test --test cli nightly`,
  `cargo test --lib albums`, and `cargo test --lib capture`.
- Run `just check`. Do not run `just check-full`.
- No docs, help text, or JSON contract changes are expected. If an assertion shows real
  behaviour that differs from `docs/records.md` "Record walk", fix the code to match the
  documented contract (or stop and report). Never weaken the test to pass.

### 7. Close epic bob-cli-na (final step)

1. Run `sase bead epic-symbols bob-cli-na`. At landing time it listed none. For any
   entry that appears, resolve the symbol (wire it up, privatize it, or delete it), or
   re-key it only to a still-open bead that needs it.
2. Close the epic. Record the outcome; do not use `--force`:

   ```bash
   sase bead close bob-cli-na --note "<verification: what the lander verified (all 8 phases' code vs plan, decisions honored: bare footer, glossary-only memory edit, no decision record; just check green on b96bbe7d; dead-code scaffolding removed) plus this tale's new tests, the single-plan sync refactor, and the just check result>"
   ```

3. Run `just symvision` if the justfile defines it. At landing time it did not, so
   record that fact in the close note instead.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read bob-cli-na`, `202611/record_walk_ring.md` in the plans repo.
5. `bob-cli-na` has no `parent_bead`, so nothing further cascades.

## Out of scope

- Any change to walk bytes, order, states, JSON fields, or plugin behaviour.
- The bob-plugins timing flakes (tracked by `bob-cli-km`, `bob-cli-io`, and
  `bob-cli-ki`), and the docs-check mount collision (`bob-cli-it`).
