---
tier: epic
title: 'Continuation highlights: a `<` note folds a highlight into the one before
  it'
goal: 'A Highlights note that starts with `<` makes `bob ref` render that highlight
  and its notes as one quote with the highlight before it, across pages or within
  a paragraph. Block identity, tasks, and published anchors stay stable. A marker
  that cannot merge stays visible and is counted.

  '
decisions:
  decision_record:
    ask: Record a `decisions` strand for the render-time `<` continuation fold?
    memory:
    - decisions:continuation-highlights-fold-at-render
    default: false
    answer: false
phases:
- id: render-fold
  title: Parse, plan, and render continuation groups
  depends_on: []
  size: medium
  description: 'render-fold: add `page_index` to sidecar annotations, a pure `continuation.rs`
    planner (marker grammar, nearest-earlier-text-highlight walk, page-aware seams,
    range labels), and a group render path. Every text highlight renders through the
    group path, so unmerged notes stay byte-identical. Also add head-ID identity,
    redirect stubs for published follower anchors, and per-piece task identity with
    group 🔖 links, plus a Gas Town golden test and a live-vault dry-run regression
    check.'
- id: scan-visibility-docs
  title: Continuation counters, scan output, and docs
  depends_on:
  - render-fold
  size: small
  description: 'scan-visibility-docs: surface merged and dangling continuations in
    the verbose plan, write report, per-PDF scan line, and scan JSON. Document the
    gesture, seam rule, identity rules, and stubs in docs/highlights-ref-sync.md and
    the README. Record the decisions strand if accepted.'
- id: mac-pp-label
  title: Plural page labels in Bob Mac Capture
  depends_on: []
  size: xsmall
  description: 'mac-pp-label: in the linked bob-mac-capture repo, make `RefsCopyFormat.shortPageLabel`
    render `Pages 11–12` as `pp. 11–12`, matching `Page 11` → `p. 11`, with tests.'
proposed_by: bbugyi200.athena.research.4u.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 07:59:12
status: done
bead_id: bob-cli-9p
---

- **PROMPT:** [prompts/202610/highlight_continuations.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/highlight_continuations.md)
- **BEAD:** [bob-cli-9p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9p/README.md)

# Plan: Continuation highlights (`<`)

## Why

Highlights cannot make one highlight, or one note, span a page break or two separate
points in a paragraph. A PDF highlight's quads live on a single page. Bryan's first case
is the Gas Town Emergency User Manual. The vault sidecar
`~/bob/lib/blogs/gas_town_emergency_user_manual.md` has a page 11 highlight ending "…PR
Sheriff Standing Orders. On every". The page 12 highlight continues "session startup,
the sheriff checks open PRs … merging and cleanup", and its note is
`< PR Merge Master?!`. Today `~/bob/ref/blogs/gas_town_emergency_user_manual.md` renders
half a sentence under `### Page 11` (`^h-06d27eae3c84`) and half under `### Page 12`
(`^h-434a5e5a2bc4`). The `<` leaks into the comment as `Comment < PR Merge Master?!`.

Design source: the research report
`research:202610/highlight_continuation_marker/highlight_continuation_marker.md`. Read
it with `sase artifact read`. Bryan accepted all of its requirement adjustments, A1–A9.
Where this plan is more specific than the report, this plan wins.

## The contract

### Gesture

Bryan types the marker in Highlights on the **later** piece, while looking at the
overflow. The first piece is never touched. The fold happens only when `bob ref` renders
the ref note. It is stateless and deterministic, and it is undone by deleting the `<`.
Nothing is ever written to the PDF or the sidecar.

| Note typed on the later highlight                              | Result in the ref note                                  |
| -------------------------------------------------------------- | ------------------------------------------------------- |
| `<`                                                            | Joins the previous highlight; adds no comment           |
| `< PR Merge Master?!`                                          | Joins; comment paragraph `PR Merge Master?!`            |
| `- < keep going` / `Comment: < keep going`                     | Joins; comment `keep going`                             |
| `<` then a new line `- #task Check the sheriff`                | Joins; the task imports normally                        |
| `- < #task Check the sheriff`                                  | Joins; task text has no `<`                             |
| `<` then a new line `~/tmp/screenshots/x.png`                  | Joins; the image embeds under the merged quote          |
| `<… why skip this?` / `<... why skip this?`                    | Joins with a forced `[…]` gap; comment `why skip this?` |
| `\< 5 ms is the target`                                        | No merge; renders `< 5 ms is the target`                |
| `<div>`, `<-`, `<= 5`, `<3`, `<https://x.org>`, `</tmp/x.png>` | Ordinary note; `<` kept                                 |
| `…see above`, or a `<` on any later line                       | Ordinary note                                           |

### Marker grammar

The grammar runs only on `SidecarAnnotationKind::Highlight` annotations that are not
marker mirrors. A `<` on a sticky note (`StandaloneNote`) or on a TextBundle image
selection's caption is literal text: it is not matched, escaped, or counted.

It reads the highlight's `task_source`, which `strip_comment_label_only` has already
stripped of a leading `Comment:`/`Note:` label:

1. Take the first line whose trimmed text is non-empty. Every other line stays
   byte-identical.
2. Set aside leading whitespace. Also set aside at most one unordered-list prefix, using
   the same rule as `strip_unordered_list_marker`: `-`, `*`, or `+`, then whitespace.
3. At the start of the remaining body, test the tokens in this order. Each token must be
   followed by whitespace (`char::is_whitespace`, including NBSP and `\r`) or the end of
   the line:
   - **Gap token:** `<…` (U+2026) or `<...`. macOS smart punctuation turns `...` into
     `…`, so both spellings must match.
   - **Join token:** `<`. ASCII only.
   - **Escape:** `\` immediately followed by a gap or join token that meets the same
     boundary rule. Remove that one backslash and treat the note as ordinary. Only a
     would-be marker is unescaped, so `\<div>` stays exactly as typed.
4. For a token, the **effective** first line is the set-aside prefix plus the text after
   the token, with leading whitespace trimmed. If that text is empty, drop the line. The
   effective `task_source` is `None` if no non-blank line remains. The effective
   `comment` is `strip_comment_list_markers(effective task_source)`, or `None` when
   empty. The parser derives `comment` from `task_source` this same way, so the two
   always agree.

The effective note is used only when the follower actually merged, or for an escape. A
**dangling** marker keeps its raw `comment` and `task_source`, so the `<` stays visible.
The rule Bryan learns is: **if you still see a `<`, it did not merge.**

### What "previous" means

- **Head:** the follower's head is the nearest earlier annotation, in sidecar order,
  that is a text highlight and not a marker mirror. Sidecar order is Highlights' sidebar
  order. The walk skips marker mirrors, sticky notes, and image selections; those keep
  their own blocks.
- **Chains:** if that nearest text highlight is itself a follower, the new piece joins
  the same group. So A, `<`B, `<`C is one group.
- **Dangling:** if no earlier text highlight exists, the marker dangles. The highlight
  renders on its own, verbatim.
- **No distance limit.** A wide page jump shows up through the gap and the heading.
- **Placement:** the group renders at the head's position. Skipped items between members
  (images, sticky notes) render after the group, in their own sidecar order. They get a
  page heading when their label differs from the group's.

### Seam between pieces

The seam is computed between each member and the member just before it, not the head:

```text
seam(prev, next) = Gap   if next used the gap token (<… / <...)
                 = Join  if both page_index values are known and next == prev + 1
                 = Gap   otherwise (same page, skipped pages, or an unknown page)
```

- **Join:** concatenate the pieces' raw texts with `\n` and run
  `beautify_annotation_text` once over the run. If the PDF has `return_links: true`, run
  `strip_return_link_glyphs` on each piece first. The existing reflow joins the lines
  and heals PDF hyphens: `imple-` + `mentation` → `implementation`, while `Marie-` +
  `Curie` keeps its hyphen.
- **Gap:** beautify each run on its own, then join the runs with `[…]`: space, U+005B,
  U+2026, U+005D, space. If the left run already ends with `[…]` or `[...]`, or the
  right run already starts with one, join with a single space instead.
- **Why the rule errs toward the gap:** within one page, Highlights can select
  contiguous text, so a same-page split means skipped words. Across a page break the
  split was forced, so the text continues. A false gap is harmless; a false join would
  fabricate a sentence. No `<+` force-join ships in v1.

### Rendering

Gas Town after the first scan with the new binary:

```md
### Pages 11–12

> [!quote] One last Crew tip: … PR Sheriff Standing Orders. On every session startup,
> the sheriff checks open PRs … merging and cleanup
>
> > [!note] Comment PR Merge Master?!

^h-06d27eae3c84
```

The heading, quote, comment, and anchor are built like this:

- **Heading:** the group label comes from the members' page labels.
  - When all members share one label, use that label. A same-page group stays
    `### Page 10`.
  - When every distinct label yields a page number, use a plural prefix plus compressed
    ranges. The number is the text after the `is_sidecar_page_label` prefix, parsed as
    an integer. Sort ascending, dedupe, join consecutive runs with an en dash (U+2013),
    and separate runs with `, `. Examples: `Pages 11–12`, `Pages 11–13`, `Pages 11, 14`,
    `Pages 11–12, 14`. The prefix is `Pages`, or `pp.` when the head label uses the
    `p.`/`p ` form.
  - Otherwise, use the head's label.
  - The renderer already emits a heading whenever the label changes. Page 11 can
    therefore read `### Page 11` → `### Pages 11–12` → `### Page 12`. That outline is
    accurate and intended, and no heading is ever left empty.
- **Quote:** one `[!quote]` callout holding the assembled text.
- **Image notes:** taken from every member's effective note, in member order, and placed
  under the quote exactly as today. Call `resolve_image_note` with each member's own
  order and the group's block ID. Asset paths depend only on the source text, so merging
  causes no asset churn.
- **Comment:** one nested `[!note] Comment` callout. For each member in order, compute
  `strip_comment_label(beautify(remaining_text))` on its own; that is today's
  per-annotation cleanup. Keep the non-empty parts and join them with a blank line, so
  they render as separate `> >` paragraphs. Do not join first and then strip:
  `strip_comment_list_markers` is all-or-nothing per text, so that would change how the
  head's comment renders. A bare `<` adds nothing. Never deduplicate or rewrite notes.
- **Block ID:** the head's own `^h-` ID.

A same-page "points in a paragraph" group:

```md
### Page 10

> [!quote] Do this until every crew member is spinning, working on some task you gave
> them […] hook it so they will continue working on it after a crash

^h-…
```

### Identity

| ID                                         | Rule                                                                                                                                                 |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Group `^h-`                                | The head's own `annotation_block_id`, computed from the raw annotation. Adding, removing, or editing followers never changes it                      |
| Follower `^h-`                             | Computed the same raw way, so it is identical merged or not. It is **absorbed**: not rendered as a block, not in `current_ids`, and never tombstoned |
| Follower anchor that was already published | If the follower ID is in the existing region's `generated_block_ids`, emit a redirect stub under `### Removed highlights` instead of a tombstone     |
| Task `[h::]` (`processed_id`)              | Hashed from the **piece's own** ID, so toggling `<` before or after a task exists never re-creates it. Do not bump `HIGHLIGHT_TASK_ID_VERSION`       |
| Task 🔖 link                               | Points at the **group** ID, through `block_ids_by_annotation_order`, which maps every member's order to the head's ID                                |

The redirect stub is a two-line callout. Its body line carries the link, which renders
reliably in Obsidian, where a link inside a callout title might not:

```md
### Removed highlights

> [!info] Merged highlight Now part of [[#^h-06d27eae3c84|the Pages 11–12 highlight]].

^h-434a5e5a2bc4

> [!warning] Removed highlight This annotation is no longer present in the Highlights
> sidecar.

^h-028f34fba39e
```

- **Alias:** `the <group label> highlight`, or `the merged highlight` when the group has
  no label.
- **Order:** stubs come first, in follower sidecar order, then the existing tombstones
  in their existing order.
- **Parsing:** the region parser already treats every `^h-` under
  `### Removed highlights` as removed, whatever its callout says. Stubs must never go
  under any other `###` heading, because the parser would take that heading as a page
  label.
- **Behavior:**
  - Idempotent: the stub ID stays in `existing_ids`, so the stub re-renders while the
    merge holds.
  - Reversible: deleting the `<` renders the follower under its own ID again, and the
    stub disappears.
  - Deleting the follower in Highlights turns the stub into an ordinary tombstone.
  - A follower whose `<` was typed before its first scan never gets a stub.

### Scope

- **In:** text highlights as heads and followers.
- **Out:**
  - Forward markers.
  - Automatic merging of page-split pairs.
  - A `<+` force-join.
  - Config keys or CLI flags.
  - Region-format or parser changes.
  - PDF or sidecar writes.
  - Fixes for the text-extraction damage tracked in `bob-cli-85` (`Sheriff.One`,
    `B eads`).
  - Ordinal-hash churn (`bob-cli-9o`). The fold computes IDs from raw ordinals before
    folding, so it adds no new churn.
  - bob-plugins, which does not parse highlight blocks.

## Phase `render-fold`: parse, plan, and render continuation groups

Everything lives in `src/native/highlights_ref/`. Behavior must stay **byte-identical**
for every sidecar without a `<` marker. The existing render, region, sidecar, task, and
image-note tests are the regression net. They must pass with no expectation changes; the
only permitted edit is adding the new `page_index` field to struct literals.

The repo's lint gate is `cargo clippy --all-targets --all-features -- -D unused …`. Do
not leave new items or fields unused outside tests. Phase `scan-visibility-docs` adds
the counter fields. This phase may keep dangling and absorbed information inside the
plan and renderer.

### `page_index`

- In `model.rs`, add `page_index: Option<u32>` to `SidecarAnnotation` and to
  `SidecarPageHeading`. Never hash it: `annotation_block_id` stays exactly as it is.
- In `sidecar_page_heading_details` (`sidecar_render.rs`), parse the linked heading's
  destination. For example,
  `[Page 12](highlights://gas_town_emergency_user_manual#page=12)` yields `12`: decimal
  digits after `#page=`, ending at the end of the string or at `&`. Unlinked headings,
  and linked headings without `#page=`, fall back to the label's page number as defined
  under Rendering. Expose one shared `label_page_number` helper for that.
- In `sidecar.rs`, track the current heading's `page_index` next to `page_label` in
  `parse_sidecar_markdown`. Stamp it on annotations in `flush_sidecar_chunk`, the same
  way `order` and `ordinal_on_page` are stamped. Struct literals elsewhere set `None`.
  Update every `SidecarAnnotation { … }` literal: about 16 across `sidecar.rs`,
  `model.rs`, and `tests/{image_notes,sidecar,region}.rs`.

### New `continuation.rs` (pure, unit-tested)

Register the module in `mod.rs` the way its siblings are registered. A suggested shape
follows; adjust names freely, but keep the module free of I/O:

```rust
pub(super) enum ContinuationToken { Join, Gap }
pub(super) enum Seam { Join, Gap }

/// Grammar result for one highlight's task_source.
pub(super) enum MarkerScan {
    None,
    Escaped { task_source: String },          // backslash removed
    Marker { token: ContinuationToken, task_source: Option<String> }, // token stripped
}
pub(super) fn scan_continuation_marker(task_source: &str) -> MarkerScan;

pub(super) struct ContinuationPlan {
    /// head order -> followers (order + seam before it), in sidecar order
    pub(super) followers: BTreeMap<usize, Vec<(usize, Seam)>>,
    /// absorbed follower order -> head order
    pub(super) head_of: BTreeMap<usize, usize>,
    /// effective (task_source, comment) for merged followers and escapes
    pub(super) effective: BTreeMap<usize, (Option<String>, Option<String>)>,
    /// orders of highlights whose marker could not merge, in sidecar order
    pub(super) dangling: Vec<usize>,
}
pub(super) fn plan_continuations(annotations: &[SidecarAnnotation]) -> ContinuationPlan;

/// Join and gap assembly over raw piece texts (already return-link stripped).
pub(super) fn assemble_group_quote(pieces: &[(&str, Seam)]) -> String;
/// `Pages 11–12`-style label from member labels (head first).
pub(super) fn group_page_label(labels: &[Option<&str>]) -> Option<String>;
```

Give the plan an accessor such as
`effective_annotation(&SidecarAnnotation) -> Cow<SidecarAnnotation>`, which rewrites
only `comment` and `task_source`. Render and tasks then read the same effective note.

### Render integration (`sidecar_render.rs`)

- Compute `plan_continuations(&sidecar.annotations)` once.
- In the main loop, skip absorbed followers (`head_of`). For every text highlight,
  render through one group path; an unmerged highlight is a group of one. Its quote is
  `beautify(raw)`, its label is its own, and its comment is today's single-annotation
  comment. That keeps today's bytes. Image selections and sticky notes keep their
  current paths. Factor the group body out of `render_annotation_block` instead of
  duplicating callout code.
- Insert every member's order into `block_ids_by_annotation_order`, mapped to the head's
  ID.
- Use the group label for the `current_page` heading logic.
- Collect the followers' raw IDs into an `absorbed` set:
  - `removed_ids` becomes existing − current − silently dropped − absorbed.
  - Stubs go to `absorbed ∩ existing_ids`, in follower sidecar order.
  - Emit `### Removed highlights` when stubs or tombstones exist; stubs come first.
  - Build each stub with `push_callout_block`, passing text that starts with `\n` (as
    the image-note warnings do). The header then stays alone on its line, and the link
    sits on the body line.
- `count` stays `current_ids.len()`, so a group counts once.

### Tasks (`annotation_tasks.rs`)

- In `annotation_task_candidates`, compute the same plan. Read each highlight's
  **effective** `task_source`.
- Split `annotation_task_candidates_from_text` and
  `annotation_task_candidate_from_source_line` into two IDs:
  - `identity_block_id` feeds `annotation_task_processed_id`.
  - `link_block_id` becomes `AnnotationTaskCandidate.source_block_id`, the 🔖 target.
- **Highlights:** the identity is `annotation_block_id(raw)`. The link is
  `rendered.block_ids_by_annotation_order[order]`, falling back to the identity.
- **Image selections and sticky notes:** pass the same ID for both, so their behavior is
  unchanged.
- Existing tasks never have their 🔖 rewritten. A task created before a retroactive
  merge keeps pointing at the follower ID, whose redirect stub forwards to the group.

### Tests

Unit tests go in a new `src/native/highlights_ref/tests/continuations.rs`, registered in
that directory's `mod.rs`:

1. **Grammar table:**
   - Bare `<`; `< x`; `- < x`; `* < x`; `<` followed by NBSP; `<\t`; `<…`/`<...`
     followed by text and alone.
   - Escaped forms: `\< x`, `\<… x`, and `\<div>`, which is unchanged.
   - Ordinary notes: `<div>`, `<-`, `<=`, `<3`, `<https://x.org>`, `</tmp/x.png>`, a `<`
     on a later line, and `…see above`.
   - CRLF input.
   - `- < #task x` yields `- #task x`; a bare `<` with no other lines yields `None`.
2. **Planner:**
   - Chains of three.
   - A marker mirror, an image selection, or a sticky note between pieces is skipped and
     renders after the group.
   - A `<` on the first highlight, or with only images above it, dangles.
   - A `<` on a sticky note or image caption is literal and not counted.
3. **Seams and quote:**
   - Next page joins, with hyphen healing across the page turn.
   - Same page, skipped pages, and an unknown `page_index` gap.
   - `<…` forces a gap across a page turn.
   - An existing `[…]` is not doubled.
   - Return-link glyphs are stripped per piece before joining.
4. **Labels:** `Pages 11–12`, `Pages 11–13`, `Pages 11, 14`, `Pages 11–12, 14`, `pp.`
   for `p.` labels, a single shared label, and a non-numeric fallback to the head's
   label.
5. **`page_index` parsing:**
   - Linked `#page=N`, including a link whose label disagrees with N.
   - Unlinked `Page N`.
   - A label with no number.
6. **Gas Town golden test:**
   - Use the real page 11–12 sidecar text, linked-heading shape, with PDF path
     `lib/blogs/gas_town_emergency_user_manual.pdf`. Its vault-relative IDs reproduce
     the real `h-06d27eae3c84` (head) and `h-434a5e5a2bc4` (follower).
   - Assert one `### Pages 11–12` block with the joined sentence
     `… On every session startup, …` and comment `PR Merge Master?!`.
   - Given a prior region that published both old blocks, assert a redirect stub for
     `h-434a5e5a2bc4` and no tombstone for it.
   - Re-rendering with that output as the existing region is byte-identical.
7. **Identity:**
   - The head ID is the same with and without followers.
   - The follower ID is the same merged and unmerged.
   - The merge → rescan → unmerge → rescan sequence ends byte-identical to the pre-merge
     render.
   - No new tombstones appear.
   - A stub appears only for a previously published follower.
8. **Tasks:**
   - A follower `#task` keeps its `processed_id` across a `<` toggle, and its 🔖 points
     at the head.
   - `- < #task x` creates the task text `#task x`.
   - Image-selection and sticky-note task candidates are unchanged.
9. **Image notes:** screenshot paths on any piece embed under the group, in piece order.
10. **Round trip:**
    - `parse_managed_region` on the rendered group gives one Highlight block with label
      `Pages 11–12`, the joined quote, and the multi-paragraph comment.
    - The stub ID lands in `removed`, and `unparsed` is empty.

Add one CLI test, `tests/cli/highlights/continuations.rs`, registered in
`tests/cli/highlights/mod.rs` and modeled on `image_notes.rs`:

- Write a PDF with `write_highlights_pdf` and a two-page linked sidecar whose second
  highlight's note is `< PR Merge Master?!`.
- Run `bob ref scan`, then assert the ref note shows the merged block under
  `### Pages 11–12` with no `<`.
- Run `bob ref scan` again and assert the note is unchanged.
- Check that `bob ref show` lists the annotation once under `Pages 11–12`.

### Live-vault regression check (read-only)

Build the branch. Run `bob ref scan --dry-run --no-hooks -f json` against the real vault
(`BOB_DIR=~/bob`) twice:

1. With the new binary (`cargo run -q -- ref scan …`).
2. With the installed `bob`, as the baseline.

Diff the `notes[].path` sets. The only path the new binary adds must be
`ref/blogs/gas_town_emergency_user_manual.md`. Every other listed path must also appear
in the baseline, which means pre-existing drift.

Never run a writing scan or sync against the live vault. Bryan's scheduled Mac scan
applies the change after deploy.

Finish with `just check`.

## Phase `scan-visibility-docs`: counters, scan output, and docs

### Counters

- **`RenderedHighlights`:** add `continuation_count: usize`, the number of absorbed
  follower pieces. Also add `dangling_continuations: Vec<DanglingContinuation>`, holding
  `page_label: Option<String>`, in sidecar order. The renderer fills both from the plan.
  Mirror both fields on `SyncWriteReport` and fill them at the two construction sites in
  `sync.rs`, next to `image_note_count`.
- **Verbose plan and write report** (`report.rs`): add a `continuation_report_lines`
  helper next to `image_note_report_lines`. It prints `continuations: N` when N > 0 or
  any marker dangles. It then prints one
  `continuation_dangling: <page label | no page> (no earlier text highlight)` line per
  dangling marker. Print it everywhere `image_note_report_lines` prints: the single-PDF
  plan, the scan verbose plan entry, and the write report.
- **Concise per-PDF scan line** (`scan_details`): add `1 continuation` /
  `N continuations` and `1 dangling continuation` / `N dangling continuations`. Use
  `count_phrase`, placed after the image-note details and styled like `unresolved`.
- **Scan JSON** (`scan_json.rs`): `ScanJsonNote` gains `continuations: usize` and
  `continuations_dangling: usize`. Both are additive, skip-serialized when zero, and set
  at both construction sites (the scan plan and ingest). `schema_version` stays `1`.
- **Tests:** `continuations: 1` and the dangling line appear in
  `bob ref sync <pdf> --dry-run` output. The JSON fields appear only when non-zero.
  `highlights_count` counts a group once. Extend the CLI test from `render-fold`.

### Docs

- **`docs/highlights-ref-sync.md`:**
  - Add a `### Continuation highlights (`<`)` section after `### Image notes`, plus a
    Contents entry. It covers:
    - the gesture table;
    - the grammar, including the escape, the gap token, and smart punctuation;
    - what "previous" means, including chains, skipped items, and dangling markers;
    - the seam rule and why it errs toward `[…]`;
    - the heading ranges;
    - one before/after example each for a page turn, a same-page gap, and a stub;
    - the rule "if you still see a `<`, it did not merge";
    - troubleshooting with `bob ref sync <pdf> --dry-run`, which shows `continuations:`
      and `continuation_dangling:`.
  - Note that exporting Highlights sorted by color or date breaks "previous" and is
    unsupported.
  - In the Generated Body Contract's block-ID paragraph, add the absorbed-ID and
    redirect-stub rule, and the per-piece `[h::]` with group 🔖 rule.
  - In the JSON output table, add rows for `notes[].continuations` and
    `notes[].continuations_dangling`.
- **`README.md`:** in the `bob ref scan` bullet, which already mentions image notes, add
  continuation highlights: a note starting with `<` folds that highlight into the one
  before it.

### Decision record

> [!decision] decision_record

If accepted, use the `/sase_memory_write` skill to add
`sase/memory/decisions/continuation-highlights-fold-at-render.md`, then run
`sase memory init`.

- **Applies to:** bob-cli `bob ref` rendering and the Bob vault ref notes.
- **Claim:** a text highlight whose note starts with `<` folds into the nearest earlier
  text highlight at render time. The head keeps its `^h-` ID, followers are absorbed
  (with a stub only if they were published), task `[h::]` stays per piece, and nothing
  writes the PDF or the sidecar.
- **Rejected alternatives:** auto-merge without a marker, parse-time fusing that
  re-hashes IDs, a forward `>` marker, Readwise-style counters, and sidecar or PDF
  rewrites.
- **Cost:** stateless "previous" means deleting a head re-targets its follower.
- **Reopens when:** quotes are re-extracted from the PDF text layer.
- **Evidence:** this plan, the research ref, and `docs/highlights-ref-sync.md`.

If declined, a phase worker records a `PROPOSED FOLLOW-UP:` note on its phase bead only
when no human reviewed the plan.

Finish with `just check`.

## Phase `mac-pp-label`: plural page labels in Bob Mac Capture

Open the linked repo with the `/sase_repo` skill (`sase repo open bob-mac-capture -r …`)
and read its `AGENTS.md` first.

- **Change:** in `Sources/RefsCore/RefsCopy.swift`, make `RefsCopyFormat.shortPageLabel`
  map a case-insensitive `Pages ` prefix to `pp. `, so `Pages 11–12` becomes
  `pp. 11–12`. Check it before the existing `Page ` → `p. ` branch. Every other label
  passes through unchanged.
- **Already correct:** `displayPageLabel` in `RefsInspector.swift` passes `Pages 11–12`
  through as-is, since it starts with `page`, and needs no change.
- **Tests:** in `Tests/RefsCoreTests/RefsCopyTests.swift`, cover `Pages 11–12`,
  `pages 3, 5`, the existing `Page 11` and `Appendix A` cases, and a
  `RefsHighlightPicker` page label for a plural label if a test seam exists there.
- **Verify:** run the host-appropriate gate from that repo's `AGENTS.md`
  (`just portable-check` on a host without Xcode).

This phase is independent of the bob-cli phases: before the fold ships, the label simply
never occurs.

## Validation on the Mac (for Bryan after deploy; not agent work)

- Confirm that `<`, `<…`, and `\<` survive the TextBundle `text.md` export as they do in
  `.md`.
- Check the Obsidian reading view and Live Preview of the Gas Town group, a
  multi-paragraph comment, the `[…]` seam, and the stub's link.
