---
tier: tale
title: Give Google Keep tasks a compact linked source icon
goal:
  Replace generated Source sub-bullets with a linked lightbulb while preserving import
  reliability and task metadata.
size: small
proposed_by: bbugyi200.apollo.6a
create_time: 2026-10-10 09:24:29
status: wip
---

# Give Google Keep tasks a compact, linked source icon

## Outcome

Replace the generated `Source: [Google Keep](...)` child bullet in newly written
`bob gkeep pull` tasks with one linked lightbulb immediately after the task description.
Remove the redundant human-readable Keep creation timestamp. Preserve the source
destination, useful note content, labels, revision indication, and the existing
duplicate-import/archive guarantees.

This is one focused implementation for one coding agent. The root cause is localized:
`src/native/gkeep/render.rs::render_note_in` always appends `source_line_in`, which
combines the source link, timestamp, labels, revision flag, and hidden import marker
into one visible child bullet. Separate those responsibilities without changing the pull
protocol or task semantics.

## Design contract

### Task appearance

Use the ordinary Markdown link `[💡](URL "Open in Google Keep")`. The bulb is a small,
recognizable cue for Keep and remains a usable link in ordinary Markdown. The fixed link
title names the action and destination on hover. Use one separating space; no extra
badge, image download, HTML, CSS, or plugin dependency. The symbol is a source
affordance, never task priority, status, or a new tag.

Place the link after the escaped task description and before the existing
`[created::YYYY-MM-DD]` field. Construct that decorated description and pass it to
`capture::format_task_line`. Do not append arbitrary prose or comments after the task's
fields: the Tasks-compatible parser in
`src/native/dataview/tasks/task.rs::parse_details` consumes trailing fields. A block ID
assigned later must still be the final token on the task line.

Example before:

```markdown
- [ ] #task Call dentist about crown [created::2026-09-27]
  - They close at 5 on Fridays
  - Source: [Google Keep](https://keep.google.com/u/0/#NOTE/note-1) · 2026-09-27 21:14
    %%gkeep:v1:note-1:47582521e307%%
```

Example after (two-space indentation shown; use the actual target indent):

```markdown
- [ ] #task Call dentist about crown
      [💡](https://keep.google.com/u/0/#NOTE/note-1 "Open in Google Keep")
      [created::2026-09-27]
  - They close at 5 on Fridays %%gkeep:v1:note-1:47582521e307%%
```

The visible result has the task, its source icon, and its real content. The provenance
continuation is an Obsidian comment, with no list marker and no blank separator. It must
not leave an empty bullet in rendered Markdown. A title-only import has one task line
and one hidden comment continuation, with zero visible child bullets.

Retain the normal `[created::...]` date and its existing local-time conversion/fallback.
The removed timestamp is the redundant `YYYY-MM-DD HH:MM` text formerly printed in the
`Source:` child. The standard date also supports task age and ordering; do not remove
creation data from the adapter model, list API, or sorting.

### Source destination and missing URLs

Use `KeepNote.url`, exactly the destination source used previously. Preserve scheme
(`http` or `https`), account path, fragment, and query information; do not manufacture a
destination from the note ID, substitute the Keep homepage, downgrade HTTPS, or use a
clipped article's URL instead.

Reuse `encode_source_url` and retain its existing protection for `%`, parentheses,
brackets, angle brackets, and whitespace. Because the new link has a quoted title, also
protect quote and backslash characters that could break Markdown parsing. The title
itself is a fixed literal, never Keep text. Retain the existing percent-encoding
semantics rather than redesigning URL normalization. Test unusual destinations and
spoofed marker text.

If the URL is absent, empty, or whitespace-only, omit the source icon/link. Never emit
an empty Markdown link or a decorative icon that suggests an available action. Import
the task and preserve its hidden marker normally.

### Labels, revisions, and remaining children

Keep label data when present as one ordinary child `- 🏷 errands, weekend`, using the
existing normalization/escaping and label order. No label child for an empty label list.
Labels remain plain labels, not Obsidian tags or inline fields. Keep list items, note
text, attachment summaries, and OCR children in their existing order and indentation.

For a revision, append ` · revised` to the escaped task description before the source
link and fields. Example:

```markdown
- [ ] #task Hardware store · revised
      [💡](https://keep.google.com/u/0/#NOTE/note-2 "Open in Google Keep")
      [created::2026-09-26]
  - [ ] wood screws
  - 🏷 errands, weekend %%gkeep:v1:note-2:32a5e5e2fd2c%%
```

Keep the marker fingerprint derived exclusively from the existing canonical Keep
content; the generated annotation does not participate in hashing. Permanent clipping
failures keep their existing escaped warning/retry child as the last visible child,
after labels if any, followed by the hidden marker. Normal imports, revisions,
`--no-ref`, and permanent clip-failure tasks all use the same source-link rendering.
Successful reference clips continue through their existing path.

### Hidden marker and compatibility

Emit exactly one unchanged `%%gkeep:v1:<encoded-id>:<fp12>%%` marker per rendered block,
on a final line prefixed with the target's indent unit and no `- `. Do not change its
version, ID encoding, fingerprint, journal shape, or archive verification logic.

This placement is deliberate: `note_tasks::clean_description` does not remove Obsidian
comments, so placing the marker on the task line would leak bookkeeping into task
pickers. Keeping it as an indented continuation also leaves trailing-field parsing
alone. Both `gkeep::ledger::task_block` and `collect_done::transform::task_block_end`
carry indented non-bullet lines.

The linked `bob-plugins` repository was inspected read-only at commit
`bec01f6e7bc35dbbe9d0e49c039842b6963d803d`:
`plugins/bob-navigation-hotkeys/src/240-cancel-and-lanes.js` functions
`taskMoveLineIsDescendant`, `captureTaskMoveSubtree`, and `rebaseTaskMoveBlock`,
together with `src/250-task-move.js`, preserve these indented lines during task moves.
No plugin change or deployment is part of this plan. If reinspection is needed, open
`bob-plugins` with `sase repo open` and use the returned path.

Ledger scanning already accepts markers anywhere in a task block. Keep accepting legacy
`Source:` child markers, the new comment continuation, and markers on task lines.
Old-format tasks remain byte-for-byte untouched on an archive-only pull, including tasks
moved to another note or `done/`. This is a change to newly generated tasks and
revisions, not a migration of existing vault content. Do not bump fingerprints just to
force new output.

### Readable command output

Adding an ordinary Markdown link to a task would otherwise put its full URL and tooltip
in the plain-text `bob gkeep list` table, consuming the column. In the human vault-table
presentation only, collapse this generated link to `💡` before measuring
width/truncating the task text. Limit recognition to marked Keep tasks and the exact
generated bulb-link shape with the fixed `Open in Google Keep` title. Preserve
surrounding prose and other links.

Leave the JSON `description` as the ordinary parsed task description, including its new
Markdown link, and keep all JSON keys/schema versions. Dry-run and pull JSON `markdown`
must remain the exact Markdown written; do not apply the human table projection to
stored content, preview Markdown, identity digests, or other capture/task commands.

## Implementation

1. Update `src/native/gkeep/render.rs` to build the inline source link, revision
   annotation, optional label child, and final comment continuation. Retire
   `source_line_in` and the now-unused `created_datetime_in`; retain `created_date_in`.
   Rename `extra_before_source` and revise fallback comments/tests to express its new
   last-visible-child placement. Keep escaping, `RenderedBlock`'s no-trailing-newline
   contract, and indentation behavior. Update the stale `Source:` placement comment in
   `src/native/gkeep/pull.rs::build_writes`.
2. Add the narrow human-table projection in `src/native/gkeep/list.rs` (or a small
   reusable helper local to gkeep) and apply it before width calculation. Do not alter
   shared task parsing or JSON descriptions.
3. Update existing rendering expectations and add the behavioral coverage below in the
   existing gkeep test modules. Keep explicit legacy-format fixtures instead of
   mechanically converting every `Source:` occurrence. Add a focused archival
   preservation case in `src/native/collect_done/tests/unit.rs`.
4. Update `docs/gkeep.md` rendering examples, marker location, fallback placement, list
   description, and compatibility explanation. Explain `💡`, its tooltip, missing-URL
   behavior, and the retained creation-date field. Do not add flags, configuration, a
   migration command, or memory files. No live Google Keep/vault operations are needed
   for implementation.

## Verification and acceptance

Use the existing fake Keep adapter and temporary vaults. Extend the current exact-output
and integration tests where possible rather than duplicating the whole rendering matrix.

- Renderer: title-only, text, list, attachment/OCR, revision, labels, and
  permanent-failure forms all obey the new layout. For generated metadata, there is no
  `Source:` child and no standalone Keep creation timestamp. Do not erase user-authored
  body text merely because it says `Source:` or contains a date. Keep tab and two-space
  indentation, escaped task text, checked/nested list children, and no trailing newline.
- URLs: ordinary HTTP and HTTPS note URLs retain their destination; query and fragment
  survive. Missing/null/empty/blank URLs produce no link. Delimiters, quotes,
  backslashes, whitespace, and marker-looking URL/text/ label content cannot split a
  task line or introduce an additional parseable marker. Assert exactly the real
  `(id, fingerprint)` is found.
- Task compatibility: read rendered blocks through `read_target_tasks` with and without
  children, check date/status/checkbox counts and marker association, and ensure the
  following task is not consumed. Exercise `dataview::parse_details` on the generated
  task body to prove the created date is still recognized with the link and revision
  text. A trailing block ID must still be recognized after assignment.
- Pull: update the complete Markdown golden fixture in `tests/gkeep_pull.rs`, its
  space/CRLF case, revision expectations, and permanent clip fallback case. Extend
  dry-run/real Markdown equality to a URL-bearing note. Existing write/verify/archive
  guards must still pass.
- Idempotency: in a temporary vault containing both legacy and new markers, re-pull
  unchanged notes with a fresh/empty local journal; assert no new blocks and the
  existing bytes unchanged. Include a marked block moved outside the target note and a
  completed block in `done/`, proving the vault ledger itself prevents duplicates.
  Edited Keep content produces one new revision using the new layout; another identical
  pull does not.
- Archival: the real `collect_done` transform must move the task line, source link,
  payload, and indented comment together, including CRLF. The resulting archive marker
  must still scan. Test a neighboring task to ensure the comment stays associated with
  its original task.
- Human list: a generated source link renders as one `💡`, without its URL, tooltip, or
  hidden marker; preserve normal truncation, child counts, and `still in Keep` hints.
  Other Markdown links and malformed/lookalike links are not swallowed. JSON preserves
  the Markdown description and Keep ID; legacy tasks continue to list correctly.

During implementation run targeted checks as useful:

```bash
cargo test --bin bob native::gkeep
cargo test --test gkeep_pull --test gkeep_list --test gkeep_cli
```

Finish with the repository's canonical `just check` (format, clippy, and all Rust
tests), using the SASE monitor workflow if it needs a long-command handoff. Fix failures
caused by this change; report unrelated failures accurately. No live authentication or
Keep account is required.

If an Obsidian UI is available, use a disposable preview note with these fixtures to
inspect Reading view and unfocused Live Preview: exactly one linked bulb on URL-bearing
tasks, a descriptive hover title, hidden marker, no empty source bullet, unchanged child
indentation, and ordinary handling of a task without a source URL. Verify Ctrl+Shift+M
carries the comment. If UI access is unavailable, report that limitation; do not claim
the Markdown fixtures alone validate actual Obsidian rendering or interaction.

The feature is complete when the new output and compatibility cases pass, documentation
reflects the exact format, and no application source was changed before this plan
received approval.
