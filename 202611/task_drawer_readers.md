---
tier: tale
title: Dual-read task drawers in Rust (drawer-contract)
goal:
  Every bob-cli vault reader recognizes both `- **Title**` and legacy ALL-CAPS task
  drawers with their authored titles, imported text can never become a drawer, and every
  header writer and capture draft grammar stays byte-for-byte unchanged.
size: medium
bead: bob-cli-ph.1
proposed_by: bbugyi200.apollo.bob-cli-ph.1
proposed_with: claude/opus
status: done
---

- **BEAD:**
  [bob-cli-ph.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ph.1](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ph.1.md)
- **COMMITS:**
  - [37b2080](https://github.com/bobs-org/bob-cli/commit/37b2080e9efca170f542954f088ce4169d430cfa)
    — feat(drawers): add dual-read task drawer readers

# Plan: Dual-read task drawers in Rust

## Scope and authority

This tale completes the assigned phase `bob-cli-ph.1` (`drawer-contract`) of epic
`plan:202611/true_case_drawers.md` (bead `bob-cli-ph`). Before you write code, read
these three sources with audited reads:

- that epic plan, especially design-of-record §§1–4 and §8 and the `drawer-contract`
  phase steps;
- `research:202611/true_case_drawers/true_case_drawers.md`;
- `sase bead read bob-cli-ph.1`.

If `sase artifact read` reports the research ref as `missing`, run
`sase repo open bob-cli--research -r "<why>"` once and read it again. Where the epic and
the research differ, follow the epic: readers ship first, writers and capture drafts
stay on the legacy contract, and imports defuse both legacy ALL-CAPS and whole-bold
text.

**History.** An earlier planner proposed the same scope as
`plan:202611/dual_read_task_drawers.md`. It was approved on 2026-11-23, but no coder
ever ran: no transcript, no notes, and no commits exist. This plan refreshes that one
against master `2652b794`. Since that plan's base (`7703011d`), four commits landed:

- requirement depth and state annotations on capture preview and agenda lines
  (`capture/block_diff.rs`, `capture/task_blocks.rs`, `capture_pomodoros_agenda.rs`);
- the requirement criterion glyph;
- two docs commits.

None of them adds a new drawer predicate. Their requirement annotations read through
`task_reqs::req_line_map`, so they inherit whatever the `task_reqs` reader decides.

**Preconditions, already verified.** `bob-cli-on` is closed (done). Its landing commit
`7703011d` fixed the `^prj` Notes home: it reuses `## Notes` with column-0 bullets and
lands vectors RL53–RL60 and RW54–RW57. Do not redo epic step 0; keep these tests green.
The docs registry currently holds 56 contracts, and `tools/docs` contract tests pass on
this base.

**Epic decisions are final.** `drawer_glyph = tray` does not touch this phase. The two
memory decisions belong to the later `memory` phase, so this tale edits no memory.

**Out of scope:**

- plugin code or plugin fixture copies (`plugin-readers` owns them);
- the Mac app, vault migration, and host rollout;
- capture draft syntax, `GRAMMAR_REVISION`, and spans or preview annotations;
- the `rename-drawer` NEW casing rules;
- bold-header creation;
- the Cargo version and `CHANGELOG.md`.

Record discovered work only as `PROPOSED FOLLOW-UP:` notes on `bob-cli-ph.1`. Create no
beads, and close no ancestor.

## Result and compatibility boundary

Take a task with these direct children:

```markdown
- [ ] #task Example ^example
  - NOTES
    - old note
  - **API Design**
    - new note
  - **Empty One**
```

`bob capture-drawers` and every vault reader recognize all three drawers, keep their
exact titles (`NOTES`, `API Design`, `Empty One`), and report the markers `legacy_caps`,
`bold`, and `bold`.

These are never drawers:

- `__API Design__`;
- a bold lead ending in a colon (`**Note:**`);
- a bold body with trailing text, an `^id`, a tag, or a field outside the closing
  delimiters;
- managed lines and reserved names;
- checkbox lines;
- lines inside fences or quotes;
- whole-bold bullets nested below a note, a drawer, or a requirement.

A `**Notes**` header is the existing Notes home, matched by key. Adding a note must not
create a second `NOTES` header or rewrite the existing one.

New headers keep today's legacy bytes (`- NOTES`, ALL-CAPS drawer creation). New project
sections keep today's casing. Capture drafts, completion rows, every existing
writer-vector expectation, and `GRAMMAR_REVISION` stay unchanged.

Only two kinds of bytes change on purpose:

- imported prose that would form a drawer header is written as `__…__`;
- because the gcal span hash covers rendered bodies, untouched Google spans re-render on
  their next sync.

## Implementation

### 1. One recognition contract in `capture_drawers`

`capture_drawers` stays the public owner. If its 3.3k-line scanner gets crowded, move
the new recognizer into a child module, for example `capture_drawers/recognize.rs`.

- **Decouple Pomodoro names first.** Give `is_pomodoro_name` its own predicate before
  touching the drawer grammar. Its ASCII alphabet and punctuation behavior stay exactly
  as they are: `C++` and `A+B` are accepted, `WHY?` and `OPTIONS:` refused. Its existing
  tests must pass untouched.
- **Rename the legacy predicate.** Today's `is_drawer_title` becomes
  `is_legacy_drawer_title`, with its grammar kept byte for byte: a trailing `:` is
  allowed, while internal colons, non-ASCII letters, and `+` are refused.
  - Draft parsing (`capture_language/drawer_bullet.rs`, `editor_parse.rs`,
    `editor_classify.rs`), `capture_project_note::is_project_section_title`,
    `normalize_drawer_literal`, and legacy creation validation keep calling the legacy
    predicate in this phase.
  - Do not alias the old name to the new recognizer, because that would silently widen
    draft grammar.
- **Body shape, `drawer_body_title(body) -> Option<(title, marker)>`.** This is a pure
  check: it needs no vault and no context. It returns `bold` when the trimmed body is
  exactly `**title**` and the title qualifies:
  - nonempty and single-line;
  - no whitespace just inside either delimiter;
  - no `*` anywhere;
  - at least one Unicode alphanumeric character;
  - does not end with `:`.

  Internal colons and punctuation (`Q1: Goals`, `C++ Notes`, `Q&A`, `iOS / OAuth 2.0`,
  `Café`) are valid. The check returns `legacy_caps` when the trimmed body passes
  `is_legacy_drawer_title`. A body starting with a checkbox is never a drawer body.
  `__title__` is always a note.

- **Contextual recognizer, `recognize_drawer_header`.** This is the only entry point
  vault readers use. A line is a drawer header only when all of these hold:
  - It is a direct list-item child of a real `#task` line, by today's
    `direct_child_list_indexes` / `note_tasks` rules.
  - It is outside fences (including list-item fences, the same interpretation
    `task_children` and requirements use) and outside quotes.
  - The managed-line matchers have run first and none matched: managed task logs, Cancel
    Log, Depends-On, BLOCKS, LINKS / LINKED FROM, and SUB-PROJECTS.
  - Its key is not reserved.
  - `drawer_body_title` qualifies the body.

  Children are optional for recognition. The legacy form passes through the same
  contextual guard, which is no broader than today.

- **Reserved keys.** Use one shared set: today's managed-owner keys, plus `TASKS`, plus
  every name `is_structural_h2_key` knows (Tasks, the status groups, Pomodoros, Events,
  Highlights, and the generated catalogs such as Projects and Notebooks). A reserved
  bold title is a note.
- **Key.** `drawer_key` collapses whitespace, then applies Unicode `str::to_uppercase`.
  Pin `Straße` → `STRASSE` and `Café` → `CAFÉ`.
  - Update every key comparison and query fold that does its own local ASCII upper, such
    as `reserved_selector_key` and `drawer_query_match`.
  - Keep ASCII transforms whose job is to create legacy titles, or that serve Pomodoro,
    task-type, or notebook contracts.
  - Titles are always the authored bytes, never rebuilt from a key.
- **Model.** `Drawer` gains `marker: DrawerMarker { Bold, LegacyCaps, Section }`, a
  `header_range`, and a `title_range`. Both ranges are half-open UTF-8 byte offsets into
  the supplied source. `header_range` covers the full header line without its line
  ending. `title_range` covers the title alone, without `**` or the `## `.
  - `project_h2_drawers` reports `Section` with accurate ranges, keeping its existing
    eligibility and insertion geometry.
  - Test the ranges under LF, CRLF, tabs and spaces, and multibyte titles.
- **Notes role.** Split `NOTES_KEY = "NOTES"`, used for every comparison, from
  `NOTES_TITLE = "NOTES"`, used by writers and unchanged in this phase. Keep the landed
  `NOTES_SECTION_TITLE = "Notes"` handling for `^prj` notes. A later phase,
  `bob-writers`, flips `NOTES_TITLE`.
- **Creation policy, `can_create_drawer`.** Keep it separate from recognition. Title
  validation returns named refusals: `not_a_drawer_title` (empty, contains `*`, ends in
  `:`, no alphanumeric) or `reserved`. Creation additionally requires note content. Do
  not wire it into capture drafts or header emitters yet.
- **DH helpers.** These are pure mappings, unused by writers in this phase:
  - `heading_for_drawer` returns a bold title verbatim, or today's
    `format_project_section_title` result for `legacy_caps`.
  - `drawer_for_heading` keeps a valid title verbatim and returns `not_a_drawer_title`
    or `reserved` otherwise. An empty `##` section maps to a valid empty drawer.

### 2. Retarget every vault reader

Move each vault reader onto the contextual recognizer. Do not use a line-only check,
because a nested bold note must never become a drawer.

| Surface                                                            | Adaptation                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `capture_drawers.rs`                                               | `drawers`, `parse_drawer_title`, catalog `scan_drawer_note`, existing-target matching in `resolve_drawer_for_write`, key and query folds, marker and range construction, JSON listing.                                                                                                      |
| `task_children.rs`                                                 | `TaskChildKind::Drawer { title }` carries the unwrapped authored title, after the checkbox and managed checks. Direct-child and fence filtering stay authoritative.                                                                                                                         |
| `task_reqs.rs`                                                     | `is_structural_container` recognizes bold headers, so no requirement exists under them and `req_in_drawer` refuses there. `req_line_map` and the new preview and agenda req annotations then omit bold drawers and their content, exactly as they do for legacy ones.                       |
| `task_block_layout.rs`, `task_block_layout/{tidy,writers}.rs`      | Notes-home lookup by `NOTES_KEY` (`drawers(..).find(key == NOTES_KEY)`), zone classification, existing-home reuse, and the `writers.rs:518` `req_in_drawer` parent refusal. Header creation bytes stay the same.                                                                            |
| `task_split/{plan,carry}.rs`                                       | Drawer scope and labels use the unwrapped title from `TaskChildKind::Drawer`. A deeper bold note stays splittable note content.                                                                                                                                                             |
| `task_merge/containers.rs`                                         | Read and select both kinds of container. Mutation geometry and legacy creation stay the same.                                                                                                                                                                                               |
| `task_drawer_rename/{mod,links}.rs`                                | `old_bullet_lines`, owner detection, lookalikes, `header_kind`, and content-preservation accounting recognize both forms. Existing NEW validation, the same-key refusal, and legacy retitling bytes stay the same; rewriting a bold OLD header in place is the later `drawer-rename` phase. |
| `task_tidy/mod.rs`                                                 | Find the Notes home by key. Keep writing the legacy `- NOTES` header.                                                                                                                                                                                                                       |
| `capture/plan.rs`, `capture_complete/{drawers,transfer_assist}.rs` | Classify each use. Vault lookups take the recognizer; draft-grammar checks stay on the legacy predicate.                                                                                                                                                                                    |

Then audit the rest with `rg` across `src/native` for `is_drawer_title`,
`parse_drawer_title`, `NOTES_TITLE`, literal `- NOTES`, and `to_ascii_uppercase` near
drawer code. Sort every remaining caller into one of three groups, and say in the commit
body which group each one is in:

- vault recognition (retarget);
- legacy draft or creation validation (keep);
- unrelated title grammar (keep).

`gcal/description/bullets.rs` is handled in §3.

**Ambiguity.** Existing-target lookups return the source title and marker and keep
today's ambiguity handling. `NOTES` beside `**Notes**`, or Unicode case twins, is
`ambiguous_drawer` naming both lines; it never merges silently.

Test Notes-home selection and matching through the real capture, tidy, split, and merge
paths, not only with isolated lexical checks.

**Catalog.** Bump `DRAWER_CATALOG_VERSION` from 2 to 3 and keep marker provenance per
spelling. The same title can occur as both bold and legacy, so storing one marker per
key, or letting a later occurrence overwrite a title's marker, is not enough. Keep
occurrence counts, distinct-note counts, recency, and deterministic tie-breaks. A v2
cache rebuilds.

Do not apply any explicit-spelling preference yet; that is the `capture-drawers` phase.
Add a test proving that catalog v3 cannot make existing creation callers emit a
mixed-case bare header. The catalog may now hold a `Future Work` spelling, but
`#future-work` must still create `- FUTURE WORK`.

**CLI.** `bob capture-drawers` JSON drawer objects gain an additive `marker` field
(`bold`, `legacy_caps`, or `section`). Add it to the recent-drawer objects too, if those
represent a chosen source spelling. Existing fields, `SCHEMA_VERSION`, and verbatim
human titles stay the same. Add no options. Read the `cli_rules.md` reference memory
with `/sase_memory_read` before touching the command.

### 3. Imported text never becomes a drawer

Add `task_text_escape::defuse_drawer_header(body)` beside
`escape_bullet_start_checkbox`. When `drawer_body_title` qualifies a body as `bold` or
`legacy_caps`, the helper returns `__title__`: it strips only the `**` markers and keeps
the title bytes. Everything else is returned unchanged:

- existing `__…__`;
- invalid or reserved bold leads;
- mixed prose and URLs;
- non-header bodies.

The helper must be idempotent and must keep the surrounding layout whitespace. Run it
after the other escapes (checkbox, field, task-token, block-ID), and never unescape text
that is already inert.

Apply it to every imported child body at any depth, matching today's gcal defuse, which
ignores depth. Apply it only after inline markup is assembled, and never to metadata or
to a whole serialized document. The call sites:

- **gcal, `gcal/description/bullets.rs::render_line`.** Replace the `is_drawer_title` →
  `**…**` branch with the defuse of the rendered body, so a plain `AGENDA` lands as
  `__AGENDA__` and `<b>Agenda</b>` lands as `__Agenda__`.
  - Keep managed-line italicization and URL handling.
  - Owned-span hashes must pick up the change, and private or user-edited span ownership
    must stay as it is.
  - Update the existing `!is_drawer_title(body)` test assertions (near lines 639
    and 787) to assert against the production recognizer.
- **gkeep and highlights, through `escape_child_text`.** Defuse the final rendered child
  body, and confirm that truncation or wrapping cannot recreate a header afterwards.
- **Gmail, `gmail/body/inert.rs`.** Its serializer does not use `escape_child_text`.
  - Guard completed list-item and child bodies after `Inline::Strong` rendering.
  - Audit the task-child embedding call sites too.
  - Leave normal mail headings, links, code, and mid-paragraph emphasis unchanged.
  - Add behavioral coverage for a plain ALL-CAPS list item and for a list item the
    renderer turns into whole bold.
- **`%c` clipboard, `capture_clip/render.rs::rendered_lines`.** Guard the final inline,
  lines, list, and history child bodies.
  - Keep attachments, code snippets, list depth, and raw authored capture text.
  - Today `%c` does not escape checkboxes, and the requirements contract does not list
    it as an outside writer. Do not add checkbox escaping here. If you confirm that a
    clipboard `[ ] x` line becomes a requirement, record a `PROPOSED FOLLOW-UP:`
    instead.
  - Test real committed clipboard notes, not only rendered output, so a later
    normalization cannot undo the escape.

**What the tests must show.** Take externally produced child text, place it below a real
task, and scan it with the production drawer reader: it produces no drawer. For the
gcal, gkeep, Gmail, and highlights sources, which already escape checkboxes, it also
produces no requirement. Cover `%c` both into a new task and into an existing task that
owns requirements. Draw the adversarial titles from the DV vectors, and assert on the
final bytes and on the scanner results, not only on helper return values.

### 4. Conformance fixtures and regressions

Create the Rust-owned `tests/fixtures/task_drawers/vectors.json` and its `README.md`.
Give each case a stable ID and document the schema. The README says plugin twins update
bob-cli first, then copy the file byte for byte. Load the fixtures with the existing
RQ/RL/SP harness style; do not add a second test framework.

- **DV (recognition).** Each case records title, key, marker, ranges, and recognized or
  excluded. Cover at least:
  - bold and legacy headers, empty headers, ordinary mixed-case prose, and `__…__`;
  - whitespace just inside the delimiters, empty and non-alphanumeric titles;
  - internal versus trailing colons, and an embedded `*`;
  - trailing prose, `^id`, tag, or field;
  - every reserved and structural family, and every managed line;
  - checkboxes, quotes, and plain and list-item fences;
  - direct versus deeper nesting, and nested task ownership;
  - tabs and spaces, LF and CRLF;
  - Unicode folds and same-task case twins.

  Pin the preserved legacy grammar: `OPTIONS:` is accepted, while `NOTE: X`,
  `KEY:: VALUE`, and `OPTIONS::` are refused. Pin `is_pomodoro_name` unchanged.

- **DH (title mapping).** Byte-identical round trips for `API Design`, `C++ Notes`,
  `Q1: Goals`, `Café`, and internal whitespace. Legacy Title Case mapping. Empty
  sections. Named refusals for reserved and invalid titles. Test the round trip on its
  own, outside any project transformation.

Append bold-reader cases to the RQ, RL, and SP fixtures in `tests/fixtures/task_reqs/`
and `tests/fixtures/task_split/`, and update their READMEs. Cover Notes by key, drawer
zones and tidy, requirement exclusion and `req_in_drawer`, and split scope and labels.
Keep every existing writer expectation, including RW54–RW57 and the legacy `NOTES`
output, byte for byte.

Add focused Rust and CLI regressions for:

- discovery JSON `marker` values, verbatim human output, and the v2 → v3 cache rebuild;
- appending a note to a bold `**Notes**` drawer with no second header;
- ambiguous bold/legacy pairs and Unicode twins;
- split, merge, and rename lookups, including content-preservation checks;
- imported ALL-CAPS and whole-bold text from each of the five sources;
- capture preview and agenda req annotations under a bold drawer;
- unchanged new-header bytes and unchanged draft parse and completion, including legacy
  creation when the catalog also holds mixed-case explicit spellings;
- unchanged Pomodoro names, ordinary emphasis, and `^prj` Notes-home behavior.

Do not touch the plugin fixture copies. Record the fixture and API details on the phase
bead for `plugin-readers`.

### 5. Contract page, docs, and gates

Read the `docs.md` reference memory with `/sase_memory_read` and follow its registry
workflow.

- **The contract page.** Create `docs/task-drawers.md` for:
  - recognition, identity and the key, and Notes by key;
  - the dual-read window;
  - the outside-text rule;
  - the pure DH mapping;
  - byte-range conventions and the JSON `marker` values;
  - the rule that capture creation stays nonempty.

  Separate shipped reader behavior from the later writer and capture phases. Do not
  teach `- **Title**` as capture draft syntax yet. Leave the broader project/task
  isomorphism out of this contract.

- **Registry.** Register the page in `docs/_meta/contracts.yml` with a unique order.
  Regenerate `docs/develop/contracts.md`, `docs/README.md`, mkdocs nav, and llms.txt
  with `just docs-gen`. Update the pinned count in `tools/docs/tests/test_contracts.py`
  (56 → 57, unless other work has landed) and its README-table digest from the generated
  output.
- **Links and references.** Link the page from `docs/capture.md`,
  `docs/task-requirements.md`, and `docs/reference/vault-format.md`. Extend the
  requirements page's "Pasted text never becomes a requirement" sentence with "or a
  drawer". Document the additive `marker` field in `docs/reference/json-outputs.md` and
  in the CLI reference. Change importer docs only where the new escape makes them false.
  Respect page budgets and mounted-repository ownership.

**Gates.** Run focused suites while you work. Then run `just check`, `just docs-gen`,
`just docs-check`, and the docs contract tests. Hand any command that may outlast a tool
timeout to `/sase_monitor`, and wait for it to finish.

Do not broaden this tale to fix unrelated failures. If a failure reproduces identically
on the clean base, record the exact evidence as a `PROPOSED FOLLOW-UP:` on
`bob-cli-ph.1` and continue.

## Completion

The commit is additive, readers-first, and has no `!`:
`feat(drawers): recognize bold task drawers alongside legacy headers`.

Add a note to `bob-cli-ph.1` recording:

- the implemented API names and fixture case-ID ranges, for `plugin-readers`;
- test outcomes and any clean-base failures;
- evidence that both on-disk forms are read and that imports are inert;
- evidence that writer bytes and draft grammar stay on the legacy contract;
- evidence that the `^prj` Notes-home vectors still pass.

Run `sase bead epic-symbols bob-cli-ph.1`. Resolve each leftover symbol, or re-key it to
the still-open parent epic or a later phase. Then close only this phase with
`sase bead close bob-cli-ph.1 --note "<verified evidence>"`. Never close `bob-cli-ph` or
any ancestor. End the turn through `/sase_final`.
