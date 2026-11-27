---
tier: epic
title: Complete Alt+Z Snooze Card and Task Card z
goal: Snooze one task safely from Obsidian, retime or remove its row, and wake it
  through bob, with keyboard and undo behavior verified before closing bob-cli-q5.6.
phases:
- id: planner
  title: Snooze add, retime, and drop planner
  size: medium
  description: 'planner: build the pure navigation snooze mutation planner, the narrow
    BIP planning adapter, and executable shared snooze vectors.'
  depends_on: []
- id: runtime
  title: Resolve snooze targets and commit guarded note edits
  size: medium
  description: 'runtime: resolve supported cursor contexts, apply same-note and cross-note
    plans safely, provide card facts, and delegate explicit Wake now to bob.'
  depends_on:
  - planner
- id: card
  title: Alt+Z Snooze Card and Task Card z
  size: medium
  description: 'card: build the time picker, ranked suggestions, consequence footer,
    snoozed actions, collision-safe Alt+Z registration, and Task Card integration
    and reference docs.'
  depends_on:
  - runtime
- id: acceptance
  title: Verify the Snooze Card in real Obsidian
  size: medium
  description: 'acceptance: add and run isolated Obsidian acceptance for keyboard
    gestures, undo, refusal and wake behavior; repair integration defects, deploy,
    and record evidence for closing bob-cli-q5.6.'
  depends_on:
  - card
proposed_by: bbugyi200.athena.bob-cli-q5.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.6
create_time: 2026-11-27 01:34:13
status: done
bead_id: bob-cli-q5.6.1
---

- **PROMPT:** [prompts/202611/snooze_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snooze_card.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.6.1.md)

# Finish the Snooze Card (bob-cli-q5.6)

## Outcome and scope

Alt+Z on one task or Task Link opens a Snooze Card. Enter moves that task's live ledger
link and its notes into today's Snoozed section, or creates a row and commits a Ready
task to Next. Task Card `z` opens the same card. An already snoozed task can be retimed,
woken now through bob, or unsnoozed without changing its lane.

This is a child epic implementing **only bob-cli-q5.6**, the `snooze-card` phase of
`plan:202611/snoozed_tasks.md`. Its child phases close only themselves. Its land agent
completes and closes bob-cli-q5.6 after the child work has landed and been verified. It
must never close bob-cli-q5 or any ancestor of bob-cli-q5.6. Do not manually create
beads or change assigned status. Discovered work outside this scope goes into a
`PROPOSED FOLLOW-UP:` note on the assigned phase, for land triage.

The four phases are deliberately serial: each consumes the prior phase's API, and all
touch the same fragment build. Each is one bounded coding-agent unit. Pure mutation
parity, editor write safety, interactive UI, and application acceptance need separate
verification; none is an open-ended replanning phase.

## Evidence and final decisions

The planning inspection found no previous notes or child plans on bob-cli-q5.6. The
bob-cli tree was clean at `0e32b956`; the opened bob-plugins tree was clean at
`c476086a`. The CLI, capture syntax, planned-today logic, cleanup, minute waker, and
display marks have landed. No `snooze-task` command or Snooze Card exists.
`scripts/snooze-model-vectors.cjs` explicitly reserves the 15 `snooze` add/drop cases
and 10 `wake` cases; four plugins already execute `rows`, `when`, `live`, and `cleanup`.
There were no `--epic-symbol` entries for bob-cli-q5.6 at inspection. Recheck these
facts before editing; preserve newer landed work.

Read the approved parent through `sase artifact read`; it and the current
`docs/snooze.md` are the behavior authority. The parent's referenced research artifact
`research:202611/snoozed_tasks/snoozed_tasks.md` was unavailable through the audited
reader at planning time. No new behavior is inferred from it.

The parent DECISIONS are final: `uncommitted_rows=ignore`, `wake_notice=hud`,
`hold_in_meetings=yes`, `glossary_snoozed_task=yes`, `decision_record=no`. This child
epic edits no memory. The glossary remains the outer close-out phase's work. It adds no
Mac UI, CLI command, wake writer, or capture syntax.

Governing records, read during planning:

- `decisions:q-and-ctrl-bracket-mirror-esc`: close aliases share Esc dispatch; `q`
  remains text in an actual text control and Ctrl+[ never types.
- `decisions:consumption-keeps-notes-and-recorded-time`: preserve authored subtrees,
  match resolved identities, remove only entries this operation empties, and retain
  recorded history.
- `decisions:today-is-read-from-the-ledger`: only dedicated links under open Pomodoros
  are Today. A Snoozed row adds no Today field or link-budget count.
- `decisions:task-lanes-are-sticky`: Next and In Progress remain unchanged; Unsnooze is
  not release. The parent plan specifically refuses Blocked tasks.
- `decisions:answering-advances-the-walk`: use the existing capture/continue review
  lifecycle for a resolving Task Card edit. Successful snooze from a review landing
  advances once; cancellation, failure, and off-landing actions do not advance.

## Source map and common constraints

Use `/sase_repo` to open bob-plugins and read its AGENTS.md. Paths below are relative to
the named repository. Edit fragments under `src/`, keep each at or below 1000 lines,
update `src/fragments.json`, and generate `main.js` with `npm run build`. Add new unit
runners to `package.json`'s `test:unit` list.

In bob-plugins:

- Navigation `091`–`094` are the shared row/time/state/cleanup twins;
  `162-snoozed-cleanup-compose.js` already handles explicit row removal and Snoozed-row
  Alt+N resolution. Retain their shared parity.
- Navigation `180-editor-transactions.js` has `applyEditorContentTransaction`;
  `510-plugin-link-commit-and-lane.js` has preimage and rollback precedents. Its
  _deferred cleanup_ can fail after a task write: that weaker success model is
  unsuitable for the required snooze daily-note write, so do not use it unchanged.
- Navigation `200` supplies the filtered-picker precedent, `310` constructs the Task
  Card model, `314` owns key dispatch/order/shortcuts, `350` owns Task Card stages and
  actions, and `490` registers commands. Add focused fragments instead of growing those
  large files past their limit.
- BIP `070-daily-and-activation-plans.js` owns `planTargetTaskUpdate`;
  `100-plugin-lifecycle.js` publishes its versioned APIs. The target planner is
  currently a helper, not a public planning API. `140` and `132` show target
  snapshot/editor handling. Preserve existing consumers' behavior.
- Ledger-tools `170` publishes APIs, `290a` owns snoozed snapshots, `290b` decodes and
  spawns wake reports, and `291-plugin-gcal-events.js` already parses today's Events.
  Reuse those facts through narrow versioned adapters.
- `scripts/obsidian-snooze-marks-e2e.cjs` and the startup-smoke library provide the
  isolated-vault/CDP acceptance pattern; do not drive the live vault for destructive
  fixture tests.

In bob-cli, `docs/snooze.md`, `src/native/snooze/plan.rs`, and
`tests/fixtures/snooze/vectors.json` pin mutation semantics. Never edit Cargo.toml
versions or CHANGELOG.md. No Rust behavior change is expected.

## Phase planner: pure mutations and target planning

Build a pure navigation planner accepting explicit snapshots, resolved task
identity/facts, today's daily path, an injected Bob-zone clock, and a BIP
target-planning callback. Return a refusal or per-path preimage/postimage plans with
operation metadata. It must not read the vault, spawn, or use ambient time.

Expose a narrow versioned **planning-only** BIP adapter that calls the existing
`planTargetTaskUpdate` with `forceNext`, `freshDateText`, and injected ledger-tools
`api.freshness.stampLine`. Keep eligibility outside that general helper: it can remove
future schedules, but Snooze must refuse effective blocking before invoking it. Ready
promotes and stamps; already identified Next and In Progress lines remain
byte-identical. Do not stamp on a mere retime or drop. If naming is needed, compose
BIP's existing validated block-ID naming plan; the runtime phase owns the prompt and
commit.

Implement add/retime/drop to match the Rust vectors exactly:

1. Resolve the time with the existing `snoozeParseWhen` twin. Support `1530`, `15:30`,
   `3:30p`, `3p`, `19`, `330`, `+45m`, `+2h`, and `+1h30m`. Relative values round upward
   to five minutes; strictly future, same-day only. Keep gap/fold and refusal behavior
   identical to the canonical `when` vectors.
2. Refuse Event, effective Blocked, done, cancelled, non-task, ambiguous/missing
   targets, and a missing Pomodoros section. Derive identities with the existing
   resolver; never remove links by basename-only guesses.
3. Move the first dedicated live link subtree into a canonical column-zero
   `- **HHMM** [[note#^id]]` row. Preserve other duplicate links' authored notes as
   display bullets. Remove only newly emptied untimed entries. Retain running entries
   and all closed/recorded-time history.
4. Retime an active row without losing its children; replace this identity's struck
   Snoozed history when re-snoozing. Sort using the existing canonical planner's
   ordering. Insert the heading after the bounded Pomodoros section, preserving later
   headings and prose. Same-path task and daily edits compose into one final postimage
   even when promotion shifts offsets.
5. Unsnooze removes matching active rows and their subtrees, deleting an empty Snoozed
   section only when no rows/prose remain. It does not edit the task. Missing active
   membership returns `not_snoozed`. Ordinary closed ledger history remains untouched.

Extend the vector driver to execute all 15 `snooze` cases for the new navigation planner
and retain all existing `cleanup` cases. Represent coverage per consumer: do not falsely
claim all four plugins implement add/drop, and do not weaken source-twin, API parity, or
byte-identical fixture checks. The `wake` mutation group remains Rust-owned; this epic
never adds a JavaScript wake writer. Add behavior tests beyond the shared fixtures for
ambiguous aliases, CRLF, fences/trailing H2, note-ID naming, duplicate notes, and
effective blocking before BIP activation. Keep the fixture bytes identical to bob-cli
unless a real contract defect requires a separately justified correction.

Deliver the stable planner/adapter interface and exact coverage in the phase note. Run
the focused vector/planner/BIP tests and generated-build checks.

## Phase runtime: contexts, transactions, and read facts

Provide one controller used by both Alt+Z and Task Card. Resolve task definitions,
dedicated live links, woken links, closed history links, and active Snoozed rows.
Require one task; a counted/multiple-target Task Card offers a clear single-task
refusal. A historical source may identify the task, but writes always target today's
daily note and preserve the historical source. Use BIP's existing naming flow when an
eligible definition lacks an ID; cancel must leave both notes unchanged. Refuse prose
mentions and unsafe ambiguity.

Read authoritative live editor contents before disk. Differing open buffers refuse
instead of guessing. Capture the target identity and relevant task, daily,
configuration, and blocking facts; revalidate them before commit. A task that gained
dependencies, a schedule/project block, completion, or another identity while the card
was open must not be committed using stale facts. Revalidate the clock and day at
confirmation; no write crosses into a new day using yesterday's preview.

Apply one composed transaction per open note using the editor transaction helper,
preserving selection and undo. The daily note's entire snooze operation must undo with
one Ctrl+Z. If task and daily are the same path, emit one transaction for both. For
closed notes, use `vault.process` with a preimage comparison inside its callback. Across
two paths, check all inputs before writing; if the second required write fails, roll
back only while the first still equals the postimage this operation wrote. Report
rollback failure truthfully without overwriting concurrent user edits. Do not report
success with only the promotion or only the row installed. Existing type/property
settling must compose into the same transaction, not a later undo step.

After success refresh the existing Today/snoozed/task-index consumers. Use the review
capture/continue API once when the gesture resolves a captured landing. Keep duplicate
Enter/repeat presses from committing twice.

Expose read-only card facts from existing ledger-tools providers (add narrow versioned
adapters where absent): Bob-zone date/time, today's parsed Events, current snoozed row,
task effective type and clock classification, source Pomodoro name, and current on-clock
link count/cap. Add a pure destination projection following the parent's never-running
wake placement: untimed exact type, untimed containing type, otherwise a next-up
placeholder; no type uses the first untimed entry or the bare fallback. This projects
the footer only. Do not start `bob plan` or `bob snooze add` to preview or commit
snoozing. Missing preview data shows unavailable text, never invented zero counts or
destinations.

Wake now is the one exception: run `bob snooze wake TASK -q -f json` through the
existing bounded JSON spawn/decoder machinery with the correct vault. Pass argv, not a
shell-interpolated title or link. Keep bob as the sole wake writer; never implement a
link-now substitute (normal linking can target a running entry). Flush/settle or wait
for live dirty note buffers through existing supported save handling before handing the
same notes to an external writer; if this cannot be verified, refuse with a retry
message. Handle unsupported old bob, mobile, timeout, malformed JSON,
deferred/skipped/no-op and successful reports explicitly. Show the 8-second local wake
Notice only for nonempty `woken` with `notified: none`, then refresh. Other reports are
not success.

Test every context, same-path composition, open/closed notes, stale inputs, missing
providers, midnight, duplicate submit, second-write failure and guarded rollback, real
spawn argv/report handling, and no subprocess for add/retime/drop. Document the
controller/facts interface for the card phase.

## Phase card: picker, command, Task Card, and docs

Create a small Snooze Card using the filtered-picker/modal and shared close-key
machinery. Title: `⏰ Snooze “<task title>”`. Focus the one time field on open. Rank
future same-day suggestions in these groups: remaining event end times (chronological,
`15:30 · after Yogi interview`), `+30m`, `+1h`, `+2h`, then times used today.
Deduplicate resolved instants with the earlier group winning; discard
invalid/past/next-day candidates. Use Events' actual end-time facts, not a new parser or
an all-day/overnight time guessed to be today. Preselect the top suggestion, so Alt+Z
then Enter works without typing. If none exists, leave confirmation unavailable until a
valid time is entered.

Ctrl+N/P and arrows move; Tab completes the selection; Enter confirms the current valid
time or selected action. Typed invalid input never silently submits a suggestion.
Explain past/next-day input with `same-day only — Schedule (s:) defers to another day`;
reuse parser-specific DST teaching text. Keep focus, selection, IME, and repeat behavior
tested.

The selected-time footer states the actual consequence, for example:
`leaves WRITE · wakes into WRITE at 15:30 · in 2h 10m · TODAY 9/13 now, 10/13 after`.
Compute now from the proposed post-snooze ledger and after from this selected task's
subsequent wake, deduplicating identities and respecting off-clock classification. Do
not double-count an existing snoozed row or count closed history. Omit inapplicable
departure text. Over-cap is informative, never a refusal. Refresh
relative/clock-sensitive facts while open and revalidate on Enter; preserve authored
input during refresh.

Snoozed mode reads `Snoozed until 15:30 · in 2h` (or the truthful due/blocked state) and
adds distinct **Wake now** and **Unsnooze** rows. Retiming uses the same input. An
existing blocked row may be unsnoozed; retime/Wake now remain unavailable with the
blocking reason. Event/new Blocked/done/cancelled/non-task entry attempts teach why no
snooze can be made. Include the Event span or the known prerequisite/schedule/project
reason when available.

Esc, `q` outside a text control, and Ctrl+[ dispatch through one close/back route.
Preserve Task Card stage-return semantics; cancellation writes nothing. Use accessible
labels, keyboard selection, readable state words, light/dark theme tokens, reduced
motion and forced-colors support.

Register `bob-navigation-hotkeys:snooze-task` with Alt+Z only when it does not conflict
with an existing effective user/plugin binding or vimrc mapping. Inspect hotkeys.json,
the configured vimrc, and registered plugin defaults through existing app/vault access
at installation/loading; never rewrite those files or erase an explicit user unbinding.
On conflict keep the command and Task Card entry available, preserve the existing
binding, and give a concise notice naming the conflict. Test option-key handling on Mac,
including KeyZ where the platform supplies a different event.key.

Add `snooze` to the Task Card rows, order, shortcut labels, repeat guard, key resolver
and dispatcher (`z`); preserve Schedule as the default selected row. Feature-detect
compatible providers and expose a meaningful refusal when unavailable instead of
half-working controls. Prevent generic modal/global keys from double-dispatching the
action.

Update bob-plugins `docs/bob-navigation-hotkeys.md` and `docs/task-card.md`, and bob-cli
`docs/snooze.md` with the gestures, context support, single-task scope, fallback and
undo semantics. Read `docs.md` through the memory skill; regenerate inventory only with
its generator. No memory changes or outer close-out guide/glossary work. Run
model/keyboard/DOM/runtime integration tests and docs checks.

## Phase acceptance: application proof and deployment

Extend the isolated Obsidian runner pattern for the new card, with bounded startup,
owned-process cleanup and retained diagnostics on failure. Add an offline runner
self-test to normal unit tests. Exercise actual commands and keystrokes against fresh
fixture notes, not only direct planner functions.

Required proof:

- Alt+Z, Enter uses the first event suggestion; Task Card `z` opens the same surface.
  Typed/relative time, arrows/Ctrl+N/P, Tab, and all close aliases work.
- A live link's nested notes move once; a Ready task promotes/stamps; Next/WIP stays
  unchanged; same-note task/daily composition works. One daily-editor undo restores the
  entire daily preimage, including source link and row. Cross-note undo remains per
  editor and must be described honestly.
- Re-snooze a woken link, retime a row, Unsnooze without lane change, and snooze from
  closed history without changing history. Exercise open and closed daily notes and a
  target lacking an ID, including cancel during naming.
- Blocked/Event/done/cancelled targets, invalid/past/next-day times, stale edits, and
  midnight refuse without writes. Existing blocked rows can be removed.
- Wake now runs the current bob against the scratch vault, uses untimed placement even
  with a running matching entry, and refreshes the marks/card. Control notifications for
  the fixture; separately verify report-driven no-double-notice behavior with harness
  cases.
- Validate footer numbers for live-linked, unlinked, duplicate, snoozed, off-clock and
  over-cap tasks. Simulate an Alt+Z conflict and confirm the original binding and config
  bytes remain intact.
- Capture readable light/dark screenshots of ordinary and snoozed modes plus
  invalid/refusal and over-cap states. Check keyboard focus and forced-colors
  readability; retain screenshots and a structured result as audited artifacts.

Fix defects caused by this integration. Reuse the runner's setup and diagnostics instead
of broad changes to unrelated plugin acceptance infrastructure. Commands that require
waiting beyond the turn use `/sase_monitor`, with the exact next action and result
location supplied.

## Gates and land responsibilities

Every plugin-editing phase runs `npm run build`, focused tests, and the repo's required
`bob plugins sync` against its opened checkout (`--repo` and `--no-pull`). Apply the
deployment guard recovery documented in `docs/develop/deployment.md`; never force over
another agent's build. A phase must not close while its required verification/deploy
command is still running.

Final integration gates in bob-plugins: `npm run validate`, `npm test`, and
`npm run docs:check` with `BOB_CLI_DIR` pointing to the current bob-cli checkout; run
the real Obsidian acceptance above. In bob-cli run `just docs-check` and `just check`
with appropriate monitored completion. Do not rerun unchanged passing broad gates
without a reason. A reproducible clean-base failure is recorded as `PROPOSED FOLLOW-UP:`
with evidence (and an existing task ID if known), and does not leave this phase open
under the outer work instruction.

Record the final source revisions, vector results, test summaries, deployment result and
acceptance artifact refs on bob-cli-q5.6. Before closing each child, run
`sase bead epic-symbols <its-id>` and resolve/re-key leftovers to a genuinely open
owner. After all child phases and the child plan have settled through their normal host
lifecycle, the land agent runs `sase bead epic-symbols bob-cli-q5.6`, resolves any
remaining entries, and closes **only bob-cli-q5.6** with
`sase bead close bob-cli-q5.6 --note "<what was verified>"`. Never close the outer epic
or its other phases. Finish through `/sase_final`; use completion status only when all
of this assigned phase's work is actually done.
