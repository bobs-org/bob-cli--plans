---
tier: tale
title: Finish tucked-group counts and migration verification, then land bob-cli-ap
goal: Correct grouped-highlight disclosure counts, prove legacy ID migration, finish
  the documentation sweep, and close bob-cli-ap after verification.
size: small
proposed_by: bbugyi200.athena.bob-cli-ap.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ap
status: done
---

- **PARENT:**
  [202610/tucked_continuation_groups.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/tucked_continuation_groups.md)
- **BEAD:**
  [bob-cli-ap](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ap/README.md)

# Finish and land tucked continuation groups (bob-cli-ap)

## Outcome

Finish the remaining defects and verification gaps in epic `bob-cli-ap`, then close that
epic and mark `plan:202610/tucked_continuation_groups.md` done in this same
implementation turn. One agent can implement this bounded work directly. There is no
later land agent for this tale, and closeout must not wait for this turn's commit, SHA,
push, or CI run: the host commits after the turn ends.

## Reviewed context and constraints

Read the epic and its current notes with
`sase bead read bob-cli-ap -r "Need the landing findings and decisions"`, and read the
approved design with
`sase artifact read plan:202610/tucked_continuation_groups.md "Need the approved group contract"`.
Use `/sase_repo` to open `bob-mac-capture` before touching its files, and follow its
`AGENTS.md`. Read SASE bead and documentation memory through `/sase_memory_read`.

The landing review already read all three children and every note. `bob-cli-ap.1`, `.2`,
and `.3` are closed. Their implementation commits are bob-cli `5376dc6e` (renderer,
parser, identity, snapshots, CSS/docs), `f2c24531` (`ref show` pieces), and
bob-mac-capture `da5faa8` (inspector, Copy, accessibility). Preserve this work.

The approved decisions remain final: implement the Mac inspector; install the CSS; do
not edit any memory. CSS installation is verified independently: the published vault
snippet at vault commit `87191475` matches the full documented CSS block byte-for-byte,
SHA256 `1edafd1f155374c59f5499df7fa7ca5ac7c8bbffe8df13264fb30b5979bdf62a`. No further
vault writes are needed. Phase `.1` records the four real-note scratch migrations,
unchanged block-ID sets and Tasks, no removed-highlight warnings, and idempotent second
syncs. The phase notes record successful CLI and Mac gates, including the Mac bundle and
test typechecks; this tale must verify its own changes.

Integration review fetched bob-cli `origin/master` and found only the two epic commits
since the epic began. In bob-mac-capture, intervening commit `8160db3` changes Refs
reset from Ctrl-Z to Ctrl-C. Its router/model/docs changes are already under `da5faa8`,
preserve group rendering/Copy behavior, and need no adapter. Refresh both bases and
inspect any newer drift before closeout.

Follow-up triage is finished and recorded on `bob-cli-ap`: none of the child notes
contains a `PROPOSED FOLLOW-UP`, and no distinct unrelated follow-up was identified. The
findings below are epic work, not task beads. At review time the epic has no
`parent_bead`, `sase bead epic-symbols bob-cli-ap` is empty, and neither source repo
defines a `symvision` recipe.

## 1. Fix entry counts in the Mac disclosure and header accessibility

In `Sources/RefsCore/RefsInspector.swift`, the new `noteCardCount` correctly counts each
piece's note card for the header chip. However, `hiddenNotedCount(visible:)` now sums
those card counts, although its text says how many hidden **entries** have notes.
`headerAccessibilityLabel` also reuses the entry-based `notedPhrase` with the card
count.

The lander executed the actual Swift model with one group containing two commented
pieces. It produced:

```text
total=1, noteCards=2
+1 more · 2 with notes · ⌃D shows it
1 more highlight, 2 with notes
Highlights, 1, 2 with notes
```

Restore the hidden noted count to the number of hidden entries whose `hasNoteCard` is
true. Keep `feed.total == 1`, paging/reveal increments of one, `feed.notedCount == 2`,
and the visual header chip's two-card count. Make the header VoiceOver wording accurate
for multiple note cards within one entry: describe the cards as cards/notes, or use a
separately computed entry count with the existing “with notes” wording. Preserve
ordinary single-entry behavior where possible; do not lower the chip's card count to fix
the prose.

Add regression assertions in `Tests/RefsCoreTests/RefsInspectorTests.swift` using one
group with two commented pieces, both entirely hidden and after a visible single. Expect
one hidden highlight and one hidden entry with a note; ensure the header describes its
units truthfully and the chip still counts two cards. Cover clamped visibility/reveal
behavior through existing APIs. Clarify the distinction in the source-owned
`docs/refs-inspector.md` if needed. The existing documentation already says disclosure
counts hidden entries while the header counts cards.

This fix should stay in portable RefsCore and its tests. If an AppKit source or test
must change, follow the repo's best-effort scratch Mac compilation procedure
(`bundle.sh` and `just typecheck-tests`); never install the scratch app.

## 2. Make the legacy migration regression exercise the legacy matcher

In bob-cli `src/native/highlights_ref/tests/continuations.rs`,
`legacy_fused_group_migrates_without_id_churn_or_tombstones` uses an abbreviated quote
(`One last Crew tip: … PR Sheriff…`) which does not match the full `GAS_TOWN_SIDECAR`
text. It also uses the current generated head ID. This cannot demonstrate preservation
through the exact `legacy_full_text` matching pass.

Replace the abbreviated fixture with literal, exact output of the pre-epic renderer from
`5376dc6e^` for the matching sidecar. Inspect that revision locally; do not synthesize
the expected legacy bytes using the current rendering helper inside the test. Keep the
follower redirect stub and assert the complete ID set, no removed-highlight warnings,
correct reworded stub, and byte-identical second render. The current exact tucked
gas-town golden remains unchanged.

Add a focused historical hyphen-seam case, such as a head ending in `inter-` and a
next-page follower starting `national`, whose old fused quote was healed to
`international`. Give the old group a valid non-generated `h-` ID, so regenerating the
current deterministic ID cannot accidentally satisfy the assertion. The old fused quote
does not start with the complete head text plus whitespace; preserving that ID must
therefore exercise `legacy_full_text`, not the later prefix fallback. Assert the new
quote keeps `inter-` and `national` as separate pieces, retains the old ID without a
tombstone, and is stable on a second render. No production identity change is expected;
if this exposes one, fix it within this epic.

In `src/native/highlights_ref/continuation.rs`, document that `assemble_group_quote` and
the legacy seam rules remain only to recognize pre-tucked identities. Do not remove them
or use them for current display. Keep accepted `<…`/`<...` grammar.

## 3. Finish the required documentation sweep and verify

In `docs/highlights-ref-sync.md`, change the introduction's “a `<` note folds a
highlight into the one before it” to grouping language, and change
`notes[].continuations_dangling`'s “could not merge” description to “could not form a
group.” Sweep this document for remaining obsolete continuation-fusion claims. Underline
merging is still real and its terminology must remain intact.

Run the focused Rust continuation tests and Swift regression tests, then bob-cli
`just check`, bob-mac-capture `just portable-check`, and the applicable
`just docs-check` gates. The latter validates mounted source-owned Mac docs. Do not run
`just check-full`. A check-full-only failure is infrastructure work, not unfinished epic
behavior.

The landing review found `sase tool run check` unavailable (`unknown tool 'check'`). If
that remains true, use the documented `SASE_TOOL_BYPASS` fallback with a reason and
`just check`. Follow `/sase_monitor` for long commands, with a continuation explicitly
requiring the remaining closeout below; do not end a turn with an unobserved running
command. No full verification gate was run in the planning turn, so the previous phase
results do not replace these gates.

## 4. Complete this epic's landing in the implementation turn

After the corrections and required checks pass:

1. Re-read `bob-cli-ap`, all descendants and their notes, and the linked approved plan.
   Confirm all three original children are still closed, the findings in the landing
   notes are addressed, and any new drift is integrated. Review every new
   `PROPOSED FOLLOW-UP`; use `/sase_new_task` only for genuinely distinct issues not
   caused by the epic. Record each outcome, including declined proposals, on the epic
   and in its close note. Carry forward the already-recorded zero-proposal triage and
   the verified CSS decision.
2. Run `sase bead epic-symbols bob-cli-ap`. Resolve every listed exemption by wiring it
   up, privatizing it, adding a justified non-test pragma, or deleting it under
   Symvision policy. Re-key only when a still-open later bead actually needs the
   exemption. Do not leave stale entries for another agent.
3. Close normally with
   `sase bead close bob-cli-ap --note "<verified child work, commits and integration, corrected disclosure counts and migration coverage, gate results, CSS decision, and follow-up outcomes>"`.
   Use actual results in the note. If rejected for exemptions, clean them and retry. If
   rejected for unfinished named phases, finish or reopen the required work. Never use
   `--force` merely to succeed or to advance a successful nested landing;
   cancellation/supersession requires a deliberate reason and non-done resolution.
4. After closing, run `just symvision` if the recipe is available, and report the
   absence explicitly if it still is not. Use `/sase_repo` to open `plans` before
   changing its files. The epic's PLAN is `plan:202610/tucked_continuation_groups.md`;
   within the path returned for `plans`, set `status: done` in
   `202610/tucked_continuation_groups.md` frontmatter. Use `sase artifact read` for its
   content rather than unaudited artifact reads.
5. Run `sase bead read bob-cli-ap -r "Need the parent link"` after closure. There was no
   parent at the landing review, so normally stop ancestor processing here. If a parent
   was added meanwhile, follow the original landing rules: a phase parent closes only
   after its scope is verified, leaving its containing epic for its waiting lander; a
   plan parent requires review of its prior landing note, descendants/notes, linked
   plan, and post-child drift before normal closure, symbol cleanup, available
   Symvision, and plan status update. Repeat only through complete directly parented
   plans; note and report the first ambiguous/incomplete parent. Never force a
   successful nested landing.
6. Submit `/sase_final` as the last action of the completed turn. Include every repo
   edited by this tale in its commit declaration, including the plans sidecar. Do not
   wait for this turn's future commit or CI to close the epic.
