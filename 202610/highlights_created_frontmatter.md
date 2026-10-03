---
tier: tale
title: Set creation datetimes on Highlights reference notes
goal:
  Every new Highlights reference note records its creation datetime in frontmatter and
  preserves it on subsequent syncs.
size: small
proposed_by: bbugyi200.athena.0vr
create_time: 2026-10-03 14:45:48
status: wip
---

# Set creation datetimes on Highlights reference notes

## Objective

Every new reference Markdown note written by `bob highlights sync` or
`bob highlights scan` must contain exactly one `created` frontmatter field recording
when that note is created. This includes the default `ref/` tree, nested reference
categories, configured reference directories, notes with no sidecar, and notes created
after `xlib/` intake. Subsequent syncs must preserve the original timestamp.

## Findings and root cause

- `src/native/highlights_ref/sync.rs::plan_pdf_sync` is the shared note-creation path
  for targeted sync and recursive scan. It reads the existing note and renders its
  frontmatter through `ParsedNote::render_with_projection` in
  `src/native/highlights_ref/model.rs`.
- That renderer writes the selected marker properties, command-managed `type` and
  `ref_type`, and pipeline metadata. Neither it nor `PipelineMetadata` generates
  `created`. The generated body contains the PDF reading task ending in `^ref`.
- `note.rs::pipeline_metadata` generates `highlights_synced_at` only for rendered
  sidecars. That timestamp can change on refresh and is not a note creation time.
  Annotation tasks' `[created::YYYY-MM-DD]` fields also do not meet this requirement.
- `created` is currently an unknown marker key: it can be opted into round-trip sync
  through `highlights_marker_fields`. Automatically adding it to that projection would
  spuriously require PDF write-back or let source metadata change a note's creation
  timestamp.
- `execute_pdf_sync` sometimes reuses the planning-time rendered note and sometimes
  refreshes metadata before writing. `refresh_stable_rendered_note` also rerenders after
  annotation-task planning. Both paths need consistent handling.

## Required behavior

- Use a timezone-bearing datetime with seconds, matching the existing note creation
  convention: `YYYY-MM-DDTHH:mm:ss±ZZZZ`, for example
  `created: 2026-10-03T15:42:18-0400`.
- Derive it from the actual note-writing invocation's clock, honoring the existing
  `BOB_NOW` override. Reuse `bob_env::current_datetime()` and
  `capture_project_note::format_created_timestamp` rather than adding another clock or
  datetime formatter.
- Treat `created` as note-local provenance. It must not participate in the synced
  projection, marker hash, base snapshot, or `highlights_marker_fields`, and setting it
  must not require `--write-pdf` / `--write-pdfs` or change PDF bytes.
- Preserve an existing note's authored `created` field and its raw representation. Do
  not replace it with the current time, source PDF metadata, `captured`, or
  `highlights_synced_at` during body updates, marker updates, or preference overrides.
- This change guarantees timestamps at creation going forward. Existing notes missing
  `created` retain their current contents; their historical creation times cannot be
  established just by syncing today. No vault backfill belongs in this plan.
- Dry runs remain read-only. A timestamp computed for a preview must not become a
  persisted historical timestamp for a later invocation.

## Implementation

1. Add a `FIELD_CREATED` constant and explicit note-local field handling in the
   Highlights module. Exclude `created` from frontmatter projection extraction even if
   an old `highlights_marker_fields` list contains it. Preserve its raw existing
   frontmatter line when rendering. Reserve `created` in the PDF marker parser with a
   clear diagnostic explaining that it records reference-note creation time; reject
   markers containing it before any note or PDF write. Existing markers that used this
   formerly unknown key should receive that explicit diagnostic rather than silently
   overwriting the note timestamp or stripping PDF metadata. Do not broaden changes to
   unrelated synced properties.
2. Carry an optional new-note creation timestamp in `PipelineMetadata` (or an equally
   small explicit render input). Generate a preview value only when creating a new note,
   and emit exactly one `created` line from `render_with_projection` in that case. Keep
   repeated planning renders stable by reusing the captured metadata. Existing notes use
   their preserved frontmatter instead of a fresh timestamp.
3. In `execute_pdf_sync`, finalize a fresh new-note timestamp immediately before
   rendering and atomically writing the new note, including when there is no sidecar and
   no marker write. Preserve the existing post-marker PDF hash and
   `highlights_synced_at` refresh behavior. Keep the pre-write race checks and
   dirty-target guards intact; a refused write must not create a note.
4. Update `docs/highlights-ref-sync.md` to describe the generated `created` field,
   datetime format, creation-only behavior, note-local ownership, and the distinction
   from `captured` and `highlights_synced_at`. State that this does not backfill older
   notes. No new CLI flags, commands, dependencies, or SASE memory changes are needed.

## Regression coverage and validation

Extend the existing temporary-vault tests; do not touch the real vault or require
Highlights, a browser, or Pandoc for this verification.

- In `tests/cli/highlights/sync.rs`, cover creation without a sidecar at a fixed
  `BOB_NOW` and timezone. Assert exactly one `created` field with the expected
  timezone-bearing datetime, the existing `^ref` task, and unchanged PDF bytes.
- In `tests/cli/highlights/sync_tasks.rs`, cover creation with a sidecar and a later
  sidecar update at a different clock value. `created` stays fixed while highlights
  update normally. This exercises execution-time metadata rerendering.
- In `tests/cli/highlights/scan.rs`, assert timestamps on recursive category notes and a
  note created after `xlib/` intake. Retain dry-run no-write assertions and verify an
  unchanged repeat scan writes nothing at a later clock value.
- Cover an existing note with an authored timestamp, plus an existing managed note
  missing that field. A marker/body sync preserves the authored timestamp and does not
  invent one on the older note. Add focused projection/parser tests proving `created`
  remains local even with stale field opt-in metadata and marker `created` fails clearly
  without writes.
- Check that the timestamp is excluded from marker hash/base/field-list metadata; a
  repeat sync at a different time remains `note_action: none` / `writes: none` without a
  PDF-write opt-in.

Run `cargo test --test cli highlights`, `cargo fmt --check`, and the repository's
standard `just all` checks. If a check needs a long handoff, use the SASE monitor
workflow and wait for its launch command to finish.

## Acceptance criteria

New `^ref` notes from both sync and scan contain one creation datetime before their
first successful atomic write. The timestamp survives later syncs unchanged,
sidecar-free creation works, dry runs write nothing, and adding the field does not cause
PDF mutation or repeat-sync churn. Existing notes retain their prior metadata.
