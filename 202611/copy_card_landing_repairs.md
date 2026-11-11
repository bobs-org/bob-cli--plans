---
tier: tale
title: Finish Copy card correctness and land bob-cli-kc
goal:
  Close the verified Copy card epic after fixing fenced-task resolution, card feedback,
  contract edge cases, and missing regression proof.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kc.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kc
create_time: 2026-11-11 17:21:42
status: wip
---

- **PARENT:**
  [202611/cmd_y_copy_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/cmd_y_copy_card.md)
- **BEAD:**
  [bob-cli-kc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kc/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-kc.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-kc.land.md)
- **COMMITS:**
  - [d532aad](https://github.com/bobs-org/bob-plugins/commit/d532aad74573f4a8e66a0fb8fb0a4a33c7415b9d)
    — fix(copy-card): reject fenced tasks and honor copy config

# Finish the Cmd+Y Copy card and land bob-cli-kc

Complete only the gaps found while landing epic `bob-cli-kc`, then close that epic
yourself. This is one bounded implementation task, with no later land agent. Do not wait
for this tale's own commit, SHA, push, or CI result before the final closeout.

Read `bob-cli-kc` and its landing notes, its four closed phases (`bob-cli-kc.1` through
`.4`), and `plan:202611/cmd_y_copy_card.md` through the audited SASE commands. The
original plan's PLAN path is `sase/repos/plans/202611/cmd_y_copy_card.md` relative to
the host workspace. Re-resolve it from the bead before editing. The original epic has no
`parent_bead` as of the landing audit.

The epic decisions are final: `cmd_y_twice = yes`; `copy_card_record = no`. Preserve the
fresh second Mod+Y copy gesture. Do not edit any SASE memory or decision record. Copying
remains read-only: no vault writes, generated IDs, stamps, or undo entries.

## Existing work and evidence

Open the linked `bob-plugins` repo with `/sase_repo` and read its `AGENTS.md`. The
implementation is in `plugins/bob-navigation-hotkeys/src/101-copy-card-model.js`,
`101a-copy-card-keys.js`, `102-copy-card-config.js`, `103-copy-card-modal.js`, and
`641-plugin-copy-card.js`. Fragment registration, mixin installation, palette commands,
and exports already exist. Edit source fragments, regenerate `main.js`, and keep each
hand-edited fragment at or below 1000 lines (101 is already 893).

Audited epic commits: bob-plugins `353eedb` (model), `d1d294b` (card), `934bbbb` (real
Obsidian e2e/polish); bob-cli `0b4483be` (configuration reference); chezmoi `68f2fbba`
(Task ref template). No epic commit edits memory. The lander fetched and reviewed master
through bob-cli `ee2d2cbf`, bob-plugins `934bbbb`, and chezmoi `09725bd3`. All were
current with origin at that point.

Fresh landing checks: 55/55 Copy card model/view/offline-e2e tests pass;
`npm run build:check` and `npm run docs:inventory:check` pass; Reading Board/Stats
stylesheet controls pass 46/46. These passing checks miss the cases below. The lander
did not rerun the real Obsidian walkthrough or the full suites and did not run
`just check-full`.

## 1. Keep fenced examples out of task resolution

`resolveCopyCardTask` calls `findOwningTaskLine` without rejecting fenced cursor
positions. Reproduction using the built helpers from
`scripts/navigation-hotkeys-harness.cjs`, with a collapsed selection on line 2:

````js
const lines = [
  "- [ ] #task Parent ^parent",
  "  ```",
  "  - [ ] #task Example ^example",
  "  ```",
];
// Current result: ready, blockId "parent". Required: none.
````

With four spaces before the fence and example, the current result instead returns the
literal example's `example` ID. Also cover that four-space fence nested under an
intervening child bullet whose source text is `  - code:`.

Reject task resolution when either endpoint relevant to the selection is inside a fenced
code block, including list-nested backtick and tilde fences. Do not fall back to the
surrounding task from inside its fenced example. Preserve normal task lines, child
notes, managed relation/dependency lines, completed tasks, and equal-owner selections.
Start with a Copy card guard; reuse existing Markdown context machinery where it is
correct. If shared parsing must change, add regressions for its existing consumers
rather than globally reinterpreting all indented text as fences. Test both the card and
`yank-block-id` through the actual plugin entry points.

## 2. Honor the configuration and modifier contracts

Close these small confirmed gaps, keeping built-in rows usable on configuration errors:

- Explicit `key: ""` and `key: "  "` currently silently become the positional key;
  `key: " t "` silently becomes `t`. Only an omitted optional key gets the positional
  default. Validate supplied strings against the single-letter/digit rule after
  lowercasing, without trimming invalid input into a valid key. Preserve YAML numeric
  digit support and collision behavior. Test explicit null according to the same
  supplied-versus-omitted rule.
- Parsed `{copy: null}` currently returns empty templates with no diagnostic, although
  an explicitly present non-mapping `copy` must diagnose `copy must be a mapping`.
  Missing `copy` and an empty templates list remain quiet. Cover real YAML null/blank
  forms and cache reload; do not replace these with a parser stub that hides shape
  errors.
- `resolveCopyCardKey(model, {key: 'y', metaKey: true, ctrlKey: true}, ui)` currently
  copies; Alt+Escape currently closes. Ignore modified chords except the platform's
  actual Mod+Y and exact Ctrl+N/P gestures. Preserve bare Escape, Caps Lock letters,
  Shift-letter suppression, busy suppression, and repeat/IME suppression. Make the
  platform choice testable without introducing a user setting.

## 3. Make long cards explain themselves and keep the highlight visible

The tenth template without `key` has model reason `add key: to bind`, but the `ready`
rendering branch drops it. A live `CopyCardModal` with ten templates renders the tenth
label/value and no reason. Render this binding hint without disabling the row or
including the hint in the clipboard payload. Preserve collision warnings.

`setHighlight` currently rebuilds the rows and footer and never scrolls or focuses the
selected row. In the bounded-height card, keyboard selection and refused keys must
reveal their row. After render, scroll the selected/refused row into the card's visible
region using nearest alignment; preserve the scroll position when no movement needs it.
Do not move the keyboard highlight on hover. Make a failed clipboard write's status
visible even when the list overflows. Keep changes scoped to `.bob-copy-card*` and the
Copy card modal.

Add behavioral tests for more than nine templates, keyless/collision selection,
wraparound, refused `I`/`T`, and visible status. Verify actual element geometry in the
real Obsidian fixture; a FakeNode assertion alone cannot prove scrolling.

## 4. Complete the promised snapshot proof and real-app regressions

The test named
`snapshot: the card copies the frozen file after the active file switches` in
`scripts/test-navigation-copy-card-view.cjs` currently constructs model A and presses P;
it never switches files. Replace it with a test that opens through
`plugin.openCopyCard()` on A, changes the workspace file/view and editor buffer to B,
then proves both the displayed snapshot and the clipboard still use A. Cover the path
and task/template payloads, with the real clipboard callback stubbed only at the
external write boundary.

Extend `scripts/obsidian-copy-card-e2e.cjs` and its fixture/offline tests for the fence
and overflowing-card regressions. Run the walkthrough in its isolated scratch
vault/profile, never the live vault. Cover source and Live Preview where task resolution
is involved. Seed a unique clipboard sentinel before each copy case, including Enter,
Ctrl+N/Enter, and Mod+Y twice, so an old equal value cannot make a no-op write pass.
Keep the existing all-keys, refusal, no-write, closed-task, diagnostic, and light/dark
coverage. Include a real file switch while the card is open if the app allows it; the
plugin integration test above remains mandatory.

Regenerate all ten existing light/dark screenshots, add long-list/visible-refusal
evidence, inspect every image, and fix clipping or unreadable states within this
feature. Publish the report/screenshots as audited artifacts. Report Linux Ctrl+Y
evidence honestly; do not claim a Mac UI run unless one actually happens.

## 5. Preserve later work and repair the touched documentation

Later plugin changes add Vim yank flash (`7197394`, `99ccca1`), Keep-moving CSS cleanup
(`4e1f377`), bank data/actions (`8a1daca`, `c35662b`), and prj/hide marks (`3423ba6`).
They do not replace the Copy card. Keep all test registrations; the new bank transaction
checkbox rows without `#task` must remain file-only Copy card contexts. Keep copying
outside the Vim register/yank-flash hook. Do not implement the deferred prj/hide
picker-title work from `bob-cli-kg` as part of this tale.

Bryan's later chezmoi commit `09725bd3` intentionally changed the configured template to
`the ^{block_id} task in the {path} file`. Preserve that suffix. Add a model regression
for that configured string; do not replace his config with the older example. Open
chezmoi with `/sase_repo` only if rechecking it. No chezmoi edit or post-commit apply
dependency is required for this repair.

Read `docs.md` with `/sase_memory_read` before documentation edits. The Copy card commit
grew `docs/bob-navigation-hotkeys.md` from 6186 to 6200 words, measured with
`bob_docs.markdown.count_words` in bob-cli. Trim duplicated prose in that source page to
at most 6000 words while preserving command facts, published anchors, and the link to
`docs/copy-card.md`. Keep the detailed Copy card contract on its own page; update it
only where these fixes need explanation. Regenerate the inventory. Do not raise the
reference cap or silently hand-edit generated blocks.

All original proposed follow-ups have been triaged by the lander and recorded on
`bob-cli-kc`; carry those outcomes into the close note, without filing duplicates. The
navigation page's added content belongs to this epic. The remaining Mac docs budgets and
unrelated plugin timing failures do not.

## 6. Verify and deploy the repaired tree

- Run the focused Copy card tests, then `npm run build`, `npm run build:check`,
  `npm test`, and `npm run docs:inventory:check` in bob-plugins. Run the real Obsidian
  walkthrough above and inspect the screenshots. Check touched fragment lengths.
- Run `just check` in bob-cli for file-change verification. Never run `just check-full`.
  Run `just docs-check` with the current linked mounts and `npm run docs:check` as
  appropriate. Compare failures with the original landing note and existing trackers,
  retaining any new epic-caused issue as this work. Known unrelated failures must be
  reported accurately, not treated as missing Copy card functionality or concealed as a
  green full-suite pass.
- Use `/sase_monitor` before starting long commands. Carry this tale's remaining
  closeout steps in a monitor continuation if needed; there is no separate lander
  waiting to do them. Wait for the staging command to exit before finalizing.
- Deploy the actual plugin checkout using
  `bob plugins sync --repo "$PWD" --no-pull -p bob-navigation-hotkeys` from bob-plugins.
  This explicit path also works across the newer bob-cli `6a88d810` enclosing-checkout
  selection behavior. Follow the linked repo's protected-deploy recovery instructions;
  never force an overwrite without authorization. Deployment and verification may
  operate on the working tree; do not order epic closure after this tale's own future
  commit.

## 7. Finish bob-cli-kc's landing in this tale

Re-read `bob-cli-kc`, all its descendants and notes, and its linked original plan.
Confirm all four original phases remain closed, every original note has an outcome, the
final decisions are honored, and any newer commits on the base branches have been
integrated. Keep the baseline follow-up triage outcomes from the lander's note in the
close note, and distinguish tests run now from older phase evidence.

Run `sase bead epic-symbols bob-cli-kc`. For every exemption keyed to this epic or one
of its phases, resolve it (wire it up, privatize it, add an appropriate non-test pragma,
or delete it) or re-key the Justfile entry only to a still-open later bead that actually
needs it. None existed at the initial landing audit. Do not leave the judgment for
another agent.

Close normally with:

```sh
sase bead close bob-cli-kc --note "<source and commit review, post-start integration, fixes, verification, and every follow-up disposition>"
```

If stale symbols reject closure, clean them and retry. If incomplete descendants reject
it, finish/reopen them. Never use `--force` to advance a successful landing or merely to
make the close command succeed. Force is only for an intentionally canceled/superseded
outcome with the explicit required reason and resolution.

After closure run `just symvision` if available (the initial bob-cli justfile has no
such recipe), and check any opened repository's available Symvision gate as needed. Open
the plans sidecar with `/sase_repo`, audit-read the original artifact, then set
`status: done` in the frontmatter of the original epic plan's resolved PLAN file,
`sase/repos/plans/202611/cmd_y_copy_card.md`. Include that changed repository in
`/sase_final`.

Finally run `sase bead read bob-cli-kc -r "Need the parent link"`. This audit found no
parent, so normally finish. If a parent link was added later, obey the original landing
instructions: a completed phase parent is closed normally and its containing epic is
left to its lander; a plan parent requires review of its previous landing note, all
descendants/notes, linked plan readiness and post-child drift before normal closure,
symbol cleanup, Symvision and plan status update. Repeat only through fully complete
directly parented plans. Record and report any ambiguous or incomplete parent instead of
forcing it.

Finish through `/sase_final`, committing the implementation and all other changed
repositories. The final response must state whether `bob-cli-kc` closed and identify any
remaining external verification limitations.
