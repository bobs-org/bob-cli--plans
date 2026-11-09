---
tier: tale
title: Finish sized-close acceptance and land bob-cli-ix
goal:
  Complete the sized-close Mac presentation, documentation, and execution evidence, then
  close bob-cli-ix and mark its plan done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ix.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ix
create_time: 2026-11-09 09:05:37
status: wip
---

- **PARENT:**
  [202611/sized_pomodoro_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/sized_pomodoro_close.md)
- **BEAD:**
  [bob-cli-ix](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ix/README.md)

# Finish the sized-close contract and land bob-cli-ix

Complete the remaining acceptance gaps in `bob-cli-ix` and close that epic in this
coding turn. This is one bounded repair, not a new feature or another epic. There is no
later land agent waiting for this tale: its coder owns the closeout below.

## Context and fixed decisions

Read `sase bead read bob-cli-ix -r "Need the landing audit and decisions"` and
`sase artifact read plan:202611/sized_pomodoro_close.md "Need the sized-close acceptance contract"`.
The four phases are closed; their implementation commits are bob-cli `070d2aac`,
`135793bf`, `5f361c5a`, and bob-mac-capture `8cc25b3`. The land audit read every child
note and inspected the implementation. Preserve the final choices:

- `chain_scope = front`: a sized close can follow only adjust/shift operators on its
  line. `x 2x faster` remains a close with log text; `= 2x` remains task text.
- `decision_record = no`: edit no memory files. Do not change release versions or
  CHANGELOG.md. Current grammar revision is 22, correctly advanced from the then current
  21; do not lower it to the original plan's stale 20.
- Bob alone owns grammar, timing, mutation and alias replacements. Swift presents Bob's
  fields and performs only the existing coarse send filtering.
- Sized timing is `5N` recorded minutes, with the ordinary sub-five-minute round-up
  allowance. All clock effects use `C = min(S + 5N, now)` with date-safe arithmetic, not
  invariably the recorded range end. Sized `2x0` closes and never resets.

Open bob-mac-capture with `/sase_repo` and use only its returned checkout; read its
AGENTS.md. Read `docs.md` through `/sase_memory_read` before documentation edits. No
live vault writes or installation are needed; all fixture capture runs use isolated
scratch vaults and a bob binary built from this checkout.

## 1. Finish the Mac close card and typing contract

In `Sources/CaptureCore/CapturePomodoroClosePresentation.swift` and
`Sources/BobMacCapture/CapturePanelView.swift`:

- When `summary.size` is present, render one `Nx · as of HH:MM` chip in the
  session-operator tint, replacing the early/on-time/over chip. The existing view
  currently renders both chips and uses secondary tint for size. Preserve unsized
  rendering and old payload decoding.
- Render `size.from.timeRange → closed.timeRange · Nm` for shorter and longer ranges,
  and the range plus duration without an arrow for equal size. The current equal-size
  view loses the duration because it displays `sessionPlannedText` and only appends
  duration inside `sessionChangeText`; ensure the visible card keeps it.
- Give sized cards the required accessible sentence, such as “Close BUILD at 10 minutes,
  2 blocks, 15 minutes shorter than planned.” Use “longer than planned” and “as planned”
  for the other cases, and retain useful task/destination information. Consume
  `minutes`, `blocks`, and `deltaMinutes`; do not recompute timing from clocks. Do not
  describe a rounded-up close as closing after wall time: the chip reads Bob's
  `closed_at`.
- Add focused presentation and view assertions for shrink, lengthen, equal size,
  round-up (`2x` at 09:08 displays “as of 09:08”), accessibility, and unsized/old Bob.
  Render the affected design fixtures and inspect that a single chip and the duration
  remain legible at the existing compact panel widths.

Extend `Scripts/generate-close-alias-fixtures.sh` with real-Bob sized rows: `2xx`,
`12xs`, `--1 2xx`, `^r:id=2xs`, and literal cases `Plan 2xx`, `x made 2xx`, `2Xx`. Keep
old-bob fixtures. Test the aliases through the existing intent/rewrite lane, preserving
size digits and native undo. The coarse Swift prefilter may send prose to Bob, but Bob
must return it unchanged; do not add a second grammar just to avoid that request.
Exercise the eligible prefilter positions and literal outcomes. Add a comma-assist
regression proving typing `12x` never produces `1,2x`, while selection digits after its
`x` retain normal comma assistance.

Record a same-size and a not-elapsed refusal payload from real
`bob capture --dry-run -f json`, alongside the existing shrink/lengthen fixtures. Extend
the applicable fixture harness/fake-bob routes, test that both refusal lines remain
visible and that refusal leaves the draft available for correction, and retain an old
payload without `size`. Record reproducible fixture commands/provenance.

## 2. Complete the promised Rust execution evidence

Strengthen `tests/cli/capture/pomodoro_close_size.rs` using the existing isolated vault
helpers and `BOB_NOW`. These are observable semantic tests, not assertions that merely
repeat implementation calculations. Fix any epic-caused defect they expose while
retaining the shared sizing helper and ordinary close planner.

- Compare actual task statuses, selections, carry and written ledger outcomes for `2x!2`
  versus `-3 x!2` at 09:12, and sized versus unsized equivalent selections. The existing
  equivalence test compares only the range. Assert `2x0` closes a timed row and defers
  every plain link, without accepting `in_progress` as an alternative or permitting an
  empty loop to prove the assertion.
- Add the promised two-meeting test: a 0900-1000 MEET with Design review 0900-0930 and
  Vendor sync 0930-1000, closed with `6x` at 10:15, must complete Design review and
  carry Vendor sync as `not_started`. Explicitly selecting the vendor's number completes
  it. Also prove the within-round-up case classifies at wall-time C rather than the
  future rounded range end. Check resulting tasks and ledger as well as reported
  classes.
- Exercise `^r:id=2x!1` and a newly created task with a sized close suffix, proving
  selection uses the post-link lineup, timestamps use C, and refusals roll back the new
  task/link as well as earlier batch items. The current composition test only checks
  `@bob:ready=2x` returned `ok`.
- Make `2x =` assert the exact closed and newly started spans, including wall-time start
  and no overlap for the grid example. Cover `--1 2x` and `2x x=fix~` with actual staged
  ranges/outcomes. Require success in the negative-tip sized-chain case instead of
  permitting any error as a passing result.
- Complete the multiple-running refusal case and make the unnamed-session assertion
  unconditional. Verify unchanged vault bytes for refusal paths, and the exact off-clock
  caption plus tip exclusion for shift, link close, further trimming, sized close and
  JSON. Keep existing unsized regression tests and omission of the additive `size`
  field.

Use the current code's selection/alias helpers and existing fixture setup. Avoid
production refactors unrelated to an observed failure. Run the focused Rust tests while
developing, then the canonical `just check` once the changes are ready. Never run
`just check-full`.

## 3. Finish documentation and account for intervening work

Update the actual Mac close-card owner (`docs/capture-previews.md`, with the relevant
links from `docs/capture-panel.md`) and `docs/keyboard.md` for sized timing, one chip,
`2xx`/`2xs`, literal prose, and the prefix/selection distinction. Link to
`https://bobs-org.github.io/capture/#closing-at-a-size`. Keep current Record Card
documentation intact; the original plan predates the preview-document split.

Correct two narrow bob-cli documentation ambiguities in `docs/capture.md`: C is the
earlier of the sized end and now, including the 09:08 round-up case, and the serialized
refusal's `close_size` is a top-level member (the internal `CaptureError.extra` map is
flattened into the JSON object). Do not change the working error wire format to match
the internal field name.

The lander fetched bob-cli master `7fe12b33` and Mac master `8cc25b3`. It reviewed
record-update fixes `941ee724`, Templater guard `c28edbee`, Similar records `44427b30`,
records init `262f3cfc`, calendar skips `a449193b`, task-loss tripwire `7fe12b33`, and
Mac Record Card integration `c09e2bc`. No additional sizing integration was needed in
those paths. Recheck later drift and preserve the shared parser mode routing and
record-card keyboard ownership when repairing the Mac view.

Follow-up triage is already complete and recorded on bob-cli-ix:

- ix.1 notes #1/#2: duplicate grammar-revision proposals resolved by 21 → 22; declined
  as new tasks.
- ix.3 note #1: `root_help_matches_sectioned_short_and_long_snapshots` fixture drift was
  introduced by `44427b30` in the active Record Notes epic. Searches and the one-week
  task sweep found no duplicate. The lander recorded a `DISCOVERED ISSUE:` on bob-cli-ir
  with attribution to ix.3; no standalone task. If the canonical check still fails
  solely for that known root-long row, coordinate via that epic note and refresh the one
  stale fixture row so the shared gate can pass, recording the repair there. Do not turn
  it into new sized-close work or create a duplicate task.
- All card/test/doc gaps above are caused by this epic and stay in this repair.

## 4. Verify the final tree, then finish this epic's landing

Run `just check` for bob-cli changes and `just docs-check` for both affected doc trees.
For Swift, follow bob-mac-capture AGENTS.md: `just portable-check` (use athena if this
host lacks Swift), compile an isolated scratch copy on the Mac with `bundle.sh` and
`just typecheck-tests`, then use `Scripts/macos-ci-check` to gate the worktree before
landing. Read tailnet memory if using those hosts, never touch Bryan's checkout or
install there, and delete scratch copies. Read any non-project repository with
`/sase_repo`. Treat missing Mac compilation as unfinished work.

Use `/sase_monitor` for long checks as required by SASE; its successor must resume this
same tale through the closeout. The existing ix.4 pushed-SHA run is
`https://github.com/bobs-org/bob-mac-capture/actions/runs/42629185109` for
`8cc25b315086b4bd7fce2725f82a279ceb07a8da` (still running during the land audit). Record
its eventual result; a successful precommit worktree gate covers the repaired Swift. Do
not make epic closeout wait for this tale's own final commit, push, SHA, or post-push
CI. Do not make an early final-code push solely to create such a dependency. Closing the
epic with verified, uncommitted final code in this turn is the required ordering; the
host finalizer commits afterward. If a later push occurs, its exact-SHA CI monitoring
remains required, separate from this closeout dependency.

The final implementation step is this concrete closeout, not a handback to a lander:

1. Re-read bob-cli-ix and all descendants/notes. Confirm their accepted scope plus this
   repair is complete, review new drift, and inspect linked-plan readiness. Set any
   already-complete descendant plan files to done where appropriate before closing.
   Re-read the epic parent link; the audit found **no parent_bead**.
2. Run `sase bead epic-symbols bob-cli-ix`. Resolve every listed symbol by wiring it,
   privatizing it, adding a justified non-test pragma, or deleting it per the Symvision
   policy. Re-key a Justfile exemption only to a still-open later bead that actually
   needs it. The audit found no entries, but recheck now.
3. Run
   `sase bead close bob-cli-ix --note "<verified parser/execution/docs/Mac contract, exact verification results, intervening-commit integration and the follow-up outcomes above>"`.
   If rejected for symbols, clean them up and retry. If named phases remain incomplete,
   finish/reopen them. Never force a successful landing; use
   `--force --reason ... --resolution canceled|superseded` only for an actual deliberate
   abandonment, never to bypass unfinished work.
4. Run `just symvision` if available; otherwise record that this repository has no such
   recipe. Open the plans sidecar with `/sase_repo`, audited-read the original plan
   above, then change only its frontmatter `status` to `done`. The logical path is
   `plan:202611/sized_pomodoro_close.md`; use the path resolved in this coder's
   workspace, never a previous agent's path. Include the plans repository in the final
   declaration if modified. With no parent bead, no ancestor close is needed. If the
   link has changed, follow the parent-close rules in the lander prompt and record any
   incomplete ancestor without forcing it.
5. End through `/sase_final`, declaring every edited repository and reporting the epic
   closed. Do not stop after tests or leave the epic plan in progress.
