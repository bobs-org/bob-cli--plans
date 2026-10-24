---
tier: epic
title: 'Backlog batch 6: color-grouped quote wraps, BLOCKS-shifted line indexes, project
  promotion anchor race, and Mac capture panel CI'
goal: Close five prioritized task beads. Stop color-grouped Highlights sidecars from
  rendering wrapped quote text as Bryan's Comment, stop a generated BLOCKS line from
  mis-aiming the review-walk settle and the counted Depends-On planner, stop project
  promotion from failing when a concurrent frontmatter write lands between the seed
  and anchor writes, and make bob-mac-capture's three CapturePanelModelTests submit-argv
  failures pass. The lander continues the backlog through further epics of at most
  five phases and five task beads until every project task bead is closed.
parent_bead: bob-cli-an.6.6.6.5
decisions:
  color_grouped_quote_id:
    ask: When the color-grouped quote fix changes a live highlight's quote text, should
      bob re-mint its ^h- id?
    choices:
      remint: Re-mint and tombstone the affected id, since quote text is part of identity
      reuse: Keep the existing ^h- id even though the quote text changes
    default: remint
    why: Matches batch 5's linked-page quote fix; one live note is cheap to remint
    answer: remint
phases:
- id: color-quote-wrap
  title: Keep hard-wrapped quote lines inside color-grouped highlights
  size: medium
  depends_on: []
  description: 'color-quote-wrap: resolve bob-cli-bf by applying bare-line quote continuation
    to color-grouped Highlights sidecar chunks, not only linked-page chunks, and apply
    color_grouped_quote_id.'
- id: blocks-shift-indexes
  title: Map review-walk and counted Depends-On line indexes through BLOCKS insertions
  size: large
  depends_on: []
  description: 'blocks-shift-indexes: resolve bob-cli-94 and bob-cli-95 by threading
    the answered or pending task''s post-write line through generated BLOCKS-line
    edits in bob-navigation-hotkeys instead of reusing pre-write indexes.'
- id: promotion-anchor-race
  title: Persist project anchors despite a concurrent frontmatter write
  size: medium
  depends_on:
  - blocks-shift-indexes
  description: 'promotion-anchor-race: resolve bob-cli-79 so that promoting a task
    with converted child bullets persists its project anchors when bob-project-tasks
    or Templater touch the new note after the seed write.'
- id: mac-panel-argv-ci
  title: Make the three CapturePanelModelTests submit-argv tests pass on macOS CI
  size: large
  depends_on: []
  description: 'mac-panel-argv-ci: resolve bob-cli-be by finding why the bare x*!2
    and = override-restart submit argv no longer reach the fake-bob record, then fix
    the client or fixture without weakening the assertions.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 09:36:37
status: wip
bead_id: bob-cli-an.6.6.6.5.6
---

- **PROMPT:** [prompts/202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md)
- **PARENT:** [202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.md)

# Backlog batch 6: color-grouped quote wraps, BLOCKS-shifted line indexes, project promotion anchor race, and Mac capture panel CI

## Scope and triage

This continues the backlog campaign of bounded epics. Batches 1–5 (`bob-cli-an`,
`bob-cli-an.6`, `bob-cli-an.6.6`, `bob-cli-an.6.6.6`, `bob-cli-an.6.6.6.5`) are closed.
Batch 5 closed `bob-cli-4r`, `bob-cli-2l`, and `bob-cli-b7`. `bob-cli-ba` and
`bob-cli-bb` were closed by concurrent epics `bob-cli-bd.1` and `bob-cli-bc.1` on top of
batch 5's implementation. Batch 5's lander filed `bob-cli-be` and `bob-cli-bf`. It also
recorded a DISCOVERED ISSUE on active epic `bob-cli-bd`: bd.1's level-aware page
headings (`6ec2f16a`) turned bob-cli master CI red on
`tests/cli/highlights/continuations.rs::continuation_scan_tucks_two_pages_and_is_idempotent`
and `::continuation_scan_reports_dangling_marker`. Those two failures belong to
`bob-cli-bd`. Treat them as a recorded owner, not as this epic's work.

On 2026-10-24, after that closeout, a fresh enumeration found **127 unresolved tasks:
123 ready and four snoozed** (`bob-cli-3o`, `bob-cli-6q`, `bob-cli-aa`, `bob-cli-ah`).
None are claimed or in progress. This epic owns exactly these five task beads; no phase
may add a sixth:

| Rank | Task         | Reason for priority                                                                                                                                                                                                                                       | Phase                   |
| ---- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| 1    | `bob-cli-bf` | Content misattribution: in color-grouped sidecars a hard-wrapped quote stops at the first line without `>` and the rest renders as Bryan's Comment. Live: `ref/chat/remote_sase_agents_consolidated.md`. The `ba` fix only covered linked-page sidecars.  | `color-quote-wrap`      |
| 2    | `bob-cli-94` | Silent walk defect: `settleReviewOrigin` reads the answered task's after-line at its pre-write index, which a generated BLOCKS line above shifts, so the walk can advance or stay incorrectly (`decisions:answering-advances-the-walk`).                  | `blocks-shift-indexes`  |
| 3    | `bob-cli-be` | Deterministic bob-mac-capture macOS CI blocker: three CapturePanelModelTests submit-argv assertions fail on `54131b1` (run 40090432753) and `6d41b2e` (run 40095658827). Nothing else in the 2938-test run fails.                                         | `mac-panel-argv-ci`     |
| 4    | `bob-cli-95` | Loud refusal from the same root cause as `94`: counted same-note Depends-On (`N<Ctrl+Shift+B>`) with the prerequisite above the sources fails "Could not plan dependency navigation for every source task" whenever BLOCKS mirroring is on (the default). | `blocks-shift-indexes`  |
| 5    | `bob-cli-79` | Loud refusal Bryan hit four times: promoting a linked task with child bullets shows "project anchors could not be persisted" because a debounced frontmatter write lands between the seed and the preimage-checked anchor write.                          | `promotion-anchor-race` |

Ranking: content misattribution first (`bf`), then the silent walk defect (`94`), then
the deterministic CI blocker (`be`), then loud write refusals (`95`, `79`). `95` shares
`94`'s root cause class, so they share a phase. These were deferred with reasons, not
silently absorbed:

- `bob-cli-b8` repairs live vault identities and is a discretionary feature. It needs
  its own implement/decline decision and a reviewed live mutation.
- `bob-cli-3w` (+23), `bob-cli-5l` (+9), `bob-cli-4k` (+4), `bob-cli-b9`, and
  `bob-cli-8q` are flakes; deterministic failures come first.
- `bob-cli-86` (gist clip numeric author) and `bob-cli-7u` (URL dedupe `%40` vs `@`) are
  silent metadata defects with narrower blast radius than this batch.
- Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x` are monitoring and
  UI health items with no current content or security loss evidence.
- `bob-cli-3o` is still snoozed until live hooks report `legacy_dependency_children: 0`.
  `bob-cli-6q` (wake 2026-10-26), `bob-cli-aa` (2026-11-06), and `bob-cli-ah`
  (2026-11-04) keep their publication/data gates.

Open epic plans at this snapshot: `bob-cli-28`, `bob-cli-3j`, `bob-cli-5w`,
`bob-cli-66`, `bob-cli-87`, `bob-cli-8b`, `bob-cli-9k`, `bob-cli-9k.8`, `bob-cli-bc`,
and `bob-cli-bd`. None owns this batch. **`bob-cli-bd` (annotation sections) is actively
editing `src/native/highlights_ref/sidecar.rs` and `sidecar_render.rs`** (bd.2, bd.4,
bd.6, bd.7, and bd.8 in progress). `color-quote-wrap` must rebase onto whatever bd has
landed and must not undo bd's level-aware page headings or section grammar. `bob-cli-5w`
(Successor Links) touches bob-plugins completion paths, not counted dependency planning
or project promotion. Recheck ownership before implementation.

## Working rules and handoffs

Read the assigned task, this epic's finalized decisions, and phase dependencies through
`sase bead read`. Reproduce against the current tree before changing anything. A task
that has meanwhile been fixed can be closed with independently checked evidence instead
of reimplemented. Record baseline, chosen fix, validation, and remaining constraints on
the task and phase. Use `sase bead close` with `done`, `superseded`, or a
human-authorized `canceled` resolution; never hand-edit the store or runtime-owned
statuses. Phase workers leave task beads open for the lander; the lander closes them
after rechecking evidence. Phase workers never create beads: record out-of-scope
discoveries as `PROPOSED FOLLOW-UP:` notes on your own phase.

Use `/sase_repo` before accessing any other repository (`bob-plugins`,
`bob-mac-capture`). Read each opened repo's `AGENTS.md` and follow its build and test
commands. Read sidecar evidence and attached artifacts only through
`sase artifact read`. bob-mac-capture builds and tests on the Mac (`ssh mac`, see the
`tailnet` reference memory); if the Mac is unreachable, use `/sase_monitor` to wait and
retry. Do not claim a Swift change verified without running the affected XCTests on the
Mac; a typecheck alone is not verification. Bob Mac Capture is a thin client of bob
(`decisions:mac-capture-is-a-thin-client`).

bob-cli phases run the focused tests for the code they touch, then `just check` once on
their integrated change (use `/sase_monitor` if it runs long), and `just docs-check`
when they change docs. Until `bob-cli-bd` fixes them, the two
`highlights::continuations` CLI failures are a recorded owner. Follow the daily release
train: do not edit package versions or `CHANGELOG.md`, and do not publish a manual
release. This workspace builds into `$CARGO_TARGET_DIR`; the workspace-local
`target/debug/bob` can be weeks stale, so never use it as evidence. bob-plugins phases
run the plugin test suite their `AGENTS.md` names, and deploy only through
`bob plugins sync`. Never mutate the live vault during validation except through a step
the phase explicitly allows. Use isolated `BOB_DIR` fixtures for capture, highlights,
and hooks tests.

`blocks-shift-indexes` and `promotion-anchor-race` both edit
`plugins/bob-navigation-hotkeys`, whose generated `main.js` is committed. They are
sequential so the second phase rebuilds on the first. The other two phases are
independent and can run in parallel.

## Phase color-quote-wrap — bob-cli-bf

Evidence (bob-cli-bf, 2026-10-24): `parse_sidecar_chunk`
(`src/native/highlights_ref/sidecar.rs`) applies bare-line quote continuation
(`is_quote_continuation_line`) only when `linked_page_style` is true. That flag comes
only from a linked `[Page N](highlights://...)` page heading. Color-grouped exports
(`#### Yellow` color headings, `##### YYYY-MM-DD:` date headers, no page headings) use
the same hard-wrap shape, but every wrap line without `>` becomes comment text. Live
read-only repro: `~/bob/lib/chat/remote_sase_agents_consolidated.md` (second chunk)
versus `~/bob/ref/chat/remote_sase_agents_consolidated.md`. The quote stops at "...run
the" and `existing SASE mobile gateway there. Other > machines ...` renders as a
Comment. A read-only scan found one such live sidecar.

1. Recheck `bob-cli-bd`'s landed sidecar changes, then reproduce from the live sidecar
   (read-only) and add a color-grouped fixture: color heading, date header, and a quote
   whose wrap lines alternate between `>` and bare lines, followed by a blank-line
   comment.
2. Apply quote continuation to color-grouped Highlights chunks as well as linked-page
   chunks. Detect them from the export shape (a color heading or annotation-date header
   in a sidecar with no page headings); never enable it for simple `## Page N` sidecars.
   Keep every boundary from the `ba` fix: a blank line, page headings, annotation-date
   headers, `Comment:`/`Note:` labels, and marker-list fields all end the quote. Do not
   disturb the batch 5 color-grouped marker-mirror exclusion in `region.rs`.
3. Apply **color_grouped_quote_id**.
4. Add regressions: the wrapped quote stays whole, a blank-line comment still becomes
   the Comment, a simple-style sidecar is unchanged, and the color-grouped marker mirror
   is still excluded.
5. Run focused `highlights_ref` and CLI `highlights::` tests, then `just check`. Do not
   hand-edit the live vault. Confirm with a fresh-build `bob ref sync --dry-run` on the
   live PDF that the next sync would rewrite the note (`note_action: update`), or show
   the same with a real sync on a scratch copy.

Acceptance: the live repro's parse keeps the whole quote, and the comment is only the
blank-line-separated remark. Fixtures cover continuation versus a real comment. Ids
follow the finalized decision. `just check` passes aside from recorded owners.

> [!decision] color_grouped_quote_id = remint Re-mint and tombstone the affected `^h-`
> id when quote text changes, as batch 5 did for linked-page quotes.

> [!decision] color_grouped_quote_id = reuse Keep the existing `^h-` id even though
> quote text changes; document why identity is unchanged.

## Phase blocks-shift-indexes — bob-cli-94 and bob-cli-95

Evidence (bob-cli-94): `BulletPropertyPickerModal.settleReviewOrigin`
(`plugins/bob-navigation-hotkeys/src/350-picker-task-card.js`, `reviewLineIndex` near
line 790) reads the after-line with `getEditorLine(this.editor, this.reviewLineIndex)`,
which is the pre-write index. When a same-note prerequisite above the reviewed task
gains a generated `🧱 BLOCKS:` line, the reviewed task moves down one line. The settle
then reads the BLOCKS line, and `continueReviewWalkAfter` may misjudge whether the row
left today's walk.

Evidence (bob-cli-95): `planCountedLocalTaskDependency`
(`plugins/bob-navigation-hotkeys/src/270-counted-batches.js`) plans counted sources
bottom-to-top and assumes each pending source keeps its snapshot line. With the
prerequisite above the sources and `blocks_lines` on, planning the lowest source inserts
a BLOCKS line above the others. Their `target.line` goes stale and the planner fails
"Could not plan dependency navigation for every source task". The bead includes a Node
harness repro. The cross-note counted writers in `src/420-picker-counted-dependency.js`
(~l.333 and ~l.603) stamp freshness at the pre-write line too; `plan.parentLineNext` is
the post-plan line, and `applyDependencyEdit` already uses it.

This phase is **large**: plan before implementing.

1. Reproduce both: a review-walk harness where a same-note prerequisite above the landed
   task gains a BLOCKS line, and the `95` Node harness call.
2. Use one line-mapping approach for both. Thread each answered or pending task's line
   through the accumulated line edits (`mapLineThroughLineEdits`, the alignment
   `applyEditorContentTransaction` computes, or `plan.parentLineNext`) instead of
   reusing snapshot indexes. Stamp freshness at the post-write line.
3. Add regressions: the walk settles on the correct row after the BLOCKS insertion, and
   it advances or stays per `decisions:answering-advances-the-walk`. Counted add
   succeeds with the prerequisite above the sources, with BLOCKS on and off. The
   cross-note stamp lands on the dependent.
4. Run the plugin suite the bob-plugins `AGENTS.md` names (including the generated
   entrypoint check), deploy with `bob plugins sync`, and record the
   bob-navigation-hotkeys version.

Acceptance: both repros pass, the walk honors the decision record, the full plugin suite
passes, and the deployed version is recorded.

## Phase promotion-anchor-race — bob-cli-79

Evidence (bob-cli-79): promoting a task with Ctrl+Shift+Alt+N
(`createProjectNoteFromTask`) fails "Created project, but project anchors could not be
persisted; source task was kept" when the source task (a) has a live Task Link under
today's open Pomodoro and (b) has child bullets that the seed converts into resident
tasks without block ids. The anchor write (`670-plugin-project-files.js` ~L170 →
`writeProjectPromotionFileChange`) is preimage-checked against the seed content.
bob-project-tasks' 250 ms debounced `processFrontMatter` (adding
`task_count`/`open_task_count`) or the Templater-opened editor changes the note first.
Four live failures are recorded (vault commits `3b478123`, `bf9ba3f9`, `55c4dd0e`,
`b1302c0c`).

1. Reproduce with the bead's Node harness factory
   (`scripts/test-navigation-hotkeys-forward-promotion.cjs`), adding a frontmatter-only
   change between the seed and anchor writes.
2. Make the anchor write tolerate concurrent changes that cannot conflict with it
   (frontmatter-only, or anything outside the anchored task lines). Re-read and re-plan
   the anchors against current content, keep the preimage check for the task lines the
   anchors touch, and keep the editor and file paths consistent. Never drop real user
   edits to the converted tasks.
3. Add regressions: a frontmatter write between seed and anchors still persists the
   anchors, while a conflicting edit to an anchored task line still refuses and keeps
   the source task.
4. Run the plugin suite, deploy with `bob plugins sync`, and record versions.

Acceptance: the harness repro persists anchors, a real conflicting edit still refuses,
the plugin suite passes, and the deployed version is recorded.

## Phase mac-panel-argv-ci — bob-cli-be

Evidence (bob-cli-be): bob-mac-capture macOS CI fails only these three nodes on
`54131b1` and `6d41b2e`:

- `CapturePanelModelTests.swift:2472` `testBareXSelectionPreviewSubmitsUntouchedDraft`:
  the `x*!2` iteration's `argv=capture --format json --refuse-override -- x*!2` is
  missing from the fake-bob record (`x*`, `x!`, and `x*2` pass)
- `:3260` `testOverrideRestartLivePreviewAndSubmitUseRestartArgvFooterAndStatus`:
  `argv=capture --format json -- =` missing
- `:3359` `testUnpreviewedOverrideRefusalKeepsDraftAndSecondReturnCommits`:
  `argv=capture --format json -- =` missing

They are what remains of the 21 capture-argv failures from batch 4's capability-gate
change (`54131b1`). That change was verified only by a typecheck. The root cause is not
confirmed: `Tests/Fixtures/fake-bob` advertises no `pull_in` capability, so the
`--require-pull-in-choice` gate is probably not the cause.

This phase is **large**: plan before implementing once the root cause is known.

1. Open bob-mac-capture with `/sase_repo` and reproduce on the Mac with the `AGENTS.md`
   scratch flow (rsync without `.build`/`.git`, `bundle.sh`, then run the three
   XCTests). Linux `just portable-check` does not execute them.
2. Diff the fake-bob record against the expected argv for each failing draft, and decide
   whether the client (submit path, refuse-override fallback, guard fallback) or the
   fake-bob fixture branch is wrong. Fix it so each submit records its real argv. Do not
   weaken the assertions or drop the `x*!2` iteration.
3. Run the full `BobMacCaptureTests` on the Mac and `just portable-check`. Do not use
   Bryan's checkout or `just install`.

Acceptance: the three named tests pass on the Mac, and macOS CI on the landed SHA is
green aside from other recorded task owners. The lander confirms the GitHub Actions test
job, using `/sase_monitor` to wait.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a fifth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and CI state, including whether
another epic changed sidecar quote parsing (especially `bob-cli-bd`), review-walk
settle, counted dependency planning, project promotion, or CapturePanelModel submit. Run
appropriate integrated checks once and document genuine preexisting failures with their
existing task owners. Build bob through `$CARGO_TARGET_DIR`, never the stale workspace
`target/debug/bob`. Require real evidence before closing each task:

- `bf`: the fixture plus a fresh-build live dry-run, or a scratch real sync, keep the
  whole quote with no Comment remainder. Honor `color_grouped_quote_id`.
- `94`: review-walk regression with a BLOCKS insertion above the landed task.
- `95`: counted add with the prerequisite above the sources succeeds with BLOCKS on.
- `79`: the harness persists anchors across a concurrent frontmatter write.
- `be`: the three tests pass on the Mac, and macOS CI on the landed SHA is green aside
  from other recorded owners. Use `/sase_monitor` for the wait.

Because `sase plan propose` terminates the runner without running the commit finalizer,
the lander must commit every repository it changed (bob-cli, bob-plugins,
bob-mac-capture, the plans sidecar status edit, and any other opened repo) with
`/sase_git_commit`, which this approved plan explicitly authorizes for that purpose,
before proposing the successor.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include `bob-cli-b8` for live identity repair (needs an implement/decline
   decision); `bob-cli-86` and `bob-cli-7u` for silent metadata defects; `bob-cli-2t`
   for an empty close-summary line; flakes `bob-cli-3w`, `bob-cli-5l`, `bob-cli-4k`,
   `bob-cli-b9`, and `bob-cli-8q`; Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and
   `bob-cli-9x`. This is guidance, not a preassigned batch, and concurrent work may
   obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.
