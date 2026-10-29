---
tier: tale
title: Cycle between review groups with uppercase Obsidian keymaps
goal:
  Make [S and ]S jump to the first task of the previous and next nonempty review groups,
  with cycling in both directions.
size: medium
proposed_by: bbugyi200.apollo.b2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-29 02:06:31
status: wip
---

# Cycle between review groups with `[S` and `]S`

## Goal and scope

Change the Obsidian uppercase review keymaps so `[S` lands on the first due task in the
previous nonempty review group and `]S` lands on the first due task in the next nonempty
review group. Both directions cycle. A group is a review tier, not a note, section, task
status, or the separate `]w` lane walk.

This is one bounded implementation for one coding agent: the navigation plugin,
regression tests, and the behavior documentation. It is a medium tale because navigation
shares identity, answer, return, and stale-queue handling, but requires no
queue-producer or CLI contract change.

Open `bob-plugins` using `/sase_repo` and use the path returned by
`sase repo open bob-plugins -r "Implement cycling review-group navigation"`. Read its
`AGENTS.md`. All plugin paths below are relative to that checkout; the two Bob
documentation paths are relative to bob-cli. Edit source fragments and regenerate
`main.js`; never hand-edit generated bundles.

## Current implementation and constraints

- `plugins/bob-navigation-hotkeys/src/490-plugin-lifecycle.js` registers
  `jump-to-first-due-task` and `jump-to-last-due-task`, the existing command surfaces
  for `[S` and `]S`. They currently call `jumpToDueTask(1, { endpoint: "first" })` and
  `jumpToDueTask(-1, { endpoint: "last" })` respectively.
- `480-review-jump-and-nav-api.js` contains `planReviewJump`.
  `536-plugin-review-advance.js` contains `jumpToDueTask`, count capture, the answer
  lock, bounded stale retry, shared landing, and notices.
- `470-keydown-and-freshness.js` owns tier normalization, cursor matching, and boundary
  notices. `538-review-walk-identity.js` owns verified exclusions and text-identified
  anchors. `537-plugin-review-move-park.js` owns parked move resumes.
  `539-plugin-review-walk-return.js` implements lowercase return-to-current behavior.
- `542-lane-walk-model.js::planLaneWalkLaneJump` and `544-plugin-lane-walk.js` are
  precedents for cycling to a group's first task, counting group steps, and reusing the
  group planner on stale retry. Reuse the pattern, not the lane walk's state or
  membership.
- The current review order is PRE → NEW → PROJECTS → PENDING → NEXT → RECURRING →
  TICKLER → ROTTEN → POST. Ledger Tools owns eligibility and ordering within each tier.
  `reviewEntryMachineTier` also handles older providers' `returned`, `resurfaced`, and
  marked legacy `references` rows.
- A landing anchor's `keys` includes the landed task even when it has not been answered.
  Treating all anchor keys as completed would incorrectly remove live tasks, especially
  during single-group cycling.
- `470-keydown-and-freshness.js` and `480-review-jump-and-nav-api.js` are already 998
  and 996 lines. Add a small fragment for the new pure planner, register it in
  `src/fragments.json`, and keep every edited fragment at or below the repository's
  1000-line limit.

Before implementation, read the applicable decision strands through `/sase_memory_read`:
`decisions:review-walk-is-tiered`, `decisions:recurring-occurrences-walk-when-due`,
`decisions:answering-advances-the-walk`, and
`decisions:task-move-never-advances-the-walk`, plus `docs.md`.

## Behavior contract

### Group selection

1. `[S` selects the preceding nonempty tier; `]S` selects the following nonempty tier.
   Both select its first eligible row in the existing queue order. Backward navigation
   must not select the tier's last row. From either the first or a later row in NEXT,
   `[S` goes to PENDING's first row, and `]S` goes to RECURRING's first row, when those
   groups exist.
2. Skip empty tiers. At the ends, POST's `]S` wraps to the first nonempty tier and PRE's
   `[S` wraps to the last nonempty tier. More generally, wrap at the occupied ends even
   when PRE or POST is empty.
3. With only one occupied group, either key cycles to its first row, including when the
   cursor is already on that row. A queue with no eligible rows keeps the existing
   empty-review notice and does not move.
4. `N]S` / `N[S` move N occupied groups in one landing, including multiple circuits.
   Normalize and consume the Vim count before the first await, as other counted walks
   do. Show a wrap indication if any step crossed the boundary, even when the final
   group equals the origin. Avoid loops proportional to an arbitrarily large count.

### Origin and changing queues

Resolve the origin in this order:

1. An exact live due row at the cursor, using path and text identity with the
   established duplicate-line disambiguation. This takes precedence even if another task
   was the previous review landing. Do not mistake unrelated text at a reused line
   number for that task.
2. The same-day `reviewWalkCurrent` row if it still resolves by path and text in the
   live queue. Use its current tier; jump straight to the adjacent group without first
   returning to that task.
3. The existing same-day review anchor's tier, including the origin of a just-answered
   task or parked move whose row/group has disappeared. Select the nearest occupied tier
   strictly before/after that tier in the canonical order, wrapping if necessary. Do not
   take an ordinary one-task resume and then skip that successor's group: that would
   skip an extra tier when the origin group emptied.
4. With no usable origin, `]S` enters the first nonempty group and `[S` enters the last
   nonempty group, always at its first row. That initial entry counts as the first group
   step for an explicit count and is not itself a wrap. Expired anchors/current-task
   references cannot supply an origin after a day change or reload.

Use the current queue's normalized tiers and retain its order within each group.
Preserve existing legacy normalization: a supported legacy REFERENCES group sits between
TICKLER and ROTTEN; current providers must not acquire that retired group. Unknown tier
values must not become new groups or throw. No second freshness evaluator or Rust schema
change is needed.

Exclude verifiably answered rows still present in a lagging Tasks cache, using the
existing identity-aware answer evidence and same-day validity rules. Keep position
anchors distinct from answer evidence: a successful navigation alone never excludes a
task. If necessary, add a narrowly scoped helper/marker to distinguish an answer or move
anchor from a plain landing anchor, preserving all lowercase semantics. Never use an old
line key to exclude a different task now occupying that line.

### Landing and interaction

- Every successful group jump goes through `landOnReviewQueueEntry` and the existing
  guarded note-opening path, records `<C-o>` history, updates the anchor/current task,
  and arms the same answer gestures as a normal review landing. Failed landings do not
  install the attempted position.
- Read the queue afresh for each command. If the target is stale, reread and retry once
  with the group planner, the original direction/count, and the original resolved origin
  tier. If the target group disappears, recompute the next eligible group from that
  origin; do not advance from the failed target. An empty reread or a second stale
  failure uses the existing empty/changed-queue behavior.
- Preserve the answer-in-flight and settle-window guard. Uppercase group jumps bypass
  `returnToReviewWalkCurrent`; lowercase `]s` / `[s`, their counts, Ctrl+Alt+J/K, and
  footer clicks keep their present return behavior.
- Use the normal landing notice, correct queue/tier ranks, wrap marker, quick-roll hint,
  and POST remaining-count tail. Apply the existing forward boundary notice when jumping
  into ROTTEN/POST; skipping groups must not report their tasks completed. Remaining
  counts must include visited-but-unanswered rows.
- From ROTTEN, `]S` reaches POST's first row when POST exists. Multiple POST rows are
  visited in order by ordinary review navigation/answers; do not jump to the last POST
  row. Manual group navigation may wrap out of POST; the existing POST
  completion/auto-advance stop rules remain.
- Both commands still select bracket-repeat family `s`. Subsequent `]]` / `[[` perform
  the ordinary lowercase review walk, not group jumps. Navigation itself does not stamp,
  answer, write task metadata, or journal a review outcome. Existing save-before-switch
  behavior remains in place.

## Implementation steps

1. Add and export a pure `planReviewGroupJump` helper, with a small origin resolver as
   needed, under the navigation plugin's source fragments. Return a result compatible
   with the existing landing pipeline (`kind`, `entry`, `rank`, `total`, `wrapped`,
   `originTier`). Test it through the existing helper export mechanism in
   `700-exports.js`. Factor only what is needed to share reliable identity handling.
2. Add an explicit `groupJump` option to `jumpToDueTask`. Select the group planner on
   both initial selection and stale retry, capture counts synchronously, and bypass
   lowercase return interception only for group navigation (and the existing internal
   endpoint option). Keep automatic answer advancement routed through its existing
   branch.
3. Keep the two installed command IDs so existing vault mappings and callers pick up the
   new behavior with the plugin update. Change their labels and callback arguments as
   follows; retain `rememberBracketRepeatType("s")` and the returned promise:

   | Key / existing ID               | New command label                                   | Call                                     |
   | ------------------------------- | --------------------------------------------------- | ---------------------------------------- |
   | `[S` / `jump-to-first-due-task` | Jump to the first task of the previous review group | `jumpToDueTask(-1, { groupJump: true })` |
   | `]S` / `jump-to-last-due-task`  | Jump to the first task of the next review group     | `jumpToDueTask(1, { groupJump: true })`  |

   Explain the retained legacy IDs in a source comment and the reference. Their previous
   direction arguments must reverse. No new key bindings, duplicate command IDs, vault
   vimrc edit, or chezmoi edit is needed. The internal `endpoint` option may remain for
   existing internal tests and callers; do not keep describing its tests as
   uppercase-key tests.

4. Update `docs/bob-navigation-hotkeys.md` (Review walk and Bracket repeat), the
   relevant manifest description, and bob-cli's `docs/freshness.md` review
   ritual/current-task sections and `docs/guides/run-morning-review.md`. Replace
   unconditional statements that `[S` starts at PRE, `]S` always jumps to POST, or
   uppercase jumps never wrap. Describe entry without an origin, counts, empty groups,
   previous-group first-row behavior, and wrapping. Qualify the closeout hint as ROTTEN
   → POST; clarify that POST answer advancement stops while manual group navigation
   cycles. Regenerate the command inventory using `npm run docs:gen` instead of editing
   generated table rows. Follow the plugin's version/metadata conventions without
   changing bob-cli's Cargo version or CHANGELOG.

## Verification

Add focused behavioral coverage, preferably a new
`scripts/test-navigation-review-groups.cjs` registered in `npm test`, and update
existing integration expectations rather than duplicating the implementation in tests.

- Use all nine current tiers with several rows per tier. Verify both directions from
  first/middle/last rows; previous always chooses row one.
- Cover gaps, absent PRE/POST, wrap in both directions, one occupied group, one row,
  empty queue, and counts spanning multiple circuits. With only NEW, NEXT, and POST
  present, from NEXT: `[S` → NEW first, `]S` → POST first, `2]S` → NEW first with wrap.
- Cover no origin, an off-queue cursor with a live current task, a cursor on another due
  row, a stamp/close/move that empties the source group, old-day anchors, line shifts,
  duplicate text, and reused line keys. A landing anchor alone must not hide its row on
  the next circuit; verifiably answered stale-cache rows must not reappear as targets.
- Cover legacy v3 state normalization, returned/tickler aliases, supported legacy
  REFERENCES, and unknown-tier handling.
- Through actual registered command callbacks verify labels, reversed directions, count
  consumption before await, and bracket-repeat family retention. Update
  `scripts/test-navigation-bracket-repeat.cjs` and command registration assertions in
  `test-navigation-freshness.cjs`.
- Exercise one stale retry, disappearing target group, an empty reread, repeated
  staleness, hard landing failure, missing API, and busy locks. Verify a single
  successful landing/history entry, truthful notices, and no review-answer
  writes/outcomes caused by navigation.
- Exercise real group landings followed by lowercase navigation and a resolving answer.
  Update the uppercase scenarios in `test-navigation-checklist-ctrl-enter.cjs` and
  `test-navigation-freshness.cjs`: reaching POST must use the group path from ROTTEN and
  land on POST row one. Keep internal endpoint tests clearly named and retain lowercase
  return/park/answer regression tests in `test-navigation-review-advance.cjs` and its
  gesture suite.

Build and regenerate inventory before tests: `npm run build` and `npm run docs:gen` in
bob-plugins. Run the focused group/freshness/review advance/checklist/bracket-repeat
tests, then the required `npm test` and `npm run validate`. Run `npm run docs:check`
with `BOB_CLI_DIR` set to the current bob-cli checkout; this invokes `just docs-check`
with the modified plugin docs mounted. These paths must be resolved by the implementing
agent, not copied from the planning environment. If a gate needs a long command handoff,
use `/sase_monitor` and wait for the handoff command to exit. Do not claim unrun checks
passed.

After checks, deploy from the opened plugin source with
`bob plugins sync --repo <opened-bob-plugins-path> --no-pull -p bob-navigation-hotkeys`,
as required by the plugin repo instructions. Honor any existing deployment provenance
refusal rather than automatically forcing it. If an Obsidian session is available,
reload the plugin and smoke-check `[S` / `]S`, both wrap directions, a counted jump, and
an answer after landing. Otherwise report that live Obsidian verification remains
unperformed.

## Acceptance

The installed uppercase commands cycle across occupied review groups and always land at
the destination group's first due task, including backward jumps and multi-row POST.
They retain working answer landings, history, counts, and stale-queue protections.
Lowercase review navigation and automatic answers retain their existing behavior. Tests
and documentation describe the changed uppercase behavior, and the built plugin is
deployed or a concrete deployment blocker is reported.
