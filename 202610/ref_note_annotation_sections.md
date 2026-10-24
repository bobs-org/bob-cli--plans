---
tier: epic
title: Annotation sections for ref notes (`# Name` + `@name`)
goal: "A Highlights sticky note whose first line is `# Name` declares a section, and
  every highlight, underline, image, or sticky note tagged `@name` renders grouped under
  it: real headings with a quiet section card in Obsidian, native section headers in the
  Bob Mac Capture Refs panel, and grouped output in every `bob ref show` format. Notes
  that never declare a section stay byte-identical, the grouping is re-derived from the
  annotations on every scan (nothing new is persisted), and adding, removing, or moving
  a tag never changes a `^h-` ID or re-creates a task.

  "
decisions:
  decision_record:
    ask:
      When the epic lands, record the annotation-sections design as a decisions strand?
    memory:
      - decisions:annotation-sections-are-name-keyed-folds
    default: false
    answer: false
  glossary_term:
    ask: When the epic lands, add an "Annotation Section" glossary term?
    memory:
      - glossary:annotation-section
    default: false
    answer: false
phases:
  - id: gates
    title: Sidecar parser gates
    depends_on: []
    size: medium
    description:
      "gates: fix bob-cli-ba so hard-wrapped quote lines starting with `# ` stay quote
      text, and make sidecar page-heading detection level-aware so a `# P values` sticky
      is never a page break."
  - id: grammar
    title: Section grammar and pure planner
    depends_on:
      - gates
    size: medium
    description:
      "grammar: teach the sidecar parser to keep `# Name` sticky declarations, and add
      sections.rs with the declaration and tag scanners and a pure plan_sections over
      displayed blocks (homes, consumed text, diagnostics). Unit tests only; no
      rendering change."
  - id: region
    title: Sectioned managed-region parser
    depends_on: []
    size: medium
    description:
      "region: teach parse_managed_region to read sectioned regions (section H3 +
      summary|section card + ^s- line, #### page captions, Other highlights) into
      RegionParse.sections and RegionBlock.section, leaving every unsectioned region
      parsed exactly as today."
  - id: render
    title: Sectioned rendering and identity
    depends_on:
      - grammar
      - region
    size: large
    description:
      "render: restructure render_sidecar_highlights into collect-then-assemble, emit
      the sectioned layout with consumed tags, apply the both-sides @-insensitive
      identity keys, route declaration #task lines to the s- block, and prove
      byte-identical output for undeclared notes plus ID stability through tag toggles."
  - id: show
    title: Sections in bob ref show
    depends_on:
      - region
    size: medium
    description:
      "show: add the additive sections list and per-annotation section key to bob ref
      show JSON, and grouped human and Markdown output, with -c/-N semantics and
      docs/ref.md updates."
  - id: edges
    title: Section stubs, diagnostics, and route hints
    depends_on:
      - render
    size: medium
    description:
      "edges: add redirect stubs for a published declaring sticky and for a vanished but
      linked section, the dry-run/concise/JSON section diagnostics, the missing-route
      hint, and the tag-on-task diagnostic."
  - id: docs
    title: Docs, Obsidian styling, and cross-surface checks
    depends_on:
      - edges
      - show
    size: medium
    description:
      "docs: write the user-facing Sections guide in docs/highlights-ref-sync.md, add
      the section-card CSS to the vault ref-highlights snippet and its doc mirror, cover
      rename and note-snapshot behavior with tests, run the vault-wide zero-change
      check, and apply any approved memory decisions."
  - id: mac
    title: Section headers in the Bob Mac Capture Refs panel
    depends_on:
      - show
    size: large
    description:
      "mac: in bob-mac-capture, decode sections, project section headers into the
      inspector feed without shifting entry indices, draw the header view with tokens
      and VoiceOver, group the Copy card, and update docs and design renders."
proposed_by: bbugyi200.apollo.research.2d.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 07:59:19
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_note_annotation_sections.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_note_annotation_sections.md)

# Plan: Annotation sections for ref notes (`# Name` + `@name`)

## What Bryan gets

Two rules of thumb, which the docs lead with and every surface follows:

- **A sticky note whose first line is `# Name` starts a section.** Anything after that
  line is the section's description.
- **Tag a note with `@name` at the start or end of a line to file it there. If you can
  still see an `@name`, it didn't group.**

While reading in Highlights:

```text
Sticky on page 1:      # Memory
                       Everything that belongs in the blog's memory section.
Comment on page 4:     @memory
Comment on page 5:     Compare with the intro @memory
```

The managed region in the ref note:

```md
### Memory

> [!summary|section] @memory · 2 highlights · Pages 4–5 Everything that belongs in the
> blog's memory section.

^s-memory

#### Page 4

> [!quote] Make a point about how if I were building AI from scratch right now, …

^h-496b1ca10117

#### Page 5

> [!quote] Mention that you've implemented project memory every wrong way possible …
>
> > [!note] Comment Compare with the intro

^h-6bbab22c999c

### Other highlights

#### Page 2

> [!quote] POTENTIAL QUOTES: …

^h-38a082b00de9

### Removed highlights

…
```

- In Obsidian the sections are real headings. They show up in the Outline pane, fold,
  and can be linked or embedded: `![[ref/notes/sase_blog_261019#Memory]]` is a live view
  of the curated subset.
- The card is a built-in `summary` callout, so it looks distinct even without CSS. With
  the `ref-highlights` snippet it becomes a quiet violet rule with a faint small-caps
  caption, and yellow quotes and gold underlines stay the loudest marks on the page.
- The Mac Refs panel draws native section headers, using the same violet for structure.
- `bob ref show` (human, Markdown, and JSON) groups the same way, so agents see the
  synthesis directly.

The design follows
`research:202610/ref_note_annotation_sections/ref_note_annotation_sections.md`, which
Bryan accepted in full. The deliberate refinements are listed under
[Refinements to the research](#refinements-to-the-research).

## Before approving: the two-minute Highlights export test

Nobody has confirmed that Highlights exports a sticky note beginning with `# ` verbatim.
On the Mac, in a scratch PDF:

- a sticky with `# Test section`;
- a sticky with `# Other`, a blank line, then a description;
- a highlight comment with `@test-section`;
- an underline comment with a tag;
- a comment containing `\@x`.

Then read the sidecar bytes, in both the Markdown and TextBundle exports. If you
annotate on the iPad, repeat it there too (autocapitalization, smart punctuation).

The phases can start before this test is done. The declaration spelling lives in one
function (`scan_section_declaration`), and no note changes until a declaration exists.
If the test shows `#` is escaped or dropped, send plan feedback so the spelling changes
before `grammar` lands.

## Shared design contract

Every phase implements against this section. Phase sections below say only what is
specific to them.

### Declaring a section

- **Where:** only in a standalone (sticky) note. A highlight, underline, or image
  comment never declares.
- **The declaration line** is the first non-empty line of the sticky chunk, found after:
  - skipping heading lines of level 2–6 (Highlights' `##### 2026-10-19:` annotation
    headers);
  - stripping an optional `Note:` / `note:` / `[note]` / `[Note]` marker.

  It must be exactly one `#`, then whitespace, then non-empty text.
  - `#Memory` (an Obsidian tag), `## Memory`, and `\# Memory` (the escape) are not
    declarations.
  - Neither is a `# ` line anywhere other than the first line.

- **Aliases:** a trailing run of `@alias` tokens on the heading line. They are removed
  from the title and added as match keys: `# Open questions @q`.
- **Title:** the rest of the heading line, trimmed, with `#`, `|`, `^`, `[[`, and `]]`
  removed and whitespace collapsed. If no letter or digit is left, the declaration is
  rejected (`section_title_empty`).
- **Description:** every later line, with blank edges trimmed. It renders like note
  text: beautified, image-note paths embedded, `#task` bullets imported.
  - Heading-shaped lines inside a description render escaped (`\## Sub`) instead of
    being deleted.
  - `@tokens` in a description are literal, because sections do not nest.
- **Match key:** `key(s)` is the lowercased letters and digits of `s` (Unicode-aware via
  `char::is_alphanumeric` and `to_lowercase`; everything else is dropped). A section's
  keys are `key(title)` plus `key(alias)` for each alias. `@open-questions`,
  `@OpenQuestions`, and `@open_questions` all match "Open questions". No new crate.
- **Canonical tag** (shown on the card and in JSON): `@` + the first alias, or else `@`
  - the title's letter/digit runs, lowercased and joined with `-` (`@open-questions`,
    `@記憶`).
- **Section block ID** `s-<slug>`:
  - The slug comes from the first alias, or else the title: its ASCII letter/digit runs,
    lowercased and joined with `-`, capped at 40 characters at a run boundary.
  - If no ASCII alphanumerics remain, the slug is the first 8 hex digits of
    `sha256(source_pdf_value ‖ 0 ‖ first key)`.
  - Slug collisions within one note get `-2`, `-3`, … in declaration order.
  - The ID passes `is_valid_block_id` and never starts with `h-`, so counts, tombstones,
    rename, and task code can never mistake a section for an annotation.
  - The section's public `id` is the slug (the block ID without `s-`).
- **Duplicates:** declarations that share any key merge into the first one in sidecar
  order. The first supplies the title, tag, and slug; descriptions are concatenated with
  a blank line in between; images and tasks from all of them are kept. Reported as
  `section_duplicate`.
- **Reserved titles:** a title key of `otherhighlights` or `removedhighlights` is
  rejected (`section_reserved_title`).
- **A rejected declaration never disappears.** It renders as an ordinary sticky note
  whose first line shows the heading literally (escaped `\# …`).
- **A `# ` line in a highlight, underline, or image comment** renders literally
  (escaped) instead of being deleted, as today's filter does. Reported as
  `section_declaration_in_comment`.
- **Order:** sections render in declaration (sidecar) order. A merged section sits at
  its first declaration's position.

### Joining a section

- **Text scanned:** only the user's own words in the effective source each renderer path
  already uses:
  - a highlight's effective `task_source`/`comment`, after the `<` fold has removed its
    marker;
  - `effective_underline_source` for absorbed underlines;
  - an image's `task_source`/`comment`;
  - a non-declaring sticky's `text`.

  Image-note path lines are not scanned. Quoted PDF text is never scanned.

- **Tag token:** `@`, then a Unicode letter or digit, then letters, digits, `_`, or `-`,
  then optionally one of `.,;:!?` (part of the token). The `@` starts the line or
  follows whitespace, so email addresses never match. `\@name` is the escaped form.
- **Tag position:** a token is positional only when it belongs to the maximal run of
  tokens at the start of its line or at the end of it. A line of only tags counts as
  both. Positions are judged after:
  - (first line only) stripping a leading `Comment:` / `Note:` label;
  - stripping one leading list marker (`- `, `* `, `+ `).

  Never positional:
  - tokens on a line containing the `#task` token (a trailing `@name` there stays a task
    route, exactly as today);
  - tokens inside inline code spans;
  - tokens on fenced-code lines.

- **Allowlist:** a positional tag resolves only when its key matches a section declared
  in the same PDF. Forward references work (a tag on page 2 can name a section declared
  on page 9).
- **Consumption:** once a block's home is chosen, every positional tag in that block
  whose key belongs to the home section is removed.
  - The token goes together with its tolerated punctuation; whitespace collapses.
  - A line or list item left empty disappears. A comment left empty means no Comment
    card.
  - Unknown tags and tags for other sections stay exactly as typed.
  - A positional `\@name` renders as `@name` and never resolves.
  - A mid-sentence `@word` is always prose, left as typed.
- **A sticky whose only content is tags** does not join. Its tags stay visible
  (`section_tag_only_note`).
- **A PDF with no declarations** gets no tag consumption and no section diagnostics. Its
  render is byte-identical to today's.

### Membership: one home per displayed block

The planner runs **after** `plan_annotation_folds` and works over displayed blocks:

- a highlight head with its `<` followers and every absorbed underline of those members;
- a non-declaring sticky note;
- an image;
- a standalone underline.

Within a block, tags are read in this order:

1. the head's;
2. the followers', in sidecar order;
3. the absorbed underlines', in sidecar order.

The first resolved key is the block's home. Further tags for that same section are
consumed silently. Tags for a different section stay visible and are reported as
`section_conflict` (the first home is kept). Declaring stickies are never members. The
plan is a pure function of the sidecar annotations, so deleting a tag or a declaration
and rescanning restores the earlier layout exactly.

### The rendered region (exact contract shared by `render` and `region`)

**No valid declaration:** the current `### Page N` stream, byte for byte.

**At least one valid declaration:**

1. **Each section, in order:**
   - `### <Title>`, then a blank line.
   - The card: a depth-1 callout whose header is exactly `[!summary|section]`.
     - Its header line is `@<tag> · <N> highlight|highlights[ · <pages>]`, or
       `@<tag> · nothing tagged yet` for an empty section.
     - `<N>` counts member blocks.
     - `<pages>` is `group_page_label` over the members' page labels (`Page 4`,
       `Pages 4–5`, `pp. 4–5` for `p.`-style labels), omitted when no member has a
       label.
     - The following `> ` lines are the description, then image-note embeds and
       unresolved-image warning groups, exactly as a `[!note]` body renders them.
   - A blank line, the `^s-<slug>` line, and a blank line.
   - The members, in sidecar order. Each run of equal page labels is captioned with
     `#### <page label>` (the same label string the `### ` stream would use, including
     group labels such as `Pages 3–4`). The run resets at every H3. Member markup is
     byte-identical to today's block markup, `^h-` line included.
2. **`### Other highlights`**, only when an ungrouped block exists. The ungrouped blocks
   follow in sidecar order with the same `####` captions.
3. **`### Removed highlights`**, as today.

Blank-line spacing between headings and blocks must match today's `### Page` stream.
Reuse `push_page_heading` with a level parameter rather than hand-rolling newlines.

**Region parser signature.**

- A region is **sectioned** when at least one H3 is followed (after blank lines only) by
  a depth-1 `[!summary|section]` callout and then a `^s-<id>` line.
- In a sectioned region:
  - an H3 with such a card opens a section;
  - `### Other highlights` opens the ungrouped group;
  - `### Removed highlights` keeps its meaning;
  - `####` sets the page label, and every H3 resets it to `None`;
  - blocks are `in_preamble` only before the first H3.
- Any other H3 in a sectioned region degrades to today's behavior (a page label, no
  section).
- **Every region without a card parses exactly as today.**

### Identity rules (no churn)

1. **Grouped blocks keep their true page labels** (from `####`), never the section
   title. ID reuse keys on (kind, page label, text), so it needs exactly these labels.
2. **Matching keys ignore `@name` tokens, on both sides.** These keys are the primary
   key for sticky notes and the comment tiebreaker for highlights. Remove every
   `@name`-shaped token, escaped or not and wherever it sits, together with its
   tolerated punctuation and the whitespace before it. Apply this on both sides:
   - the sidecar head (`head_text`/`full_text` for notes,
     `head_comment`/`full_comment`);
   - the parsed existing region (`ExistingLive.text` for notes, `.comment` for
     highlights).

   If removal leaves nothing, use the unstripped key. Why it is shaped this way:
   - Ignoring tokens anywhere, rather than only at tag positions, keeps the key stable
     when beautify reflows a line and moves an unresolved tag mid-paragraph.
   - Stripping both sides means existing notes that already contain `@word` keep their
     published IDs through the normal reuse path.

3. **One key per note, everywhere.** `plan_block_ids` hashes fresh IDs and checks the
   `stable_base_block_id` shortcut from the same `head_text` it matches on. For sticky
   notes, that key (and `annotation_block_id` for notes) becomes the `@`-insensitive
   key, so the shortcut also survives tag toggles.
   - This changes only IDs that are not yet published. Already-published notes that
     contain `@word` keep their IDs through the primary match above.
   - Highlight quote keys, image IDs, and `legacy_ordinal_block_id` stay unchanged.
   - Every comparison inside the fresh-allocation loop (for example
     `conflicting_primary`) must compare keys of the same form.
4. **The declaring sticky is a card, not an `^h-` block.**
   - Tasks: its description `#task` bullets use `s-<slug>` as both the identity and the
     link block ID, mapped in `block_ids_by_annotation_order` and
     `standalone_ids_by_annotation_order`. Their 🔖 links read `[[#^s-memory|…]]` and
     stay stable across description edits.
   - A previously published `[!note]` ID (possible only if an older `bob` synced the
     sticky) is handled like an absorbed underline (`edges`): kept out of tombstones,
     with a redirect stub only when it is linked or already tombstoned.
5. **A section that disappears leaves no tombstone.** Its members fall back to Other
   highlights with their tags visible again, which explains itself. A
   `[!info] Removed section` stub appears only when the existing region had that `^s-`
   ID and vault Markdown links to it or it was already stubbed (`edges`).
6. **Routed and imported tasks never re-create.** Tag lines are never `#task` lines, and
   task identity already excludes the route suffix.

### `bob ref show` JSON contract (additive, `schema_version` stays 1)

Each shown row:

```json
"sections": [
  {
    "id": "memory",
    "title": "Memory",
    "tag": "@memory",
    "description": "Everything that belongs in the blog's memory section.",
    "block_id": "s-memory",
    "link": "[[ref/notes/sase_blog_261019#^s-memory]]",
    "count": 2,
    "pages": "Pages 4–5"
  }
],
"annotations": [{ "…": "…", "section": "memory" }]
```

- `sections` is omitted when the region has none.
- `description`, `pages`, and `images` (card image-note paths) are omitted when empty.
- `count` and `pages` are computed from the parsed member blocks, never by parsing the
  card's caption text.
- `annotations[].section` is the section `id`, omitted for ungrouped blocks.
- `annotations` follow region order, so sectioned blocks come first.
- There is no new `kind`: the Mac feed drops unknown kinds, and every consumer would
  have to filter it.
- `-N` omits `sections`. `-c` keeps the full `sections` list, because like the status
  and `excluded` counts it describes the whole region.

### Diagnostic vocabulary

Diagnostics never fail a PDF. Their style follows `continuation_dangling` and
`underline_standalone`.

| Dry-run line (`bob ref sync <pdf> --dry-run`, verbose scan)                                                      | When                                                                                                                                |
| ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `sections: 2 (Memory 2, Visuals 2)`                                                                              | At least one valid section                                                                                                          |
| `section_tag_unknown: Page 4 @memry (did you mean @memory?)`                                                     | Positional tag, PDF declares sections, no key matches. The hint appears only when a declared key or alias is within edit distance 2 |
| `section_conflict: Page 7 @memory, @visuals (kept @memory)`                                                      | A block names two sections                                                                                                          |
| `section_duplicate: "Memory" (Page 9) merged into "Memory" (Page 1)`                                             | Merge                                                                                                                               |
| `section_reserved_title: "Other highlights" (Page 1) ignored`                                                    | Reserved title                                                                                                                      |
| `section_title_empty: Page 1 "# !!!" has no letters or digits`                                                   | Rejected title                                                                                                                      |
| `section_declaration_in_comment: Page 3 "# Memory" is literal; declare sections on a sticky note`                | `# ` line in a comment                                                                                                              |
| `section_tag_only_note: Page 6 @memory (a note needs text to join a section)`                                    | Tags-only sticky                                                                                                                    |
| `section_tag_on_task: Page 5 @memory routes this #task to memory.md; put @memory on its own line to join Memory` | A `#task` route name whose key matches a declared section                                                                           |

- **Concise `bob ref scan`** adds `N sections` and `N unknown section tags`, each only
  when nonzero.
- **Scan JSON** adds `notes[].sections` and `notes[].section_tags_unknown`, each omitted
  when zero.
- **A missing route target** keeps today's error and gains a hint when the name matches
  a declared section: `…; to join section “Memory”, put @memory on its own line`.

### Compatibility and rollout

- **No feature flag is needed.** A note changes only after Bryan writes its first `# `
  sticky, so he holds the switch.
- **Install first.** Install the release on every host (Mac, athena, apollo) before
  writing the first declaration.
- **Old readers degrade without damage.** Annotation sync is the Mac's full scan
  (`decisions:references-are-ingested-at-birth`), so an outdated Linux `bob` can misread
  a sectioned note but never rewrites it. Until that host updates, `bob ref show`
  reports `annotations_status: unparsed` for the note.
- **Release train.** bob-cli ships through the daily release train
  (`decisions:bob-cli-releases-ride-a-daily-train`):
  - never edit the `Cargo.toml` version or `CHANGELOG.md`;
  - every JSON change here is additive, so no commit needs `!`.
- **The Mac app follows bob-cli.** The JSON grows additively in both directions:
  - an old app with a new `bob` shows a flat feed in the new order, losing only the
    headers;
  - a new app with an old `bob` shows today's feed.

### Refinements to the research

These are small, deliberate departures, each with its reason:

- **Matching keys ignore `@name` tokens anywhere, not only at tag positions** (identity
  rule 2). Positional stripping breaks when beautify reflows an unresolved tag into the
  middle of a paragraph.
- **A tag's tolerated trailing punctuation is consumed with it.** "Great point @memory."
  renders "Great point". Keeping the period would make the rendered text and the
  identity key disagree on whitespace before punctuation.
- **Absorbed underlines contribute tags in sidecar order, not emphasis-run order.** This
  keeps the planner pure; run order needs rendering.
- **Aliases are not persisted in the region or in JSON.** The card shows the canonical
  tag (the first alias, when one exists). Aliases matter only while matching sidecar
  text.
- **A tags-only sticky does not join; it stays visible.** Joining would render an empty
  card body.
- **A rejected declaration renders literally** instead of vanishing.
- **Mac descriptions clamp to 3 lines (compact) or 5 (wide), with a `.help` tooltip
  holding the full text.** This replaces a "More" control and matches how the inspector
  already truncates long quotes.
- **Declarations are recognized in every sidecar, including ones without page
  headings.** The rules stay uniform, and the vault-wide zero-change check covers the
  hypothetical case of an ATX document title in a page-less sidecar.

## Phases

### gates: sidecar parser gates

This phase absorbs ready bead `bob-cli-ba` (`sase bead read bob-cli-ba -r "<why>"`). It
holds the full reproduction against `~/bob/lib/docs/gastown_readme.textbundle` and
`~/bob/lib/chat/sase_listen_antennapod_setup.md`.

1. **Quote continuation (`bob-cli-ba`).** In `parse_sidecar_chunk` (`sidecar.rs`) and
   `is_quote_continuation_line` (`sidecar_render.rs`):
   - In linked-page style, a heading-shaped line directly after a quote line (no blank
     line between) continues the quote, unless it is a Highlights annotation header (a
     level-5 heading ending in `:`).
   - A comment still starts only after a blank line.
   - Treat the continued line exactly like any other bare continuation line.
2. **Level-aware page headings.** The sidecar's page-heading level is the level of its
   first page-shaped heading (`sidecar_page_heading_details`). Page-shaped headings at
   any other level are never page breaks. They stay in their chunk, where today's
   heading filter still applies until `grammar`. Use the same rule for the
   `seen_page_heading` pre-scan in `parse_sidecar_markdown`. Grep for every other caller
   of `sidecar_page_heading_details` and keep them consistent.
3. **Tests:**
   - a linked-style fixture reproducing each bead case;
   - `# P values`, `# Page turners`, and `# Page layout` stickies in linked
     (`####`-paged) and simple (`##`-paged) sidecars, which no longer split the page
     stream;
   - all existing tests pass unchanged.
4. **Vault check (read-only).** Run `bob ref scan --dry-run -f json` from this phase's
   build and from a build of its base commit, both against `~/bob`. Only the two bead
   notes may enter `notes[]`. Their one-time `^h-` re-mint and tombstone is the accepted
   cost recorded on the bead. If either old ID is linked from vault Markdown, say so in
   the phase notes.
5. Close `bob-cli-ba` with
   `sase bead close bob-cli-ba --note "<what you verified, commit>"`.

### grammar: section grammar and pure planner

1. **Sidecar model.** Add `section_heading: Option<String>` to `SidecarAnnotation`
   (`model.rs`).
   - Set it only on `StandaloneNote` chunks whose declaration line matches the contract.
     It holds the raw text after `# `, aliases included.
   - `text` becomes the description: today's filtered text, except that heading-shaped
     description lines are kept, escaped.
   - An all-heading chunk that is a declaration (`# Memory` alone) must now yield an
     annotation instead of `Vec::new()`.
   - Non-declaration stickies and every other chunk parse exactly as today.
   - A `# ` comment line (highlight or image) is kept escaped instead of filtered.
   - Update the many `SidecarAnnotation { … }` test literals; deriving `Default` and
     using `..Default::default()` is fine if it reads better.
2. **New `src/native/highlights_ref/sections.rs`** (wired into `mod.rs` like
   `continuation.rs` and `underline.rs`):
   - `section_key`,
     `scan_section_declaration(heading) -> Result<Declaration, Rejection>`, and the
     canonical tag and slug builders (the hash fallback takes the `source_pdf_value`
     string as an argument so the module stays pure).
   - `scan_section_tags(text)`: positional, escaped, and on-task token hits, each with
     its line, byte span, and key.
   - `display_source(text, home_keys) -> String`: the consumption rewrite, including
     unescaping.
   - `section_match_key(text) -> String`: the identity rule 2 normalizer. It lives here
     so `render` and any later caller share one definition.
   - `plan_sections(annotations, &folds, source_pdf_value) -> SectionPlan` with:
     - `sections: Vec<PlannedSection>`: title, tag, slug/block ID, keys, declaration
       orders, merged description, and the page label of the first declaration;
     - `declarations: BTreeSet<usize>` and `rejected: BTreeMap<usize, Rejection>`;
     - `home_of: BTreeMap<usize, usize>` (display-head order → section index);
     - `rewritten: BTreeMap<usize, String>` (annotation order → the consumed display
       source, only where it differs);
     - `diagnostics: Vec<SectionDiagnostic>` (the vocabulary above, structured: kind,
       page label, detail).
   - Displayed blocks come from the existing fold plans (`ContinuationPlan.head_of` /
     `followers`, `UnderlinePlan.host_of` / `standalone`); do not re-derive folds.
3. **Unit tests** (`tests/sections.rs`): every example in the contract, plus:
   - `< @memory` on a follower;
   - `- @memory` list lines;
   - `Comment: @memory`;
   - inline code and fenced code;
   - `@memory @visuals` (conflict);
   - same-section repeats;
   - `@Memory` / `@open_questions` folding;
   - aliases;
   - merge and reserved titles;
   - an empty title;
   - non-Latin titles (`@記憶`, hash slug);
   - slug collisions;
   - forward references;
   - email addresses;
   - escapes;
   - tags-only stickies;
   - a `- #task Draft @sase` line staying a route;
   - a PDF with no declarations producing an empty plan.
4. **The renderer is untouched in this phase.** It ignores `section_heading`, so until
   `render` lands a declaring sticky renders as an ordinary note of its description. No
   real sidecar has one, which the vault check confirms. Run the gates vault check
   again: `notes[]` must match the base build exactly.

### region: sectioned managed-region parser

In `region.rs`:

- `RegionParse` gains `sections: Vec<RegionSection>` and
  `removed_sections: Vec<String>`.
- `RegionSection` has `id`, `block_id`, `title`, `tag`, `description`, `images`,
  `unresolved_images`, and `unresolved_image_states`.
- `RegionBlock` gains `section: Option<String>` (the section `id`).

Implement the signature and rules from the region contract:

- Pre-scan for the card signature.
- Add a `region_section_id` helper for `^s-` lines, because `region_block_id` only
  accepts `h-` IDs.
- Parse the card body with `parse_callout_body` at depth 1. The caption is the header
  line's text, and `tag` is its first token when that starts with `@`.
- Collect `^s-` lines under Removed highlights into `removed_sections`.

Leave these untouched: `generated_block_ids` (still `h-` only), `annotation_counts`,
`has_managed_highlights`, and `silently_dropped_block_ids`. Cards never count as
annotations, and `highlights_count`, `✎ N`, library counts, and `doctor` keep their
meaning.

Tests use hand-written regions in the exact contract shape:

- sections plus Other plus Removed;
- an empty scaffold section;
- a section without page labels;
- a group spanning pages (`#### Pages 3–4`);
- a section titled "Page turners";
- a stray non-card H3 in a sectioned region;
- `annotation_counts` unchanged for a sectioned fixture;
- every existing region test byte-for-byte.

### render: sectioned rendering and identity

This is the core phase. Plan it before coding; the identity work is subtle.

1. **Collect, then assemble.**
   - Turn the main loop of `render_sidecar_highlights` into a collector of
     `DisplayBlock { head_order, page_label, markdown }`. Each block keeps exactly
     today's per-kind markup.
   - Remove `push_page_heading` from the loop and record the label instead.
   - With no valid declaration, assemble with `### ` captions exactly as today.
   - Otherwise assemble the sectioned layout from the contract. Cards are built with
     `push_callout_block` and `push_image_note_renders` (description images resolve
     through `append_resolved_image_notes` against the `s-` block ID).
2. **Consumption plumbing.** Wherever the renderer reads a display source, use
   `SectionPlan.rewritten` when it is present:
   - highlight members (`source` from the effective annotation);
   - absorbed underlines (`effective_underline_source`);
   - images;
   - notes.

   Do this before `split_image_note_lines` and beautify. Task sources stay raw. Rejected
   declarations and `# ` comment lines render as specified.

3. **Identity.** Apply identity rules 2 and 3 in `plan_block_ids` with the shared
   `section_match_key`. That covers note `head_text`/`full_text`, the comment
   tiebreakers, `ExistingLive.text` for notes, `ExistingLive.comment`, and every
   fresh-allocation comparison. Declaring stickies are not heads.
4. **Tasks.** Map each declaration order (merged ones included) to `s-<slug>` in both ID
   maps so `annotation_task_candidates` (`annotation_tasks.rs`) builds description tasks
   with the section as source. Add a test that the 🔖 link is `[[#^s-…|…]]` and that
   editing the description text (not the task) keeps the processed ID.
5. **Report data for `edges`.**
   - `RenderedHighlights` gains `sections: Vec<RenderedSection>` (title, tag, block ID,
     member count, and the page label of the first declaration) and
     `section_diagnostics: Vec<SectionDiagnostic>`.
   - Keep `count` (live `^h-` blocks) unchanged: cards are not annotations.
6. **Tests** (`tests/sections_render.rs` plus the existing suites):
   - Every existing highlights test and fixture under `tests/fixtures/highlights_ref`
     passes byte-identical.
   - A full render of the contract example in linked and simple sidecars.
   - Round trip: render, `parse_managed_region`, and identical blocks, page labels,
     sections, and `section` homes.
   - Rescan stability: render, then rerender against the new region, gives identical
     bytes.
   - ID stability with no tombstones and no re-created tasks across: adding a tag;
     removing it; adding the declaration after the tags exist; deleting the declaration;
     unresolved tags that reflow mid-paragraph; escaped tags; tags on `<` followers and
     absorbed underlines; and an existing published sticky ending in `@word`.
   - The twin-quote case: identical quotes on one page in different sections, with
     tag-stripped comments that differ and then that are equal. Document the residual
     swap the research accepted.
   - Empty sections and sections with only images.
7. **Vault check (read-only).** Same as gates: `notes[]` matches the base build exactly,
   because no real sidecar declares a section yet.

### show: sections in `bob ref show`

In `src/native/ref_library/show.rs`:

- Add `ShowSection` and `ShowRow.sections` (skipped when empty), plus
  `ShowAnnotation.section` (skipped when `None`). Compute `count` and `pages` from the
  member blocks with the same `group_page_label` the renderer uses; expose it
  `pub(crate)` instead of duplicating it.
- Apply `-c` and `-N` per the JSON contract.

Human output keeps today's indentation for members and adds:

```text
ANNOTATIONS · 17 highlights · 15 comments

▍Memory  @memory · 2 highlights · Pages 4–5
▍Everything that belongs in the blog's memory section.
  Page 4
    “Make a point about how …”
  Page 5
    …

▍Related  @related · nothing tagged yet

Other highlights · 13
  Page 2
    …
```

- The `▍` bar is magenta (`styler.paint("35", …)`), the title bold, the caption dim. The
  description is dim, wrapped to the width, and every line carries the bar.
- A blank line separates groups, and the page-run rule resets per group.
- With `-c`, a section header prints only when one of its members is kept.
- Unsectioned rows print exactly as today.
- In TASKS, a task whose `block_id` is a section's gets the dim suffix `   § <Title>`.

Markdown digest (`render_one_markdown`), under `### Annotations`:

- each section is `#### <Title>`;
- then an italic caption line (`*@memory · 2 highlights · Pages 4–5*`);
- then the description as `>` lines;
- then the members with today's `**Page N**` captions (reset per group);
- empty sections add `_Nothing tagged yet._`;
- then `#### Other highlights`;
- unsectioned digests stay byte-identical.

Tests: fixture notes in the contract shape (tests may write them by hand), covering:

- JSON shape and omissions;
- `-c` and `-N`;
- human output with color off and on;
- the Markdown digest;
- an unsectioned row unchanged.

Docs: update the `bob ref show` contract in `docs/ref.md` (fields, human and Markdown
descriptions). If its table needs a note, update `docs/reference/json-outputs.md`; the
schema version is unchanged.

### edges: section stubs, diagnostics, and route hints

1. **Declaring-sticky redirect.** Treat a declaring sticky like an absorbed underline in
   `plan_block_ids`:
   - match it against `existing_live` as a `note` using its description;
   - add a match to `absorbed_standalone_ids` (never tombstoned);
   - when the ID is already tombstoned or linked
     (`vault_links::markdown_links_to_block_ids`), render under Removed highlights:

     ```md
     > [!info] Section declaration
     >
     > Now the card of [[#^s-memory|Memory]].

     ^h-…
     ```

2. **Removed-section stub.** A `^s-` ID that was live in the existing region (or already
   in `removed_sections`) and is no longer declared gets a stub when it is linked from
   vault Markdown or was already stubbed:

   ```md
   > [!info] Removed section
   >
   > “Memory” is no longer declared in the Highlights notes.

   ^s-memory
   ```

   Re-declaring the section removes the stub and restores the live card with the same
   ID. Only call the vault link scan when such IDs exist.

3. **Diagnostics.** Wire `section_diagnostics` and `sections` into:
   - the per-PDF dry-run lines (`report.rs`, beside `continuation_report_lines`);
   - the concise scan counts;
   - scan JSON (`scan_json.rs`, beside `underlines_merged`, omitted when zero).
4. **Route hint.** In `annotation_tasks.rs`, pass the declared section keys through to
   `annotation_task_candidate_from_source_line`. A missing route target whose name key
   matches a section keeps today's error and gains the hint. An existing route target
   whose key matches a section emits `section_tag_on_task` (non-failing) and still
   routes.
5. **Tests:** each stub's appear, persist, and disappear rules; every diagnostic line
   and JSON field (present and omitted); the route error text; and route-plus-section
   precedence (`@memory` on its own line plus `- #task Check @sase` both work).

### docs: docs, Obsidian styling, and cross-surface checks

1. **`docs/highlights-ref-sync.md`.** Add a "Sections (`#` and `@`)" subsection after
   Underline emphasis, plus a Contents entry. It covers:
   - the two rules of thumb first;
   - the declaring and joining rules;
   - the Joining examples table (`@memory`; `@Memory Compare with table 2`;
     `Use these as placeholders @visuals`; `Keep @chat and @agent!`;
     `- #task Draft the intro @sase`; tag plus task; `@memry`; `@memory @visuals`;
     `\@memory`);
   - the rendered shape;
   - one home per block;
   - empty scaffolds and sections-first order (tip: stack the declaring stickies on page
     1 to set the outline order);
   - identity guarantees;
   - diagnostics;
   - the rollout ("install everywhere first; you hold the switch");
   - the export caveat;
   - the accepted costs (one home per block, page order lost inside a sectioned note, no
     autocomplete in Highlights).

   Also add the new scan JSON fields to the JSON table, and append the CSS below to the
   existing snippet block in the doc.

2. **Vault CSS.** Append to `~/bob/.obsidian/snippets/ref-highlights.css`, editing in
   place; vault-sync commits it. Do not touch `appearance.json`; the snippet is already
   enabled.

   ```css
   /* bob ref: a `# Name` sticky note opens a section. Its card sits right
      under the section heading: a quiet violet rule, a faint small-caps line
      with the @tag, count, and pages, then your description in muted ink.
      Yellow quotes and gold underlines stay the loudest marks on the page. */
   .callout[data-callout="summary"][data-callout-metadata="section"] {
     --callout-color: var(--color-purple-rgb);
     margin: -0.35em 0 1.2em;
     padding: 0.1em 0 0.1em 0.85em;
     border: 0;
     border-left: 3px solid rgba(var(--callout-color), 0.55);
     border-radius: 0;
     background: transparent;
   }
   .callout[data-callout-metadata="section"] > .callout-title {
     font-size: var(--font-smallest);
     font-variant: all-small-caps;
     letter-spacing: 0.06em;
     color: var(--text-faint);
   }
   .callout[data-callout-metadata="section"] > .callout-title .callout-icon {
     display: none;
   }
   .callout[data-callout-metadata="section"] > .callout-content {
     color: var(--text-muted);
   }
   ```

   Nobody has tested this CSS in Obsidian. Say so in the phase's final report, and ask
   Bryan to check reading view and Live Preview, light and dark themes, and mobile.

3. **Cross-surface tests:**
   - `bob ref rename` rewrites `[[old#^s-memory]]` and `[[old#Memory]]` like any other
     link to the note (`rename/links.rs` tests);
   - the note-to-PDF snapshot of a sectioned ref note leaves no raw `^s-` residue and
     keeps section headings (`note_snapshot/tests.rs`, which already checks `^id`
     residue);
   - `bob ref list` / reading-board counts are unchanged for a sectioned note.
4. **Final vault check** across the whole epic: `notes[]` from the epic head matches the
   pre-epic base, apart from the two `gates` notes. `just docs-check` passes.
5. **Memory**, only as approved:

   > [!decision] decision_record Write
   > `decisions:annotation-sections-are-name-keyed-folds` through `/sase_memory_write`.
   > Claim: name-keyed membership, positional tags, routes win on `#task` lines,
   > sections render first, persist nothing new. Cite the research ref and this epic's
   > commits.

   > [!decision] glossary_term Add `glossary:annotation-section` (a `# Name` sticky plus
   > `@name` tags; one home per block), linked to `glossary:reference-note`, through
   > `/sase_memory_write`.

### mac: section headers in the Bob Mac Capture Refs panel

Open `bob-mac-capture` through `/sase_repo` and read its `AGENTS.md` first. Follow its
verification rules exactly:

- `just portable-check`;
- a scratch rsync compile on the Mac with `bundle.sh` and `just typecheck-tests` for
  `BobMacCapture*` targets, deleted afterwards (never Bryan's checkout, never
  `just install`);
- if the Mac is unreachable, report that the macOS targets are unverified.

1. **Decode** (`Sources/RefsCore/RefsInspector.swift`):
   - `RefsShowRow.sections: [RefsShowSection]` (default `[]`) and
     `RefsShowAnnotation.section: String?`;
   - both lossy (`try?` with `decodeIfPresent`), so a malformed section never drops the
     row.
2. **Feed** (`RefsInspectorFeed`):
   - **Headers never become entries.** Entry indices, `total`, `RefsNotesPage`
     pagination, `RefsInspectorNoteFade`, and `RefsInspectorRevealAnchor` all key on
     flat entry indices and must not shift.
   - Instead, `Entry` gains `leadingHeaders: [FeedHeader]` (`.section(…)` or `.other`):
     - a member's section header attaches to its first member entry;
     - empty-section headers attach, in order, to the next entry;
     - trailing empty sections render after the last entry, only once nothing remains to
       reveal;
     - `.other` attaches to the first ungrouped entry, only when sections exist.
   - The page-run label rule resets at every header.
   - The disclosure text gains `· N sections` when the hidden entries include section
     starts.
   - With no sections, the feed is identical to today's.
3. **View** (`Sources/BobMacCapture/Refs/RefsInspectorView.swift` and
   `RefsVisualTokens.swift`):
   - The section header row has a 3 pt rounded leading bar, adaptive `Color.purple` at
     0.6 opacity and fully opaque under Increase Contrast. It spans the title and the
     description.
   - The title is `.subheadline.weight(.semibold)`, with `@tag · N` trailing in
     `.caption2.monospacedDigit()`, tertiary.
   - The description is `.callout`, secondary, clamped to 3 lines (compact) or 5 (wide),
     with `.help(fullDescription)`.
   - A new `inspectorSectionGap` (14) sits above every header that is not first.
   - "OTHER HIGHLIGHTS N" reuses the `HIGHLIGHTS` caption style, so the hierarchy reads
     HIGHLIGHTS → section → page.
   - Members render exactly as today.
4. **Accessibility:**
   - headers carry `.isHeader`, labelled "Section Memory, 2 highlights. <description>";
   - entry labels add the section ("Highlight, Memory, Page 4: …");
   - color is never the only cue.
5. **Copy card** (`Sources/RefsCore/RefsCopy.swift`, `RefsHighlightPicker.swift`,
   `RefsCopyCardView.swift`):
   - **A** groups `yourNotes` under `### <Title>` with the description as a `>` line,
     for sections with at least one note. Then comes `### Other highlights` (only when
     sections exist), then `### Notes` as today.
   - The **H** picker shows section titles as non-selectable group rows that never take
     a digit key.
   - Copy-one is unchanged. `G` stays free for a later section copy.
6. **Tests and renders:**
   - a `Tests/Fixtures/refs-show-sections.json` fixture (served by `fake-bob` through
     `FAKE_BOB_REFS_SHOW_FIXTURE`);
   - RefsCore tests for decoding (including malformed sections), header attachment, page
     runs, pagination and disclosure text, and Copy A/H;
   - scroll and reveal tests proving indices are unchanged;
   - `RefsPanelDesignTests` renders `renderInspectorSections` in light and dark at 880
     and 1344 pt, with one at a large accessibility text size.
7. **Docs:** in the bob-mac-capture repo, update `docs/refs-inspector.md` (the
   HIGHLIGHTS feed with section headers) and `docs/refs.md` (the Copy card). The bob-cli
   site mounts those docs; do not edit `docs/reference/mac-capture/` in bob-cli.

## Epic-level verification

- `just` CI gates in bob-cli (fmt, clippy, tests, `just docs-check`) pass on every
  phase's commit. bob-mac-capture's macOS CI passes for `mac`.
- Before any declaration exists, the read-only vault-wide
  `bob ref scan --dry-run -f json` comparison shows no notes changed beyond the two
  `gates` notes.
- A scratch end-to-end check: copy one real sidecar and PDF into a temp `BOB_DIR`, add a
  declaration and tags to the copy, then `bob ref sync`. Check:
  - the note matches the contract shape;
  - a second sync is a no-op;
  - removing the tags restores the original IDs;
  - `bob ref show -f json` and `-f markdown` show the sections.

  Never write a declaration into the real vault.

## Deferred on purpose

- Nested sections, several homes per block, and several declarations in one sticky.
- Highlight-anchored declarations.
- Positional chapters and grouping from the PDF outline.
- A section-rename tool.
- Derived `highlights_sections` frontmatter.
- `cssclasses: [bob-ref]` caption styling.
- Mac jump chips and the `G` section copy.
