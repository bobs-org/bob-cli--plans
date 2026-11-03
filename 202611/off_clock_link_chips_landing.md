---
tier: tale
title: Finish off-clock chip integration and land bob-cli-gf
goal:
  Ship a distinct combined plugin version, verify both ledger decorations together, and
  close bob-cli-gf with its plan marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-gf.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gf
create_time: 2026-11-03 12:01:10
status: wip
---

- **PARENT:**
  [202611/off_clock_link_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/off_clock_link_chips.md)
- **BEAD:**
  [bob-cli-gf](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gf/README.md)

# Finish off-clock chip integration and land bob-cli-gf

The off-clock chip epic is implemented. Finish the narrow integration gap with the Event
time-pill commit, then close **bob-cli-gf in this coding turn**. This tale has no land
agent or later landing continuation. Do not wait for this work's own commit, SHA, push,
or CI before closing the epic; the host commits after the turn ends.

## Verified scope and remaining gap

The land audit read every note on bob-cli-gf and its three closed phases, the approved
plan `plan:202611/off_clock_link_chips.md`, actual sources, and epic commits: bob-cli
`5f35f0b3` and `e9deb7f8`; bob-plugins `23d401b` and `46b25c6`. The approved choices are
final: **history_rows=yes, prose_links=no**. Preserve them. The contract is bob-cli
`docs/off-clock-link-chips.md`.

The lander's `just check` passed on bob-cli `e9deb7f8` (ToolRun
`cf4d4c7b732e2cbdc989d06117b2b26b`, exit 0, 699 seconds). It ran formatting, Clippy, all
Rust test binaries, and doctests; no check-full was run.

While landing, bob-plugins master gained `8ce2feb` (bob-cli-gg.2, Event link times). It
preserves both hosts in `ledgerMarkClass`, both cache-fill refresh callbacks, gf's
single source-cache freshness update, the renamed `--bob-off-clock-icon`, mixins,
exports, docs, and test registrations. Combined focused tests pass 241/241. However, its
manifest still says `1.96.0`, the same version gf already deployed; the epic
coordination note explicitly requires the second feature to take `1.97.0`. The EP3 unit
test also constructs a fake chip rather than running both production Reading sweeps.
Complete that integration coverage while correcting the version.

Unrelated commits since gf began were gg's display contract (`b9c482da`) and gh's
glossary edits (`2a9640a6`, `4ce79094`). The contract cross-links coexist correctly.
Glossary generation drift belongs to gh and is already recorded on that epic.

## Implementation

1. Open bob-plugins through `/sase_repo`, read its AGENTS.md, and fetch the latest
   master. Preserve any newer integrated work. Set bob-ledger-tools' manifest to
   **1.97.0** if it is still 1.96.0; if another commit has already made the required
   bump, preserve that version and avoid a second bump. No Cargo version or changelog
   edits. Regenerate `npm run docs:gen` inventory and build from fragments with
   `npm run build`; never hand-edit generated main.js.

2. Extend `scripts/test-ledger-tools-event-link-times.cjs` (or its off-clock sibling)
   with a meaningful combined-feature regression. Use an off-clock same-note Event Task
   and a dedicated ledger link. Invoke **both production Reading sweeps**, in both
   orders, then refresh again. Assert exactly one pill before the link and one chip
   after it, stable `pill, link, unlock, frees, chip` ordering when those marks are
   present, and that `progressMarkReadingContext` and the ring's ledger-row exclusion
   still recognize the row with both hosts. Exercise real per-link clock resolution for
   explicit #off, inherited offclock:true through [root::], and a #on override: the
   pill's off-clock state and chip presence must agree. Toggle either feature and verify
   the other survives. Reuse existing fixture/harness facilities; do not redesign
   rendering or repair unrelated baseline Reading issues. If this uncovers an
   integration regression, fix it within this tale.

3. Verify the integration and deploy. Run both feature suites plus the clock,
   gcal-events, frees, unblocked, progress, task-link-mark neighbors and the chip e2e
   self-test. The lander's combined command passed 241 tests before these edits:

   ```sh
   node --test scripts/test-ledger-tools-off-clock-link-chips.cjs scripts/test-ledger-tools-event-link-times.cjs scripts/test-ledger-tools-pomodoro-clock.cjs scripts/test-ledger-tools-gcal-events.cjs scripts/test-ledger-tools-frees-marks.cjs scripts/test-ledger-tools-unblocked-glyph.cjs scripts/test-ledger-tools-progress-marks.cjs scripts/test-ledger-tools-task-link-marks*.cjs scripts/test-obsidian-off-clock-link-chips-e2e.cjs
   ```

   Run `npm run build:check`, `npm run validate`, and `npm run docs:check`. For the
   latter, use the already opened bob-cli workspace through the docs script's
   `BOB_CLI_DIR` override instead of implicitly obtaining another checkout. Known
   generated glossary drift is documented below and is not a reason to expand this tale
   into gh work. Run `bob plugins sync --repo <opened-bob-plugins-path> --no-pull`
   without `-F`. Respect the guard: never overwrite another feature's
   uncommitted/non-ancestor deployment. Incorporate already committed master changes and
   retry safely; record any remaining external guard refusal rather than using force.

   If an isolated real-Obsidian host is available, rerun the existing off-clock e2e with
   both features installed and inspect the overlap row (pill before, off after, no new
   loss of marks). Never use the live vault for e2e. The original plan permits recording
   an unavailable Obsidian/xvfb run as skipped; the independent combined regression
   above must pass. Preserve explicit pre-existing Reading skips and their follow-up
   attribution rather than claiming they passed.

   File-change verification is `just check`, never `just check-full`. A check-full
   discrepancy is test infrastructure, not epic scope. Use `/sase_monitor` for long
   checks as required by the host. No step depends on this tale's own future commit.

## Follow-up triage already completed by the lander

These dispositions are recorded on bob-cli-gf; include them in its close note.

- **gf.2 note #2:** Five failures reproduced independently on clean bob-plugins
  `46b25c6`, 75/80 targeted tests pass. Added +1 evidence and the retained log to
  existing tasks **bob-cli-f1** (Chromium lookup), **bob-cli-et** (freshness reload date
  fixture), **bob-cli-eh** (note-ready and relation-index startup fakes), and
  **bob-cli-ff** (expired future schedule date). No duplicate tasks.
- The same note's base-only large-child performance failure was declined as new work:
  the phase's worktree passed, and existing flake bob-cli-ej covers that node.
- **gf.3 notes #2 and #3:** Accepted together as **bob-cli-gi**, a ready, large bug
  investigation for Reading ledger status rings and missing In Progress marks. Both call
  the shared ledger classifier, but the precise root cause remains open. These are
  baseline behavior, not introduced by gf. The 88/88 live run explicitly skipped these
  checks; do not describe those skips as passes.
- Plugin docs-check on current bob-cli `e9deb7f8` reports stale
  `docs/reference/glossary.md` after gh.1's glossary commits. A DISCOVERED ISSUE note
  was attached to **bob-cli-gh**. No duplicate task and no glossary edits in this tale.

## Final step: close bob-cli-gf in this turn

After the fixes and verification, reread
`sase bead read bob-cli-gf -r "Need final readiness and parent link"` and confirm all
descendants remain completed. This landing tale must count as completed for descendant
readiness; if it has a separate child bead, close that normally once its implementation
is complete before closing gf. If its assigned bead is gf itself, use the gf close
below. Never use --force to bypass a live child. Recheck drift since the audit and
preserve gg integration. Run `sase bead epic-symbols bob-cli-gf`. Resolve each listed
entry by wiring it up, privatizing it, adding an appropriate non-test pragma, deleting
it, or re-keying it only to an identified still-open later bead that genuinely needs the
exemption. The land audit found none.

Close with
`sase bead close bob-cli-gf --note "<verified approved decisions, original and integration commits reviewed, combined-feature checks and version/deploy outcome, and every follow-up disposition listed above>"`.
Never force merely to make close succeed. If named phases are incomplete, finish or
reopen them; a deliberate canceled or superseded result alone can justify --force with a
reason, never successful landing. Then run `just symvision` if available (the audit
found no recipe in bob-cli), and set `status: done` in the frontmatter of the epic plan
**plan:202611/off_clock_link_chips.md**: use `/sase_repo` to open `plans`, audited
`sase artifact read` for its contents, and update the `202611/off_clock_link_chips.md`
file in that returned plans checkout. Mark this tale done as its normal workflow
requires. No manual git commit; finish with `/sase_final` repository commit decisions.

The epic currently has **no parent_bead**, so there is no ancestor to close. Confirm
that after gf closes with `sase bead read bob-cli-gf -r "Need the parent link"`. If a
parent has since been added, follow the original landing instruction: close only a
completed parent phase normally and leave its containing epic; for a parent plan,
recheck its previous landing notes, every descendant/note, linked plan, post-child
drift, and readiness before normally closing it, retiring its epic symbols, checking
symvision, and marking its plan done. Stop at the first incomplete/ambiguous parent,
record its blocker, and report it. Do not wait for this tale's own commit or CI.
