---
tier: tale
title: Repair project-note move integration and finish bob-cli-gl.5.1
goal:
  Project-note moves preserve tracked identities, offer eligibility, blocking and final
  reports; close the verified epic and its parent phase.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-gl.5.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gl.5.1
status: done
---

- **PARENT:**
  [202611/capture_move_execution.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_move_execution.md)
- **BEAD:**
  [bob-cli-gl.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gl/bob-cli-gl.5.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-gl.5.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-gl.5.1.land.md)
- **COMMITS:**
  - [b738d8f](https://github.com/bobs-org/bob-cli/commit/b738d8f23b21ac855d83745bdd415af52dfbdc98)
    — feat(capture): repair project-note move landing integration

# Repair project-note move integration and finish bob-cli-gl.5.1

Complete the remaining work in epic **bob-cli-gl.5.1**, then finish its landing and
close only its parent phase **bob-cli-gl.5**. This is one coding agent's bounded
planner/tracker repair, not another multiphase epic. The tale coder owns closeout; there
is no later land agent for this tale. Do not wait for this work's commit SHA, push, or
CI before closing. The host commits after the final declaration.

Read `sase bead read bob-cli-gl.5.1 -r "Need landing audit and triage"` and its three
closed children, plus these audited artifacts:

- `plan:202611/capture_move_execution.md` — the child epic being completed.
- `plan:202611/capture_move_bullets.md` — the governing contract and MV1–MV17.

The original decisions remain binding: keep the current grammar revision (9 on the
audited tree), retain the additive unversioned capture envelope, and edit no memory,
release version, or CHANGELOG. No new capture syntax or Mac work is required. Use the
shared move engine and existing-task link gesture; keep one move snapshot/plan per item,
atomic snapshot guards and rollback, stable destination-before-source writes, and the
no-move path's lack of new vault-wide scans.

## What the landing already verified

The audit read all child notes, governing plans, and commits `6650faa3`, `14a57732`, and
`ddb0a73f`, along with the current renderer, move adapter/engine, batch writer, offer
settlement, tracker, and CLI tests. Basic ordered moves, full child-block preservation,
collisions, typed refusals, staged chains, freshness, dependency/link repairs, move
JSON/human output, and the hint are implemented. The interim refusal is gone. The
current CLI `project_note` filter passes 79 tests; the focused catalog protocol-1 test
also passes. Existing MV coverage should be preserved, not rewritten wholesale.

The baseline is `ddb0a73f`, then equal to `origin/master`. The integration audit
reviewed nine intervening commits: transplant extraction `aac3947c`, target-run
execution `a7c4b07f`, note-pull settlement `4db3200c`, target-run editor `6a5b69b0`,
merge preparation `adf20825`, attachment/catalog fixes `dd90f8c7`, merge preparation
cleanup `672954d4`, target-run activation `c341649a`, and task split `734214e9`. Recheck
later drift before editing. In particular, preserve the shared transplant path,
note-pull's earliest-offer ownership and settled ordinals, and target-run source ranges.
A mixed note-pull/project-move dry run succeeds without writes; a
target-run/project-move batch exposes the tracker defect below.

## Reproductions and required repairs

Use disposable test vaults, fixed `BOB_NOW=2026-09-20 14:31:07`, `TZ=UTC0`, an explicit
`BOB_DAY_FILE`, and an isolated config. The tests' existing `write_toggle_task_settings`
helper is essential for project-blocking probes: it makes `?` a recognized TODO status.
Do not test against the live vault.

### 1. Remap tracked task blocks during project moves

Given area `cash.md` with open roots `^a` and `^b`, this draft currently exits 101 in
debug builds, including dry runs:

```text
@cash+a @cash+b A shared note

Project @cash^x@
- ^a
```

The panic is `capture/task_blocks.rs:163`, `tracked task headline lost without a ref`.
`plan.rs` calls `TaskBlockTracker::track_item` after the move removed a tracked root
from its source. Only the later inbox-route path currently calls `remap_routed_tasks`;
project moves never supply equivalent relocation facts.

Carry typed move identities/landings from project planning into the batch loop and remap
tracked roots before ordinary same-note forwarding can declare them lost. Preserve
first-touch roles and cumulative block reporting. Design the ordering so remapped
entries have valid pre/post handling; merely calling the existing inbox remap before
`track_item` with the old snapshot is insufficient. Reuse/generalize the relocation
representation where appropriate instead of weakening the assertion or dropping reports.
Cover previously touched descendants carried inside the moved block, chained moves,
moves followed by further edits, and untouched tracked tasks. The real batch must
preserve the shared note on both tasks, move `a` once, and report its final destination.
Dry-run JSON must agree with real JSON after normalizing only `dry_run`, and the dry run
must write nothing.

Primary files: `capture/project_note.rs`, `capture/batch.rs` or the internal planned
item type, `capture/plan.rs`, `capture/task_blocks.rs`, and existing tracker/CLI tests.

### 2. Use exact live ledger identities for follows and offers

Two independent ad hoc checks in `capture/project_note.rs` disagree with the ledger: the
pre-move offer filter uses `peek_text` plus whole-note `contains`, and
`followed_live_link` accepts any substring inside an open entry. Replace them with a
shared staged-aware query built on the existing dedicated Task Link parser (for example
`capture_active_tasks::ledger_from_contents` or the Today/queued-link engine), with
exact route/block identity and established alias/extension handling. Load the day only
when needed; a never-loaded disk day must have the same answer as the same day already
staged by an earlier item.

Confirmed failures:

- Open WORK contains `  - Discuss [[cash#^a]]`. `Project @cash^x@#admin` with `- :a`
  reports `WORK link follows` and leaves `a` Ready and unlinked. A prose mention is
  inert; create the requested dedicated ADMIN link.
- Open WORK links `[[cash#^ab]]`. Moving `- :a` and `- ^ab` into the same project
  falsely treats `a` as already linked because `cash_x#^a` prefixes `cash_x#^ab`.
- A closed OLD entry links `[[cash#^a]]`. With earlier `Warm @cash:warm` in the batch,
  Offer-form `Project @cash:x@` plus `- ^a` suppresses the offer. Without that earlier
  item it offers `a`: `peek_text` has not loaded the day yet.
- A first-item Offer move whose genuine live link follows can be removed only in the net
  pass, leaving offer numbering/fold accounting derived from the wrong initial candidate
  set. Freeze the correct candidates and folds up front while retaining the final net
  pass.

Test historical, prose, struck, fenced, deeper nested and exact dedicated links,
including moved prefix IDs. Genuine live links stay under their original open Pomodoro,
are not duplicated or relocated, and count as `already_planned`.

### 3. Fix hidden eligibility without changing movability

The offer filter indexes `parent_lines.get(resolved.root.line)`, but `MovableRoot` lines
are one-based. A final parent row `- [ ] #task Secret #hide ^a` is offered and `-p all`
links it. Conversely, visible `^a` immediately before hidden `^b` is wrongly excluded.
Read the actual root metadata/line with the correct coordinate system. Hidden tasks
still move when explicitly named; they are excluded from offers and count in the
existing hidden fold. Test hidden final/adjacent/child-line cases and all/none/subset
picks. Avoid imposing a daily-note requirement when there are no actual links or offer
destinations to resolve.

### 4. Preserve inherited blocking at the new project boundary

With recognized custom statuses, use this parent:

```markdown
---
type: "[[project]]"
status: wip
---

- [?] #task Parent #prj #hide [scheduled:: 2026-09-25] ^prj
- [?] #task Alpha #prj_is_blocked ^a
```

Direct `@cash:a` correctly reports `project_schedule`. But `Project @cash^x@` plus
`- :a` currently makes `a` Next and links it: the rendered child `^prj` is Ready without
the inherited blocking proxy, so the existing guard sees an unblocked destination. This
contradicts the approved blocked-parent interaction.

Read `decisions:subprojects-inherit-project-blocking` and
`decisions:capture-link-drops-blocking-deps` through memory. Use the existing project
blocking machinery against staged parents to establish the new project's inherited state
before links/offers. Keep the parent's reasons on its own `^prj`; do not copy
dates/prerequisites into the child or run vault-wide reconcile. The child `^prj` and
open residents must carry the appropriate Blocked/tag proxy, and moved colon tasks must
stay unlinked. Preserve `needs_reconcile` and truthful guard/report details. Do not
discard nested guard consumption, promotion or dependency-drop details when composing
the project result. Test parent schedule and dependency blocking, a deeper ancestor, a
staged parent, and local-plus-project dependency cases. Local-only prerequisite dropping
(MV16) must continue to link safely.

### 5. Resolve reports after all relevant mutations

Given blocked `a` with a managed Depends-On child pointing to parent sibling `b`,
followed by Ready `c`, draft `Project @cash^x@ / - :a / - ^c` drops `a`'s dependency
child and links `a`. The final destination places `c` on line 14, but
`moved_tasks[c].line` still reports 15. Current code refreshes only the row actively
linked, not other moved rows affected by its child edits.

Re-resolve every moved root by stable identity after the item's links and at the
appropriate final batch stage when later edits shift those rows. Refresh followed entry
lines/names after insertions or destination-name merges too. Preserve authored sigil,
source coordinates, original UTF-8 draft ranges, previous status and move history; never
derive identity from title text or rebuild moved blocks from offers. Keep selected offer
rows, task links, block trackers and moved rows consistent with the final staged state.
For a later chained move, preserve the per-item move history contract while ensuring
final block reports identify the actual final destination. Test dependency-child
removal, schedule-log changes, later same-note edits, mixed new/moved named links and a
followed WORK entry shifted by creating ADMIN above it.

## Verification and already-triaged failures

Add focused behavior regressions to the existing project-note/tracker suites, using the
shared fixtures and exact output assertions for the reproduced bugs. Preserve MV1–MV17,
existing-task link/dependency-drop, offer settlement, inbox routing and target-run
coverage. Confirm no new full-vault work on a no-move draft and one move engine call per
moving item. Existing warm-fixture timing is recorded in gl.5.1.3 note #2 (50 notes;
72.64 ms median); repeat a comparable measurement only if the fixes materially change
scanning.

Run formatting and focused checks, then **`just check`** (`sase tool run check` records
it). Do not run `just check-full`. Run `just docs-check` for the epic's final contract
gate; if help/docs change, read docs memory and regenerate the affected reference
through repository conventions. Use `/sase_monitor` for commands that need a handoff,
with a concrete continuation; do not end while a foreground command or handoff-staging
command is still running.

The lander completed `/sase_new_task` triage before proposing this tale. Carry these
outcomes into the close note, rechecking current state rather than opening duplicates:

- gl.5.1.2 notes #1–2: catalog 69-versus-67 failures were already routed to
  **bob-cli-de** and repaired by `dd90f8c7` in both assertions. The lander recorded
  resolution evidence and a passing focused CLI test; no new task or stale reproduction
  +1.
- gl.5.1.3 note #1: `just docs-check` still passes lint then fails generated drift on
  `glossary/task-merge.md` lacking frontmatter. Independently corroborated on active
  causal owner **bob-cli-go**; no new task, and no memory edits here.
- gl.5.1.3 note #3: unused `prepare_task_merge` re-export is resolved by `672954d4`; it
  no longer appears in clippy. Declined a new task.
- Landing `just check` run **21a7758643f1dbbd1810737832a1c896** failed in clippy because
  `734214e9` added five task-split/command-shell lint errors: `task_move/command.rs:354`
  useless_format; `task_split/command.rs:236` and `task_split/plan.rs:347`
  collapsible_if; `task_split/plan.rs:1063` bool_comparison; `task_split/plan.rs:1170`
  collapsible_str_replace. The lander recorded exact evidence on **bob-cli-go**, the
  active causal epic, after duplicate searches. These are not remaining move-epic work.
  Do not broaden this tale to repair task split; document unchanged failures if they
  persist.

New failures caused by these repairs must be fixed. For unrelated failures, use
`/sase_new_task` and record every outcome. Existing base failures may be documented
without falsely claiming a green gate or leaving completed move work open solely for the
unrelated epic.

## Final step: complete this landing in the coding turn

After implementing and verifying the repairs, inspect new commits since `ddb0a73f` and
integrate anything relevant. Read the latest epic/child notes, confirm all descendants
and linked plans are ready, and include this tale's completed scope in that readiness
check. If this tale has its own descendant bead, close it normally once its
implementation and verification are complete so it does not block the epic; do not wait
for the host's commit or force through unfinished work.

1. Run `sase bead epic-symbols bob-cli-gl.5.1`. Resolve each entry by wiring,
   privatizing, deleting or a justified non-test pragma; re-key only to a genuinely
   still-open later owner. There were no entries at the audit baseline. Run the same
   check for parent phase **bob-cli-gl.5** before its close.
2. Close the epic normally with
   `sase bead close bob-cli-gl.5.1 --note "<verified move fixes, intervening integration, exact checks, and every follow-up outcome above>"`.
   If rejected, finish the named readiness or symbol cleanup and retry. Never use
   `--force` merely to succeed or to advance this successful nested landing.
3. Run `just symvision` if the recipe is available (none existed on the audited
   justfile); explicitly record absence otherwise. Open the plans sidecar with
   `/sase_repo` before editing, use the path it prints, and set `status: done` in the
   frontmatter of **`plan:202611/capture_move_execution.md`** (the epic's PLAN path from
   `sase bead read`). Audit artifact reads with `sase artifact read`.
4. Re-read `sase bead read bob-cli-gl.5.1 -r "Need the parent link"` and
   `sase bead read bob-cli-gl.5 -r "Verify completed child fulfills parent phase"`. The
   parent is phase **bob-cli-gl.5**. Verify the full execution/link/offer/output scope
   is now satisfied, ensure its symbol list is clean, then close only that phase with
   `sase bead close bob-cli-gl.5 --note "<combined child implementation, integration and checks verified>"`.
   Leave containing epic **bob-cli-gl**, its outer plan status and Mac phase gl.7 to
   their already-waiting agents. If the parent is still incomplete, record the concrete
   blocker on gl.5 and report it instead of forcing closure.
5. Finish through `/sase_final`, declaring every changed repository (including the plans
   sidecar). Use completeness, not the visibility of a post-turn commit, to select the
   assigned bead action. Report the actual close/check outcomes.
