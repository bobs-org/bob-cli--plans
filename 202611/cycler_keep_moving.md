---
tier: tale
title: Obsidian cycler keep-moving offers
goal:
  Complete bob-cli-fb.4 by offering leaf-project work from every managed Obsidian Done
  gesture with safe picks, clear copy, and same-day reopen receipts.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fb.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-fb.4
create_time: 2026-11-01 13:24:03
status: wip
---

- **PARENT:**
  [202611/leaf_project_keep_moving_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/leaf_project_keep_moving_offers.md)
- **BEAD:**
  [bob-cli-fb.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/bob-cli-fb.4.md)

# Complete bob-cli-fb.4: Obsidian keep-moving successor offers

## Objective and scope

Complete the assigned phase `bob-cli-fb.4` of
`plan:202611/leaf_project_keep_moving_offers.md`. When a managed Done gesture closes the
task behind a leaf project's last live Task Link today, offer that project's eligible
residents in the existing successor picker. Nothing is selected automatically. The
explanation must distinguish these tasks from dependents.

Implementation is in the linked **bob-plugins** repository, principally
`plugins/task-status-cycler`, with tests and its plugin reference. Open it through
`sase repo open bob-plugins -r "Implement bob-cli-fb.4 cycler keep-moving offers"` and
read its `AGENTS.md`. All plugin paths below are relative to that returned checkout. Do
not edit installed vault plugins or import another plugin's source.

This is a medium tale: the contract and navigation renderer are already complete, and
the remaining work is one bounded extension of the existing successor planner, writer,
and receipt. It needs no new architecture, UI, or cross-repository feature.

Binding decisions from the parent epic:

- `offer_lanes = open`: offer Pending, Next, and Ready, in that order.
- `memory_glossary = yes`: the contract phase has already made the authorized glossary
  updates; this implementation needs no memory edits.
- `memory_decision_record = no`: do not write or edit a decision record or other memory.
  Do not reopen either decision in this tale.

Do not implement Rust capture or Mac presentation, alter the contract to accommodate a
failing implementation, edit the CLI version/CHANGELOG, or close `bob-cli-fb`.

## Verified starting point

- `sase bead read bob-cli-fb.4 -r "Need the phase scope and design file"` reports in
  progress, assigned to this phase, without prior attempt or remainder notes.
  Dependencies `bob-cli-fb.1` (contract) and `bob-cli-fb.3` (nav presentation) are
  closed. Do not set the phase's status manually or reopen it unnecessarily.
- bob-cli commit `d6b16f4f` pins the rule in `docs/task-dependencies.md` §§12.15–12.17,
  including K1–K8, G1–G5, KM1–KM32, and LP1–LP12.
- bob-plugins commit `524ffc8` implements navigation `api.successors` v2 and
  `api.notice` v2, keep-moving copy, lanes, folds, icons, and styling. Reuse it.
- There is no `keep_moving`/`last_planned` implementation in the cycler yet. Its
  manifest is currently `1.42.2`.
- The existing successor planner/offers/wiring/polish and navigation picker/notice
  suites passed together: **158 tests, zero failures** on the clean plugin tree.
- `sase bead epic-symbols bob-cli-fb.4` currently reports no entries. Recheck before
  closing; the answer can change while implementation runs.

Read the epic with `sase artifact read` and the current phase with `sase bead read`
before implementation. Consult required reference memory through `sase memory read`,
particularly `docs.md`, the relevant glossary terms,
`decisions:unblocked-tasks-are-offered`, and `decisions:task-lanes-are-sticky`. The
bodies below make the implementation self-contained; the contract owns exact behavior
when verifying vectors.

## Behavioral contract

### Leafness and triggering

A leaf is an eligible project note that is not its own R2 root parent. It lacks
`root: true` and its parent walk encounters another project before an area. Walk untyped
wrappers, stop safely on missing/ambiguous parents, detect cycles, and cap the walk at
16 notes. A project may have children and still be a leaf. A task's `[root::]` never
changes its note's leafness. Top-level projects, own-root projects, orphans, areas,
daily/inbox notes, `done/` archives, and terminal projects do not qualify.

Let C be the tasks this gesture actually moves from open to Done, including closed
embedded subtasks. For each residence N in C, all eight predicates must hold:

1. N passes the existing §12.10 recognition: project frontmatter, non-terminal, outside
   excluded surfaces, exactly one open `^prj` R, no scan issue.
2. N is a leaf by the R2 walk.
3. At least one c in N is not R, has a block ID, and has its **own** live link on the
   gesture baseline. These c form PRED(N); an inherited root anchor is insufficient to
   trigger this feature.
4. No baseline live link targets an open task resident in N outside C. R counts;
   already-closed targets and tasks resident in child projects do not count.
5. R is not in C and is not Blocked, future-scheduled, dependency-blocked, or tagged
   `#prj_is_blocked`.
6. N is not empty by the existing project-empty definition.
7. This gesture generated no `project` group or `project_finished` row for N.
8. `plan.link_unblocked` and `plan.keep_moving` are true, this is a Done gesture, and
   the runtime has a working navigation successor picker version >= 2.

The gesture baseline is today's `## Pomodoros` before the close strikes, moves, retires,
or drops links; use the caller's `dailyBefore` when supplied. Note task statuses come
from staged post-gesture text. Live links are the existing parser's unstruck
plain/embed/deferred Task Links under open Pomodoros, including `()`. Carried or
explicitly dropped links still block K4 because they were present on that baseline. Do
not infer draining from the post-close day.

### Residents, selection, and reporting

Enumerate N's own open `#task` residents, excluding R and C. Do not enumerate descendant
project notes. Eligible `#ref` residents, nested subtasks, and recurring occurrences due
today/past are included. Fold blocked marks, own open prerequisites, and inherited
project blocking as `blocked`; future schedules as `scheduled`; whole-token `#hide` as
`hidden`, applying the contract's classification precedence. Custom marks are ignored.
Ready, Next, and Pending are otherwise selectable.

Dedup priority is `hand_off > project > project_finished > keep_moving`. Keep-moving
groups sort last, then by anchor ledger position, note path, lane Pending/Next/Ready,
and source line. If a group has no selectable rows after classification/dedup, omit the
group and its folds entirely and stay silent.

Keys and numbering reuse existing stable block-ID/digest keys, including duplicate
digest ordinals. Predecessor rows are PRED(N). Reuse slot/closing anchors, D7 type
routing, explicit destination overrides, and guarded writes. K3 supplies an anchor; do
not invent an implicit destination to rescue a failed keep-moving trigger.

A selected row raises only Ready to Next, preserves Next/Pending, mints a missing block
ID, stamps freshness through the existing writer (clearing keeps/unblocked when
eligible), and inserts a plain Task Link. An unselected keep-moving row is byte-for-byte
unchanged: no recovery, no unblocked stamp, no mint, no new link.

Expose `keep_moving` separately from `unblocked`, using the existing row shape with
`trigger: "last_planned"`, `offer.group: "keep_moving"`, `offer.project: N`, and
`unblocked_by: PRED(N)`. Unpicked rows retain their original status and use
`not_linked: "not_selected"`. Keep-moving rows never contribute to the recovered
Blocked-task count. Shared offer counts, order, selections, and placement bookkeeping
must nevertheless include both kinds of rows.

## Implementation

### 1. Add the leaf resolver and pure keep-moving rule

Create `plugins/task-status-cycler/src/089c-successor-keep-moving.js`, register it in
`src/fragments.json`, and export testable helpers from `210-exports.js`. Every
hand-edited source fragment must stay <=1000 lines. `085-successor-plan.js` is already
at that cap: extract cohesive helpers if its small integration hooks would overflow it.
Never hand-edit generated `main.js`.

Implement `successorLeafProject(app, path)` over metadataCache, with a pure walk core if
useful for fixture tests. Match the type/root parsing of
`plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js` (`pomodoroHomeType`,
`pomodoroHomeRootFlag`, and the R2 walk) and bob-cli's
`src/native/pomodoro_homes/{facts,resolve}.rs`. Boolean true and case-insensitive,
trimmed string true are roots. Resolve parent wikilinks through `getFirstLinkpathDest`;
preserve LP7's ambiguity refusal rather than accepting an arbitrary duplicate basename.
Use metadata/path information, not full-vault body reads. Prefer held staged facts for
the just-edited note.

Add cheap candidate-path/needs-prerequisite helpers and the pure K1–K8 planner. Reuse
`projectSurfaceState`, `projectClosedSkipKeys`, task parsers, live-link matching, and
anchor helpers. Inject leaf resolution into the planner instead of making pure planning
read the vault or importing ledger-tools.

A dependency snapshot's availability must be explicit, separate from an empty index.
With a complete snapshot, retain the existing dependency semantics. Without one, fold
residents with nonempty dependsOn as blocked (KM31), and suppress a group whose R cannot
be proved unblocked. Other proven-independent residents may remain offers. Do not
reinterpret a failed lookup as an empty complete snapshot.

### 2. Integrate with the shared offer/write path

Feed keep-moving marks into `planSuccessors` and `applySuccessorVerdicts`, including the
branch where C has no `[id::]` and no project-empty candidates. A `^block-id` is enough
for this trigger; it must not depend on a dependency ID.

Update group priority and `successorOfferOrderCompare` in `085b-successor-offers.js`.
Pass real task note/line/lane data to the comparator and put the lane comparison after
anchor/path comparison for keep-moving only. Reuse existing D7 destination and
theme-intent handling. Preserve all other groups' behavior and ordering.

Separate public `unblocked` and `keep_moving` arrays without breaking row indexes. Use
an explicit internal shared rows collection/accessor for placement indexes, with a
legacy fallback for plans containing only `unblocked`. The public arrays must reference
the same row objects so post-write link/failure updates agree. Route placement creation
and all row lookups through that convention, including `135-plugin-successors.js` note
edits, insertion, and failure handling; `137-plugin-successor-project.js` stale/failure
filtering and final model; and `136-plugin-successor-receipt.js`. Keep
`130-plugin-references.js` recovery counts based exclusively on the actual recovery
rows.

In the verdict applier, branch unpicked keep-moving rows before recovery rewriting.
Never send them through `rewriteRecoveredTaskLine` or project pull-in unhide logic.
Selected rows reuse the existing guarded rank/mint/theme/freshness/link writer. Do not
add a separate writer or a second picker.

### 3. Wire gates, configuration, and revalidation

Extend `planAndApplySuccessorsNow` in `137-plugin-successor-project.js`:

- G1/G2 use held note text: ordinary closed task with a block ID, eligible project
  frontmatter, and at least one potentially selectable resident. Preserve the existing
  no-ID, non-project zero-vault-read path.
- For a possible keep-moving close only, obtain today's day text (prefer held
  `dailyBefore`/editor/cache), then G3's `#^block-id` byte check. A failed G3 must not
  trigger ancestor reads, full scans, or a dependency-pool build.
- Resolve leaf ancestors only after the cheap gates; parse live links and evaluate full
  eligibility after that. Do not add per-note scans for unrelated closes.
- Extend `needsDependents` only when surviving candidates or R require a nonempty
  prerequisite check; ordinary dependency/pull-in requirements remain intact. Reuse a
  warm pool and its editor overlays; record a cold/unavailable failure conservatively
  rather than throwing away all independent keep-moving offers.
- Feature-detect picker v2 **before planning keep-moving offers**. V1/missing picker
  still receives existing groups and recovery behavior but never a keep-moving group it
  would silently discard. Cancelled closes never plan this group.

Add `parseSuccessorKeepMoving`/`loadKeepMoving` adjacent to the existing
`parseSuccessorLinkUnblocked`/`loadLinkUnblocked` in `087-successor-place.js`, or
extract a small shared boolean loader if necessary for the fragment cap. Follow the same
config path, block/flow YAML acceptance, stat-cache, default-true, mobile,
unreadable-file, and invalid-value behavior. Avoid an extra config file read for every
unrelated close. `keep_moving: false` suppresses only this group;
`link_unblocked: false` suppresses all offering while preserving recovery.

Keep the existing one-picker/await/fresh-read/re-plan flow. Preserve the original
pre-gesture eligibility evidence: a fresh post-close ledger must not erase K3 or forget
K4 blockers that were carried/dropped. Separately revalidate candidates and relocate
anchors against current note/day text before applying. KM24 requires a picked resident
that became blocked, closed, or live-linked while the picker was open to be refused
individually; other valid picks still apply. A linked stale row must not accidentally
suppress every original offer solely because the re-plan uses fresh text as a
replacement trigger baseline. Do not revive deleted notes from stale text. Keep existing
ambiguous idless-rebinding refusal and do not show a second picker. Refused picks must
have neither a queued edit nor a placement.

### 4. Build the existing v2 presentation and reopen receipt

Extend `buildSuccessorPickerModel` (`085b`) with keep-moving groups, group-local
predecessors, the fold counts, and row lane names. Use the exact accepted copy:

- Header: `Keep ‹N› moving`; tag: `ready in N · not dependents`.
- Caption: `✓ T was the last N task planned today`; for k >=2,
  `✓ T and k−1 more were the last N tasks planned today`.
- T is cleaned and truncated to 48 characters with an ellipsis.
- Title when all groups are keep-moving: `Keep ‹N› moving` or `Keep 2 projects moving`;
  retain the existing mixed-picker title.
- Fold line: blocked, scheduled later, hidden, omitting zero parts. No individual
  blocked resident rows in a keep-moving group. Lane names are Pending/Next/Ready.

Match the already-landed nav v2 shapes and typographic delimiters; do not duplicate its
renderer or CSS. Existing no-preselection, filtering, retargeting, and budget warning
behavior is reused.

Extend `successorNoticeText` (`087`) and `presentSuccessorModel` (`135`):

- Keep-only selected rows use ⏭, e.g.
  `⏭ Next in FIX: Ship the badge renderer · keeps ‹N› moving` or
  `⏭ 2 linked → FIX · keeps ‹N› moving`.
- Mixed dependency/keep-moving results retain the existing 🔓 forms and count
  keep-moving picks as linked. Unpicked keep-moving offers do not create a toast or an
  inaccurate recovered-task count.
- Send keep-moving rows to `api.notice.showUnblocked` only with notice version >=2. If
  picker v2 exists but notice v2 does not, use one accurate plain-text fallback for the
  complete result instead of letting an old card swallow these picks.
- Walk callers get the composed text via the existing return path, with no extra
  card/toast. Keep-only output never calls these rows unblocked or uses 🔓.
- Retain truthful failure and destination/theme reporting.

Extend `rememberSuccessorPass` to include successful keep-moving picks and their
complete before/after lines (freshness, keeps, unblocked fields, minted IDs included).
Use PRED(N) ownership. Reuse same-day, unchanged-line, untouched-link, and single-use
guards when reopening; edited rows/links and next-day receipts must survive or be
ignored according to the existing receipt contract. A failed/unpicked row must not gain
a receipt. Do not make reopening itself run the keep-moving trigger.

### 5. Add conformance and real-wiring regression tests

Add `scripts/test-task-status-cycler-keep-moving.cjs` and register it in
`package.json`'s test script. Use the existing cycler harness, real exported pure
helpers, fake metadata cache, in-memory editors/vault, and a fake nav picker. Split
fixture helpers or runtime tests into another registered script if that improves
readability. Cite vector IDs in test names/comments and assert observable results, not
source-text patterns.

| Coverage                    | Required assertions                                                                                                                                                                                                                                                                    |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| LP1–LP12                    | Nested/top-level/own-root/root-ancestor/sub-area/orphan/missing/ambiguous/wrapper/middle-project/task-root-override/excluded-surface/cycle behavior; also root scalar parsing and depth bound.                                                                                         |
| KM1, KM9, KM11, KM22        | Three lanes and source order, ID-less predecessor dependency identity, stable/digest keys and ordinals, explicit subset/none/all, slot/placeholder/closing destinations, D7 override, two-project anchor order.                                                                        |
| KM2–KM8, KM10, KM12         | Other live open resident, closed target, struck/closed-entry links, linked R, non-leaf, child residence isolation, absent own predecessor link, carried/drop baseline. Include plain/embed/deferred links.                                                                             |
| KM13–KM18, KM23, KM29, KM31 | No selectable rows is silence; empty-project precedence; hand-off dedup; blocked/scheduled/inherited R; fold precedence/counts; references/nested/due recurring residents; task root ignored; closing R excluded; unavailable snapshot folds only dependent candidates.                |
| KM19, KM20, KM25, KM32      | Both switches, config defaults/cache/errors, cancel/reopen/park exclusions, Esc with no mint/stamp/link/notice, non-project zero added reads, no ancestor/pool reads on a failed day-byte gate.                                                                                        |
| KM24                        | Picker-time blocked/closed/linked task and changed/deleted note; stale row refused, another valid pick applies; fresh placement relocation; no stale queued writes.                                                                                                                    |
| KM26–KM28                   | Real task-line/link/Pomodoro Ctrl+Enter and Done cycling route to the picker; walk lock stays held until settle, including Esc, advances once and emits one composed toast; same-day reopen restores the full untouched row and removes its link, but protects edits and old receipts. |
| Compatibility               | Picker v2 offers vs v1/absent suppression; notice v2 separate array vs older plain fallback; legacy dependency/project paths unchanged; one mixed picker and last-ranked keep-moving groups; exact copy/lanes/folds; partial write failures and mixed placement row indexes.           |

Exercise the nav Links-stage complete-label caller through the existing shared
`api.finalizeClosedTasks` path; ensure cancel labels stay excluded. A Depends-On line
alone must never be counted as the predecessor's own Pomodoro Task Link. Use actual nav
hold/composition behavior or its existing integration harness where needed, rather than
asserting a mocked picker call alone proves walk behavior.

KM21's multi-item net pass and KM30's `--require-pull-in-choice` refusal are Rust
capture responsibilities, so document them as not expressible by this JS engine; do not
invent a JS capture implementation. Compare applicable keys/order/destinations with the
contract fixtures, and with landed Rust fixtures if available without blocking this
phase on the independently scheduled Rust work.

### 6. Document, build, verify, and deploy

Bump only task-status-cycler's manifest minor version (from the current 1.42.2 this is
1.43.0; recheck the version at implementation time). Update the description and the
authored portion of `docs/task-status-cycler.md` to explain leaf keep-moving offers,
both kill switches, explicit choice, v2 compatibility, and reopen behavior. Remove the
now-incomplete claim that every offer is gated solely on `[id::]`. Regenerate managed
plugin inventory with `npm run docs:gen`.

Run focused tests while editing, then all required gates in the linked checkout:

```sh
npm run build
npm run docs:gen
npm test
npm run docs:check
```

Build must regenerate `main.js`; build-check and the <=1000-line fragment guard must
pass. Review the diff and ensure generated files/manifest/reference agree. Run
`bob plugins sync` as required by the linked repo instructions, verifying that the
updated linked source is the deployed source. Do not hand-edit installed files. If a
scratch vault is available in Obsidian, close a leaf project's last planned task and
verify picker, pick, Esc, and reopen there. Never use the live vault for a destructive
smoke test. Otherwise state explicitly that live Obsidian validation remains for Bryan;
automated tests and sync are still required.

## Completion and bead handling

The implementing agent owns completion of `bob-cli-fb.4`; this proposal does not claim
it is implemented. Do not hand-create beads or alter its status. If new out-of-scope
work is found, append only
`sase bead note bob-cli-fb.4 'PROPOSED FOLLOW-UP: <summary — evidence>'`. For a failed
check, reproduce it on the unchanged base before classifying it as pre-existing. An
identical base failure is recorded with evidence and any existing tracking bead, and
does not prevent this phase's close.

Before closing, run `sase bead epic-symbols bob-cli-fb.4`; resolve any remaining symbol
or re-key the Justfile entry to a still-open owning bead, never leave a closed-phase
exemption. Then close only the assigned phase:

```sh
sase bead close bob-cli-fb.4 --note "<implemented scope, tests/gates, sync evidence, and live-Obsidian result or limitation>"
```

Do not close `bob-cli-fb` or any ancestor plan bead. Finish through `/sase_final`,
declaring the modified linked repository so the host commits its work. If a long
verification command needs a handoff, use `/sase_monitor` and wait for the handoff
command itself to exit; never end a provider turn with a live exec session.
