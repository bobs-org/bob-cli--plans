---
tier: tale
title: "Write ## Notes in record notes only when note bullets exist"
goal:
  "Record notes carry a ## Notes section only when they hold note bullets: creates and
  updates add it on demand in its canonical place, and bob records tidy (run by bob
  nightly) removes empty Notes sections from existing records."
size: medium
decisions:
  first_cleanup:
    ask:
      Should the coder run the real bob records tidy on the live vault once after
      landing?
    choices:
      nightly:
        Coder only dry-runs; the next bob nightly with the new bob cleans the ~105
        records
      now:
        "Coder runs it once after just install: one commit across ~105 records, then
        nightly"
    default: nightly
    why:
      Nightly already cleans periodically, so the coding turn never writes the live
      vault
    answer: now
proposed_by: bbugyi200.apollo.em
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 15:14:08
status: wip
---

# Write `## Notes` in record notes only when note bullets exist

## Goal

Today every record note bob writes carries a `## Notes` heading, even when it has no
note bullets. In the live vault, 105 of 186 record notes have an empty `## Notes`
section: 42 end with it, and 63 have it directly before a legacy `## Source` block.

After this change:

1. **Create.** `bob records new` and capture `@album/` write `## Notes` only when the
   new record has at least one ordinary note bullet.
2. **Update.** `bob records update -n` and capture `@album/<stem>` create `## Notes` on
   demand when the first ordinary bullet arrives. It goes in its canonical place, right
   after the H1 region and before the first other `##` section, never after `## Source`.
3. **Cleanup.** A new `bob records tidy` removes empty `## Notes` sections from existing
   record notes. `bob nightly` runs it every night, so stray empty sections (for
   example, a heading Bryan typed and never used) get cleaned up periodically.

The change is bob-cli only. bob-plugins and Bob Mac Capture need no change; see
[Out of scope](#out-of-scope).

## Starting evidence (inspected at bob-cli `3236ceef`)

- **Render.** `src/native/records/render.rs` `render_record_with_identity` always writes
  `format!("# {}\n\n## Notes\n", request.title)`, then the ordinary bullets, then named
  sections (`RecordNoteInput::section`, ALL-CAPS bullets with children, title-cased).
  Its doc comment says "The body always holds H1 and `## Notes`." `RenderedRecord.body`
  feeds capture's `note_files[].body` JSON.
- **Update.** `apply_record_update` calls `append_notes_to_body`, which calls
  `append_bullets_under_notes`. That code finds `## Notes` with `parse_body_headings`
  (fence-aware: `level == 2 && title.trim() == "Notes"`). When the heading is missing,
  it appends `## Notes` and the bullets at the **end of the body**, after any
  `## Source` or named section. Named sections (`append_section_bullets`) reuse a
  matching heading or append at the end. Field-only updates never touch the body.
- **Callers.** Both capture paths use the same two functions through
  `records::plan_create*` and `records::plan_update`:
  `src/native/capture/record_note.rs` and `src/native/capture_record.rs`. So do
  `bob records new` and `update` (`src/native/records/cli.rs`) and the in-progress
  envelope writer. No other bob code writes record bodies.
- **Readers.** No bob reader requires `## Notes`: `records check`, `show`, the
  fingerprint (which excludes the body), and the album index all ignore it. The
  bob-plugins record sheet (bob-cli-ms.3, `60b5602`) anchors after the first H1 and
  never looks for `## Notes`. Bob Mac Capture only renders the `body` string bob
  returns.
- **Periodic hook.** `src/native/nightly.rs` registers each maintenance step as one
  `Step` in `STEPS`, with a lock-free core. `src/native/albums/sync.rs` is the closest
  template for a guarded maintenance writer: `dry_run`, `run_standalone` through
  `host_catalog::run_locked`, `sync_with_existing_lock` and `nightly_step` through
  `apply_and_commit`, `FileEdit` full-preimage guards, `print_human`, and `to_json`. The
  plugin auto-runs `bob albums sync` after frontmatter edits, so the cleanup must
  **not** live inside albums sync. It would rewrite record bodies under Bryan's cursor,
  and it would race the Record Card's guarded writes.
- **CLI registration.** `bob records <verb>` members live in `src/runner.rs` (member
  table, `RECORDS_AFTER_HELP`, group about), `src/native.rs` (`NativeCommand` and
  dispatch), and `src/native/completion/tree.rs`. The help fixtures are
  `tests/fixtures/help/root-{short,long}.txt`; they include the records group about
  line. The nightly step list appears in `nightly.rs` `help_text()`,
  `docs/vault-git-sync.md` "Nightly maintenance", and `docs/records.md` "Syncing records
  indexes".

## Coordination with the bob-cli-ms epic (Record Card)

bob-cli-ms is in flight. Its only remaining bob-cli phase is **bob-cli-ms.2
(record-envelope)**, which edits these files:

- `records/cli.rs` (`execute_update`, `build_cli_update`);
- a new records protocol module;
- possibly `plan.rs` (a non-comma-splitting input variant);
- albums list/check JSON (the capability flag);
- `docs/records.md` ("Updating with an envelope");
- `docs/reference/json-outputs.md` (Record reports rows).

Phases 4–7 are bob-plugins work and never touch bob-cli.

This work runs in parallel. To keep merges trivial:

- Put all new logic in **new files**: `src/native/records/tidy.rs`, plus
  `src/native/records/notes_section.rs` if you split out the pure helpers. Edit
  `records/cli.rs` and `records/plan.rs` not at all, or only to re-export. `render.rs`
  is not on phase 2's path. If you must widen visibility of `split_preserving_ends`,
  `parse_body_headings`, or `SplitLine` to `pub(super)`, do that there.
- Do not change `bob records update` arguments, its JSON, or `execute_update`.
- The envelope design says "bob edits only YAML spans". This plan keeps that true: body
  bytes change only when notes are appended, never on field-only updates. `tidy` never
  runs inside `update`, capture, or albums sync.
- Before the final gate, rebase onto the latest `master`. If a bob-cli-ms commit landed
  in the meantime, re-read it. Update any newly landed tests that assert a `## Notes`
  heading in a freshly created no-notes record. When `docs/records.md` or
  `json-outputs.md` conflict, keep both sides' rows and prose.
- Do not edit the epic's beads, plan, or in-flight files. If you find a real conflict
  you cannot resolve, file a task bead through `/sase_new_task` and describe it there.
- Wire the new module in with `pub(crate) mod tidy;` in `records/mod.rs`. The command's
  builder and runner live in `tidy.rs`, not in `records/cli.rs`.

## Design

### 1. Create: Notes only with ordinary bullets

In `render_record_with_identity`, start the body with `# {title}\n`. Write
`\n## Notes\n\n` plus the bullets only when `ordinary` is non-empty. Named sections keep
rendering as `\n## {title}\n\n- …\n`. The results:

| Notes supplied     | Body                                                         |
| ------------------ | ------------------------------------------------------------ |
| none               | `# T\n`                                                      |
| ordinary only      | `# T\n\n## Notes\n\n- a\n`                                   |
| section only       | `# T\n\n## Must Order\n\n- Dragon roll\n`                    |
| ordinary + section | `# T\n\n## Notes\n\n- a\n\n## Must Order\n\n- Dragon roll\n` |

The full Markdown stays `"{frontmatter}\n{body}"`. Update the doc comment. Nothing else
changes: dated notes, nesting, section merging, or frontmatter.

### 2. Update: create Notes on demand, in its canonical place

Change only the "no `## Notes` heading" branch of `append_bullets_under_notes`. The
existing-heading branch stays byte-identical. Insert the new section before the **first
heading of level 1 or 2, outside fenced code, that follows the body's first H1**. When
the body has no H1, use the first such heading. When there is no such heading, append at
the end exactly as today.

Spacing rules for the inserted lines (`## Notes`, a blank line, the bullets, a blank
line):

- When the line before the insertion point is non-blank, insert one blank line first.
- Never leave two blank lines in a row.
- Use the note's own newline style (`detect_newline`, CRLF preserved).

Examples:

- `# B\n\n## Source\n\n```text…` + note →
  `# B\n\n## Notes\n\n- 2026-11-17: x\n\n## Source\n\n```text…`
- `# B\n## Source\n` + note → `# B\n\n## Notes\n\n- x\n\n## Source\n`
- `# T\n\n## Must Order\n\n- Dragon roll\n` + note → Notes inserted above
  `## Must Order`
- `# Talk Python\n` + note → `# Talk Python\n\n## Notes\n\n- x\n` (end append, as today)

Named-section handling (`append_section_bullets`) is unchanged.

### 3. The empty-section rule (pure, shared)

Add a pure function, for example
`strip_empty_notes_sections(original: &str) -> Option<String>`. It returns `None` when
nothing changes. It operates only on the body after a well-formed frontmatter block
(`---` … `---`). It refuses (returns `None`) when the note has no closed frontmatter.

- **What counts as a Notes section.** A level-2 heading whose trimmed title is exactly
  `Notes`. Use the same matcher as `append_bullets_under_notes`. Headings inside fenced
  code never count (`parse_body_headings`). The section runs to the next heading of
  level ≤ 2 outside a fence, or to EOF.
- **When it is empty.** Every line after the heading inside the section is
  whitespace-only. Any other content keeps the section: bullets, prose, a `###` heading,
  `%%` or HTML comments, a fence, or a block id.
- **Removal.** Delete the heading line and the blank lines that follow it in the
  section.
  - If the section ran to EOF, also drop the blank lines left at the end of the file.
    The note then ends with its last content line and that line's original ending.
  - If another heading follows, keep the blank lines that preceded the removed heading
    as they were, so they now separate the earlier content from the next heading.
- **Multiple sections.** Remove every empty `## Notes` section and keep non-empty ones.
- **Preservation.** Frontmatter bytes, BOM, line endings, and every other line stay
  identical.
- **Invariants to test.**
  - `strip(old_render(no notes)) == new_render(no notes)`, byte for byte. For example,
    `---…---\n\n# T\n\n## Notes\n` becomes `---…---\n\n# T\n`.
  - `strip` is idempotent.
  - `strip` of any new render is `None`.

Live-vault shapes this must handle:

- `# B-Unos\n\n## Notes\n\n## Source\n\n```text…` → `# B-Unos\n\n## Source\n\n```text…`
- `# T\n\n## Notes\n` → `# T\n`
- One live note lacks a final newline (`# T\n\n## Notes`) → `# T\n`

### 4. `bob records tidy`

```text
bob records tidy [TYPE] [-b DIR] [-d] [-f human|json] [-h]
```

- **About:** "Remove empty `## Notes` sections from Record Notes".
- **Long about:** say what counts as empty, that only record bodies change, and that a
  real run follows the shared maintenance lifecycle. `TYPE` accepts a noun or album
  stem, as `records check` does. An unknown `TYPE` refuses with exit 2. Without `TYPE`,
  every album is processed.
- **Options** (alphabetical, each with a short alias, per `cli_rules`):
  - `-b/--bob-dir`, `-d/--dry-run`, `-f/--format`, `-h/--help`;
  - reuse the records arg helpers.
- **Examples:** `bob records tidy`, `bob records tidy -d`, `bob records tidy eat -d`,
  and `bob records tidy -f json`.
- **Membership.** For each album in `AlbumRegistry::from_snapshot` (refuse like
  `plan_indexes` on partial discovery), take the direct Markdown children of the album
  folder. Count a child as a record when its frontmatter type is `record` or the album's
  legacy noun type, read as `check.rs` reads it (`frontmatter_type_names`). Read full
  contents from the `VaultSnapshot`. Skip unreadable notes and report them in
  `io_errors[]`. Never touch notes outside album folders, album declarations, or
  non-record notes in the folder.
- **Dry run.** Plan the edits and print unified diffs (reuse the albums-sync diff
  styling). It takes no lock, runs no Git, and does no vault-sync.
- **Real run.** Follow `albums sync` `run_standalone`:
  1. take the shared lock and pre-sync;
  2. stage a guarded transaction with one full-preimage `FileEdit` per changed record;
  3. verify that each written record no longer has an empty Notes section;
  4. make one scoped commit `records: tidy (N notes)`;
  5. post-sync only when it committed.

  A no-op writes and commits nothing and skips post-sync. A preimage race fails the
  transaction with nothing written, exit 1.

- **Lock-free core.** Add a core the nightly step calls under its own lock, mirroring
  `sync_with_existing_lock` and `nightly_step`.
- **Human output.** Colored. Show a header with counts (records scanned, records
  tidied), then one row per tidied path (or its diff with `-d`), the commit line, and
  warnings.
- **JSON** (`schema_version: 1`, modeled on the albums sync report):
  - `ok`, `command: "tidy"`, `dry_run`;
  - `counts {scanned, changed}`;
  - `changes[] {path, diff}`, where `diff` is present on dry run;
  - `commit {sha, subject}|null`, `sync {pre, post}|null`;
  - `warnings[]`, `io_errors[]`;
  - a nullable `error {code, message, hint}`.
- **Exit codes.** 0 on success or no-op, 1 on a runtime or guard failure, 2 on a usage
  error or unknown `TYPE`.
- **Registration.**
  - `src/runner.rs`: add the member between `show` and `update`, and add one
    `RECORDS_AFTER_HELP` example. Update the group about to "List, inspect, check,
    create, update, and tidy Record Notes", and update the long about to match.
  - `src/native.rs`: add `NativeCommand::RecordsTidy` and its dispatch.
  - `completion/tree.rs`: add the entry.
  - Refresh the help fixtures.

### 5. Nightly

Add a `records` step to `STEPS` **after `albums` and before the final `vault-sync`**,
with blurb "Remove empty record Notes sections". The `docs/records.md` sentence "album
sync runs directly after `notebooks`" stays true. Update `help_text()`: the summary list
and the numbered workflow, which gains a step "remove empty record `## Notes` sections
(one scoped commit when any changed)" before the final vault-sync. A failure must not
stop the final publish; this is the existing loop behavior. Update any nightly help or
step-count test.

## Tests

- **render.rs unit tests.** Rewrite the "H1 and Notes always present" assertion. Add the
  four create cases from the table, `dated_notes` on and off, and the section-only case.
- **Update unit tests.**
  - Insertion before `## Source` (blank line already present, and missing).
  - Insertion before a named section.
  - No H1.
  - End append when no heading follows.
  - CRLF and a missing final newline.
  - A fence containing `## Source` is not an insertion point.
  - The existing-heading path is byte-identical to today (the existing goldens in
    `plan.rs` and `render.rs` keep passing unchanged).
  - Field-only updates leave body bytes identical.
- **Strip unit tests.**
  - Every live shape above.
  - Multiple Notes headings.
  - A non-empty section with only a `###` subheading or a `%%` comment is kept.
  - A fenced `## Notes` is ignored.
  - BOM, CRLF, and no frontmatter (returns `None`).
  - The idempotence and round-trip invariants.
- **CLI tests** (`tests/cli/records.rs`, following the albums_sync test patterns):
  - dry run writes nothing and leaves HEAD unchanged;
  - a real run commits once with the subject and changes only record bodies;
  - a no-op makes no commit;
  - `TYPE` scoping, and an unknown `TYPE` exits 2;
  - non-record notes in the folder and album declarations are untouched;
  - JSON shape.
- **Capture and records CLI tests.** Update the expectations for no-notes creates in
  `tests/cli/records.rs`, `tests/cli/capture/record.rs`, and any capture JSON
  `note_files[].body` assertion. Add one capture update test: a record whose body is
  `# T\n\n## Source\n\n```text\nrow\n```\n` gains `## Notes` above `## Source`.
- **Nightly.** Cover the new step in the help test or fixture.

## Docs

Read `docs.md` and `cli_rules.md` through `/sase_memory_read` first.

- `docs/records.md`, "The note Bob writes": replace "`## Notes` is always present,
  including when empty" with the create rule. Add a no-notes example body.
- `docs/records.md`, "Updating notes" and the `bob records update` paragraph: notes
  append under an existing `## Notes`; otherwise Bob creates the section directly under
  the H1 region, before the first other `##` section.
- `docs/records.md`: add a "Tidying empty Notes sections" subsection after "Syncing
  records indexes". Cover the rule, the command, the lifecycle, JSON, and the nightly
  step. Add the command to the `bob records` table there.
- `docs/reference/vault-format.md` "Record notes": say that `## Notes` exists only when
  there are bullets.
- `docs/vault-git-sync.md` "Nightly maintenance": add the step and renumber.
- `docs/reference/json-outputs.md` "Record reports": add a `bob records tidy` row.
- Run `just docs-gen` to regenerate the CLI reference, then `just docs-check`.

## Verify

1. Run `just check` (the full gate) through `/sase_monitor`, after rebasing onto the
   latest `master`.
2. Run `just install` from the landed tree.
3. Run a read-only live spot check:
   - `bob records tidy -d` reports about 105 changed records (42 at EOF, 63 before
     `## Source`), and its diffs only remove the heading and blank lines;
   - `bob records tidy -d -f json` parses;
   - `bob records new … -d` for a no-notes record prints a body ending at the H1.

> [!decision] first_cleanup = now After the spot check, run `bob records tidy` once on
> the live vault. Confirm the single `records: tidy (N notes)` commit and that a second
> `bob records tidy -d` is a no-op. With `nightly`, do not run the real command; report
> the dry-run count and leave the cleanup to `bob nightly`.

Commit with a conventional subject such as
`feat(records): write ## Notes only when record notes have bullets`. This is not a CLI
or JSON contract break, so it takes no `!`. Never edit the `Cargo.toml` version or
`CHANGELOG.md`.

## Out of scope

- **bob-plugins.** No change. The record sheet anchors after the H1 and never reads
  `## Notes`.
- **Bob Mac Capture.** No change. Its `Tests/Fixtures/record-capture-*.json` files are
  static samples that still parse, and the app renders whatever `body` bob returns.
- **SASE memory.** No change. The `glossary:record-note` strand already says the
  `## Notes` bullets are optional, and no decision record covers this rule.
- **Other note kinds.** Albums, notebooks, Gmail, and period notes keep their own
  `## Notes` behavior.
