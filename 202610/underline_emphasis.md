---
tier: epic
title: Underline emphasis inside highlights
goal: 'An underline drawn inside a highlight renders as bold emphasis inside that
  highlight''s quote in Obsidian, bob ref show, and the Bob Mac Capture preview. Its
  comment, tasks, and image notes join the highlight, anchors stay valid, and nothing
  is written to the PDF or the sidecar.

  '
decisions:
  quiet_quote_text:
    ask: Should the snippet also set every highlight quote in regular-weight muted
      text, so emphasis stands out more?
    default: false
    why: Keeps the ~330 quotes without emphasis as they look today; ink contrast alone
      reads clearly.
    answer: false
  decision_record:
    ask: Record the underline-emphasis rule in the decisions web once the epic lands?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: fold-render
  title: Classify, match, and fold underlines into their highlights
  depends_on: []
  size: medium
  description: 'fold-render: add underline.rs (classifier, match key, host predicate,
    word-snapped runs), plan underlines before continuations in one shared fold plan,
    render runs, comments, image notes, and task links into the host block, absorb
    underline IDs with link-aware Merged emphasis stubs, heal standalone underlines,
    and rehearse the rollout on a scratch copy of the affected notes.'
- id: visibility-docs
  title: Scan counters, terminal bold, docs, and the Obsidian snippet
  depends_on:
  - fold-render
  size: small
  description: 'visibility-docs: report merged and standalone underlines (with reasons)
    in dry-run, scan, and JSON output; render quote emphasis as ANSI bold in bob ref
    show; document the feature; install and enable the ref-highlights.css vault snippet.'
- id: mac-preview
  title: Emphasis-aware quote preview in Bob Mac Capture
  depends_on: []
  size: small
  description: 'mac-preview: a pure RefsCore excerpt that keeps late emphasis inside
    the 4-line inspector preview, a plain-text quote tooltip, and bold runs instead
    of raw `**` in the Copy card picker.'
proposed_by: bbugyi200.athena.research.4z.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 11:48:17
status: done
bead_id: bob-cli-9u
---

- **PROMPT:** [prompts/202610/underline_emphasis.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/underline_emphasis.md)
- **BEAD:** [bob-cli-9u](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9u/README.md)

# Plan: Underline emphasis inside highlights

## Why

Bryan uses Highlights underlines almost only to stress part of a larger highlight.
Highlights exports each underline as its own fully bold quote, wrapped per PDF line
(`> **…**`). bob therefore renders it as a second `[!quote]` block with its own `^h-`
ID. Ref notes and the Refs inspector show the stressed words twice: plain inside the
highlight, then again as a bold card, often with `**a** **b**` seams. When the underline
carries the comment (8 of 19 today), the thought is stranded on the fragment instead of
sitting with the passage. In Obsidian the bold barely shows: the quote is the grey
semibold callout title, and bold inside it stays grey.

Design source: the research report
`research:202610/underline_emphasis_in_highlights/underline_emphasis_in_highlights.md`.
Read it with `sase artifact read`. Bryan accepted all of its requirement adjustments,
A1–A11. Where this plan is more specific, this plan wins.

Precedent: the continuation-highlight epic (`plan:202610/highlight_continuations.md`,
bob-cli commits `3924abc8`, `0bed62a1`, `1097ae90`) built the same kind of stateless
render-time fold. Reuse its shapes: `continuation.rs`, the group render path, the
absorbed-ID set, redirect stubs, split identity/link task IDs, and its counters.

## What Bryan sees

Gas Town p. 6 today:

```md
> [!quote] Sling swarm work in Convoys. Polecats thrive on well-defined, fully-spec’ed
> beads epics. Doing it in convoys allows you to track your completed features and
> high-level work items in the dashboard.

^h-d50f0de8673f

> [!quote] **beads epics**

^h-16732d7876b7
```

After the first scan with the new binary:

```md
> [!quote] Sling swarm work in Convoys. Polecats thrive on well-defined, fully-spec’ed
> **beads epics**. Doing it in convoys allows you to track your completed features and
> high-level work items in the dashboard.

^h-d50f0de8673f
```

With the vault snippet, the emphasized words render in full ink against the grey quote,
in light and dark mode. The Mac inspector shows the same quote with the same words in
bold. If the bold run starts late in a long quote, the 4-line preview opens with a short
`… lead-in` so the emphasis stays visible.

**The one rule to learn:** a bold run inside a quote is an underline that merged; a
whole quote in bold is an underline that found no highlight around it.

| Change in Highlights                     | Result on the next scan                                                 |
| ---------------------------------------- | ----------------------------------------------------------------------- |
| Underline words inside a highlight       | Those words turn bold inside it; no extra card                          |
| Add a comment, `#task`, or image to it   | Joins the highlight's Comment callout, task 🔖, and images              |
| Delete the underline                     | Only its bold run disappears                                            |
| Delete the highlight, keep the underline | The underline returns as one clean bold block under its **original** ID |
| Change the underline's extent            | Recomputed from scratch; no formatting accumulates                      |
| The match becomes ambiguous              | Separate blocks again, with a reason in `bob ref sync <pdf> --dry-run`  |

There is no gesture, marker, flag, frontmatter key, or config. Undo means deleting
either annotation and rescanning.

## Design

### Underline rows

An annotation is an **underline row** when all of these hold:

- `is_text_highlight` holds: it is a `Highlight` and not a marker mirror.
- Its raw `text` is one or more `**X**` pairs separated only by whitespace (newlines
  included). Surrounding whitespace is allowed. No other `**` appears. Delimiters pair
  from left to right.
- At least one `X` contains non-whitespace.

Its **inner text** is the `X` pieces joined by the exact whitespace that separated them.
For example, `**a b\nc,**\n**d e-\nf**` becomes `a b\nc,\nd e-\nf`. Every later step
uses the inner text, so reflow and hyphen healing never see delimiters.

- An underline row whose note starts with a `<` continuation marker
  (`scan_continuation_marker` returns `MarkerScan::Marker`) is never absorbed and
  renders exactly as today. The explicit gesture wins. An escaped `\<` is an ordinary
  note and does not block absorption.
- Structure the classifier so a second markup kind could be added later as another
  variant. Highlights exports strikethrough as `~~…~~`. Ship underline only.

### Match key

`key(text)` works on the text after the render-only cleanup its quote gets:
`strip_return_link_glyphs` when the PDF has `return_links: true`, then
`beautify_annotation_text`. From that cleaned text:

- drop whitespace and the hyphen-like characters U+002D, U+00AD, U+2010, and U+2011;
- keep case and every other character;
- record, for each key character, its char offset in the cleaned text.

Hyphens are dropped because the two exports wrap at different points. One side can keep
`embedding-based` mid-line while the other heals `embedding-⏎based` into
`embeddingbased`.

There is no NFKC step. Both texts come from the same PDF text layer through the same
exporter. Both also pass through `clean_pdf_text_artifacts`, which already expands
ligatures and drops soft hyphens and zero-width characters. That covers the real
differences without adding a dependency.

### Finding the host

**Candidates.** The candidates for underline row U are the text highlights on U's page
that are not underline rows. "Same page" compares `page_index` when both rows have one,
otherwise `page_label`. A continuation follower can be a candidate. Image selections and
sticky notes never are.

**Containment.** A candidate contains U when `key(U)` occurs in the key of the
candidate's own cleaned text. Overlapping occurrences count separately.

**Single-word guard.** A single-word underline is one whose trimmed inner text has no
whitespace. For these, count only occurrences whose two ends fall on word boundaries in
the candidate's cleaned text. Word characters are letters, digits, `’`, and `'`. This
guard narrows the research's predicate on purpose: lone single words are where a
coincidental match is most plausible ("proves" inside "approves"). A miss stays a
visible separate block, which is better than a silent wrong merge.

| Candidates containing U | Occurrences in that candidate | Result                                          |
| ----------------------- | ----------------------------- | ----------------------------------------------- |
| 0                       | –                             | standalone: `no_host`                           |
| 2 or more               | –                             | standalone: `ambiguous_host`                    |
| exactly 1               | 2 or more                     | standalone: `repeated_in_host`                  |
| exactly 1               | exactly 1                     | absorbed; that candidate is U's **host member** |

What the rule deliberately leaves out:

- **No proximity window and no "previous" rule.** 3 of 19 real underlines sort before
  their host.
- **No fuzzy fallbacks**, LLM matching, or PDF geometry in v1.

### Drawing the emphasis

The host member renders inside a **block**: its own block, or the continuation group it
belongs to. Work on that block's final quote Q, assembled exactly as today.

1. **Window.**
   - Project Q and each member's own cleaned text into key space.
   - Members appear in Q in order, so find each member's key in `key(Q)` sequentially,
     starting from the end of the previous member. The host member's window is its
     range.
   - Gap joins add `[…]` characters between members, so search rather than add lengths.
   - A block of one uses the whole quote as its window.
2. **Find.** Locate `key(U)` inside the window. By construction it occurs there exactly
   once. Map it back to a char range `[a, b)` in Q.
3. **Snap.** Widen outward to whole words, using the word characters above. `what didn’`
   becomes `what didn’t`.
4. **Trim and split.** Trim whitespace. Split at line breaks: after beautify, paragraphs
   and list items are separate lines, so emit one run per line.
5. **Flank.**
   - The opening `**` must be CommonMark left-flanking, and the closing `**` must be
     right-flanking.
   - When an end fails, widen it outward to the nearest whitespace or line edge, as in
     `foo(bar)` → `**foo(bar)**`.
   - The real data never needs this.
6. **Merge.** Merge runs that overlap or are separated only by whitespace on one line.
   The output never contains `****` or `**a** **b**` seams.
7. **Insert** the delimiters from right to left.

An underline that covers the whole highlight makes the whole quote bold. The highlight
was marked twice, so it is still one card.

### Comments, image notes, and tasks

**Comments.** The block keeps one `[!note] Comment` callout, with its paragraphs in this
order:

1. the members' comments, exactly as today;
2. each absorbed underline's comment, in the order its run appears in Q (ties go by
   sidecar order).

Each underline comment uses today's per-annotation cleanup: `split_image_note_lines`,
then `strip_comment_label(beautify_annotation_text(remaining))`. Never deduplicate or
rewrite a comment.

**Anchors.** When the block shows two or more comment sources, or two or more runs,
prefix each underline comment with `“<first ≤6 words of its run>…” — `. Drop the `…`
when the run has six words or fewer. With one run and one comment, the two explain each
other, so no anchor is added. No block in today's vault triggers an anchor.

```md
> [!quote] … stop **everything, ignore everyone else** and read what it has to say. …
>
> > [!note] Comment Great section.
> >
> > “everything, ignore everyone else” — This feels wrong!
```

**Image notes.** An underline's image notes embed under the quote, after the members'
image notes, in run order. Call `resolve_image_note` with the underline's own order and
the block ID.

**Tasks.**

- `[h::]` (`processed_id`) is hashed from the underline's **own** raw block ID. Toggling
  the merge therefore never re-creates a task. Do not bump `HIGHLIGHT_TASK_ID_VERSION`.
- The 🔖 link targets the **block** ID, through `block_ids_by_annotation_order`.
- A task that already exists keeps its 🔖 link. The redirect stub forwards it.

### Identity and redirect stubs

- **Host and block IDs** are unchanged. They hash the raw text and the ordinal, and the
  emphasis exists only at render time.
- **An absorbed underline ID** is computed from the raw annotation, exactly as today, so
  it is the same whether merged or not. It is not rendered and is not in `current_ids`.
  It is **never tombstoned**: add it to the absorbed set that already keeps continuation
  followers out of `removed_ids`.
- **Redirect stub.** Emit an `[!info] Merged emphasis` stub for an absorbed ID only when
  both of these hold:
  1. The ID is in the existing region.
  2. Either it is already listed under `### Removed highlights` (as a stub or a
     tombstone), or some Markdown file in the vault contains `#^<id>`.

  The vault search covers `[[#^id]]`, `[[ref/…#^id|🔖]]` task bookmarks, Markdown links,
  and the provenance links that Copy-card copies carry. Run it only when at least one
  absorbed ID is still a live block in the existing region, which means it was newly
  absorbed. Reuse the vault Markdown walker in `src/native/vault_links.rs`; expose a
  small helper there rather than writing a second walker.

  ```md
  ### Removed highlights

  > [!info] Merged emphasis Now emphasized in [[#^h-cdd7eb850000|the Page 2 highlight]].

  ^h-476bb727cc72
  ```

  - **Alias:** `the <block page label> highlight`, or `the merged highlight` when the
    block has no label. This is the continuation stub's rule.
  - **Body:** pass text starting with `\n` to `push_callout_block`, as continuation
    stubs do, so the link sits on the body line.
  - **Order:** continuation stubs first, then emphasis stubs in sidecar order, then
    tombstones.
  - **Region parser:** it already treats every `^h-` under `### Removed highlights` as
    removed, so no parser change is needed.

- **Rollout produces exactly one stub:** `ref/docs/gastown_readme.md` `^h-476bb727cc72`,
  which a routed task in `sase.md` links to.
- **Why not stub every published ID, as continuations do?**
  - That would add 19 stubs to protect one link.
  - Bryan often draws the underline first, and the Mac scans every 15 minutes. A scan
    that lands between the two strokes publishes the underline on its own, and that rule
    would then leave a permanent, useless stub.

### Continuations

- Plan underlines **first**. Then run `plan_continuations` over the annotations minus
  the absorbed underlines, so absorbed underlines are invisible to the `<` "previous"
  search.
  - This fixes a latent bug. An underline can sort after its host at the bottom of
    page 11. If a `<` piece sits on page 12, today it joins the underline instead of the
    highlight around it.
- Use one entry point, for example
  `plan_annotation_folds(annotations, strip_return_links)`, that returns both plans.
  - `render_sidecar_highlights` and `annotation_task_candidates` must both use it.
  - `annotation_task_candidates` calls `plan_continuations` on its own today and has no
    return-links flag. Pass the flag in.

### Standalone underlines

An underline row that is not absorbed renders as one clean run instead of today's
`**a** **b**` seams:

1. strip return-link glyphs if needed, then beautify the inner text;
2. wrap each non-empty line in one `**…**`.

Its ID is unchanged. This applies to a block of one. A standalone underline that heads a
continuation group, or a `<`-marked underline, keeps today's assembly.

### Scope

**In:**

- Underline rows folded into same-page text highlights, including highlights inside
  continuation groups.
- Healing standalone underlines.
- Counters, docs, and terminal bold.
- The vault CSS snippet.
- Presentation-only Mac polish.

**Out:**

- **Ordinal-hash churn (`bob-cli-9o`).** Underlines still count toward
  `ordinal_on_page`. Until 9o lands, underlining an already-published highlight can
  re-mint the IDs of later blocks on that page, and the host's own ID when the underline
  sorts first.
  - Do not change the ordinal scheme here: that would re-mint IDs once across these
    notes.
  - 9o is an identity migration for every ref note and keeps its own plan. This fold
    neither causes the churn nor makes it worse.
- **A PDF-geometry check** through lopdf. This is the planned escalation if a false
  merge is ever seen.
- A Copy-card row that copies only the underlined words.
- Strikethrough.
- Extraction damage such as `embeddingbased` (`bob-cli-85`).
- bob-plugins.
- Any write to the PDF or the sidecar.

## Phase `fold-render`: classify, match, and fold

Everything lives in `src/native/highlights_ref/`.

**Byte stability.** Output must stay byte-identical for every sidecar without an
underline row. The existing render, region, sidecar, task, continuation, and image-note
tests are the regression net, and they must pass with no expectation changes. The only
permitted edits to them are call-site updates for new parameters, such as the
return-links flag on `annotation_task_candidates`.

**Lint gate.** The gate is `cargo clippy --all-targets --all-features -- -D unused …`.
Keep only what rendering reads. Standalone reasons, counters, and report fields arrive
in `visibility-docs`.

### New `underline.rs` (pure, unit-tested)

Register it in `mod.rs` the way `continuation.rs` is registered. A suggested shape
follows; adjust the names freely, but keep the module free of I/O:

```rust
/// Inner text of an underline row, or None when the quote is not fully `**`-wrapped.
pub(super) fn underline_inner(text: &str) -> Option<String>;

pub(super) struct UnderlinePlan {
    /// absorbed underline order -> host member order
    pub(super) host_of: BTreeMap<usize, usize>,
    /// underline rows that stay their own blocks, in sidecar order
    pub(super) standalone: BTreeSet<usize>,
}
pub(super) fn plan_underlines(
    annotations: &[SidecarAnnotation],
    strip_return_links: bool,
) -> UnderlinePlan;

pub(super) struct AnnotationFolds {
    pub(super) underlines: UnderlinePlan,
    pub(super) continuations: ContinuationPlan,
}
pub(super) fn plan_annotation_folds(
    annotations: &[SidecarAnnotation],
    strip_return_links: bool,
) -> AnnotationFolds;

/// Char ranges in `quote` to embolden for one absorbed underline (window → snap → flank).
pub(super) fn emphasis_ranges(
    quote: &str,
    member_texts: &[String],
    host_member: usize,
    underline_text: &str,
) -> Vec<Range<usize>>;
/// Merge ranges and insert `**` delimiters.
pub(super) fn apply_emphasis(quote: &str, ranges: &[Range<usize>]) -> String;
/// One healed `**…**` run per line for a standalone underline.
pub(super) fn standalone_underline_quote(cleaned_inner: &str) -> String;
```

### Render integration (`sidecar_render.rs`)

- Compute the fold plan once. Skip absorbed underlines, as followers are skipped today.
- For each block:
  - Collect the underlines absorbed into any of its members.
  - Compute their ranges on the final quote, apply them, and append their comments and
    image notes as described in the Design section.
  - Map every absorbed underline's order to the block ID in
    `block_ids_by_annotation_order`.
- Render standalone underline rows through `standalone_underline_quote`.
- Absorbed IDs join the set excluded from `removed_ids`.
- Emphasis stubs follow the link-aware rule. Read `removed` from `parse_managed_region`
  on the existing region.
- `count` stays `current_ids.len()`, so absorbed underlines no longer count.

### Tasks (`annotation_task_candidates`)

- Use the shared fold plan.
- Identity: `annotation_block_id(raw underline)`.
- Link: `block_ids_by_annotation_order[order]`, falling back to the identity.
- Image-selection and sticky-note candidates are unchanged.

### Tests

Unit tests go in `src/native/highlights_ref/tests/underlines.rs`, registered in that
directory's `mod.rs`. Write synthetic text modeled on the real shapes, and do not copy
third-party text into fixtures.

1. **Classifier:**
   - a single pair;
   - per-line pairs with linked-page continuation lines (`> **a\nb**` then `> **c**`);
   - pairs split by a blank line;
   - surrounding whitespace;
   - not an underline: partially bold text, an unpaired `**`, an empty `****`, a marker
     mirror;
   - a `<`-marked row classifies but is never absorbed;
   - a `\<`-escaped row can be absorbed.
2. **Key:**
   - a hyphen at a line end on one side only;
   - a soft hyphen;
   - a ligature;
   - return-link glyphs stripped on both sides when the flag is on.
3. **Host:**
   - an underline sorted after its host, and one sorted before it;
   - the same text on another page → `no_host`;
   - two same-page hosts → `ambiguous_host`;
   - the text twice in one host → `repeated_in_host`;
   - a single word `proves` beside the host word `approves` → `no_host`;
   - a single word matching a whole word → absorbed;
   - a host that is a continuation follower → absorbed into the group;
   - image selections and sticky notes never host.
4. **Drawing:**
   - snapping at the end (`didn’` → `didn’t`) and at the start;
   - flanking widening;
   - a run that crosses a paragraph or list-line break → one run per line;
   - two underlines in one host → two runs, ordered;
   - overlapping or whitespace-adjacent underlines → one run;
   - an underline covering the whole host → the whole quote is bold;
   - a host with return-link glyphs.
5. **Render golden** (synthetic shapes of Gas Town pp. 6, 7, and 10):
   - one block per host, with its run;
   - the underline's comment moves to the host;
   - host and underline comments both present → the anchor appears;
   - an image-note path on the underline embeds under the host;
   - `count` drops.
6. **Identity:**
   - The host ID is the same with and without the underline.
   - The absorbed ID is the same merged and unmerged.
   - Merge → rescan → delete the underline → rescan ends byte-identical to the pre-merge
     render.
   - Deleting the host brings the underline back healed, under its original ID.
   - A published, unlinked absorbed ID gets no stub and no tombstone.
   - A published absorbed ID that a temp-vault Markdown file links (`#^id`) gets a stub.
     The stub is sticky across rescans, and deleting the underline turns it into a
     tombstone.
   - A never-published absorbed ID produces nothing.
7. **Continuations:**
   - H on p. 11, an underline inside H sorted after it, and `<`H′ on p. 12: H′ joins H,
     and the run is bold inside the group quote.
   - A host member whose key also occurs in another member of its group: the run lands
     only in the host member's window.
8. **Tasks:**
   - An underline `#task` keeps its `processed_id` across a merge toggle.
   - Its 🔖 points at the host block.
   - Continuation, image-selection, and sticky-note behavior is unchanged.
9. **Standalone:** a healed `**a b c**` with no seams; a multi-paragraph inner text
   gives one run per line.
10. **Round trip:**
    - `parse_managed_region` on a merged block gives one Highlight whose quote contains
      `**run**`, plus the comment.
    - The stub ID lands in `removed`, and `unparsed` is empty.
    - Re-rendering with that output as the existing region is byte-identical.

Add one CLI test, `tests/cli/highlights/underlines.rs`, registered in
`tests/cli/highlights/mod.rs` and modeled on `continuations.rs`:

- A linked sidecar holds a host highlight and an underline inside it that carries a
  comment.
- `bob ref scan` writes one block with the bold run and the comment, and no separate
  bold block.
- A second scan leaves the note unchanged.
- `bob ref show -f json` lists one annotation whose quote contains the run.

### Rollout rehearsal (the live vault stays read-only)

**1. Path diff on the live vault.** Run
`BOB_DIR=~/bob cargo run -q -- ref scan --dry-run --no-hooks -f json`, and run the
installed `bob` the same way as a baseline. Diff the `notes[].path` sets.

- As of 2026-10-19, the new binary must add exactly these 7 notes:
  - `ref/blogs/karpathy_llm_wiki.md`
  - `ref/blogs/gas_town_emergency_user_manual.md`
  - `ref/blogs/steve_kinney_agent_memory.md`
  - `ref/papers/2026_goals_as_first_class_abstractions_in_human_ai_collaboration_automationxp26.md`
  - `ref/chat/sase_launch_post_outline.md`
  - `ref/chat/merge_queue_fenced_master_and_exact_sha_releases.md`
  - `ref/docs/gastown_readme.md`
- Any other added path must have a fully bold quote in its sidecar, which means Bryan
  drew a new underline.
- Paths that both runs list are pre-existing drift.

**2. Scratch vault.** Build a scratch vault in a temp directory:

- copy those PDFs with their sidecars (`.md` or `.textbundle`) to the same `lib/` paths;
- copy their ref notes to `ref/`;
- copy `sase.md`.

Run a writing `ref scan --no-hooks` with the new binary against the scratch vault, then
check:

- No fully bold `[!quote]` block is left (0 standalone).
- Each of the 19 underlines appears as a bold run inside its host quote.
- Exactly one `[!info] Merged emphasis` stub exists, for gastown_readme
  `h-476bb727cc72`.
- No new `[!warning] Removed highlight` entries appear.
- Gas Town's highlight count goes from 15 to 10.
- A second scan changes nothing.

Never run a writing scan against the live vault. Bryan's scheduled Mac scan applies the
change after deploy.

Finish with `just check`.

## Phase `visibility-docs`: counters, terminal bold, docs, and the snippet

### Counters

- **Data.**
  - `RenderedHighlights` gains `underlines_merged: usize` and
    `standalone_underlines: Vec<StandaloneUnderline>`. Each entry holds `page_label` and
    a `reason` (`NoHost`, `AmbiguousHost { hosts }`, or `RepeatedInHost`), in sidecar
    order.
  - The renderer fills both from the fold plan.
  - Mirror both on `SyncWriteReport`, filled at its two construction sites in `sync.rs`
    next to the continuation fields.
- **Report lines.** In `report.rs`, add `underline_report_lines` next to
  `continuation_report_lines`.
  - It prints `underlines_merged: N` when N > 0 or when any underline is standalone.
  - It then prints one `underline_standalone: <page label | no page> (<reason>)` line
    per standalone underline. The reasons read `no highlight around it`,
    `inside N highlights`, and `text repeats inside its highlight`.
  - Print these lines wherever the continuation lines print: the single-PDF plan, the
    scan's verbose plan, and the write report.
- **Concise scan line.** Add `1 merged underline` / `N merged underlines` and
  `1 standalone underline` / `N standalone underlines` after the continuation details,
  using `count_phrase`.
- **Scan JSON.** `ScanJsonNote` gains `underlines_merged` and `underlines_standalone`.
  - Both are skip-serialized when zero.
  - Set both at the two construction sites: the scan plan and ingest.
  - `schema_version` stays `1`.
- **Tests.**
  - Both lines appear in `bob ref sync <pdf> --dry-run`.
  - The JSON fields appear only when non-zero.
  - Extend the CLI test from `fold-render`.

### Terminal bold in `bob ref show`

In `src/native/ref_library/show.rs` `quote_lines`, render paired `**` runs as ANSI bold
when the styler is colored.

- Pair the delimiters from left to right, as `RefsQuoteEmphasis` does. An unpaired `**`
  stays literal.
- Wrap with the delimiters still in place, then convert them per line. A run that spans
  wrapped lines closes at the end of a line and reopens at the start of the next.
- Plain output (piped or `NO_COLOR`) stays byte-identical, `**` included.
- Add a `bold` helper to `Styler` if one is missing.
- Test both modes.

### Docs

- **`docs/highlights-ref-sync.md`.** Add a `### Underline emphasis` section after
  `### Continuation highlights (`<`)`, plus a Contents entry. It covers:
  - the trigger;
  - the one rule to learn;
  - the host predicate, the single-word guard, and the three standalone reasons;
  - word snapping, with a before/after example;
  - comments and anchors, image notes, and tasks;
  - absorbed IDs and the link-aware `Merged emphasis` stub;
  - the `<` interplay;
  - troubleshooting with `bob ref sync <pdf> --dry-run`;
  - the Obsidian snippet, with its full CSS and how to enable it;
  - the `bob-cli-9o` limitation.
- **Elsewhere in that file:**
  - In the Generated Body Contract's block-ID paragraph, add the absorbed-underline and
    emphasis-stub rules.
  - In the JSON output table, add rows for `notes[].underlines_merged` and
    `notes[].underlines_standalone`.
  - Add a MacBook validation bullet.
- **`README.md`.** In the `bob ref scan` bullet, add: an underline drawn inside a
  highlight renders as bold emphasis inside that highlight.

### Obsidian snippet (the epic's only live-vault write)

Create `~/bob/.obsidian/snippets/ref-highlights.css`:

```css
/* bob ref: an underline drawn inside a highlight renders as **bold** inside
   that highlight's [!quote] callout. Bring those words to full ink against
   the quote's muted text. Nested Comment callouts are not affected. */
.callout[data-callout="quote"] > .callout-title strong,
.callout[data-callout="quote"] > .callout-content > p strong {
  color: var(--text-normal);
}
```

- **Selectors.** The `>` combinators keep the rule off bold text inside nested Comment
  callouts. The rule applies to every `[!quote]` callout in the vault, which helps
  anywhere bold sits inside a grey quote title.
- **Rejected treatments:**
  - accent underlines, which read as links;
  - a green wash, which is heavy on long runs;
  - `==mark==` and `<u>`, which show as raw markup on the Mac and in copied text.

> [!decision] quiet_quote_text

If accepted, append this rule so the context recedes and the emphasis gains weight as
well as ink:

```css
.callout[data-callout="quote"] > .callout-title {
  color: var(--text-muted);
  font-weight: var(--font-normal);
}
.callout[data-callout="quote"] > .callout-title strong {
  font-weight: var(--font-bold);
}
```

**Enable the snippet.**

- Add `"ref-highlights"` to `enabledCssSnippets` in `~/bob/.obsidian/appearance.json`.
  Keep the existing entries and their order, keep the 2-space JSON formatting, and
  change nothing else. Validate the result with `jq`.
- Do not run git in the vault: the vault sync service commits and propagates both files.
- Put the same CSS in the docs section so the snippet can be restored from the repo.

### Decision record

> [!decision] decision_record

If accepted, use the `/sase_memory_write` skill to add
`sase/memory/decisions/underlines-render-as-emphasis.md` ("Underlines Inside A Highlight
Render As Its Emphasis"), then run `sase memory init`.

- **Applies to:** bob-cli `bob ref` rendering, the Bob vault ref notes, and the Bob Mac
  Capture Refs preview.
- **Claim:** A fully bold sidecar quote is a Highlights underline. When its text occurs
  exactly once in exactly one other text highlight on the same page, it renders as a
  word-snapped `**run**` inside that highlight's quote.
  - Its comment, image notes, and task links join the host block.
  - Its ID is absorbed, with a stub only when the ID was published and is linked.
  - There is no flag, marker, or config, and the Mac only presents the result.
- **Rejected alternatives:**
  - a config flag or opt-out, or a gesture;
  - never merging underlines that carry a comment;
  - `==mark==`, `<u>`, or colored accents;
  - fuzzy or LLM matching, or PDF geometry in v1;
  - merging in Swift;
  - moving quotes into the callout body;
  - stubbing every published ID.
- **Cost:** a standalone underline whose words happen to occur once in a same-page
  highlight would merge wrongly. The single-word guard and the dry-run reasons limit
  that risk. Emphasis inside quote titles also depends on the vault snippet.
- **Reopens when:** a false merge is seen (add the lopdf geometry check), or Highlights
  changes how it exports underlines.
- **Evidence:** the research ref, this plan, and `docs/highlights-ref-sync.md`.

If declined, record a `PROPOSED FOLLOW-UP:` note on this phase's bead only when no human
reviewed the plan.

### Rollout check (read-only)

`BOB_DIR=~/bob cargo run -q -- ref scan --dry-run --no-hooks -f json` must report
`underlines_merged` summing to 19 across the 7 notes and no `underlines_standalone`, as
of 2026-10-19. Explain any difference by pointing to a newly drawn underline.

Finish with `just check`.

## Phase `mac-preview`: emphasis-aware quote preview in Bob Mac Capture

Open the linked repo with the `/sase_repo` skill (`sase repo open bob-mac-capture -r …`)
and read its `AGENTS.md` first.

This phase is presentation only. The Mac never matches or merges underlines; bob does
(decision `mac-capture-is-a-thin-client`). It does not depend on the bob-cli phases:
today's standalone underline quotes already contain `**`.

1. **Excerpt (pure `RefsCore`, tested).** Add `RefsQuoteExcerpt.preview(_ quote:)` next
   to `RefsQuoteEmphasis`, with named constants `leadThreshold = 140` and
   `leadInWords = 8`.
   - Find the first strong run using the `RefsQuoteEmphasis` pairing rule. If there is
     none, or it starts within the first `leadThreshold` plain characters, return the
     quote unchanged.
   - Otherwise return `"… "` followed by the original quote from the start of the last ≤
     `leadInWords` whitespace-separated words before the run's opening `**`. Those
     lead-in words never cross a `\n`.
   - Keep the original spacing and every delimiter, so the view's attributed builder
     still sees the runs.
   - Without this, the emphasis in the Karpathy, goals-paper, and sase-launch quotes
     (starting at characters 366, 315, and 274) falls past the 4-line clip.
   - **Tests:** no run; an early run; a late run; a run at the start of a later line; a
     run with fewer than 8 words before it on its line; a fully strong quote; an
     unpaired `**`; text with `’` and emoji.
2. **Inspector.** `quoteView` displays `RefsQuoteExcerpt.preview(quote)`.
   - `.help` shows the full quote as plain text. Add a `RefsQuoteEmphasis.plainText(_:)`
     helper and reuse it in `Entry.accessibilityLabel`.
   - VoiceOver keeps the full quote.
3. **Copy card picker.** The picker's quote preview lines (`line1`, `line2`, and the
   Quote-only alternate preview) render strong runs as bold instead of raw `**`.
   - Move the inspector's private `quoteText` builder into a small shared helper under
     `Sources/BobMacCapture/Refs/` and use it in both views. Bold inside the picker's
     italic secondary text reads as bold italic.
   - Payloads and the copy tooltips keep `**`, so the clipboard stays Markdown.
4. **No new colors or rules.** The yellow rule already says "highlight", and weight says
   "emphasis".

**Verify** with the host-appropriate gate from that repo's `AGENTS.md`:

- `just portable-check` on a host with Swift (athena);
- the best-effort scratch-copy compile on the Mac for `Sources/BobMacCapture*`
  (`bundle.sh`, `just typecheck-tests`).

If the Mac is offline, say in the final report that the macOS targets are unverified.

## After deploy (Bryan; not agent work)

- Deploy the new `bob` to the Mac, the annotation writer. Its next scheduled scan
  applies the fold.
- In Obsidian on the Mac, check Settings → Appearance → CSS snippets: `ref-highlights`
  should be on. Toggle it on if a running Obsidian kept an older `appearance.json`.
- **The beauty gate:** review Gas Town pp. 4, 6, 7, and 10 and the goals paper in
  reading view and Live Preview, in light and dark mode.
- In the Refs panel, check the Karpathy note (its late emphasis should open with
  `… lead-in`) and the Copy card picker.
- Follow the `gastown_readme` task 🔖 in `sase.md`. It should land on the
  `Merged emphasis` stub and forward to the host.
