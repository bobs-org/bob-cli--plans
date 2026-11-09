---
tier: tale
title: Finish modifier-alias escapes and land bob-cli-j6
goal:
  Restore aliases throughout picker opening, repair fixture and documentation drift,
  verify the integrated feature, and close bob-cli-j6.
size: medium
proposed_by: bbugyi200.athena.bob-cli-j6.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j6
create_time: 2026-11-09 13:34:45
status: wip
---

- **PARENT:**
  [202611/dot_semicolon_modifier_aliases.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/dot_semicolon_modifier_aliases.md)
- **BEAD:**
  [bob-cli-j6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j6/README.md)

# Finish modifier-alias escapes and land bob-cli-j6

## Outcome

Finish the remaining Mac escape, fixture, documentation, and integration gaps in
`bob-cli-j6`, verify the combined implementation, and close that epic. This tale's coder
owns the landing; there is no later land agent for this tale.

Read `sase bead read bob-cli-j6 -r "Need landing audit and approved decisions"`, both
phase beads (`bob-cli-j6.1`, `bob-cli-j6.2`), and
`sase artifact read plan:202611/dot_semicolon_modifier_aliases.md "Need the original alias acceptance contract"`.
Open `bob-mac-capture` through `/sase_repo` and obey its `AGENTS.md`. Read the docs
reference memory before editing documentation. Keep every repository path relative to
its own checkout.

## Established evidence and constraints

- The original decisions are final: `leading_dot=after_text`; only
  `glossary:capture-alias` memory was authorized. The glossary already describes the new
  aliases and fast escapes correctly. This tale needs no memory edits.
- bob-cli `0f31c592` implements completion-gated `.` → `&` and `;` → `*`, with existing
  `^`, `@`, and `:` precedence, byte-range/Unicode/negative tests and CLI contract
  tests. `3c1b6451` updates the glossary escape sentence.
- Phase 1 note #1 correctly resolves the original plan's mistaken literal case: `  ;`
  continues to become `  :`. Preserve that tested precedence.
- Later bob-cli `4b59ae81` (bob-cli-j9.1) already stamps the two aliases as `dependency`
  and `blocked_task`; its reverse-proof tests cover both. Its additive `alias` JSON
  field is compatible with schema 1 and must survive this work.
- Mac `ef5dc4a` implements the reply-keyed arms, fast second keys, probes, own-picker
  routing and most tests. It has the concrete defects below. The Mac portable typing-tip
  model (`bd06c21`) already includes both alias IDs.
- The lander reviewed all original epic/child notes. The only child
  `PROPOSED FOLLOW-UP:` is bob-cli-j6.2 note #1: pre-existing mounted Mac docs budgets.
  It was corroborated on `bob-cli-h9` (+4), rather than duplicated.
- Current `just docs-check` on bob-cli `4b59ae81` + Mac `ef5dc4a` fails only five word
  budgets: json-outputs 6024; Mac capture-panel 6013, capture-previews 6424, keyboard
  6028, protocol 6048. The cap is 6000. Historical counts prove json-outputs grew 5999 →
  6024 in `84ff0bb8` (ir.11); Mac previews grew 6174 → 6314 → 6424 in
  `fe9ce15`/`a5152aa` (ir.11/ir.12), and protocol grew 5996 → 6062 in `fe9ce15`, then j6
  reduced it to 6048. Evidence was recorded on `bob-cli-ir` as well as h9. These
  unrelated overages are not alias work; add none.
- Exact Mac CI for `ef5dc4ab02138241f3f73b83e54df67d623fd00a`, run
  [42673185238](https://github.com/bobs-org/bob-mac-capture/actions/runs/42673185238),
  failed Build before tests: three AppKit/SwiftUI switches omit the `.addTo` case
  introduced by Record Notes commit `a5152aa`. The lander recorded this on owning active
  epic `bob-cli-ir`; no duplicate task was created.
- `sase bead epic-symbols bob-cli-j6` returned none during the audit. Neither checkout
  currently defines `just symvision`; recheck at closeout. The epic's `parent_id` is
  null and its ancestors array is empty.
- No `just check` was rerun in the landing audit; run it during this tale. Do not run
  `just check-full`. A check-full-only failure is test infrastructure, not remaining
  epic work.

## 1. Restore aliases before their picker appears

In Mac `Sources/BobMacCapture/CapturePanelModel.swift`, `relationEqualsOpenerRestorable`
ends its no-picker path with
`modifierAliasPickerSource(for: arm.sigil) == nil && !editorInputLocked`. This rejects
`&`, `*`, and even the existing `~` opener between Bob's rewrite reply and completion.
Before j6, that path returned `!editorInputLocked`.

After the existing arm/draft/sigil validation, allow an eligible opener to restore while
no picker exists and the editor is unlocked. Keep the stricter own-picker identity and
empty-filter checks when a picker exists. Keep submission, native caret, marked-text,
modifiers, repeat, and intervening-key protections; do not implement capture grammar in
Swift. Space restores the literal key followed by a space; Escape restores the literal
key; neither allows a stale completion reply to reopen the picker. Undo returns to the
sigil.

Add bounded hosted regression coverage in
`Tests/BobMacCaptureTests/CaptureModifierAliasTests.swift` for both new aliases and both
restoration keys in the post-rewrite/pre-picker interval. Cover `=` → `~` too so the
existing behavior cannot regress. Assert that late completion does not reopen a picker
or replace the restored draft, and preserve the existing tests for fast/slow second
keys, pre-reply probes, unchanged replies, non-empty filters, caret/navigation/modifier
disarming, and the older `.`/`;` meanings.

## 2. Make the race fixture real and integrate the sized-close fixture cleanup

In Mac `Tests/Fixtures/fake-bob`, the `@home &` and `Buy milk *` `capture-complete`
branches print fixtures and `exit 0` before the
`FAKE_BOB_MODIFIER_ALIAS_COMPLETE_DELAY_SECONDS` sleep. The lander set that variable to
0.8 seconds and measured 0.010/0.009 seconds for the two calls. Move the delay ahead of
those returns (or consolidate dispatch) so it actually controls both responses. Use a
bounded, observable synchronization point in the race tests; do not rely on merely
happening to observe a brief `picker == nil` state. The existing selection-analysis race
test must now exercise its intended delay.

Mac `90fa59d` deleted the uppercase `2Xx` rewrite fixture and its dispatcher branch
because uppercase X is prose and should not enter that lane. j6's `ef5dc4a` accidentally
restored the branch without the file. Calling fake-bob
`capture-rewrite --cursor 3 --format json -- '2Xx'` now exits 1 with missing
`capture-rewrite-sigil-alias-close-sized-prose-caps.json`. Remove that resurrected
branch, preserving `CaptureSeparatorToggleTests` and the structural sized-close fixture
edits. Do not recreate the intentionally deleted fixture.

## 3. Correct the decision-dependent documentation and compile integration

In Mac `docs/capture-panel.md`, the paragraph currently says both that leading dots stay
literal under `after_text` and that item-start `.bashrc`/`.5` needs `..`. Remove the
rejected-branch claim and the internal decision label: item-start `.bashrc` and `.5` are
literal, while the earlier sealed Move aliases retain their precedence. Keep the page's
net word count non-increasing. Preserve the accurate keyboard, protocol, and
relationship-picker descriptions.

Review drift since bob-cli `4b59ae81` and Mac `ef5dc4a` before editing shared files.
Preserve any later alias-tip host work. Incorporate an existing Record Notes fix for the
`.addTo` compile failure if present. If it remains, apply the minimal integration
adaptation and record its completion on `bob-cli-ir`:

- `CapturePanelModel.completeSimilarTaskIDAssignment`'s replacement switch must
  explicitly reject `.addTo` along with non-task-ID actions; record updates use Bob's
  explicit record splice and must not synthesize task-ID replacements.
- The `.similarTask` action switches in `CapturePanelView`'s string hint and
  `similarTaskInsertsText` need the equivalent no-task-ID hint case.
- Keep normal record Add to it splices in `SimilarTasksBandEdits.editedDraft` and
  ordinary task Depends On/Blocks/Relate/Add as note behavior intact. Use existing
  record/task tests and a focused regression only if needed to pin that boundary.

Do not broaden this tale into Record Notes feature work or the docs-budget cleanup. The
original capture rewrite rules, CLI contract, release version, and changelog need no
changes.

## 4. Verify before final closeout

Run bob-cli `just check`. Run both repos' `just docs-check`, mounting the exact linked
Mac checkout, and classify any remaining failures against the five measured budget
baselines and owning notes above; no new alias-related documentation failure is
acceptable. Confirm the two contract rewrites (`@home .`, `Buy milk ;`), preserved
precedence (`Buy milk .`, `@errands.`, `;`, `  ;`), and literal item-start dots. Retain
additive alias metadata from the later typing-tip contract.

For Mac, run `just format-lint` and `just portable-check`. Compile a disposable scratch
copy on the Mac using `Scripts/bundle.sh` and `just typecheck-tests`, obeying its AGENTS
instructions and tailnet memory; never use Bryan's checkout or install the scratch copy.
If the Mac is offline, report that explicitly and use the macOS CI gate for the
platform's required validation.

Run `Scripts/macos-ci-check` against the final working tree through `/sase_monitor` and
inspect Build, Test, Bundle and install results. Its temporary snapshot lets
verification happen before this tale's finalizer commit. The monitor continuation must
finish this tale, including step 5; it is not a new land agent. Resolve any epic-caused
failures. Inspect all alias, separator, and relation regression suites, not just the new
file. Do not accept the historical scratch compile as a substitute for testing the final
integrated tree.

Do not make step 5 wait for this tale's own commit SHA, push, or the CI result from that
push. Close the epic in the same turn as the final code once working-tree verification
succeeds; finalization commits afterward. Observe any post-push CI through the required
monitor workflow without making it a predecessor of closeout.

## 5. Finish bob-cli-j6's landing

Re-read the epic and every child and note to ensure no new blocker appeared. Recheck the
linked original plan and any new commits since this audit. Record the resolved escape
race, actual fixture delay, restored sized-close cleanup, after_text docs, typing-tip
compatibility, and precise verification results in the close note.

Include every follow-up outcome: bob-cli-j6.2 note #1 was corroborated on `bob-cli-h9`
(+4), not duplicated; later Record Notes docs growth and `.addTo` build failures were
recorded on `bob-cli-ir`, with the compile fix's final disposition. The four
alias/fixture/docs defects remain epic work and must all be fixed, not filed away. No
original proposal was silently declined.

Run `sase bead epic-symbols bob-cli-j6`. Resolve every entry (wire it, privatize it, add
an appropriate non-test pragma, or delete it under Symvision policy), or re-key only an
exemption genuinely needed by a still-open later bead. Then run
`sase bead close bob-cli-j6 --note "<concrete verification and follow-up outcomes>"`. If
the close rejects stale symbols, fix them and retry. Do not force a successful landing
or bypass unfinished descendants; finish/reopen them as appropriate.

After closing, run `just symvision` if available and record its result; if the recipe is
still absent, explicitly record that rather than claiming a pass. Open the plans repo
through `/sase_repo` before modifying it, and set `status: done` in
`202611/dot_semicolon_modifier_aliases.md` (canonical ref
`plan:202611/dot_semicolon_modifier_aliases.md`). Preserve its other frontmatter.

Run `sase bead read bob-cli-j6 -r "Need the parent link"` after closing. The audited
parent is absent, so normal completion is expected. If the relationship changed, follow
the user-authorized ancestor rules: verify and normally close only a directly parented
phase, leaving its epic to its land agent; for a parent plan, recheck its previous
landing note, all descendants/notes, linked plan, and post-child drift, retire its
symbols, close normally, run available symvision, mark its plan done, and repeat through
complete directly parented plan ancestors. Stop and note the first incomplete or
ambiguous parent; never force to advance a nested landing.

Finish via `/sase_final`, declaring every changed repository, including the linked Mac
checkout and plans repo. Include the epic ID in the appropriate commit message.
