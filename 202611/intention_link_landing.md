---
tier: tale
size: medium
title: Finish intention link repair and close its landing
goal:
  Preserve YAML, link targets, and draft ownership through every rewrite, then close
  bob-cli-nh.4.1 and phase bob-cli-nh.4.
proposed_by: bbugyi200.athena.bob-cli-nh.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-nh.4.1
create_time: 2026-11-19 17:39:02
status: wip
---

- **PARENT:**
  [202611/intention_link_repair.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/intention_link_repair.md)
- **BEAD:**
  [bob-cli-nh.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/bob-cli-nh.4.1.md)

# Finish intention link repair and close its landing

Implement the four bounded gaps below, verify the integrated feature, then close
`bob-cli-nh.4.1` and its parent **phase** `bob-cli-nh.4` in this coding turn. This tale
is the continuation of the landing; there is no later land agent for the tale. Do not
wait for this work's future commit, push, SHA, or CI to perform closeout.

## Context already verified

The lander read the epic, all four child scopes and all eight child notes, the linked
`plan:202611/intention_link_repair.md`, the enclosing
`plan:202611/period_intentions.md`, and the parent phase's final decisions. Preserve
`day_summary_cap = sixty`. This work needs no memory changes, plugin changes,
installation, or live-vault writes. Keep all fixtures temporary.

The main implementation exists: `749efc28` implements typed owned-field repair, archive
integration, and digest carry; `ea538f78` covers reference rename and overwrite;
`992fd33d` supplies findings and documentation. Task phase `bob-cli-nh.4.1.2` committed
`0f990b1d7fc39dec993aa7e921a72d78121dea59` on detached HEAD. It was absent from both
master and origin/master when this landing started. The lander recovered it from the
current checkout's reflog and applied it with `git cherry-pick --no-commit`, merging the
later reference adapter additions cleanly. This landing's finalizer preserves those 12
source/test files before the tale starts. Verify their presence; do not reimplement that
phase or blindly apply the detached commit again. Formatting and diff whitespace checks
passed; the combined tree's Rust and docs gates have not yet run in this landing.

The four gaps are established by source review, not yet by newly executed failing tests.
Start by pinning them with focused regressions. The lander's note on `bob-cli-nh.4.1`
records the complete audit and follow-up routing.

## Remaining implementation

1. **Protect unclosed canonical YAML in reference rewrites.** In
   `src/native/highlights_ref/rename/links.rs`, `canonical_frontmatter_end` returns
   `None` when an opening `---` has no closing delimiter. `rewrite_text` consequently
   treats all property lines as body text, and `rewrite_canonical_owned_fields` returns
   that raw result. A canonical daily note containing
   `---\ncustom: "[[ref/blogs/old]]"\nintentions: "[[ref/blogs/old]]"\n` is therefore
   rewritten despite the malformed-frontmatter preservation contract. Match the
   conservative protection already used by
   `collect_done/link_repair.rs::frontmatter_end_for_repair`: an unclosed opener
   protects through EOF. Keep the distinction between absent and unclosed YAML;
   body-only notes still repair. Closed but invalid YAML keeps its block exact while
   body repair proceeds. Cover `plan_vault_links` and `rewrite_contents`, BOM/CRLF,
   unrelated properties, and valid typed repairs. Preserve noncanonical reference
   metadata behavior and the All/HighlightFragments/known-highlight guards.

2. **Carry provenance even when only the body changed.** In
   `task_move/transplant.rs::repair_owned_postimages`, the early continue when
   `owned.contents == current` bypasses `carry_rewrite_digests`. A drafted weekly
   bystander with an unchanged summary and a link only in `## Review` is rewritten by
   `repair_vault_postimages`, but its Review digest stays stale. Both move and split
   share this omission. Carry digests from the captured original into every changed
   relevant final postimage, whether or not an owned value changed. Preserve accurate
   link counts (a stamp is not another link), no-op identity, and refusal propagation.
   Test a Review-only legacy draft, a split-summary/Review period where only Review
   changes, and both parts changing. Edited or reflected notes must never acquire
   refreshed ownership. Verify finalization also covers source and destination changes
   after insertion, and derive eligibility from original bytes, never a partially
   restamped intermediate. Keep reported landing/cursor locations aligned with the final
   planned bytes.

3. **Repair complete split payloads on every cross-note extraction.** In
   `task_split/plan.rs`, payload repair currently requires `!moved_targets.is_empty()`.
   A bullet without a prior anchor receives a minted anchor, so an otherwise empty
   relocation map bypasses qualification of links to stayed source anchors. For example,
   select an unanchored child whose descendant links to `[[#^stay]]` while `^stay`
   remains in the source; the inbox payload must link back to the source even though no
   old identity moved. Keep fabricated retargets out of the relocation map, but do not
   gate this qualification on the map having entries. Also, `generated` contains the
   task line followed by individual child lines; repairing each entry separately loses
   multiline fence and comment context. Pass complete logical payload blocks through the
   existing fragment resolver, preserving ordering and source metadata. Test minted and
   retained anchors, descendant anchors, a pathless stayed-anchor link, fenced and
   commented examples, same-note extraction, counts, and final task/cursor lines. Keep
   the existing 31 split conformance vectors and merge source-anchor retention behavior.
   Use the existing single transaction and capture dispatch.

4. **Integrate the newer drafting writer with strict delimiters.** Since the first
   repair commit, `a29a64f6` added `intentions/write.rs::apply_with_review`. Its
   post-edit delimiter rescan uses `line.text.trim() == "---"`, duplicating the old
   behavior that `749efc28` corrected in `apply` and `split_document`. An indented `---`
   inside a multiline reflection scalar can be mistaken for the YAML close when new
   summary/Review stamps are appended. Share the strict delimiter rule across both
   writer entry points; factor a small common helper if useful rather than broadening
   this into a serializer redesign. Add a regression with a valid reflection block
   containing an indented `---`, absent agent stamps, and a first summary/Review draft.
   Assert successful verified YAML, exact unrelated values/body preservation, and
   correct resulting stamps. Retain the accepted first-draft-after-reflection behavior,
   day cap, and guarded update fingerprint.

## Integration and acceptance

Refresh the local source history and recheck commits landed after the lander's review at
`992fd33d`. Previously reviewed intervening bank commits `5b547f13` and `c94a718d` do
not retarget note links and need no integration. The summary writer interaction above
does. Reconcile any newly landed migration work with the shared typed reader/writer and
report-only legacy detection; do not redo sibling phases.

Confirm caller coverage: archive moved-block and archive-file rename; direct, capture,
restore/Gmail move through the shared planner; merge's copy-and-retire source identities
(its real relocation dispatch goes through move); cross-note split; reference rename and
overwrite identity/highlight/frozen-document paths. Generic `vault_rewrite::run` callers
are theme rename/split/migrate, relation-label rename/reverse, and periods
ensure/migrate. Preserve their vocabulary/scaffold semantics; check that provenance
finalization cannot make a displayed proposal and its applied bytes disagree. Do not
invent target retargeting where none exists.

Run focused regression suites for intentions, periods note/draft, archive, move, merge,
split, and reference rename/overwrite, then `just check` and `just docs-check`. Do not
run `just check-full`. Use the SASE monitor skill if a gate requires a handoff; its
continuation must still finish this tale's closeout. If changing docs or help becomes
necessary, read docs memory first and regenerate required outputs.

Exercise one isolated temporary fixture through archive preview and apply, then
`bob periods list -p`: include a moved block referenced from a weekly intention, a daily
reflection, and a drafted summary. Assert valid YAML, unchanged unrelated keys, the
expected planned/applied bytes, no dangling finding for repaired links, and retained
drafted state. Check that findings leave fixture bytes unchanged.

The known failures were triaged before proposing this tale:

- `bob-cli-nh.4.1.2` note #1, `.3` note #1, and `.4` note #2 report the same seven
  clean-base test failures. Four capture completion expectations were attached as a
  `DISCOVERED ISSUE` to active `bob-cli-nj` because `nj.6`/`ddf28f64` introduced the
  WorkLog context. Three periods-update failures were attached to active `bob-cli-nh`:
  missing catalog and completion decisions plus the ordered-ops test incorrectly
  expecting final `one` instead of `middle`. They belong to `nh.2`/`a0b91a8d`, outside
  this child epic. No new task duplicates were created.
- `.4` note #1 duplicates ready task `bob-cli-jz`; the lander added +1 with fresh
  canonical counts on Mac `b6b7f6cf`: capture-previews 6171 and keyboard 6027
  against 6000. An additional issue on active `bob-cli-nj` records its `nj.7` growth of
  capture-previews from 6096 to 6171. No proposal was discarded; redundant new tasks
  were declined because the existing tracker and active causal owners cover them.

Compare any gate failures to these exact known cases; do not claim a failing gate
passed, blanket-waive new failures, or let an unrelated clean-base failure prevent the
approved child closeout indefinitely. Epic-caused failures remain this tale's work.
Record actual gate commands/results and caller coverage on `bob-cli-nh.4`.

## Final step: complete the landing in this coding turn

After implementation and verification, re-read `bob-cli-nh.4.1` and every child with
`sase bead read ... -r "Recheck intention link landing readiness"`, including any new
notes or descendants. Confirm all scoped work and decisions are honored. Include all
follow-up dispositions above in the close note.

1. Run `sase bead epic-symbols bob-cli-nh.4.1`. Resolve every listed entry by using,
   privatizing, applying an appropriate non-test pragma to, or removing its symbol;
   re-key only when a specific still-open later bead actually needs the exemption. The
   prior audit found none, but recheck. Then close normally:
   `sase bead close bob-cli-nh.4.1 --note "<source/commit audit, fixes, drift review, exact verification and follow-up outcomes>"`.
   A refusal is work to resolve, not permission to force success. Never use `--force`
   merely to make a close succeed or to advance a successful nested landing.
2. Run `just symvision` if available; if the recipe is still absent, record that fact
   and the clean epic-symbol inventory. Through `/sase_repo`, open the plans repository
   before editing. Re-read the linked plan with
   `sase artifact read plan:202611/intention_link_repair.md "Complete the verified epic plan"`
   and set `status: done` in its frontmatter at the PLAN path returned by the bead. Keep
   other frontmatter intact; include this plans-repository edit in the final
   declaration.
3. Run `sase bead read bob-cli-nh.4.1 -r "Need the parent link"`. Its parent is the
   **phase** `bob-cli-nh.4`, not a plan. Re-read that phase, verify this child fulfills
   its owned-key link-repair and findings requirements, and run
   `sase bead epic-symbols bob-cli-nh.4`. Resolve/re-key any entries before closing only
   that phase normally with
   `sase bead close bob-cli-nh.4 --note "<verified caller coverage, integrated fixes, acceptance and gate results>"`.
   Leave containing epic `bob-cli-nh` and `period_intentions.md` to its waiting land
   agent. Record a specific blocker instead of closing if parent-phase readiness has
   changed.
4. Finish through `/sase_final` with the primary source and any changed plans repository
   committed. Closeout must precede this final commit declaration; no step depends on
   this work's commit SHA, publication, or post-commit CI.
