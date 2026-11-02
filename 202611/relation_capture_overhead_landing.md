---
tier: tale
size: medium
title: Trim relation-successor capture overhead and land bob-cli-fs.4.1
goal:
  Relation-enabled capture closes add little beyond the shared byte pass, with unchanged
  contracts and re-measured §12.7 budgets; epic bob-cli-fs.4.1 and its parent phase
  bob-cli-fs.4 are closed.
proposed_by: bbugyi200.athena.bob-cli-fs.4.1.land
proposed_with: claude/opus
bead: bob-cli-fs.4.1
create_time: 2026-11-02 18:13:19
status: wip
---

- **PARENT:**
  [202611/capture_relation_successors.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_relation_successors.md)
- **BEAD:**
  [bob-cli-fs.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.4.1.md)

# Trim relation-successor capture overhead and land bob-cli-fs.4.1

## Why this work remains

Epic **bob-cli-fs.4.1** ("Finish capture relation successors for bob-cli-fs.4") has all
six phases closed. Its land agent checked the delivered behavior and found it complete:
snapshots, Done, parks, batches, presentation, and SA coverage, including the SA20
warning assertion. No phase proposed a follow-up, and no `--epic-symbol` entries exist.
One unresolved problem, caused by the epic, remains. Relation-enabled capture is much
slower than it needs to be. The parent design (`plan:202611/starts_after_relation.md`,
"Performance") asks for one parallel byte pass "generalised to several predicates". The
implementation instead retains every note's raw bytes and rescans them serially, and it
hands far more text to the successor engine than relations need.

Release builds were measured on athena (64 cores) against a scratch copy of the vault
(6846 `.md`, 23.7 MB of Markdown, 872 notes containing `LINKS`, 110 containing
`dependsOn`/`id::`). Each run was a dry run with `-f json`,
`BOB_NOW=2026-11-02 09:30:00`, a config containing only
`relations: successor_labels: [starts after]`, and hyperfine with 5 warmups and 30 runs:

| Case                                                        | Pre-feature `8c7e7103` | Current master | Same master, relations disabled |
| ----------------------------------------------------------- | ---------------------- | -------------- | ------------------------------- |
| `hello`                                                     | 18.7 ms                | 18.6 ms        | n/a                             |
| `x*` parking a live-linked T with one `starts after` source | 79.3 ms                | 143.3 ms       | n/a                             |
| `!` T with `[id::]` and one source                          | 95.8 ms                | 142.7 ms       | 94.8 ms                         |
| `!` T without `[id::]` and no incoming relation             | 81.2 ms                | 137.5 ms       | n/a                             |

Apollo numbers in `docs/task-dependencies.md` §12.7 show the same pattern, at roughly
three times athena's times: `x*` 170.79 → 360.72 ms and `!` 219.87 → 356.28 ms.

Temporary `Instant` timers placed in the current code (athena) attributed the extra cost
as follows:

- `SharedNoteBytes::build` with `retain_raw=true` takes 16 ms, against 11 ms for the
  dependency-only worker-side prefilter. `DependentsSnapshot::from_raw` then rescans
  every retained note serially, once per predicate, and clones the bytes again: 3–4 ms
  for relations and 2 ms for dependencies.
- `collect_relation_sources` spends 12 ms between the snapshots and the source match,
  even when zero sources match. It rebuilds `staged_note_index` even though both callers
  already built one. It also calls `relation_snapshot.overlaid(staged)`, which clones
  all 872 LINKS notes before the per-gesture `#^id` token filter runs. Separately,
  `dependency_snapshot.overlaid(staged.clone())` clones the full dependency set on every
  gesture.
- `plan_successors` takes 18–22 ms with relations, against 3.9 ms without. Callers
  overlay `read.dependency_notes` (about 60 notes) into `note_texts`, even when no
  source matched. Inside the engine, `integrate_relation_offers`
  (`src/native/task_complete/successors.rs`) then passes every held file as
  `source_texts` to `classify_relation_successors`, which parses each one as a potential
  LINKS source (`classify.rs`, around the `for (path, contents) in input.source_texts`
  loop). It also clones the whole 6.8k-path index into `known_paths`, an `archive_paths`
  vector, and the `MapTargetReader` set on every call.

`x*` previously made no full-vault pass. Its new byte pass is inherent to the approved
design, which forbids a persistent or cached index. Everything else listed above is
avoidable. The pre-feature `!` budget miss (apollo 219.87 ms against the 70 ms §12.7
target) is **not** this epic's work. It is filed separately as task **bob-cli-gc**; do
not work on it here.

## Constraints

- Contracts stay unchanged: JSON shape, human output, SA1–SA36 behavior, held and check
  semantics, and selection. This work changes performance only. Keep `schema_version` at
  1. Do not edit `Cargo.toml` versions, `CHANGELOG.md`, any memory note, or generated
     `AGENTS.md`/`CLAUDE.md`. Use a `perf(capture): …` or `fix(capture): …` commit. This
     is not a breaking change.
- Keep the design invariants. The snapshot stays batch-scoped and lazy behind the G0–G3
  gates, with no persistent or 60-second cache and no LINKED FROM. Each catalog path is
  read once per batch when both consumers run, in either request order. Failure sets
  stay per predicate: an unrelated invalid-UTF-8 note must not taint the other snapshot.
  A staged addition, edit, or deletion overlays disk. Notes under `done/` stay excluded,
  but a file named `done.md` does not. Relation completeness stays independent of
  dependency completeness. `hello` and plain `x` keep zero added reads (SA36).
- Target notes for R6 are read only through `CaptureBatchPlanner` (the narrow T′
  closure), as the design specifies ("Rust released checks read only the T′ notes
  involved, through the staged planner").

## Changes

### 1. One parallel pass with both predicates evaluated in workers

In `src/native/capture/dependencies.rs`, replace the `retain_raw` mode
(`SnapshotRead::Raw`, `SharedNoteBytes.raw`, `from_raw`, `cached_text`) with worker-side
classification.

- When relations are configured (non-empty successor labels and `link_unblocked`, the
  current `retain_raw` condition), each worker reads a path once and evaluates both
  `contains_dependency_markers` and `contains_relation_markers` on the bytes. It
  UTF-8-validates once and returns the text for whichever predicates match. Invalid
  UTF-8 goes into the failure set of each predicate the bytes matched. A read error goes
  into both failure sets, as `read_failed` does today. Marker-negative notes are dropped
  in the worker.
- Without relations, keep the existing dependency-only fast path unchanged.
- Build both `DependentsSnapshot`s eagerly from the merged worker output. This preserves
  "either consumer order, one pass". Share text between the two maps where a note
  matches both predicates (for example with `Arc<str>`, or by building the second map
  from the first), so a note that matches both is not stored twice. Any representation
  is fine as long as `overlaid` callers keep their semantics.
- Delete `DependencyContext::cached_note_text` and the raw-byte retention docs. Update
  the struct and field comments to describe the new retained-text lifetime: marker-
  positive text only, for the lifetime of the batch.

### 2. Cheap per-gesture source selection in `collect_relation_sources`

In `src/native/capture/relation_sources.rs`:

- Add a `&NoteIndex` parameter and pass the caller's already-built staged index (both
  `task_complete.rs` and `pomodoro_close.rs` hold `note_index`). Stop rebuilding
  `staged_note_index` inside this function.
- Iterate the relation snapshot by reference. Apply `contains_event_block_token` to the
  borrowed base text first, then to staged overlay text, and clone only notes that pass
  the token filter. A small `DependentsSnapshot` helper that yields borrowed overlaid
  entries, or that clones only those accepted by a predicate, keeps the overlay rules in
  one place.
- Compute `dependency_notes` only when at least one source matched (`source_notes`
  non-empty). With zero matched sources, return an empty `dependency_notes`. The input's
  `dependency_snapshot_complete` flag keeps its current value.
- `load_target_note`: with `cached_note_text` gone, use the text of a matched source or
  of a snapshot entry (relation or dependency) when that note is present. Otherwise call
  `planner.current_contents(&absolute)`, a single narrow read. Failures still go into
  `unavailable_targets`.

### 3. Hand the engine only relation-relevant text

- In both callers (`capture/task_complete.rs` around the `collect_relation_sources`
  call, and `capture/pomodoro_close.rs` in `plan_close_successors`), do not overlay
  `read.dependency_notes` into `note_texts` when they are already present in `snapshot`
  (the `needs_snapshot` path). Overlay only the missing paths, or none.
- In `src/native/task_complete/successors.rs` `integrate_relation_offers`, pass as
  `source_texts` only the held files that can be relation sources: files whose text
  contains `LINKS` and a `#^<id>`/`^<id>` token for some event target. Keep every held
  file available to `MapTargetReader` for target reads. Build `known_paths`,
  `archive_paths`, and the reader's known set without cloning the whole index more than
  once per call. Borrowing, or a single `BTreeSet` reused for both, is fine, as long as
  ambiguity and unavailability detection still use the complete index.
- Re-time `plan_successors` after these changes. If relation-enabled planning with zero
  matched sources still costs more than about 2 ms over the relations-disabled run, find
  the remaining per-call work and remove it. Do not change classification results.

### 4. Tests

Keep every existing suite green, especially the read-accounting tests in
`relation_sources.rs` (`either_consumer_order_walks_and_reads_the_catalog_once`,
`cheap_gate_rejects_without_catalog_or_source_reads`,
`source_snapshot_uses_shared_parser_resolver_and_narrow_target_reads`, and the
staged/failure tests). Also keep `capture/tests/template.rs` (SA36) and
`relation_successor_tests` vectors green. Add or adjust unit tests to prove:

- An invalid-UTF-8 note that matches only `LINKS` leaves the dependency snapshot
  complete, and the reverse also holds, under the new worker classification.
- A marker-negative note is not retained. A target note with neither marker loads
  through the planner exactly once and still reaches R6, and a missing target still
  lands in `unavailable_targets`.
- A Done or parked event with zero matching sources returns empty `source_notes` and
  `dependency_notes`.
- `integrate_relation_offers` produces identical rows, held rows, and checks when
  `source_texts` is narrowed. The shared vectors cover this; add a regression if a
  narrowing edge case appears (for example a source note that is also an event target
  note).

Run `cargo test --lib relation`,
`cargo test --test cli -- capture::relation_successors`,
`capture::relation_presentation`, `capture::keep_moving`, and the pomodoro-close,
successor, and complete-label suites. Then run `just fmt` and `just lint`.

### 5. Re-measure and document

Before remote work, read `docs.md` and `tailnet.md` through `/sase_memory_read`. Repeat
the §12.7 method on **apollo**: a release baseline `8c7e7103` against the new candidate,
an isolated scratch copy of `~/bob` (never the live vault), isolated `BOB_CONFIG_FILE`
and `XDG_*`, the fixed `BOB_NOW`, 5 warmups, and 25 interleaved runs. Time `hello`, `x`,
`x*` park, `!` with one source, `!` with `[dependsOn::]`, and `!` on a T with no
incoming relation. Use `/sase_monitor` for long builds or benchmarks. If apollo is
unreachable, measure on the coder's host with the same method and say so plainly.

Update `docs/task-dependencies.md` §12.7. Add a dated "Re-measured after relation
overhead fixes" block with revision, toolchain, host, vault size, sample count, and
median/p95 per case. Keep the earlier measurement as history. State plainly which
budgets still miss, and that the remaining `!` gap predates the epic (bob-cli-gc).
Expected direction on athena: `!` with relations enabled within a few ms of relations
disabled, and `x*` adding little more than the shared byte pass. Report real numbers
even if they fall short. Never declare a target met without running it. Keep private
task text out of the doc. Run `just docs-check`.

### 6. Gate

Run `just check` (not `just check-full`) and `just docs-check` on the final tree. Smoke
SA1 `x*` and SA4 `!` on a tiny scratch vault in both human and JSON modes. Record exact
commands and results in the close notes below.

### 7. Close out epic bob-cli-fs.4.1 (final step, in this same turn)

1. Run `sase bead epic-symbols bob-cli-fs.4.1`. For every listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it per
   the Symvision epic-whitelist policy. Re-key the Justfile line only to a still-open
   bead that truly needs the exemption. None existed at landing time; recheck anyway.
2. Close the epic: `sase bead close bob-cli-fs.4.1 --note "<verification>"`. The note
   must state that all six phases were verified against the source, that integration
   with concurrent commits was checked (bob-cli-fs.3 authoring `d8b4d4fb`,
   merged-Pomodoro guards `debee563`, and lineup restamping `37b2de64` are compatible),
   that this tale removed the relation overhead (give the before/after numbers), and the
   `just check` and `just docs-check` results. Also record the triage: there were no
   `PROPOSED FOLLOW-UP` entries, and the pre-existing `!` budget miss was filed as
   bob-cli-gc. Never use `--force` merely to make the close succeed.
3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the `EPIC PLAN` path
   printed by `sase bead read bob-cli-fs.4.1 -r "Need the epic plan path for closeout"`
   (`202611/capture_relation_successors.md` in the plans repository).
5. Handle the parent. bob-cli-fs.4.1's `parent_bead` is the **phase** bead
   **bob-cli-fs.4** ("Capture events, snapshot, reporting, and bob-authored group
   copy"). Confirm that this child plan completed that phase's scope: parked events from
   `x*`, done events from every Rust managed Done, the batch-scoped relation-sources
   snapshot, relation offers in JSON and human output, bob-authored copy on every
   `pull_in` group, the extended net pass, and measured budgets. Run
   `sase bead epic-symbols bob-cli-fs.4` and resolve any entries. Then close only that
   phase with
   `sase bead close bob-cli-fs.4 --note "<verified behavior, checks, and measured budgets>"`.
   Leave epic **bob-cli-fs** and every other ancestor to their own land agents. If
   anything blocks the phase close, add a note on bob-cli-fs.4 describing the blocker
   and report it.
