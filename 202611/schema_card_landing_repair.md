---
tier: tale
title: Repair Schema Card geometry and finish bob-cli-mu
goal: Field keycaps and labels share the approved axis, refusal motion is correct,
  and bob-cli-mu is verified and closed.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-mu.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-mu
status: done
---

- **PARENT:**
  [202611/schema_card_field_jumps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/schema_card_field_jumps.md)
- **BEAD:**
  [bob-cli-mu](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mu/README.md)

# Repair Schema Card geometry and finish bob-cli-mu

## Outcome and scope

Finish the remaining work of epic **bob-cli-mu**, then close that epic in the same
coding turn. This is a single-agent repair, not another epic. Its three original phases
are already closed. Do not wait for another land agent or for this repair's commit SHA,
push, or CI before performing closeout.

Read `sase bead read bob-cli-mu -r "Need the landing audit and follow-up dispositions"`
and its latest landing notes. Read the approved original plan through
`sase artifact read plan:202611/schema_card_field_jumps.md "Need the original acceptance requirements"`.
Its final decision is `field_number_record = no`: **edit no memory files**.

The reproducible browser/runtime probes and the complete audit evidence are in
`file:explicit:8c900ff7e5039639f9276cc6`; read it through `sase artifact read`.

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. All implementation paths
below are relative to that checkout; `F/` means `plugins/bob-ledger-tools/src/`. Edit
source fragments and regenerate `main.js`; never hand-edit the generated bundle. Keep
each fragment within 1000 lines.

## Verified starting point

- Epic commits are `137c77f` (field jumps), `e32deb6` (field keycaps/visuals), and
  `f80d0dd` (bank group jumps). The landing audit read every child note and the current
  implementations, including shared order, selection, motion, accessibility, hotkeys,
  hint gates, and generated registration.
- The only non-epic plugin commit since the first epic commit is `f49e175` (Record Card
  update session/writer). Its direct-apply mode is opt-in; existing schema navigation
  still makes no writer calls. Focused album, arrangement, bank, record-sheet,
  record-update, and e2e-wrapper suites passed together. `npm run build:check` and
  `npm run docs:inventory:check` passed.
- The bob-cli base is `6c409ab2`. Concurrent records CLI contracts and scan work do not
  require changes for field jumps. Both checked origins matched their local heads during
  the audit. Recheck drift before editing and before closing.
- `bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools` now succeeds: 0 copied,
  0 skipped, 3 unchanged. The original phase deployment refusal is resolved without
  `-F`. Recheck and sync the repaired bundle normally.
- No `--epic-symbol` entries were found for bob-cli-mu. Neither checkout currently has a
  symvision recipe. The epic currently has **no parent_bead**.

## 1. Repair the measured geometry defects

The lander serialized the real card painter's Zettel fixture into Chromium, with the
committed stylesheet and a 16px row font. This is a browser CSS reproduction, not a
real-Obsidian acceptance pass. It measured:

| Measurement                         | Observed                     | Required                         |
| ----------------------------------- | ---------------------------- | -------------------------------- |
| Keycap width                        | 18.09375px                   | about 23.2px (1.45 row em)       |
| Status / Claim / Section label left | 518.40625 / 488 / 582.8125px | one common axis                  |
| First choice and + field left       | 60.796875px                  | the field-label axis, within 1px |

Two concrete causes in `plugins/bob-ledger-tools/styles.css`:

1. `.bob-album-schema-dialog .bob-album-schema-field` still sets
   `justify-content: space-between`, distributing free space between the grip, number,
   label, and filled-count cells. The new fixed gutter does not account for that
   distributed space.
2. `--bob-schema-keycap: 1.45em` is resolved on the keycap after its
   `font-size: 0.78em`, shrinking the box while the cloud gutter uses the parent font.
   The original plan explicitly warned against this.

Use a fixed start-aligned grip/keycap/label arrangement; keep the filled count at the
trailing edge independently (for example, with an auto margin). Make keycap box
dimensions use the row's font basis while keeping the digit at 0.78em. Account for the
actual border, padding, and gap widths in the shared gutter. Preserve the flat body DOM,
invisible field-10+ slot, dialog-only styling, and all ghost/gap/ drag behavior. Ensure
optional grips during stages do not shift labels unexpectedly.

Measure every field label, not only field 1. Check chip-cloud and + field alignment,
ghost text, a field with no choices, and trailing filled-count text at desktop and 375px
widths. Preserve transparent rest rows, filled chips, visible selection rings, 0.8em
group breaks, and the existing reduced-motion treatment. Do not weaken the geometry
tolerances to accept the defect.

## 2. Correct the empty-schema refusal direction

`F/241yf-plugin-album-schema-field-jump.js::schemaFieldStep` calls
`schemaFieldJumpMiss(1, ...)` when there are no fields, including Shift+K. The lander's
runtime probe captured `[1]` for `schemaFieldStep(-1, {})`, where the approved feedback
vocabulary requires `[-1]` (upward). Use the requested direction for this branch; digit
misses still bump downward. Add a focused regression for empty-schema J/K, repeat
suppression, reduced motion, and no writes in
`scripts/test-ledger-tools-album-schema-field-jump.cjs`.

## 3. Complete visual and integration acceptance

Extend the real geometry probe in `scripts/obsidian-album-ui-e2e.cjs` to assert the
keycap's row-relative box size and all field-label axes. Existing tests use FakeNode and
cannot validate CSS layout; retain those behavioral tests and add a real layout check
rather than a test that merely matches stylesheet text.

Run the existing album and bank Setup real-key probes against the repaired build in an
isolated Obsidian scratch vault. On Linux use `xvfb-run -a`. Apollo lacked both Obsidian
and xvfb-run during the land audit; use a supported Obsidian host and the exact patched
source tree if local prerequisites are absent. Follow `/sase_repo` for repository access
and read the tailnet reference memory before remote access. A self-test or a
generic-browser layout pass does not substitute for Obsidian's theme CSS. Preserve the
scripts' reports and screenshot artifacts.

Open and judge the light, dark, and 375px field screenshots. Confirm crisp keycaps, one
label axis, separated Claim/Section groups, unclipped selection rings, and visible
close/more hints. Confirm Status -> Shift+J -> Claim, 3 -> Section, parent-first Shift+K
from a chip, field 9 scrolling into view, projected-order j/k and digit navigation, and
bank group jumps with lowercase k and digit hotkeys preserved. Keep navigation
writer-silent.

Inspect commits that landed after `f80d0dd`, particularly bob-cli-ms changes to record
entrypoints, shared schema classes, and update sessions. Preserve the inline record
sheet's unnumbered presentation. If the new Record Card now owns `==` on record notes,
verify the remaining explicit album Schema entrypoint and do not restore an obsolete
`==` binding. Integrate only actual conflicts with this epic; unfinished Record Card
feature work stays on bob-cli-ms.

Run `npm run build`, the affected suites, `npm test`, and `npm run docs:check` in
bob-plugins. Set `BOB_CLI_DIR` to the current authorized bob-cli checkout for the docs
check so it does not read a separate cache clone. Run `just check` for file-change
verification in bob-cli; **never run `just check-full`**. Use `/sase_monitor` for long
commands. Reproduce unrelated failures on an unchanged base and report them accurately;
do not call known infrastructure flakes remaining feature work or blindly raise their
timing budgets.

Sync through `bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools` and follow
the repo's deployment/post-sync smoke protocol. Do not force an overwrite; follow the
landed-build recovery procedure if protection reappears. Verification and deployment can
use the working tree: closeout must not depend on this turn's future commit or CI
result.

## 4. Finish this epic's closeout yourself

The lander completed follow-up triage before proposing this tale. Carry these outcomes
into the final close note; do not create duplicates:

- mu.1 #2 and mu.2 #2: JSON-reference word budget routed as corroborating
  `DISCOVERED ISSUE` evidence to causal active epic **bob-cli-ms**. Independent
  `bob_docs.markdown.count_words` counts: 5999 before ms.1, 6058 at `99005975`, 6089 at
  `48a3116d` and current HEAD. A budget bump was not adopted.
- mu.1 #3, mu.2 #1, mu.3 #1: timing reports corroborate existing tasks **bob-cli-km**
  (event-link-times), **bob-cli-io** (frees-marks), **bob-cli-ki**
  (keep-moving-marks-surface), **bob-cli-ju** (off-clock-link-chips).
- mu.1 #3 also corroborates **bob-cli-3w** (dependencies-stage-entry-and-view).
  **bob-cli-m1** is a different growth-ratio node and was deliberately not corroborated
  for this proposal. Blanket budget increases were not adopted.
- mu.1 #1 was an accidental note containing read-command text, not unfinished work.
  Deployment-blocked notes on all phases are superseded by the successful clean sync.
  Missing real visual acceptance is retained in this tale.

Re-read `sase bead read bob-cli-mu -r "Need final readiness and the parent link"`, all
descendants and their notes, and the original linked plan. Check for new descendants,
new notes, and relevant post-audit commits. Finish any epic-caused issues before
closing. If this tale is represented by a child plan bead, close that completed child
normally first so descendant readiness permits epic close; do not force a successful
nested landing.

Run `sase bead epic-symbols bob-cli-mu`. Resolve every listed symbol by wiring,
privatizing, a justified non-test pragma, or deletion; re-key only when a named
still-open later bead genuinely needs the exemption. Then execute
`sase bead close bob-cli-mu --note "<concrete original-scope and integration verification, repaired geometry and empty-schema feedback, test/deploy evidence, and every follow-up disposition above>"`.
If the close is refused, resolve the named blockers and retry normally. Never use
`--force` merely to get a successful close. Cancellation/supersession is a different
outcome and requires an explicit justified resolution.

After close, run `just symvision` if available; explicitly record absence if it is still
unavailable. Open the plans repository via `/sase_repo` and set `status: done` in the
frontmatter of **plan:202611/schema_card_field_jumps.md**, at the PLAN path returned by
the audited bead read.

There is currently no parent of bob-cli-mu. Recheck after closing; if unchanged, finish
normally. If a parent has since been attached, close only a fully verified parent phase
normally, leaving its containing epic to its lander; for a directly parented plan,
re-audit its previous landing note, all descendants/notes, linked plan, drift,
readiness, and symbols before normal close, symvision, and plan-done update. Stop at an
incomplete or ambiguous ancestor and note its blocker there.

Finish through `/sase_final`, committing every edited repository. The repaired code,
closed epic, and done plan belong to the same finalizer turn.
