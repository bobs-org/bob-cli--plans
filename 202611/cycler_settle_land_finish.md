---
tier: tale
size: medium
title: Finish cycler gesture settle and land bob-cli-q4.9.1
goal:
  Make tier-1 cycler settles undo as one step in real Obsidian. Pin the missing
  close-settle behaviors in tests. Fix the counted-cycle identity snapshot and complete
  the cycler docs. Then close epic bob-cli-q4.9.1 and its parent phase bob-cli-q4.9.
proposed_by: bbugyi200.athena.bob-cli-q4.9.1.land
proposed_with: claude/opus
bead: bob-cli-q4.9.1
status: done
---

- **BEAD:**
  [bob-cli-q4.9.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q4/bob-cli-q4.9.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-q4.9.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-q4.9.1.land.md)
- **COMMITS:**
  - [c665869](https://github.com/bobs-org/bob-plugins/commit/c665869ec38b912f5ba7bb179003789fe96e7516)
    — fix(cycler): join gesture settles to undo history

# Finish cycler gesture settle and land bob-cli-q4.9.1

This tale is the remaining work found by the land agent of epic `bob-cli-q4.9.1`,
"Settle cycler gestures without moving rapid-cycle targets". The epic has four phases,
q4.9.1.1 to q4.9.1.4, all closed. Their bob-plugins commits are `071e0e9f`, `77bb466e`,
`1c2534cc`, and `6d364347`. Task Status Cycler 1.64.0 and Bob Ledger Tools 1.166.0 are
deployed and byte-identical to master. Most of the epic is verified. The gaps below are
still this epic's work. Nothing else resumes this landing, so the final step of this
tale closes the epic and its parent phase.

Read these first:

- `sase bead read bob-cli-q4.9.1 -r "<why>"`, including its land-triage note, and the
  four phase beads.
- The epic plan `plan:202611/cycler_gesture_settle.md` through `sase artifact read`,
  especially "Acceptance and docs" and "Child epic land agent".
- The parent phase `bob-cli-q4.9` and its DECISIONS. These are final:
  `settle_cursor = stay` (record a Ctrl+O jump to the moved task),
  `o_in_group = intake_end`, and `memory_settle_record = no` (edit no memory files).

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Paths below are relative to
that checkout. Edit `src/` fragments and rebuild with `npm run build`; never hand-edit a
generated `main.js`. Keep every hand-edited fragment at 1000 lines or less. Do not
change bob-cli's Cargo version or CHANGELOG. Do not touch the Rust settle transform, the
settle vectors, or q4.12's Ctrl+Shift+O work.

Leave bob-navigation-hotkeys source alone. Active epic `bob-cli-q4.10.1` is editing
`181a`/`181b` right now. Its nav jump-order defect is already recorded as a DISCOVERED
ISSUE on that epic and is out of scope here. The same goes for its stale nav `main.js`
on master, which `npm run build:check` and 3 nav `test:unit` nodes report. If those
nav-only failures are still present, reproduce them on the clean base and cite that
note. Do not fix them.

## 1. Make the tier-1 history join real

Measured at bob-plugins `7d53d189`: `npm run test:e2e:task-settle-undo` reports
`historyJoined: false`. The settle is a second undo step, even when the caller passes
`joinHistory: true`.

The root cause is in `plugins/bob-ledger-tools/src/139g-task-settle-editor-apply.js`, in
`taskSettleDispatchApply`. It dispatches with `userEvent: "input"`. Obsidian's bundled
CM6 history only merges an event into the previous one in two cases:

- Its userEvent is undefined or matches `/^(input\.type|delete)($|\.)/`, and the time,
  no-selection-change, and adjacency conditions hold.
- Its userEvent is exactly `input.type.compose`. CM6 always joins that event to the
  previous one. Obsidian's own Vim adapter uses this to group edits.

`grep -a` of `/snap/obsidian/current/app/resources/obsidian.asar` shows both rules.
Plain `"input"` never joins. The provider's unit mock in
`scripts/test-ledger-tools-task-settle-editor.cjs` (around lines 120-130 and 555-570)
treats `"input"` as joinable, so the unit suite hides the bug.

As a result, the following tier-1 consumer paths that pass `joinHistory: true` undo in
two steps today:

- The Tasks-backed close: `settleCloseInActiveEditor` in `143-plugin-close-settle.js`,
  called from `160-plugin-completion.js` at roughly lines 354, 470, and 546.
- The nonempty `o`/`O` scaffold commit: `141-plugin-open-line.js`, around line 160.
- Nav's own joined settles.

The epic plan requires one-step undo for these: "the already-landed provider's two-step
fallback does not waive the containing plan's tier-1 contract".

The fix:

- Make the provider's joinHistory dispatch use a mechanism that real Obsidian merges.
  The recommended mechanism is `userEvent: "input.type.compose"` on the joined dispatch
  only. Keep `bob.taskSettle` for non-joined and staged writes.
- If measurement shows a side effect, choose another supported CM6 mechanism and record
  why. Side effects to check: Vim mode state, editor-suggest popups, autocomplete, and
  IME handling. A history-config `joinToEvent` contribution is one alternative, but it
  still requires no intervening selection change.
- Keep `historyJoined` honest. It is true only when the history depth stays unchanged
  and the restored buffers prove the join.
- Update the unit mock so it applies the real CM6 joinable rule, then assert the new
  userEvent.
- Update `docs/develop/task-settle.md` (Undo section) with the measured behavior.
- Update `scripts/test-obsidian-task-settle-undo.cjs` so a joined apply must restore the
  original buffer on the first undo and the settled buffer on the first redo.
- Bump the ledger-tools manifest per repository convention.

This changes behavior for every joinHistory consumer (cycler, nav, block-id-prompt), and
that change is intended. Run their settle unit suites.

The delayed-cycle path is scheduler-driven. The scheduler requests a join only while the
buffer still matches the last tap, so this fix may turn that path's accepted two-step
undo into one step. Measure it. Then make `docs/task-status-cycler.md` and
`scripts/test-obsidian-task-cycle-settle-e2e.cjs` state the observed count exactly.
Neither may say "two steps" if the measurement shows one.

## 2. Real-Obsidian tier-1 measurements

Extend the disposable-vault harness. Either extend
`scripts/test-obsidian-task-cycle-settle-e2e.cjs` or add a sibling with its own
`package.json` `test:e2e:*` script. Drive actual cycler commands or Vim actions, with
ledger-tools and the Tasks plugin loaded. Copy `obsidian-tasks-plugin` read-only from
`~/bob/.obsidian/plugins` the way `scripts/obsidian-task-reqs-e2e.cjs` (around lines
490-540) does. Never mutate the live vault.

Measure undo, redo, and the stay cursor plus recorded jump for:

1. A staged in-place promotion (Ctrl+Shift+] conversion into a grouped note): expect one
   undo step.
2. A nonempty `o` scaffold commit (`o`, type text, leave insert mode): the settle must
   not be a separate undo step.
3. A Tasks-backed Ctrl+Enter close in an area note, including a recurring task whose
   spawn is future-scheduled: expect one undo step back to the pre-close buffer, and one
   redo back to the settled buffer.
4. The delayed single cycle, which already exists: report the measured count.

Assert the restored buffers, never a flag. For the jump, assert that the recorded pair
makes the first Ctrl+O land on the moved task. Check this against the real nav
jump-history semantics, using nav's exported `recordVimJumpTransition` and
`refreshVimJumpCurrentLocation` helpers on the recorded `(from, to)` pair, or the real
nav plugin when its build is current. The cycler records `from` = the moved task and
`to` = the stay cursor, which is correct. Keep each script's JSON summary. The closeout
note quotes it.

## 3. Close-settle test gaps

The plan required several cases that `scripts/test-task-status-cycler-close-settle.cjs`
does not pin. Add them with the real provider loaded. Bob's Tasks fields use Dataview
syntax: `[repeat:: every week] [scheduled:: YYYY-MM-DD]`, not emoji.

- A Ctrl+Enter / `completeTaskAtCursor` close of a recurring task in an area note:
  - A future-scheduled spawn becomes `[?]` under `### Blocked`.
  - A today or past spawn stays `[ ]` among the open tasks.
  - Cover an id-bearing and an idless recurring row in both cases.
  - `locateSettledClosedTask` returns the `[x]` row, never the spawn.

  The land agent's probe already showed the future and today cases behave correctly, so
  these are pinning tests.

- An ungrouped or ineligible note (for example a daily note with no area or project
  type): it closes without grouping.
- Idempotence: running `settleNote` again on every settled output reports
  `changed: false`.
- Walk interplay with the real cycler and provider loaded:
  - Landed close, last-PRE crossing, POST, and pinned-successor parking each perform one
    close and at most one advance.
  - Successor or walk focus wins over the settle cursor.

  Follow the existing stubs in `scripts/test-navigation-checklist-ctrl-enter.cjs` and
  `scripts/test-task-status-cycler-successor-wiring.cjs`. Add new test files rather than
  editing nav source.

Register any new file in `package.json` `test:unit`.

## 4. Counted-cycle identity snapshot

In `plugins/task-status-cycler/src/150-plugin-commands.js`, `cycleTaskStatusRange`
captures `settleIdentity` via `taskSettleIdentityAtEditorLine` before it calls
`applyBlockedStatusRetirementInEditor`. That call rewrites the row (removing
`[scheduled::]`) and may insert a Schedule Log line. So an idless Blocked first row
hands the provider stale text, and the single scheduled settle is silently dropped.

Fix: capture the identity after the retirement, from the row's newest text, as the plan
requires. The single-cycle `writeCycledLocalTask` in `145-plugin-cycle-settle.js`
already does this.

Add a unit test in `scripts/test-task-status-cycler-task-settle.cjs`: a counted range
whose first row is an idless `[?]` with a past `[scheduled::]` regroups once after the
idle timer.

## 5. Cycler docs

`docs/task-status-cycler.md` § "Status cycling and note settle" covers only the delayed
cycle. Per the epic plan, add:

- The two timings:
  - Ctrl+Enter, transcluded completion, promote and demote, and `o`/`O` commits settle
    immediately.
  - Alt+]/Alt+[ settles after about 500 ms idle.
- The recurring-spawn outcome: a future-scheduled spawn becomes `[?]` in Blocked.
- The stay cursor and Ctrl+O behavior.
- The measured undo behavior for each tier-1 gesture and for delayed cycling.
- Graceful degradation.
- A link to the provider contract (`docs/develop/task-settle.md`) and to bob-cli's
  task-status-hooks contract. Link them; do not copy them.

Keep the page under its word budget. Run the docs memory guidance
(`sase memory read docs.md`) first. Bump the cycler manifest and regenerate the docs
inventory (`npm run docs:inventory` or the repository's documented command).

## 6. Verify and deploy

- Run the focused suites (the ledger-tools settle editor and pending suites, all cycler
  settle and open-line suites, and the nav and block-id-prompt settle suites), then
  `npm run test:unit`, both e2e scripts, `npm run build:check`, and
  `npm run docs:check`.
- Reproduce any failure outside this tale's files on a clean base before classifying it.
  Known pre-existing failures:
  - The nav stale bundle and its 3 nav tests, recorded on `bob-cli-q4.10.1`.
  - The docs word budgets, tracked by `bob-cli-ob`, `bob-cli-il`, and `bob-cli-jz`.
- Run bob-cli `just check`. Its known bob-obsidian lib and cli settle-drift failures are
  tracked on `bob-cli-q4` notes #1 and #3. Classify them only after reproducing them
  identically on a clean tree.
- Deploy only the changed plugins with `bob plugins sync -p bob-ledger-tools` and
  `bob plugins sync -p task-status-cycler`. If sync refuses, follow AGENTS.md and
  `docs/develop/deployment.md`. Never pass `-F` on your own.
- Hand long commands to `/sase_monitor` when needed, and finish through `/sase_final`.

## 7. Closeout (final step; do not defer)

Do this in the same turn as the final code. It must not wait for this work's own commit,
push, or CI.

1. Run `sase bead epic-symbols bob-cli-q4.9.1` and resolve or re-key every listed entry.
   It was empty at landing time.
2. Close the epic: `sase bead close bob-cli-q4.9.1 --note "<verification>"`. The note
   must cover:
   - All four phases and the deployed cycler and ledger-tools versions.
   - The actual tier-1 undo/redo results quoted from the e2e JSON: staged promotion,
     scaffold commit, Tasks-backed close, and the delayed cycle's measured step count.
   - The Ctrl+O order evidence.
   - The new close-settle, recurrence, and walk tests.
   - The counted-identity fix.
   - No memory, Rust transform/vector, or q4.12 changes.
   - The land-triage outcomes from the epic's note: follow-ups declined as already
     tracked, and the nav DISCOVERED ISSUE routed to `bob-cli-q4.10.1`.

   Never use `--force` to make the close succeed.

3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read bob-cli-q4.9.1` shows, `202611/cycler_gesture_settle.md` in the plans
   repo.
5. Handle the parent, phase bead `bob-cli-q4.9` ("Cycler gestures settle, including
   recurring spawns and cycling").
   1. Verify that this child plan completed the phase scope:
      - Tier-1 settle on Ctrl+Enter close, including the future recurring spawn becoming
        `[?]`.
      - Transcluded completion.
      - Promote and demote.
      - The `o`/`O` scaffold commit.
      - Alt+]/Alt+[ settling on idle and flushing on cursor-leave, note-leave, or the
        next Bob command.
   2. Run `sase bead epic-symbols bob-cli-q4.9` and resolve any entries.
   3. Close only that phase:
      `sase bead close bob-cli-q4.9 --note "<what you verified>"`.

   Leave the containing epic `bob-cli-q4` to its own waiting land agent. Never close
   `bob-cli-q4` or another ancestor.
