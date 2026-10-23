---
tier: epic
title: Tucked continuation groups — keep every `<` highlight distinct inside its group
goal: "A `<` note groups a highlight with the one before it without erasing either: the
  group renders as one quote card under one heading and one `^h-` ID, and inside it
  every highlight keeps its own words, its own page, and its own images and comment, in
  reading order. Existing ref notes migrate on the next scan with no block-ID churn, no
  tombstones, and no change to any note that has no group. `bob ref show` (and, if
  approved, the Bob Mac Capture Refs inspector) shows the same pieces.

  "
decisions:
  mac_inspector:
    ask: Also show grouped pieces distinctly in the Bob Mac Capture Refs inspector?
    default: true
    why:
      The inspector shows the same fused quote today; one look across both surfaces is
      intuitive
    answer: true
  install_css:
    ask:
      Install the updated ref-highlights CSS snippet into the live vault
      (~/bob/.obsidian/snippets)?
    default: true
    why: The dashed seam and page caption only appear once the vault carries the snippet
    answer: true
  memory_decision_record:
    ask:
      Record the "groups tuck pieces, never fuse them" rule as a new decisions-web
      record?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: tucked-groups
    title: Render, parse, and identify tucked groups in bob-cli
    depends_on: []
    size: medium
    description:
      "tucked-groups: replace the fused group quote with a head quote callout that tucks
      each follower as its own nested quote callout (own page metadata, images, and
      comment); teach the region and note-snapshot parsers the new shape; keep block IDs
      stable through a legacy fused-key match; reword follower stubs; rewrite the
      continuation docs and CSS snippet; verify on a scratch copy of the real notes."
  - id: show-pieces
    title: Expose group pieces in bob ref show
    depends_on:
      - tucked-groups
    size: small
    description:
      "show-pieces: add an additive `pieces` array to `bob ref show` JSON for groups,
      keep the flattened top-level fields, and print each piece with its own comment and
      a page seam in the human and Markdown formats; document it in docs/ref.md."
  - id: mac-inspector-pieces
    title: Render group pieces in the Bob Mac Capture Refs inspector
    depends_on:
      - show-pieces
    size: medium
    description:
      "mac-inspector-pieces: in bob-mac-capture, decode the optional `pieces` array and
      render a group as one HIGHLIGHTS entry whose pieces each carry their own quote
      rule and note card, separated by a captioned dashed seam; update Copy payloads,
      VoiceOver labels, fixtures, tests, and docs/refs-inspector.md."
proposed_by: bbugyi200.athena.12y
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 05:11:19
status: wip
---

- **PROMPT:**
  [prompts/202610/tucked_continuation_groups.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/tucked_continuation_groups.md)

# Plan: Tucked continuation groups

## The problem

`bob ref` folds a highlight whose Highlights note starts with `<` into the highlight
before it (`src/native/highlights_ref/continuation.rs`, rendered by the group path in
`src/native/highlights_ref/sidecar_render.rs`). The fold _fuses_ the pieces:

- `assemble_group_quote` concatenates every piece into one quote, heals hyphens across
  the seam, and inserts `[…]` on gaps, so you cannot see where one highlight ends and
  the next begins.
- Every member's comment and image note is pooled under the fused quote, so you cannot
  tell which highlight a note was written on.

Real example (`ref/blogs/gas_town_emergency_user_manual.md`). The sidecar has
`…Standing Orders. On every` on page 11, and `session startup, the sheriff checks…` on
page 12 with the note `< PR Merge Master?!`. Today it renders as:

```md
### Pages 11–12

> [!quote] One last Crew tip: … PR Sheriff Standing Orders. On every session startup,
> the sheriff checks open PRs and divides them into easy wins and those needing human
> review, then slings the easy wins to other crew for merging and cleanup
>
> > [!note] Comment PR Merge Master?!

^h-06d27eae3c84
```

The comment looks like it is about the whole passage. It was written on the page-12
piece. The same thing happens in `redesigning_source_control.md` (a screenshot and
"Changesets are what 'patches' should be!" on follower pieces),
`solving_code_review_at_anthropic.md`, and `sase_blog_261019.md`.

Bryan's intent for `<` was to **group** highlights, not merge them. Each highlight must
keep its identity.

## The design: a tucked group

**One rule:** _every piece renders exactly as it would on its own; the group only tucks
the later pieces inside the first one's card._

- **Grouping** — one quote card per group, under the group's page-range heading
  (`### Pages 11–12`) and one `^h-` block ID (the head's, unchanged).
- **Identity** — each piece keeps its own words (no fusing, healing, or `[…]`), its own
  page (as callout metadata), and its own image notes and Comment callout, directly
  under its own words.
- **Reading order** — head words → head images → head comment → each follower (words →
  images → comment), in sidecar order.

### Rendered Markdown

The gas town example becomes:

```md
### Pages 11–12

> [!quote|Page 11] One last Crew tip: … PR Sheriff Standing Orders. On every
>
> > [!quote|Page 12] session startup, the sheriff checks open PRs and divides them into
> > easy wins and those needing human review, then slings the easy wins to other crew
> > for merging and cleanup
> >
> > > [!note] Comment PR Merge Master?!

^h-06d27eae3c84
```

A three-piece group with a head comment, a screenshot on the middle piece, and a comment
on the last piece:

```md
### Pages 6–8

> [!quote|Page 6] In Scion, the unit of work is **a “changeset,” an entry in a
> write-ahead log.** … along with
>
> > [!note] Comment Changesets again.
>
> > [!quote|Page 7] content-addressed pointers to the compressed file contents in blob
> > storage. …
> >
> > ![[ref/blogs/redesigning_source_control.assets/20261020_182622-dcaced15.png|568]]
>
> > [!quote|Page 8] support refs, but since neither creating nor fetching commits
> > involves them, …
> >
> > > [!note] Comment Changesets are what “patches” should be!

^h-40fb23af3c7b
```

Format rules:

- A highlight that is not in a group renders **byte-identically to today**: no metadata,
  no new lines. Only notes that contain a group change.
- Every piece of a group (head included) carries `|<page label>` metadata with the
  piece's own sidecar page label, verbatim. Omit the metadata (plain `[!quote]`) when
  the piece has no label or the label contains `]`, `|`, or a line break. Obsidian hides
  callout metadata and exposes it as `data-callout-metadata`, so it costs nothing
  visually and lets the parser recover each piece's page.
- Head content uses today's depths: quote at 1, image embeds at 1, warning and Comment
  callouts at 2. Each follower is a depth-2 `[!quote…]` callout after a bare `>` spacer.
  Its content is one level deeper: quote and image embeds at 2, warning and Comment
  callouts at 3. Followers are siblings, never nested in each other.
- The group heading (`group_page_label`) and the placement of skipped image selections,
  sticky notes, and marker mirrors (after the group, in sidecar order) are unchanged.

### How it looks in Obsidian

Without any CSS, the group is one gray quote card with each later piece in its own inner
quote card, and each blue Comment card sits inside the piece it belongs to. With the
updated `ref-highlights` snippet, the inner cards dissolve into **one card stitched by
dashed seams**. Each seam is captioned faintly at its right end with the page the next
piece was highlighted on, like a page turn:

```text
┌───────────────────────────────────────────────────────────────────┐
│ ❝ One last Crew tip: … PR Sheriff Standing Orders. On every        │
│ ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ ᴘᴀɢᴇ 12 ┄ │
│ ❝ session startup, the sheriff checks open PRs and divides them … │
│   ┌──────────────────────────────────────────────┐                │
│   │ ✎ Comment PR Merge Master?!                   │                │
│   └──────────────────────────────────────────────┘                │
└───────────────────────────────────────────────────────────────────┘
```

Starting snippet, appended to the documented `ref-highlights.css` (tune spacing if
needed; keep it scoped to quote callouts nested in quote callouts, which only `bob ref`
groups produce — the vault has none today):

```css
/* bob ref: a `<` group tucks each later highlight into the first
   highlight's quote callout. Draw the group as one card: every tucked
   highlight opens at a dashed seam, captioned at the right with the page
   it was highlighted on, and keeps its own Comment callout. */
.callout[data-callout="quote"] .callout[data-callout="quote"] {
  margin: 0.8em 0 0;
  padding: 0.8em 0 0;
  border: 0;
  border-top: 1px dashed rgba(var(--callout-color), 0.6);
  border-radius: 0;
  background: transparent;
}
.callout[data-callout="quote"]
  .callout[data-callout="quote"][data-callout-metadata]:not(
    [data-callout-metadata=""]
  )::before {
  content: attr(data-callout-metadata);
  float: right;
  margin: -0.55em 0 0 1em;
  font-size: var(--font-smallest);
  font-variant: all-small-caps;
  letter-spacing: 0.06em;
  line-height: 1;
  color: var(--text-faint);
}
```

The existing underline rule (`.callout[data-callout="quote"] > .callout-title strong`)
already matches tucked pieces, so underlined runs keep their gold rule inside followers.

### Grammar

Unchanged: the marker is still typed on the later piece, previous-text-highlight
chaining (A, `<`B, `<`C is one group), dangling markers, escapes, and the rule that
sticky notes and image captions are never scanned. Since pieces never fuse, nothing is
left to force a gap: `<…` and `<...` stay accepted as spellings of `<` (existing PDFs
keep grouping) and no longer add `[…]`.

### Identity and migration

- The group's block ID stays the head's ID. Absorbed follower IDs, follower redirect
  stubs, task 🔖 links to the head block, and per-piece `[h::]` processed IDs are all
  unchanged.
- Block IDs inside a callout are not reliably linkable in Obsidian, so pieces get no IDs
  of their own. The head ID plus the visible piece is the anchor.
- The follower stub's wording follows the new vocabulary:

  ```md
  > [!info] Grouped highlight
  >
  > Now shown in [[#^h-06d27eae3c84|the Pages 11–12 group]].
  ```

  The alias falls back to `the grouped highlight` without a label. `Merged emphasis`
  stubs (underlines really do merge) are unchanged.

- Scan re-renders every note and writes only when bytes differ, so the four notes with
  groups migrate on the Mac's next `bob ref scan` and no other note changes. The ID
  planner must recognize both the old fused block and the new tucked block (see the
  phase), so migration keeps every ID.
- Counts and machine contracts are unchanged: `highlights_count` and `count` count a
  group once; `continuations` / `continuations_dangling` and the dry-run lines stay;
  scan `schema_version` stays 1.

### Rejected alternatives

- **Keep the fused quote, add `“anchor…” —` prefixes to pooled comments** (the underline
  trick): the piece boundaries in the quote stay invisible. This is the "merge" Bryan
  never wanted.
- **Separate top-level blocks, each with its own `^h-` ID, joined only by CSS**: the
  grouping disappears without the snippet, and it changes page headings and IDs for
  every existing group.
- **Flat paragraphs inside one callout**: Obsidian styles the first line as the callout
  title, so the head and the followers would look different, and comments at the end
  would stay ambiguous.
- **Per-piece `^id` inside the callout**: Obsidian does not reliably resolve block IDs
  nested inside callouts.

## Phase tucked-groups — render, parse, and identify tucked groups (bob-cli)

Read `src/native/highlights_ref/continuation.rs`, the group path and `plan_block_ids` in
`sidecar_render.rs`, `region.rs`, `src/native/highlights_ref/tests/continuations.rs`,
and `tests/cli/highlights/continuations.rs` first.

### Renderer (`sidecar_render.rs`)

1. Refactor the `SidecarAnnotationKind::Highlight` group branch to build one
   `RenderedPiece` per member:
   `{ page_label, quote, image_note_renders, comment_parts }`. Each piece is produced
   exactly as that highlight would render alone:
   - quote = `beautify_annotation_text` of the member text (return-link glyphs stripped
     when `strip_return_links`), or `standalone_underline_quote` when the member is a
     standalone underline;
   - absorbed-underline emphasis hosted by this member is applied to this piece only
     (`emphasis_ranges(&quote_i, &[member_text_i], 0, …)`; the function already supports
     a single-member window);
   - image notes come from the member's effective source (`effective_annotation`, so the
     `<` is already stripped) plus image notes of underlines hosted by this member;
   - comment parts are the member's comment plus underline comments hosted by this
     member. The `“anchor…” —` rule (≥2 parts or ≥2 merged runs) is evaluated **per
     piece**.
2. Replace `render_highlight_group_block` with a renderer over `&[RenderedPiece]`. A
   one-piece slice must produce today's bytes exactly. Generalize
   `push_image_note_renders` and the Comment-callout helper to take a base depth instead
   of hard-coding 1/2.
3. Emit `|<page label>` metadata on every piece of a 2+ piece group, per the format
   rules above. Never on single highlights.
4. Stop using `assemble_group_quote` for display. No `[…]`, no cross-piece healing.
5. `continuation_redirect_stubs` rendering: `[!info] Grouped highlight` /
   `Now shown in [[#^<head>|the <label> group]].`

### Region parser (`region.rs`)

1. Strip `>` levels generally (any depth), not just 1 or 2.
2. Teach `split_callout_header` the `[!name|metadata] rest` form: return the name
   without metadata plus the metadata. `quote|Page 11` must classify as a quote.
3. Highlight grammar: head piece (quote text, image embeds, warning groups, at most one
   Comment), then zero or more follower groups. A follower is a depth-2 `[!quote…]`
   callout after a bare depth-1 spacer. Parse it with the same piece grammar one level
   deeper. Anything after a follower other than another follower is unparsed: keep it in
   `unparsed` as raw text, never drop it silently.
4. Add
   `pub(crate) struct RegionPiece { page_label: Option<String>, quote: String, comment: Option<String>, images, unresolved_images, unresolved_image_states }`
   and `RegionBlock.pieces: Vec<RegionPiece>`. Empty for every block without followers.
   For a group, the head comes first. For groups, the block-level fields hold the
   flattened view so every existing consumer (`annotation_counts`, doctor, `ref show`,
   rename) keeps working unchanged:
   - `quote` = piece quotes joined with `"\n\n"`;
   - `comment` = non-empty piece comments joined with `"\n\n"`, or `None`;
   - `images` and `unresolved_*` = concatenated in piece order;
   - `page_label` stays the H3 heading label;
   - mirror detection uses the head comment as today.

   Old fused blocks parse exactly as today, with `pieces` empty.

### Note snapshot (`note_snapshot/obsidian.rs`)

`parse_callout_header` rejects any kind containing `|`, so `[!quote|Page 11]` would
print literally in snapshots. Accept `[!kind|metadata] title`: drop the metadata and
keep the kind before the `|`. This is Obsidian's own syntax, so it also fixes
user-authored `[!info|wide]`-style callouts. Add a unit test.

### Block-ID identity (`plan_block_ids`)

1. Add one shared `group_identity_text(pieces: &[&str]) -> String`. For one piece it
   returns `identity_quote_key(piece)`, exactly today's key. For 2+ pieces it joins the
   per-piece keys with a separator that never appears inside a key, and it must keep the
   existing "existing key starts with head key + whitespace" prefix fallback working (a
   `\n` separator does both).
2. `ExistingLive.text`: `group_identity_text` over the parsed `pieces` when present;
   otherwise today's `identity_quote_key(quote)`.
3. `CurrentHead.full_text` = `group_identity_text` over the member texts (return-link
   glyphs stripped, as today). Add `legacy_full_text` = today's
   `identity_quote_key(assemble_group_quote(…))`. Keep `assemble_group_quote`, `Seam`,
   and `seam_between` only for this, documented as the migration key for pre-tucked
   notes.
4. After the full-text pass, add a pass that pairs still-unassigned group heads with
   unmatched existing blocks by `legacy_full_text`, using the same secondary-key and
   order pairing. All later fallbacks stay as they are.

### Tests

Update the existing continuation tests to the tucked shape, and add:

- **Golden:** the gas town fixture renders the exact tucked block, with the comment
  inside the page-12 piece and `### Pages 11–12` above it.
- **Round trip:** `parse_managed_region` on rendered groups: nothing unparsed, one
  highlight block, `pieces` with page labels `Page 11`/`Page 12`, the comment on piece 1
  only, and the flattened fields as specified.
- **Migration:** a prior region holding today's fused rendering (capture the current
  renderer's exact output as a literal fixture _before_ changing the renderer) plus a
  follower stub. Re-rendering keeps the head ID, keeps the stub (reworded), and adds no
  `Removed highlight`. A second render is byte-identical.
- **Toggle:** the merge → unmerge → merge stability test still passes.
- **Three pieces:** a head comment, an image note on the middle piece, and a comment on
  the last each land in their own piece, in order.
- **Same-page and `<…`:** same-page `<` and `<…` produce no `[…]` and keep pieces
  separate.
- **Underline into a follower:** emphasis lands in the follower quote, its comment in
  the follower's Comment callout; an underline into the head is unaffected.
- **Tasks:** a follower `#task` still links the head ID with an unchanged processed ID.
- **No churn elsewhere:** every non-continuation test passes without fixture edits. If
  one needs an edit, single-highlight output changed — fix the code, not the fixture.
- **CLI** (`tests/cli/highlights/continuations.rs`): tucked block, idempotent second
  scan, and `ref show` still reports one highlight whose quote holds both pieces.

### Docs

- Rewrite `docs/highlights-ref-sync.md` § "Continuation highlights (`<`)". Keep that
  heading text so `#continuation-highlights` stays valid. Cover:
  - the intro in group/tuck vocabulary;
  - the table (`Joins` → `Groups`; `<…` is a spelling of `<`);
  - the rules (no seams);
  - the before/after examples above;
  - the `Grouped highlight` stub;
  - unchanged troubleshooting.

  Update the CSS snippet section with the appended rules, and sweep that doc for
  "fold/merge/join" wording about continuations.

- Run `just docs-check`.

> [!decision] install_css Write the full documented snippet to
> `~/bob/.obsidian/snippets/ref-highlights.css`. That is the only write outside the
> workspace: the vault's own git sync commits it, so do not run git in `~/bob`.

> [!decision] install_css = no Change only the documented snippet. Tell Bryan in the
> phase report to copy it into the vault.

### Real-notes check (read-only on the vault)

Copy into a scratch bob dir — never write `~/bob` here — the PDFs, sidecars, and ref
notes for every library entry whose sidecar has a `<` note:

- `lib/blogs/gas_town_emergency_user_manual`
- `lib/blogs/redesigning_source_control`
- `lib/blogs/solving_code_review_at_anthropic`
- `lib/notes/sase_blog_261019`

Run the workspace build's `bob ref sync <pdf> --bob-dir <scratch>` on each, then
confirm:

- the managed region's set of `^h-` IDs is identical before and after;
- no new `[!warning] Removed highlight`;
- `## Tasks` is untouched;
- every group is tucked;
- a second sync is a no-op.

Record the results on the phase bead.

### Gate

`just check`. Conventional commits such as
`feat(highlights): tuck continuation pieces instead of fusing them` — no `!`. The
generated Markdown changes only for groups, and `ref show` JSON stays compatible.

## Phase show-pieces — expose group pieces in `bob ref show` (bob-cli)

In `src/native/ref_library/show.rs`:

1. Add `ShowPiece { page_label, quote, comment, images, unresolved_images }` and
   `ShowAnnotation.pieces: Vec<ShowPiece>`, serialized only when non-empty
   (`skip_serializing_if = "Vec::is_empty"`). Fill it from `RegionBlock.pieces`. The
   top-level `quote` / `comment` / `images` / `unresolved_images` keep the flattened
   group view, so current consumers still see every word. For groups `quote` now
   separates pieces with a blank line instead of fused text: a deliberate, compatible
   content change under `schema_version` 1.
2. Human format: for a group, print each piece's wrapped quote, image lines, and cyan
   `↳` comment in order. Precede each follower with a dim seam line at the quote indent:
   `┄ Page 12`, or a bare `┄` without a label. Single annotations print exactly as
   today.
3. Markdown format: each piece is its own `>` quote, then its image lines and `Comment:`
   lines. Precede each follower with an italic `*┄ Page 12*` line.
4. `-c/--comments-only` keeps a group when any piece has a comment or image note, and
   shows all of its pieces.
5. Tests:
   - JSON carries `pieces` for a group and omits it for single highlights;
   - human and Markdown output for a group, plus an unchanged single;
   - `-c` keeps a group commented only on a follower.
6. Docs: in `docs/ref.md`'s `ref show` field list and human-output paragraph, document
   `pieces` (present only for `<` groups; head first; per-piece `page_label`) and the
   seam lines. Run `just docs-check` and `just check`.

## Phase mac-inspector-pieces — Refs inspector shows pieces (bob-mac-capture)

> [!decision] mac_inspector = no Close this phase without code changes. File a `feature`
> task bead with `/sase_new_task`: "Refs inspector: render `ref show` `pieces` for `<`
> groups".

Open the repo with `/sase_repo` (`bob-mac-capture`) and follow its `AGENTS.md`. Never
land Swift you have not compiled: run `just portable-check`, and do the best-effort Mac
typecheck for `Sources/BobMacCapture*` changes. The app stays a thin client: it renders
what `bob ref show` returns and never parses notes.

1. `RefsCore/RefsInspector.swift`: decode an optional `pieces` array (`RefsShowPiece`:
   `page_label`, `quote`, `comment`, `images`, `unresolved_images`) on
   `RefsShowAnnotation`. A missing array (older `bob`) keeps today's behavior.
2. HIGHLIGHTS feed (`RefsInspectorView.swift` and its model):
   - a group stays **one entry** (paging, "Preview highlights per page", and the
     disclosure counts count it once);
   - inside it, each piece renders like an ordinary highlight: its quote with the 2 pt
     yellow rule and balanced `**` underline styling, and its own pale-yellow note card
     with its images and comment directly beneath;
   - followers start at a dashed hairline seam with the piece's page label as a dim
     caption at the trailing end, mirroring Obsidian;
   - the header note chip counts one note card per commented piece.
3. VoiceOver: one element per piece, e.g. "Highlight, continued on Page 12: …. Your
   note: …".
4. Copy (`RefsCopy.swift`):
   - a group's `noteBlock` / `highlightBlock` emit its pieces in order, each quote
     followed by its own `Comment:` lines, with a blank line between pieces;
   - "Quote only" joins piece quotes with a blank line;
   - `commentedHighlights` includes a group when any piece is commented.
5. Add a `Tests/Fixtures` `refs-show` fixture with a two-piece group (comment on the
   follower) and tests covering decode, entry count, note chip, Copy payloads, and
   decoding without `pieces`.
6. Document it in `docs/refs-inspector.md` (mounted into Bob's site; run
   `just docs-check`).

## Rollout

Bryan's hosts track master. After the bob-cli phases land, the Mac's next scheduled
`bob ref scan` rewrites only the four notes that contain groups. athena never writes
annotation notes: annotation sync stays the Mac's full scan, per
`decisions:references-are-ingested-at-birth`. No manual migration step exists.

> [!decision] memory_decision_record The land agent adds
> `sase/memory/decisions/continuation-groups-tuck-pieces.md` (metadata
> `status: accepted`, summary "A `<` group renders one quote card with one head `^h-` ID
> and tucks each later highlight as its own nested quote with its own page, images, and
> comment; pieces are never fused"). It follows the web's record shape: Applies to
> (bob-cli, Bob Mac Capture, vault), Claim, Why (the rejected alternatives above), Cost
> (nested callouts; CSS needed for the stitched look), Reopens when (Obsidian gains
> reliable nested block IDs), and Evidence (this plan and the phase commits). Then it
> runs `sase memory init`.
