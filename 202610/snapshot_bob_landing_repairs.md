---
tier: tale
title: Finish Bob snapshot projection and nested landing
goal:
  Repair the seven reproduced projection defects and close bob-cli-87.3.1 and parent
  phase bob-cli-87.3 with verified evidence.
size: medium
proposed_by: bbugyi200.athena.bob-cli-87.3.1.land
bead: bob-cli-87.3.1
status: done
---

- **PARENT:**
  [202610/snapshot_bob_layer.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/snapshot_bob_layer.md)
- **BEAD:**
  [bob-cli-87.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-87.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-87.3.1.land.md)
- **COMMITS:**
  - [145ecf8](https://github.com/bobs-org/bob-cli--plans/commit/145ecf832b575d7f9640e5a62a0001c5bbc70f18)
    — docs(plans): mark snapshot_bob_layer done after landing repairs

# Finish the Bob snapshot projection landing

Repair the seven reproduced composition defects below, then finish the landing of
**bob-cli-87.3.1** and its directly enclosing phase **bob-cli-87.3** in this same
implementation turn. This is one bounded change to the existing snapshot projection,
with regression tests. No additional phases or land agent are needed.

## Established scope and audit

Read `plan:202610/snapshot_bob_layer.md` through `sase artifact read` for the approved
contract. Its linked plan file is `202610/snapshot_bob_layer.md` in the plans repo (open
it with `/sase_repo`; use the returned checkout path). The containing plan is
`plan:202610/note_snapshots.md`. The parent is a **phase**, not another plan bead. Do
not close bob-cli-87 or mark `note_snapshots.md` done.

The lander reviewed all notes on bob-cli-87.3.1 and its three closed children, the three
implementation commits `57c3df2`, `87bc784`, `6c53e13`, and the actual Bob, visibility,
transclusion, source-attribution, serializer, and shared-project-helper code. Existing
snapshot tests pass: 128 tests, including serializer and print-profile checks on athena.
The 54 `every_` lib tests also pass. Seven additional synthetic regression probes fail
against `6c53e13`; their exact source and output are in
`file:explicit:d0d45785d14fd9cc8f6cbfc1` (read with `sase artifact read`). The probes
were temporary and removed, leaving production source unchanged.

Settled decisions remain `default_text_layer=yes`, `obsidian_command=yes`, and
`decision_record=no`. The first two belong to other outer phases. Do not edit any SASE
memory. Use invented fixtures only; do not read or mutate the live vault. Keep the
engine-neutral model, sole projected-Markdown serializer, and pandoc/xelatex profile. No
raw TeX assembled from user text, vault reconciliation, CLI additions, image work,
Scheduled/Waiting-on lane split, or section-selection feature work.

The source-cache and source-identity guarantees still apply: one read per canonical
source, occurrence-specific embed anchors, explicit caller date, original Tasks checkbox
statuses, and zero source writes. Ordinary notes retain Obsidian-only policy.

## Repairs

1. **Apply Bob policy throughout inline embeds, exactly once.** `bob.rs::apply_to_block`
   and the visibility/count/pruning walkers only visit block children. They skip
   `Inline::EmbeddedBlock`. With an area body `Context ![[Child]] after.` and Child
   containing one high-priority open task plus one done task, reading mode renders the
   done task, leaves the priority muted, reports zero tasks/omissions, and adds
   `No open tasks`.

   Extend the existing passes to visit embedded blocks in every inline-bearing
   container, including paragraph, task/list content, callout title, rich wrappers,
   chips, and labeled/log rows as applicable. Do not flatten surrounding rich content or
   build another renderer. Ensure visibility extraction, self-snapshot filtering,
   full/condensed logs, counts, empty-section decisions, and final anchors all agree
   with the rendered result. Exercise repeated and selected inline embeds and both
   reading modes. Closed appendix extraction must preserve occurrence-specific anchors
   and leave surrounding prose intact. Keep ordinary-note behavior covered.

   In `bob.rs::convert_task_children`, recognized relationship-row leftovers are
   transformed immediately and then again by `apply_to_task`'s child recursion. A task
   under a DEPENDS ON row gets two identical high chips. Give recursion one owner,
   preserving the child task and transforming it once. Add the regression with a nested
   task carrying priority, scheduled, and repeat fields.

2. **Read visible source semantics without losing original source attribution.**
   `TaskSource.raw_line` deliberately retains comments; Bob's field/tag/ref readers
   currently consult those raw bytes. For
   `- [ ] Visible %% [priority:: high] [repeat:: secret] %%`, output leaks `[high]` and
   `repeats secret`, though the syntax layer hid both fields.

   Preserve raw bytes for diagnostics and identity, while passing the syntax layer's
   effective, comment-stripped source spelling (including blockquote/callout prefix
   handling) to Bob semantic readers. Prefer carrying that already-computed spelling in
   source attribution, or equivalently reuse the existing stateful scanner;
   independently stripping each raw line must not resurrect text hidden by an earlier
   multiline comment. Audit field chips, `#ref` detection, self-snapshot links,
   outcome-created dates, relationship recognition, and managed logs for the same issue.
   Test percent and HTML comments, multiline boundaries, comments containing ref
   links/tags, and protected literal code. Do not duplicate the comment grammar.

3. **Preserve real project status while promoting its history.**
   `prepare_project_outcome_for_visibility` temporarily sets a closed `^prj` task to
   Open to avoid omission. `convert_log` therefore discards its CANCEL LOG even with
   `include_closed=true`. A canceled project outcome with a dated cancellation
   explanation loses that explanation entirely.

   Exempt the promoted root outcome from task filtering without falsifying its status
   during metadata/history projection. Preserve cancellation and completion meaning and
   the required history in the existing outcome-history location; respect full log mode.
   Continue excluding the outcome row itself from shown/omitted task counts, and keep
   ordinary outcome children and relationships. Test canceled and completed outcomes
   with histories in both modes and verify status/date chips.

4. **Keep appendix order and omission categories correct.** `project_snapshot` appends
   its temporary preamble section after normal sections, so an include-closed snapshot
   puts a closed task from the preamble after later tasks from the body. Feed
   visibility/extraction in document order, restoring the preamble without changing
   visible layout. Assert exact preamble/body/embedded appendix order and no duplicate
   descendants or anchors.

   `filter_task` uses `closed_count_in` for an omitted closed subtree without checking
   descendant self-snapshot provenance. A closed parent with a closed child linking a
   ref whose `source_note` is the root reports two closed omissions and zero self
   omissions. The contract requires self-snapshot precedence: one of each. Account for
   all omitted descendant occurrences using the same provenance rules, even when the
   ancestor is omitted; never let descendants escape. Test self refs below closed and
   self-ref ancestors in both modes, with no overlap between counters, unchanged source
   bytes, and CountingLoader's one-read guarantee.

5. **Project readable parent labels through existing syntax rules.** `parent_label` uses
   `first_source_h1`, a second ad hoc heading parser returning raw Markdown. A parent H1
   `# [[Goals|Project Goals]]` yields the literal kicker
   `AREA · [[Goals|Project Goals]]`. Reuse the syntax projector's title-candidate/
   visible-inline rules with the same SourceStore, or factor the appropriate narrow
   existing helper. Preserve explicit parent aliases and route fallback. Do not expand
   parent-note embeds or introduce recursive parent rendering. Test linked, emphasized,
   commented, and fenced title cases plus the one-read invariant.

## Integration and verification

The lander fetched origin and found master equal to `6c53e13`. Since this child epic
started, non-epic commits are `8e96375` (article formatter/shared Markdown comments and
type parsing), `f2ced0c` (capture-template repairs), `a3e45a3` (article capture
selectors), `280321a` (article outline/link planning), `ff97c62` (article CLI), and
`0e59ed0` (one-placeholder template rows). The shared comment helper extraction is
already integrated; retain it and the existing article type behavior. Article notes
remain ordinary snapshot notes unless their classified root is area/project. These
changes do not need to call Bob's projection seam. Other outer snapshot phases still own
CLI wiring and content-selection policies.

At implementation time inspect commits added since `6c53e13` for fresh drift and
integrate any relevant shared-model or projection callers. Keep `project_bob_snapshot`,
`SnapshotProjectionOptions`, and summary meanings documented. Add the seven regressions
as normal synthetic tests, extending them as specified above. Run
`cargo test --lib note_snapshot` and shared helper tests for any helper changed, then
the canonical **`just check`** gate. Use `/sase_monitor` for commands that need a long
handoff; any continuation must finish this tale's closeout. Do not run `just check-full`
or wait for this work's commit, push, SHA, or CI.

The only child `PROPOSED FOLLOW-UP:` was bob-cli-87.3.1.1 note #1: the two old
capture-template kinds/native-path failures owned by bob-cli-8b. The lander completed
`/sase_new_task` searches, recent-task sweep, active-epic review, and code/test
verification: `f2ced0c` fixes both and both now pass. Declined a new task or reopen
because the report is resolved. This disposition is recorded on bob-cli-87.3.1; carry it
into the close note. No follow-up remains for the coder to triage from the original
three children. The seven defects above are epic work, not task beads.

## Final step: finish this landing in the same turn

After the repairs and gate assessment, re-read bob-cli-87.3.1 and all three children
with `sase bead read ... -r ...` to check for new notes or unfinished descendants.
Confirm the approved Bob-layer scope and DECISIONS, the audit findings above, and any
post-audit integration changes are satisfied. Fix epic-caused failures; for any
genuinely unrelated failure, establish clean-base evidence and follow the task triage
workflow before recording the outcome. Existing apollo-only print-profile issues are
already noted on outer epic bob-cli-87 and are not a reason to ignore a different
failure here. Current athena snapshot/profile tests pass.

Run `sase bead epic-symbols bob-cli-87.3.1`. Resolve every entry by wiring, privatizing,
adding a justified non-test pragma, or deleting per policy; re-key a Justfile exemption
only to a still-open later bead that truly needs it. The audit found no entries and no
`symvision` recipe, but recheck because the tree may advance. Then close normally:

```bash
sase bead close bob-cli-87.3.1 --note "<actual verification of all three phases, seven repairs, intervening integration, gate results, symbol cleanup, and resolved follow-up from bob-cli-87.3.1.1 note 1>"
```

Run `just symvision` after the close if the recipe is available; otherwise explicitly
record its absence. Open the plans repo through `/sase_repo` and set **`status: done`**
in the frontmatter of **`202610/snapshot_bob_layer.md`**, the plan linked above.

Inspect the parent link with `sase bead read bob-cli-87.3.1 -r "Need the parent link"`.
The expected parent is phase **bob-cli-87.3**. Re-read it, verify this implementation
satisfies its Bob-layer scope, run `sase bead epic-symbols bob-cli-87.3`, resolve/re-key
any entries, and close **only that phase** normally:

```bash
sase bead close bob-cli-87.3 --note "<Bob-layer behavior, seven landing repairs, source/cache guarantees, integration seam, and actual verification results>"
```

Leave containing epic bob-cli-87 to its already-waiting land agent. If either close is
refused for unfinished descendants or symbols, finish the named work and retry; never
use `--force` to advance a successful nested landing. If an unexpected parent link or
incomplete parent prevents closeout, record the precise blocker on that parent and
report it. Do not mark an unfinished plan done. Use `/sase_final` for the eventual
normal final response; the host commits only after the coder's turn, so this closeout
must happen before and independently of that commit.
