---
tier: tale
title: Finish collection title ranges and land the collection-card epic
goal:
  Hide trailing title-block blanks, integrate the shared editor badge, verify the
  plugin, and close bob-cli-o6.4.1 and parent phase bob-cli-o6.4.
size: medium
proposed_by: bbugyi200.athena.bob-cli-o6.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-o6.4.1
create_time: 2026-11-20 21:22:31
status: wip
---

- **PARENT:**
  [202611/collection_cards_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/collection_cards_title_block.md)
- **BEAD:**
  [bob-cli-o6.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.4.1.md)

# Finish collection title ranges and land the collection-card epic

## Outcome and boundaries

Finish the remaining title-range contract gap in **bob-cli-o6.4.1**, then close that
epic and its parent **phase bob-cli-o6.4** in this tale. One coding agent can complete
this bounded shared-parser change, editor-badge integration, and the landing checks.
There is no separate land agent after a tale.

Read through the audited interfaces:

- `sase bead read bob-cli-o6.4.1 -r "Need the landing audit and follow-up dispositions"`,
  and its three closed children `.1`, `.2`, and `.3` with their notes.
- `sase artifact read plan:202611/collection_cards_title_block.md "Need collection-card acceptance and closeout"`.
- `sase artifact read plan:202611/property_card_title_block.md "Need the binding title-range contract"`.
- Open `bob-plugins` with `/sase_repo`; read its `AGENTS.md` and
  `docs/develop/deployment.md`. Open `plans` through `/sase_repo` before changing the
  archived plan's frontmatter; use the returned checkout, never another workspace.

The outer DECISIONS remain final: this tale edits no memory or decision records. The
original glossary work belongs to `bob-cli-o6.7`. Do not close `bob-cli-o6`,
`bob-cli-o6.6`, another sibling phase, or either Record Card epic (`bob-cli-ms`,
`bob-cli-ms.8`). The outer proof phase owns the full cross-species walkthrough,
grayscale/forced-colors matrix, live album tidy, and wider host rollout. Never modify
`scripts/obsidian-record-writer-e2e.cjs` or its unit wrapper while those Record Card
epics remain open.

## What the landing audit established

The collection changes are on bob-plugins master:

- `f16ddea`: bank/editor identity badge, preserving the Bank setup label and keys.
- `a68e29b`: album local title, title-slot adoption, report-body join, schema fold.
- `b8b7baf`: record local title and slot, shared ownership classes, docked walk rail.

Current plugin version at audit was `1.148.0`; bob-cli was `8f703ccb`. All three phase
notes and commits were reviewed, including the stable selector consumers, local title
precedence, row/title ownership separation, failure cleanup, identity headers, docs, and
protected writer files. Fresh `build:check` passed, and seven focused suites passed
213/213 (title range/ownership, adopter, slot, album, record, bank Setup, and day/period
title-block tests). That green result does not cover the contract gap: existing tests
explicitly assert the wrong trailing-blank behavior. The fresh audit log is
`file:explicit:853697ae77344b5b2c077384`.

Audited phase proof includes album report `file:explicit:5911d19e7c9aae058cbad08d`
(129/130, only known Ctrl+J failure), record full report
`file:explicit:f0c604b18ca234048e388234` (180/180 including editing workflows), record
walk `file:explicit:8a936446acea27130b062c71` (16/16), preview
`file:explicit:889bc05d45700ad156272d37` (28/28), and bank/base comparison summary
`file:explicit:c18918352df0c879b13cfa09`. These are previous-phase evidence, not a claim
that the repaired tree has been tested or deployed.

Post-start drift was reviewed through current origin/master in both repositories. Plugin
`eed318e` added day/period adapters using the same title-range kit; their controller
additionally manages temporal models, render components, and content-gated keys, so this
tale need not replace that controller. Later command-chip and successor-picker commits
(`c7a9221`, `63cd4ee`, `1a6ff79`, `6e0d764`, `570c3a1`) do not change collection title
inputs or query body-only collection selectors. CLI successor-jump and chip-glossary
changes likewise need no collection adapter. The required integration is to repair the
shared range for **all** its adopters, including the day/period work that landed in
parallel, and replace that parallel work's duplicate editor identity builder with this
epic's reusable badge. Recheck subsequent drift.

## 1. Repair the shared range, preserving editing boundaries

In bob-plugins, `plugins/bob-ledger-tools/src/240h-plugin-property-card-title-range.js`,
`propertyCardTitleSourceRangeFromLines` skips empty follower lines without advancing
`toLine`. Consequently, for `# Restaurants\n\n\nBody`, the hidden range is only
`# Restaurants`. Day and period ranges similarly stop at their last recognized nav or
child line. The contract explicitly includes the blank lines following the recognized
title block. Phase `bob-cli-o6.4.1.1` note #4 reported the visible gap.

Extend the recognized range through its trailing blank lines, stopping before the first
unrelated nonblank line. Keep intervening blanks and recognized followers in the same
range. Preserve exact offsets for LF, CRLF, CR, BOM, EOF, and an absent final newline. A
title/walk-only note may consume trailing blanks to EOF. Keep the first unrelated body
line entirely outside the range, including its start offset, so a cursor on that line
does not reveal the hidden title. Keep the inclusive selection touch rule: touching a
consumed blank reveals the source; selections in frontmatter or unrelated body content
do not. Do not absorb frontmatter, arbitrary leading spacing, mismatched or late H1s,
fences, prose, or legacy day lines. H1-less album notes continue to hide nothing, and
records may begin with a classified walk line.

Use the shared kit, not album-only CSS or a duplicate recognizer. Keep native Source,
Reading, embeds, fallback behavior, ownership generation, and note bytes unchanged.
Update the existing range tests that explicitly exclude trailing blanks; add contract
cases for album, record walk (alone and after an H1), day navigation, and period child
lines, with reveal on a consumed blank and no reveal at the next content line. Verify
slot cards and rails remain mounted while the raw range is revealed. Do not merely
change snapshots to bless a gap. Keep fragments under 1000 lines and regenerate
`main.js` rather than editing it.

If touching architecture prose, read `docs.md` through `/sase_memory_read` first.
Clarify the range boundary concisely. Also scope its existing "Adapters do not call the
slot and ownership primitives directly" sentence to collection adapters: the day/period
controller has its own render-component lifecycle. Do not misdescribe it as already
using `propertyCardAdoptTitle` or broaden this into a controller rewrite.

## 2. Integrate the shared identity badge into the parallel editor work

`eed318e` added `intentionCardEditorIdentityRow` in
`plugins/bob-ledger-tools/src/297cc-plugin-intention-card-editor-paint.js`. It manually
builds a glyph tile plus species word, duplicating the additive
`propertyCardBuildIdentityBadge` in `240j-plugin-property-card-identity-badge.js` that
landed during this epic. The shared helper already supports leaf `band` and `value`, so
use it for day, week, quarter, and year editor headers while preserving the
`.bob-intention-editor-identity` hook, path-derived identity, temporal accent, and
heading-free semantics. Preserve the surrounding editor's title, focus, fields, and
keys. Remove only the redundant construction and now-unused CSS for the old species-word
selector; keep any temporal overrides the shared badge needs.

Update the existing title-block suite's editor assertions (around line 883) and the real
intention-card editor identity probe (around line 866) to verify a single shared badge,
correct leaf band/value and species word for all four kinds, no extra level-one heading,
and preserved temporal styling. Include editor captures in both themes and at narrow
width. This is helper adoption, not a rewrite of the separate day/period slot controller
or any change to its content/ghost rules.

## 3. Verify and deploy the integrated tree

Fetch and integrate current origin/master before final checks. Preserve both sides'
fragment and test registrations, rebuild generated files, choose the next unused
bob-ledger-tools minor version, and regenerate the docs inventory. Run build,
build:check, npm test, and `BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check`.
Run `just check` in bob-cli for file-change verification; use `sase tool run check`
where available. **Do not run `just check-full`.** A full-check-only failure is test
infrastructure, not remaining feature work. Use `/sase_monitor` for long commands;
continue this same tale's closeout after results arrive, never leave it waiting for
another land agent.

Re-run the existing album UI, record-card full-scope (includes workflow), record-walk,
and day/period intention-card CDP acceptance on athena because the shared range changed.
Use a current explicit BOB_BIN, BOB_CLI_DIR, `TMPDIR=/tmp`, isolated scratch
vaults/profiles, and `xvfb-run -a`. Add real-editor evidence that consumed blank lines
collapse and reveal, the next body line stays outside the hidden block, both title slots
retain their card/rail, mode switches restore native content, and Markdown bytes do not
change. Keep every existing assertion meaningful. If a command fails, diagnose it and
distinguish a real regression from the attributed baseline failures below; rerun
load-sensitive tests in isolation. Never convert a skipped proof to a pass. Rerun bank
acceptance only if its code or a shared dependency affecting it changes; its
range-independent badge is already proved by the phase evidence.

Render `scripts/preview-property-cards.cjs` under Xvfb, inspect every current album,
record, and bank capture (both themes and widths), and preserve the structural gates.
Register the new reports and key screenshots with
`sase artifact create --bead bob-cli-o6.4.1` and cite them in the close note.

Deploy through the normal guarded
`bob plugins sync --repo <opened plugin checkout> --no-pull -p bob-ledger-tools`,
dry-run first, then real sync; verify managed-file hashes and run the documented
deployed-vault smoke gate. A clean baseline sync can establish the same-checkout stamp
before dirty iteration; follow the documented refusal procedure and do not discard
another agent's deployment. The verified local build can deploy before this turn's final
commit. Do not make closeout depend on this tale's own future commit SHA, push, or CI
result. `/sase_final` commits the code after the turn, as normal.

## Follow-up dispositions already completed by the lander

All 13 phase proposals were triaged, with outcomes recorded on bob-cli-o6.4.1 before
this proposal. Include these dispositions in the eventual close note:

| Proposing phase notes                 | Outcome                                                                                                                                                                                                            |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `.1 #1`, `.3 #2` album Ctrl+J         | Existing `bob-cli-nq`, corroborated.                                                                                                                                                                               |
| `.1 #2` held-lock writer              | Existing `bob-cli-n0`, corroborated; do not edit the protected record-writer harness.                                                                                                                              |
| `.1 #3`, `.2 #2`, `.3 #1` docs budget | Existing `bob-cli-oc`, corroborated; causal active outer epic `bob-cli-o6` also received evidence (its `.3` commit crossed the cap).                                                                               |
| `.1 #4` trailing blanks               | This tale's remaining epic work; no task.                                                                                                                                                                          |
| `.1 #5` long TMPDIR                   | Existing `bob-cli-np`, corroborated.                                                                                                                                                                               |
| `.2 #1` event-link timing             | Existing `bob-cli-km`, corroborated.                                                                                                                                                                               |
| `.3 #2` bank Ctrl+K                   | Existing `bob-cli-nr`, corroborated.                                                                                                                                                                               |
| `.3 #3` arrangement                   | Existing `bob-cli-ns`, corroborated, with the intermittent preview-2 timeout retained as secondary diagnostic evidence; no proven distinct root cause to split yet.                                                |
| `.3 #4` bank UI failures              | Invalid fence -> `bob-cli-m6`; walk-learn -> `bob-cli-m7`, both corroborated. Obsolete queue import assertion -> active `bob-cli-no`, whose `c3ca244` changed the footer; evidence attached there, no task.        |
| `.3 #5` inherited bank H1 typography  | New small bug `bob-cli-og`, ready; diagnosed pre-existing em/weight inheritance, audited screenshot `file:explicit:06ac5bda11561f50cbb70939`. Outside this tale.                                                   |
| `.3 #6` executable default            | Declined as a distinct fix: later album/record reports successfully use the raw executable. The launch symptom is recorded on `bob-cli-np` for environment diagnosis, without claiming a proven shared root cause. |

No fresh full-suite/CDP failure was claimed when attributing earlier phase reports.
Known baseline failure does not authorize weakening a check. Record its actual current
result and owner; finish every regression caused by this tale before closing.

## 4. Finish this epic and its parent phase in this tale

This is the final step, not a handoff to another lander. Re-read bob-cli-o6.4.1, its
descendants and notes, its linked plan, and any commits since this audit. Confirm the
original album/record/bank scope plus this repair is complete and retain the follow-up
dispositions above in the verification note.

1. Run `sase bead epic-symbols bob-cli-o6.4.1`. Resolve every entry keyed to this epic
   or its phases (wire up, privatize, non-test pragma, or delete per policy), or re-key
   only to a still-open later bead that actually needs it. The audit found no entries,
   but recheck rather than assuming that remains true.
2. Close normally with
   `sase bead close bob-cli-o6.4.1 --note "<original-scope audit, integration/range repair, verification and deployment evidence, follow-up outcomes>"`.
   If rejected, resolve the actual symbols or unfinished descendants. Never use
   `--force` to make a successful nested landing advance.
3. Run `just symvision` if the repository exposes it; otherwise record its absence. Set
   `status: done` in the frontmatter of the epic's linked plan
   `plan:202611/collection_cards_title_block.md` (the opened plans checkout's
   `202611/collection_cards_title_block.md`). Preserve all other frontmatter.
4. Re-read `sase bead read bob-cli-o6.4.1 -r "Need the parent link"`. Its parent is
   **phase bob-cli-o6.4**. Read that phase, verify this child completed its full
   collection-cards scope, run `sase bead epic-symbols bob-cli-o6.4`, and resolve or
   legitimately re-key every exemption. Then close only that parent phase normally:
   `sase bead close bob-cli-o6.4 --note "<collection scope, commits/build version, artifact refs, gates and deployment rechecked>"`.
   Recheck symvision if available. Leave containing epic bob-cli-o6 and its plan to its
   already-waiting land agent. If this parent is incomplete or ambiguous, note the
   concrete blocker on it and report it rather than forcing closure.
5. Mark this tale done as its workflow requires and finish through `/sase_final`,
   declaring every modified repository for commit. Closing the epic in the same turn as
   the final code is required and does not await that commit's SHA.
