---
tier: tale
title: Relate Highlights annotation tasks to their ref task
goal:
  'When highlights.annotation_task_relation is configured, every task Highlights sync
  creates from a #task annotation carries a LINKS relation with that label to the
  reference''s reading task; Bryan''s chezmoi config sets it to "inspired by".'
size: medium
proposed_by: bbugyi200.athena.166
proposed_with: claude/opus
create_time: 2026-10-30 10:11:39
status: wip
---

# Plan: Link Highlights annotation tasks to their reading task with a configured relation label

## Goal

When Highlights sync (`bob ref scan`, `bob ref sync`, and create-time ingestion) creates
a task from a `#task` bullet in a PDF annotation, it can also write a task relation from
the new task to the reference's reading task (the "ref task"). The user picks the
relation label in config. The feature is opt-in: without the key, output stays
byte-identical to today.

Configure Bryan's chezmoi `config.yml` to use the label `inspired by`.

Example (v2 reference, default route; the follow-up lands in the reading task's
residence note):

```md
## Tasks

- [ ] #task #ref [[ref/books/example|Example]] [created::2026-10-01] ^ref-example
- [ ] #task Compare this claim with the appendix.
      [[ref/books/example#^h-2b91f0a4c7de|🔖]] [h:: 4c0a13d2...] [created::2026-10-30]
  - 🪢 **LINKS:** inspired by [[#^ref-example]]
```

On a later `bob task reconcile`, the reading task gets its generated
`🪢 **LINKED FROM:** [[#^<minted-id>]] (inspired by)` line, and the follow-up task gets
a minted `^block-id`. Reconcile already owns both (`docs/task-relations.md` §§5, 7).
Because the ref note embeds the reading task (`![[<residence>#^ref-example]]`), the ref
note then shows which tasks it inspired.

## Background (read these first)

- `docs/task-relations.md`: §3 (grammar and writer form), §4 (slot: the first direct
  child when the task has no other children; indent = parent indent + one tab), §5
  (identity, link form, and minting), §7 (reconcile is the authority for LINKED FROM and
  mints missing source ids), §8 (immediate writers table), §17/§18 (cancel and complete
  labels fire only on explicit gestures).
- `docs/highlights-ref-sync.md`: the annotation-task section ("Any unordered Markdown
  bullet line whose item text contains `#task`…") and "Reading Tasks In Parent Notes
  (v2)".
- `src/native/highlights_ref/annotation_tasks.rs`: `render_annotation_task_line`,
  `annotation_task_source_link`.
- `src/native/highlights_ref/sync.rs`: `finalize_annotation_task_plans` (renders lines
  at plan time; v1 local lines go into `plan.rendered_body`, everything else goes into
  `RoutedInsertionIntent`s), `execute_pdf_sync_inner` (v1 execution),
  `execute_pdf_sync_v2`, `execute_v2_reading_action` / `V2ReadingOutcome` (its `embed`
  is the reading task's final `(<vault path without .md>, <block id>)` after allocation
  and revalidation), and `execute_routed_intents`.
- `src/native/highlights_ref/note.rs`:
  `insert_annotation_task_lines_into_tasks_section`, `append_task_lines`. Capture's
  `insert_task_line` (`src/native/capture/sections.rs`) already accepts a `\n`-joined
  block and converts it to the document's line ending.
- `src/native/task_relations/format.rs::format_links_line` and
  `src/native/task_relations/label.rs::normalize_relation_label`.
- `src/native/task_dependencies/format.rs::canonical_link` / `cross_note_link` (the §5
  link form), plus `src/native/vault_links.rs` (`NoteIndex`, `VaultLinkResolver`).
- `src/native/config/mod.rs`: `RawHighlights`, `HighlightsConfig`,
  `parse_highlights_config`. `src/native/config/relations.rs` shows the label-validation
  and warning wording conventions.

## Design

### Config: `highlights.annotation_task_relation`

```yaml
highlights:
  annotation_task_relation: inspired by
```

- Type: string. If the key is missing, `null`, or empty after trimming, the feature is
  **off** and output is byte-identical to today.
- Add `annotation_task_relation: Option<String>` to `RawHighlights` and keep the raw
  trimmed string on `HighlightsConfig`, with an accessor. **Do not** validate it inside
  `parse_highlights_config`. That function feeds capture URL routing, doctor, hooks, and
  marker writes, so a bad label there would break unrelated features.
- Validate the label where it is used, with `normalize_relation_label` (§3.2 rules; `💬`
  is reserved). An invalid label is a `CommandError` raised in
  `finalize_annotation_task_plans`, which runs before any write. The scan creates
  nothing rather than creating tasks that are missing their relation (nothing backfills
  later). Message:
  `invalid highlights.annotation_task_relation in <path>: <reason>; fix or remove it`.
  For `ReservedMention`, use the existing wording `💬 is reserved for text mentions`.
- The label is stored as normalized (`normalize_relation_label` output). Case is kept.
- Load it inside `finalize_annotation_task_plans` with
  `load_highlights_config(&config_path())` (map errors with `hooks::config_error`), but
  only after the existing "no candidates" early return, so the common path still reads
  nothing new. Factor out an inner function that takes `Option<&str>` so unit tests can
  inject the label without touching the user's real config. Store the normalized label
  on each `PdfSyncPlan` that creates tasks (for example
  `annotation_task_relation: Option<String>`), so execution never re-reads config.
- Unknown keys are already ignored (no `deny_unknown_fields`). Older bob builds
  therefore ignore the new chezmoi key, and the two repos can land in either order.

### What gets written

For every annotation task that sync **newly creates** while the label is configured and
the ref task resolves, the task's block becomes two lines:

1. the unchanged task line from `render_annotation_task_line`;
2. `format_links_line("\t", &[(link, Some(label))])`, which is the task's only child, so
   it sits in the §4 LINKS slot with the parent indent (column 0) plus one tab.

Only the LINKS line is written. Do **not** mint a `^block-id` on the new task and do
**not** write the reading task's LINKED FROM line. Reconcile mints and projects both
(§7; it honours `relations.linked_from_lines`). Live chips render from the live index,
so Obsidian shows the relation immediately. This keeps sync's single-destination writes
and parent-first ordering unchanged; in particular, an explicitly `@routed` follow-up
never needs a second write into the reading task's note.

The relation is inert, like every relation (§2). Even when the configured label is also
in `relations.cancel_labels` or `complete_labels`, nothing is cancelled or completed:
§17/§18 fire only on explicit relation gestures, and Highlights sync is not one. Never
add a relation to an existing task, and never re-add one the user removed. Dedup still
keys on `[h::]`, so a task that already exists is untouched.

### The target: the "corresponding ref task"

| Reference shape                                                                                                       | Target task                                                                                                                                                                                         | Known at                                                                         |
| --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| v1 (in-note `^ref` tracker in the ref note)                                                                           | the ref note's `^ref` line, block id `ref`                                                                                                                                                          | plan time; link only when `parse_pdf_task_line(plan.rendered_body)` is `Present` |
| v2 (reading task in its residence)                                                                                    | `V2ReadingOutcome.embed`, i.e. the reading task's **final** address after `execute_v2_reading_action` (a new birth may land past the planning preview id; an archived task keeps its `done/…` path) | execution time                                                                   |
| no ref task (trackerless legacy note, v2 `open_ref_without_task`, refused or ambiguous reading task, `embed == None`) | none: write the task exactly as today, with no LINKS line                                                                                                                                           | —                                                                                |

Link text comes from the destination note the task is written into, following §5:

- Same note (v1 local `## Tasks` tasks; v2 default follow-ups, which are retargeted to
  the reading task's residence): `[[#^ref]]` / `[[#^ref-example]]`.
- Cross note (explicit `@name` routes, or v2 fallback to `mac_inbox` when the reading
  task is archived): use `canonical_link`/`cross_note_link` on vault-relative paths.
  Archive targets keep the explicit `[[done/…#^id]]` form. Basename uniqueness needs a
  `NoteIndex`; build it lazily, at most once per run (for example cached on
  `RunWriteState`), and only when a cross-note link is actually rendered.
- Never emit a link to an id that will not be on disk after the write (§5). For v2 that
  means rendering the LINKS line inside execution, after the reading-task action, from
  the actual outcome, not from the planning preview.

### Plumbing

- Represent each created task as a **block** (`Vec<String>`: task line plus an optional
  LINKS line) instead of one line, and emit every block line with the destination's line
  ending:
  - v1 local: push the block lines into
    `insert_annotation_task_lines_into_tasks_section`'s `lines` in order. The target is
    the fixed `^ref` id, so render at plan time.
  - v2 default routes: join the block with `\n` for capture's `insert_task_line`, which
    converts to CRLF when needed.
  - explicit routes: flatten the blocks into `append_task_lines`'s line list. Never pass
    an embedded `\n` there, or CRLF files would get mixed line endings.
- `RoutedInsertionIntent` keeps its per-candidate dedup (`candidates` parallel to the
  rendered entries). Record, per intent, whether a relation is wanted, and resolve its
  link in `execute_routed_intents`. Pass that function the resolved ref-task address: v1
  `(ref note path, "ref")` when the tracker is present; v2 `reading.embed`. Keep the
  existing preimage-retry loop rendering against fresh bytes.
- `execute_pdf_sync_inner` (v1) and `execute_pdf_sync_v2` both call
  `execute_routed_intents`. Thread the address through both.
- Dry runs write nothing, as today. Do not add JSON or human output fields: the scan
  JSON shape and the concise counts stay unchanged.
- Callers that only plan (`rename/plan.rs`, `reference_mutation.rs`) need no behaviour
  change beyond compiling.

## Tests

Unit tests (next to the code you touch, for example
`src/native/highlights_ref/tests/tasks.rs` and `src/native/config/mod.rs` tests):

- Config parsing: the key is absent, `null`, empty, or whitespace-only (off); a valid
  label is kept raw; the lenient parse never errors on a bad label.
- Label validation at use time: an invalid character, too long, and `💬…` each produce
  the error message above.
- Block rendering: the label is off (single line, byte-identical); same-note link;
  cross-note basename link; nested path link; archive `done/` link; CRLF destination for
  both the `insert_task_line` and `append_task_lines` paths.
- `parse_pdf_task_line` still finds exactly one generated line when a follow-up's
  `inspired by [[#^ref]]` LINKS line and a `^ref` LINKED FROM child are present in a v1
  note.

CLI tests under `tests/cli/highlights/` (the support code points `BOB_CONFIG_FILE` at a
missing file by default; write a temp `config.yml` like `marker_write.rs` /
`overwrite_annotated.rs` do):

- Without the key, existing annotation-task fixtures stay byte-identical (the existing
  tests already cover this; keep them green).
- With `annotation_task_relation: inspired by`:
  - v1 local task gets `\t- 🪢 **LINKS:** inspired by [[#^ref]]`;
  - v1 `@routed` task links `[[<ref note link>#^ref]]`;
  - v2 birth and follow-up in one run: links the **final** reading-task id, including
    when a destination collision suffixes it (pattern:
    `v2_destination_collision_suffixes_preview_id`);
  - v2 explicit `@route` to another note links cross-note to the residence;
  - a rerun is a no-op (no duplicate task, no second LINKS line);
  - a reference with no reading task still creates the task, with no LINKS line;
  - an invalid label fails the scan before any write (vault unchanged).
- A follow-up scan after `bob task reconcile` has added the LINKED FROM line under the
  reading task (and minted the follow-up's id) stays a no-op for that PDF. This proves
  sync's locator and status logic ignore the new children.

Run `just fmt`, `just lint`, `just test`, and `just docs-check`.

## Docs (bob-cli)

- `docs/reference/configuration.md` → `### highlights` table: add a
  `highlights.annotation_task_relation` row (string relation label; default unset = off;
  what it writes; invalid label fails scans that create annotation tasks; link to the
  new highlights section).
- `docs/highlights-ref-sync.md`: in the annotation-task section (after the 🔖 backlink
  paragraph), add a short "Relation to the reading task" subsection. Cover the config
  example, v1/v2/routed link forms, LINKS-only writes with reconcile minting and
  mirroring, new tasks only, inertness (including cancel/complete labels), and the
  no-ref-task case.
- `docs/task-relations.md` §8 "Immediate writers" table: add a row for Highlights
  annotation tasks (opt-in via `highlights.annotation_task_relation`). It writes only
  the new task's LINKS line, at creation time; there is no mint and no mirror (reconcile
  does both); it never edits existing LINKS lines; it is not an explicit gesture, so
  §17/§18 never fire.
- Do not edit `CHANGELOG.md` or the `Cargo.toml` version (release-train policy). Use a
  conventional `feat(highlights): …` commit; this is not a breaking change.

## Chezmoi (Bryan's config)

Open the linked `chezmoi` repo with `/sase_repo` (`sase repo open chezmoi -r "…"`), read
its `AGENTS.md`, and edit `home/dot_config/bob/config.yml`. Add to the existing
`highlights:` block, after `coordinate_pdf_writes`, in the block's comment style:

```yaml
# Link each task created from a Highlights `#task` annotation back to its
# reference's reading task on the new task's 🪢 LINKS line ("<task> inspired by
# <reading task>"). Unset links nothing. See "Relation to the reading task" in
# bob-cli docs/highlights-ref-sync.md.
annotation_task_relation: inspired by
```

The chezmoi repo's own instructions require `chezmoi update -a --force` after its commit
lands; follow them. Released bob builds that predate this feature ignore the key.

## Non-goals

- Backfilling relations onto annotation tasks created before this change.
- Writing the LINKED FROM mirror or minting the source id during sync (reconcile owns
  both).
- Adding `inspired by` to the display-only inverse phrase table, or any bob-plugins or
  Bob Mac Capture change.
- New JSON fields, CLI flags, or a per-PDF or per-annotation label override.
