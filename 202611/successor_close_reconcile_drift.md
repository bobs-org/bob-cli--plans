---
tier: tale
title: Successor-card Done/Cancel survives a concurrent bob task reconcile
goal:
  A staged Done or Cancel on an Obsidian successor card applies even when bob task
  reconcile recovers that row (Blocked to Ready, unblocked stamp, regroup) while the
  card is open, and any other change still refuses.
size: medium
proposed_by: bbugyi200.athena.1fi
proposed_with: claude/opus
create_time: 2026-11-23 08:13:00
status: wip
---

# Successor-card Done/Cancel must survive a concurrent `bob task reconcile`

## Problem (confirmed from vault history)

Bryan closed `bob.md#^micro-tasks` from a PENDING `]s` landing. On the successor card
that followed, he staged `Ctrl+X` (Done) on `^micro-tasks-cleanup`, `Ctrl+Shift+X`
(Cancel) on `^s-tasks-requirements`, and pulled in `^better-o` and
`^tasks-parent-projects`. Only the pull-ins landed. His suspicion is right about both
closes.

Vault git timeline for 2026-11-23 (all commits from the Mac's vault sync):

| Time        | Commit                 | What happened                                                                                                                                                                                                                                                                                                                     |
| ----------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 07:30:33    | `7237a7a2a`            | `^micro-tasks` goes `[/]` → `[x]` in place. All four successors are still `[?]` in `### Blocked`. The successor card is now open.                                                                                                                                                                                                 |
| 07:30–07:33 | `c246b697b` (07:33:32) | `bob task reconcile` (from `bob tick`) rewrites 6 notes. It recovers all four successors `[?]` → `[ ]`, moves them to the top Ready list, and moves `^micro-tasks` to `### Done & Canceled`. It stamps `[unblocked:: 2026-11-23]` only on `^s-tasks-requirements`, the one row with `[fresh::]` (SV20 WRITE skip for the others). |
| 07:33:33    | review journal         | `review.answer` `tier: pending`, `outcome: done`, `key: bob.md#micro-tasks`. The card settled.                                                                                                                                                                                                                                    |
| 07:33:49    | `3fc77e8c0`            | Only the two pull-ins are written: `[*]` + `[fresh::]` + links under `() — BUILD`. There is no `[x]` on `^micro-tasks-cleanup` and no `[-]` on `^s-tasks-requirements`.                                                                                                                                                           |
| 07:56:52    | `4fedd85e9`            | `^s-tasks-requirements` is cancelled by a TICKLER **drop** in the review walk. That is the only reason it is `[-]` today. `^micro-tasks-cleanup` is still `[ ]`.                                                                                                                                                                  |

## Root cause

The successor card ran from 07:30:33 to 07:33:33. During that window,
`bob task reconcile` did the same Blocked → Ready recovery the card had planned.

Obsidian binds each staged Done/Cancel to the line preimage the card showed (the `[?]`
line). In bob-plugins `task-status-cycler`:

- `successorOfferCloseExpectedLines` (`src/137a-successor-offer-close-integration.js`)
  accepts only the plugin's _own_ planned recovery edit
  (`edit.before === card preimage`) as a different expected line. After the post-pick
  revalidation replans from fresh text, the rows are already `[ ]`. The plan has no
  recovery edit for them, so the expected line stays the card's `[?]` line.
- `successorOfferCloseValidateApplyRow` (`src/089i-successor-offer-close-apply.js`)
  refuses on `current !== expectedLine` with `changed_identity`. The receipt says
  `2 not closed · changed since the card opened`, and nothing is written.
- Pull-ins survive because they re-plan by key against the fresh text.

A scratch reproduction used `scripts/task-status-cycler-harness.cjs` and
`finalizeClosedTasks` with a fake v4 picker that rewrites `Tasks.md` while "open" and
then returns one Done and one Cancel. It isolates the trigger:

| Drift applied while the card is open    | Result today                           |
| --------------------------------------- | -------------------------------------- |
| none                                    | both apply                             |
| line move only (section regroup)        | both apply (key relocation works)      |
| `[?]` → `[ ]` only                      | **both refused**                       |
| `[unblocked:: today]` stamp only        | **stamped row refused**                |
| full reconcile (flip + stamp + regroup) | **both refused**: matches the incident |

So the defect is narrow. Bob's own ordinary recovery of the row counts as "changed since
the card opened". It is not a user edit, and it cannot change what the verdict means.

## Fix

Treat recovery drift as the same preimage. Recovery drift is the exact rewrite that
Bob's ordinary dependency or project recovery applies to a row
(`rewriteRecoveredTaskLine` in `src/088-project-blocking.js`, and `bob task reconcile`).
Every other change still refuses.

### 1. Contract and shared fixture (bob-cli owns these)

- `docs/task-dependencies.md` §12.22, **Write failures differ by surface**: after "Both
  bind the verdict to the preimage shown in the card", add that Obsidian treats Bob's
  ordinary recovery of the row as the same preimage:
  - Blocked `[?]` → Ready `[ ]`
  - an added (or restamped) `[unblocked:: <date>]` field
  - a removed `#prj_is_blocked` tag
  - a move to another line of the same note, such as a reconcile regroup

  So a `bob task reconcile` that runs while the card is open never refuses a staged
  verdict. Any other change to the row is still refused alone.

- §12.23 table:
  - Narrow SV14's situation to "changes before apply (other than Bob's ordinary
    recovery)".
  - Add **SV26**: "Bob recovers an Obsidian verdict target before apply (status flip,
    `[unblocked::]` stamp, regroup)" → "The verdict applies to the recovered row. The
    receipt reports its current status. A row that also changed otherwise is still
    refused."
- `tests/fixtures/offer_close/vectors.json`: append
  `{"id": "SV26", "kind": "ui_only", "surface": "obsidian", "expected": {...}}` with
  booleans that mirror the row, for example `recovery_drift_applies`, `status_flip`,
  `unblocked_stamp`, `prj_tag_removed`, `line_move`, and `other_change_refused`.
  - Keep the ids sequential: the Rust test
    `fixture_lists_all_shared_scenarios_without_renumbering` asserts `SV{index+1}`.
  - Keep the file's existing formatting style.
- `tests/fixtures/offer_close/README.md`: where it says SV14–SV16 pin Obsidian and
  review-walk behavior, add SV26 to the Obsidian surface-only cases.
- No Rust code change. The capture engine re-resolves verdicts by key at submit time
  inside one locked run, so it has no card-time line preimage to race.
- Never edit `Cargo.toml`'s version or `CHANGELOG.md` (release train).

### 2. Plugin fix (bob-plugins, `plugins/task-status-cycler`)

Open the linked repo with `/sase_repo` (`bob-plugins`) and read its `AGENTS.md`. Edit
`src/` fragments only, then run `npm run build`. Never hand-edit `main.js`.

1. **New pure helper in a new fragment.** `089i` is at 986 lines, and the cap is 1000.
   Create, for example, `src/089k-successor-offer-close-drift.js` and add it to
   `src/fragments.json` right after `089j-successor-offer-close-identity.js`. Provide
   `successorOfferCloseRecoveryDrift(expected, current)`, which returns `true` only when
   all of these hold:
   - Both lines parse as task lines (`getTaskStatusForLine`).
   - The status pair is unchanged, or exactly `?` → ` `. Reject `?` → `*`/`/`, ` ` →
     `?`, any closed status, and every other transition.
   - The lines are byte-identical after normalizing both:
     - set the status box to the same symbol
     - drop every `[unblocked:: YYYY-MM-DD]` inline field together with its one leading
       space
     - drop `#prj_is_blocked` via the existing `stripProjectBlockedResidentTag`

   Return `false` for identical lines too; callers handle exact equality first. Export
   it from `src/210-exports.js` for tests.

2. **Route every expected-vs-current comparison in `089i` through it.** One helper keeps
   the stages consistent:
   - `successorOfferCloseValidateApplyRow`: when `current !== expectedLine` and the
     helper accepts, adopt `current` as the row's expected line. Continue all existing
     identity, eligibility, live-link, and writer checks against `current`, and return
     `expected: current`. Otherwise refuse as today.
   - `planOfferCloseRoots`: in the per-path `currentLines[row.line] !== row.expected`
     check, accept recovery drift and adopt that line.
   - `applyOfferClosesNow` live revalidation (`liveLines[row.line] !== row.expected`):
     accept recovery drift and carry the adopted live line into the `expectedLines` map
     passed to the recompute. A reconcile that lands after phase 1 must not refuse
     either. The write-time retry already re-enters `planOfferCloseRoots`, so it
     inherits the rule.
   - Leave the block-id and digest identity checks unchanged. The ID-less digest hashes
     the body after the status box through `cleanDescription`, so verify in tests that
     an ID-less row keeps its key across the drift. If it does not, refuse it as today
     and note why in the test rather than widening identity rules.
   - Leave card-time descriptor building in `089g` (`successorOfferCloseTaskSource`)
     alone. It runs against the card's own snapshot.
3. **Undo and receipts need no new code. Pin them in tests.** The guarded Undo preimage
   becomes the recovered line, so Undo restores `[ ]`, not the stale `[?]`. The
   receipt's `{Status}` comes from the current line ("Ready → Done").
4. `manifest.json`: bump `task-status-cycler` `1.58.1` → `1.58.2`, matching earlier
   `fix(task-status-cycler)` commits. Then run `npm run docs:gen` so the generated
   inventory in `docs/index.md` matches.
5. `docs/task-status-cycler.md`, in the `api.successors` v4 paragraph ("Each choice is
   re-read and checked against the card's line before it writes; a changed row is
   refused on its own…"): add that Bob's ordinary recovery of the row while the card is
   open still counts as the card's line.
6. Copy `tests/fixtures/offer_close/vectors.json` and `README.md` from bob-cli
   byte-for-byte into `plugins/task-status-cycler/scripts/fixtures/offer_close/`.
   `scripts/test-task-status-cycler-successor-verdicts.cjs` asserts byte equality with
   the bob-cli copy, reached at five `..` levels above the linked checkout.

### 3. Tests (bob-plugins)

- `scripts/test-task-status-cycler-successor-wiring.cjs`, next to the existing "SV14:
  stale close is refused per row…" test: add an **SV26** end-to-end test. Model it on
  SV14 with `installOfferCloseNavStub` and `finalizeClosedTasks`.
  1. Start `Tasks.md` with two `[?]` dependents of an `[x]` root in a `### Blocked`
     section. Only one dependent has `[fresh::]`.
  2. Inside `pick()`, rewrite the file the way reconcile does:
     - flip both to `[ ]`
     - stamp `[unblocked:: TODAY]` on the fresh row, placed after `[fresh::]` (bob-cli's
       placement)
     - regroup both into a section above `### Blocked`
  3. Return Done for one and Cancel with a reason for the other.

  Assert:
  - both appear in `offer_closes`, with no `offer_close_refused`
  - exact `[x]` + `[completion::]` and `[-]` + `[cancelled::]` bytes, plus the Cancel
    Log
  - "Ready" in the receipt
  - Undo restores the recovered `[ ]` lines

- Same file, negative and edge cases:
  - Recovery drift **plus** a text edit is still refused, and its neighbour applies.
  - A row re-blocked ` ` → `[?]` while open is refused.
  - An ID-less dependent with recovery drift applies, or is refused with the reason
    pinned (see Fix 2).
  - A `#prj_is_blocked` resident recovered by project recovery (tag stripped plus status
    flip) applies.
- `scripts/test-task-status-cycler-successor-apply.cjs`:
  - a pure matrix for `successorOfferCloseRecoveryDrift`: every accepted drift, alone
    and combined, and every rejected transition
  - an `applyOfferClosesNow` case where the drift lands only on the **live** re-read
    (stub `readSuccessorNoteText` to return the drifted note), which must still apply
- Keep the existing SV14 test passing unchanged.

## Validation

- bob-cli:
  - `cargo test offer_close`
  - `just docs-check`
- bob-plugins:
  - `npm run build`
  - `node --test scripts/test-task-status-cycler-successor-wiring.cjs scripts/test-task-status-cycler-successor-apply.cjs scripts/test-task-status-cycler-successor-verdicts.cjs`
  - `npm test`
  - `npm run docs:check`
- After the bob-plugins changes land, run `bob plugins sync` to deploy to the vault. If
  it refuses an overwrite, follow bob-plugins `AGENTS.md`. Never pass `-F` on your own.

## Out of scope

- **Vault repair.** `bob.md#^micro-tasks-cleanup` is still open. Bryan closes it himself
  with Ctrl+Enter. Agents do not write the live vault for this. `^s-tasks-requirements`
  is already cancelled by the later walk drop.
- **Requirement close cards.** Their owner-preimage guard
  (`reqCloseRefuseStaleOwnerChoices`, `198*` fragments) may have the same race when
  reconcile regroups the owner block during the card. If a quick probe confirms that,
  file a task bead through `/sase_new_task` instead of widening this fix.
- Making refusal receipts more prominent is not part of this change.
