---
tier: tale
title: Finish paired return links and land bob-cli-5j
goal:
  Correct rendered link pairing and atomic highlight cleanup, verify the remaining
  acceptance cases, and close bob-cli-5j normally in the coding turn.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5j.land
bead: bob-cli-5j
create_time: 2026-10-07 16:32:32
status: wip
---

- **PARENT:**
  [202610/ref_create_return_links.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_return_links.md)
- **BEAD:**
  [bob-cli-5j](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5j/README.md)

# Finish paired return links and land bob-cli-5j

Implement only the remaining defects and acceptance gaps below, then finish the landing
of epic bob-cli-5j in this same coding turn. This is bounded work for one agent. Do not
wait for this tale's commit, a push, its SHA, or CI before closing: the host commits the
coder's work after its turn ends. A tale has no land agent that would resume this
landing.

## Context and completed audit

Read these recorded artifacts for the approved contract and reproducible evidence:

- `plan:202610/ref_create_return_links.md`, the original epic plan.
- `file:explicit:6820361e338367179bec63c4`, the lander's audit and reproductions.

Use `sase artifact read` for those reads. The lander reviewed both closed phase beads,
every note, epic commits d4fab34 and 6fb936d, and actual code/tests on fetched master
bb66952. Rendering, reporting, precise marker stamping and sync flag threading are
implemented; existing gates passed (23 return_links lib tests, 10 create lib tests
including both XeLaTeX tests, 65 create CLI tests, exact completion kinds coverage, and
the 3 cleanup/sidecar tests). The gaps below are epic defects missed by those tests.

The lander also reviewed non-epic drift since the epic began: a5bb9ae's audio completion
and Pandoc ampersand fixes, c9b17c9's thread-local env facility, d08df0e's install hint
and bb66952's foreign plugin-checkout guard. All remain intact; the typed compose_marker
callers preserve the existing PDF/web ingest routes. Refresh the checkout through the
normal SASE integration workflow and review any additional landed drift before closeout.
Preserve `bob_env` reads and both supported Pandoc ampersand forms. No feature
integration edit was otherwise needed on bb66952.

Follow-up triage is already complete and recorded on bob-cli-5j. Carry these outcomes
into its final close note; do not create duplicates:

- bob-cli-5j.1 note #1 matches bob-cli-4u; #2 matches bob-cli-4j. Both tasks are closed,
  fixed by a5bb9ae, and their tests now pass. Supplementary resolution evidence was
  added; fresh tasks/reopens were declined as already resolved.
- bob-cli-5j.1 #3 and bob-cli-5j.2 #1 are one DISPLAY/xclip failure. The exact
  capture::r#ref::capture_url_with_markers_or_flags_stays_a_task test still failed on
  bb66952. Active bob-cli-5k.3 explicitly owns clipboard isolation and the canonical
  check gate; a DISCOVERED ISSUE note was added to bob-cli-5k. Its reported 3cbef27 was
  absent from fetched master, as was the `check` recipe. This infrastructure remediation
  stays with that epic; no new task was filed.
- Conditional Mac/iPad device checks in the original plan have no observed failure and
  are not grounds for speculative tasks.

## 1. Make return-link indexing and decoration respect rendered contexts

Work in `src/native/highlights_ref/return_links.lua` and its existing filter tests in
`return_links.rs`. Keep the original eligibility matrix, alias order, tag
alphabet/order, prefix collision handling, opt-out and report contract. Maintain exactly
one rendered return pill per rendered source anchor and tag. Keep the second filter
after the code-break/listen filter; listen cards and document metadata stay untouched.

Fix the capability scan's incomplete recursion. Its separate `scan_skip_spans` does not
descend into DefinitionList definitions, table-body cell blocks or inline
containers/Notes, and misses inline Image descriptions. Either share a context-aware
walker or use equivalent complete traversal so id counting, capability classification
and rendering decisions agree. Spans inside headings, captions/image descriptions and
table head/foot rows must remain non-capable at every nesting depth. Preserve eligible
normal prose, definition lists, table body cells, figures and footnotes; do not solve
this by disabling an entire supported container. Do not introduce pills into heading
text, TOC entries or bookmarks.

Add regressions using both filters and assert on LaTeX **and** report counts:

```markdown
[go](#inner)

Term : # [caption]{#inner}
```

Currently this gives paired=targets=1 and inserts BobBackInline inside the nested Header
and its optional section argument. It must instead keep the canonical untagged forward
link, report paired=targets=0 and untagged=1, and insert no pill macro into the heading.
Also cover a nested header in another supported container and table head/foot skip
contexts; native/JSON Pandoc fixtures are appropriate for AST forms that Markdown cannot
express conveniently.

```markdown
[go](#inner)

Text ![[caption]{#inner}](image.png) inline.
```

Currently this reports paired=targets=1 while LaTeX drops the image description, so the
tagged source has no visible return pill. This Span must be non-capable; the link
remains untagged, report paired=targets=0, untagged=1, and no generated source
anchor/tag or pill appears. Preserve Pandoc's existing forward-link behavior; adding new
return target kinds is out of scope.

Fix Cite traversal too. The current branch decorates `citation.prefix`/`suffix` metadata
but ignores rendered `Cite.content`. With Bob's actual pipeline (no citeproc), this
input produces a pill referencing bob:ret:1 but no rendered BobReturnAnchor or BobTag:

```markdown
# Target

[See [the target](#target) @smith].
```

Pandoc stores a Link in citation.prefix and literal fallback Strs in Cite.content. Count
and decorate rendered content, rather than invisible citation metadata; avoid counting
one occurrence twice. For this fixture the literal fallback stays unchanged,
paired=targets=0 and no source/pill macros are emitted. Add a constructed Cite whose
visible content contains a real Link to verify that visible eligible links still get one
tag, anchor and pill. Exercise the one-to-one invariant on the new cases, including
consistency with report.paired and actual rendered macros. Keep dead-link
formatting/warnings and untouched external links correct.

## 2. Consume highlight modifier runs atomically

Fix `strip_tag_runs` in `src/native/highlights_ref/text.rs`. It currently processes the
first glyph separately when whitespace precedes it, then strips the rest of the same run
because their preceding character is non-whitespace. The strip rule applies to a
**maximal run**: consume the entire run once, and decide from the character before its
start whether to keep or remove it.

Regression examples:

- `tag ᵃᵈ alone` stays `tag ᵃᵈ alone` (current output incorrectly loses the ᵈ).
- A longer run after space/newline/tab likewise survives whole.
- `see rankingᵃᵈ.` becomes `see ranking.`.
- `ᵃᵈ, row 12` becomes `, row 12`.
- `Text ↩ p. 2ᵈᵃ continues` becomes `Text continues`.

Preserve the existing pill-fragment cleanup order and every original shape test. Extend
sidecar coverage with a spaced multi-glyph run; stripping remains limited to flagged
Highlight text. Comments/standalone notes and unflagged phonetic text remain unchanged.
Block IDs, task sources and [h:: ...] markers continue to derive from raw sidecar text
and stay identical with stripping enabled/disabled.

## 3. Finish the existing acceptance checks and docs

In `create.rs`'s `return_link_xelatex_render_pairs_pills_with_destinations`, actually
link to `Near bottom landing`. The current fixture has no inbound link to that Header,
so its layout assertion never tests a pill-bearing heading/needspace guard. Update
expected counts after adding the link and tune deterministic filler as needed. On the
**stamped** PDF, verify all source destinations and return GoTo actions resolve, every
pill's page/tag agrees with its source, heading text/TOC/ bookmarks stay clean, and the
near-bottom heading, its return row and first table cell are on the same page.
Render/rasterize the updated fixture at 110 dpi and inspect the relevant pages,
including wrapped rows and inline Span pills.

In `docs/highlights-create.md`, state that dry runs preview the planned marker without
`return_links`; the flag is added only after an actual render pairs a link. Keep the
existing eligibility/opt-out/marker documentation aligned with the fixes. No new CLI
options, target kinds, narration design or speculative device fixes.

## 4. Verify the final tree

Run the targeted filter/cleanup/sidecar tests and the create library/CLI tests above
with Pandoc and XeLaTeX available. Add only meaningful regressions for the observed
defects and acceptance gap. Run `cargo fmt`, then `just check` for file-change
verification. Never run `just check-full`.

If the check-gate changes from bob-cli-5k.3 have landed by then, integrate them and use
their canonical gate. If `just check` still lacks its recipe, record that known
infrastructure gap and run the existing native equivalents (`cargo fmt --check`,
`cargo clippy --all-targets --all-features`, and `cargo test --no-fail-fast`) so every
binary is exercised. The DISPLAY/xclip defect stays with bob-cli-5k if still present:
distinguish it from any failures caused by these fixes and record the exact result
rather than presenting a red gate as green. Follow the required SASE long-command
workflow when necessary. Do not expand this tale to fix unrelated test infrastructure
and do not use missing infrastructure to conceal an epic bug.

## 5. Finish bob-cli-5j's landing in this same turn

This is the final implementation step, after all epic fixes and verification:

1. Re-read
   `sase bead read bob-cli-5j -r "Recheck scope, children, notes and parent link before final closeout"`
   and its descendants. All named phases must be completed. Check the original linked
   plan's readiness against the completed fixes and any new landed drift. Include the
   audit, fixed defects, verification outcomes and **each follow-up triage outcome
   above** in the close note.
2. Run `sase bead epic-symbols bob-cli-5j`. For every listed --epic-symbol entry,
   resolve it by wiring it up, privatizing it, using a justified non-test pragma, or
   deleting it per policy. Re-key a Justfile exemption only if a concrete still-open
   later bead still needs it. There were no entries at audit time; check again and leave
   none keyed to this epic or its closing phases.
3. Close normally with
   `sase bead close bob-cli-5j --note "<verified scope, post-start integration, fixes, final checks and all follow-up outcomes>"`.
   If leftover epic-symbol entries reject the close, resolve/re-key them and retry. If
   an unfinished named phase rejects it, finish/reopen that work; never force merely to
   make it succeed and never force a successful nested landing. A deliberate
   canceled/superseded outcome alone may use --force with an explanatory --reason and
   that resolution; this plan intends normal done.
4. After the close, run `just symvision` when the recipe is available and confirm the
   whitelist is clean. If unavailable (as at audit time), record that fact.
5. Set `status: done` in the frontmatter of the epic's linked plan,
   `plan:202610/ref_create_return_links.md`. Resolve its current PLAN path from the
   bead/artifact; do not assume a numbered workspace path. Before editing its sidecar
   repository, use `/sase_repo`
   (`sase repo open plans -r "Mark bob-cli-5j's successfully landed plan done"`) and
   honor that repository's instructions. Include its change in the root `/sase_final`
   declaration.
6. Inspect the current `parent_bead` of bob-cli-5j after it closes. The lander's audited
   read showed **no parent**, so finish normally if still absent. If one has been
   deliberately added, follow the original land prompt's parent rules: a phase parent is
   closed only after verifying this child satisfies it, leaving its containing epic to
   its own lander; a plan parent needs its previous landing note, every descendant/note,
   linked-plan readiness, post-child drift and epic-symbol cleanup rechecked before
   normal close, symvision and plan status done. Repeat only through fully complete
   directly parented plan ancestors; record a blocker and stop at the first
   incomplete/ambiguous parent.

Do not wait for this tale's final commit to perform any of the above. Declare all
repositories this turn changed through `/sase_final` as the last normal-turn action.
