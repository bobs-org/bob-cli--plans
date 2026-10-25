---
tier: tale
size: medium
title: Narrow the Medium URL alias and finish backlog batch 7 landing
goal:
  Preserve distinct URL resources, verify batch 7 against current sources and Mac CI,
  and complete its epic closeout and backlog continuation.
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5
create_time: 2026-10-24 13:33:34
status: done
---

- **PARENT:**
  [202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.md)

# Narrow the Medium URL alias and finish backlog batch 7 landing

## Scope and ownership

Finish the remaining work of epic `bob-cli-an.6.6.6.5.6.5`, then perform its closeout in
this same tale. This is one bounded Rust correction plus acceptance and landing work; it
needs no additional implementation phases. There is no tale land agent and no automatic
return to the interrupted lander.

Read the epic and its two landing notes, all four phase beads and their notes, and
`plan:202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md` through
the audited SASE commands. That epic has no reviewer DECISIONS. Its five assigned tasks
remain `bob-cli-7u`, `bob-cli-86`, `bob-cli-2t`, `bob-cli-bg`, and `bob-cli-bh`; do not
absorb another backlog assignment into this batch.

At this audit, bob-cli HEAD and fetched origin are `2da7bbef`; the opened
bob-mac-capture repo is `c5d560c5492bf41215bf33213c5b6b51ffeabefa`. Use `/sase_repo` to
open linked repositories and the plans sidecar, and read their `AGENTS.md`. Never use
paths from another agent's numbered workspace. Use `sase artifact read` for plan or
evidence artifacts. No live vault changes, release version edits, or manual releases are
part of this work.

## 1. Restrict the reserved-at-sign URL identity exception

The approved URL phase explicitly required a narrow Medium exception: `@` is reserved
and literal validity does not establish universal equivalence. However, `f2524e80` added
`normalize_path_escapes(path)` with no host input and
`if decoded == b'@' || is_unreserved(decoded)`. Consequently it merges
`https://example.com/a%40b` with `https://example.com/a@b` on every host. The lander
compiled the helper and its supporting functions verbatim from current source and
confirmed both paths normalize to `/a@b`. This is an epic-caused defect, not a follow-up
task.

In `src/native/highlights_ref/clip_url.rs`, retain generic unreserved-byte decoding and
uppercase spelling of retained escapes. Apply the verified `%40` alias only to the
leading author sigil in the first path segment of the exact normalized `medium.com`
host, preserving the existing `www.` normalization. Do not extend it to arbitrary hosts,
suffix lookalikes, Medium subdomains, or at-signs elsewhere in a Medium path without
primary evidence. Keep the stored source URL and fetch URL unchanged. Correct the helper
documentation so it states the implemented boundary.

Add regression coverage that fails on `2da7bbef`: non-Medium `%40` versus `@` stay
distinct, a lookalike host stays distinct, and non-author occurrences stay escaped. Keep
the reported Medium author pair equivalent. Preserve the current unreserved,
escape-case, non-ASCII, malformed-escape, no-recursive-decode, and encoded `/ ? # %`
tests. Exercise at least one shared consumer so generic reserved paths cannot acquire a
false duplicate diagnostic/refusal. Existing Medium tests cover library identity,
`ref find` returning both twins with `duplicate_identity`, create dedupe, and collection
overlap; retain those assertions.

## 2. Recheck integration and the other phase acceptance

The lander read the actual three CLI commits `e3754c0d`, `f2524e80`, and `ae993b5b`, and
the Mac commit `c5d560c`. It also read every phase note and all five task reports. The
following are evidence and focused recheck targets, not new implementation assignments:

- Gist extraction in `scripts/web_clip/{snapshot.js,web_clip_adapter.py}` gets the owner
  from the final Gist URL, chooses a Markdown H1 then a readable filename, and applies
  explicit overrides afterward. Non-Gist numeric authors remain valid.
  `gist_karpathy.html` and `gist_multiple_no_h1.html` exercise browser facts;
  `clip_adapter.rs` tests Rust metadata/provenance decoding. Run
  `just check-web-clip-adapter` and require the browser fixture portion to execute, not
  merely report a no-browser skip. Run the focused Rust contract test.
- Close-summary commit `ae993b5b` stores a section-relative offset in
  `PomodoroCloseSummary` and the serde-skipped JSON-planner field; all three close call
  paths refresh the headline after staged edits and reuse its index for block
  references. Run `cargo test --test cli capture::pomodoro_close::`, covering the
  no-shift baseline, multiple inserted log lines, identical repeated names, unnamed
  sessions, dry-run/commit equivalence, and `task_line` equality. Do not alter close
  writes, time rounding, derived homes, or public field names.
- Mac `c5d560c` moves `HighlightsSaveKeyEvent.init(NSEvent)` into an extension,
  restoring its memberwise initializer. It condenses `docs/refs.md`, retaining save-scan
  defaults, permissions, limitations, and keyboard contracts. Phase .4 recorded
  portable-check (1587 tests) and the mounted docs gate (447 tests) as passing, but
  could not reach the Mac for scratch compilation.

Post-start changes reviewed so far are `2eb98129` (annotation-section docs and
cross-surface tests) and `2da7bbef` (theme completion and catalog cache). They do not
duplicate the shared URL-key normalization or change the close writer/headline locator.
The Gist and close outputs still feed the existing consumers. Re-fetch and review any
newer commits in bob-cli and the opened Mac repository, including base-branch changes if
the workflow has moved to a PR. Integrate any actual new conflict with this batch's
behavior, without taking over active `bc`/`bd` work.

### External Mac acceptance prerequisite

The lander independently inspected Mac CI
`https://github.com/bobs-org/bob-mac-capture/actions/runs/40113349760` on exact SHA
`c5d560c5492bf41215bf33213c5b6b51ffeabefa`. Lint and Build pass. Test fails before
XCTest because `RefsPanelModelTests.swift:2402` calls
`waitForModel { harness.library.snapshotRefreshGeneration > generation }` without
binding or ignoring the helper's one model argument. Blame is standalone `72cd721e`,
predating this epic; this is distinct from the fixed memberwise-init error. The new
error is owned by ready task **`bob-cli-bi`**, with `bob-cli-bg depends on bob-cli-bi`.
Evidence is `file:explicit:ba6ede72ea8315b6c596fe2a`.

Do not duplicate or implement `bi` in this tale. Read its current status and consume its
independently landed fix/CI result once available. If it is still running, use
`/sase_monitor` for a bounded status/CI wait with a concrete resume instruction; leave
this epic and `bg` open until the acceptance evidence exists. If triage declines or
otherwise blocks that prerequisite, record the blocker on this epic and use the
supported questions/monitor path to resolve the dependency, not a false successful close
or another identical plan. This is a wait on external work, never on the tale's own
commit.

Require the save-key XCTest suite to execute successfully on a Mac SHA containing
`c5d560c` and the external fix, and inspect the rest of that CI run rather than equating
a green Build with passing tests. On the opened sources rerun the shared
`just docs-check` with `BOB_CLI_DIR` and `BOB_DOCS_BOB_MAC_CAPTURE_DIR` explicitly set.
Read docs memory before changing documentation. Any additional Swift edits would require
portable-check and best-effort Mac scratch `bundle.sh` plus `just typecheck-tests`,
following the repo's instructions; no live install.

## 3. Verify the completed tree and prepare continuation

Run focused URL/library/collection regressions and the checks above, then the canonical
**`just check`** after the Rust repair. Use `/sase_monitor` for long commands. **Never
run `just check-full`.** Classify new failures from current evidence; historical
baseline notes are not blanket permission to ignore failures. The previous check
failures are the completion registry owned by `bob-cli-bc`, the two continuation nodes
owned by `bob-cli-bd`, and the independently tracked parallel executable-stub flake
`bob-cli-5l`. Do not weaken assertions to pass.

All lander follow-up triage was completed before this proposal, with outcomes on the
epic. Carry those outcomes into its final close note:

1. Phase .1 note #4, .2 note #1, and .3 note #1: `themes rename:old/new` coverage
   failure corroborated on causal active epic `bob-cli-bc`/phase `.6`; no new task.
   Closed `4j` concerns a different historical `--audio` argument.
2. The same three proposal entries: both `continuation_scan_*` failures corroborated on
   causal active epic `bob-cli-bd`/phase `.1`; no new task.
3. Phase .1 note #4 only: curl ETXTBSY corroborated as independent report #10 on
   `bob-cli-5l`, explicitly relaying the phase evidence; duplicate declined.
4. Phase .4 has no follow-up proposals. Its missing Mac acceptance led the lander to the
   distinct compile failure now owned by `bob-cli-bi`, filed with the audited skill,
   sized small, linked to `bg`, and marked ready.

For any genuinely new unrelated discovery, use `/sase_new_task` and append its outcome;
defects caused by this epic remain this epic's work.

Prepare the next bounded backlog proposal required by the continuation contract below,
using fresh all-status enumeration and current ranking. Do not propose it until the
current batch closeout is complete. The campaign must not silently end when this batch
lands. Preserve the original epic's explicit repository-obligation instructions when
handing off the continuation, with any required commit after the closeout, never as a
prerequisite to closing this successfully verified work.

## 4. Final step: finish this epic's closeout and its permitted parent traversal

This step belongs to the tale coder. Closeout must happen in the same turn as the final
code and must never wait for this tale's own SHA, push, or CI result. Do not defer it to
a nonexistent tale land agent. External `bi`/Mac evidence is the only independent
prerequisite described above.

1. Close only independently verified target tasks `bob-cli-7u`, `bob-cli-86`,
   `bob-cli-2t`, `bob-cli-bg`, and `bob-cli-bh`, each with
   `sase bead close <id> --note "<concrete evidence>"`. Re-read status/notes first; do
   not interfere with another worker's claim.
2. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5`. Resolve every entry by wiring
   it, privatizing it, adding an appropriate non-test pragma, or deleting it; re-key
   only to a still-open later bead that demonstrably needs it. The lander found no
   entries, but recheck. Review all descendants and linked-plan readiness before
   closing.
3. Run
   `sase bead close bob-cli-an.6.6.6.5.6.5 --note "<source/commit and child-note verification, narrow URL fix, integration review, exact test and Mac CI evidence, all follow-up dispositions>"`.
   A stale exemption rejection requires cleanup and another normal close. Never use
   `--force` to make a successful nested landing advance. If genuinely
   canceled/superseded work exists, record that deliberate resolution and reason rather
   than masking it.
4. Run `just symvision` if available; this checkout currently has no recipe, so record
   that fact if it remains absent. Set `status: done` in the frontmatter of
   `202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md` within the
   opened plans sidecar. Read it through its `plan:` artifact first.
5. Re-read this epic's parent link. Its concrete parent is plan bead
   **`bob-cli-an.6.6.6.5.6`**, already closed at the initial audit. Review that parent's
   previous landing notes, all descendants and their notes, its linked plan
   `plan:202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md`,
   and drift since its landing. Rerun descendant and linked-plan readiness checks. If
   still complete, retire every entry from
   `sase bead epic-symbols bob-cli-an.6.6.6.5.6`, close it normally with the rechecked
   evidence (an already-closed close is a no-op; append an audit note), run available
   symvision, and ensure its plan is done.
6. Repeat through directly parented plan ancestors `bob-cli-an.6.6.6.5`,
   `bob-cli-an.6.6.6`, `bob-cli-an.6.6`, `bob-cli-an.6`, and `bob-cli-an`, using fresh
   linked-plan and descendant checks for each. Stop at the first incomplete or ambiguous
   parent, append a concrete blocker note there, and report it. If a parent is now a
   phase, verify the child completed its scope and close only that phase, leaving its
   containing epic for its waiting lander.
7. Complete all repository obligations and submit the prepared bounded successor through
   `/sase_plan`, obeying the inherited continuation contract below. Close the present
   epic before any commit needed solely for that handoff. Wait for the proposal command
   itself to exit. If a fresh complete enumeration instead proves zero unresolved tasks,
   finish normally through `/sase_final`.

## Inherited mandatory backlog continuation contract

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
