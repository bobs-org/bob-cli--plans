---
tier: tale
title: Exclude title-prefixed marker mirrors from Highlights annotations
goal:
  Color-grouped sidecars and legacy rendered notes exclude marker mirrors while
  preserving genuine annotations and their IDs.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.2
status: done
---

- **PARENT:**
  [202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.2.md)
- **COMMITS:**
  - [bff7bc7](https://github.com/bobs-org/bob-cli/commit/bff7bc724b5ac854c554ed916a13f5802dede634)
    — fix(highlights): exclude color-grouped setext title-prefixed marker mirrors

# Exclude title-prefixed marker mirrors from Highlights annotations

Complete assigned phase `bob-cli-an.6.6.6.5.2`, addressing the still-open color-grouped
variant of `bob-cli-4r`. One coding agent can finish this: the remaining defect is in a
shared text classifier, with existing rendering, library filtering, and silent-cleanup
consumers. This phase has no notes from an earlier failed implementation attempt, so a
child epic is unnecessary.

## Verified baseline and scope

Read these through their audited commands before implementation:

- `sase bead read bob-cli-an.6.6.6.5.2 bob-cli-4r -r "Need phase scope, finalized decisions, and reopened color-grouped reproduction"`.
- `sase artifact read plan:202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md "Need the parent marker-mirror phase contract"`.

The parent plan's original setext-title-plus-Page-1 explanation is already implemented.
Current `sidecar.rs` drops preamble before a page heading; `sidecar_render.rs` and
`annotation_tasks.rs` skip every annotation matching `is_sidecar_marker_mirror`, without
a one-shot flag. Existing tests include
`setext_preamble_yields_no_annotations_and_keeps_genuine_ids_stable`,
`leaked_mirror_and_preamble_blocks_drop_silently_while_genuine_tombstones`, and
`highlights_ref_sync_drops_leaked_mirror_without_tombstone`. Preserve them.

The task was closed on October 7 and reopened on October 15 for **color-grouped exports
with no page headings**. Their shape is:

```md
# 「remote_sase_agents_consolidated」

#### Yellow

##### 2026-06-06:

- status: wip
- parent: sase_ref
- agent: research.final-11

---

#### Yellow

##### 2026-06-06:

> A genuine highlight.

Comment: A genuine comment.
```

With no page heading, `parse_sidecar_markdown` retains the title and underline in the
first chunk. `parse_sidecar_chunk` removes the color/date headings but keeps that prefix
before the marker list. `region::is_marker_mirror_text` passes the entire text to strict
`parse_marker`, which rejects the title. Consequently the marker renders as a note and
receives an `h-` ID. `beautify_annotation_text` joins the title and underline onto one
line in the rendered callout. The region parser rejects that form too, so library reads
expose it and sync cannot silently remove its ID.

Read-only installed-bob queries on October 24, 2026 at 08:16 EDT still reproduce:

| Reference                                             | Phantom ID       | Current library counts                       |
| ----------------------------------------------------- | ---------------- | -------------------------------------------- |
| `ref/chat/remote_sase_agents_consolidated.md`         | `h-0803707b4f20` | 2 annotations, 2 comments; one is the marker |
| `ref/chat/apollo_remote_agents_workspace_topology.md` | `h-d003d0a4b3ab` | 1 annotation, 1 comment; it is the marker    |

Both have `page_label: null` and `excluded.marker_mirrors: 0`. The first note also has a
genuine highlight with ID `h-e2d9e4fa6422` that must survive unchanged. The stored notes
report the October 7 sync snapshot. A complete `bob ref list -R all -n 10000 -f json`
query returned 1,121 listed references, 86 with positive stored highlight counts.
Showing the 86 annotated references and checking standalone comments for both marker
fields found exactly these two unexcluded candidates and zero already-recognized mirror
notes. This is the library API's scope, not a new raw-file claim of the historical
113/115 count; the envelope reported 1,123 indexed paths before normal library
exclusions.

The source baseline inspected was `adb9c324`; prerequisite `215d93c7` has landed and
keeps heading-shaped wrapped quote lines. The inherited decision
`highlight_id_on_quote_fix = remint` remains final. This phase must preserve that fix
and its remint tests; it changes no genuine quote text or ID policy.

## Implementation

1. **Lock down the reopened reproduction first.** Add a small color-grouped sidecar
   fixture under `tests/fixtures/highlights_ref/`, using the shape above plus a genuine
   standalone note. Add a failing unit regression through the existing parse/render path
   in `src/native/highlights_ref/tests/sidecar.rs`: the title-prefixed marker must be
   excluded, genuine quote/comment/note text must survive, and page labels must remain
   absent. Keep the existing page-grouped fixture and regression rather than
   reimplementing its already-fixed parser.

2. **Extend the shared marker classifier conservatively.** Prefer a small helper beside
   `is_marker_mirror_text` in `src/native/highlights_ref/region.rs`. Preserve strict
   whole-text marker recognition first. For the additional supported shape, recognize
   only a leading nonempty setext title plus its underline followed by a complete valid
   marker list. Support both the raw two-line title/underline and the already-rendered
   single-line `title =================================` form. Allow separating blank
   lines. Validate the entire remainder with the existing `parse_marker` and its
   required `status`/`parent` rules. Do not change PDF marker parsing to ignore
   arbitrary prose, search for an arbitrary marker suffix, or classify all standalone
   notes as mirrors. A title prefix followed by extra prose, malformed marker items, or
   missing required fields must remain visible.

   The existing shared consumers should then handle the new shape naturally: renderer
   exclusion, annotation-task exclusion, ID assignment, region classification, reference
   counts/show output, doctor diagnostics, and `silently_dropped_block_ids`. Keep color
   groups page-less; do not manufacture page labels or broadly discard no-page-heading
   sidecars. A general color parser or quote-wrap change is outside this fix. Update
   nearby comments to describe the newly recognized shape accurately.

3. **Verify old-note recovery and identity stability.** Add region/render tests with the
   previously rendered title-on-one-line callout and a planted phantom ID. Reading must
   mark it as a mirror. Re-rendering must remove both the block and ID without putting
   the phantom in `Removed highlights`. Real highlight and standalone-note IDs remain
   stable, including when the marker is present before them or appears more than once. A
   genuinely deleted highlight must still tombstone normally. Test that ordinary
   no-page-heading prose, a real titled note, and an ordinary marker-looking discussion
   with extra prose are retained. Preserve strict PDF marker validation. Exercise
   annotation task candidates with an otherwise valid mirrored field containing
   task-like text so filtering cannot accidentally create a task from the mirror.

4. **Exercise the public paths in isolated vaults.** Extend
   `tests/cli/highlights/sync_tasks.rs` with a color-grouped counterpart of the existing
   leak-cleanup regression. Use its synthetic PDF/`BOB_DIR` helpers, plant an old leaked
   callout and genuine IDs, assert dry-run reports an update without writing, then
   perform a fixture-only sync. The mirror's phantom ID must disappear without a
   tombstone, real IDs/content must survive, and a second sync must be stable. Add
   coverage in `tests/cli/ref_library/show.rs` for reads of an unsynced old leaked
   callout: annotations and counts exclude it and `excluded.marker_mirrors` increases;
   genuine annotations remain. Include the existing doctor diagnostic path if necessary
   to verify the shared predicate rather than adding new production filtering to each
   consumer.

5. **Document the bounded behavior.** Read `docs.md` through `/sase_memory_read`, then
   update the sidecar marker-exclusion contract in `docs/highlights-ref-sync.md`:
   color-grouped marker notes may include a setext title prefix, both newly rendered and
   legacy persisted mirrors are excluded, and the next normal scan/sync removes phantom
   IDs without tombstones. General sidecars without page headings still retain ordinary
   notes. No new CLI options, memory edits, version changes, or changelog changes are
   needed.

## Validation and live evidence

- Run the focused new regression before and after the fix, then
  `cargo test --lib native::highlights_ref`, `cargo test --test cli highlights::`, and
  `cargo test --test cli ref_library::`. Reuse the established fixture helpers.
- Run `just docs-check` and `just check` once on the integrated change. Use
  `/sase_monitor` for long commands; wait for the monitor handoff itself to exit and
  resume from its recorded result. Do not end a turn with an unmanaged background
  command still running.
- Using `/bob_ref`, repeat the read-only two-note queries and scoped library census with
  a freshly built binary from the implementation tree, not a stale `target/debug/bob`.
  Before live sync, the new read path should exclude one phantom from each of these
  notes while their stored `highlights_count` may still reflect the old snapshot. Record
  build/source provenance and the query timestamp; report the counts as library-API
  counts. Capture the current census before changing the implementation if these numbers
  have moved.
- Do not edit the live vault, run a live scan/sync, or install/release a binary. Prove
  persisted cleanup in the isolated-vault CLI regression and document the normal next
  scan/sync as the live recovery path for the parent lander. A live read must not be
  represented as a completed persistent rewrite.
- Live source files need not be accessed directly: the reopened bead gives the sidecar
  shape and `bob ref show` reproduces the old persisted form. The planner's attempted
  `sase repo open bob` failed because no such repository is configured; do not bypass
  `/sase_repo` with direct vault repository reads. Use read-only Bob interfaces for the
  census. Any other repository access requires `/sase_repo`.

## Completion and ownership

Record the baseline, actual fix, tests, phantom-ID behavior, and scoped live counts on
`bob-cli-4r` and assigned phase `bob-cli-an.6.6.6.5.2` with `sase bead note`. Leave task
`bob-cli-4r` open for the parent epic's lander. Create no task beads. Any discovered
out-of-scope work belongs in a `PROPOSED FOLLOW-UP: <summary — evidence>` note on the
assigned phase.

If a check failure reproduces identically on the clean base, record that comparison and
any known tracking bead in a `PROPOSED FOLLOW-UP:` note; it does not keep this phase
open. Do not change unrelated code to make the gate pass.

Immediately before closing, run `sase bead epic-symbols bob-cli-an.6.6.6.5.2`; the
planning audit found no entries, but recheck at completion. Resolve any newly found
entries or re-key them to a still-open owner. Then close **only** `bob-cli-an.6.6.6.5.2`
with a note stating what was verified. Never hand-set its status or close the task,
parent epic, or any ancestor plan bead. Complete the required `/sase_final` declaration
when ending the implementation turn normally.
