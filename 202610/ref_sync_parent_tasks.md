---
tier: tale
title: Scan writes reference reading tasks into parent notes
goal:
  Complete bob-cli-5y.7 with guarded v2 births, located-task sync, parent projection,
  follow-ups, verification, and phase closure.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5y.7
bead: bob-cli-5y.7
create_time: 2026-10-09 15:06:38
status: wip
---

- **PARENT:**
  [202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_tasks_live_with_parent.md)
- **BEAD:**
  [bob-cli-5y.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5y/bob-cli-5y.7.md)

# Complete bob-cli-5y.7: scan writes reading tasks into parent notes

Implement the assigned `ref-sync-v2` phase of
`plan:202610/ref_tasks_live_with_parent.md`. New reference notes show a live embed of
one ordinary reading task in its owning area/project/inbox note. Existing v2 refs sync
against that located task, including archived tasks; v1 refs retain their current
behavior. Finish and close only `bob-cli-5y.7`.

## Scope, authority, and starting state

- Read `sase bead read bob-cli-5y.7 -r "Need the approved scope and current notes"` and
  the epic design with `sase artifact read` before implementation. The planning read
  found no earlier worker notes or remaining-work report. The dependent locator phase
  `bob-cli-5y.5` is closed, and its implementation is present.
- This is one bounded implementation in bob-cli: the locator, strict parent resolver,
  capture insertion function, and preimage-checked writer already exist. Keep the
  planning/execution changes cohesive in one coding turn, using small helper modules.
- The epic's decisions are final: `cancel_dropped_wrapper_refs = no`,
  `memory_ref_parent_decision = no`, `memory_glossary_ref_terms = no`. Edit no memory
  files and cancel no wrappers. Record the two skipped memory changes as
  `PROPOSED FOLLOW-UP:` notes on this phase if they have not already been recorded.
- Use temporary fixture vaults exclusively. Do not mutate the live vault, install bob,
  change plugins or the Mac frontend, add CLI options/subcommands, implement migration,
  or close any ancestor. Preserve JSON schema versions and existing fields.
- The phase is already assigned and `in_progress`; never set its status manually. Record
  out-of-scope discoveries with `sase bead note bob-cli-5y.7 'PROPOSED FOLLOW-UP: ...'`;
  create no beads.

## Available interfaces and files

`src/native/parent_notes.rs` exports `resolve_parent(bob_dir, input)` and returns the
canonical route, kind, label, and alias match. `mac_inbox` is a pinned capture target
even in a minimal vault. Follow capture's existing missing-inbox behavior when creating
it; never manufacture an arbitrary invalid parent note.

`src/native/ref_tasks/` exports `RefTaskIndex::build(bob_dir, ref_dir)`,
`candidates(ref_path)`, `select(ref_path, v1_hits)`, `find_trackers`, and
`find_managed_embed`. Index keys and `LocatedRefTask.path` are vault-relative paths with
`.md`; located tasks carry exact original `line`, absolute line index, mark, optional
block ID, archived flag, and optional residence. Path-qualified candidates are indexed
even before their reference note exists, enabling orphan adoption. Selection already
diagnoses duplicate open tasks and open tasks in `done/`.

`src/native/capture/` re-exports `insert_task_line`, `write_staged_files`,
`StagedTextFile`, and `Placement`. The writer checks disk preimages before staging,
after staging, and before replacement. Use these functions rather than a parallel task
insertion implementation.

The main integration points are `highlights_ref/sync.rs`, `model.rs`, `note.rs`,
`projection.rs`, `annotation_tasks.rs`, `guard.rs`, `region.rs`, `audio.rs`,
`report.rs`, and `scan_json.rs`. Human scan and JSON scan have separate entrypoints that
both call `plan_pdfs`; update both. Existing integration tests are registered in
`tests/cli/highlights/mod.rs`; unit tests in `highlights_ref/tests/mod.rs` and
`ref_tasks/tests.rs` provide narrow helper tests. Preserve existing read-side seams.

## Implementation

### 1. Shared task rendering and guarded insertion

Extend `ref_tasks/line.rs` with reusable `render_ref_task_line`,
`allocate_ref_block_id`, and `managed_embed_line`, exposing them through
`ref_tasks/mod.rs`. Preserve the candidate parsing functions already in that file. Add
`ref_tasks/insert.rs` with the reusable
`insert_ref_task(bob_dir, destination, line, children)` API; return the final block ID
and placement (and enough location information to render its embed). Keep these names
because `migrate-tasks` depends on them. Document concrete argument and result types in
the phase completion note so the migration worker can reuse them.

- Render exactly
  `- [m] #task #ref [[ref/<type>/<stem>|<title>]] [created::YYYY-MM-DD] ^ref-<slug>` as
  one physical line. The link targets the ref note, never the PDF; no `#hide`, verb, or
  `[fresh::]`. Honor configured ref-dir paths and `BOB_NOW` for the invocation's local
  creation date.
- Sanitize the title alias by removing `[`, `]`, `|`, `#`, `^`, and backticks; collapse
  whitespace; truncate at a word boundary to at most 100 characters plus `…`. Title
  comes from frontmatter title, then H1, then humanized note stem.
- Slug the note stem to lowercase ASCII alphanumerics separated by `-`, trim it, and
  shorten at a separator boundary to keep the full ID at most 44 characters. Empty slugs
  yield `ref-reading`. Collisions append `-2`, `-3`, etc., preserving the length bound.
  Check real block IDs in the fresh destination, its `done_tasks` archive, and IDs
  reserved during this run. Identity never depends on the ID.
- Generalize close-stamp placement to any trailing valid `^id`, preserving v1 output.
  Born-read/abandoned tasks carry `[completion:: YYYY-MM-DD]` or
  `[cancelled:: YYYY-MM-DD]` immediately before the ID. Preserve existing stamps.
- The inserter rereads destination and archive bytes and allocates the final ID at
  execution, combines the reading line and optional child bullets, then uses
  `insert_task_line` and `write_staged_files`. Use bounded retries only for an actual
  preimage mismatch, rebuilding from the latest bytes and reallocating. Propagate
  permission/IO errors. Preserve capture's Tasks-section placement, no-section behavior,
  child placement, and line endings.
- Dry-run planning reserves preview IDs deterministically without any writes; execution
  must render the embed and reports from the actual allocated ID.

### 2. Locate once and model v1/v2/birth/reopen explicitly

Build one immutable `RefTaskIndex` per `sync_pdf` or scan invocation, after intake and
before parallel PDF planning; share it across workers. The JSON scan entrypoint must use
the same index-backed planning. Keep a small explicit plan for reading-task actions
(none/adopt/insert/edit/reopen) instead of mixing external task bytes into the ref
note's own body parser.

- A missing reference note is born v2. An existing note is v2 when the locator finds its
  task outside that note or its body has the managed embed. Existing in-note `^ref`
  trackers and trackerless legacy notes stay on the v1 branch.
- For birth, first adopt a uniquely located orphan; otherwise resolve the marker's
  parent hint. An invalid/noncandidate/ambiguous hint routes to `mac_inbox`, with child
  text
  `⚠️ parent '<hint>' is not an open area or project · refile me with Ctrl+Shift+M`. Use
  the canonical alias-resolved route for a valid parent.
- Existing v2 notes with missing tasks never get a replacement task automatically.
  Surface `open_ref_without_task` with restore-from-git/set-abandoned guidance.
  Duplicate open candidates refuse status/parent writes; open archive candidates are
  diagnostic and never mutated. Do not guess an ambiguous location or parent.
- Feed the located mark into the existing lifecycle mapping, `[?]` overlay policy,
  normalization, and task-versus-marker/frontmatter conflict handling. For v2,
  distinguish an unchanged mark agreeing with the stored status base from a new task
  edit, so an explicit marker/frontmatter change can drive a line edit rather than being
  mistaken for a competing task gesture. Conflicting changed inputs still fail, and v1's
  existing signal algorithm remains unchanged.
- A closed archived task continues to supply terminal state. An explicit reopen signal
  inserts a fresh open task into the archive's source parent, retaining the old ID when
  free; otherwise allocate a unique ID. Never edit `done/`. If the source parent is no
  longer an open candidate, use the inbox and warning child. Keep the archived terminal
  mark from overriding the deliberate reopen.

### 3. Project residence independently of PDF metadata

For v2 notes remove `parent` from the marker/frontmatter sync projection, hashes, and
stored base. Render frontmatter `parent: "[[<residence>]]"` separately as a
command-managed projection; do not add it back into the sync snapshot by accident.
Normalize an existing base/hash consistently when dropping parent so a moved or migrated
note does not produce a false two-sided conflict. Preserve the PDF marker's parent birth
hint during normal scans and retain required-marker validation.

Only `--write-pdf`/`--write-pdfs` refresh a stale marker parent to residence. Keep that
full marker rendering separate from the parent-free projection used for hashes. A
residence-only move must heal the ref note without requiring PDF write opt-in;
task-driven lifecycle changes retain today's pending status/write-opt-in behavior. Keep
unrelated projection conflicts, `--prefer`, created provenance, and v1 hashes and output
unchanged.

Render/heal exactly one `![[<task-file-without-md>#^<id>]]` one blank line below the H1,
in the old tracker's position. This includes archived paths such as `done/sase_done`.
Recognize it as anatomy in `region.rs`, and anchor companion audio after it in
`audio.rs`. Preserve authored text, manual sections, highlight region, assets,
tombstones, and legacy rendering. Existing v2 notes without a located task must not
acquire an invented embed destination.

### 4. Execute sequentially without losing parent edits

Keep parallel planning and deterministic PDF order. Execute each PDF's destination task
writes before its opt-in marker write and finally its ref-note write. Validate the
ref-note/PDF and task-edit preconditions before starting writes for that PDF. The final
note rendering must use actual insertion results and final metadata.

- A planned cross-file mark change rereads the file immediately before writing. Replace
  the exact original at its planned index if still equal; otherwise replace the unique
  exact equal line elsewhere. If absent or nonunique, fail this PDF with
  `reading task changed during sync; rerun` before writing its marker/ref note. Preserve
  every other byte, original line ending, block ID, and existing dates. Unrelated
  additions to the parent are allowed; a changed reading line is not.
- Destination notes use preimage checks, with no git-dirty guard. Keep v1 routed
  annotation behavior intact. For v2 annotation groups, store insertion intentions and
  reread/rebase at execution rather than replaying stale whole-file snapshots. Multiple
  births, edits, and follow-ups into one parent must retain every task.
- Retain the reference-note git guard. For v2 allow a tracked reference note whose body
  differs from HEAD only by the managed embed (including insertion, deletion, or
  repointing), plus permitted frontmatter edits; unrelated body edits still fail. PDFs
  and assets retain their existing guards.
- Parent-first write ordering intentionally needs no birth journal: if a later ref write
  fails, the next invocation locates and adopts the already-written line. Preserve
  partial scan failures and continuation to subsequent valid PDFs.

### 5. Annotation follow-ups, output, and documentation

Keep intake eligibility before/after the closing signal and processed-ID deduplication.
Default v2 annotation tasks to the live task's residence, else `mac_inbox`; preserve
explicit `@name` route behavior and v1 ReferenceNote defaults. Write v2 back-links with
the full ref path `[[ref/<type>/<stem>#^h-…|🔖]]`. Use the shared insertion path for v2
destination writes, including notes without a Tasks section, and preserve deduplication
on repeat runs and after archive moves.

Report actual reading-task counts separately from annotation counts:
`📖 N reading tasks created · sase.md (1) · bob.md (1)` and `↻ N reading tasks updated`.
When any open v1 refs remain, report `N open v1 ref tasks · run bob ref migrate-tasks`.
Verbose/dry-run/single-PDF sync name destinations, block IDs, adoption, and planned
edits. Keep concise no-op output and JSON stdout clean; preserve schema versions and
existing JSON contracts, adding only optional task details if needed.

Update `docs/highlights-ref-sync.md` for birth/adoption, generated v2 note anatomy,
located reading-task status, archive reopen, v2 parent projection, annotation Tasks
placement, PDF opt-in, and dirty-note rules. Clearly retain documented v1 behavior.

## Verification and completion

Use fixed `BOB_NOW` subprocess fixtures. Add focused unit/integration tests for:

1. Render sanitization/truncation/ASCII slug/empty slug/length and numeric collisions;
   archive and reserved-ID collisions; closed-at-birth stamps; CRLF preservation.
2. Birth into an area/project with and without `## Tasks`; aliases; invalid/closed
   parent fallback and warning; a born-read task and embed with no in-note task.
3. Two PDFs born into one note under parallel planning, same stems in different
   categories, and destination/archive ID collisions: all tasks and embeds survive.
4. Parent-written/ref-missing crash fixture adopts once; repeat sync is a no-op; dry-run
   predicts destinations/IDs and leaves vault/PDF bytes unchanged.
5. Marker/frontmatter-driven mark changes and both close stamps in dirty parent notes;
   user task changes still yield pending read-side state and require PDF opt-in; `[?]`
   remains the neutral overlay; conflicting status inputs refuse.
6. Execution-time race fixtures: insert unrelated lines to move the expected index
   (unique original still updates); alter/delete/duplicate the expected line (refuse
   with no marker/ref write); preimage retries preserve unrelated destination edits.
7. Archived terminal task sync leaves archive bytes untouched; reopen creates one live
   task in the source parent/inbox, retains or reallocates its ID correctly, and points
   the embed at it. Open archive and multiple-open diagnostics are safe.
8. Ref moved between parents updates frontmatter/embed without touching the marker or
   triggering write-opt-in errors; explicit PDF opt-in refreshes its parent hint.
9. Deleted/stale embed healing and git-guard acceptance; unrelated body edits refuse;
   region own-notes omit the managed embed; companion audio follows it.
10. V2 follow-up defaults, explicit routes, full-path back-links, close-run intake,
    repeat deduplication, and mixed births/follow-ups in one destination. Missing v2
    task does not recreate it. Stable v1 sync preserves the note bytes and still
    supports its original task/annotation/status/dirty-guard contracts.
11. Human/single-PDF/verbose/dry-run reports and JSON scan follow the same behavior;
    sequential and parallel output stay deterministic with unchanged JSON versions.

Update existing tests that assumed every newly born ref contains `#hide ^ref` or
receives default annotation tasks locally. Seed intended area/project/inbox fixtures and
assert v2 placement; keep explicit v1 fixtures to protect frozen-note compatibility. Do
not weaken error/guard/status expectations or remove no-op/deduplication tests.

Run focused Rust tests while implementing, then `just check` (format, clippy, all test
binaries). Use `/sase_monitor` for long verification commands, preserving the task state
and exact next steps in the monitor continuation. If a failure reproduces identically on
the clean base tree, record `PROPOSED FOLLOW-UP:` with reproduction evidence and any
already-tracking task ID; it does not keep this phase open.

Before closing, run `sase bead epic-symbols bob-cli-5y.7`. Planning found no entries,
but check again after edits; resolve leftovers or re-key them to a still-open later
phase or the parent. Record the exported insertion API and verified behavior on the
phase, then close only it with
`sase bead close bob-cli-5y.7 --note "<implementation and checks actually verified>"`.
The parent and any plan ancestor remain open for their land agent. Use `/sase_final` as
the final action before a normal provider response, with no manual git commit unless
separately authorized.
