---
tier: tale
size: medium
title:
  Finish the Snooze Card's header, labels, footer, Alt+Z, and acceptance, then land
  bob-cli-q5.6.1
goal:
  The Snooze Card shows truthful state, clean event labels, real TODAY counts, and
  conflict-safe Alt+Z, proven in real Obsidian, and epic bob-cli-q5.6.1 and phase
  bob-cli-q5.6 are closed.
proposed_by: bbugyi200.athena.bob-cli-q5.6.1.land
proposed_with: claude/opus
bead: bob-cli-q5.6.1
create_time: 2026-11-27 07:08:04
status: wip
---

- **BEAD:**
  [bob-cli-q5.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.6.1.md)

# Finish and land the Snooze Card (bob-cli-q5.6.1)

## Why this exists

Epic `bob-cli-q5.6.1` (plan `plan:202611/snooze_card.md`) built the Alt+Z Snooze Card in
four closed phases. Its land agent verified the code and gates and found defects that
the epic caused. They are visible in the phase-4 acceptance screenshot
`snoozed-light.png` (attached to bead `bob-cli-q5.6.1.4`, note #1). The 34-assertion e2e
passed because it never asserted the text it rendered. This tale fixes those defects,
adds the missing acceptance proofs, and then performs the epic's landing closeout. **No
other agent resumes this landing after you finish.**

The land agent has already committed (do not redo):

- Three epic-caused unit failures are fixed in bob-plugins:
  - `scripts/test-navigation-relation-links-stage.cjs` now expects the `snooze` row.
  - The `744-snooze-card-modal.js` time input declares `// bob-text: literal`.
  - `scripts/test-navigation-task-card-view.cjs` no longer treats `z` as inert, and a
    new test covers Task Card `z` dispatch and its repeat guard.
- Follow-up triage is done and recorded on the epic (notes on `bob-cli-q5.6.1`). Do not
  re-triage those items.
- Baseline gates on bob-plugins `c6fbeb36` plus those edits:
  - full `npm test`: 10219/10222. The 3 failures are tracked elsewhere: `bob-cli-qt`
    (full /tmp, ENOSPC), `bob-cli-io` (frees-marks perf), and the z test that is now
    fixed.
  - `npm run validate` and `npm run docs:check` pass.
- bob-cli `just check`: 13330 passed, 2 failed (`bob-cli-q9`, `bob-cli-kp`). Neither is
  snooze-related, and no Rust source changed in this epic.

## Governing decisions

- `decisions:today-is-read-from-the-ledger`: TODAY counts only dedicated Task Links
  under today's open Pomodoros. A Snoozed row adds no TODAY count.
- `decisions:off-clock-pomodoros-count-apart`: on-clock counts exclude off-clock links.
  Reuse the shared effective-clock classification; do not invent one.
- `decisions:q-and-ctrl-bracket-mirror-esc`: close aliases are unchanged. Do not regress
  them.
- `decisions:task-lanes-are-sticky`: Unsnooze never changes the lane.
- `decisions:consumption-keeps-notes-and-recorded-time`: snoozing from history leaves
  history intact.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit `Cargo.toml` versions or
  `CHANGELOG.md`.

The parent behavior authority is `plan:202611/snoozed_tasks.md`; read it with
`sase artifact read`. Also read `docs/snooze.md` in bob-cli. Pinned strings:

- Snoozed tooltip and state words: `Snoozed until 15:30 · in 2h 15m` and
  `Snoozed until 15:30 · due — waking`. A blocked row reads muted
  `waits on prerequisites`.
- Suggestion label: `15:30 · after Yogi interview`.
- Footer:
  `leaves WRITE · wakes into WRITE at 15:30 · in 2h 10m · TODAY 9/13 now, 10/13 after`.

## Where the work is

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Edit `src/` fragments, keep
each at or below 1000 lines, and regenerate `main.js` with `npm run build`. Add any new
unit runner to `package.json` `test:unit`.

Key files in bob-plugins:

- Navigation Snooze Card fragments:
  - `plugins/bob-navigation-hotkeys/src/743-snooze-card-model.js`: ranking, footer, and
    header builders.
  - `744-snooze-card-modal.js`: the modal.
  - `745-snooze-card-commands.js`: Alt+Z registration and the Task Card wrapper.
  - `683-snooze-runtime-resolve.js` and `686-snooze-runtime-commit.js`: the runtime, the
    facts assembly in `getSnoozeCardFacts`, and the commit.
  - `738-snooze-mutation-planner.js`: the pure planner.
- Ledger-tools pieces:
  - `plugins/bob-ledger-tools/src/387-snooze-facts-api.js`: the `snoozeFacts` v1
    adapter.
  - `035-meet-event-run.js` `eventRunCleanTitle`: the JS mirror of Rust
    `clean_event_title`.
  - `143c-snoozed-plan-meter.js`: the forecast precedent, `linksCount + on_clock` wakes.
  - `050-plan-budget.js` `computePlanBudget`.
- Rust parity reference: `src/native/capture_complete/snooze.rs` `add_event_ends` in
  bob-cli. It drops spans that cross midnight, already-ended events, and closed Event
  Tasks, and sorts by end time.
- Acceptance: `scripts/obsidian-snooze-card-e2e.cjs` and its offline self-test
  `scripts/test-obsidian-snooze-card-e2e.cjs`.
- Vault-config read precedent: `plugins/bob-command-mode/src/175-args-vim.js`. It reads
  `obsidian-vimrc-support/data.json`, then `vimrcFileName`, then `adapter.read`.

## Work

### 1. Snoozed-mode header speaks the ledger's state words

`activeRow.state` from ledger-tools is `asleep`, `due`, or `blocked`.
`buildSnoozeCardSnoozedHeader` treats any state other than `active` as a raw label, so a
snoozed task's card reads just `asleep`. Map the states:

- `asleep` → `Snoozed until HH:MM · in <Xh Ym>`
- `due` → `Snoozed until HH:MM · due — waking`
- `blocked` → `Snoozed until HH:MM · blocked — <known reason>`, or
  `waits on prerequisites` when no reason is known.

Use the known blocking reason when the runtime has it. Keep the `Wake now` row disabled
with the same reason. Unit-test every state with the real ledger-tools vocabulary, not
invented `active` values.

### 2. Event suggestion labels are clean and filtered like `z:` completion

`snoozeFacts.todayEvents` strips only the leading time. It leaks inline fields, block
IDs, and other trailing syntax, for example
`06:42 · after Yogi interview [root:: bob] [type:: MEET] ^event-suggestion`. Fix it:

- Derive the title with `eventRunCleanTitle`. Keep the `event` fallback for an empty
  title.
- Match Rust `add_event_ends`: exclude spans that cross midnight (end before start) and
  closed or cancelled Event Tasks. Expose whatever additive fields that needs (for
  example `open`) on the v1 adapter without breaking existing consumers.
- Rank by end time with source order breaking ties.

Unit-test inline fields, a trailing block ID, an off-clock tag, a Markdown link, an
overnight span, and a done Event Task.

### 3. The footer's TODAY counts are the real consequence

Two problems: `refreshFooter` passes `afterCount: null`, and `now` is today's pre-snooze
count. Also, `planCounts` calls `computePlanBudget(..., null)` without the daily path,
so same-note `[[#^id]]` links canonicalize wrongly.

- Compute **now** from the proposed post-snooze daily ledger: the pure planner's daily
  postimage for the selected time, with the real daily path. Use no vault writes and no
  subprocess. A retime or Unsnooze of an existing row leaves now unchanged.
- Compute **after** as now plus one only when this task's wake would add a new on-clock
  dedicated link:
  - its identity is not already counted in the post-snooze ledger;
  - its projected destination is on-clock, using the same on-clock classification as
    `snoozePlanMeterModel`.
- Do not count closed history or an existing Snoozed row twice. Over-cap is informative,
  from after, and never refuses.
- Missing facts omit the segment. Never show invented zeros.
- Recompute when the selected time changes and on the existing refresh timer, without
  disturbing authored input.

Unit-test live-linked, unlinked Ready, duplicate-linked, already snoozed (retime),
off-clock, and over-cap tasks. Each test must state the expected `now` and `after`
values explicitly.

### 4. Alt+Z honours conflicts and the configured vimrc

The phase plan required registering Alt+Z only when it conflicts with nothing. Today
`registerSnoozeTaskCommand` always declares the Alt+Z default, even when it has just
detected a conflict.

- On conflict, keep the command and the Task Card entry but declare no default hotkey.
- Also detect conflicts in the configured vimrc file, read through the `175-args-vim.js`
  pattern (`<A-z>`, `<M-z>`, and `Alt+Z` style mappings in any map mode). Do not rely on
  guessed in-memory plugin fields.
- Run the conflict check after other plugins have loaded, so plugin defaults are
  visible. Registering the command at layout-ready is acceptable.
- Never write `hotkeys.json`, the vimrc, or another plugin's config.
- Keep the concise conflict Notice. Keep the docs inventory exact: run the inventory
  check and update the generated table only through its generator.
- Keep macOS option-key `KeyZ` handling.

Unit-test every conflict source and the no-conflict path.

### 5. Acceptance that asserts what it renders

Extend `scripts/obsidian-snooze-card-e2e.cjs` by reusing its fixture, startup, and
diagnostics. Add these assertions against real commands and keys:

- The exact strings: the snoozed header (`Snoozed until 07:52 · in …`), the clean event
  label (`06:42 · after Yogi interview`), and the footer's `now` and `after` numbers for
  the live-linked and Ready fixture tasks. Assert the numbers from the fixture, not from
  the code under test.
- A Ready task promotes to Next and gets its freshness stamp. A Next or In Progress task
  stays byte-identical.
- An eligible definition without a block ID goes through the naming flow. Cancel during
  naming leaves both notes unchanged.
- Re-snoozing a woken link, and snoozing from a closed history link, leave history
  unchanged.
- Snoozing works when today's daily note is closed (the `vault.process` path).
- A simulated Alt+Z conflict leaves `snooze-task` without an Alt+Z default. The
  conflicting binding stays effective and the config bytes stay identical.
- Screenshots of an over-cap footer and a forced-colors render, kept with the existing
  screenshots.

Midnight and stale-edit refusals stay unit-proven by
`test-navigation-snooze-runtime- commit.cjs`; say so in the report rather than faking a
clock in Obsidian. Keep the offline self-test in `test:unit` passing.

Run the real-Obsidian e2e under `xvfb-run -a`. Athena's `/tmp` tmpfs is full
(`bob-cli-qt`). Point scratch and `TMPDIR` at disk-backed storage with a short path,
because Chromium's SingletonSocket limits the path length (see `bob-cli-np`). Do not
delete other agents' `/tmp` files. Hand any command that may outlive your turn to
`/sase_monitor` before starting it.

## Verification

In bob-plugins, with `TMPDIR` on disk:

- `npm run build`, then the focused snooze, Task Card, ledger-tools facts, and
  plan-meter suites.
- `npm run validate`.
- `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`.
- full `npm test`. The only acceptable failures are the already-tracked `bob-cli-qt`
  (ENOSPC on hardcoded `/tmp/bm-e2e`) and `bob-cli-io` (a perf flake that passes in
  isolation). Rerun any other failure in isolation, then fix it or show it reproduces on
  the base tree.
- The real-Obsidian snooze card e2e must pass all assertions.
- `bob plugins sync --repo <opened bob-plugins path> --no-pull`, then a dry-run showing
  the managed files unchanged. Follow the `AGENTS.md` refusal procedure; never pass `-F`
  on your own.

In bob-cli, run `just check` only if you changed bob-cli files. The known `bob-cli-q9`
and `bob-cli-kp` failures are not this work's. Update bob-cli `docs/snooze.md`,
bob-plugins `docs/bob-navigation-hotkeys.md`, and `docs/task-card.md` only where the
behavior wording changes (Alt+Z conflict behavior). Read the `docs.md` memory with
`/sase_memory_read` first.

Attach the e2e JSON report and the new screenshots as audited evidence on
`bob-cli-q5.6.1` with `sase bead note ... @<file>`.

## Closeout (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-q5.6.1` and
   `sase bead epic-symbols bob-cli-q5.6`. For each listed entry, resolve the symbol
   (wire it up, privatize it, add a non-test pragma, or delete it), or re-key it only to
   a still-open bead that needs it. At the land agent's check, neither bead had any
   entries.
2. Close the epic:
   `sase bead close bob-cli-q5.6.1 --note "<what you verified: the fixes in items 1-5, test and e2e counts, deploy result, and the land agent's earlier triage>"`.
   Never use `--force` to make the close succeed.
3. Run `just symvision` in bob-cli if the recipe exists, and confirm the whitelist is
   clean. At the land agent's check, bob-cli had no `symvision` recipe; if it is still
   absent, say so in your final report.
4. Set `status: done` in the frontmatter of the epic's PLAN file: the path that
   `sase bead read bob-cli-q5.6.1` prints under EPIC PLAN, which is
   `202611/snooze_card.md` in the plans sidecar.
5. The epic's `parent_bead` is the **phase** `bob-cli-q5.6`, under epic `bob-cli-q5`.
   - Check that the work satisfies that phase's description: the Alt+Z command and Task
     Card `z`, ranked suggestions, the consequence footer, retime, Wake now, and
     Unsnooze, teaching refusals, and the JS planner pinned to the shared vectors.
   - Record the final bob-plugins and bob-cli source revisions, vector results, test
     summaries, deploy result, and acceptance artifact refs on it.
   - Close only that phase: `sase bead close bob-cli-q5.6 --note "<what you verified>"`.
   - Never close `bob-cli-q5` or its other phases; its own land agent is waiting.
