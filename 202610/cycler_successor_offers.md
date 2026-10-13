---
tier: tale
title: Choose successor offers in Task Status Cycler
goal: Complete bob-cli-7c.4 with explicit picks, fresh-text revalidation, guarded
  insertion, reversible freshness writes, and verified plugin deployment.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7c.4
bead: bob-cli-7c.4
status: done
---

- **PARENT:**
  [202610/successor_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/successor_offers.md)
- **BEAD:**
  [bob-cli-7c.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/bob-cli-7c.4.md)

# Plan: Complete bob-cli-7c.4 — choose successor offers in Task Status Cycler

## Scope and authority

Implement the `cycler_offers` phase of `plan:202610/successor_offers.md`, then verify
and close only the reserved phase bead `bob-cli-7c.4`. Read it first with
`sase bead read bob-cli-7c.4 -r "Need the phase scope, final decisions, and notes"`.
Read the epic with
`sase artifact read plan:202610/successor_offers.md "Need the binding cycler offer design"`.
Its DECISIONS are final.

The contract phase is closed. `docs/task-dependencies.md` §§12.1–12.13 and its SO
vectors are the implemented contract, including project offers, selection, freshness,
destinations, and the nav picker model. `docs/freshness.md` §5 governs unpicked
recovery. This tale implements one engine and its existing runtime seam; the nav
modal/walk lock, destination override, Rust, Mac, and memory records have separate epic
phases. Do not extend this tale into those phases or edit memory.

No previous-worker notes or remaining work were present on this phase when read. One
coding agent can complete this bounded refactor using the existing parser, planner,
write guards, freshness APIs, and receipt helpers. A child epic is not needed. Do not
create task beads; record discoveries on this phase as
`PROPOSED FOLLOW-UP: <summary — evidence and detail>` for the epic land agent. Do not
manually change the reserved bead's status or close any ancestor.

## Checkout and known starting state

Use `/sase_repo` to open `bob-plugins` with a task-specific reason, read its
`AGENTS.md`, and use only the printed checkout path. Never edit deployed vault plugin
files. Edit `plugins/task-status-cycler/src/` fragments and regenerate `main.js` with
`npm run build`; each hand-edited fragment must remain ≤1000 lines. Register any new
fragments in `src/fragments.json` and new test files in the root `package.json` test
script. Follow the local instructions if they have changed.

Exploration baseline: bob-plugins commit `76924cc`, cycler version `1.37.0`, clean
working tree. `npm run build:check` and these four unchanged suites passed:

```sh
node --test --test-reporter=dot scripts/test-task-status-cycler-successors.cjs scripts/test-task-status-cycler-project-pull-in.cjs scripts/test-task-status-cycler-successor-wiring.cjs scripts/test-task-status-cycler-successor-polish.cjs
```

Relevant source seams:

- `085-successor-plan.js`: `planSuccessors`, ledger parser, anchors, normalization,
  ordering; presently 947 lines and declares `SUCCESSOR_BREAKER_LIMIT = 5`.
- `080-dependencies.js` and `088-project-blocking.js`: project recovery discovery,
  resident enumeration, tag stripping, unblocked freshness guard, idless pull-ins.
- `089-successor-project.js`: pull-in triggers and `applySuccessorVerdicts`, presently
  970 lines; residents are recover-only and the verdict applier auto-links marks.
- `087-successor-place.js`: `planSuccessorInsertions` and `successorNoticeText`.
- `135-plugin-successors.js`: warm/cold pool, editor-first reads, note writes, daily
  insertions, and presentation; its editor insertion branch lacks the snapshot
  comparison that the vault branch already has.
- `137-plugin-successor-project.js`: `planAndApplySuccessorsNow`, the plan/apply seam;
  currently reuses daily text captured before any picker could be awaited.
- `136-plugin-successor-receipt.js`: receipt creation and same-day restoration; only
  project-empty rows currently save a complete `before_line`.
- `130-plugin-references.js`: all managed batch closes funnel through
  `finalizeClosedTasks` and the reference mutation queue;
  `successorNoticeTarget: "return"` composes a walk/cursor toast.
- `010-core.js`: `applyFreshStampLine`; `190-plugin-targets.js`: `getFreshnessStampLine`
  and `getFreshnessMarkUnblockedLine`.

Do not add substantial helpers to the nearly full 085/089 fragments. Extract the verdict
applier and/or add a small pure offer helper fragment as needed, keeping the shared
script scope and declared build order. Expose helpers through `210-exports.js` only when
the existing test harness needs them. Deployed plugins must not import another plugin's
generated entrypoint.

## Implementation

### 1. Make the pure planner produce offers and consume an explicit selection

Extend `planSuccessors(options)` with `selection` and a freshness `stampLine` callback.
Missing/null/empty selection links nothing; allow explicit all and a set of offer keys
for vector fixtures. The runtime passes only the picker’s returned keys. Both planning
passes are side-effect-free: the first pass may describe recovery edits but must mint no
IDs or plan successor-only placeholders. Delete the breaker constant, all breaker
branches, and new-output breaker copy. Keep the unrelated embedded-tree caps of 25
levels and 250 targets.

Produce selectable offers in three groups:

1. `hand_off`: direct dependents fully unblocked by this close.
2. `project`: enumerate all open residents of each project whose `^prj` this gesture
   actually recovered and which has no own/inherited reason left. Remove the `[?]`-only
   and tagged-resident-only enumeration restrictions; include stale `[ ]`, `#ref`, and
   inbox residents. The `^prj` is a header while work remains, not a selectable
   resident. Residents with open own prerequisites/future dates are muted
   `still_blocked` rows carrying `project`; strip their inherited tag per PDC3 without
   clearing their own reasons. Fold hidden and already-planned residents into counts.
   Count open sub-projects from the managed marker for a folded line; leave descendant
   notes untouched. Preserve recovery behavior on cancel and other existing
   recovery-only paths.
3. `project_finished`: retain all three §12.10 pull-in triggers and steps 1–4, including
   unconditional whole-token `#hide` removal; replace the link step with an offer.
   Unpicked project tasks keep the recovered rank.

Deduplicate task identities before selection; a resident also directly dependent appears
once in `hand_off`, merging causes. A still-blocked task, hidden task, or task
live-linked today cannot become a pick. Preserve baseline already-planned behavior for
same-gesture explicit drops and re-check post-gesture live links.
`plan.link_unblocked: false` produces recovery only (`disabled`); cancelled closes
produce recovery only (`cancelled`) and never open the picker.

Use `path#^blockId` keys when possible. For idless rows use
`path@<8 lowercase hex digest of clean description>` and `~2`, `~3`, etc. for same-note
duplicates in document order. A deterministic synchronous digest such as FNV-1a is
allowed by the cycler contract and works without a desktop-only crypto import. Clean
descriptions with the existing body/description helpers; calculate duplicate ordinals
from the note’s task rows, not the selected subset. Never use line numbers as offer
identity. Keys must survive unrelated line insertions, status/freshness changes, and
subset selections. Refuse ambiguous rebinding of duplicate idless tasks after a
concurrent edit.

Preserve the ladder through own/project reasons, `^prj`, hidden, and already planned. A
direct offer uses the earliest predecessor slot/closing/inherit anchor; a resident uses
the earliest anchor of the tasks that unblocked its `^prj`. If no anchor exists, resolve
from current staged daily text: exactly one open timed entry, otherwise the first open
entry in document order. Missing day, section, open entries, or multiple open timed
entries yields `no_destination` and no offer. Extend the existing ledger
parser/insertion helper to handle an implicit entry anchor explicitly rather than
treating it as a closing anchor.

Every offered row must be emitted even when recovery leaves its status unchanged. Attach
§12.5 `offer` metadata: key, number, group, optional project, destination (`day_file`,
`entry_name`, `entry_role`, `fallback`), selected, block_id_needed. Retain project
headers and folded counts for the picker model. An unpicked offer has no link, no mint,
and `not_linked: not_selected`; preserve the existing
`rewriteRecoveredTaskLine`/`markUnblockedLine` WRITE rule. Emit neither `breaker` nor
`not_planned_today`.

For picked rows, raise Ready/Blocked to Next without demoting Next/Pending, mint with
the existing deterministic mint helper only when necessary, strip the project tag, and
call `applyFreshStampLine(line, stampLine, today)` to stamp freshness and clear
keeps/unblocked through ledger-tools. Pass the callback from
`this.getFreshnessStampLine()`; preserve its existing safe behavior when the provider is
missing. Do not invoke the manual-link schedule-retirement writer or change
schedules/IDs/dependency fields. Consolidate tag/status/mint/stamp changes into one
guarded edit per task line so two edits cannot invalidate each other's preimages.
Include the same pick behavior for `project_finished` rows.

### 2. Await one picker and re-plan from authoritative text before writing

Add defensive feature detection for
`app.plugins.plugins["bob-navigation-hotkeys"].api.successors` with version ≥1 and a
callable `pick`. In `planAndApplySuccessorsNow`, retain the existing gate, pool
construction, closed identities, original planned baseline, and project pull-in
discovery. Plan with no selection. If selectable offers exist and the API is available,
construct exactly the §12.13 model and await one `pick(model)` for the entire close
batch. The model uses camelCase row fields, group headers, destination labels, muted
reasons, and camelCase folded counts. Include `{links, max}` budget only when the
existing ledger API can supply it without an extra vault scan. Keep every offer
unselected.

Normalize `{picks: [{key}]}`; null, empty, malformed results, and an unexpected picker
exception mean no picks. Preserve the optional `destination` field as the future
override seam, but do not implement §SO23 in this tale. Without a picker API, apply
recovery with no picks and show the plain "not pulled in" fallback, even if the older
nav notice API exists. No offers means no prompt.

After a picker settles, invalidate per-pass text caches and re-read the daily note,
predecessor notes, affected resident/dependent/project notes, and pertinent prerequisite
state. Prefer live editor buffers. Rebuild the dependents index with freshly parsed
affected documents overriding stale Tasks-cache locations and markdown. Keep warm-cache
discovery scoped to candidate/project notes and preserve the idless/no-candidate
zero-read gate; do not introduce a new full-vault scan on a warm pass. Cold fallback may
retain its existing scan.

Re-plan with the explicit selected keys and current daily/note text. A row that was
closed, re-blocked, linked, hidden, deleted, or no longer identifies the original offer
is refused individually and reported, while unaffected selected rows can proceed. Newly
appearing offers are unpicked and do not open a second picker. Never fulfill a refused
pick by linking its prerequisites.

Daily anchors require revalidation too: retain the original already-planned baseline but
recompute/map slot and closing positions against the re-read text using their task/entry
preimages. Do not carry an old bullet offset into a newly edited ledger. Fail a
moved/ambiguous/closed destination safely when it cannot be identified, rather than
guessing an equally named entry. Build insertion splices from the fresh daily text.
Account for successful same-file task-line edits before capturing the insertion snapshot
so they do not cause a false stale failure. Keep slot indentation/subtree placement and
closing continuation rules; implicit links append as one-tab entry children and replace
a lone stub.

Apply the existing note preimage guards, remove placements belonging to failed note
edits, and then apply daily insertions. Add an exact
`editor.getValue() === insertion.snapshot` check before the first editor splice,
mirroring `vault.process`. A mismatch marks every affected placement failed, inserts
nothing, and surfaces the reason. Preserve cursor handling and existing best-effort
close behavior.

Return the settled model through `successorNoticeTarget: "return"` unchanged in shape so
nav can compose the result into one walk toast after awaiting the close. Do not advance
the walk, manage its lock, or open a second notice/card on that path; the nav phase owns
its picker lock and double-Enter guard.

### 3. Extend receipts and report the result accurately

For each successfully picked row, save the complete pre-pass line and exact post-write
line in `rememberSuccessorPass`, including freshness/keeps/unblocked fields and any
newly minted ID. Same-day reopen removes only untouched owned links and restores the
pre-pass line only while the applied line is unchanged. Keep the existing ownership,
expiry, edited-row preservation, project hide receipt, and created-placeholder cleanup
rules. Do not store receipts for failed picks or invent receipts for capture.

Update `successorNoticeText` for "not pulled in", "no open Pomodoro", project member
recovery, and picked counts, including mixed picked/unpicked batches and failure
reporting. Keep the returned toast, normal nav notice, and plain fallback paths
consistent; a missing picker must never imply an automatic link. Use §12.6 copy and
preserve useful destination/next-up information. Check `finalizeClosedTasks`'
unconditional recovery-failure Notice as well: route successor failures through the same
composed presentation so a returned walk outcome does not also show a separate failure
Notice.

## Verification and acceptance

Adapt existing pure SL/PP link fixtures to pass explicit select-all; retire SL17 and
PP15 breaker assertions. Update old no-anchor assertions to the implicit
destination/no-destination contract and PDC2 to resident offers. Existing runtime tests
expecting links need a fake nav picker that explicitly returns all offers and a
freshness provider where byte-level stamps are asserted. Tests for missing nav must now
assert recovery and zero links/mints.

Extend the four existing successor suites, or add registered SO test files, with vector
IDs in comments. Verify actual planner edits and runtime writes:

- SO1/7–11/20/24/31: `^bar` → `sase_foo`, Ready/Blocked/ref/inbox residents, project
  header rather than a selectable `^prj`, own reasons muted, hidden/live counts,
  hand-off dedup, unchanged-status offers, descendant count without edits, one pick
  mints/raises/stamps/links while unpicked freshness follows WRITE.
- SO2–5/32: 1, 6, 11, and 20 offers, no cap, selection null/none/all/subset; only
  selected rows are linked/minted/stamped, and none creates no placeholder.
- SO14–19/28–30: implicit single timed/first open destination and stub placement;
  missing day/section/open entry and ambiguous timed entries; empty/parent-project
  pull-ins with unconditional hide removal; kill switch and cancel never prompt.
- Stable keys with block IDs, idless duplicate ordinals, and unrelated inserted lines;
  reject stale/ambiguous picks rather than selecting a different row.
- Fake picker subset, null, absent/old/malformed API, and defensive rejection; exactly
  one call for a multi-predecessor/batch close and none for a quiet close.
- Edits during a deferred picker: task becomes blocked/linked/done, project gets a new
  reason/work, line moves, or daily entries move/change. Replanning preserves
  independent valid picks and refuses stale ones; no old offset writes.
- Editor and vault preimage failures after replanning, plus task-line edits in the daily
  file itself. Explicitly test the missing editor snapshot guard.
- Receipt restores original freshness/keeps/unblocked/minted-ID bytes; edited task/link
  and next-day receipt remain protected.
- Existing warm-cache no-unrelated-reads and idless zero-read tests stay green;
  walk/cursor-return closes await the picker and compose exactly one notice.

SO12/13 are capture-batch-only, SO21/22’s lock/key guard and SO25’s marked/highlight
semantics are nav-owned, and SO23 belongs to the destination phase. Preserve the
interfaces they require; do not broaden this phase into their implementations.

Run targeted successor tests during development. Finish with `npm run build`,
`npm run build:check`, `npm run validate`, and the full `npm test` in the opened plugin
checkout, and verify modified fragment lengths. Bump only the cycler manifest version
(next minor, initially 1.38.0) and its README version/behavior row; update manifest copy
that still promises automatic successor links. Regenerate build output after final
source changes. Review the diff for accidental generated-only edits, removed unrelated
safeguards, and other phases' files.

Deploy the tested checkout using its actual opened path:

```sh
bob plugins sync --repo <opened-bob-plugins-path> --no-pull --plugin task-status-cycler
bob plugins sync --repo <opened-bob-plugins-path> --no-pull --plugin task-status-cycler --dry-run
```

Verify that deployment occurred and the dry-run reports no drift. Do not silently deploy
the canonical checkout instead. If a check fails identically on the clean base, record
its reproduction as a `PROPOSED FOLLOW-UP:` note (cite an existing tracking task if
known); it does not keep this phase open. Fix failures introduced by this implementation
before closing.

Immediately before closing, run `sase bead epic-symbols bob-cli-7c.4` again. The initial
audit had none. Resolve any remaining symbols or re-key each Justfile line to the
still-open parent epic or later phase. Then run:

```sh
sase bead close bob-cli-7c.4 --note "<implemented offer/picker/revalidation/receipt behavior; targeted and full plugin check results; version and sync verification; any clean-base follow-up evidence>"
```

Leave `bob-cli-7c`, all ancestor plan beads, and other assigned phases open for their
owners. Use `/sase_final` for the implementation turn's declaration; include the opened
bob-plugins repository as a commit obligation. Report the completed phase, validation,
deployment, and any material limitations concisely.
