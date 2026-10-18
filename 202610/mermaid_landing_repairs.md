---
tier: tale
title: Fix Mermaid fence detection and warning positions, then land bob-cli-8y
goal:
  All explicit Mermaid blocks are considered, warnings identify their real positions,
  and bob-cli-8y closes with verified readiness.
size: medium
proposed_by: bbugyi200.athena.bob-cli-8y.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-8y
create_time: 2026-10-17 22:36:21
status: wip
---

- **PARENT:**
  [202610/ref_create_mermaid_figures.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_mermaid_figures.md)
- **BEAD:**
  [bob-cli-8y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8y/README.md)

# Finish Mermaid fence detection and diagnostic numbering, then land bob-cli-8y

## Scope and inherited choices

Finish the two demonstrated defects below and close epic **bob-cli-8y** in this same
coding turn. One agent can implement the bounded changes directly; there is no
subsequent land agent for this tale. Do not wait for this turn's commit SHA, push, or CI
to close the epic. The host commits after the turn ends.

Read `bead:bob-cli-8y` and `plan:202610/ref_create_mermaid_figures.md` through their
audited commands. The original epic's choices remain final:

- `mac_check=manual`: preserve the checklist; do not contact the Mac.
- `decision_record=no`: no memory edits. The optional decision record is a separate
  task, bob-cli-90, requiring its own permission.
- Preserve the pinned offline Mermaid 12.2.0 adapter, static TrueType font, per-block
  visible fallback, one batch/browser per document, both Markdown routes, no new CLI
  flag, kill switch, no backfill, and unchanged non-Markdown routes.
- Use scratch vaults and the workspace build, never `just install` or the real vault.

## Landing audit already completed

At `99b2ac4` the lander read the epic, all four child scopes and every note, the
approved plan, implementation sources, and commits `54670b4`, `ad036f9`, `58c9614`, and
`99b2ac4`. All four children are closed. Phase 8y.2's temporary `diagrams: None` hook
was enabled by 8y.3; the ETXTBSY helper repair is present. Phase 8y.4 recorded the real
12-fence Athena run: 12 rendered, +4.70 seconds, no Type 3 fonts or raster image rows,
clean body-text checks, and scale warnings at 46% and 49%. The manual Mac checklist
satisfies the selected branch.

Integration review covered every non-epic commit since epic creation: `860f0be` and
`724333f` repair snapshot projection, file resolution, profile, and corpus validation;
the projection precedes Mermaid collection and the first-position swap preserves the
final profile's centering/framing. `3ec4d0d` and `30eafe2` concern dependency
documentation. None needs a product integration edit. A fresh origin fetch matched HEAD
at `99b2ac4`; recheck drift before closeout.

There were no `--epic-symbol` entries and no `symvision` recipe. The epic's `parent_id`
was null. Recheck both facts at closeout.

## 1. Keep actual document positions while deduplicating Mermaid blocks

`src/native/highlights_ref/diagrams.rs::parse_manifest` collapses blocks to
`CollectedDiagram { id, text, occurrences }`. `first_occurrence` then sums every
occurrence of earlier unique IDs. That loses the original order whenever a source
appears again after a different source.

Reproduced through the current workspace `bob ref create` with a scratch vault and stub
adapter: the document contains sources **A, B, A**; the adapter successfully renders A
and rejects B. It prints `rendered 2 of 3`, correctly, but warns
`Mermaid diagram 3 of 3 not rendered` although B is block **2**.

- Retain each unique block's true first document occurrence when parsing the ordered
  manifest (e.g. a one-based `first_occurrence` field populated from the manifest
  index). Keep duplicate counts for the rendered/total summary.
- Use that position for failure, missing-result, over-cap, and scale warnings; do not
  infer position from counts. Keep one adapter item and one applicable warning per
  unique source.
- Replace the misleading grouped-occurrence helper test with meaningful interleaved
  cases. Cover A/B/A and a longer interleaving, asserting true first positions and
  unchanged occurrence totals. Add/extend a stub CLI regression to prove B is reported
  as 2 of 3 and the adapter still receives only two items; cover scale-warning position
  as well, preferably in the same bounded tests.
- Preserve the existing 200-unique-item limit and reader/hash agreement.

## 2. Make the cheap fence pre-scan conservative for definition lists

`strip_quote_and_list_prefix` only accepts quote, bullet, and decimal-list prefixes.
Pandoc also recognizes Mermaid fences starting on a definition-list definition:

```markdown
# Definition diagram

Term : `mermaid     flowchart LR       A --> B     `
```

`pandoc -f markdown -t json` produces a `DefinitionList` containing a `CodeBlock` with
class `mermaid`. Yet current `bob ref create` reports no diagrams, prints source with no
note or warning, and never calls the adapter. Its dry run also omits the diagram count.
This was reproduced with the adapter set to a nonexistent path, which should have caused
visible fallback. The alternate Pandoc definition marker `~` also reproduces the AST gap
(e.g. `~   ~~~mermaid`).

- Extend the cheap scan to detect those valid prefixes, including combinations with
  quote/list nesting. A conservative over-match is allowed; the collect filter remains
  the authority and must match only an actual Mermaid CodeBlock. Do not mistake an
  ordinary tilde fence for a definition marker.
- Add pre-scan tests for both `:` and `~` definition markers, backtick/tilde Mermaid
  fences, and a representative nested form. Retain the existing text, untagged, ordinary
  Markdown, kill-switch, and no-Mermaid fast-path behavior.
- Add Pandoc-backed collection coverage and a generic-route CLI regression for the
  concrete definition fixture: rendering invokes the adapter and replaces source;
  unavailable rendering leaves a visible note and source with one warning; dry-run
  counts candidates without spawning pandoc, the adapter, or a browser. Use existing
  stubs and skip conventions. Do not change either route's reader flags to accommodate
  the test: the generic route uses Pandoc's inferred reader.

## 3. Verify and check for drift

- Run focused meaningful tests for the two regressions, then `just check` and
  `just check-web-clip-adapter`. The latter must include browser checks on Athena;
  confirm the existing live PDF test in `just check` actually ran, with its font,
  no-raster, malformed-fence, and long-TMPDIR assertions.
- Do not run `just check-full`. A separate infrastructure problem does not expand this
  epic. The current project has no named tool catalog; use the available prescribed
  check path. Follow `/sase_monitor` if a command needs a handoff, and resume all
  closeout steps after the result; never leave a process running when ending a turn. The
  recorded prepared-completion stage bug is bob-cli-91.
- Re-read every child if notes/statuses have changed, and inspect commits after
  `99b2ac4` (including origin/master) for integration needs. Finish any epic-caused
  issue before closeout. Honor every inherited decision above.
- Keep the patch bounded to the two demonstrated issues, their coverage, and any
  necessary integration uncovered by this recheck.

## 4. Finish this epic's closeout in the same turn

Follow-up triage is already complete and recorded on bob-cli-8y before proposal. Carry
these outcomes into the close note; do not create duplicates:

- Decision record (8y.1 #1, 8y.2 #1, 8y.3 #1, 8y.4 #8): **bob-cli-90**, small memory
  task, related to bob-cli-8p. Requires separate approval; no memory edited here.
- Prepared completion / CHECK_STAGES (8y.2 #3): **bob-cli-91**, large infrastructure
  bug. No named catalog and no stage markers were independently confirmed; the failure
  itself is attributed to the phase's recorded prepared-completion runs.
- Landscape figures and authoring nudges (8y.4 #3 and #6): **bob-cli-92**, one large
  feature task for the measured small-label problem, evaluating both proposals.
- SVG image embeds (8y.4 #4): **bob-cli-93**, large feature task. Existing SVG
  placeholders are deliberate and A10 excludes this capability; related to bob-cli-85
  for PDF-font context, not a duplicate of its web-article bug.
- Listen-skip marker (8y.4 #5): declined for now; no narrated-diagram failure or
  concrete requested interaction was demonstrated, and A10 excludes it.
- Browser-free mmdr/svg2pdf backend (8y.4 #7): declined for now; no new fidelity or
  selectable-text evidence reopens the explicitly rejected backend choice.
- Deployment via next `just install` (8y.4 #9): recorded as an operational reminder, not
  a separate software defect. The epic forbids replacing installed Bob; research hooks
  gain the feature with the next planned Bob deployment.

Then, in this exact lifecycle order:

1. Run `sase bead epic-symbols bob-cli-8y`. Resolve each entry keyed to the epic or any
   phase by wiring/privatizing/deleting it or using the applicable non-test pragma.
   Re-key only if a genuinely still-open later bead needs the exemption. Do not defer
   judgment or leave a stale exemption.
2. Close normally with `sase bead close bob-cli-8y --note "<verification>"`. Write the
   note from the actual results: all child notes and DECISIONS reviewed; commits and
   post-start drift integrated; interleaved warning numbering and definition-list fence
   recognition repaired; actual focused, just check, and adapter/live test results; all
   eight follow-up outcomes above; no outstanding symbols. Never force to make a close
   succeed or to advance successful nested landing. If rejected for entries or
   unfinished descendants, finish them and retry normally. Only deliberate
   cancellation/supersession can use force with the required reason and resolution.
3. Run `just symvision` if the recipe is available. At audit it was absent; record that
   fact if still absent rather than adding an unrelated recipe.
4. Open the plans repo through `/sase_repo` and set `status: done` in the YAML
   frontmatter of **202610/ref_create_mermaid_figures.md**, the PLAN file resolved by
   `sase bead read`. Preserve all other metadata and content. Use the path printed by
   `sase repo open plans`, never a numbered workspace path in this plan.
5. Run `sase bead read bob-cli-8y -r "Need the parent link"`. It has **no parent** at
   this audit, so no ancestor close is needed. If a parent was attached since then, stop
   and record that new relationship for deliberate review instead of blindly closing it.
6. Finalize with `/sase_final`, declaring any changed main and plans repositories.
   Report the completed epic and passing checks, and retain the short manual Mac
   reminder: recreate in scratch, select/highlight a diagram label in Highlights, copy
   body text, and confirm malformed source has a visible note. Do not claim these manual
   Mac actions were performed.
