---
tier: tale
title: Repair recurrence fallback and checklist guards and land bob-cli-ak
goal: Recurrence marks preserve unreadable rules, guard root checklist rows, and bob-cli-ak
  is verified and closed.
size: small
proposed_by: bbugyi200.apollo.bob-cli-ak.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ak
status: done
---

- **PARENT:**
  [202610/gtd_checklist_glyphs_and_recurrence_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/gtd_checklist_glyphs_and_recurrence_marks.md)
- **BEAD:**
  [bob-cli-ak](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ak/README.md)

# Finish recurrence fallback and checklist guards, then land bob-cli-ak

## Scope and prior verification

Finish only the recurrence defects found by the land audit of epic `bob-cli-ak`, then
close that epic in this same implementation turn. This is a single-agent tale; there is
no later land agent to complete its closeout. Do not wait for this tale's own commit
SHA, push, or CI: the host commits after the turn ends.

Read `bob-cli-ak` and its current notes, then use `sase artifact read` for
`plan:202610/gtd_checklist_glyphs_and_recurrence_marks.md`. Its final decision is
`when_done_label = no`: omit that suffix from visible labels and preserve it in the
tooltip. Keep the feature display-only, retain both existing toggles, preserve
`api.dateMarks` v1 as dates-only, and do not change storage or recurrence semantics.

The lander read every phase and every note (`bob-cli-ak.1`, `.2`, `.3`), reviewed the
source, CSS, tests and documentation diffs, and checked these implementations:

- bob-plugins `328d212`, `f98f97e`, `3e014b9`: PRE/POST identity glyphs and Today
  placement, recurrence in Live Preview/rendered views, and full/short Tasks results.
- bob-cli `8a4936f4`, `6e34d507`, `8e81f3cc`: corresponding display contracts and
  vault-format documentation.
- Vault `b9507dc1`: precisely one `hide recurrence rule` line in the RECURRING query in
  `blocked.md`, verified in an audited checkout of `gh:bobs-org/bob`.

Fetched bob-cli origin/master and opened current bob-plugins origin/master. The only
unrelated commit after the first epic commit is bob-cli `8bcc3e2b`, which adds a
relation-label range test in `src/native/capture_complete/tests/relations.rs` and needs
no mark integration. The additional CLI commit since the original plan baseline,
`bcc201fe`, landed before epic creation and only changes capture relation-label
completion/rewrite; it also needs no mark integration. No unrelated plugin commit landed
in that interval. Recheck newer drift before landing, especially shared plugin fragments
and generated files. The epic has no parent bead as of the landing audit.

The six mark/Today suites pass **169/169**. Evidence:
`file:explicit:ae784b01fc84d2cbb42ff449`. Reproductions of the three defects below,
while the original recurrence suite still passes **21/21**, are in
`file:explicit:17290fc9a7bd57a9561ab5c3`. Use audited artifact reads for either log.

## 1. Repair the three recurrence defects

Open `bob-plugins` through `/sase_repo` and read its `AGENTS.md`. The relevant source is
`plugins/bob-ledger-tools/src/136-recurrence-marks.js`; tests are in
`scripts/test-ledger-tools-recurrence-marks.cjs`, with Tasks integration coverage in
`scripts/test-ledger-tools-date-marks-tasks.cjs`.

1. **Guard the rendering root itself.** `recurrenceMarkRenderedGuard(textNode, root)`
   currently returns false when the nearest task li equals `root`. Reuse the existing
   `fakeRenderEl` / `renderTaskLi` fixture with status `?`, tags `#task #gtd #pre`, and
   `Brush [repeat:: every day]`. The guard is true with the wrapper root but false with
   the li root, and `plugin.renderDateMarksIn(li, { sourcePath: 'example.md' })` wrongly
   emits a mark. Remove that exemption so a root task is evaluated using its own status
   and tags. Add an end-to-end postprocessor regression for li roots, covering PRE and
   POST, the permitted `when done` case, and closed rows. Preserve nested-list/code
   exclusion and behavior for a containing wrapper.

2. **Keep unknown rules verbatim.** `recurrenceWeekdayEntry` and `recurrenceMonthAbbrev`
   index ordinary objects without checking own properties.
   `recurrenceMarkLabel('every constructor')` consequently returns `undefineds`. Make
   both dictionaries safe against inherited names. Assert unknown valid-charset inputs
   such as `every constructor`, `every week on constructor`, and
   `every constructor on the 1st` use the documented verbatim fallback, with a trailing
   `when done` omitted only from the label. Check model/tooltip output and one
   Tasks-host integration case, and retain all existing normal cadence vectors. This is
   display fallback, not a request for a complete Tasks recurrence parser.

3. **Validate before whitespace normalization can repair invalid input.**
   `recurrenceMarkCanonicalField('[repeat:: every\tday]')` currently returns a canonical
   `every day` model, because collapsing whitespace precedes validation. The approved
   contract requires padding spaces only and the Tasks character set
   `^[a-zA-Z0-9, !]+$`. Reject disallowed raw whitespace in field values/padding before
   collapsing valid spaces. Apply consistent source validity to rendered fields and
   Tasks rules; do not silently normalize an unreadable source into a valid mark. Add
   tests for internal tab/NBSP and invalid padding, alongside valid repeated ASCII
   spaces, uppercase cadence text, and the existing bracket/key rejection cases. Invalid
   sources keep native text/pills; Live Preview's existing repeat repair-flag selector
   continues to cover them.

The recurrence fragment currently has 978 lines. Keep every hand-edited fragment at or
below 1,000 lines; remove redundant code or factor into a sibling fragment if needed,
registering it in `src/fragments.json`. Do not edit fragment 276 or generated `main.js`
by hand. Keep the code synchronous and defensive.

## 2. Verify, regenerate, and deploy

Read the `docs.md` reference memory before editing documentation. Extend the existing
`docs/date-marks.md` conformance vectors for the root-li guard, inherited-name fallback,
and raw whitespace rejection. These clarify the existing contract.

Use a patch bump from the then-current bob-ledger-tools version (the audited version was
1.63.0), regenerate plugin facts with `npm run docs:gen`, and build with
`npm run build`. Run `npm run build:check`, `npm run docs:check`, the six focused mark
suites, and `npm test`. Set `BOB_CLI_DIR` to your current bob-cli checkout when running
plugin docs checks, so their script uses the authorized checkout rather than falling
back to its separate managed cache. The six focused suites are:

```sh
node --test scripts/test-ledger-tools-task-tag-marks.cjs \
  scripts/test-ledger-tools-today-marks.cjs \
  scripts/test-ledger-tools-today-marks-reading.cjs \
  scripts/test-ledger-tools-date-marks.cjs \
  scripts/test-ledger-tools-recurrence-marks.cjs \
  scripts/test-ledger-tools-date-marks-tasks.cjs
```

In bob-cli, run `just docs-gen`, `just docs-check`, and **`just check`**. Never run
`just check-full`. Use `/sase_monitor` for a command that cannot finish inline, with an
explicit continuation that finishes this tale and epic closeout; do not end a turn with
an unmonitored command running.

Deploy with `bob plugins sync --repo <opened-bob-plugins-path>`. Confirm the deployed
version and that a dry-run has nothing left to copy. Do not edit the vault query: the
required line already landed. Report visual/device checks as performed only when
actually observed; the original plan's Bryan-in-Obsidian light/dark/iOS checklist does
not become an automated test claim. Exercise the repaired DOM paths through the real
plugin postprocessor, not only its pure helper.

The lander's full npm run on clean `3e014b9` was **3408/3415**: four unblocked-glyph
failures (`bob-cli-6v`), the Keep notice failure (`bob-cli-6w`), reverse-project
conversion leaving `Work_ship#^prj` (`bob-cli-7i`), and the stage-ranker 16 ms assertion
(`bob-cli-3w`, 16.92 ms). The ranker passed 23/23 in isolation unchanged. All three
phase proposals duplicate these four existing reports. The lander handles their
`/sase_new_task` corroboration and records the outcomes on `bob-cli-ak` before proposing
this tale. Read that note and carry those outcomes into the close note; do not create
replacement tasks or expand this tale to repair unrelated baseline failures. Investigate
any new failure caused by this repair before closing. The full-suite evidence is
`file:explicit:d5d1a0b22b67564637f53139`.

## 3. Complete this tale and close bob-cli-ak

After fixes, verification, deployment, and the final drift review are complete:

1. Re-read `bob-cli-ak`, its descendants and notes; confirm every named phase is
   completed and no new epic-caused issue remains. If this tale is represented by an
   open child plan bead, complete and close that tale normally before attempting the
   epic close, so the readiness check has no unfinished descendant. Update the tale's
   linked plan status as appropriate. Do not require its own commit to exist first.
2. Run `sase bead epic-symbols bob-cli-ak`. It was empty at audit time. Resolve every
   listed exemption by wiring, privatizing, adding an appropriate non-test pragma, or
   deleting the symbol under the Symvision policy. Re-key only if a specifically
   identified, still-open later bead needs that exemption. Leave no entry keyed to this
   epic or a closed phase.
3. Close normally with `sase bead close bob-cli-ak --note "<verification>"`. Write a
   concrete note naming the three repaired defects, the reviewed phases and commits,
   decision compliance, integration/drift result, actual test and deployment results,
   and all four follow-up dispositions copied from the lander's triage note. If close is
   rejected for symbols, finish the cleanup and retry. If a phase is incomplete, finish
   or reopen it. Never use `--force` merely to close successfully or advance this nested
   landing. Cancellation/supersession requires a deliberate non-done resolution and
   reason; successful work closes normally.
4. Run `just symvision` after close if available (the current bob-cli justfile has no
   such recipe); report the absence if it remains unavailable.
5. Open the `plans` repository with `/sase_repo`, resolve the epic's PLAN path from
   `sase bead read bob-cli-ak`, and set **`status: done`** in the frontmatter of
   `202610/gtd_checklist_glyphs_and_recurrence_marks.md` in that opened plans repo. Read
   its contents through `sase artifact read`, not an unaudited sidecar read.
6. Run `sase bead read bob-cli-ak -r "Need the parent link"`. The audited epic has no
   parent, so finish normally. If a parent link has been added, follow the user's
   landing rule: a phase parent closes normally only after verifying its scope; a plan
   parent requires its prior landing note, all descendants/notes, linked plan and
   post-child drift/readiness to be rechecked before normal close, symbol cleanup,
   symvision and linked-plan completion. Stop and note the blocker on an incomplete or
   ambiguous ancestor; never force a successful nested landing.
7. Use `/sase_final` as the last action of a normal turn, declaring all repositories
   changed by this tale. No Cargo.toml version, CHANGELOG.md, memory, or unrelated vault
   changes belong in this repair.
