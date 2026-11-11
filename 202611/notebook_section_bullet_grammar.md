---
tier: tale
title: Grammar 26 notebook section bullets
goal:
  Complete bob-cli-ka.2 with section-bullet parsing, correct notebook placement, shared
  seed planning, editor metadata, docs, and regression coverage.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ka.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ka.2
create_time: 2026-11-11 10:34:36
status: wip
---

- **PARENT:**
  [202611/notebook_section_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/notebook_section_bullets.md)
- **BEAD:**
  [bob-cli-ka.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ka/bob-cli-ka.2.md)

# Grammar 26: notebook section bullets

Implement the **grammar** phase assigned to `bob-cli-ka.2`, from
`plan:202611/notebook_section_bullets.md`. A route-only notebook list or a
`@parent^id@ %n` item must accept `- #Section` with nested one-line notes, write those
notes into that H2 section, and keep plain siblings in the top region regardless of
their position in the draft. Deliver the parser, writer, editor protocol, documentation,
and regression coverage together.

## Scope and verified starting point

At planning time the clean checkout includes `f9d998c1`, the completed `bob-cli-ka.1`
resolver phase. It already provides:

- `resolve_notebook_section` with Existing/Create/NearMiss/Ambiguous, typed-capitals
  creation titles, and duplicate-title/slug refusal;
- `staged_notebook_sections` and draft provenance for earlier whole items;
- a policy-aware completion entry point that suppresses the task drawer probe for
  notebook notes;
- notebook list `note(s)` wording and the guarded `plan_notebook_note` writer with
  section placement, creation, and per-note JSON metadata.

`bob-cli-ka.2` has no earlier-work notes. Its grammar work has not landed:
`GRAMMAR_REVISION` is 25, `claim_list_notebook` returns only one note per list bullet,
`seed_body` writes `%n` bullets in draft order, and the provisional seed catalog
recognizes only ALL-CAPS sections. There are no `--epic-symbol` entries assigned to this
phase at planning time. Recheck the branch and bead notes before implementing; preserve
landed work.

This is one medium tale because the resolver, transaction, catalog maintenance, and
ordinary note writer already exist. The remaining work is one coupled grammar change in
bob-cli, with a shared body plan for its two entry points. Keep the grammar bump and all
observable changes in the same implementation, avoiding partially enabled syntax between
phases.

The parent epic's decisions are final: typed capitals stay enabled; the later menu phase
uses `create_row_return = preselect_clean`. Decision record edits were rejected. The
glossary authorization belongs to the epic's memory phase; this tale edits no memory
files. Menu rows/chrome, the slash alias, Mac code/fixtures, and memory are later
phases. Do not implement their interfaces here beyond the shared grammar and staged-view
inputs they need. `@@` behavior and the separate issues `bob-cli-k9` and `bob-cli-k8`
remain outside this phase. Add no CLI verbs or flags.

## Observable contract

```text
@dev_tools
- #Git
  - Use worktrees ^worktrees
- #"API Design"
  - Preserve meaning across versions
- #docker
  - Pin base images
- Prefer fd over find
  - Keep this detail with the top note
```

The final file contains the plain note and its detail before the first H2, then the
section notes in their sections. Each section child becomes one stamped note, and its
optional trailing `^id` is preserved. The selector line never becomes a note or a
checkbox. Existing H2 order stays unchanged; missing sections append at EOF in
first-mention order; repeated sections merge notes in source order. The same rules apply
to a new notebook seeded by `%n`.

| Position of first-level bullet                                | Leading `#` behavior                                          |
| ------------------------------------------------------------- | ------------------------------------------------------------- |
| Route-only `@nb` notebook list                                | Section bullet                                                |
| `@parent^id@ %n` creation body                                | Section bullet                                                |
| List headed by `@nb#Section`                                  | `section_bullet_position` refusal                             |
| Detail under `Text @nb`, `@nb+id`, or an `@@nb` note          | Opaque detail; no section/drawer interpretation or diagnostic |
| Under a new task, project note, or note-less task `@route+id` | Existing drawer behavior                                      |
| `@area` / `@project` task list                                | Existing `drawer_bullet_position` behavior                    |

The opener is an ASCII letter, straight/curly opening double quote, or bare `#` before
whitespace/end of line. `#1`, `##`, and `#-` remain prose. Selectors use glued
notebook-section spelling, not drawer slug rules: `#Git`, `#API-keys`, `#coc.nvim`,
`#"API Design"`, and `#“API Design”`. Unquoted create words containing capitals keep
those capitals, so `#API-keys` creates `API Keys`; quoted titles stay verbatim. ALL-CAPS
first-level bullets with children permanently remain section seeds in `%n` only, with
the existing Title Case convention. In `@nb` lists they remain plain notes with detail.
`- \#git` writes a literal `#git` note.

| Input                                                      | Interactive parse                                                | Strict capture / dry-run                                    |
| ---------------------------------------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------------- |
| Bare `- #`, `Note @nb#`, or `@nb#` header                  | `incomplete`, need `section`, `interactive_placeholder` over `#` | `notebook_section_required`                                 |
| Complete section selector without real nested notes        | `incomplete`, need `section_notes`, no error                     | `empty_section_bullet`                                      |
| Text or a `^id` after the section selector on its own line | Precise diagnostic                                               | `section_bullet_text`                                       |
| Section bullet under `@nb#Section`                         | Precise diagnostic                                               | `section_bullet_position`                                   |
| Third-level line under a section child                     | Precise diagnostic                                               | `section_bullet_depth`                                      |
| Near miss or duplicate winning match                       | Existing shared resolver diagnostic                              | `notebook_section_near_miss` / `notebook_section_ambiguous` |

Use the parent epic's teaching copy. Same-line text teaches nesting or `\#` for a
literal tag. Wrong-position copy names the actual header and resolved section and tells
the user to remove the header's section. Depth copy teaches a `@dev_tools#git` header
list for a note with detail; for `%n`, name the new route. No empty sections are
written. Ordinary note details remain opaque and retain their existing depth rules.

## Implementation

1. **Introduce the notebook section-bullet scanner and shared body plan.** Add a focused
   module beside `capture_language/drawer_bullet.rs` and register it in
   `capture_language/mod.rs`. Return Complete, Incomplete, Invalid, or Escaped with
   absolute UTF-8 byte ranges, the decoded selector/exact flag, and the consumed
   span/remainder. Reuse the opener and `lex_exact_name_after_hash` behavior and the
   glued selector grammar in `tokens.rs`; do not use drawer title validation or create a
   second quoting dialect. A caret outside the selector's quotes counts as forbidden
   same-line content, while a caret inside quotes is title text. Share lexical
   primitives with the glued path where needed.

   Build a date-independent seed/body representation in capture_language from the
   existing `CaptureItem` physical lines and authored-line classification. Retain
   original offsets, line numbers, selector ranges, per-note IDs, and raw detail. It
   must represent ordinary top notes and ordered section groups, plus incomplete/error
   outcomes for the editor. Strict `%n` planning collects top notes in source order
   before sections in first-mention order and merges repeated groups through
   `resolve_notebook_section`. Treat ALL-CAPS compatibility groups as exact formatted
   titles resolved against that same catalog. Near misses/ambiguities within a new
   notebook refuse the create.

   Inspect four-space third-level lines while their owning section group is known,
   before the generic classifier turns them into `invalid_child_line`. Do not broaden
   the global authored-depth grammar. Empty child placeholders are unfinished typing,
   never real notes.

2. **Claim route-only notebook section lists and fix bare hashes.** Preserve
   `draft.rs::push_list_items`' original-source slicing: each first-level list bullet is
   already an item whose first line is its stripped body. Extend the notebook
   member-claim path so a section bullet emits one `ParsedCaptureItemOutcome` per nested
   note, modeled on `fanout_notebook_plus`, with
   `CaptureKind::NotebookNote { section, exact_section, block_id }`. Preserve list
   context and exact child source ranges; set the existing fanout fields to the selector
   line, 1-based position/count of its notes, and the authored marker such as `#Git`.

   Put section claiming ahead of generic marker/depth checks that would otherwise
   interpret the selector as a task type. Preserve checkbox, task-only marker,
   duplicate-ID, and notebook ownership protections on actual note bodies. Keep escaped
   selectors literal through writing. Nonsection list bullets continue through
   `finish_notebook_claim` and retain nested detail. Under a section-specific header
   return the position refusal; beneath a single notebook note keep `#` detail opaque.

   The exact bare-hash bug has two entry points: `tokens.rs` reports an empty suffix
   without `incomplete_section`, so `scan_route_marker` loses `section_range`;
   `draft.rs` represents `@nb#` as Notes with `section: None`, and list claiming
   currently treats it as top-region placement. Preserve the hash's presence and range
   in both paths and refuse it only under notebook policy. Check it before generic
   empty-list errors, with and without child notes. Keep ordinary non-notebook `@area#`
   behavior intact.

3. **Use one `%n` seed plan and extend the staged view by source offset.** Replace the
   parallel derivations in `notebook_create.rs::{seed_body,seed_sections}` and
   `notebook.rs::{notebook_create_seed_sections, provisional_create_sections}` with the
   shared plan. The writer only adds dates/formats bytes from that plan; parsing,
   provisional route discovery, and section staging use the same structure. Validate the
   new groups in `refuse_notebook_create_children` without routing them through the task
   drawer machinery. Preserve `%n` summary/parent validation and its existing
   note-file/catalog output.

   Evolve `staged_notebook_sections` from whole-item origins to accepted section
   operations carrying their own offsets. It must show disk H2s first, then valid
   earlier creates/bullets in source order, deduped by the shared resolver. Earlier
   bullets of the current `%n` item must be visible before the caret while that bullet
   and later bullets stay excluded. Retain disk/draft provenance and meaningful origin
   offsets. Reject failed completed items from the staged catalog. For an item currently
   being edited, evaluate its valid earlier body prefix so an unfinished selector under
   the caret does not erase its earlier groups. Avoid recursive staging -> claim ->
   staging calls: use shared lexical body/claim primitives and a source-order fold of
   accepted operations. A later `More @dev_nvim#lsp` must resolve a section seeded
   earlier in `%n`, and the existing glued menu must see the same catalog.

   Continue passing list notes through `plan_notebook_note`, which resolves against
   staged file bytes and already handles section order, EOF creation, and top-region
   insertion. Keep the existing batch transaction and preimage checks for notebooks, new
   files, owner catalogs, and metadata reads; completion/parse data is never write
   authority.

4. **Expose accurate editor state and additive span metadata.** Update
   `editor_view_for_item`, `notebook_create_editor_view`, their editor-outcome adapter,
   and `editor_model::Need` so the table above reports incomplete mode, `section` /
   `section_notes`, or precise diagnostic ranges. Emit `NotebookSection` spans covering
   `#` through the selector, with the resolved/to-be-created title. Carry an optional
   structured `section_action` (`existing` or `create`) through the internal editor
   representation and `CaptureParseSpan` serialization; add it to glued notebook section
   spans as well. Resolve it at the selector's source offset, not against a catalog
   containing that same operation. Preserve `key` semantics and omit the new field on
   unrelated spans. Audit span-copy, sorting, enrichment, and wire paths in
   `capture_parse.rs` so metadata is not lost. All notebook editor outcomes must retain
   `drawer_bullet_target: None`.

   Keep capture JSON's existing shape: each emitted note has `notebook.section`,
   `notebook.section_action`, and existing fanout context.
   `print_human_notebook_success` already prints the section and new-section wording;
   ensure fanout ordinals are correct and list summaries count emitted notes rather than
   section-marker bullets.

5. **Bump grammar and document the complete behavior.** Set
   `capture_language::GRAMMAR_REVISION` to 26. Update only assertions and fixtures
   pinning grammar 25, not unrelated byte offsets or historical prose. Keep schema
   version 1 for the additive JSON fields. Update `docs/capture.md` (Notebook notes,
   authored bullets, parse needs/diagnostics/spans), `docs/notebooks.md` (Capture,
   including a draft and the resulting file), `docs/reference/capture-syntax.md`
   (section bullets and escape), and `capture_parse.rs` long help (scope the old
   assertion that `- #` always names a drawer). Document the permanent `%n` ALL-CAPS
   path, sibling placement, and one-line section notes. Avoid promising the later menu
   or alias before those phases land. Run `just docs-gen` and review its generated diff.

   This changes authored meaning and must land as `feat(capture)!: ...`. Do not edit the
   Cargo.toml version or CHANGELOG.md; the daily release train owns both.

## Verification and completion

Use the existing Rust and CLI notebook fixtures, adding a focused section-bullet test
module if that keeps the cases readable. All vault mutation tests use temporary vaults
and a fixed `BOB_NOW`.

- Unit tests: every scanner hit and opener exception; straight/smart and escaped quotes;
  UTF-8 byte ranges; both claim paths; every row of the position/refusal tables;
  placeholder vs empty commit; IDs on notes vs rejected IDs on selectors; precise
  third-level diagnostics; ordinary note-detail opacity and ALL-CAPS compatibility.
- Shared-plan/staging tests: plain siblings before/between/after groups, repeated `#`
  groups, mixed ALL-CAPS/`#` merges, existing section order, first-mention creation
  order, earlier/later/failed operations, and offset snapshots inside the same `%n`
  item. Include quoted/unquoted near misses and ambiguous titles/slugs, preserving
  resolver-phase tests.
- CLI tests in `tests/cli/capture/notebook_notes.rs`, `notebook_create.rs`, and parse
  tests: verify resolved titles/actions, needs and span ranges, fanout metadata, human
  `n/N` and total note count, `%n` plain-sibling regression, later same-draft writes,
  and the glued completion catalog. Every invalid draft refuses both dry-run and commit
  without changing any target or owner catalog, even if earlier draft items were valid.
- Text/file cases: `#API-keys`, mixed case, `#coc.nvim`, quoted emoji and non-Latin
  headings, smart/escaped quotes, CRLF/BOM source files, fenced fake H2s, and trailing
  note IDs. Compare predicted note lines/file output with committed bytes, and `%n`
  dry-run `note_files.markdown` directly with committed contents. Dry-run itself changes
  no files.
- Atomicity tests at the existing planner/commit seam: build a mixed draft including
  notebook notes, a new notebook, and owner-catalog edits; retype or rename a
  destination after policy/planning and before commit. The stale preimage must abort all
  writes and preserve the intervening external change. Exercise live re-resolution
  rather than treating a previous parse result as an authority token.
- Regression tests: task drawers, area/project lists, forced destinations, notebook
  sub-notes, the existing create behavior, and notebook note bullets opening no drawer
  menu. Section-menu row round trips and Mac rendering remain later phases'
  responsibility.

Run focused library capture-language/notebook tests and CLI capture tests while
developing, then `just docs-check` and `just check`. Follow the SASE monitor/finalizer
skills for checks that need a handoff; do not end with a live foreground subprocess. The
previous resolver phase's clean-base highlights failure was fixed by `7eb82440`, already
present in this starting tree; do not assume that old failure remains applicable. If
another failure reproduces identically on the clean base, record its evidence as
`PROPOSED FOLLOW-UP:` on `bob-cli-ka.2` and follow the user's instruction to complete
the phase despite that unrelated baseline issue.

Do not create task beads. Record any discovered out-of-scope work with
`sase bead note bob-cli-ka.2 'PROPOSED FOLLOW-UP: ...'`. Before closing, run
`sase bead epic-symbols bob-cli-ka.2` and resolve any entries (or re-key legitimate
later work to a still-open owner). Once the whole scope is verified, close only
`bob-cli-ka.2` with a note naming the implemented behavior and checks. Never hand-set
its status, reopen it without the user's specified closed-bead host notice, or close
`bob-cli-ka` or any ancestor plan bead. Finish through `/sase_final`.
