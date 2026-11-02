---
tier: tale
title: Repair MEET continuation anchors and successor targeting, then land bob-cli-fx
goal: MEET continuations stay outside fences, preserve block spacing, receive their
  own successors, and bob-cli-fx closes with verified fixture parity.
size: medium
proposed_by: bbugyi200.athena.bob-cli-fx.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fx
status: done
---

- **PARENT:**
  [202611/meet_continuation_goes_last.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/meet_continuation_goes_last.md)
- **BEAD:**
  [bob-cli-fx](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fx/README.md)

# Finish MEET continuation placement and land bob-cli-fx

## Scope and fixed decisions

Complete the remaining defects found while landing epic **bob-cli-fx**, then close that
epic in this same coding turn. The approved plan is
`plan:202611/meet_continuation_goes_last.md`. Both phases (`bob-cli-fx.1` and
`bob-cli-fx.2`) are closed, but the landing audit found the gaps below. This is one
bounded plugin repair, fixture refresh, and verification/closeout, not a new feature.

Preserve the approved `close_card_says_last = yes`: Rust JSON and human output and the
Mac close card keep their placed-last reporting. Rust's continuation placement is the
reference: a close-created MEET goes after the last real open Pomodoro block below the
closing entry, **before that block's trailing blanks**. It never anchors on fenced
examples or entries above the close. Interior blanks and nested children belong to the
block. With no anchor, and for other names, preserve existing bytes. Do not change close
selection, clock rules, carry order, unnamed-start selection, successor offers, or
successor fallback case 3 (directly below the closed entry). An already-open MEET
remains separate from the new continuation.

Open `bob-plugins` using `/sase_repo` and read its AGENTS.md before changes. Open
`bob-mac-capture` the same way only as needed for verification; its approved Swift
implementation already landed in `ac6c3d6`. Read artifacts via `sase artifact read`, and
read reference memories through `/sase_memory_read`. No memory edits or new CLI options
are planned. Never edit generated plugin main.js directly.

## Landing evidence already established

The lander read the epic, every child and every note, the approved plan, and these
implementation commits and their source/tests:

- bob-cli `d32da8fa`, `876ae775`: placement, reported line, JSON/human output, eight
  unit tests, six CLI tests, shared vectors and docs.
- bob-plugins `1115a14`: planner/cursor port, generated source, manifest release,
  completion tests and event-task-close vector mirror.
- bob-mac-capture `ac6c3d6`: compatible decoding, close-card caption and two tests.

Fetched bases matched bob-cli `876ae775`, plugins `1115a14`, and Mac `a58e4f0`. Reviewed
changes since creation as well as after the first epic commit: gcal description
parsing/filtering is independent; default merge (`50887216`, Mac `a58e4f0`) scans
current ledger order and skips MEET; Stats day-strip, capture undo, and blocked-mark
work are independent. The concurrent relation-successor repairs (`4502b73`, `ffe1727`)
share the plugin successor pass and must be retained while repairing continuation
targeting. Recheck any subsequent drift before closeout.

No new `just check` pass is claimed by the lander. A focused plugin run passed 138/143
tests: the 31 Pomodoro-completion tests passed and five failures matched existing
infrastructure tasks. A separate performance run failed two existing CPU gates.
`just docs-check` passed all implemented gates, including 447 docs unit tests with one
optional font-dependency skip. Current epic-symbol audit is empty; there is no symvision
recipe. Repeat the symbol audit at closeout. Placement probes and fixture/documentation
findings are preserved in `file:explicit:09e9d220720061d1919e7c2e`.

## 1. Fix the plugin's new placement anchor

In `plugins/task-status-cycler/src/060a-pomodoro-placement.js`,
`getPomodoroContinuationPlacement` currently calls `isPomodoroTaskLine`, which only
checks section bounds and column-zero task syntax. It does not exclude fences. Reuse the
existing `getFencedLineNumbers` policy when selecting both first and last open anchors.
Keep section bounds, real Pomodoro syntax, open status, and the normalized MEET-name
check. Do not broaden unrelated parsers just to fix this path.

Reproduction (one line per array element, zero-based):

````text
## Pomodoros
- [ ] (**0800-0815**) — MEET
  - [[#^later]]#
- [ ] () — BUILD
```md
- [ ] () — FAKE
```
````

The current completion plan inserts at line 6, inside the fence, hiding the carried
meeting as code. It must insert before the fence, after BUILD. Also cover only fenced
candidate entries (no valid later anchor), tilde fences, and indented fake headlines.
Keep the cursor on the first real open entry below the close.

`getFullPomodoroEntryEndLine` includes trailing blank lines, unlike Rust's
`capture_pomodoros::pomodoro_block_range`. Trim trailing blank lines locally for the new
MEET anchor (or introduce a narrowly scoped helper), without changing existing
full-block ownership elsewhere. For a final PLAN with an indented note, a blank, then a
completed DONE row, insert immediately after the note and before the blank. Retain
interior blanks and nested content. Test exact resulting text and cursor coordinates,
including a deferred-only close that removes the old headline.

## 2. Make successor targeting retain the created continuation's identity

The placement change invalidates two old adjacency assumptions:

- In `src/135-plugin-successors.js`, `relocateSuccessorClosingEntry` resolves a
  deferred-only close by the first matching `createdLineText` at/after the old line.
- In `src/137-plugin-successor-project.js`, the normal close path resolves
  `createdEntry` by the first later open same-name entry.

Both choose an older MEET when one already existed below the close. The new continuation
now sits after it. Confirmed deferred-only probe: old MEET resolves at post-edit line 1
while the created MEET is line 4. A close retaining a note resolves the older MEET at
line 3 instead of the created MEET at line 6.

Carry sufficient creation identity from `buildPomodoroCompletionPlan` through the hint
constructed in `src/170-plugin-pomodoro.js`, then consume it in both relocation paths.
Account for removed rows, Work Log writes, and note edits that shift lines; do not
substitute a bare first-match or last-match name heuristic. Prefer existing
edit-coordinate and snapshot/relocation machinery; keep failure conservative if the
identified block cannot be safely resolved. A created continuation must beat the
same-name reuse fallback even when its headline matches another placeholder.

Add actual successor-wiring/editor tests for both deferred-only and retained
closed-entry cases with an older MEET plus BUILD/PLAN below. Assert that an explicit
successor pick lands under the newly created final MEET, the older MEET is unchanged,
the cursor remains on first planned work, and `next_up` reflects the actual post-write
order. Include a line-shifting Work Log/recovery write. Preserve both dependency and
relation offers in the shared path, and keep case 3 directly below the closed entry. Do
not consider a relocation-helper-only test sufficient evidence of final placement.

## 3. Refresh the remaining shared fixture and verify integration

Copy bob-cli `tests/fixtures/pomodoro_clock/vectors.json` byte-for-byte to
`bob-plugins/plugins/bob-ledger-tools/scripts/fixtures/pomodoro_clock/vectors.json`. The
only current difference is `close-defers-not-started-off-clock.expected_day`: the plugin
copy still puts MEET before PLAN. This is phase `bob-cli-fx.1` note #2, left unresolved
by phase 2. Update the local fixture provenance README if needed. Keep the
already-identical event-task-close vector copies identical too.

Add focused parity/regression tests for the placement cases above rather than testing
private helper names. Recheck the CLI's moved-line reporting, preview Next block, `x =`
normal and back-to-back starts, and successor target. Cover the newer default merge
through a narrow integration scenario: after a close and start, bare `=*` consumes the
next ordinary planned session while the deferred MEET remains separate. Use the existing
CLI fixture helpers; fix only a demonstrated integration defect if that scenario exposes
one. Link/task close forms must continue through the same ledger planner and report the
moved continuation.

Keep hand-edited fragments at most 1000 lines; extract helpers/register fragments as
necessary. Bump the cycler manifest patch version using the repo convention, regenerate
source and inventory, and update only documentation needed to describe the corrected
rule. Read `docs.md` memory before docs changes. Never edit the bob-cli Cargo.toml
version or CHANGELOG.md.

Run the focused completion, event-clock/vector, successor-wiring/placement and
relation-offer suites, `npm run build:check`, the plugin's `npm test` and
`npm run docs:check`, and `bob plugins sync`. Inspect any deploy conflict rather than
overwriting another agent's unpublished work blindly. Run **`just check`** in bob-cli
and `just docs-check` against the assembled sources. Do not run `just check-full`. Use
`/sase_monitor` if required for a long command and carry the remaining closeout in its
next instructions. Use `just portable-check` to verify the existing Mac close-card
behavior if verification is needed; compile any Swift you change under that repository's
instructions. No Swift change is expected.

## Follow-up dispositions to carry into the close note

The lander completed `/sase_new_task` search, last-week all-status sweep, and active
epic causal review. No new task was warranted:

- `bob-cli-fx.1` #1 and `bob-cli-fx.2` #2 (docs cap): declined as new tasks because
  already resolved by Mac `a58e4f0`. The prior "pre-existing" attribution was wrong:
  canonical word counts are `ac6c3d6^` 5999, `ac6c3d6` 6023, current HEAD 5997. The epic
  added the excess; newer merge-card documentation trimmed it, and the lander's full
  docs-check passed. Preserve the newer trim and caption.
- `bob-cli-fx.1` #2 (clock fixture mirror): epic-caused remaining work, handled by step
  3, not a separate task. The event-task-close mirror was already fixed.
- `bob-cli-fx.2` #1: reproduced and corroborated existing tasks `bob-cli-et` (real-date
  freshness reload), `bob-cli-f7` (UTC/local feedback), `bob-cli-eh` (both startup
  callback-fake failures), and `bob-cli-ff` (expired future schedule). Evidence:
  `file:explicit:0201e20ab57a269a1d560ad4`.
- `bob-cli-fx.2` #3 performance observation: corroborated existing `bob-cli-f8`. Current
  snapshot/order 81.0 ms exceeds 50 ms; rank 13.4 ms exceeds 12 ms. A phase standalone
  pass does not establish a load-only flake. Evidence:
  `file:explicit:dd79208b8c6d14c0a47fb839`.
- The new fence, trailing-blank, and duplicate-MEET targeting defects belong to
  bob-cli-fx and must be fixed here, never deferred as unrelated follow-ups.

## 4. Finish the epic landing in this coding turn

This tale has no land agent. The coder must perform this entire final step after
verification, **before its turn ends**, without waiting for its own commit SHA, push, or
CI. The host commits only after the turn ends; closing the epic with the final verified
code still in the working tree is normal.

1. Re-read `bob-cli-fx` and both children with `sase bead read ... -r ...`, all notes,
   and `plan:202611/meet_continuation_goes_last.md` with `sase artifact read`. Confirm
   every approved requirement, the defects above, and any post-audit drift are resolved.
   Both phases must remain completed; do not force a successful close.
2. Run `sase bead epic-symbols bob-cli-fx`. Resolve every entry by wiring, privatizing,
   legitimate non-test pragma, or deletion; re-key only when a specific still-open later
   bead needs it. Do not leave stale fx/phase exemptions.
3. Close normally:
   `sase bead close bob-cli-fx --note "<verification of both phases, placement and successor repairs, integration drift, exact passing checks and known external failures, and every follow-up disposition above>"`.
   If refused, finish the named work and retry. Never use `--force` merely to pass;
   never use it to advance a successful nested landing.
4. Run `just symvision` if available (none at audit), confirming the whitelist. Open the
   plans sidecar with `/sase_repo`, use the PLAN path returned by the bead read, and set
   **`status: done`** in the frontmatter of `202611/meet_continuation_goes_last.md`.
   Read that sidecar artifact only through `sase artifact read`. The audit found **no
   parent bead** for bob-cli-fx; recheck with
   `sase bead read bob-cli-fx -r "Need the parent link"` and finish normally if it
   remains absent. If a parent was added, apply the original landing prompt's
   phase-versus-plan parent readiness and normal-close rules, stopping with a note at
   the first incomplete or ambiguous ancestor.
5. Finish with `/sase_final`, declaring every repository changed (including plugins and
   the plans sidecar). Report the completed epic and verification concisely.
