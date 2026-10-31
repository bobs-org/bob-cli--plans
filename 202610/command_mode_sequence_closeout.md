---
tier: tale
title: Finish command-mode sequence UI and land bob-cli-er
goal:
  Command sequences work in the real DOM, preserve single-command behavior, show live
  history honestly, and bob-cli-er closes after verification.
size: medium
proposed_by: bbugyi200.athena.bob-cli-er.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-er
status: done
---

- **PARENT:**
  [202610/command_mode_sequences.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/command_mode_sequences.md)
- **BEAD:**
  [bob-cli-er](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-er/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-er.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-er.land.md)
- **COMMITS:**
  - [d81f778](https://github.com/bobs-org/bob-cli/commit/d81f778b66f81759fafb7829209ec5da2b99b493)
    — docs(guides): document sequence chip, summary, and rerun-last

# Finish command-mode sequences and land bob-cli-er

Repair the bounded defects found during landing, complete the missing documentation,
then close **bob-cli-er in this coding turn**. The four original phases are closed, but
the epic is not complete. This tale has no land agent; do not wait for its own commit,
push, SHA, or CI to close the epic.

## Context and fixed decisions

Read `sase bead read bob-cli-er -r "Need the landing audit and final decisions"` and the
original plan with
`sase artifact read plan:202610/command_mode_sequences.md "Need the approved sequence contract"`.
The lander already reviewed every note on er.1–er.4 and their commits. This plan covers
only the remaining work, not another implementation of the sequence engine.

All three decisions are final: **attention continues**, **Escape clears pending marks
before closing**, and **Rerun last command replays the whole stamped sequence**. Keep
catalog protocol 1, record schema 1 with an optional stamp, and `api.commandMode` v1.
There is no public sequence API to add and no memory change to make.

Use `/sase_repo` to open **bob-plugins**, read its `AGENTS.md`, and edit its source
fragments, never generated `main.js`. Build after edits. Each hand-edited fragment must
remain at most 1000 lines; `177-sheet-sequence.js` currently has 811. The main bob-cli
change is the guide. Read `docs.md` through `/sase_memory_read` before editing
documentation. Open the plans sidecar through `/sase_repo` when updating plan status.

## Verified baseline and integration

Audited bob-cli `3965f749` and bob-plugins `af35d48`; both matched fetched
`origin/master`. Epic commits are plugins `cdd61a4`, `a8f33a2`, `38da67d`, `af35d48` and
CLI `3965f749`. All 158 command-mode tests, plugin build:check, and validate (8/8) pass,
but their sheet DOM fake masks the defects below.

Audited evidence is available through `sase artifact read`:
`file:explicit:a0f49d2561ee189f6902ef4a` (production DOM and routing reproduction),
`file:explicit:e36d5e43c260c84cd22570ce` (full plugin test log), and
`file:explicit:c74e72c69664b05e05280be8` (isolated index performance log).

Reviewed intervening plugins commits `145ea37`, `458907b`, `4bb8821` (task wikilinks),
`c391a0d` (type ink), `4e31220` (Events caption), and CLI `4a95e5b7`, `e08f859f`,
`4f29a824`. These introduce no sequence callers or duplicate drivers. The Calendar stamp
JSON is additive; the existing command outcome adapter still consumes its original
fields. Capture grammar is outside this sheet's accepted free-text input. Recheck newer
commits at implementation time and integrate any actual overlap.

Full plugin baseline is 4544/4549 passing: cold-index chunk 17.0 ms versus 16, warm
snapshot/order 131.2 ms versus 50, keystroke rank 28.1 ms versus 12, and the two known
ledger startup assertions. Isolated unchanged index tests passed 3/5; snapshot 84.2 ms
and rank 16.0 ms still failed. These are not sequence defects. The lander's epic note
records all follow-up outcomes: startup proposals from er.1–er.4 corroborate
**bob-cli-eh**; index performance proposals from er.2–er.4 belong to causal active epic
**bob-cli-eq** (eq.3 introduced the test in `4ef761a`). The old `bob-cli-3w`
dependency-stage test is a different, fixed node. The unspecified transient fifth
failure in er.4 is declined as a separate task because it has no identified node or
reproduction. Do not repeat this triage or expand this tale to fix those test suites.

## 1. Make sheet rendering and clicks work in the real DOM

`177-sheet-sequence.js` wraps `170-sheet.js::renderRow`, but uses the test fake's
`el.classes`, `el._listeners.click`, and `el.children.find`. Real HTMLElements have none
of those properties. The base click listener remains registered, and the broad catch
hides rendering errors.

Headless Chrome loading the production bundle reproduces:

- A plain unmarked row click calls `activateRow` **twice**.
- Ctrl-click calls `activateRow` once as well as adding the mark.
- A row with pending marks still activates; its badge and tint are absent.
- A RECENT sequence row has no separator spans or wall-duration chip.

Use DOM APIs (`classList`, selectors or `Array.from(children)`) and one owned click
dispatch path. A small base-row event hook that the sequence layer overrides is
preferable to attempting to delete anonymous listeners. No production code may reach
into the test harness's listener storage. Ensure one ordinary activation per plain
click, zero activations for Ctrl/Meta-click or any click with pending marks, and correct
group toggling for RECENT sequence rows. Keep required-argument routing and refusal
hints. Restore badges, selected-plus-marked tint, separators, and exactly one duration
chip. Keep query selection and row focus behavior.

Add meaningful regression coverage using real Chromium or a faithful DOM adapter with
HTMLCollection children and native listener semantics, no public `classes` or
`_listeners`. Use the existing Chromium preview harness as a starting point; on this
host use `TMPDIR=/tmp` for Chrome's socket path. Exercise actual production render and
MouseEvent dispatch, including clicks on descendants. Extend the sheet preview to show
marked rows, tray, review card, and RECENT sequences in light and dark; inspect the
rendered frames. Do not merely add more assertions against ElementStub internals.

## 2. Preserve ordinary argument and single-mark behavior

Fix the argument branch of the outermost key handler in `177-sheet-sequence.js`:

- With **no marks**, Shift+Enter currently shows the sequence-preview warning and never
  calls `runArgumentStage`. Other Enter paths also take sequence handling. Keep Ctrl+M
  available, but delegate ordinary no-mark keys to the existing handler. Verify preview,
  run-now, completion acceptance, and Escape retain their behavior.
- With marks and a highlighted completion, Ctrl+Enter currently delegates to the old
  handler, runs only the typed command, and leaves the tray behind. Plain Enter still
  accepts a highlighted completion; Ctrl+Enter must add the complete command if missing
  and run the marked steps now. Without a selected completion, Enter adds and runs under
  review rules. Incomplete arguments must leave marks intact and show the missing
  positional. Shift+Enter warns only while marks are pending.
- Steps loaded by Ctrl+M on a RECENT sequence have `row: null`. If the user removes all
  but one, `cmSheetSequenceEnterRun` closes the sheet and directly calls the runner.
  This bypasses report, preview-first, and confirmation UI. Reconstruct a current
  actionable row and use the normal single-row activation chain, including the
  stale-entry argv fallback. The single remaining mark must behave identically whether
  loaded from history or marked in the catalog, and must never gain a sequence stamp.
  Enter opens the normal report/preview/confirm surface; Ctrl+Enter uses the existing
  force behavior.

Add regression tests that drive real key handling for these paths, including a RECENT
group loaded and reduced to one report, confirm, and preview-first command. Keep Caps
Lock/layout-independent Ctrl+M, Ctrl+[ aliasing, and the WeakSet guard.

## 3. Wire live sequence state into RECENT

`cmSheetRecentRows` accepts `options.activeSequenceIds`, but its only sheet caller in
`170-sheet.js::rebuildRows` never supplies them. With an actual running index-1 record
of a two-step sequence, the sheet says **Interrupted after step 1 of 2** and shows `◌`,
even though the plugin map holds the active id. Calling the model directly with that id
correctly returns `running`.

Pass the plugin's current active IDs through the actual sheet rebuilding path. Cover
opening/rebuilding the sheet while a sequence is running and between steps, then its
terminal or interrupted state. An active sequence must show `⟳`, occupy one RECENT slot,
and refuse a duplicate rerun without closing the sheet. After plugin reload, an
incomplete sequence with no active id must still say interrupted. Keep
newest-record-per-index resume folding and single-run history behavior unchanged.

## 4. Complete the approved documentation and verification

In bob-plugins `docs/bob-command-mode.md`, complete the original docs phase's observable
contract: a Sequences section after Command sheet; keyboard and mouse marking and
argument-mode rules; one-mark behavior; review, halting, Resume from k, Rerun all, Copy
joined by `&&`, and Output focus; RECENT grouping; the status chip's N/M, current title
and sequence wall clock; pager step label with `r`/`Y` remaining single-step; and
optional sequence record stamps with serial reads inside sequences. Correct the
unconditional Escape and Rerun-last sentences and distinguish ordinary parallel reads
from sequence ordering. Preserve generated sections and published URLs. In bob-cli
`docs/guides/run-bob-from-obsidian.md`, add the missing chip/summary guidance and fix
the later Rerun-last sentence that still describes only a single command. Keep this
concise and within the existing docs budget; do not edit mounted copies.

Run plugin `npm run build`, `npm run build:check`, all command-mode tests, `npm test`,
`npm run validate`, and `npm run docs:check`. Run bob-cli `just docs-check` and
**`just check`**. Do not run `just check-full`. Record the known unrelated failures
explicitly; a clean command-mode suite does not excuse a new failure caused by this
repair. Use `/sase_monitor` for long commands and give the continuation this plan's
remaining verification and closeout steps; never end with a live command.

Run `bob plugins sync` and verify `bob plugins list` reports bob-command-mode synced. Do
not force unrelated plugins over another workspace's protected deployment; record such
refusals separately. If a live Obsidian instance is reachable, exercise marking, review,
chip, summary, and RECENT there. Otherwise run the available startup smoke and real
Chromium UI coverage, and state the live-app limitation honestly.

## 5. Finish bob-cli-er landing in this turn

After the repairs and checks above, reread the epic and all descendants, ensure no new
notes or incomplete linked work remain, and update its verification note with the actual
results. Include the prior follow-up outcomes (bob-cli-eh corroboration, bob-cli-eq
performance routing, and declined unidentified er.4 transient). Resolve any new
epic-caused issue before closing.

1. Run `sase bead epic-symbols bob-cli-er`. Resolve every listed entry by wiring it,
   privatizing it, giving it a legitimate non-test pragma, or deleting it. Re-key only
   to a still-open later bead that genuinely needs the exemption. The audit currently
   found no entries. Follow the Symvision policy if any have appeared.
2. Close normally with `sase bead close bob-cli-er --note "<verification>"`. The note
   must state that all four phases, their notes and commits, all three decisions,
   concurrent drift, the real-DOM/argument/single-mark/RECENT repairs, documentation,
   actual checks and deployment, and all proposed follow-up outcomes were rechecked. If
   refused for stale epic symbols, clean them and retry. If a phase is incomplete,
   finish or reopen it. Never force a successful landing; use force with a canceled or
   superseded resolution only for a deliberate abandonment, not to bypass checks.
3. After closing, run `just symvision` if the recipe exists (currently absent in the
   primary justfile; record absence rather than inventing a pass).
4. Set `status: done` in the frontmatter of the original epic plan
   `plan:202610/command_mode_sequences.md`, file `202610/command_mode_sequences.md` in
   the plans sidecar opened through `/sase_repo`. Read artifacts through the audited
   command, and use only the path that repo open prints for writes.
5. Run `sase bead read bob-cli-er -r "Need the parent link"`. The audited epic has **no
   parent_bead**, so there is no ancestor to close. If the relationship has changed,
   follow the original landing prompt's normal parent-phase or parent-plan readiness
   procedure; never force through an incomplete parent.
6. Finish through `/sase_final`, declaring every changed repository for host commit,
   including bob-plugins, bob-cli, and the plans sidecar. Do not wait for this tale's
   own commit to appear; the host commits after the turn ends.
