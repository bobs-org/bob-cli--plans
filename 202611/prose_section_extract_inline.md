---
tier: epic
title: Extract prose sections into prose notes and inline them back
goal: "One Obsidian gesture turns an authored section of an area, project, or prose note
  into a prose note, and the same gesture folds it back. bob prose extract and bob prose
  inline own the conversion in Rust: previewed, preimage-guarded, all-or-nothing,
  link-repairing, crash-recoverable, and lossless on a round trip. Extraction records
  provenance that the later reversible-projects work can consume.

  "
decisions:
  keymap:
    ask: Which gesture should extract a section and inline it back?
    choices:
      new_chord:
        Ctrl+Shift+E toggles prose sections; Ctrl+Shift+Alt+N keeps its task/project
        meaning
      overload:
        Ctrl+Shift+Alt+N also extracts on section lines and inlines in prose notes
    default: new_chord
    why: Keeps the project gesture's refusals intact; Ctrl+Shift+E is verified unbound
    answer: new_chord
  stub_form:
    ask: What should extraction leave under the heading it keeps?
    choices:
      link: One `- [[stem|Title]]` bullet that renders as a plain link
      embed:
        One `- ![[stem]]` bullet that reads in place but repeats the promoted headings
    default: link
    why: Renders cleanly and stays a one-note drawer for later project reversal
    answer: link
  intro_prose:
    ask: How should extraction treat text above a note's first heading?
    choices:
      refuse:
        Refuse with a hint to type a heading first, so every extraction has a real
        heading
      overview:
        Extract it under a synthesized `## Overview` stub that inline removes again
    default: refuse
    why: One typed heading does the same job and keeps every round trip heading-anchored
    answer: refuse
  prose_catalog:
    ask:
      "Add a generated ## Prose catalog to areas and projects, with the reversal
      catalog-skip fix?"
    default: false
    why:
      Stubs already show extracted notes in place, and Ctrl+= finds children by parent
    answer: false
  memory_prose_section:
    ask: Add a Prose Section glossary term to SASE memory?
    memory:
      - glossary:prose-section
    default: false
    answer: false
phases:
  - id: txn-trash-recovery
    title: Trash files and recover interrupted vault rewrites
    depends_on: []
    size: medium
    description:
      "txn-trash-recovery: vault_rewrite gains a guarded trash operation, no-replace
      creates, and a durable intent journal that rolls back or refuses an interrupted
      rewrite, with crash tests at every write point."
  - id: section-model
    title: Prose section selection and eligibility
    depends_on: []
    size: medium
    description:
      "section-model: a pure prose-module classifier that resolves --section and --line
      to a heading subtree in an area, project, or prose note and returns the named
      refusal for every ineligible case."
  - id: link-relocation
    title: Relocate links around a moved section
    depends_on:
      - section-model
    size: medium
    description:
      "link-relocation: a relocation map and vault-wide rewriter that repairs links in
      every direction for extract and its inverse, preserving aliases, embeds, and link
      form."
  - id: extract
    title: bob prose extract planner and hidden CLI
    depends_on:
      - txn-trash-recovery
      - section-model
      - link-relocation
    size: medium
    description:
      "extract: the extract planner, the ProseConversion v1 JSON types, the fingerprint,
      and a hidden bob prose extract that writes through vault_rewrite."
  - id: inline
    title: bob prose inline and the round-trip guarantee
    depends_on:
      - extract
    size: medium
    description:
      "inline: the inline planner and bob prose inline, both commands unhidden together,
      and a property test that inline(extract(S)) restores S."
  - id: contract-docs
    title: Contract docs, host probe, and vault-clone dogfood
    depends_on:
      - inline
    size: small
    description:
      "contract-docs: document the conversion contract in docs/prose.md and the
      reference pages, add the prose-extract-preview host probe, round-trip real
      sections on a throwaway vault clone, and apply the accepted memory decision."
  - id: keymap
    title: Bob Prose extract and inline gesture in Obsidian
    depends_on:
      - inline
      - contract-docs
    size: medium
    description:
      "keymap: bob-articles gains extract, inline, and toggle commands with a preview
      modal driven only by bob's JSON, bound per the keymap decision, with Command Mode
      rows, docs, tests, marker, and deploy."
  - id: prose-catalog
    title: Optional generated Prose catalog
    depends_on:
      - inline
    size: medium
    description:
      "prose-catalog: only when prose_catalog is accepted, add a generated Prose catalog
      through host_catalog and fix project reversal to skip every generated catalog;
      otherwise close as a no-op."
proposed_by: bbugyi200.apollo.bob-cli-qj.5
proposed_with: claude/opus
parent_bead: bob-cli-qj.5
decided_by: auto
create_time: 2026-11-27 07:18:25
status: wip
---

- **PROMPT:**
  [prompts/202611/prose_section_extract_inline.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/prose_section_extract_inline.md)
- **PARENT:**
  [202611/prose_note_type.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/prose_note_type.md)

# Plan: Extract prose sections into prose notes and inline them back

## Context

Bryan renamed the `article` note type to `prose` (epic `bob-cli-qj`,
`plan:202611/prose_note_type.md`) so that a section inside an area, project, or prose
note can implicitly be a **prose section**, which one keymap turns into a full prose
note and back. This epic builds that capability. It carries the research report's Phase
2 (extract and inline). Phase 3, reversible projects, is out of scope, but the
provenance designed here is what Phase 3 will read.

**Required reading for every phase.**

- `research:202611/prose_rename_and_section_extraction/prose_rename_and_section_extraction.md`
  (read it with `sase artifact read`): "Phase 2 extract and inline", adjustments A2, A3,
  A5, and A6, and "What extraction leaves behind in the source note". The researcher
  appendices `__cld` (rules E1–E5 and I1–I4), `__cdx` (link-repair table and acceptance
  matrix), and `__gem` (refusal list) back the tables below.
- `docs/prose.md`: the prose-note contract (heading identity, N6/N9 number ownership,
  F-rules, word invariant, write safety).
- The [Conversion contract](#conversion-contract) section of this plan. It binds every
  phase. Where a phase needs to change it, record why on the phase bead.

Bryan endorsed every requirement in the report. The parent epic restated them as binding
(see "prose-sections" in `plan:202611/prose_note_type.md`), and this plan carries all of
them.

## Verified current state (bob-cli `16a2497d`, bob-plugins `c6fbeb36`)

- **The prose engine** lives in `src/native/prose/` (`cli.rs`, `scan.rs`, `sections.rs`,
  `links.rs`, `inbound.rs`, `marker.rs`, `plan.rs`, `vault.rs`, …). `bob prose` has
  `adopt`, `format`, `list`, and `outline`; `bob articles` is a hidden alias.
  ArticlePlan stays `schema_version: 1`. The plugin `bob-articles` (display name Bob
  Prose) formats through `--stdin` plans and is the only ArticlePlan client.
- **The Markdown classifier.** `prose/scan.rs` `classify` gives every line a `Region`
  (frontmatter, fence, unclosed fence, math, comments, HTML, quote, list item and
  continuation, footnote continuation, indented code, table, paragraph, ATX heading,
  Setext underline). Headings inside callouts and list or footnote bodies never count
  (N1). `prose/sections.rs` `plan_headings` returns headings with their original levels.
  Multi-line Setext headings are left out of that list. No helper returns subtree ranges
  yet. The only subtree walk is in `links.rs` `resolve_heading` (`:414`), which resolves
  `#A#B` nested paths. The `scan`, `sections`, and `marker` modules are private to
  `prose`, so the new code lives inside `src/native/prose/`.
- **Identity.** `sections.rs` `strip_section_number`, `heading_identity`, and
  `heading_plain_identity` implement the documented rule. Capture's `@note#Section`
  targeting uses them for prose notes (`capture/sections.rs:236`).
- **Bob-owned structure.** `capture_drawers.rs:698` `is_structural_h2_key` covers Tasks,
  Next & In Progress, Blocked, Done & Canceled, Pomodoros, Events, Highlights, Snoozed,
  and every heading in `host_catalog::GENERATED_CATALOGS` (Areas, Notebooks, Albums,
  Projects; `host_catalog/region.rs:113`). `host_catalog::owned_regions`
  (`region.rs:458`) returns the bob-owned callout lines. The records index is a separate
  spec (`albums/index.rs:57`). `task_status_groups::reserved_group_heading_lines`
  (`mod.rs:377`) finds status groups under any Tasks heading. A project note's
  `## Notes` section is where bob writes `^prj` notes (`task_block_layout.rs:70`).
- **Body refusals.** `prose/marker.rs:392` `body_refusals` reports conflict markers,
  `^prj`/`^ref` anchors, and `#task` checkbox lines, with masking. It already works on
  any slice of classified lines. It is private.
- **Note kinds.** `projects/scan.rs` has `frontmatter_is_area`,
  `frontmatter_is_project`, `frontmatter_is_prose`, `frontmatter_is_notebook`,
  `frontmatter_is_album`, and `frontmatter_is_record`. Daily and period notes are
  recognized by `intentions::subject::NoteSubject::canonical`.
- **Transactions.** `vault_rewrite::run` (`src/native/vault_rewrite/mod.rs:122`) runs
  lock, worktree check, pre-sync, plan, stale check, dirty rules, atomic writes with
  rollback, verify, one commit, and post-sync. `FileRewrite` supports rewrites and
  creates, but not removals. Its before-images live only in memory, so a crash mid-write
  leaves half a rewrite that the next online sync would commit. Creates are checked with
  `exists()` and then renamed, so a file that appears in between is overwritten.
  Precedents:
  - Durable intents: `highlights_ref/overwrite/recovery.rs`, including
    `refuse_sync_if_overwrite_pending` (`:688`, called from `vault_sync.rs:977`).
  - Recovery manifests: `task_status_hooks_write/recovery.rs`.
  - No-replace creates through a hard link: `capture/commit.rs:479`.
- **Link tools.**
  - `vault_links.rs`: `NoteIndex` and `VaultLinkResolver`.
  - Ref rename's tokenizer: `highlights_ref/rename/links.rs` (`plan_vault_links` `:253`,
    `rewrite_contents` `:344`). It swaps paths and keeps `#H`, `#^id`, and aliases
    across wikilinks, embeds, and percent-encoded Markdown links.
  - Moved block ids: `collect_done/link_repair.rs` (`repair_links_in_note_with` `:933`).
  - Heading-fragment renumbering inside the same note: prose `links.rs` and
    `inbound.rs`.
  - Nothing yet maps `[[owner#Heading]]` to a different file.
- **Creation helpers.**
  - `capture_project_note::format_created_timestamp` (`:151`) writes `created:`.
  - `notebooks/scan.rs:544` `link_destination` writes the `parent` link form.
  - Collisions are refused, compared case-insensitively (`notebooks/create.rs:274`).
- **The vault.**
  - `.trash/` is Obsidian's vault trash, and the vault's `.gitignore` ignores it.
  - `app.json` sets `alwaysUpdateLinks: true`.
  - Live notes already hold extraction candidates, such as `job_2026.md`
    `## Why I left Google?` after `## Tasks`.
  - Live notes also hold the reference definition hazard: `sase_blog_contents.md` uses
    `[1]:` definitions.
- **Plugins** (bob-plugins `plugins/<id>/src/`):
  - `bob-articles` spawns bob with a 1.5 s timeout and decodes only ArticlePlan
    (`020-article-core.js:90-125`). Its adopt preview modal is `040-modal-settings.js`.
  - Multi-file bob writes from Obsidian follow the relation-rename precedent
    (`bob-navigation-hotkeys/src/429-relation-label-management.js`). It saves the active
    view, refuses dirty open editors on touched paths (`:120-163`), re-checks the
    preview before applying (`:389-415`), and uses a 20 s spawn helper
    (`bob-ledger-tools/src/236-plugin-stats-data.js:7`).
  - The task/project gesture is `bob-navigation-hotkeys:create-project-note-from-task`,
    bound to Ctrl+Shift+Alt+N in the vault's `hotkeys.json`. It dispatches by cursor
    line in `667-plugin-project-promotion.js:7-35`, and its reversal trashes the project
    note with `app.fileManager.trashFile`.
  - Ctrl+Shift+E is bound nowhere in `hotkeys.json` or in any plugin default.
  - Command Mode's prose rows are in `bob-command-mode/src/160-sheet-rank.js:55-72`.

## Governing decisions

The roster was read for this plan
(`decisions:decision-rules-are-read-at-planning-time`).

- `decisions:prose-notes-are-owned-prose`: a section's prose type is **derived
  extraction eligibility**, never in-place formatting and never a stored section type.
  The host note keeps its role, and bob never formats an area or project note.
- `decisions:drawers-keep-their-names`: the heading extraction keeps, `title:`, and
  `extracted_heading:` keep the exact authored bytes. The list-only stub reads as a
  one-note drawer under today's project reversal.
- `decisions:task-reqs-are-met-with-their-task`: extraction never reinterprets
  checkboxes. Plain checkboxes move as content. Anything holding a `#task` line refuses.
- `decisions:projects-block-from-prj-line`: the `^prj` line and its block never move
  (`holds-anchor`, `structural-section`), so project blocking is untouched.
- `decisions:unblocked-tasks-are-offered`: no cross-note effect happens without consent.
  Both commands preview by default, the plugin always shows the preview, and only an
  explicit confirm writes.
- `decisions:relation-label-rename-rewrites-history`: the precedent for one previewed
  `vault_rewrite` transaction with one revertable commit.
- `decisions:notebook-routes-write-notes`: notebooks are never hosts, and a prose note
  is never a notebook variant.
- `decisions:reference-sources-are-preserved-write-once`: link scans never read or
  rewrite `.bob/`.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the preview modal closes with Esc, `q`, and
  `Ctrl+[`. `q` types only in the stem field.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture changes nothing.
- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` or `CHANGELOG.md`
  edits. The new commands and contract are additive, so nothing here takes a `!`.

## Conversion contract

This section is the spec every phase implements. Codes are kebab-case like the existing
prose codes. Every refusal carries a 1-based line (when it has one), a message, and a
hint. An ineligible context never falls back to converting the surrounding note.

### Terms

- **Host note:** a note that is an area note, a project note, or a prose note (either
  spelling) and nothing else. Everything else refuses with `not-host-note`: notebooks,
  albums, records, refs (by type or ref folder), daily and period notes, mail notes,
  untyped notes, mixed-type notes, `done/`, and the excluded directories (`_conflicts`,
  `_templates`, `_generated`, `.obsidian`, `.git`, `.trash`, `.bob`).
- **Heading:** an ATX or single-line Setext heading that the real classifier counts in
  the top-level body (N1). A fake heading in a fence, callout, comment, or list body is
  never a boundary.
- **Prose section:** a heading H at level L (1–6) plus its subtree, which runs to the
  line before the next heading with original level ≤ L, or to end of file. Its **body**
  is everything after H's line (after the underline, for Setext). Blank lines at the
  body's start and end are **seam blank lines**.
- **Stub:** a section whose body, ignoring seam blank lines, is exactly one top-level
  bullet holding one link (`[[stem]]`, `[[stem|alias]]`) or one embed (`![[stem]]`) that
  resolves to a prose note.

### Selection

- `-s, --section IDENTITY` uses the documented heading identity, including `#A#B` nested
  paths. An exact raw-title match comes first, then a single identity match.
  - No match refuses with `no-section` and lists close matches.
  - Several matches refuse with `ambiguous-section` and list their lines.
- `-l, --line N` selects the innermost heading whose section contains line N; the
  heading line itself counts. A line in the frontmatter or above the first heading is
  `no-section`.
- Exactly one selector is required.

> [!decision] intro_prose = overview In an area or project note, `--line` above the
> first heading selects the **intro**: the authored lines between the last bob-owned
> preamble block (the `^prj` task block and any generated catalog) and the first
> heading. An empty intro is still `no-section`. Extracting the intro inserts
> `## Overview` at its place, followed by the stub. The new note records
> `extracted_intro: true` in place of `extracted_heading`. Inlining a note that carries
> `extracted_intro: true` removes the `## Overview` line again, so the round trip stays
> lossless. Prose notes never have an intro section; their preamble always means inline
> (see [Toggle dispatch](#toggle-dispatch)).

### Eligibility refusals

Checks run in this order, and the first failure is reported.

| Code                                                | Condition                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `not-host-note`                                     | The note is not a host note (see [Terms](#terms)).                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `no-section` / `ambiguous-section`                  | See [Selection](#selection). With `intro_prose = refuse`, the hint for a preamble line is "Type a heading above this text, then extract it".                                                                                                                                                                                                                                                                                                                                          |
| `structural-section`                                | In an area or project note, H, an ancestor of H, or a heading inside the subtree is structural. That means `is_structural_h2_key` (Tasks and everything under it, status groups, Pomodoros, Events, Highlights, Snoozed, the generated catalogs), the records index, or a project note's `## Notes` home. Also refused: any subtree that intersects a bob-owned region (`owned_regions` of every catalog spec, a status-group badge row, or a `<!-- bob:` marker block), in any host. |
| `holds-tasks` / `holds-anchor` / `conflict-markers` | `marker::body_refusals` over the subtree's classified lines, with the formatter's masking. The messages are worded for a section, not a note.                                                                                                                                                                                                                                                                                                                                         |
| `stub-section`                                      | H's section is already a stub for a prose note whose `parent` is this note. The refusal's `next` is `{command: "inline", note: <that note's path>}`.                                                                                                                                                                                                                                                                                                                                  |
| `unclosed-fence`                                    | A fence opens in the subtree and never closes.                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `setext-multiline`                                  | A multi-line Setext heading lies inside the range, so bob cannot place the boundary.                                                                                                                                                                                                                                                                                                                                                                                                  |
| `owned-number-heading`                              | The host is an area or project note, and a moved sub-heading's title starts with an N6 token, such as `### 1.5 million users`. The new note's numbering-off pass would strip it. Hint: write `1\.5`.                                                                                                                                                                                                                                                                                  |
| `definition-boundary`                               | A footnote label (`[^x]`) or a reference-link label is used on one side of the boundary and defined on the other. Labels case-fold as in CommonMark, and definitions are document-global.                                                                                                                                                                                                                                                                                             |
| `depth-overflow`                                    | A promoted heading level would fall outside 2–6. This is a defensive check, because promotion never deepens a heading.                                                                                                                                                                                                                                                                                                                                                                |

Lists, bullet-only sections, plain checkboxes, tables, quotes, code, and empty sections
are all eligible. Bullet-only sections can start as bullets and graduate to prose.

### Extract (`bob prose extract`)

| Rule           | Behavior                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| X1 Name        | The stem is `<owner>_<slug>`, where the slug is the de-numbered title, lowercased, with each run of characters outside `[a-z0-9]` replaced by `_`, trimmed, and capped at 60 characters. `-n, --name STEM` overrides it. The note goes in the owner's folder, so relative attachment links stay valid. A case-insensitive `.md` collision refuses `name-taken`. An empty or invalid stem refuses `bad-name`.                                                 |
| X2 Frontmatter | Keys in this order: `parent: "[[owner]]"` (the `link_destination` form), `type: "[[prose]]"`, `created:` (`format_created_timestamp`), `title:` (H's title bytes exactly as written, de-numbered only when the host is a prose note), `extracted_from: "[[owner]]"`, `extracted_heading:` (the same bytes as `title`), and `section_numbers: false`. Strings use `yaml_string_scalar`.                                                                       |
| X3 Body        | The body lines are kept verbatim. Every heading in them is promoted by L − 1, so H's children become `##`. Moved owned number tokens drop: in a prose host they are bob's (N6), and in other hosts they already refused. The new note then gets its prose format pass, with numbering off, in the same plan. The preview counts its layout edits (`counts.layout`); those and link rewrites are the only other byte changes. Line endings and BOM follow F8. |
| X4 Stub        | H's line stays byte for byte. Its body becomes one blank line, the stub bullet `- [[stem\|Title]]` (with no alias when the title contains `\|`, `[[`, or `]]`), then one blank seam line before whatever follows. When the host is a prose note, the host's format pass joins the plan so its numbers and in-note links stay current.                                                                                                                        |
| X5 Links       | Apply the [link relocation](#link-relocation) to the host, the new note, and every linking note in the vault.                                                                                                                                                                                                                                                                                                                                                |
| X6 Invariants  | The word-token invariant holds across the plan. The host before equals the host after with the stub replaced by the new note's body. The comparison ignores heading markers, owned number tokens, and link spans, which are compared through the relocation map. Each linker's tokens are unchanged outside rewritten link spans. Formatting the new note again is a no-op. The written stub must classify as `stub-section`.                                |
| X7 Focus       | `focus` names the new note's first body line.                                                                                                                                                                                                                                                                                                                                                                                                                |

> [!decision] stub_form = embed X4's stub bullet is `- ![[stem]]` instead of
> `- [[stem|Title]]`. Inline, the stub matcher, and every test accept both forms under
> either answer. Only the form extract writes changes.

### Inline (`bob prose inline`)

| Rule               | Behavior                                                                                                                                                                                                                                                                                                                                        |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| I1 Note            | The note must be a prose note (`not-prose`) with no conflict markers (`conflict-markers`) and not in an excluded directory.                                                                                                                                                                                                                     |
| I2 Parent          | `parent` must resolve to exactly one host note (`no-parent` / `parent-not-host`).                                                                                                                                                                                                                                                               |
| I3 Children        | No note may name this note as its `parent` (`has-children`, listing up to five).                                                                                                                                                                                                                                                                |
| I4 Frontmatter     | These keys are absorbed: `parent`, `type`, `created`, `title`, `section_numbers`, `template`, `extracted_from`, `extracted_heading` (and `extracted_intro`). Any other key refuses `unknown-frontmatter` and lists the keys (A6). If `title` no longer matches the stub heading's identity, the plan carries a `title-dropped` warning finding. |
| I5 Stub            | Exactly one section of the parent, at any level, must be a stub that resolves to this note (`stub-missing` / `stub-ambiguous`). Provenance is not required. A note without provenance (root prose) inlines only into a stub someone wrote by hand.                                                                                              |
| I6 Depth           | The shift is L − 1, where L is the stub heading's level. Every heading level h in the note must satisfy 2 ≤ h and h + L − 1 ≤ 6. Otherwise inline refuses `depth-overflow` and names the line.                                                                                                                                                  |
| I7 Body            | When the note's numbering is on, strip owned number tokens (N6/N9) first. Then demote by L − 1 and splice the body into the stub. The heading line is unchanged, and each seam gets one blank line. When the parent is a prose note, its format pass joins the plan.                                                                            |
| I8 Collisions      | A block id in the note that the parent already uses outside the stub refuses `block-id-collision`. A footnote or reference label defined in both notes refuses `definition-collision`.                                                                                                                                                          |
| I9 Links and trash | Apply the inverse relocation, and trash the note through `vault_rewrite`'s trash operation. `focus` names the parent's stub heading line.                                                                                                                                                                                                       |

### Link relocation

Extract maps the moved headings and block ids to the new note. Inline applies the
inverse. T is the kept heading, S is a moved heading, and `^id` is a moved block id.

| Link                                                              | Found in                                          | Extract writes                                                                | Inline writes                                                                              |
| ----------------------------------------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `[[#S]]`, `[[#^id]]`                                              | the host, outside the move                        | `[[stem#S]]`, `[[stem#^id]]`                                                  | pathless again                                                                             |
| `[[#Other]]`, `[[#^kept]]`, `[[#T]]`                              | the moved text                                    | `[[owner#Other]]` and so on                                                   | pathless again                                                                             |
| `[[#S2]]` between two moved headings                              | the moved text                                    | unchanged                                                                     | unchanged                                                                                  |
| `[[owner#S]]`, `[[owner#^id]]`, `![[owner#S]]`, `[x](owner.md#S)` | any other note, including `done/` and daily notes | the path becomes the stem, and the fragment becomes S without an owned number | `[[owner#S]]`, using the shortest fragment that resolves uniquely (`T#S` only when needed) |
| `[[owner#T]]`                                                     | anywhere                                          | unchanged (it lands on the stub)                                              | unchanged                                                                                  |
| `[[stem]]`, `[[stem\|alias]]`, `![[stem]]`                        | anywhere except the stub itself                   | —                                                                             | `[[owner#T]]`, keeping the alias and embed                                                 |
| `[[owner]]`                                                       | anywhere                                          | unchanged                                                                     | unchanged                                                                                  |

- **Preserved:** the alias, the embed `!`, display text, percent-encoding, the `.md`
  suffix style, and stem versus path form.
- **Same-note links:** a link whose result lands in the note that contains it is written
  pathless.
- **Never scanned** (the prose rule): links inside frontmatter, code, math, comments,
  and HTML blocks; the directories `.bob/`, `.trash/`, `_conflicts`, `.obsidian`, and
  `.git`; and non-UTF-8 files.
- **Ambiguous fragments:** a fragment that matches both a moved heading and one that
  stays refuses `ambiguous-link`, naming `path:line`.

### Losslessness (A6)

- **L1:** take an eligible section S in a layout-clean host, meaning the prose format
  pass would not change S's promoted body, with no owned-number ambiguity.
  `inline(extract(S))` restores the host byte for byte, apart from bob-owned number
  tokens (a prose host renumbers) and seam blank lines.
- **L2:** after a round trip, every link in every other note resolves to the same
  heading or block as before. Fragment spelling may canonicalize, for example
  `[[owner#T#S]]` → `[[owner#S]]`.
- **L3:** anything that would be lost either refuses or appears in the preview: layout
  normalization (`counts.layout`), unknown frontmatter keys, a dropped title, and
  collisions.

### Provenance (A5)

`extracted_from` (a frontmatter link) and `extracted_heading` mark **derived prose**. A
prose note without them is **root prose**.

- In v1 they are informational. The stub is what places a note for inline.
- Phase 3 reads them: derived prose may return to its origin, and root prose is placed
  and never poured into a task.
- Keep both fields when a note is reparented. `title` stays the user-editable display
  title; `extracted_heading` stays the origin's heading.
- Document both fields in `docs/reference/vault-format.md`.

### ProseConversion v1 (JSON)

This is a new contract that is versioned on its own and never shares ArticlePlan's
version or keys. It only grows. Clients reject an unknown `schema_version` or a
`contract` other than `"prose-conversion"`. bob owns every display string.

```json
{
  "schema_version": 1,
  "contract": "prose-conversion",
  "command": "extract",
  "status": "preview",
  "fingerprint": "sha256:<hex>",
  "note": "job_2026.md",
  "section": { "line": 31, "end_line": 44, "level": 2, "title": "Why I left Google?" },
  "target": {
    "path": "job_2026_why_i_left_google.md",
    "stem": "job_2026_why_i_left_google"
  },
  "changes": [
    {
      "path": "job_2026.md",
      "action": "update",
      "sha256": "<before>",
      "diff": "<unified>"
    }
  ],
  "links": [
    {
      "path": "dev.md",
      "line": 12,
      "before": "[[job_2026#Why I left Google?]]",
      "after": "…"
    }
  ],
  "counts": { "notes": 3, "links": 2, "layout": 0, "replaced": 0 },
  "findings": [],
  "refusal": null,
  "focus": { "path": "job_2026_why_i_left_google.md", "line": 10 },
  "notice": "Extracted “Why I left Google?” into job_2026_why_i_left_google · 2 links repaired",
  "commit": null,
  "sync": { "pre": "skipped", "post": "skipped" },
  "warnings": []
}
```

- `status` is `preview`, `written`, or `refused`.
- `action` is `create`, `update`, or `trash`. `sha256` is the before-image's hash,
  `null` for a create.
- `refusal` is `{code, message, hint, path, line, next}`, where `next` is `null` or
  `{command, note}`.
- Exit codes: 0 for a preview or a write; 2 for a refusal, a usage error, or a write or
  transaction failure, which `vault_rewrite` failures report with their codes.

### CLI

```text
bob prose extract NOTE (-s SECTION | -l LINE) [-n STEM] [-w] [-e FINGERPRINT] [-o] [-b DIR] [-f human|json]
bob prose inline NOTE [-w] [-e FINGERPRINT] [-o] [-b DIR] [-f human|json]
```

- Both commands preview by default, as `adopt` does. `-w, --write` writes.
- `-e, --expect FINGERPRINT` refuses `plan-changed` and writes nothing when the
  re-planned fingerprint differs.
- `-o, --offline` commits locally and skips vault sync, with the wording
  `bob relations rename` uses. Online writes need `BOB_DIR` to match the vault.
- Human previews print a colored unified diff per file, the counts, the findings, and a
  write hint. Writes print bob's notice and the commit.
- Follow the `cli_rules` reference memory: sorted options, a short alias for every long
  option, and help that is clear and complete.
- The `bob prose` group copy becomes "Format, inspect, extract, and inline prose notes",
  and the root-help line changes to match.
- Both paths get `Excluded` catalog entries in `catalog/table.rs`, next to
  `["prose", "format"]`, with the reason "Use Bob Prose's extract and inline gesture".

### Write safety

- **W1:** preview and write share one planner. A write plans again from disk after
  pre-sync.
- **W2:** the fingerprint is the sha256 of the inputs only: each touched path with its
  before-image sha256 (or `absent`), plus the resolved selector (heading and end line),
  the stem, and the stub form. It never includes the `created:` timestamp.
- **W3:** writes go through `vault_rewrite::run`, with `txn-trash-recovery`'s trash
  operation and journal. The verify hook re-reads every written file and re-checks X6 or
  I5–I9 on the written bytes. Undo is the inverse command, or `git revert` of the one
  commit.
- **W4:** the whole conversion is all-or-nothing. Prose `format` skips a linker that
  drifted; a conversion instead aborts on any drifted file (`stale_note`) and is never
  half applied.
- **W5:** the commit subjects are `bob prose extract: <owner> § <title> → <stem>` and
  `bob prose inline: <stem> → <owner> § <title>`.

### Toggle dispatch

The gesture's thin client decides only from bob's JSON:

1. It runs `bob prose extract PATH -l <cursor line> -f json`.
2. On a preview, it opens the extract modal.
3. On a refusal whose `next` is set, it runs that command's preview and opens the inline
   modal.
4. On any other refusal, it shows bob's message and hint.

bob sets `next` in two cases. A `stub-section` refusal names the stub's note.
`no-section` in a prose note that has a parent names the note itself, so the cursor
above a prose note's first heading inlines the note.

## Decisions in this plan

- **`keymap`.** Under `new_chord`, bob-articles' `toggle-prose-section` command declares
  the default hotkey Ctrl+Shift+E (`["Ctrl","Shift"]` + `E`, matching the vault's `Ctrl`
  convention). No vault file changes.

> [!decision] keymap = overload `toggle-prose-section` declares no hotkey. In
> bob-navigation-hotkeys' `createProjectNoteFromTask`
> (`667-plugin-project-promotion.js`):
>
> - **Prose note:** before the area/project check refuses, run
>   `app.commands.executeCommandById("bob-articles:toggle-prose-section")`.
> - **Area or project note:** a cursor line that is neither a task line nor `^prj`
>   delegates the same way, instead of showing "Place the cursor on an open #task
>   checkbox".
> - Task and `^prj` lines keep today's behavior byte for byte.
> - When bob-articles is not loaded, today's notices stay.
>
> Add nav tests for each branch, and add the `bob-prose-sections-v1` marker to the nav
> plugin too.

- **`stub_form`, `intro_prose`.** See [Extract](#extract-bob-prose-extract) and
  [Selection](#selection).
- **`prose_catalog`.** With the default (`false`), the `prose-catalog` phase records
  that the answer was no and closes without changes.
- **`memory_prose_section`.** Bryan approves memory explicitly, so it defaults to off.
  `contract-docs` runs it only when it is accepted.

## Rules every phase follows

1. **Reuse, don't fork.** Use `prose/scan.rs` for classification, `prose/sections.rs`
   for identity, `prose/links.rs` and `vault_links.rs` for resolution, ref rename's link
   tokenizer, `collect_done` block-link repair, and `vault_rewrite`. Widen visibility
   inside `prose`, or extract a shared helper. Never write a second Markdown classifier
   or link tokenizer.
2. **Never write the live vault** (`~/bob`) from tests or smoke runs. Previews against
   it are read-only, with no `-w`. Round trips run on a throwaway `git clone` of the
   vault with `-o -b <clone>`.
3. **Contracts.** ArticlePlan, its codes, and the `bob-articles` and existing command
   ids are untouched. ProseConversion v1 only grows. Don't edit `Cargo.toml` versions or
   `CHANGELOG.md`.
4. **Gates.** Each bob-cli phase passes `just all`, and `just docs-gen` leaves no diff.
   A failure that reproduces on the clean base is recorded as a `PROPOSED FOLLOW-UP:`,
   citing any bead that already tracks it.
5. **bob-plugins.** Open it with `/sase_repo` and follow its `AGENTS.md`. Edit only
   `src/` fragments, build, test, validate, and deploy with `bob plugins sync` (never
   `-F` on your own). Before closing, confirm with `git branch -r --contains <sha>` that
   the commit is on `origin/master`, and record the SHA. Phase `bob-cli-qj.3` closed on
   a commit that never reached origin.

## Phases

### txn-trash-recovery: trash files and recover interrupted vault rewrites (bob-cli)

1. **Trash operation.** Extend `FileRewrite` (or replace it with a small enum) with a
   removal that carries a before-image.
   - Installing it moves the file into the vault's `.trash/`. Create that folder if
     needed, and on a name collision use Obsidian's ` 2`, ` 3` suffixes.
   - Rollback moves the file back only while the trash copy still holds the
     before-image.
   - The commit stages the deletion. Confirm that `ob::commit_paths` stages a removed
     path, and fix it if not.
   - Existing callers compile and behave exactly as before.
2. **No-replace creates.** Install a create through hard-link-or-refuse, as
   `capture/commit.rs:479` does, so a file that appears after the stale check is never
   overwritten.
3. **Intent journal.** Before the first write, persist one intent under
   `bob_cli_state_dir()/vault-rewrite/<vault-hash>/<run-id>/` and fsync it.
   - It records each path, its action, the before and after sha256, a before-image copy
     for every rewrite and trash, and each trash destination.
   - After the commit, record the commit sha in the intent and then remove the intent.
     After a completed rollback, remove it too.
4. **Recovery.** Run it under the lock at the start of every `vault_rewrite::run`, and
   in the vault-sync pre-cycle next to `refuse_sync_if_overwrite_pending`.
   - An intent whose after-images are committed at HEAD is complete: remove it.
   - An intent whose every file holds exactly its before-image or its after-image is
     rolled back: restore the before-images, remove the creates, and move trashed files
     back. Report it with one warning (stderr, plus JSON `warnings` where the caller has
     them).
   - Anything else refuses with the new code `recovery_pending`. The message names the
     manifest and every path holding neither image, and the hint follows the overwrite
     recovery precedent. Never commit half a rewrite.
5. **Crash evidence (tests).**
   - A test-only fault point stops after k of n writes and skips the in-memory rollback.
     For every k, recovery restores the exact before-state across mixes of create,
     rewrite, and trash.
   - A file edited after the crash refuses `recovery_pending`.
   - A crash after the commit only removes the intent.
   - The sync pre-cycle recovers, and refuses when it cannot.
   - Every existing `vault_rewrite` test stays green.
6. **Docs.** Add `recovery_pending` and a short "interrupted rewrites" paragraph
   wherever the `vault_rewrite` failure codes are documented (search `docs/` for
   `stale_note`).
7. Commit as `feat(vault-rewrite): trash files and recover interrupted rewrites`.

### section-model: prose section selection and eligibility (bob-cli)

1. Add `src/native/prose/section_model.rs`.
   - It classifies the note with `scan::classify` and derives heading subtrees from
     original levels (see [Terms](#terms)).
   - It resolves `Selector::Section(identity)` and `Selector::Line(n)` (see
     [Selection](#selection)).
   - It returns either a `ProseSection` or a refusal from the
     [eligibility table](#eligibility-refusals), in table order.
   - `ProseSection` carries the host kind, the heading line and level, the raw and
     de-numbered title, the identity, the body range, the seam ranges, the moved
     headings with their original levels and lines, and the moved block ids.
2. **Host kind.** Use the `projects::scan` predicates, `NoteSubject::canonical`, and the
   excluded-directory list. Mixed types refuse.
3. **Structure.**
   - Reuse `is_structural_h2_key`, `reserved_group_heading_lines`, and the badge-row
     parser.
   - Use `owned_regions` over every `GENERATED_CATALOGS` spec, plus the records spec.
   - Treat the project `## Notes` home (`task_block_layout::NOTES_SECTION_TITLE`) as
     structural.
   - Apply structural keys only in area and project hosts; in any host, refuse a subtree
     that intersects a bob-owned region.
4. **Refusals over the slice.** Make `marker::body_refusals` `pub(super)` and run it on
   the subtree's lines. Give it section-worded messages for these callers without
   changing the format refusals' text.
5. **Definitions.** Add a protected-region-aware scanner for footnote references and
   definitions, and for reference-link uses (`[t][label]`, `[label][]`, `[label]`) and
   definitions (`[label]: dest`). Labels case-fold. Expose it for `definition-boundary`
   here and for inline's `definition-collision`.
6. **Stub recognition.** Add a `stub_target(section)` helper that returns the stub
   bullet's link, embed, and target. `stub-section` and inline I5 both use it.
7. Implement only the chosen `intro_prose` branch (see [Selection](#selection)).
8. **Tests.** Fixture corpus under `src/native/prose/tests/fixtures/sections/`:
   - Boundaries: nested and skipped levels, ATX and Setext, CRLF and BOM, Unicode
     titles, empty sections, and fake headings in fences, callouts, and lists.
   - Every refusal code.
   - Line and identity selection, including `#A#B` paths and duplicate titles.
   - Real shapes: a project with `^prj`, a `[!projects]` callout, `## Tasks` with status
     groups, and an authored section after Tasks (shaped like `job_2026.md`); an area
     with catalogs; a numbered prose note.
9. Commit as `feat(prose): select and classify prose sections`.

### link-relocation: relocate links around a moved section (bob-cli)

1. Add `src/native/prose/relocate.rs`.
   - A `RelocationMap` built from a `ProseSection` and the new stem: moved headings with
     their old and new fragments, moved block ids, and the kept heading.
   - Its inverse for inline: the stem becomes owner `#T`, and so on.
2. **Rewriter.** For one note's text, find every wikilink, embed, and Markdown link.
   - Use the shared tokenizer from `highlights_ref/rename/links.rs`, extracted into a
     shared helper if needed.
   - Resolve the target note with `vault_links` (a pathless link targets its own note),
     and resolve the fragment against the old heading tree with the prose
     `resolve_heading` rules.
   - Rewrite exactly the rows of the [link relocation](#link-relocation) table, keeping
     everything that table says to keep.
   - Use `collect_done/link_repair.rs` for moved block ids where it fits.
   - Ambiguous fragments refuse `ambiguous-link`.
3. **Vault scan.** Prefilter candidate linkers by a case-insensitive stem substring, as
   `prose/inbound.rs` does, then resolve them. Never scan the excluded directories or
   non-UTF-8 files. Return per-link `{path, line, before, after}` and each changed
   note's after-image. The caller owns the writes.
4. **Tests.** The `__cdx` link matrix, run in both directions:
   - Source-local links, moved-to-staying links, links within the move, third-party
     links, and embeds.
   - Aliases, percent-encoded Markdown links, and nested and duplicate headings.
   - Links to the kept heading, links in `done/` and daily notes, and links in protected
     regions (left alone).
   - Same-note pathless rendering, and `inverse(forward(x))` resolving to the original
     targets (L2).
5. Commit as `feat(prose): relocate links around moved prose sections`.

### extract: bob prose extract planner and hidden CLI (bob-cli)

1. Add `src/native/prose/convert/` (`mod.rs`, `extract.rs`). It holds the
   `ConversionPlan` (file changes, link rows, counts, findings, focus, notice), the
   ProseConversion v1 serde types, the fingerprint (W2), and the extract planner, which
   implements X1–X7 on top of `section_model` and `relocate`.
2. **The new note.** Render it per X2–X3. Run the existing prose format pipeline with
   numbering off on the new note, and on a prose host, in the same plan. Verify the word
   invariant and idempotency per X6, failing with `words-changed` or `not-idempotent` as
   `format` does.
3. **Writes.** Build a `ReadyPlan` holding the create, the host rewrite, and the linker
   rewrites. Its verify hook re-reads the written bytes and re-checks X6, including that
   the stub classifies as `stub-section`. Make one commit (W5), and honor `-o`, `-e`,
   and `plan-changed`.
4. **CLI.** Add `bob prose extract` as specified in [CLI](#cli), registered with
   `hide(true)`. Keep it hidden until `inline` unhides both commands (A3: the inverse
   ships with the extract). Add its `Excluded` catalog entry.
5. **Tests** (`tests/cli/prose.rs`, scratch git vaults):
   - Every refusal through the CLI.
   - Preview versus write, `-f json` shape and exit codes, and `plan-changed`.
   - The `name-taken` race (a file appearing after the preview).
   - A drifted linker aborting the whole write (W4).
   - Offline commits, and that `.bob/` and `.trash/` are never touched.
   - A prose host with numbering on.
6. Commit as `feat(prose): plan and write prose-section extraction`.

### inline: bob prose inline and the round-trip guarantee (bob-cli)

1. **Planner.** Add `convert/inline.rs`, implementing I1–I9 with `section_model`'s stub
   helper and definition scanner, the inverse `RelocationMap`, and the trash operation.
   Reuse extract's fingerprint, verify, and commit flow.
2. **CLI.** Add `bob prose inline` and unhide `extract` in the same commit. Then:
   - Update the `bob prose` group's about, long_about, and examples, and the root-help
     line.
   - Regenerate the help goldens with `just docs-gen`.
   - Add the `Excluded` catalog entry.
3. **Property tests** (proptest is already a dev-dependency).
   - Generate eligible sections in area, project, and prose hosts. Vary levels, lists,
     code, quotes, tables, block ids, and in-note links, and add third-party linkers.
   - Assert L1 for both stub forms. Assert L2 for every linker.
   - Assert that layout-dirty sections report `counts.layout` in the preview (L3).
4. **Tests.**
   - Every inline refusal.
   - `title-dropped`.
   - Root prose inlined into a hand-written stub.
   - A note with `section_numbers: true`.
   - Trash plus rollback when verify fails.
   - Recovery after an injected crash, through the CLI.
5. Commit as `feat(prose): inline prose notes back into their parent section`.

### contract-docs: contract docs, host probe, and vault-clone dogfood (bob-cli)

Read the `docs.md` reference memory first.

1. **`docs/prose.md`.** Add "Prose sections: extract and inline" (terms, selection,
   eligibility, X- and I-rules, link relocation, losslessness, provenance, write safety,
   toggle dispatch) and "JSON: ProseConversion v1". State that ArticlePlan is unchanged
   and that the two contracts version separately.
2. **Reference pages.**
   - Document `extracted_from` and `extracted_heading` (and `extracted_intro` under
     `overview`) in the Prose notes section of `docs/reference/vault-format.md`.
   - Add both commands to `docs/reference/json-outputs.md`.
   - Widen the `prose` summary in `docs/_meta/contracts.yml`.
   - Run `just docs-gen` and `just docs-check`.
3. **Host probe.** Add a `prose-extract-preview` probe to
   `scripts/dev/bob-host-currency`. It builds a throwaway vault under the host's temp
   directory with one area note, `probe.md` (`type: "[[area]]"`), holding `## Intro`,
   and passes only when `bob prose extract -b "$vault" probe -s Intro -f json` returns
   `"status": "preview"`. Update the usage text, the known-probe list, and
   `probe_detail`. Pass `just check-scripts`.
4. **Dogfood.**
   - On a throwaway clone of the vault, run extract and then inline with
     `-o -w -b <clone>` on `job_2026` § "Why I left Google?" and on two more authored
     sections you pick (one bullet-only, one in a numbered prose note). Assert the files
     are byte-identical apart from what L1 allows.
   - Run read-only previews against the live vault for those sections and for one
     structural refusal.
   - Store the transcript with `/sase_artifact`.
5. Commit as `docs(prose): document prose-section extract and inline`.

> [!decision] memory_prose_section Use `/sase_memory_write` to add the glossary strand
> `glossary:prose-section`:
>
> - Frontmatter: keyword "Prose Section", alias "prose section".
> - Body: an authored heading subtree in an area, project, or prose note that
>   `bob prose extract` can move into a prose note, leaving the heading and a stub
>   bullet; `bob prose inline` folds it back. Eligibility is derived at the cursor and
>   never stored. Derived prose records `extracted_from` and `extracted_heading`.
> - What it is not: an [[annotation-section]], a [[task-drawer]], a stored section type,
>   or a [[notebook]] section, and it is never formatted in place.
> - A pointer to `docs/prose.md`.
>
> Then run `sase memory init`, `just docs-gen`, and `just docs-check`.

### keymap: Bob Prose extract and inline gesture in Obsidian (bob-plugins)

1. **Client core** (a new `bob-articles` fragment, added to `fragments.json`).
   - A ProseConversion v1 decoder that rejects an unknown `schema_version` or
     `contract`.
   - A desktop-only spawn helper with a 10 s preview timeout and a 30 s write timeout
     (writes may sync).
   - Detection of an old bob, meaning a clap "unrecognized subcommand" error. Show at
     most one notice per session: "Update bob to extract prose sections".
2. **Commands** (ids are new and prose-named from day one):
   - `extract-prose-section` "Extract section to prose note", for area, project, and
     prose notes, judged from the metadata cache `type`.
   - `inline-prose-note` "Inline prose note into parent", for prose notes.
   - `toggle-prose-section` "Extract or inline prose section", which implements
     [Toggle dispatch](#toggle-dispatch) and is bound per the `keymap` decision.
   - Every command first saves the active view.
3. **Preview modal.**
   - It shows bob's notice-style summary, the counts, the findings, and per-file diffs.
     The host's diff is open and the others are collapsed.
   - For extract, an editable stem field (with its `// bob-text:` role comment)
     re-previews with `-n` after a debounce.
   - Enter confirms. Esc, `q`, and `Ctrl+[` close the modal through Esc's dispatch.
4. **Confirm.**
   - Refuse when an open editor on any `changes[].path` is dirty, using the
     relation-rename wording "Save or close affected open notes before …".
   - Run the same command with `-w -e <fingerprint>`, plus `-n` for extract.
   - On `plan-changed`, refresh the preview in place, saying the vault changed since the
     preview.
   - On success, open `focus.path` at `focus.line`. For inline, move the leaf to the
     parent before the trashed note disappears. Then show bob's `notice`.
5. **Command Mode.** Add surface rows for `bob-articles:extract-prose-section` and
   `bob-articles:inline-prose-note` in `160-sheet-rank.js`, update the comment above
   them, and update the tests that pin the surfaces.
6. **Keymap.** Implement only the chosen `keymap` branch (see
   [Decisions in this plan](#decisions-in-this-plan)). Under `new_chord`, confirm in
   apollo's Settings → Hotkeys that nothing else claims Ctrl+Shift+E.
7. **Docs.** Update `docs/bob-articles.md` (commands, gesture, modal keys, old-bob
   notice) and `docs/bob-command-mode.md`. Run `npm run docs:gen` and
   `npm run docs:check`.
8. **Tests.** In `scripts/test-bob-articles.cjs`, using fake-bob fixtures, cover:
   - Each dispatch branch, including following `next`.
   - A schema or contract mismatch, the old-bob notice, and the desktop guard.
   - The dirty-editor refusal, `plan-changed`, and focus after extract and after inline.

   Add the Command Mode sheet cases. Run `npm run build`, `npm test`, and
   `npm run validate`.

9. **Ship.**
   - Add the marker comment
     `// Marker searched by the prose-sections epic: bob-prose-sections-v1` to each
     changed plugin, and bump each changed plugin's version.
   - Deploy with `bob plugins sync`.
   - Run
     `scripts/dev/bob-host-currency --probe prose-extract-preview --plugin bob-articles=bob-prose-sections-v1`
     (and the nav marker under `overload`), and store the table with `/sase_artifact`.
     For a host that is behind, record a `PROPOSED FOLLOW-UP:` to update it; don't wait
     on it.

### prose-catalog: optional generated Prose catalog (bob-cli, bob-plugins)

With the default (`prose_catalog = false`), make no changes. Note on the phase bead that
the reviewer declined the catalog, then close the phase.

> [!decision] prose_catalog Build the catalog and the reversal fix it obligates:
>
> 1. **bob-cli catalog.**
>    - Add a `PROSE` `RegionSpec` (`## Prose` heading, `> [!prose]` callout, placed with
>      Notebooks and Albums before `## Tasks`) and add it to `GENERATED_CATALOGS`.
>      `is_structural_h2_key` and capture's `reject_generated_catalog_selector` then
>      protect it; add the code `prose_section_generated`.
>    - Each area or project lists its descendant prose notes down to the next root
>      parent as `- [[stem|Title]]`, plus ` — summary` when the note has one, following
>      the notebooks precedent.
>    - Add `bob prose sync` with the same flags and nightly placement as
>      `bob notebooks sync`.
>    - Extract and inline refresh the affected owner's catalog in the same plan.
>    - Document the catalog in `docs/prose.md` and `docs/areas.md`.
> 2. **bob-plugins.**
>    - Add the Prose key to the four drawer twins and keep the parity test green.
>    - Generalize nav's `skipGeneratedProjectsCatalog` (`212-project-conversion.js:111`)
>      so project reversal skips every generated catalog callout (`[!projects]`,
>      `[!notebooks]`, `[!albums]`, `[!areas]`, `[!prose]`). This is the research's
>      catalog-skip fix.
>    - Make sure ledger-tools' host-catalog card handling recognizes `[!prose]`.
>    - Add tests, bump versions, and deploy.

## Out of scope

- **The report's Phase 3, reversible projects.** That covers the reversal blockers
  (status groups, catalog skips beyond `prose_catalog`, req asymmetry, project-only
  frontmatter) and the carry and re-adopt rules for prose children. After this epic, a
  project with an extracted section still refuses reversal because of the prose child
  note. Its stub section is already drawer-shaped.
- Arbitrary selections. Extraction moves heading subtrees only.
- Notebooks, records, refs, albums, and daily notes as hosts.
- Formatting prose sections in place, and any stored section type.
- The Note Composer stopgap, a plugin id rename, ArticlePlan changes, and Bob Mac
  Capture.
- Moving attachments. The new note sits in the owner's folder, so relative links keep
  resolving.

## Done when

- `bob prose extract` and `bob prose inline` are visible and documented, and they
  preview by default.
  - They write all or nothing through one commit.
  - They refuse every ineligible case with a named code.
  - They return ProseConversion v1.
- `vault_rewrite` trashes files and recovers or refuses after an interrupted rewrite,
  with crash tests at every write point.
- The property tests prove L1 and L2. The vault-clone dogfood shows byte-identical round
  trips on real sections.
- Extracted notes carry `parent`, `type`, `created`, `title`, `extracted_from`,
  `extracted_heading`, and `section_numbers: false`.
- In Obsidian, the chosen gesture previews and applies extract and inline, follows bob's
  `next`, and opens the result. Command Mode lists both commands. The marker is deployed
  and recorded.
- The `prose_catalog` and `memory_prose_section` work happened only if accepted.
