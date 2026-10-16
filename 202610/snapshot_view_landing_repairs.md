---
tier: tale
title: Repair snapshot view landing gaps
goal:
  Snapshot projection and the shared print contract satisfy the approved design, and
  bob-cli-87.2.1 and phase bob-cli-87.2 close normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-87.2.1.land
bead: bob-cli-87.2.1
create_time: 2026-10-16 10:53:42
status: wip
---

- **PARENT:**
  [202610/snapshot_view_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/snapshot_view_core.md)
- **BEAD:**
  [bob-cli-87.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.2.1.md)

# Repair the snapshot view contract and finish its landing

## Outcome and ownership

Finish the remaining work in **bob-cli-87.2.1**, then close that epic and its enclosing
**phase bob-cli-87.2** in this same coding turn. This is one bounded implementation task
in the snapshot parser/model/serializer and their tests. Do not start another plan or
wait for another land agent. Do not close the containing epic **bob-cli-87**; its land
agent owns the other outer phases.

The approved child design is `plan:202610/snapshot_view_core.md`; the outer design is
`plan:202610/note_snapshots.md`. Read them through `sase artifact read`. The child
epic's linked plan file is **202610/snapshot_view_core.md in the plans repository**. Use
`/sase_repo` and `sase repo open plans` before modifying it; use the returned checkout
path, not a hard-coded workspace path.

Inherited decisions remain `default_text_layer = yes`, `obsidian_command = yes`, and
`decision_record = no`. Do not edit any memory. Keep Rust-owned note semantics, the
engine-neutral view, CommonMark plus attributes/YAML serialization, and the existing
pandoc/xelatex profile. Keep the approved one-level embed limits, synthetic fixtures,
cached reads, and read-only vault behavior. CLI wiring, default-parent selection,
Bob-specific filtering/chips, font policy, image conversion, and plugin work remain
owned by the outer epic's other phases.

## Completed audit; do not repeat implementation already present

The lander reviewed all four child scopes and every note, the approved plans, current
source, and epic commits **426b07c, 29f5f4e, 8da6615, a086cd9**. All four child phases
are closed. Classification, cached reads, URI extraction, most syntax, bounded
transclusions, and the existing golden harness are implemented. On a086cd9 the existing
**81 snapshot tests and two audio URI tests pass**. Six additional audit probes fail,
exposing the cases below; the temporary probes were removed after recording the
evidence. Reintroduce their assertions as durable regressions while fixing the behavior,
rather than merely accepting changed output. Read
**file:explicit:015da024c10eb8fd59ecb5fc** with `sase artifact read` for the exact
six-test patch, failure output, passing-suite output, and Pandoc AST evidence.

Non-child commits after the first child commit were **1a57e47** (outer print profile),
**c1b1cef/31528a1** (saved capture templates), and **298ebdf** (task cleanup
documentation). The profile needs the contract integration below. The template and
cleanup changes have no snapshot feature integration to implement. This checkout was
master at a086cd9, matching origin/master, with no separate PR base. At the start of
coding, check any newer commits for relevant drift.

The only child `PROPOSED FOLLOW-UP:` is **bob-cli-87.2.1.4 note #1**, containing two
metadata coverage failures. The lander completed `/sase_new_task` triage, including
duplicate searches, the last week's all-status tasks, and active epics. Both failures
were independently reproduced on clean a086cd9 and traced to **31528a1 / bob-cli-8b.2**:

- `native::completion::kinds::tests::every_value_arg_has_a_decision` lacks decisions for
  `capture-templates delete:template`, `save:template-origin`, and `show:template`.
- `runner::tests::every_native_command_is_on_exactly_one_canonical_path` counts 36
  commands against the stale expected 31.

They are recorded as a **DISCOVERED ISSUE on active epic bob-cli-8b**, and the outcome
is recorded on bob-cli-87.2.1. Evidence: `file:explicit:28b089609636028ae170391f`. No
standalone tasks were created. Closed bob-cli-4j concerned the old audio argument, so it
was correctly declined as a duplicate match; it was neither reopened nor corroborated.
Neither proposal was declined as invalid. Carry these outcomes into the final close
note. Do not fix the template epic's failures as part of snapshot work or leave snapshot
work open solely because these same baseline failures persist.

## Remaining repairs

### 1. Ensure the block parser always advances

In `obsidian.rs`, a whole-line `![remote](https://example.test/image.png)` satisfies
`is_single_embed_line`, but `parse_markdown_image_line` rejects a remote URL and
`parse_paragraph` immediately stops on the same predicate. It returns `next == index`,
so `parse_blocks` loops forever and keeps adding empty paragraphs.

Make dispatch and paragraph termination agree and guarantee cursor progress for
unhandled constructs. Preserve the remote image's readable label and URL through the
existing fallback policy; no network fetch is required. Test a standalone remote image,
adjacent prose, and malformed/unhandled image forms. A bounded test of the
dispatch/progress invariant must fail promptly if this recurs; avoid a regression test
that can hang the whole suite.

In the same dispatch repair, preserve the approved behavior for explicit embeds in prose
and task/list content: `project_inline_embed` currently turns a first-level `![[Other]]`
into an ordinary link, and `project_inline_image` turns an image into an asset link
(also losing width hints). The approved content policy has no standalone-line
restriction. Retain surrounding words and their order while carrying these embeds
through typed nodes into the same image/transclusion paths used for standalone embeds.
Add synthetic inline image/width and inline note-embed cases, including nested/cyclic
fallback, and verify actual Image/Div output. Reuse the existing expansion guards; do
not introduce recursive inclusion or remote fetching.

### 2. Emit and preserve the actual anchor set

`parse_paragraph`, `parse_list`, and the block-ID branch of `parse_heading` only push
anchors into a side vector; their model nodes do not retain those anchors.
`transclusion.rs::collect_block_anchors` consequently drops them, and serialization has
no way to emit them. For example:

```markdown
# Head ^heading

Paragraph ^paragraph

- item ^list

[[#^heading]] [[#^paragraph]] [[#^list]]
```

Currently the assembled view has only the `head` anchor and all three links lose their
targets. Extend the typed representation with the smallest coherent anchor carrier, emit
the approved `t-ID` anchors at the actual content, and carry them through collection,
filtering, and repeated-embed namespacing. Do not just put non-emitted anchors back into
the visible set. Test host prose, ordinary list items, headings with a block ID, and
these forms in repeated/selected transclusions.

Also reserve a collision-safe synthetic prelude heading:
`Prelude\n\n# Snapshot content\nBody\n` currently emits two `snapshot-content` IDs.
Assert uniqueness over the actual serialized/Pandoc anchor set, including host and
embedded content.

### 3. Resolve fragments using the same protected source semantics

`resolve.rs::comment_span_lines` sees `%%`/HTML markers inside code and incorrectly
hides later headings. This input currently makes `Visible` missing:

````markdown
```text
%%
```

# Visible

Body
````

Align fragment indexing with the syntax projector's code/comment protection. Test both
percent and HTML markers in fenced and inline code, real comments, and headings whose
visible text contains an inline comment. Preserve code bytes and meaningful visible text
around multiline comments; use a shared scanner/helper where practical instead of adding
another divergent comment parser.

Explicit self-links also bypass ambiguity rejection. In a note with two `# Same`
headings, `[[Host#Same]]` warns but still emits a link to the first heading; `[[#Same]]`
correctly becomes plain text. Require successful unique lookup before creating an
internal target in either spelling. Cover duplicate heading and block fragments,
aliases, host links, and embedded selections. A warning must never accompany an invented
first-match anchor.

### 4. Preserve the raw task record promised to the Bob layer

`parse_task` builds `ParsedTaskLine.body` from comment-stripped `WorkLine.text`. For
`- [ ] Keep %%private%% [fresh:: 2026-10-16] ^item`, the stored body incorrectly becomes
`Keep  [fresh:: 2026-10-16] ^item`. Derive the lossless body, indentation, and source
positions from the original source line/bytes while projecting only the visible text.
Verify comments, unknown fields, tabs, and tasks inside callouts retain honest
original-source records. Keep root SHA/source-cache consistency intact.

### 5. Repair serializer semantics, then integrate the print contract

`serialize.rs::Block::LabeledRow` uses `render_inlines` for its label, so the canonical
contract currently contains `- Waits on ...` instead of the required
`- [Waits on]{.label} ...`. Emit the `.label` span and verify the renderer maps it to
its label macro, including nested rows and links.

Splitting strike content around links currently puts whitespace at delimiter edges.
Pandoc parses the existing canonical fixture's strike paragraph into literal `**` and
`~~` text, loses the bold formatting, and leaves surrounding words unstruck. Fix
serialization of split strike/emphasis runs so meaningful non-link text retains its
intended formatting and links stay outside every strike node. Include visible links,
links whose target was filtered out, nested emphasis/strong, link chips, and
leading/trailing whitespace. Assert Pandoc AST semantics and absence of unintended
delimiter text, not merely absence of a Link nested inside Strikeout.

The parallel print-profile commit still reads
`tests/fixtures/note_snapshot/profile-contract.projected.md`, while the serializer tests
use `contract.projected.md`. Reconcile the two into **one canonical complete contract
fixture consumed by both suites**, preserving the useful examples and renderer
assertions from both. Keep generated goldens distinct from that hand-authored contract.
Ensure at least one test sends actual serializer output through the real profile
filters, with pandoc/xelatex availability handling matching existing tests. Preserve
existing font/glyph, return-link, clean-text-layer, and navigation coverage; do not
weaken those tests to make the fixture merge pass.

## Verification

Use invented temporary vaults only. Make the repaired cases meaningful regressions;
retain the read-once/mutation consistency and byte-identical-vault checks, exact 64
KiB/1,500-line boundaries, cycle/nesting fallbacks, residue assertions, and URI
compatibility tests. Run the focused snapshot suite and audio URI tests, then
**`just check`**. Never run `just check-full`.

Any new failures introduced by these repairs remain this tale's work. If only the two
documented template-metadata failures persist, compare them with the recorded clean-base
evidence and record the unchanged failures explicitly; they do not block snapshot
completion. Investigate and triage any genuinely new unrelated failure via
`/sase_new_task`. Use `/sase_monitor` when necessary for long verification commands,
with a continuation that still performs the closeout below; a successful check alone
must not end the work before closeout. Do not wait for this turn's commit SHA, push, or
CI result: the host commits only after the coding turn ends.

## Final step: complete this landing in the coding turn

1. Re-read `bob-cli-87.2.1` and its four child notes, confirm the repairs above and any
   relevant post-audit drift are complete, and include the existing follow-up triage
   outcomes in the close note. Read the parent phase `bob-cli-87.2` and confirm this
   child plan fulfills its view-core scope.
2. Run `sase bead epic-symbols bob-cli-87.2.1`. Resolve every listed symbol, or re-key
   it only to a still-open later bead that genuinely needs the exemption, following
   Symvision policy. The lander's audit found none, but check again after coding.
3. Close normally with
   `sase bead close bob-cli-87.2.1 --note "<specific parser, anchor, raw-source, serializer/profile integration, focused tests, just check result, intervening-commit review, and follow-up triage verification>"`.
   If rejected, fix the reported unfinished work/symbols and retry normally. Never force
   merely to make it succeed and never force to advance this successful nested landing.
   Deliberate cancellation/supersession would require its own justified
   `--force --reason ... --resolution canceled|superseded`, not a success workaround.
4. Run `just symvision` if available and confirm a clean whitelist; if the recipe
   remains absent (as at a086cd9), record that fact and the empty epic-symbol check.
   Open the plans repository via `/sase_repo`, then set **`status: done` in the
   frontmatter of `202610/snapshot_view_core.md`**. Do not mark the outer
   `202610/note_snapshots.md` plan done.
5. Re-read `sase bead read bob-cli-87.2.1 -r "Need the parent link"`. Its direct parent
   is the **phase bob-cli-87.2**. Confirm its requirements are now satisfied, run
   `sase bead epic-symbols bob-cli-87.2`, resolve/re-key all entries as above, then
   close **only that phase** normally with
   `sase bead close bob-cli-87.2 --note "<projection, source fidelity, URI, shared contract, transclusion, regression and gate verification; independently reproduced unrelated baseline failures if any>"`.
   Leave **bob-cli-87** to its already-waiting land agent. If unexpected parent work is
   genuinely incomplete, record the concrete blocker on bob-cli-87.2 and report it; do
   not force its close.
6. Finish with `/sase_final` so the host commits all changed repositories, including the
   linked plan status update, after this turn. Describe actual completeness in the
   declaration; commit visibility is not a prerequisite for closeout.
