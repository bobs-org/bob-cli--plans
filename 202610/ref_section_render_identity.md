---
tier: tale
title: Sectioned annotation rendering with stable IDs
goal: Render declared annotation sections while preserving published annotation IDs
  and task identities through tag changes.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-bd.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-bd.4
status: done
---

- **PARENT:**
  [202610/ref_note_annotation_sections.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_note_annotation_sections.md)
- **BEAD:**
  [bob-cli-bd.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bd/bob-cli-bd.4.md)

# Sectioned annotation rendering with stable IDs

Complete phase **bob-cli-bd.4** of `bob-cli-bd`. A valid `# Name` sticky declaration
should produce a section card and group displayed annotations whose comments contain its
positional `@name` tags. Existing annotations retain their published IDs and imported
tasks retain their processed identities as tags change.

## Scope and starting evidence

This is one bounded coding task: integrate the existing pure planner with the renderer,
normalize identity consistently, and verify the resulting layout and task links. There
are no earlier work or remaining-work notes on this phase, so a child epic is
unnecessary. At planning time the checkout was clean at
`453b07e74ea1aafce753b9a8de69db680079cfe9`.

Read these audited sources when implementing:

- `sase bead read bob-cli-bd.4 -r "Need phase scope and current notes before implementation"`.
- `sase artifact read plan:202610/ref_note_annotation_sections.md "Need the shared rendering and identity contract"`.
- The epic refines the accepted research at
  `research:202610/ref_note_annotation_sections/ref_note_annotation_sections.md`:
  identity ignores token-shaped `@name` occurrences anywhere, not only positional tags.
  If that research ref initially fails, prepare the research sidecar through
  `/sase_repo` and retry the audited read; do not read its file directly.

Dependencies `bob-cli-bd.2` and `.3` are closed and their code is present:

- `src/native/highlights_ref/sections.rs` supplies `plan_sections`, `SectionPlan`,
  `PlannedSection`, and `section_match_key`. `declarations` includes rejected
  declarations too; only `sections[].declaration_orders` identifies actual cards.
- `sidecar.rs` stores a declaration's raw title in `section_heading` and its description
  in `text`; heading-shaped description/comment lines are already escaped by the parser.
- `region.rs` reads section cards, H4 page captions, member section IDs, and removed
  section IDs. `continuation::group_page_label` is already `pub(crate)`.
- `sidecar_render.rs` still streams page headings inside its per-kind loop.
  `plan_block_ids` already performs primary matching, comment tiebreaks, legacy
  matching, stable-hash reuse, and fresh allocation. Preserve that matching order.
- `annotation_tasks.rs` already takes identity/link IDs from `RenderedHighlights`;
  standalone-note tasks use `standalone_ids_by_annotation_order` for both.

The epic's decisions are final: **no memory edits**. Do not edit or propose edits to the
declined decision strand, the declined glossary term, or any other memory. Record the
two required skipped-memory `PROPOSED FOLLOW-UP:` notes on this phase if they are not
already present when implementation begins.

Later phase `bob-cli-bd.6` owns declaring-sticky redirects, removed-section stubs,
user-facing diagnostics, scan JSON/counts, and route error hints. Supply its report data
here; leave those features to that phase. The already-landed `ref show` work, Mac UI,
documentation, CSS, release version, and changelog are outside this tale.

## Implementation

### 1. Preserve a baseline, then collect displayed blocks

Before source edits, record the implementation base SHA and build/preserve its `bob`
executable in ignored scratch storage within the checkout for the required before/after
vault comparison. Preserve enough baseline evidence to check any later failure against
the unchanged tree. Use `/sase_monitor` for long builds, checks, or scans; resume the
same implementation work through its handoff.

In `sidecar_render.rs`, compute `SectionPlan` immediately after `plan_annotation_folds`,
using `source_pdf_value(config, pdf)`. Continue to use the existing continuation and
underline fold plans; section membership does not alter them.

Introduce a small `DisplayBlock { head_order, page_label, markdown }` and collect one
per displayed highlight group, standalone underline, image, or ordinary sticky in
sidecar order. Keep the current per-kind renderers, image resolution, annotation-order
ID maps, and h-ID duplicate checks. Compute group page labels in the same place as
today, but store them instead of emitting headings in the loop.

With no valid section, assemble blocks using today's H3 page stream and byte spacing.
Parameterize `push_page_heading` by level, preserving its equal-label and `None`
behavior. Preserve preamble/mirror dropping, continuation/emphasis stubs, ordinary
removed-highlight processing, and image-write deduplication.

### 2. Apply display rewrites and assemble section cards

Read `SectionPlan.rewritten[annotation.order]` wherever a display source is used:
highlight heads and followers, absorbed-underlined comments, images, and ordinary
stickies. Apply the override before `split_image_note_lines` and beautification. Keep
sidecar/task sources raw; quoted PDF text is never scanned or rewritten for membership.
A consumed tag-only comment must not leave an empty Comment callout.

Accepted declaration orders do not produce `DisplayBlock`s or h IDs. Rejected
declarations still render as ordinary sticky notes with their first line restored as
literal `\# <raw heading>` followed by their description, including when there are no
accepted sections. Do not confuse rejected declarations with cards. Preserve escaped
heading-shaped comments and descriptions; do not double-escape them.

For each planned section, in declaration order:

1. Emit `### <Title>` and the existing blank-line spacing.
2. Emit a depth-one `[!summary|section]` card with exactly this first-line caption:
   `<tag> · N highlight|highlights[ · <pages>]`, or `<tag> · nothing tagged yet`. The
   planner's `tag` already includes `@`. `N` counts displayed member blocks, and pages
   use `group_page_label` over their labels, omitted if none are present. Put the
   beautified description on following quoted lines so the region parser can distinguish
   caption from description.
3. Split and resolve image-note paths from declaration descriptions through
   `append_resolved_image_notes`, using the section block ID. Render embeds and
   unresolved-image groups with `push_image_note_renders`; keep image counts and write
   deduplication correct. Preserve images/tasks from merged declarations and the first
   declaration's page label for report data. Description `@tokens` remain literal and do
   not create membership.
4. Emit a blank line, `^s-<slug>`, and a blank line, then this section's collected
   members in sidecar order. Use H4 page captions and reset page-run state at each H3,
   including between groups with the same page label.

After sections, emit `### Other highlights` only if ungrouped blocks exist, with the
same H4 page behavior. Existing Removed highlights remains last. Reuse
`push_callout_block`, `push_image_note_renders`, and the parameterized heading helper
instead of duplicating their newline rules.

### 3. Normalize identity on both sides

Keep quote matching and hashing unchanged for highlights, image-byte identity unchanged,
and `legacy_ordinal_block_id` unchanged. Use the shared `section_match_key` consistently
in addition to the existing return-link, beautification, emphasis, and whitespace
normalization:

- Sticky `CurrentHead.head_text` and `full_text`, and `ExistingLive.text` for notes.
- Highlight `head_comment`, `full_comment`, and `ExistingLive.comment` tiebreakers.
- `annotation_block_id` for standalone notes.
- The stable-base shortcut, indexed fresh hashes, primary grouping, and every comparison
  inside allocation, including `conflicting_primary`.

Use one note identity helper/form everywhere to prevent a hash from using raw tagged
text while reuse uses stripped text. The shared normalizer strips every token-shaped
occurrence, escaped or not and anywhere in the text, with tolerated punctuation and
preceding whitespace; if nothing remains it keeps the unstripped key. Do not replace it
with positional consumption. Published tagged stickies must reuse their old IDs through
primary matching even when their old hashes differ from the new formula. Only
unpublished fresh sticky IDs may change under that formula, as explicitly allowed by the
epic.

Exclude accepted declaration stickies from current ordinary-note heads, including late
standalone fallbacks; keep rejected declarations as notes. Let the later edges phase
handle migration of a previously published declaring sticky's h ID and its redirect
policy. Maintain a clear seam for that phase rather than changing the generic
removed-highlight rules here.

Check that duplicate-quote tiebreakers compare equivalent effective comment content on
both sides. The current head/full-comment construction reads both `comment` and
`task_source`; avoid counting the same prose twice when making those keys agree with the
parsed rendered comment. Keep any necessary correction local and covered by the
distinguishable-twins test.

### 4. Section task IDs and report data

Map **every** accepted declaration order, including merged declarations, to the
section's `s-<slug>` in both public annotation-order maps. Do not let later h-ID
fallback filling overwrite those mappings. Existing `annotation_task_candidates` should
then import description `#task` lines with the section as both identity and link source;
change its plumbing only if needed. Keep route parsing and raw task text unchanged.

Add `RenderedSection` with `title`, `tag`, `block_id`, `count` (member-block count), and
`page_label` (first declaration's label). Add
`RenderedHighlights.sections: Vec<RenderedSection>` and
`section_diagnostics: Vec<sections::SectionDiagnostic>`, populated from the plan and
actual displayed members. Cards never enter `current_ids`, `count`, or
`generated_block_ids`. Do not expose new report fields or CLI options in this phase.

Update module imports and narrow/remove the planner's temporary dead-code allowance
where production integration makes it unnecessary. If symbols still await edges
consumers, do not leave a suppression tied to this closing phase: resolve it or
associate it with still-open `bob-cli-bd.6`/`bob-cli-bd` using the repository's existing
mechanism.

## Verification and acceptance

Add `src/native/highlights_ref/tests/sections_render.rs`, register it in `tests/mod.rs`,
and use existing temporary-vault, image, render, and task helpers. Add fixtures under
`tests/fixtures/highlights_ref` where readable expected output helps. Prefer full-output
and parsed semantic assertions to checking only substrings.

- **Compatibility:** existing Highlights tests and fixtures pass unchanged. Add a
  no-declaration output regression spanning repeated/absent page labels, comments,
  images, stickies, continuations, underlines, unresolved tags, and task routes. No
  declarations means no tag consumption or section-specific layout; ordinary undeclared
  notes and their published IDs stay unchanged.
- **Layout:** linked-page and simple-page versions of the epic's Memory example; section
  declaration ordering; merged declarations; empty sections; image-only members;
  description images/unresolved images; literal description headings and tags; rejected
  declarations with and without valid sections; Other omitted when empty; repeated page
  captions reset at each section; absent and grouped page labels. The card caption must
  round-trip without becoming description prose.
- **Membership:** head/follower/absorbed-underline tags, standalone underlines, image
  comments, ordinary sticky notes, aliases/forward references, conflicts, unknown tags,
  escapes, and tags-only stickies. Use planner results rather than introducing a second
  grammar.
- **Round trip:** render, parse with `parse_managed_region`, and assert exact live block
  identities, true page labels, section cards and homes, no unparsed residue, and
  unchanged annotation counts. Rerender against that region and assert identical bytes
  for unambiguous identities.
- **Identity transitions:** add/remove tags, add a declaration after tags exist, remove
  a declaration, preserve an old published sticky ending in `@word`, tags reflowing into
  prose, escaped tags, and tags on followers/absorbed underlines. Assert the same h-ID
  set and task processed IDs, and no false Removed highlight tombstones or duplicate
  tasks. Separate genuine deletion/migration of a declaration from member stability;
  legacy declaration redirects belong to edges.
- **Twin quotes:** identical quote/page pairs in different sections with distinct
  stripped comments preserve their ID association through section reordering. Also test
  equal stripped comments: IDs remain unique and no annotation is lost. Document the
  accepted residual order-based swap of IDs/links between otherwise indistinguishable
  twins; do not invent persisted identity or promise association stability for that
  ambiguous case.
- **Tasks:** declaration descriptions, merged declarations, and a real routed task in
  the temporary vault. Verify the rendered source link is `[[#^s-memory|🔖]]` (or the
  existing routed-note equivalent), maps agree, and editing surrounding description
  prose preserves the unchanged task's processed ID and creates no duplicate import on
  the next sync.

Run the focused Highlights suite while iterating, then `just check` and
`just docs-check` as required by the epic. Do not expand scope to unrelated failures. A
failure that reproduces identically on the clean implementation base must be recorded as
`PROPOSED FOLLOW-UP:` on this phase, with the reproduction and an existing tracking bead
if known; it does not prevent this phase from closing.

Build the final binary and compare its `bob ref scan --dry-run -f json` against the
preserved base binary on the same real vault snapshot/state. Compare `notes[]` exactly
and report scanned/unchanged/failure counts, ignoring volatile top-level timestamps.
Both predecessors most recently reported 519 unchanged PDFs and empty `notes[]`; use
current evidence rather than assuming those counts. If the vault changes between reads,
repeat the pair against a consistent state. This is a read-only check: never add
declarations, sync annotations, or otherwise mutate the real vault. End-to-end writes
belong only in test/scratch vaults.

## Completion

Record verification and any limitations on `bob-cli-bd.4`. Do not create task beads; all
discovered follow-ups are `sase bead note bob-cli-bd.4 'PROPOSED FOLLOW-UP: …'`. Do not
edit the epic's decisions or any memory.

Immediately before closing, run `sase bead epic-symbols bob-cli-bd.4`. Remove each
remaining dependency on the closing phase or re-key its Justfile entry to an appropriate
still-open bead, and rerun the check. Planning found no such entries. Then close
**only** `bob-cli-bd.4` with
`sase bead close bob-cli-bd.4 --note "<implemented behavior and verified results>"`. Do
not close `bob-cli-bd` or any ancestor plan bead, and do not hand-set status. Use
`/sase_final` for the implementation turn's normal final declaration.
