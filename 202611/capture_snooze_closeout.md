---
tier: tale
title: Finish capture snooze parity and land bob-cli-q5.3.1
goal:
  Repair numeric-route drop previews and deduplicated completion aliases, verify reader
  integration, and close the capture epic and its parent phase.
size: medium
proposed_by: bbugyi200.athena.bob-cli-q5.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-q5.3.1
status: done
---

- **PARENT:**
  [202611/capture_snooze_token.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_snooze_token.md)
- **BEAD:**
  [bob-cli-q5.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-q5.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-q5.3.1.land.md)
- **COMMITS:**
  - [4e1ba6a](https://github.com/bobs-org/bob-cli--plans/commit/4e1ba6af01afcfed3456b0fd196e3bc12e6f45c1)
    — docs(plans): close capture snooze token and closeout tales

# Finish capture snooze parity and land bob-cli-q5.3.1

Complete the remaining capture work and the landing in one coding turn. This tale has no
subsequent land agent. The final step closes epic `bob-cli-q5.3.1`, marks its plan done,
and closes its parent phase `bob-cli-q5.3`. Leave outer epic `bob-cli-q5` open for its
waiting land agent. No closeout step depends on this turn's commit SHA, a push, or CI
for that commit.

## Audited baseline and scope

The landing audit used bob-cli `292d60c0`, also the fetched `origin/master`. Read the
landing notes on `bob-cli-q5.3.1` and the linked `plan:202611/capture_snooze_token.md`
with audited SASE commands. Its three phases `.1`, `.2`, and `.3` are closed. All twelve
child notes were reviewed; the grammar/mutation handoff is implemented, the transitional
execution guard is gone, and mutation results remain additive under schema 1 /
grammar 33. The relevant implementation commits are `1b386638`, `ae78dbb2`, and
`712d69d9`.

The audit read the parser/editor, capture dispatch and staged snooze adapter, core
add/drop/link-now planners, settlement/result refresh, completion candidate builder,
docs, and their tests. Existing coverage establishes staged eligibility and read guards,
atomic refusal, ID collisions, retime, note-preserving running-first link-now, sticky
lanes, same-note edits, and final row coordinates. Keep that behavior. Two specific
feature gaps remain below.

Intervening changes were reviewed from epic creation and the first epic commit: tie
handling and its later lint/docs changes, drawer cutover fixtures/docs, caret Go To
docs, planned-today readers (`b8678657`, `bbcae8e0`), snoozed plan forecasts
(`4a812173`), and settlement acceptance (`292d60c0`). The newer readers consume the same
canonical row identities and the existing capture settlement pass already runs before
snooze result coordinates refresh. Extend integration coverage using capture-created
rows instead of duplicating these readers or wiring disk-backed forecast code into
capture's staged budget.

This is bounded work for one agent: two local fixes and focused integration
verification. Do not redesign snooze, broaden the capture grammar, add CLI options, edit
package versions/CHANGELOG, or touch memory. Plugin and Mac presentation remain with the
outer epic's phases.

## Decisions and contracts

Preserve the approved outer choices: `uncommitted_rows = ignore`, `wake_notice = hud`,
`hold_in_meetings = yes`, `glossary_snoozed_task = yes`, and `decision_record = no`.
This tale needs no memory changes; glossary work belongs to the outer documentation
phase.

Governing records, read by the lander:

- `decisions:mac-capture-is-a-thin-client`: bob owns parse, completion, preview, and
  aggregate mutation. Preserve schema 1 and current additive fields.
- `decisions:capture-sigils-say-where-what-how` and
  `decisions:capture-reserves-clock-tags`: keep existing marker ownership and literal
  clock tags intact.
- `decisions:capture-link-drops-blocking-deps`: ordinary link-now keeps its existing
  behavior; snooze never removes dependencies to make a target eligible.
- `decisions:task-lanes-are-sticky`: drop preserves the task's lane and bytes.
- `decisions:consumption-keeps-notes-and-recorded-time`: retain nested notes, canonical
  identity matching, and recorded closed history.
- `decisions:today-is-read-from-the-ledger`: active snoozes remain absent from Today and
  the current link budget; forecasts are separate.
- `decisions:open-pomodoro-names-follow-task-links`: retain type agreement and final
  settlement before reporting coordinates.
- `decisions:event-buffers-widen-the-span`: completion uses event meeting ends, not
  buffered occupied ends.

## 1. Make numeric-route drop editor output agree with execution

`capture_language/snooze.rs::parse_snooze_drop_token` accepts the narrow
`@20261125:here!` spelling, and strict capture executes it. However,
`capture_language/editor_parse.rs` around line 5187 masks the bang and recursively
parses the remaining token through ordinary marker classification, which treats an
unescaped all-numeric route as prose. The result is `mode: task`, null route and block
ID, body `@20261125:here`, and only a bang snooze span.

Concrete evidence is `file:explicit:5bbffb8fc717828765ca25af` (read through
`sase artifact read`). The disposable fixture has `2026/20261125.md`, task
`- [*] #task Here ^here`, and active row `- **1500** [[20261125#^here]]`. With
`BOB_NOW='2026-11-25 12:00:00'`, the installed grammar-33 bob reports that incorrect
parse but `capture --dry-run` returns `snooze.action: drop` with the canonical
daily-file identity. This runtime evidence covers mutation/parser; the installed
executable predates completion, and the checkout's existing `target/debug/bob` is older
still. Build the current checkout for verification.

Fix the editor's claimed-drop path using its already-validated route, block ID, and
source range, or an equivalently narrow shared representation. Preserve original UTF-8
offsets; do not invent different-length masked input or globally relax route/block-ID
validators. Numeric ordinary prose must keep its existing meaning outside the narrow
claimed drop. Preserve alphabetic drop output, the `@route+id!` toggle, malformed-drop
diagnostics, and single-item restrictions.

Add regression coverage in `tests/cli/capture/parse_snooze.rs` and/or the shared grammar
tests: numeric basename drop and supported numeric path spellings; route/block
ID/mode/body agreement with strict parse and dry-run; exact route, block-ID and snooze
span slices; offsets after a Unicode preceding batch item; and a malformed/multi-item
near miss. Extend the existing numeric daily retime/drop fixture in
`tests/cli/capture/snooze.rs` where useful. Assert dry-run byte identity and real drop
preserving the task line.

## 2. Preserve relative query aliases when completion minutes deduplicate

In `capture_complete/snooze.rs`, event ends are inserted first, then relative
suggestions. `push_candidate` returns immediately when `seen` already contains the
resolved minute. Consequently an event ending at 13:00 at a fixed 12:00 clock keeps the
event-labelled 1300 row but discards the `+1h` filter alias; typing `z:+1` yields no
matching row. This is a current-source finding; do not mistake the old installed
binary's missing snooze completion for this defect.

Keep one candidate per minute, earliest-source label, literal four-digit replacement,
stable source ordering, and private/nonserialized filter aliases. Merge aliases into the
existing candidate when a later source resolves to its minute. Do not make query
filtering change which source owns the label.

Add a fixed-clock collision test in `src/native/capture_complete/tests/snooze.rs`: an
event end coincides with `+1h`; empty, `13`, `+1`, and `+1h` queries retain exactly one
event-labelled 1300 candidate. Include a duplicate used-today row and another distinct
event to prove ordering and no duplicate output. Accept that candidate through the
reported replacement range and confirm current `capture-parse` plus capture dry-run
accept it. Keep aliases absent from JSON. Update the completion paragraph in
`docs/capture.md` to state that relative prefixes still match a deduplicated
event-labelled row; use the docs memory workflow.

## 3. Verify integration with the newer readers and settlement

Use the existing isolated fixtures in `tests/cli/capture/snooze.rs`,
`tests/cli/plan_snoozed.rs`, `tests/cli/walks.rs`, and `tests/cli/settle_acceptance.rs`;
add only coverage needed for the boundary between their newly landed behaviors. Drive a
new-task or existing-task snooze through `bob capture`, then read the resulting fixture
with `bob plan` and the planned-today walk/freshness interface. Verify the active
identity appears once in the snoozed forecast, is absent from current Today/link counts
and review rows, and has the expected settled task/row coordinates. Follow with capture
retime and drop or ordinary running-first link-now; verify the readers see the
transition, nested notes survive link-now, and drop preserves the lane. Reuse the
settlement acceptance helper to establish no note-local reconcile drift from these
capture operations. Automatic wake remains never-running; reuse its existing core tests
rather than altering it.

Recheck any commits that landed after `292d60c0` and integrate only changes that affect
this scope. The capture `plan_budget` is a separate staged before/after contract; do not
add disk reads or replace it with `bob plan`'s report enrichment.

Run focused parser/completion and capture snooze tests, the relevant reader and
settlement tests, `just docs-gen` where required, `just docs-check`, and the file-change
gate `just check`. Never run `just check-full` without a new explicit user instruction.
Use `/sase_monitor` for long checks and finish its handoff command before ending the
turn. Any recovery continuation must carry the remaining closeout below; this tale has
no land agent.

The prior phase runs already established 98 unchanged-base failures, with owners below.
Fix any newly caused failure. If the same failures remain, compare specific names and
assertions to the recorded clean-base evidence; do not quietly declare the whole gate
green, weaken tests, or absorb unrelated epics. The approved capture plan allows
verified pre-existing failures to be recorded without holding this completed scope open.
A `just check` pass with a `just check-full` failure is test infrastructure work, not
capture work.

## Follow-ups already triaged by the lander

The durable `LAND FOLLOW-UP TRIAGE` note on `bob-cli-q5.3.1` records every proposal and
outcome. Do not create duplicate tasks:

| Proposing bead/note                                    | Outcome                                                                                                    |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `q5.3.1.1 #1` manual_contains                          | Resolved by `71f7b670`; new task declined.                                                                 |
| `q5.3.1.1 #2` 15 capture fixture failures              | DISCOVERED ISSUE corroboration on active `bob-cli-q4`.                                                     |
| `q5.3.1.1 #3` WISPr kicker                             | +1 on existing task `bob-cli-q9`.                                                                          |
| `q5.3.1.1 #3` agenda timing flake                      | +1 on existing task `bob-cli-gz`.                                                                          |
| `q5.3.1.2 #1` numeric drop preview                     | This tale's epic work.                                                                                     |
| `q5.3.1.2 #2` relative-time rounding wording           | DISCOVERED ISSUE on outer `bob-cli-q5`, matching `q5.1 #2`.                                                |
| `q5.3.1.2 #4`, `q5.3.1.3 #1` same 98 baseline failures | 80 drawer nodes to active `bob-cli-ph`, 17 settlement nodes to active `bob-cli-q4`, WISPr to `bob-cli-q9`. |

The shared parser rounds the duration, so 12:03 +45m is 12:48. Capture correctly reuses
that rule. The outer snooze epic owns reconciling the ambiguous shared wording/vector
contract; this tale must not introduce a capture-specific rule. No new task beads were
needed. Evidence relayed from phases is labelled as such.

## 4. Finish this epic and its parent phase in this turn

After fixes and verification, reread `bob-cli-q5.3.1` and its three children with
`sase bead read ... -r ...`; confirm all child notes and any new landing notes are
addressed. Inspect descendant readiness and linked plan state again. The epic's plan is
`plan:202611/capture_snooze_token.md`, at the workspace-neutral relative path
`sase/repos/plans/202611/capture_snooze_token.md`. Read it with `sase artifact read`;
open the plans repository through `/sase_repo` before editing, and use the printed
checkout path.

1. Run `sase bead epic-symbols bob-cli-q5.3.1`. Resolve every entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting the symbol per Symvision
   policy. Re-key only to a still-open later bead that needs it. The lander found none
   at `292d60c0`; recheck now.
2. Close normally with `sase bead close bob-cli-q5.3.1 --note "<verification>"`. The
   note must name the two repaired gaps, intervening reader/settlement integration
   checks, actual test outcomes and any attributed baseline failures, and all follow-up
   outcomes from the table above. If symbols block closure, finish cleanup and retry. If
   descendants are unfinished, complete or deliberately reopen them; never force a
   successful nested landing or force merely to pass the command.
3. Run `just symvision` if available. This baseline's justfile has no such recipe, so
   explicitly record unavailability if that is still true; do not claim a pass. Then set
   `status: done` in the frontmatter of the epic's linked
   `202611/capture_snooze_token.md` plan.
4. Run `sase bead read bob-cli-q5.3.1 -r "Need the parent link"` and
   `sase bead read bob-cli-q5.3 -r "Verify completed capture-z parent phase"`. The
   parent is the phase `bob-cli-q5.3`, not a plan ancestor. Verify its scope (z:
   new/existing captures, atomic row operations, teaching refusals, event-led
   completions, spans and docs) against the completed child plan. Run
   `sase bead epic-symbols bob-cli-q5.3`, resolve/re-key any entries, and close only
   that phase normally with
   `sase bead close bob-cli-q5.3 --note "<verified capture-z scope and checks>"`. Leave
   containing epic `bob-cli-q5` and its plan in progress. If new evidence makes the
   parent incomplete or ambiguous, record its specific blocker and stop ancestor closure
   rather than force it.
5. Finish through `/sase_final`, including the modified plans repository in the
   declaration. Code and closeout may land in this same turn; do not wait for this
   work's own commit, push, or CI before completing steps 1–4.
