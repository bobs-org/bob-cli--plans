---
tier: epic
title: Obsidian cycler relation successors on managed Done
goal:
  Complete bob-cli-fs.8 with explicit relation-successor picks on every managed Obsidian
  Done gesture, fresh revalidation, correct notices and undo, and a verified deployed
  cycler build.
phases:
  - id: relation_core
    title: Relation policy, pure rule, and shared copy
    size: medium
    depends_on: []
    description:
      "relation_core: implement the cycler's cached successor-label policy, pure R1-R7
      planner for Done and parked events, copy helpers, and fixture expansion without
      enabling runtime offers."
  - id: offer_pipeline
    title: Integrate relation offers with recovery and selection
    size: medium
    depends_on:
      - relation_core
    description:
      "offer_pipeline: merge relation marks into the pure successor pipeline, preserve
      recoveries and reasons through deduplication, reuse selected writes and placement,
      and produce picker v3 models with complete vector coverage."
  - id: done_runtime
    title: Wire managed Done with fresh relation reads and receipts
    size: medium
    depends_on:
      - offer_pipeline
    description:
      "done_runtime: connect relationIndex v2 to every managed Done gesture with cheap
      gates, fresh revalidation, notice v3 fallbacks, receipt take-back, and behavioral
      wiring tests."
  - id: ship
    title: Document, validate, and deploy the cycler
    size: small
    depends_on:
      - done_runtime
    description:
      "ship: update the cycler manifest and reference, run the complete plugin gates,
      reproduce unrelated failures on the clean base, sync the cycler, and record
      live-smoke evidence or its availability limit."
proposed_by: bbugyi200.athena.bob-cli-fs.8
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fs.8
create_time: 2026-11-02 07:08:23
status: wip
---

- **PROMPT:**
  [prompts/202611/cycler_relation_successors.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/cycler_relation_successors.md)
- **PARENT:**
  [202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/starts_after_relation.md)

# Obsidian relation successors on managed Done

## Objective and ownership

Complete **bob-cli-fs.8**, the `cycler` phase of `plan:202611/starts_after_relation.md`.
When a managed Obsidian gesture completes T, offer eligible S tasks whose accepted LINKS
entries use a configured successor label to point at T. The picker explains the relation
and the Done event. Only explicit picks write or link S. Preserve existing
dependency/project recovery, destination selection, walk behavior, and same-day reopen
receipts.

This is a child epic of the assigned phase. Its phases are serial because they share the
successor planner, generated entrypoint, and tests. Each is a bounded coding task; the
child epic's land agent verifies the complete result and closes **bob-cli-fs.8**. It
must never close **bob-cli-fs** or any other ancestor. Creating this plan does not
complete the assigned phase.

Implementation lives in the linked **bob-plugins** repository. Each worker must use
`/sase_repo` and `sase repo open bob-plugins -r "<specific reason>"`, then read that
checkout's `AGENTS.md`. Paths below are relative to bob-plugins unless identified as
bob-cli paths. Only task-status-cycler source, its generated build, fixtures/tests,
package test registration, manifest, and its documentation are in scope. Do not edit
deployed plugin files directly.

The accepted parent decisions are final: glossary changes were authorized only for the
three named terms and belong to the completed contract phase; a new decision record was
rejected. **This child epic edits no memory.** Do not change Rust capture,
navigation/ledger plugin implementations, macOS capture, chezmoi, release versions, or
the changelog. The later `bob-cli-fs.9` phase owns the public parked-successor API and
block-id-prompt unlink wiring. Here the pure engine supports parked events so that phase
can reuse it, but runtime activation is managed Done only.

Read the parent plan using `sase artifact read` and the assigned bead using
`sase bead read`; use audited memory reads for the relevant glossary and accepted
successor/recovery decisions. Read `docs.md` before documentation edits. Record
discovered unrelated work as `PROPOSED FOLLOW-UP:` notes on the worker's assigned phase;
consolidate relevant evidence onto bob-cli-fs.8 for the outer land agent. Do not create
task beads or set lifecycle status by hand.

## Verified starting point

- bob-cli-fs.8 was `in_progress`, with no previous work notes. No reopening is needed.
  `sase bead epic-symbols bob-cli-fs.8` reported no entries during planning.
- Prerequisites bob-cli-fs.1 and bob-cli-fs.6 are closed. The opened plugin checkout was
  clean at **7cc1735**, which provides nav picker/notice v3. **a3e3462** already
  provides the inverse pair and navigation's three-list relation-label policy. Recheck
  current history before implementation and retain newer work.
- The cycler remains **1.45.0**, has no `089d-successor-relation.js`, no relation
  planner/config loader, and no relation rows in its successor model. Do not redo
  navigation presentation or authoring.
- `085-successor-plan.js` has 993 lines and `087-successor-place.js` has 995. Fragments
  must remain at most 1000 lines. Extract small existing helpers if needed; register
  every new fragment in `src/fragments.json` and build `main.js` rather than
  hand-editing it. New policy/copy/runtime fragments may supplement `089d`.
- `planSuccessors()` calls `applyContext()` after computing existing recovery and
  project marks. Its idless early return currently recognizes keep-moving only.
  `089b-successor-verdicts.js` owns deduplication, selected edits, and row indexes.
  `085b-successor-offers.js` owns keys, ordering, and picker models.
- `130-plugin-references.js::finalizeClosedTasks` funnels managed closes to
  `137-plugin-successor-project.js::planAndApplySuccessorsNow`. That method has two
  early gates, one picker per batch, fresh re-planning, and shared guarded writes. `135`
  handles read/apply/notice helpers; `136` records and consumes reopen receipts.
  `089c::successorPlanRows` currently unifies dependency and keep-moving rows.
- Ledger-tools exposes `api.relationIndex` v2: `ready()`, `generation`,
  `incoming(path, blockId)`, and `outgoing(path, blockIdOrLine)`. Edges carry
  `source: {path, blockId, line, text, status}`, `target` including `resolved`, `label`,
  `kind`, and relation location data. It overlays editor text. Treat indexes as
  discovery, then verify authoritative source/target text. It supports idless sources
  through line identity; preserve the API's line conventions explicitly.
- bob-cli's `tests/fixtures/relation_successor/vectors.json` is canonical. It has **12
  config, 4 key, 25 rule, and 31 copy** rows plus a UI-only coverage list. The rule file
  is a base scene plus compact overrides/cases, not fully expanded notes per vector.
  Navigation already carries a byte-identical copy and pure copy helpers in
  `206b-successor-relation-copy.js`. Deployed plugins must not import each other; port
  the needed policy/copy behavior into cycler-owned fragments.

## Binding behavior

Use `docs/task-dependencies.md` sections 12.2–12.4, 12.13, 12.18–12.19 and
`docs/task-relations.md` section 20 in bob-cli, together with the parent plan, as the
contract. The following details must survive every implementation phase.

**Policy and events.** `relations.successor_labels` defaults to an empty list. Normalize
labels with the existing relation-label rules. Invalid entries warn and skip. A key
appearing in multiple cancel/complete/successor lists is removed from all effects, using
the existing exact warnings. Cache configuration by path/stat like `loadKeepMoving`,
including a reset/injection seam for tests. Missing YAML support or unreadable config
never throws from a gesture. `plan.link_unblocked: false`, empty labels, cancel, or nav
`api.successors.version < 3` disables relation planning. Do not couple it to
`plan.keep_moving` or `[id::]`.

The engine accepts explicit `(T, done)` and `(T, parked)` events. A parked event needs a
baseline live link and post-gesture Pending status with no remaining live link. Done
events must still refer to a Done T at apply time. Runtime here derives events only from
managed Done identities; raw edits, cancel, release, reopen, sync, reconcile,
configuration edits, and relation authoring never synthesize an event.

**Discovery and R1–R7.** Follow accepted LINKS edges S → T with configured labels; never
read generated LINKED FROM as the source of truth. Exclude self-edges, honor
same-note/basename/path resolution, and use fresh editor overlays. First match wins:

1. S is not an open `#task`, is under `done/`, or is in the previous daily note: skip
   without reporting.
2. S was parked by this gesture or was live-linked on its original baseline: held
   `already_planned`, silent in UI.
3. S is Pending: held `already_started`.
4. S is Blocked, has an open own prerequisite, has a future schedule, or is blocked by
   its project after this gesture's recovery: held `blocked` with the existing
   waits/schedule/project details. A source with dependency fields and no reliable
   dependency snapshot stays held; never treat unavailable data as an empty pool.
5. Whole-token `#hide`: held `hidden`.
6. Another resolved successor-label target T′ of S is not released: held `waits_for`.
   Released means Done, Cancelled, or Pending without a post-gesture live link. All
   valid events of this gesture count as released; unresolved other targets do not
   block. Read only the relevant target notes, not the whole vault.
7. Ready or Next S is offered in the group `(T, label key)`.

Compute the rule against the effective post-recovery state. A relation never itself
recovers S or stamps it unblocked. Genuine dependency/project recovery still occurs even
if a relation wins the final offer group or the user picks nothing.

**Merge and writes.** Deduplication priority is
`hand_off > relation > project > project_finished > keep_moving`; preserve all reasons
and recoveries. Relation groups sort by T's anchor ledger position, then T's path and
line, with a deterministic label-key tie break. Rows sort Next before Ready, then path
and line. One S reached by several Ts appears once with all triggering identities and
the contract suffix. Preserve stable block/digest keys, numbered offers, destination
overrides, type routing, and shared placement row indexes.

Selected rows reuse the hand-off writer: Ready becomes Next, Next stays Next; mint a
missing block ID, stamp fresh, clear keeps/unblocked, and insert the ordinary Task Link.
Relation-only rows that were not selected write no fields, status, IDs, or placeholders.
Existing recovery may still write its own legitimate changes. Use the existing
slot/closing/inherit/implicit placement rules; parked engine inputs also support a
removed-link slot and a closing anchor for the later unlink phase.

**Model and presentation.** Surviving relation rows go only in `relation_successors`,
with trigger/group `relation`, `offer.after` identifying T, `unblocked_by`, and
`relation: {label, key, event, after}`. Deduplicated hand-off rows keep their relation
second reason. Add `relation_held` and the checked/unavailable signal only when
applicable; disabled and edge-free gestures omit relation fields. No schema version
changes.

The picker receives v3 relation groups with `header`, `caption`, `tag`, `foldText`,
`after`, `event`, `label`, folded counts, and row `lane`, `suffix`, and `reason`. Title:
`Starts after “T”`; caption: `“T” is done`; tag: `your links · not dependents`. Use
configured spelling and the shared 48-character clean-text truncation rules.
All-relation pickers use the relation title; mixed pickers retain their existing title.
Navigation already supplies neutral `link-2` rendering. Never invent status coloring or
new copy.

Plain notices use `🪢 Next in DEST: S · label “T”` or the plural linked form. Mixed
dependency notices keep the existing forms and count relation picks. Held rows are shown
only if no picker opened, and already-planned rows are always silent. Empty selection
alone is silence. Missing/not-ready relationIndex after the feature gate produces no
relation rows, reports unavailable, and presents
`🪢 Couldn't check label links · nothing offered` (multiple labels: `relation links`).
The close still succeeds. Send relation-bearing models to nav only with notice API v3;
otherwise show the correct plain text. The walk's return/toast path must use the same
copy and retain its gesture lock until the picker settles, including Esc.

## Phase relation_core

Create `plugins/task-status-cycler/src/089d-successor-relation.js`, plus focused
policy/copy fragments where needed for the line cap. Export testable pure helpers
through `210-exports.js`. Add `loadRelationSuccessorLabels()` with the three-list
policy; use Obsidian's YAML parser via an explicit import/injection and update the test
harness stub as needed. Follow the landed navigation policy and warnings, without
coupling to private functions of another deployed plugin.

Define a stable internal boundary for `planSuccessorRelations(options)`: explicit
events, normalized candidate/outgoing edges, fresh note texts, original baseline and
current day, configuration, dependency availability, and effective recovery state in;
relation marks, held rows, group metadata, and availability out. It does no I/O and does
not apply writes. The later adapter discovers candidates, and the pipeline supplies
recoveries. Support both event kinds now. Keep the option/output shape documented
adjacent to the function for the next two phases.

Copy the canonical JSON to
`plugins/task-status-cycler/scripts/fixtures/relation_successor/vectors.json`
byte-for-byte. Add and register
`scripts/test-task-status-cycler-relation-successors.cjs`, using the existing cycler
harness. Build a reusable scene expander from `base` plus `input`/`cases`: generate real
note text, statuses, LINKS entries, ledger text, and explicit events. Build a fake
relationIndex from parsed scene notes with the real public edge shape. Never return
expected verdicts from the fake index or encode SA-ID-specific production branches. Add
copy parity for all 31 cases and config/key parity for all entries. Test the independent
rule classifier now; the next phase finishes full merged-rule assertions for
dependency/project/batch context.

Acceptance: policy/cache tests and pure rule/copy tests pass; disabled runtime behavior
is unchanged; registered fragments/build are current and within the cap.

## Phase offer_pipeline

Integrate the relation classifier into `085::applyContext`. Refactor a small existing
helper if necessary to keep 085 under 1000 lines. Fix its idless early return so
relation candidates and unavailable checks can reach the common result even without
dependency IDs, project candidates, or keep-moving marks. Feed the classifier the
post-recovery state while retaining exact preimage lines for the write guard.

Update `089b` deduplication, row routing, and picked/unpicked handling. Merge relation
metadata even when hand-off wins; retain recovery edits when relation wins. Extend
`successorPlanRows` and every consumer relying on `unblocked`/`keep_moving` alone,
including recovery counts, failure propagation, placement indexes, and model copies, so
moving a recovered resident into `relation_successors` loses no behavior. Existing
legacy model shapes must still work.

Extend `085b` ordering and picker grouping by target identity plus label key, not by
project alone. Carry group metadata and held counts to the v3 model and attach second
reasons to non-relation rows. Reuse existing selected edit/placement helpers and D7
routing; relation-only unselected tasks must not fall through recovery writers.

Complete execution of all 25 rule vectors, including every `cases` branch, using the
scene expander. Test SA14, SA15, and SA18 through the merged production planner, not
only the classifier. For sequence vectors, materialize before/after text and valid
gesture events at the harness boundary: no retroactive event after authoring, and later
linking/relinking invalidates the relevant row. Do not add a capture batch executor to
this plugin. Retain the capture-specific warning expectation as cross-surface fixture
context; verify the cycler-applicable no-replay result and cover the warning copy
through shared copy tests. Explicitly report that distinction in the test adapter, with
no silent vector skipping.

Add beyond-vector cases for multiple relation groups, Next/Ready ordering, missing
source block IDs, multiple reasons, real dependency recovery plus relation selection,
and no-write Esc behavior. Validate slot, closing, implicit, and overridden destinations
and no placeholder creation from a display-only plan.

Acceptance: all applicable rule projections, config/key and all copy vectors pass; the
existing successor, offer, keep-moving, project pull-in, and theme-write tests remain
green. Runtime is still inactive until the next phase's adapter is wired.

## Phase done_runtime

Add a cycler-owned relationIndex adapter and connect it to `137`. Keep new I/O helpers
outside capped fragments. Resolve configuration and compatibility first, then use
in-memory incoming lookups for eligible T identities. Thread candidates or unavailable
status through both existing quiet gates. With a ready index and no matching incoming
edge, perform **zero extra vault reads**. The feature must work when neither T nor S has
`[id::]`, and when LINKED FROM generation is disabled.

Only after a positive lookup, read candidate source notes through
`readSuccessorNoteText` (open editors first), verify the accepted edge, and inspect
outgoing successor edges and relevant other target notes for R6. Reuse the same cache
and any existing dependency pool. Request dependency data only when candidates need it;
unavailable snapshots conservatively hold affected rows. Do not introduce a persistent
cache, a TTL scan, or a full-vault relation scan. Keep unknown/read-failed state
distinct from a genuinely missing target.

Wire a second relation discovery/classification pass after picks using fresh config,
fresh source/target notes, the current index, and fresh day text. Maintain the original
gesture baseline for R2/anchors while separately checking newly live-linked S and
whether T still satisfies its event. Edge removal, changed label, source closure, new
blocks/links, lost target event, unreadable notes, or lost index readiness must refuse
the affected pick without substituting another or opening a second picker. Refresh the
affected-note set for new relevant outgoing targets, not just old offers.

Ensure every managed Done route reaches this pass: Ctrl+Enter task, Task Link,
transclusion/tree, Pomodoro embedded completion, Alt+] into Done, complete-label commits
via the existing finalize API, and PRE/POST walk completion. Preserve one picker and one
walk advance per gesture. Cancel still only recovers and never consults relationIndex.
Do not install a checkbox or vault watcher.

Use relation copy in `successorNoticeText`, `presentSuccessorModel`, and the returned
model. Track whether a picker actually opened so held notices cannot duplicate its fold
lines. Feature-detect notice v3 separately from picker v3. Preserve dependency failures
and existing notices, including mixed picked rows.

Make `rememberSuccessorPass` include successful relation picks and restore untouched
links/status lines on same-day reopen. Retain existing stale/edited/next-day guards,
single-use receipts, and conservative ownership when multiple predecessors caused a row.
Never persist event or offered-state markers to Markdown.

Behavioral wiring tests use fake nav picker/notice APIs and observable reads/writes: v3
versus v2; missing/not-ready/malformed index; no-edge zero extra reads; editor-only edge
edits; both kill switches; no dependency IDs; cancellation; mixed groups; picker Esc;
the walk lock while a picker promise is pending; per-row stale refusals; notice
fallback; held-only output; and untouched versus edited receipt take-back. Cover
SA25–SA29 and SA34–SA36 as applicable. SA31–SA33 runtime unlink remains the later outer
phase's scope; the pure parked-event contract is already tested.

Acceptance: managed Done integration tests and all existing cycler successor/walk tests
pass. Ordinary closes still succeed when relation discovery or linking fails.

## Phase ship

Update `plugins/task-status-cycler/manifest.json` with one minor bump from the current
landed version and a description of relation offers and v3 compatibility. Read docs
memory and update `docs/task-status-cycler.md` with the configured standing-offer
behavior, Done gestures, opt-in picks, held/uncertainty output, compatibility gates, and
receipt behavior. Do not advertise the future unlink API as shipped. Regenerate the docs
inventory with `npm run docs:gen` if required by the manifest edit.

Run `npm run build`, the focused suites from earlier phases, `npm test`, and
`npm run docs:check`. Confirm canonical fixture byte equality, exact public copy,
fragment line caps, generated entrypoint currency, and only intended files changed. Use
`/sase_monitor` for long commands rather than ending with a live exec session.

Prerequisite bob-cli-fs.6 recorded clean-base failures in freshness-namespace
(bob-cli-et), keep-inbox local time (bob-cli-f7), note-ready startup storm,
relation-index layout-ready, schedule-work-log's fixed future date (bob-cli-ff), and the
task-wikilink rank-budget flake (bob-cli-f8). These are leads, not blanket waivers. For
any failure, compare the exact failing test on the pre-change clean base within the
opened checkout's authorized working area, retain failure text and base SHA, fix
regressions caused here, and record confirmed preexisting failures as
`PROPOSED FOLLOW-UP:` notes. A proven clean-base failure does not leave this work open.

Every phase that changes bob-plugins must obey its `AGENTS.md` deployment requirement:
build and run
`bob plugins sync --repo <opened-checkout> --no-pull -p task-status-cycler` from the
authorized checkout, recording the result. Earlier phases remain dormant until runtime
wiring lands. The ship phase verifies the final build was actually deployed. Do not
mistake skipped/conflicting sync output for success or overwrite another deployment
blindly. Reconcile current landed source/provenance if needed.

If Obsidian can open a **scratch vault copy**, verify Ctrl+Enter on live-linked T offers
S, Esc leaves S unchanged, and a pick links it at the expected destination. Otherwise
record that interactive verification remains for Bryan; do not mutate his live tasks
just to test. Fake-app behavioral tests remain mandatory.

Acceptance: documentation and manifest are current, required checks pass or have
identical clean-base evidence, the final cycler build is synced, and live-smoke status
is explicit.

## Land verification and completion

The child land agent reads every phase's evidence, verifies the final combined source
and deployed build, and confirms that rule/copy vectors and runtime behavior are
covered. Re-run checks only where integration changes or unresolved failures justify it.
Consolidate proposed follow-ups onto bob-cli-fs.8 for the outer epic land agent.

Before closing any assigned phase, run `sase bead epic-symbols <its-id>` and resolve
every leftover or re-key it to an appropriate still-open bead. Do not introduce
temporary symbol exemptions for completed work. Use normal SASE finalization for each
changed repository; do not manually commit with git.

After all child phases and the child epic are complete under SASE's normal land flow,
run `sase bead epic-symbols bob-cli-fs.8`, resolve any remaining entries, and close
**only the containing assigned phase** with
`sase bead close bob-cli-fs.8 --note "<implementation, tests, deployment, and smoke evidence>"`.
Leave bob-cli-fs and every other ancestor open for their own land agents. The note must
identify the deployed cycler version, vector and wiring coverage, confirmed baseline
failures if any, and whether a live Obsidian smoke was possible.
