---
tier: epic
title: 'Backlog batch 5: Highlights quote and marker render, Pomodoro name reuse,
  completed-link dedupe, and Mac scan CI'
goal: Close five prioritized task beads. Stop Highlights from dropping quoted lines
  that look like headings and from rendering the page-1 marker as an annotation, keep
  completed multi-word Pomodoro names when the link form reuses them, stop completed-reference
  relocation from duplicating a Task Link already in the destination Pomodoro, and
  make bob-mac-capture RefsLibrary scan tests record ref scan argv again. The lander
  continues the backlog through further epics of at most five phases and five task
  beads until every project task bead is closed.
parent_bead: bob-cli-an.6.6.6
decisions:
  highlight_id_on_quote_fix:
    ask: When the quote/heading boundary fix changes two live highlight quote texts,
      should bob re-mint their ^h- ids?
    choices:
      remint: Re-mint and tombstone the two affected highlight ids because quote text
        is part of identity
      reuse: Keep the existing ^h- ids even though the quote text changes
    default: remint
    why: Quote text is part of highlight identity; reminting two notes is cheaper
      than mixed ids
    answer: remint
phases:
- id: highlight-quote-heading
  title: Keep hard-wrapped '# ' quote lines inside the highlight
  size: medium
  depends_on: []
  description: 'highlight-quote-heading: resolve bob-cli-ba by treating a heading-shaped
    sidecar line that immediately follows a quote line as quote continuation, applying
    highlight_id_on_quote_fix for the two notes whose quote text changes.'
- id: highlight-marker-mirror
  title: Stop rendering the page-1 marker mirror as an annotation
  size: large
  depends_on:
  - highlight-quote-heading
  description: 'highlight-marker-mirror: resolve bob-cli-4r by skipping the real page-1
    marker mirror even when a setext title preamble consumes the current one-shot
    skip.'
- id: pomodoro-name-reuse
  title: Reuse a completed Pomodoro's canonical name on the link form
  size: small
  depends_on: []
  description: 'pomodoro-name-reuse: resolve bob-cli-bb by naming a CompletedOnly
    placeholder from the completed entry, not from canonicalize_pomodoro_name(selector).'
- id: completed-link-dedupe
  title: Do not relocate a completed Task Link into a Pomodoro that already has it
  size: medium
  depends_on: []
  description: 'completed-link-dedupe: resolve bob-cli-2l by passing dedupe_into_destination
    from reconcile and matching that contract in the bob-plugins task-status-cycler.'
- id: mac-refs-scan-ci
  title: Make RefsLibraryTests record the ref scan argv again
  size: medium
  depends_on: []
  description: 'mac-refs-scan-ci: resolve bob-cli-b7 by restoring a single recorded
    ref scan in RefsLibraryTests so macOS CI is green aside from other recorded owners.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.land
proposed_with: grok/grok-4.6
decided_by: auto
create_time: 2026-10-24 07:37:06
status: wip
bead_id: bob-cli-an.6.6.6.5
---

- **PROMPT:** [prompts/202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch5_highlights_render_pomodoro_name_duplicate_link_mac_ci.md)
- **PARENT:** [202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md)
- **BEAD:** [bob-cli-an.6.6.6.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.md)

# Backlog batch 5: Highlights quote and marker render, Pomodoro name reuse, completed-link dedupe, and Mac scan CI

## Scope and triage

This continues the backlog campaign of bounded epics. Batches 1–4 (`bob-cli-an`,
`bob-cli-an.6`, `bob-cli-an.6.6`, `bob-cli-an.6.6.6`) are closed. Batch 4 closed
`bob-cli-b5`, `bob-cli-6v`, and `bob-cli-7i` in its phases, then a finish tale closed
`bob-cli-8a` (mixed short-cluster flag guard plus clap-declaration check) and
`bob-cli-al` (bob-authoritative relation labels, no Swift formatter). Mac model tests
now omit `--require-pull-in-choice` unless a same-executable preview advertised
`pull_in.supported`. Verification from that closeout: bob-cli `just check` green;
bob-mac-capture `just portable-check` 1555/0; AppKit scratch typecheck on mac PASS
`BobMacCaptureTests`. Known `bob-cli-b7` RefsLibraryTests remain.

On 2026-10-24, after that closeout, a fresh enumeration found **130 unresolved tasks:
126 ready and four snoozed** (`bob-cli-3o`, `bob-cli-6q`, `bob-cli-aa`, `bob-cli-ah`).
None are claimed or in progress. This epic owns exactly these five task beads; no phase
may add a sixth:

| Rank | Task         | Reason for priority                                                                                                                                                                                                                                                                                                                         | Phase                     |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| 1    | `bob-cli-ba` | Content loss: a hard-wrapped quote line that starts with `# ` ends the highlight early, drops those words, and renders the rest as a Comment. Two live notes (gastown_readme p.8, sase_listen_antennapod_setup p.4).                                                                                                                        | `highlight-quote-heading` |
| 2    | `bob-cli-4r` | Silent annotation corruption: a setext title preamble consumes the one-shot marker skip, so the page-1 PDF marker renders as a `[!note]` annotation. The bead's vault scan found **113 of 115** annotated ref notes, often with a stale `status: wip`. Batch 4's "affects two notes" undercount is the two _unaffected_ notes in that scan. | `highlight-marker-mirror` |
| 3    | `bob-cli-bb` | Silent name fork: link-form reuse of a completed multi-word Pomodoro names the new entry from the slug (`DEEP WORK` → `DEEP-WORK`). The live vault already has an open `— SASE-V18` under completed `— SASE V18`. 18 of 86 vault Pomodoro names contain spaces.                                                                             | `pomodoro-name-reuse`     |
| 4    | `bob-cli-2l` | Silent ledger duplicate: completing a task whose link sits in both a finished session and a carried placeholder relocates a second struck link into the completed Pomodoro. Rust already has `dedupe_into_destination`; reconcile and the plugin cycler still skip it.                                                                      | `completed-link-dedupe`   |
| 5    | `bob-cli-b7` | Deterministic bob-mac-capture macOS CI blocker: three RefsLibraryTests expect one `ref scan` argv and see zero after the scan-receipt observer. +1 from batch 4's land audit on runs 40084777082 and 40086037387.                                                                                                                           | `mac-refs-scan-ci`        |

Ranking: content loss first (`ba`), then the largest silent render corruption (`4r`),
then silent write defects (`bb`, `2l`), then the remaining deterministic CI blocker
(`b7`). Age and corroboration break remaining ties: `2l` is 24 days old with a partial
Rust fix; `b7` now leads CI because `6v` is closed. These were deferred with reasons,
not silently absorbed:

- `bob-cli-94` (review-walk settle at a pre-write index) is the next silent walk defect.
- `bob-cli-95` and `bob-cli-79` fail loudly and write nothing.
- `bob-cli-b8` repairs live vault identities and needs its own reviewed mutation.
- `bob-cli-3w` (+23) and `bob-cli-5l` (+9) are flakes; CI green comes first.
- `bob-cli-3o` is past its 2026-10-15 wake but still snoozed until live hooks report
  `legacy_dependency_children: 0`. Do not unsnooze without that evidence.
- `bob-cli-6q` (wake 2026-10-26), `bob-cli-aa` (2026-11-06), and `bob-cli-ah`
  (2026-11-04) keep their publication/data gates.
- `bob-cli-7q` remains a design change to `decisions:capture-link-drops-blocking-deps`.

Open epic plans at this snapshot: `bob-cli-28`, `bob-cli-3j`, `bob-cli-5w`,
`bob-cli-66`, `bob-cli-87`, `bob-cli-8b`, `bob-cli-9k`, and `bob-cli-9k.8`. None owns
this batch. `bob-cli-87` is note snapshots, not sidecar quote/marker parse.
`bob-cli-9k.8` is Pomodoro homes closeout, not CompletedOnly name spelling. `bob-cli-8b`
is the template shelf. Scan-receipt work that first reddened `b7` belongs to closed
`bob-cli-at`. Recheck ownership before implementation.

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
retry, and do not claim a Swift change verified without a Mac build and test run. Bob
Mac Capture is a thin client of bob (`decisions:mac-capture-is-a-thin-client`).

bob-cli phases run the focused tests for the code they touch, then `just check` once on
their integrated change (use `/sase_monitor` if it runs long), and `just docs-check`
when they change docs. Follow the daily release train: do not edit package versions or
`CHANGELOG.md`, and do not publish a manual release. bob-plugins phases run the plugin
test suite their `AGENTS.md` names, and deploy only through `bob plugins sync`. Never
mutate the live vault during validation except through a step the phase explicitly
allows. Isolated `BOB_DIR` fixtures for capture and highlights tests; do not use a stale
`target/debug/bob`.

`highlight-quote-heading` and `highlight-marker-mirror` both edit
`src/native/highlights_ref/sidecar_render.rs` (and likely `sidecar.rs`). They are
sequential so the marker-mirror planner sees the quote-boundary change. The other three
phases are independent and can run in parallel. `completed-link-dedupe` commits to
bob-cli and bob-plugins.

## Phase highlight-quote-heading — bob-cli-ba

Evidence (bob-cli-ba, 2026-10-24): `is_quote_continuation_line` in
`src/native/highlights_ref/sidecar_render.rs` rejects any `is_markdown_heading` line, so
a hard-wrapped quote line that starts with `# ` ends the quote. `parse_sidecar_chunk`
then sends the rest to the comment side and drops the heading-shaped line. Live
read-only repros:

1. `~/bob/lib/docs/gastown_readme.textbundle/text.markdown` lines 150-157 versus
   `~/bob/ref/docs/gastown_readme.md` lines 79-83: `"Set default agent"` is missing and
   the rest renders as a Comment.
2. `~/bob/lib/chat/sase_listen_antennapod_setup.md` line 63 versus
   `~/bob/ref/chat/sase_listen_antennapod_setup.md` lines 51-55:
   `# should start with <?xml` is dropped.

A read-only scan of 178 sidecars found exactly these two `#`-led lines directly after a
`>` line. Real user comments are separated by a blank line (161 of 161).

1. Reproduce from those two vault pairs (read-only) and add a fixture that wraps a `# `
   line inside a linked-style quote with no blank line.
2. In linked style, treat a heading-shaped line that immediately follows a quote line as
   quote continuation unless it is a page heading or an annotation-date header. Start
   the comment side only after a blank line.
3. Apply **highlight_id_on_quote_fix**. Closed `bob-cli-9o` preserved ids across
   annotation insertion; this event changes quote text, which is a different identity
   input.
4. Add regressions for a true comment after a blank line, a page heading, and an
   annotation-date header so those still leave the quote.
5. Run focused `highlights_ref` tests, then `just check`. Do not hand-edit the live
   vault; the next highlights sync is how the two notes recover.

Acceptance: both live repro quotes keep the `# ` line inside the quote and do not
promote the rest to Comment. Fixtures cover continuation versus real comments. Ids
follow the finalized remint/reuse decision. `just check` passes.

> [!decision] highlight_id_on_quote_fix = remint Re-mint and tombstone the two affected
> `^h-` ids when quote text changes.

> [!decision] highlight_id_on_quote_fix = reuse Keep the existing `^h-` ids even though
> quote text changes; document why identity is unchanged.

## Phase highlight-marker-mirror — bob-cli-4r

Evidence (bob-cli-4r, +1): Highlights Markdown sidecars with a setext title preamble
before `#### [Page 1](...)` cause `ref/` notes to render that page-1 marker list as an
annotation (`> [!note] - status: wip` plus a `^h-` id). Likely cause:
`render_sidecar_highlights` / `annotation_tasks` skip only the _first_ annotation for
which `is_sidecar_marker_mirror` is true (`skipped_marker_note` one-shot). A setext
title parses as a `StandaloneNote` and consumes that skip, so the real marker list
renders. Fixture `tests/fixtures/highlights_ref/linked_sidecar.md` has no setext
preamble. The bead's vault scan: 113 of 115 annotated notes; both unaffected notes lack
that preamble. Recount on the current vault before treating the blast radius as two
notes.

This phase depends on `highlight-quote-heading` because both touch `sidecar_render.rs`.
Reproduce first; if current master already skips the marker, close `bob-cli-4r` with
evidence instead of reimplementing.

1. Add a fixture with the setext title + Page 1 marker + a real highlight. Assert the
   marker does not become a `[!note]` annotation and the real highlight still does.
2. Skip every sidecar marker mirror, not only the first standalone note. Keep real
   annotations. Confirm `is_sidecar_marker_mirror` on the page-1 list after a setext
   title.
3. Re-count live `ref/` notes that still leak `status:` / `parent:` marker fields as
   annotations. Do not bulk-edit the live vault; recovery is the next highlights sync
   after the renderer is correct. Record whether phantom `^h-` ids on those mirrors
   disappear on rewrite.
4. Run focused highlights_ref tests, then `just check`. This phase is **large**: plan
   before implementing if the skip logic or identity fallout is wider than the one-shot
   flag.

Acceptance: a setext-preamble sidecar does not render the page-1 marker as an
annotation. Real highlights still render. Focused tests cover the preamble shape.
`just check` passes. The lander confirms the live count dropped after a dry-run or
documented sync path, without an unauthorized live-vault rewrite.

## Phase pomodoro-name-reuse — bob-cli-bb

Evidence (bob-cli-bb): in an isolated vault whose `## Pomodoros` holds completed
`— DEEP WORK`, `bob capture -d '=#deep-work'` reuses `DEEP WORK`, but
`bob capture -d '@inbox_t:foo1#deep-work'` creates `DEEP-WORK`. `select_named` returns
`CompletedOnly(entry)`; `insert_named_placeholder` names the new entry
`canonicalize_pomodoro_name(selector)`. The start path already prefers `entry.name`.
Audit `capture/pomodoro_insert.rs` (`insert_named_pomodoro_child_block`) and
`capture/pomodoro_link.rs` for the same CompletedOnly arm.

1. Reproduce in an isolated vault with `BOB_NOW` set; do not use the live vault.
2. When the selection is `CompletedOnly(entry)`, name the new entry with the completed
   entry's canonical name on every creating path (link, child-block insert, start).
3. Add one regression per path with a multi-word completed name. Keep slug matching
   (`#deep-work` still finds `DEEP WORK`).
4. Run focused capture Pomodoro tests, then `just check`. Do not rename live `SASE-V18`
   as part of this phase; that leftover fork is evidence, not the fix.

Acceptance: the isolated link-form repro creates `DEEP WORK`, not `DEEP-WORK`. Start and
child-block paths keep the same spelling. `just check` passes.

## Phase completed-link-dedupe — bob-cli-2l

Evidence (bob-cli-2l, vault commit `8e502f3d` on `2026/20260929.md`): completing
`^pomodoro-ledger-daily-roadmap` while its link sat under completed GTD **and** a later
open placeholder relocated a second unmarked struck link into the completed Pomodoro.
`task_status_hooks::structure::plan_structural_changes` already takes opt-in
`dedupe_into_destination` (commit 7c8d854). Capture-side `retire_completed_links` passes
`true`; reconcile in `task_status_hooks/sync.rs` still passes `false`. The bob-plugins
task-status-cycler JS relocation still lacks the same check.

1. Reproduce with an isolated vault (completed Pomodoro already holding the link, later
   open untimed placeholder also holding it, no open timed Pomodoro), then complete the
   task via `bob task-status-hooks --dry-run` and, after `/sase_repo` on bob-plugins,
   via the cycler's closed-reference retirement path.
2. Pass `true` from reconcile. Keep capture retirement passing `true`.
3. Teach the plugin cycler the same destination-dedupe contract
   (`docs/task-status-hooks.md` Pomodoro Marker relocation). A live non-transcluded link
   moved to the completed fallback still gets the tomato marker when it is the first
   copy; a duplicate is not inserted.
4. Add regressions in both repos. Run focused hooks tests and the plugin suite the
   bob-plugins `AGENTS.md` names, then bob-cli `just check`. Deploy plugins only through
   `bob plugins sync` if the JS change requires it, and record versions.

Acceptance: the isolated repro leaves a single struck link in the completed Pomodoro.
Reconcile and Ctrl+Enter/cycler agree. `just check` and the plugin tests pass aside from
recorded preexisting owners.

## Phase mac-refs-scan-ci — bob-cli-b7

Evidence (bob-cli-b7, +1): bob-mac-capture macOS CI fails the same three tests on every
SHA since scan-receipt compilation recovered (`1c35b30`, `6e1f84c`, later `22598dc` run
40084777082 and `8400bd3` run 40086037387):

- `testScanRunsOneScanArgv` expected 1 `ref scan` argv, got 0
- `testSecondScanMidFlightReturnsFalse` same
- `testFinishedAfterPostScanListWithNewRows` expected scan and post-scan list in the
  fake-bob record

Last fully green RefsLibrary run: `9fb67f3`. Lead:
`feat(refs): observe scan receipts and notify for open PDF updates` (closed
`bob-cli-at.6`) changed how RefsLibrary invokes or records scan. That is a lead, not a
verified root cause. The one `RefsRankingTests.testPerformanceGuard` 10.13s overrun on
40022204228 did not recur; leave it unless it comes back.

1. Open bob-mac-capture with `/sase_repo`. Reproduce with the Mac `AGENTS.md` commands
   (scratch rsync, `bundle.sh`, tests). Linux `just portable-check` does not execute
   these AppKit tests.
2. Restore a single recorded `ref scan` for a library scan, including mid-flight
   suppression and the post-scan list path. Do not weaken the tests to pass on zero
   argv. Keep the scan-receipt observer; fix invocation or fake-bob recording.
3. Treat other known Mac CI failures as recorded owners (`bob-cli-9m` compile, and any
   still-open XCTest baseline). Do not expand into ranking-perf flakes.
4. Typecheck a scratch copy on mac; do not use Bryan's checkout or `just install`.

Acceptance: the three named tests pass on mac. macOS CI on the landed SHA is green aside
from other recorded task owners. The lander confirms the GitHub Actions test job; use
`/sase_monitor` to wait.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a fifth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and CI state, including whether
another epic changed highlights sidecar render, Pomodoro naming, task-status-hooks
relocation, or RefsLibrary scan. Run appropriate integrated checks once and document
genuine preexisting failures with their existing task owners. Require real evidence
before closing each task:

- `ba`: fixtures plus the two live quote repros no longer drop the `# ` line (read-only
  vault, or a documented post-sync dry-run). Honor `highlight_id_on_quote_fix`.
- `4r`: setext-preamble fixture does not render the marker as an annotation; recount
  live leaking notes.
- `bb`: isolated link-form capture keeps `DEEP WORK`.
- `2l`: isolated hooks/cycler repro leaves one struck link; both repos covered.
- `b7`: the three RefsLibraryTests pass on mac, and macOS CI on the landed SHA is green
  aside from other recorded owners. Use `/sase_monitor` for the wait.

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
   candidates include `bob-cli-94` for the review-walk settle index; `bob-cli-95` and
   `bob-cli-79` for loud write refusals; `bob-cli-b8` for live identity repair; flakes
   `bob-cli-3w`, `bob-cli-5l`, `bob-cli-b9`, `bob-cli-4k`, and `bob-cli-8q`; Athena
   `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x`. This is guidance, not a
   preassigned batch, and concurrent work may obsolete it.
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
