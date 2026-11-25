---
tier: tale
title: Repair Obsidian drawer landing gaps and close bob-cli-ph.7
goal: Preserve drawer titles and visual states, repair epic-owned checks, and complete
  the nested landing.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ph.7.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ph.7.1
status: done
---

- **PARENT:**
  [202611/obsidian_true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/obsidian_true_case_drawers.md)
- **BEAD:**
  [bob-cli-ph.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.7.1.md)

# Repair Obsidian drawer landing gaps and finish bob-cli-ph.7

Finish the remaining work in **bob-cli-ph.7.1**, then close that epic and its parent
phase **bob-cli-ph.7** in this coding turn. This is one bounded plugin repair and
acceptance pass. The original four implementation phases are closed; do not reimplement
their feature or create another phased epic.

## Context and verified baseline

Read `bob-cli-ph.7.1` and its landing notes, and audited artifact
`plan:202611/obsidian_true_case_drawers.md`. Its parent is the phase `bob-cli-ph.7`,
within the still-running epic `bob-cli-ph`. The approved glyph is the **tray**. Keep
dual-read legacy support, source-backed decoration, authored title bytes, native editor
history/folding, first-note Notes birth, and existing requirement and conversion
rollback rules. No migration, Mac, memory, CLI grammar, or ALL-CAPS cutover work belongs
here.

Use `/sase_repo` to open `bob-plugins` and `plans`; use only its returned paths and read
the plugin AGENTS.md. All `plugins/` and `scripts/` paths below are relative to
bob-plugins. Edit fragments, generate bundles with `npm run build`, and use the
documented non-forcing per-plugin deployment procedure. Read the docs memory before any
documentation edits.

The landing audit reviewed every child note and the epic commits:

- `f2e8c62a`: bold writers, canonical fixture copies, conversion;
- `2796d63d`, `5ed3bd22`, `1753d730`: tray rendering, export ordering, inventory;
- `8d3c555c`: drawer typing and Toggle Drawer;
- `8254caa4`: native Obsidian acceptance and input precedence;
- bob-cli `0742684d`: behavior documentation.

At audit time bob-cli is `e532b11f`; bob-plugins was tested at `8254caa4` and
fast-forwarded to `689da092` (unrelated property-editor acceptance changes). Neither
checkout had authored source edits. Fetch and inspect subsequent drift before editing,
preserving concurrent work.

The focused drawer/conversion/copy/typing/settle/self-test sweep passed 186/186. The
three stale cycler expectation suites failed 10 of 190 tests. The isolated
event-link-times, frees-marks, and relation-chips suites passed 68/68. A targeted Rust
CLI reproduction was interrupted with exit 143 during a cold compile; this landing does
**not** claim a new Rust test result or a `just check` pass.

The existing actual-Obsidian report is `file:explicit:e9988819b2a7e52753c34291` (32/32).
Audited screenshots include light Live Preview `file:explicit:99c1ffea61076da96225acc2`
and dark reading `file:explicit:e51daf687f9c74c956f3709f`. These reveal the rendering
defects below despite the report being green. The other views are light reading
`file:explicit:a95e8a34e6fb850dac4ed7ce` and dark Live Preview
`file:explicit:d0f7de1722a08229a8ef40a4`.

## 1. Preserve title whitespace through real conversion

In `plugins/bob-navigation-hotkeys/src/100-yank-and-paths.js`,
`formatProjectReversalSectionTitle` and `projectReversalSectionRefusal` collapse
internal whitespace before calling `drawer_for_heading`. This contradicts the approved
title-byte contract. The actual harness demonstrates:

```js
const { helpers: h } = require('./scripts/navigation-hotkeys-harness.cjs');
h.drawer_for_heading('API  Design'); // {ok:true,title:'API  Design'}
h.formatProjectReversalSectionTitle('API  Design'); // wrong: 'API Design'
h.formatProjectReversalSectionTitle('Café\tDesign'); // wrong: 'Café Design'
```

Pass the authored heading title through the shared mapping without collapsing internal
spaces or tabs. Keep appropriate Markdown heading-edge handling, named refusals, the
legacy promotion mapping, Notes-role behavior, and the comparison-key normalization used
to find matching existing sections; do not change identity semantics to fix
serialization.

Add full promotion/reversal regression cases with repeated spaces, tabs, Unicode and
empty eligible sections, including LF/CRLF and preserved rollback on refusal. Pure DH
checks already pass and are insufficient: exercise
`buildProjectSeedFromChildBulletsReqs`, section insertion, and
`buildTaskBlockFromProjectNote` via their existing harnesses. Extend the real conversion
acceptance fixture to include at least one repeated-whitespace title and assert saved
bytes both ways.

## 2. Finish tray rendering and make visual acceptance meaningful

Work in ledger-tools' `264-plugin-task-drawer-model.js`,
`264-plugin-task-drawer-marks.js`, and `styles.css` only as needed. Preserve
source-backed owner/count attributes and the tray design; no added display nodes, marker
replacement widgets, document edits, or new title parser.

- **Duplicate Live Preview bullet:** the light screenshot shows a gray native note dot
  immediately beside each tray. Hiding the list token's text color does not hide every
  native bullet pseudo-element. Suppress the native dot only at annotated drawer headers
  while retaining native fold controls, indentation, caret/source reveal, and ordinary
  note dots. Verify both themes.
- **Reading-view strike leakage:** the dark reading screenshot visibly strikes through
  the done owner's `Café` drawer title. A child's computed `textDecorationLine: none`
  does not prove an ancestor decoration is absent. Prevent ancestor strike painting
  through the drawer title, preserving the owning task's completion appearance and
  layout. Verify supported native tight/loose-list shapes and avoid changing unrelated
  descendant typography.
- **Folded count:** reading CSS currently places
  `content: " · " attr(data-bob-drawer-count)` on a child `p` or `strong`, but the count
  attribute is on `li`; that cannot render the number. Put the generated count on an
  element that owns the source-backed attribute while keeping it adjacent to the header,
  with no extra DOM nodes. Verify actual native folding and the visible `· 2` count in
  reading view and Live Preview. Confirm count refresh after an edit and unfold/refold.

Strengthen `scripts/obsidian-task-drawers-look-e2e.cjs`: its current folded branch
accepts either `assert.ok(true)` or the unchanged `data-*` count when no folded badge
appears. Require an observed native folded state and a visible count on the correct
pseudo-element. Its current caret check only tests that the row exists; assert native
delimiter reveal and restoration when leaving the header. Replace weak geometry
assertions with meaningful bounds on the marker slot and compare nested note typography.
Verify settings-off cleanup and no document changes. Add regression coverage through the
existing ledger harness where it can test behavior; use screenshots and real Obsidian
for painted ancestor strike and actual marker geometry, not computed style on a
descendant alone.

## 3. Repair epic-owned assertions and preserve concurrent integrations

Update only stale **new writer output** expectations in:

- `scripts/test-task-status-cycler-req-toggle.cjs`;
- `scripts/test-task-status-cycler-open-line.cjs`;
- `scripts/test-task-status-cycler-task-reqs-vectors.cjs`.

The 10 failures follow the `f2e8c62a` writer switch to `**Notes**`. Their presence on a
later phase's clean base does not make them unrelated to this epic. Preserve all
legacy-input/reuse assertions, line-ending checks, cursor coordinates, first-note birth,
and no-duplicate-home checks. Do not blanket replace `NOTES`: it remains the identity
key and valid legacy input. If canonical fixture changes are actually needed, change
Rust-owned fixtures first and copy them to every consumer; do not diverge plugin copies.

The landing drift review found concurrent `o`/`O` work in `a971285b`, managed-link split
repair `eac9d744`, taskSettle commits `824d8e49`, `f764a4be`, `c9454208`, and
property-editor autosave/deletion fixes. Retain these changes. Recheck the parent
phase's coordination note: shared Notes creation stays pure and reports `created`; a
newly opened drawer-indent sibling accepts `*`; Smart Enter opens a drawer child while
`o` opens a sibling after its subtree. Add a focused composed regression for open-line
then drawer typing, and ensure a taskSettle move retains bold and legacy drawer subtrees
unchanged. Use the existing APIs rather than duplicating recognition. Do not implement
the separately assigned `bob-cli-q4.12` notes key.

Concurrent bob-cli changes through `e532b11f` include capture place/bold containers
(`45a54345`, `22fea308`), task settle (`93956838`, `007ef5ed`, `08a708d3`), split
(`814b6583`), snooze (`cd594550`) and the drift meter. The renderer still uses the
shared contextual twin, so these do not justify a new plugin grammar. Recheck current
canonical copies and drawer-focused Rust coverage before closeout.

## Follow-up triage already completed by the lander

Carry these outcomes into the final close note. Do not create duplicate tasks:

- `.7.1.2` note #1, `.7.1.3` #1, `.7.1.4` #3 (NOTES expectations): declined as separate
  follow-ups; epic-caused, repaired in step 3.
- `.7.1.3` #2 and `.7.1.4` #2 (docs budgets): corroborated `bob-cli-jz` (hard navigation
  cap) and `bob-cli-ob` (editing headroom/advisory API cap). Canonical counts at
  8254caa4: navigation 6068/6000, pre-epic 6046; plugin-apis 9929/8000 warning,
  pre-epic 9287. Concurrent q4 commits own the +22 and +642 increments; evidence was
  also recorded on `bob-cli-q4`. This drawer epic's conversion doc replacement is
  net-zero words.
- `.7.1.4` #4: corroborated `bob-cli-km` (event-link-times) and `bob-cli-io`
  (frees-marks), with attributed full-suite failures and a fresh 68/68 isolated combined
  control.
- `.7.1.4` #5: corroborated `bob-cli-lp` (MN-click), with the same passing isolated
  control. No new task.
- `.7.1.4` #6 (17 Rust CLI failures): attached to active `bob-cli-q4` as a
  `DISCOVERED ISSUE`, with the original phase's reported affected modules, source
  evidence from its new settle choke points, and explicit disclosure that the lander's
  cold compile was interrupted before tests. No new task and no invented fresh
  reproduction. Evaluate actual new gate results; unexplained drawer failures remain
  this epic's work.

## 4. Verify, deploy, and record evidence

Run the repaired suites, drawer/requirement twin copies, conversion suites,
open-line/requirement typing, ledger marks, taskSettle composed regression, and drawers
E2E self-test. Run `npm run build`, `npm test`, `npm run validate`, and
`npm run docs:check` in bob-plugins. In bob-cli run the required docs generation/check
gates and **`just check`**. Do not run `just check-full`. Use `/sase_monitor` for long
commands and resume this same tale's remaining closeout after the result. Do not broaden
this tale into repairing known unrelated baseline failures; retain precise commands,
revisions, diagnostics, and triage references. Repair every epic-caused failure before
closing.

Run the real Obsidian drawers acceptance in its isolated scratch vault/profile with the
strengthened visual and whitespace cases. Use the documented Athena CDP/Xvfb workflow if
needed (read tailnet memory before remote access). Never run acceptance mutations in the
live vault. Publish the new report and all four inspected light/dark Live
Preview/reading screenshots using `/sase_artifact`, attach refs to the epic and
`bob-cli-ph.7`, and describe the fixed duplicate-dot, strike, folded-count, and
conversion evidence.

Sync changed plugins via
`bob plugins sync --repo <opened-plugin-repo> --no-pull -p <plugin-id>` following the
repo's non-forcing conflict procedure. Record each changed plugin, tested/deployed
revision and dirty-build status, and bundle markers on `bob-cli-ph.7` for the later
migration gate. Historical source commits are now resolved: writer `f2e8c62a`, renderer
`2796d63d` plus `5ed3bd22`, typing `8d3c555c` plus `8254caa4`. Markers are
`bob-task-drawer-bold-writes-v1` in cycler/navigation, `bob-task-drawer-render-v1` in
ledger-tools, and `bob-task-drawer-typing-v1` in cycler. Do not guess this turn's
eventual SHA. The closeout below is **not** ordered after this work's own commit, push,
or CI result; close in the same turn as the verified final code.

## 5. Finish the landing in this tale

This tale has no separate land agent and nothing automatically resumes the interrupted
parent landing. Complete these operations yourself:

1. Re-read `bob-cli-ph.7.1`, its four child phases and all their notes, the original
   plan, and any new drift. Verify all requested behavior and repairs above are
   complete. All four original phases were already closed normally.
2. Run `sase bead epic-symbols bob-cli-ph.7.1`. Resolve every entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting it; re-key only when a
   still-open later bead actually needs that exemption. The audit found none, but
   recheck. Do not leave stale whitelist entries.
3. Close with
   `sase bead close bob-cli-ph.7.1 --note "<verified behavior, commits/drift integration, gate results, repaired gaps, deployment and artifact refs, and every follow-up disposition above>"`.
   If rejected, finish the named cleanup/work and retry. Never force merely to succeed;
   never use `--force` to advance this successful nested landing.
4. Run `just symvision` if available and confirm a clean whitelist. The audited bob-cli
   justfile currently has no symvision recipe; record that fact if still true. Through
   the opened plans repo, set `status: done` in the frontmatter of
   **`202611/obsidian_true_case_drawers.md`**, the plan resolved by
   `sase bead read bob-cli-ph.7.1`.
5. Re-read the parent link with
   `sase bead read bob-cli-ph.7.1 -r "Need the parent link"`, verify this child
   completed `bob-cli-ph.7` including its coordination note, and check
   `sase bead epic-symbols bob-cli-ph.7`. Resolve/re-key any entries and close only that
   phase normally using
   `sase bead close bob-cli-ph.7 --note "<verified drawer rendering, typing, writers, conversion, coordination, checks, deploy markers/revisions and acceptance refs>"`.
   **Leave `bob-cli-ph` open for its waiting land agent**; do not mark
   `202611/true_case_drawers.md` done or close other phases.
6. Submit `/sase_final` for every changed repository, including plans and plugins. This
   verified same-turn code plus the ordinary closeout is the intended landing; no
   successor must wait for this turn's future SHA.
