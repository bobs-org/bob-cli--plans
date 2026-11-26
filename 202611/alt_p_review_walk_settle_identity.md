---
tier: tale
title: Alt+P advances the review walk after settle regroups the row
goal:
  "Alt+P and every resolving Task Card commit on a ]s landing advance the GTD review
  walk exactly once, even when the same-write task settle moves the answered task into
  another ## Tasks group."
size: medium
proposed_by: bbugyi200.athena.1gi
proposed_with: claude/opus
create_time: 2026-11-26 07:24:38
status: wip
---

# Alt+P advances the `]s` walk again after settle regroups the row

## Goal

Alt+P (and every other resolving Task Card commit) on a `]s` landing must advance the
GTD review walk exactly once, even when the same-write task settle moves the answered
task into another `## Tasks` group. This restores
`decisions:answering-advances-the-walk`: "a resolving Task Card commit" advances once,
and Reword, no-op writes, and refusals stay.

All code changes are in the linked **bob-plugins** repo (`bob-navigation-hotkeys`). Open
it with `/sase_repo` (`sase repo open bob-plugins`) and read its `AGENTS.md` first.
bob-cli itself needs no change, and nobody edits `Cargo.toml` versions or
`CHANGELOG.md`.

## Diagnosis (confirmed: the user's suspicion holds)

**Screencast (`~/tmp/screencasts/20261126_070911.mp4`).** It shows two ⌥P presses, and
the walk never advanced on either:

- **Press 1** is on the landed TICKLER row "Fix unread flicker!" (footer `Review 1/12`).
  The decay write (P1 → P2, scheduled +23d, now Blocked) shows only the rich card
  notice, with no landing toast. The cursor stays on the same line, and the next walk
  task happens to slide up into that line once the answered block moves away. The footer
  later shows `Review 1/11` because the Tasks queue re-indexed under the cursor, not
  because the walk landed there. So the success was a coincidence.
- **Press 2** is on that slid-up row. `reviewLanding` still names the first row, so this
  press was off-landing. It rolls P2 and leaves the cursor on an empty line above
  `Next & In Progress`.

**Root cause.**

1. Alt+P goes `rollByRecommendation` → Task Card `apply-recommendation` →
   `applyRecommendedRoll` / `…DecayWrite` / `…AssignWrite`.
2. Since `ca07830a` (bob-cli-q4.10.1.3, "settle task card, schedule, priority, and
   quick-roll commits"), these writes pass `settleNav: true`. The same transaction
   therefore moves a newly Blocked task to `### Blocked`, a closed one to
   `### Done & Canceled`, and a Next one to `### Next & In Progress`.
3. When the picker closes, `settleReviewOrigin()` (`350-picker-task-card.js`) judges the
   answer from `readReviewOriginAfterLine(modal)`. That reader finds the reviewed row by
   pairing the open-time snapshot with the final editor text through `alignEditorLines`.
   - `alignEditorLines` is a monotone Myers line diff. It was added for BLOCKS shifts
     (bob-cli-94) and cannot follow a block that moves past other lines.
   - When the moved block is smaller than the lines it crosses, the reviewed row reads
     as deleted. The reader returns `null`, the card reports `{ kind: "noop" }`,
     `reviewOutcomeResolves` is false, and the walk stays.
   - When the moved block is larger, the diff pairs it and the walk advances. That is
     why the user saw it as "not reliable".
4. The q4.10.1.6 follow-up (`8645ed55`) moved review queue, landing, and resume identity
   to block IDs (`538a-nav-settle-identity.js`). It did not change this reader.
   - `test-navigation-morning-review-settle.cjs` covers only lane, close, and move
     answers.
   - The real-Obsidian quick-roll e2e fixture has no `## Tasks` grouping, so the regroup
     never happens there.

**Proof** (scratch repro with the real ledger `taskSettle` provider): a grouped area
note, a landing on a small ROTTEN row, and a P1 commit that moves the row under
`### Blocked`.

| Build                                | Outcome            | Walk                                                |
| ------------------------------------ | ------------------ | --------------------------------------------------- |
| `ca07830a^`                          | `card`, no regroup | advances (`Review 1/2`)                             |
| `ca07830a` and HEAD (block-ID rows)  | `noop`             | stays                                               |
| HEAD (ID-less rows)                  | `noop`             | stays                                               |
| HEAD, reviewed block has 12 children | `card`             | advances                                            |
| Prototype of the fix below           | `card`             | advances, in both the block-ID and ID-less variants |

The prototype also passed these existing suites unchanged: review-advance-gestures,
morning-review-settle, task-card-settle, task-settle-identity, relation-complete-writer,
and relation-cancel-writer.

**Affected paths.** Every Task Card commit that settles through `settleReviewOrigin` is
affected:

- Alt+P roll, decay, and assign
- Ctrl+Enter on the recommendation
- P-level picks and schedule dates
- Depends-on adds (which make the row Blocked)
- Card cancel (the deferred settle)
- The inbox-route gate's shared fallback reader (`455-picker-inbox-route-gate.js`)

The decay card's lane steps and Alt+N, close, and move answers advance from the walk
anchor or explicit outcomes, not from this reader.

## Governing decisions and contracts

- `decisions:answering-advances-the-walk`: a resolving Task Card commit on the landed
  row advances once; Reword, no-op writes, and Esc stay; one landing toast follows the
  rich card.
- `decisions:recurring-occurrences-walk-when-due`: RECURRING card resolution rules
  (`reviewOutcomeResolves`) are unchanged; only how the after-line is found changes.
- `docs/develop/task-settle.md` § Identity: a non-empty block ID is authoritative, a
  unique ID still resolves after the row moves, and an absent or duplicate ID is
  unresolved.
- bob-cli-q4 decision `settle_cursor=stay` is untouched: the editor cursor still stays
  through settle. The walk advance is what moves the cursor to the next due row.

## Changes

### 1. Make the review-origin after-line reader identity-first

`350-picker-task-card.js` is at 999 lines, against the 1000-line cap for hand-edited
fragments.

- Move `readReviewOriginAfterLine` out of that file into
  `plugins/bob-navigation-hotkeys/src/538a-nav-settle-identity.js`, beside the other
  settle-identity forwarding.
- Keep it a top-level `function` declaration with the same name. Fragments share one
  script scope, so `settleReviewOrigin`, the inbox-route gate, and the `700-exports.js`
  export keep working.
- Update the comment above `settleReviewOrigin` to describe the new resolution order.

New resolution, still never throwing and never substituting the cursor line:

1. Keep the current guards: a non-integer `reviewLineIndex`, or an index outside the
   snapshot, returns `null`. A missing snapshot or editor falls back to
   `getEditorLine(editor, reviewLineIndex)`.
2. `reviewed = oldLines[reviewLineIndex]`. If `navTaskSettleLineBlockId(reviewed)` is
   non-empty, the block ID is authoritative:
   - exactly one line in the final text carries that ID → return it (closed rows
     included, so a cancel still resolves through `reviewAdvanceLineIsClosed`);
   - zero or several → return `null`.
   - Never fall back to line alignment for an ID'd row.
3. With no block ID, keep today's `alignEditorLines` pairing first. This preserves the
   bob-cli-94 BLOCKS insert and delete behavior and the positional choice between
   duplicate rows.
4. If an ID-less row is unpaired, return the single final-text line whose
   `editorLineAlignmentKey` equals the reviewed line's key. Do this only when the key is
   non-blank, matches exactly one snapshot line and exactly one final line, and that
   final line is a task line. Otherwise return `null`.
   - The key already strips inline fields, the trailing block ID, and the checkbox
     character, so a re-statused and moved task still matches.
   - A Reword changes the key and correctly stays.

Rebuild with `npm run build`; never hand-edit `main.js`.

### 2. Tests

Add these cases and keep every existing review-advance and BLOCKS test green.

**Helper unit tests** for `helpers.readReviewOriginAfterLine`, placed near the
bob-cli-94 cases in `scripts/test-navigation-review-advance-gestures.cjs` or in a new
file registered in `package.json` `test:unit`:

- A block-ID row moved below a larger block returns the moved, changed line.
- A duplicate or missing block ID returns `null`, even when the line at the old index
  still looks similar.
- An ID-less row moved below a larger block returns the unique key match.
- An ID-less row whose key is duplicated returns `null`.
- A Reworded ID-less row returns `null`.

**Real-settle acceptance** in `scripts/test-navigation-morning-review-settle.cjs`, using
its real ledger `taskSettle` fixture style, or in a sibling file with the same setup:

- Build a grouped `## Tasks` area note. The landed Ready, ROTTEN, or TICKLER row's
  post-write block must be smaller than the rows between it and `### Blocked`; give the
  later tasks child notes.
- `]s`-land on it, then press Alt+P through `plugin.rollByRecommendation(editor, …)`
  (see `scripts/test-navigation-quick-roll.cjs` for options).
- Assert all of the following:
  - the row now sits under `### Blocked` with a future `[scheduled::]` and `[?]`;
  - the note is a settle fixpoint;
  - the gesture and regroup are one transaction;
  - `reviewLandingEpoch` grew by exactly one;
  - the cursor is on the next due row by identity, not the row that slid into the old
    line;
  - the notices are the rich card plus one `Review n/m` toast.

Cover these variants:

- ID-less rows
- the decay recommendation (P1 → P2)
- a Task Card P-level pick or Ctrl+Enter
- a card cancel, which moves the row to `### Done & Canceled` and advances

Add these negative cases:

- a duplicate-title ID-less row whose identity is ambiguous stays (epoch unchanged);
- the large-block case still advances exactly once (no double advance).

**Real-Obsidian e2e** (`scripts/obsidian-quick-roll-e2e.cjs` and
`scripts/obsidian-quick-roll-e2e-lib.cjs`):

- Make the walk fixture a grouped note (area `type` frontmatter plus `## Tasks`) so the
  landed walk task regroups to `### Blocked` on Alt+P.
- Each walk task is one line, and the roll adds a two-line SCHEDULE LOG, so make the
  crossed rows outnumber that block, for example by giving later walk tasks child notes.
- Assert the source row moved under `### Blocked` and keep
  `answer-lands-next-review-row`.
- Keep the offline `--self-test` passing.
- Run
  `xvfb-run -a node scripts/obsidian-quick-roll-e2e.cjs -e "$(command -v obsidian)"`.
  Ideally it fails before the fix and passes after. If the host reports `unrun` (exit
  2), record that in the final report instead of claiming the run.

### 3. Version, docs, deploy

- Bump the `bob-navigation-hotkeys` manifest patch version (currently 2.107.11),
  following the repo's version convention (`docs/develop/architecture.md`).
- Regenerate the docs inventory so `docs/index.md` matches
  (`npm run docs:inventory:check` must pass).
- Add one sentence to `docs/develop/task-settle.md` § Identity: the Task Card's review
  settle resolves the answered row by the same identity rules, with an alignment-key
  fallback for ID-less rows. This lets future settle work find this consumer. Keep
  `npm run docs:check` green.
- User-facing docs already promise this behavior ("A resolving commit on a landed review
  row advances once to the next item"), so they need no change.
- After committing, run `bob plugins sync`. Follow the AGENTS.md refusal procedure;
  never pass `-F` on your own.

## Verification

- `npm run build:check`
- `node --test scripts/test-navigation-review-advance-gestures.cjs scripts/test-navigation-morning-review-settle.cjs scripts/test-navigation-task-card-settle.cjs scripts/test-navigation-task-settle-identity.cjs scripts/test-navigation-quick-roll.cjs scripts/test-navigation-inbox-route-card.cjs scripts/test-navigation-relation-complete-writer.cjs scripts/test-navigation-relation-cancel-writer.cjs`,
  plus any new test file
- `npm test` (full unit and perf). Report pre-existing failures with evidence rather
  than weakening assertions.
- `npm run docs:check`
- The real-Obsidian quick-roll e2e under `xvfb-run`, as above.

## Out of scope

- A relation merge that deletes the reviewed row. It stays today, as it did before q4.
- Lane-walk or decay-card advance logic.
- Any change to the settle provider's regroup rules or cursor policy.
- Any bob-cli Rust change.
