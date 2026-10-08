---
tier: epic
title: Migrate zorg-era reading records into the reference library
goal: '`bob ref migrate-zorg` moves every unindexed zorg-era reading record (424 on
  2026-10-07) into legacy reference notes under `ref/zorg/<hub>/`. It runs as a dry-run-first,
  idempotent vault write: one sync-sandwiched commit that a single `git revert` undoes.
  Book chapters fold into their book''s note. On athena, `bob ref doctor` then reports
  `coverage: ok`, `bob ref find` / `bob ref list` return the migrated records, the
  JSON `coverage.scope` caveat no longer claims that zorg history is invisible, and
  bob-cli-4x is closed.

  '
phases:
- id: record-model
  title: Shared zorg record parser, multi-block mirroring, and book reading state
  depends_on: []
  size: medium
  description: 'record-model: turn coverage.rs''s counter into a shared zorg record
    parser (owner block, ID/LID, block range, fields). Make mirroring id-aware and
    let one note mirror several blocks (`source_blocks`). Derive a legacy book''s
    reading state from `legacy_chapter_statuses`. Keep doctor''s live count at 424,
    then test and document the rules.'
- id: planner
  title: bob ref migrate-zorg dry-run planner and report
  depends_on:
  - record-model
  size: medium
  description: 'planner: add `bob ref migrate-zorg` (dry run by default). It plans
    one legacy note per record, with books folding their chapters. It picks collision-free
    stems, ref_type, title, URLs, and tags, renders the notes exactly, and prints
    a human or JSON report: counts by file and status, renamed targets, records without
    a URL, identity hits, already migrated, and skipped. Add CLI tests, help and completion
    updates, and docs.'
- id: writer
  title: Reversible --write path, rollback runbook, and scope caveat
  depends_on:
  - planner
  size: medium
  description: 'writer: add `-w/--write`. Under bob_sync.lock it pre-syncs, re-plans,
    and refuses existing targets. It then writes new files only, verifies them through
    the index and coverage, and deletes its own files on any failure. Then one scoped
    commit and a post-sync. Test it against temp git vaults, document the rollback,
    and update the coverage.scope caveat.'
- id: live-run
  title: Run the migration on athena and verify coverage
  depends_on:
  - writer
  size: small
  description: 'live-run: build the landed master on athena and dry-run against ~/bob.
    Check every number against the expected table and stop for Bryan on any deviation.
    Run --write, verify doctor, find, list, and show plus an idempotent rerun, install
    the build, and record the evidence and rollback sha on the bead.'
proposed_by: bbugyi200.athena.bob-cli-5k.7
parent_bead: bob-cli-5k.7
create_time: 2026-10-07 16:17:59
status: done
bead_id: bob-cli-5k.7.1
---

- **PROMPT:** [prompts/202610/zorg_ref_migration.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/zorg_ref_migration.md)
- **PARENT:** [202610/close_top_ten_impact_beads.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/close_top_ten_impact_beads.md)
- **BEAD:** [bob-cli-5k.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5k/bob-cli-5k.7.1.md)

# Plan: Migrate zorg-era reading records into the reference library

## 1. Why this epic exists

bob-cli-4x asks for the zorg-era reading records that live outside `ref/` to move into
the reference library. Until they do, `bob ref find` / `bob ref list` cannot see that
reading history. Every ref JSON envelope's `coverage.scope` has to say so, and
`bob ref doctor` warns `coverage: warn (~424 …)`. Epic bob-cli-4w built the read-only
library and deliberately scoped this bulk vault write out. This epic is the
Bryan-reviewed migration it deferred. It is the nested epic of phase `ref-migration`
(bob-cli-5k.7) of epic bob-cli-5k.

Bryan settled the design on 2026-10-07. Section 3 states each decision as a rule.
Section 2 records the live inventory the decisions were made against. Read both before
starting any phase.

## 2. Live inventory (athena, ~/bob, 2026-10-07)

These numbers come from a read-only script that mirrors `coverage.rs` exactly. The
`live-run` phase checks the real dry run against them.

- **Records:** 424 unmirrored `status::` records in 37 hub files, all at the vault root.
  The top files are work_ref 75, nvim_ref 69, clean_arch 37, dev_ref 27, prj_yserve 18,
  ad_tech_book 17, soft_arch_hard_parts 16, and zorg_ref 14. Every record has an owner
  `^z-` list line.
- **By status (all records):** READ 135, REVIEW_FLEETING_NOTES 96,
  COLLECT_FLEETING_NOTES 92, UNREAD 57, REVIEW_LIT_NOTES 17, ABANDONED 16, BOOK 11.
- **Owner tokens:** 323 records carry `ID::`, and 101 carry `LID::`.
- **Where the LID records live:** all 101 sit in 9 book notes: clean_arch 36,
  ad_tech_book 16, soft_arch_hard_parts 15, balance_coupling 9, system_for_writing 9,
  cat_theory_for_devs 5, how_to_read_a_book 5, outlive 3, think_fast_and_slow 3. Each of
  these files holds exactly one BOOK record.
- **The extra chapter:** system_for_writing.md also holds one `ID::chapter_1` record
  (`^z-250323-0b`) whose `| BOOK: [[system_for_writing#^z-250321-0o|…]]` line ties it to
  that book. Under rule 3.4 it folds in as a chapter. That gives **102 folded chapters
  and 322 notes**.
- **BOOK records without chapters:** two, work_clean (books.md) and war_of_art
  (gtd_ref.md).
- **Shared owner blocks:** two pairs of records share one owner block id in the same
  file. In prj_gbd, `^z-250425-0f` owns both `bidder_declarations_prd` and
  `cs_bidder_dec`. In zettlr_ref, `^z-250326-0r` owns both `zettlr_wtf_is_zettelkasten`
  and `zettlr_projects`. So `(source_path, source_block)` alone is not unique; the
  `ID::` value tells each pair apart.
- **ID collisions with vault notes:** 17 IDs already name a Markdown file somewhere in
  the vault (case-insensitive, including `_generated/` and `lit/`):
  - the 9 book IDs (`clean_arch` and so on), which match their own hub notes;
  - programming_in_lua, work_clean, practical_vim, modern_vim (`lit/*.md`);
  - ilar, yserve, hyperlist (`_generated/tag_pages/**`);
  - pat_chat (`chat/pat_chat.md`).

  `<ID>_ref` is free for all 17. No two records share an ID, and no ID matches an
  existing ref note's stem.

- **URLs:** most records carry one inline http(s) `url::` (128 of them are internal
  `http://go/…` links, which pass `validate_and_clean`). 14 records hold several URLs as
  nested list items under an empty `url::` line. 12 records have no usable URL (`NONE`,
  `MULTIPLE`, a wikilink, a `~/` path, or no field at all). One of them is the folded
  `chapter_1`, so **11 notes are written without `url`**.
- **file:: kind for the 322 notes:**

  | file:: prefix | Count |
  | ------------- | ----- |
  | lib/docs      | 159   |
  | lib/blogs     | 57    |
  | lib/papers    | 48    |
  | lib/chat      | 12    |
  | lib/books     | 11    |
  | lib/slides    | 8     |
  | lib/code      | 6     |
  | lib/forums    | 5     |
  | lib/notes     | 1     |

  The other 16 have no `lib/…` path (NONE, AUDIBLE, `[[books/…]]`) and fall back to
  `zorg`. The folded chapter has no file.

- **Identity hits:** 0 normalized-URL matches against existing ref notes. bobdoto_ref
  has 4 records that cite one page URL, each for a different section.
- **Precedent:** sase-4b.1 (vault commit a478dd9) mirrored 282 AI records into
  `ref/ai/<hub>/<ID>.md`. Those records stay mirrored, and the planner reports them as
  `already_migrated`.

Expected reading states of the 322 new notes:

| State    | Count | Made up of                                                       |
| -------- | ----- | ---------------------------------------------------------------- |
| finished | 183   | READ 109, REVIEW_FLEETING_NOTES 60, REVIEW_LIT_NOTES 12, 2 books |
| started  | 69    | COLLECT_FLEETING_NOTES 62, 7 books                               |
| queued   | 52    | UNREAD 52                                                        |
| dropped  | 16    | ABANDONED 16                                                     |
| unknown  | 2     | work_clean, war_of_art                                           |

The 2 finished books are clean_arch and system_for_writing. The 7 started books are
ad_tech_book, balance_coupling, cat_theory_for_devs, how_to_read_a_book, outlive,
soft_arch_hard_parts, and think_fast_and_slow.

## 3. Accepted design (rules every phase follows)

### 3.1 Scope: every record

All unmirrored records migrate, including internal work docs (go/ links), file-only
records, and chats. The only allowed residue is a record the planner reports as
`skipped` with a reason (section 5.2). With today's vault, nothing is skipped, and
coverage reaches 0.

### 3.2 Layout and file names

- **Path:** each note goes to `<ref-dir>/zorg/<hub stem>/<stem>.md`. The hub stem is the
  source file's stem (`work_ref.md` → `work_ref`).
- **Stem:** the `ID::` value, unless that stem (compared case-insensitively) already
  names a Markdown file anywhere in the vault or another note planned in the same run.
  The scan for existing names covers every directory except hidden ones, so it includes
  `_generated/` and `ref/`.
  - On a collision the stem becomes `<ID>_ref`, then `<ID>_ref_2`, `<ID>_ref_3`, and so
    on.
  - Plan records in a stable order (source path, then owner line) so the choice is
    deterministic.
  - The report lists every renamed note and the existing file it avoided. Today that is
    exactly the 17 IDs in section 2.

  This keeps `[[clean_arch]]`-style hub links, and the new notes' own `parent` links,
  unambiguous in Obsidian.

### 3.3 Note shape (one per non-chapter record)

Frontmatter is written in this exact key order. Every scalar is a YAML double-quoted
string produced by `serde_json::to_string`; integers are bare. Lists use block style
(`  - "…"`).

| Key                       | Value                                                                                                                                                                                                                                                                  |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `parent`                  | `"[[<hub stem>]]"`                                                                                                                                                                                                                                                     |
| `type`                    | `"[[ref]]"`                                                                                                                                                                                                                                                            |
| `tags`                    | `zorg/reference`, then each whitespace token on the owner line that matches `^#[A-Za-z][A-Za-z0-9_/-]*$`, without the `#`, in order, deduplicated. Zorg ids like `250619#0T` start with a digit and are not tags.                                                      |
| `ref_type`                | Where the first `file::` value is a wikilink whose target starts with `lib/<kind>/`, the kind is `<kind>` (docs, blogs, papers, chat, books, slides, code, forums, notes, …). Otherwise `zorg`. `chat` makes the row's origin `agent-report`, the same as `ref/chat/`. |
| `status`                  | `"legacy"`                                                                                                                                                                                                                                                             |
| `legacy_status`           | The raw status, lowercased (`"review_fleeting_notes"`, `"book"`)                                                                                                                                                                                                       |
| `title`                   | See title rule below                                                                                                                                                                                                                                                   |
| `url`                     | Usable URLs only (rule 3.6). A scalar when there is one, a list when there are several, omitted when there are none.                                                                                                                                                   |
| `source_note`             | `"[[<hub stem>]]"`                                                                                                                                                                                                                                                     |
| `source_block`            | The owner block id with its `^` (`"^z-250419-09"`)                                                                                                                                                                                                                     |
| `source_id`               | The `ID::` value                                                                                                                                                                                                                                                       |
| `source_path`             | The vault-relative source path (`"work_ref.md"`)                                                                                                                                                                                                                       |
| `source_line_start`       | 1-based owner line                                                                                                                                                                                                                                                     |
| `source_line_end`         | 1-based last line of the record block                                                                                                                                                                                                                                  |
| `source_blocks`           | Books only: each folded chapter's owner block id, in source order                                                                                                                                                                                                      |
| `legacy_chapter_statuses` | Books only: each folded chapter's raw status, lowercased, in the same order                                                                                                                                                                                            |

**Record block.** The record block runs from the owner line to the last line before the
next non-blank line whose indent is at most the owner's indent. Trailing blank lines are
excluded.

**Title rule.** Use the record's `title::` field when present, with wikilinks reduced to
their display text. Otherwise humanize the ID:

- split it on `_` and `-`;
- capitalize each word;
- uppercase every word in this fixed acronym set: ai, api, cli, gtd, html, http, json,
  llm, lsp, mcp, pdf, prd, sql, ui, url, ux, yaml.

The precedent's `awesome_mcp_servers` → "Awesome MCP Servers" is a regression vector.

The body follows the sase-4b.1 precedent:

````markdown
# <title>

Original block: [[<hub stem>#<source_block>|<hub stem> <source_block>]]

## Files

- [[lib/docs/awesome_mcp_servers.pdf]]

## Related

- LINKS: [[mcp_ref#^z-250411-0p|model_context_protocol]]

## Chapters (books only; see 3.4)

## Original Record

Dataview inline field markers are escaped in this copy so the page-level fields stay
canonical.

```text
<the record block verbatim, with every `::` written as `\:\:`>
```
````

- **`## Files`** lists each `file::` value that is a wikilink, in order. Omit the
  section when there is none.
- **`## Related`** lists each `| KEY: value` line of the owner block as `- KEY: value`,
  in order. Omit the section when there is none.
- **Escaping.** Every `::` copied into the body outside the code fence is also written
  as `\:\:`, so no copied text creates a Dataview field on the ref note.
- **Code fence.** The fence is three backticks, or one more than the longest backtick
  run inside the copied text.
- **No tracker, no PDF field.** No `^ref` tracker and no `source_pdf` are ever written,
  so Highlights scan and sync never touch these notes.

### 3.4 Books: one note per book, chapters folded in

- **Which file is a book.** A source file that holds exactly one BOOK record is a book
  file.
- **Which records are chapters.** A record in a book file is a chapter of that book when
  either:
  - its owner line carries `LID::`; or
  - its owner block has a `| BOOK:` line whose wikilink targets the BOOK record's block.

  Today that is 101 LID records plus system_for_writing's `chapter_1`.

- **Chapters get no note of their own.** The book's note gets:
  - `source_blocks` and `legacy_chapter_statuses` (3.3), in source order;
  - a `## Chapters` section, placed before `## Original Record`, with one line per
    chapter: `- [[<hub>#<block>|<LID or ID>]] <chapter title> · <status lowercased>`.

    The chapter title is the chapter's `title::` field when present. Otherwise it is the
    owner-line text after the `LID::`/`ID::` token, with the block id removed and `::`
    escaped.

  - an `## Original Record` fence that holds the BOOK record block and then each
    chapter's record block, in source order, separated by one blank line.

- **No owning book.** An `LID::` record in a file with zero or several BOOK records is
  reported as `skipped` (`unassigned_chapter`) and stays counted by doctor. Today there
  are none.
- **Reading state** (implemented by `record-model`, not stored). It applies to a legacy
  note whose `legacy_status` folds to `book` and that has a non-empty
  `legacy_chapter_statuses`:
  - map each chapter status through the existing legacy mapping, then set aside the
    `unknown` and `dropped` ones;
  - if every remaining chapter is `finished`, the book is `finished`;
  - else, if any is `started` or `finished`, it is `started`;
  - else, if any remain, it is `queued`;
  - else, if any chapter was `dropped`, it is `dropped`;
  - otherwise it is `unknown`.

  `reading_state_source` is `legacy_chapters:<finished>/<total>`. A BOOK record with no
  chapters stays `unknown` via `legacy_status:book`.

### 3.5 Status: legacy era

Nothing new is stored about status. Every note is `status: legacy` plus the raw
`legacy_status`. The index derives reading state with the bob-cli-4w mapping:

| Legacy status                                 | Reading state                           |
| --------------------------------------------- | --------------------------------------- |
| unread                                        | queued                                  |
| collect_fleeting_notes                        | started                                 |
| read, review_fleeting_notes, review_lit_notes | finished                                |
| abandoned                                     | dropped                                 |
| book                                          | unknown, or derived from chapters (3.4) |

No `^ref` tracker is written, so the Dashboard reading queue (`refs.base`) and the
default `bob ref list` queue stay clear of the 2025 backlog. The latter already
collapses legacy notes into its `+ N zorg-era legacy notes hidden` line.

### 3.6 URLs

Candidates come from two places, kept in order:

- the first whitespace token of the inline `url::` value
  (`http://go/dart-devtools (and others)` → `http://go/dart-devtools`);
- the first token of each nested list item directly under a `url::` line.

A candidate is usable when `validate_and_clean` accepts it. Unusable values (`NONE`,
`MULTIPLE`, `-`, wikilinks, `~/` paths) are dropped from frontmatter. They survive in
the Original Record, and the report lists every note written without a URL. As a result,
no migrated note earns an `opaque_url` diagnostic.

### 3.7 Identity, dedupe, and idempotency

**Provenance key.** A record is already migrated when some indexed ref note has:

- the same `source_path`; and
- either:
  - the same `source_block` and a `source_id` that equals the record's ID (or the note
    has no `source_id`, as tolerated for older notes); or
  - the record's block listed in its `source_blocks`.

The planner and doctor share this one rule (`record-model`). Reruns are therefore
no-ops, and the two shared-block pairs stay distinct.

**Identity hits.** URL, arXiv, and DOI matches against existing notes are reported, but
each record still migrates as its own note. The index's existing find and supersession
rules handle the overlap.

### 3.8 Hub lines untouched

The migrator never edits a source hub note. Every `[[hub#^z-…]]` link keeps resolving,
and coverage subtracts mirrored records. The vault write only adds files under
`ref/zorg/`, so one `git revert` rolls it back.

### 3.9 Command surface

`bob ref migrate-zorg` is a permanent `bob ref` subcommand. It sits in the `Library`
help group, sorted alphabetically: find, list, migrate-zorg, show. Its about line is
"Migrate zorg-era status:: reading records into ref/zorg/ (dry run unless --write)".

Options, sorted with short aliases:

- `-b/--bob-dir` (shared builder)
- `-f/--format human|json` (default human)
- `-o/--offline`: skip both vault-sync cycles and commit locally without pushing
- `-r/--ref-dir` (shared builder)
- `-w/--write`: apply the migration

The bare command is a read-only dry run: no lock, no sync, no writes. After the live
migration it keeps working and reports 0 to migrate. Read `cli_rules.md` with
`/sase_memory_read` before touching the CLI.

## 4. Phase `record-model`: shared record parser, multi-block mirroring, book state

Goal: one record model that doctor and the migrator both use, so the migrator's
inventory always equals doctor's count.

1. **Record parser.** In `src/native/ref_library/coverage.rs`, factor the existing
   counting walk into a public-in-crate record parser. Keep the current file walk,
   status regex, and owner rule unchanged. Each `ZorgRecord` carries:
   - `source_path` (vault-relative, `/`-separated);
   - status line (1-based) and canonical uppercase status;
   - owner block, owner line (1-based), and owner indent;
   - `id: Option<String>` from an `ID::` token and `lid: Option<String>` from an `LID::`
     token on the owner line;
   - the record block line range (3.3);
   - the `| BOOK:` target block when present.

   Records without an owner are still produced, with `owner_block: None`.
   `count_zorg_records` becomes a thin count over unmirrored parsed records.

2. **Mirroring rule.** Replace the `(source_block, source_path)` set with the provenance
   rule in 3.7. `RefRow` gains two `#[serde(skip_serializing)]` fields,
   `source_id: Option<String>` and `source_blocks: Vec<String>`, read from frontmatter
   (block ids normalized to a leading `^`, as today). Update every `RefRow` literal in
   tests.
3. **Book state.** In `status.rs`, apply the 3.4 derivation in the `legacy` branch. Read
   `legacy_chapter_statuses` with `ParsedFrontmatter::get_all`.
4. **Tests** (unit, plus a doctor CLI fixture in `tests/cli/highlights/`):
   - a shared-block pair where one record is mirrored and the other is not (count 1);
   - a book note with `source_blocks` subtracting its chapters;
   - the `source_id`-less fallback;
   - every branch of the book derivation, including `legacy_chapters:` source strings;
   - `ID::` / `LID::` / `| BOOK:` extraction;
   - the record block range with internal blank lines.

   Existing coverage and doctor assertions must keep passing.

5. **Docs.** Add a `book` row note to the reading-state table in `docs/ref.md`. Update
   the mirroring sentence of the `coverage` bullet in `docs/highlights-ref-sync.md`.
6. **Live check (read-only).** Build and run `bob ref doctor` against `~/bob`. It must
   still report `~424` with the same per-file counts. Record the row in the bead note.

## 5. Phase `planner`: dry-run planner and report

Goal: `bob ref migrate-zorg` (no `--write`) computes the full migration and shows it,
without writing anything.

### 5.1 Code shape

- Put the code in a new `src/native/ref_library/migrate_zorg/` module (for example
  `plan.rs`, `render.rs`, `report.rs`, `cli.rs`).
- Wire it into `all_subcommands`, `HELP_GROUPS`, and `run` dispatch in
  `src/native/highlights_ref/{cli,mod}.rs`.
- Planning is a pure function of:
  - the parsed records;
  - the built index rows;
  - the vault's Markdown stem set;
  - each source file's lines.

  It returns planned notes, each with the target path, the exact rendered contents, and
  the metadata the report needs.

- Rendering follows sections 3.2–3.6 exactly.

### 5.2 Classification

Every parsed record lands in exactly one bucket:

- `already_migrated` (3.7);
- `skipped`, with a reason: `no_owner`, `no_id` (an owner line with neither `ID::` nor
  `LID::`), or `unassigned_chapter`;
- `chapter` (folded into a planned book note);
- `note` (gets its own planned note).

Invariant, asserted in tests: `note + chapter + skipped` equals doctor's unmirrored
count for the same vault.

### 5.3 Report

**Human output** (styled with `Styler`, plain under `NO_COLOR` or a pipe):

- a headline:
  `bob ref migrate-zorg · dry run · N records in F files → M notes under ref/zorg/`;
- by status;
- by file (all files, count descending);
- books, with chapter counts and derived state;
- renamed stems, with the colliding vault file;
- notes without a URL;
- identity hits (key → existing note);
- already migrated (count only);
- skipped (path:line and reason);
- a final line: `coverage after --write: K unindexed (K = skipped)`.

**JSON** (`-f json`): compact one-line JSON, no ANSI, with `generated_at` pinned by
`BOB_NOW` like the other ref verbs:

```json
{"ok":true,"schema_version":1,"command":"ref migrate-zorg","generated_at":"…","mode":"dry_run",
 "summary":{"records":424,"notes":322,"chapters":102,"already_migrated":282,"skipped":0,
   "renamed":17,"no_url":11,"identity_hits":0},
 "by_status":{"READ":135,…},"by_file":[{"path":"work_ref.md","records":75},…],
 "notes":[{"path":"ref/zorg/clean_arch/clean_arch_ref.md","title":"Clean Arch","ref_type":"books",
   "source_path":"clean_arch.md","source_block":"^z-250619-0t","source_id":"clean_arch",
   "legacy_status":"book","reading_state":"finished","urls":["https://…"],"chapters":36,
   "renamed":{"from":"clean_arch","avoids":"clean_arch.md"}},…],
 "identity_hits":[],"skipped":[],"commit":null}
```

`records` counts unmirrored records, the same as doctor. `renamed` is `null` when the
stem is the ID. `reading_state` is computed by running the rendered note through the
index row builder. That proves the rendered frontmatter parses the way the index reads
it.

### 5.4 Tests, help, and docs

- **Fixture vault tests** in `tests/cli/ref_library/` (temp vaults only):
  - a plain record;
  - multi-URL and unusable-URL records;
  - a tag-bearing owner line;
  - a stem collision with a vault note and with `_generated/`;
  - the shared-block pair;
  - a book with LID chapters plus an `ID::` chapter tied by `| BOOK:`;
  - a BOOK record without chapters;
  - an already-mirrored precedent note;
  - an `unassigned_chapter`;
  - human and JSON output;
  - golden rendered notes for one plain record and one book, byte for byte.
- **No-write test:** assert that the dry run leaves the vault byte-identical and takes
  no lock.
- **Help and completion:** update the `bob ref -h` snapshot
  (`tests/fixtures/help/ref-short.txt`) and any other help or completion snapshot or
  table the new subcommand touches. `every_value_arg_has_a_decision` must pass.
- **Docs:** add a `## Migrating zorg-era records (bob ref migrate-zorg)` section to
  `docs/ref.md` that covers the rules in section 3 and the report. Also add the command
  to the docs' command lists.
- **Live sanity (read-only):** run the dry run against `~/bob` and compare it with
  section 2. Note any difference on the bead; do not fix the vault.

## 6. Phase `writer`: reversible --write path

Goal: `--write` applies exactly what the dry run shows, as one revertible commit, or
changes nothing.

Mirror `bob task reroll`'s live flow (`src/native/randomize.rs` `execute`):

1. Acquire `bob_sync.lock` with `ob::acquire_lock_waiting`. Use reroll's 60 s default
   budget and its waiting message. No new flag.
2. Require a git worktree. A vault that is not one fails `--write` with a clear error:
   reversibility depends on git.
3. Unless `--offline`, run `vault_sync::run_cycle_with_existing_lock_report`. This
   pre-sync commits pending edits separately, so the migration commit holds only the new
   files. A failed pre-sync aborts with reroll's `--offline` hint.
4. Re-plan from disk under the lock. If nothing is left to migrate, print
   `nothing to migrate` and exit 0 without committing or post-syncing.
5. Refuse before writing anything when any planned target path already exists. Also
   refuse when `git status --porcelain -- <ref-dir>/zorg` is non-empty.
6. Write each note with create-new semantics: a temp file in the target directory,
   fsync, a no-clobber rename, then a re-read that compares bytes.
7. Verify. Rebuild the index and coverage. Every planned path must yield a row with no
   `invalid_yaml`, `opaque_url`, or `missing_type` diagnostic. The row must carry the
   planned `reading_state`. Coverage must drop by exactly `notes + chapters`.
8. On any failure in steps 6–7, delete every file this run created, plus any directory
   it created that is now empty. Exit 1 without committing.
9. Commit exactly the written paths with `ob::commit_paths`.
   - Subject:
     `bob ref migrate-zorg: <records> records into <notes> notes under ref/zorg`.
   - Body: a by-status line, the top files, and the rollback command.
10. Unless `--offline`, run the post-sync cycle.

The report's `mode` becomes `write`, and `commit` holds `{sha, subject, paths}`.

**Tests** (temp git vaults; set `BOB_VAULT_SYNC_LOCK_FILE` and the state file like
`tests/randomize.rs`):

- `--offline` writes the planned bytes and makes exactly one commit containing only
  `ref/zorg/**`;
- a second run is a no-op, with no commit;
- a pre-existing target refuses with nothing written;
- a forced verification failure leaves the tree byte-identical with no commit;
- a non-git vault refuses;
- a local bare remote exercises the pre- and post-sync sandwich;
- `git revert` of the commit restores the pre-migration tree and doctor count.

**Docs and caveat:**

- Add the write flow and a rollback runbook to the `docs/ref.md` migration section. The
  runbook: find the sha with
  `git -C ~/bob log --format=%h --grep='^bob ref migrate-zorg' -1`; check that no sync
  is running (`bob vault-sync status --json`); run
  `git -C ~/bob revert --no-edit <sha>`; then `bob vault-sync`. If the revert hits
  conflicts because migrated notes were edited since, say so.
- Add a one-line pointer from `docs/vault-git-sync.md` `## Rollback`.
- Replace `Coverage::scope_text()` with: "Only notes under ref/ are indexed. Zorg-era
  status:: reading records outside ref/ are not until `bob ref migrate-zorg` moves them
  into ref/zorg/; run `bob ref doctor` to count any that remain. Absence from this index
  is never proof that something was not read." This is true both before and after the
  live run. Update `docs/ref.md`'s `## Coverage` paragraph and any test that pins the
  old text.

## 7. Phase `live-run`: migrate ~/bob on athena

This is the only phase that writes the live vault.

1. **Build.** From current master, run `cargo build --release`. Use that binary for
   every step below.
2. **Pre-checks:** `bob vault-sync status --json` is healthy, and
   `git -C ~/bob status --short` is clean apart from what the pre-sync will commit. Save
   the vault's HEAD sha.
3. **Dry run.** Run `bob ref migrate-zorg -f json` and the human form. Compare against
   section 2:
   - `records` equals the current `bob ref doctor` count, and 424 is expected;
   - notes 322, chapters 102, skipped 0, identity hits 0;
   - renamed is exactly the 17 IDs listed;
   - no_url 11;
   - per-file and per-status counts match;
   - the reading-state totals match the table in section 2.

   **On any deviation, stop.** Ask Bryan with `/sase_questions`, showing the diff, and
   do not write. Deviations come from vault edits since 2026-10-07 or a planner bug.

4. **Write.** Run `bob ref migrate-zorg --write`. Record the commit sha and subject.
5. **Verify with the new binary:**
   - `bob ref doctor` prints
     `coverage: ok (no unindexed zorg-era reading records outside ref/)`, and
     `library diagnostics` gained no `opaque_url`/`missing_type`/`invalid_yaml` rows
     from `ref/zorg/`;
   - `bob ref find http://go/cs-bidder-dec https://vimhelp.org/quickfix.txt.html -f json`
     returns `in_library` for both, and the shared-block pair are two distinct notes;
   - `bob ref list -s legacy -t books -A` lists the book notes with their derived
     states;
   - `bob ref show clean_arch_ref` renders the `## Chapters` section;
   - a second dry run reports 0 to migrate;
   - `git -C ~/bob show --stat <sha>` touches only `ref/zorg/**` (322 files).
6. **Install.** Run `just install` so the installed `bob` carries the new coverage
   rules. Otherwise the installed doctor would still count the 102 folded chapters.
   Re-run the installed `bob ref doctor`. Other machines pick up the notes through vault
   sync, and their binaries need the same update.
7. **Record.** Post a bead note with the commit sha, the doctor coverage row, the
   find/list/show evidence, and the exact rollback command.

## 8. Rules for every phase

- **Read first.** Read your bead with `sase bead read <id> -r "<why>"`. Read the
  `obsidian` reference memory, `cli_rules.md`, and `docs/ref.md` before changing code.
  Line numbers in this plan drift, so re-locate them.
- **Gate.** Before closing, run `just check` (fmt, clippy, and every test binary with
  `--no-fail-fast`, added by bob-cli-5k.3). If master does not have that recipe yet, run
  `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
  `cargo test --no-fail-fast`.
- **Base-tree failures.** A failure that reproduces identically on the clean base tree
  is recorded as a `PROPOSED FOLLOW-UP:` bead note and does not block the phase.
- **Live vault.** Only `live-run` writes to `~/bob`. Other phases may run read-only
  commands against it (doctor, the dry run). Every test uses temp vaults.
- **Hub notes.** Never edit a hub note, and never hand-write ref notes; the migrator is
  the only writer.

## 9. Landing checklist (for this epic's land agent)

1. Read each phase bead. Each must be closed with its verification note, and
   `live-run`'s note must carry the commit sha and the coverage row.
2. On athena, with the installed `bob` built from the final master:
   - `bob ref doctor` reports `coverage: ok`;
   - `bob ref migrate-zorg` reports 0 to migrate;
   - `bob ref find http://go/cs-bidder-dec` is `in_library`;
   - `bob ref list -s legacy -t books` lists the books;
   - `bob ref find … -f json` shows the new `coverage.scope` text;
   - `just check` passes.
3. Close bob-cli-4x with a note citing the doctor row, the migration commit, and the
   rollback command.
4. Copy that evidence onto bob-cli-5k.7 with `sase bead note`. If bob-cli-5k.7 is still
   open once this epic is closed, close it with the same note: its only remaining
   criterion is this epic landing. Never close bob-cli-5k.
5. Turn phase `PROPOSED FOLLOW-UP:` notes into task beads with `/sase_new_task`.

## 10. Out of scope

- Editing, linking from, or deleting the zorg hub records.
- Migrating the sase-4b.1 `ref/ai/` notes to the new layout or adding `ref_type` to
  them.
- Improving book titles beyond the title rule. Bryan can retitle the notes by hand after
  the migration.
- Giving migrated notes a `^ref` tracker or Dashboard queue membership.
- Updating bob binaries on apollo or the MacBook.
