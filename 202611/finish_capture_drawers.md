---
tier: epic
title: Finish capture drawers (bob-cli-ey landing repairs)
goal: 'The drawer contract from the parent epic is fully shipped. `Text @foo+bar#requirements
  #fix` files the note and retypes `^bar` in one transaction. Drawer creation writes
  clean Markdown. bob reports the drawer object, the drawer span fields, refusal codes,
  the near-catalog warning, and the full drawer menu. Bob Mac Capture''s fixtures
  are generated from that real bob, and macOS CI fails only bob-cli-f3''s seven baseline
  tests.

  '
phases:
- id: drawer-writes
  title: bob-cli drawer writes, retype, capture output, and refusal codes
  depends_on: []
  size: medium
  description: 'drawer-writes: let a separate #type on a body-bearing @route+id item
    retype the target in the same transaction. Fix the stray blank line written when
    a drawer is created. Replace parent_section with a per-item drawer {title, key,
    created} object and show ''new drawer'' plus the type row in human and dry-run
    output. Add title/key to drawer spans, JSON codes to every drawer refusal, and
    the drawer_near_catalog warning.'
- id: drawer-menu
  title: bob-cli full drawer completion menu and drawer vocabulary sweep
  depends_on:
  - drawer-writes
  size: medium
  description: 'drawer-menu: build the drawer completion menu: task, recent, and new
    groups; bob-authored detail, consequence, and refused copy; a replacement that
    covers the # and the name; and initial_selection none on an empty query. Rename
    the leftover ''task sections'' shell group and docs wording to drawers.'
- id: mac-ci-regressions
  title: Bob Mac Capture macOS test regressions from the directive and drawer phases
  depends_on: []
  size: medium
  description: 'mac-ci-regressions: fix the twelve macOS-hosted XCTest regressions
    that the mac-directives and mac-drawers phases introduced (explicit completion
    submit, Open Hint, template shelf, unselected type menu). Leave bob-cli-f3''s
    seven baseline failures alone. Verify with a macOS CI run.'
- id: mac-drawer-fixtures
  title: Bob Mac Capture real drawer fixtures and migrated task-type generator
  depends_on:
  - drawer-writes
  - drawer-menu
  - mac-ci-regressions
  size: medium
  description: 'mac-drawer-fixtures: regenerate the drawer fixtures from a bob that
    contains drawer-writes and drawer-menu, and retire the synthetic drawer-contract
    overlays and the parent_section fallback. Move generate-task-type-fixtures.sh
    off the retired attached #type forms, regenerate the task-type fixtures at grammar
    3, update fake-bob and tests, and verify on macOS CI.'
proposed_by: bbugyi200.apollo.bob-cli-ey.land
proposed_with: claude/opus
parent_bead: bob-cli-ey
create_time: 2026-11-01 03:50:51
status: wip
bead_id: bob-cli-ey.7
---

- **PROMPT:** [prompts/202611/finish_capture_drawers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_capture_drawers.md)
- **PARENT:** [202610/capture_drawers_and_directives.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_drawers_and_directives.md)
- **BEAD:** [bob-cli-ey.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.7.md)

# Plan: Finish capture drawers (bob-cli-ey landing repairs)

## Why this plan exists

The land agent for epic **bob-cli-ey** ("Capture drawers and % directives") found that
the closed phases did not fully ship the epic's contract.

- **The parent plan stays normative.** Read it with
  `sase artifact read plan:202610/capture_drawers_and_directives.md "<why>"`. Follow it
  where this plan is silent, especially §2, §3, §6.4, and §9.
- **cli-drawers tale.** The detailed tale for cli-drawers is
  `plan:202610/capture_task_drawers.md`. Its §4 and §5 spell out the output and menu
  contract that this plan finishes.
- **Inherited decisions.** The epic's DECISIONS are final. No memory edits are planned:
  `glossary:task-drawer` already describes the target behavior, and this work makes it
  true.

Verified gaps at bob-cli `ffc5f1f9` and bob-mac-capture `9a1876d`:

| Gap                                                                                                 | Evidence                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Some note goes here. @foo+bar#requirements #fix` and `Some note. @foo+bar #fix` are refused        | `type_marker_not_applicable`: "`#fix` types a task, but this capture adds a child bullet; capture `@<route>+<block-id>#<type>` alone…" The copy also names a retired spelling                                                      |
| Creating a drawer writes a blank line between the new ALL-CAPS header and its first entry           | Note bytes after `n @foo+bar#motivation`: `\t- MOTIVATION\n\n\t\t- n`. Batches reproduce it too                                                                                                                                    |
| No per-item `drawer {title, key, created}` in `capture -f json`                                     | Only `parent_section`; `_drawer_created` is discarded in `capture/sub_bullet.rs`                                                                                                                                                   |
| Human and dry-run output never say "new drawer" or show the retype row                              | `under [ ] Upgrade Postgres  ^bar · MOTIVATON` only                                                                                                                                                                                |
| `drawer` spans carry no `title`/`key`                                                               | `capture-parse` shows only `kind`, `start`, and `end`                                                                                                                                                                              |
| Drawer refusals have no JSON `code`                                                                 | `ambiguous_drawer`, `reserved_drawer`, `drawer_near_duplicate`, and `invalid_drawer` arrive as bare `error` text                                                                                                                   |
| `drawer_near_catalog` is never emitted                                                              | `n @foo+bar#motivaton` creates MOTIVATON silently while MOTIVATION is used in 2 other notes. The create path in `sub_bullet.rs` returns early and drops the first resolve's warning                                                |
| The drawer menu is task rows only                                                                   | `capture_complete/candidates.rs` `drawer_candidates` fills only on-task rows: group `"drawers"`, empty `detail`, no `consequence`, `refused`, recent or new rows, and no `initial_selection`. The replacement starts after the `#` |
| zsh still says `task sections`                                                                      | `capture_complete/shell.rs`, `completion/providers.rs`, `tests/cli/completion/{capture_text,vault}.rs`, `docs/completion.md`                                                                                                       |
| Mac CI 41289488292 on `9a1876d`: Build and Lint pass; Test fails 28 assertions in 19 methods        | 7 are the pre-epic **bob-cli-f3** baseline; 12 are epic regressions (listed under `mac-ci-regressions`)                                                                                                                            |
| Mac drawer fixtures are partly synthetic, and the task-type generator drives retired attached forms | `Tests/Fixtures/drawer-fixtures-README.md` gap list; `Scripts/generate-task-type-fixtures.sh` lines ~219–242                                                                                                                       |

**Stale-binary trap.** bob-cli builds into `$CARGO_TARGET_DIR` (a per-workspace
directory under `~/.cache/sase/tmp/cargo-targets/`), not into `target/`. A leftover
`target/debug/bob` can be hours old. For manual smokes and fixture generation, run
`$CARGO_TARGET_DIR/debug/bob` after `cargo build`.

## Ground rules

These carry over from the parent plan:

- **Thin client.** bob owns grammar, copy, previews, and writes. The Mac renders what
  bob returns.
- **Release train.** Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
  `drawer-writes` removes the `parent_section` JSON field, and `drawer-menu` changes
  drawer row `group` values and the replacement range. Both are wire breaks: use
  `feat(capture)!:` subjects with a `BREAKING CHANGE:` footer that names the change.
  `grammar` stays 3 and `schema_version` stays 1. Bob Mac Capture already decodes the
  planned contract, and `mac-drawer-fixtures` removes its fallback.
- **No compatibility.**
  - No aliases, and no error copy that names a retired spelling.
  - Narrow negative tests are fine.
  - Never touch `CHANGELOG.md`, research reports, plan artifacts, accepted decision
    records, or `tests/fixtures/pomodoro_themes/vectors.json`.
- **CLI and docs rules.** Read `sase memory read cli_rules.md` and
  `sase memory read docs.md`. After any help or doc change, run `just docs-gen` and then
  `just docs-check`. Verify bob-cli with `just check`; never run `just check-full`.
- **Other repos.**
  - Open bob-mac-capture only through `/sase_repo` (`sase repo open bob-mac-capture`),
    and read its `AGENTS.md` first.
  - Its compile rules are mandatory: `just portable-check` on a host with Swift, and a
    best-effort Mac scratch-copy `bundle.sh` plus `just typecheck-tests`.
  - **macOS CI is authorized.** The Mac phases may run `Scripts/macos-ci-check`. It
    pushes only a throwaway `ci/precommit/<id>` branch, waits for that run, then deletes
    the branch; see that repo's `docs/develop/ci.md`. Record the run ID and results in
    the phase bead note.
  - If neither the Mac nor CI is reachable, say "macOS targets unverified".

## Phase `drawer-writes` (bob-cli, medium)

### 1. A separate `#type` retypes the `+id` target (parent plan §2)

- **Root cause.** `capture_language::type_marker::not_applicable_error` only asks
  `is_new_task`. It is called from `capture/plan.rs` (~782) and
  `capture_language/draft.rs` (~658, ~846), and it still refuses every body-bearing
  `SubBullet` subject. `item.rs`'s own gate already admits
  `is_existing_type_marker_subject`.
  - Make every path agree: a body-bearing `@route+id` / `--task-ref` child, with or
    without a drawer, is an `existing` type-marker subject.
  - `capture-parse` stops emitting the diagnostic for it.
  - `capture-complete`'s `task_type` context reports subject `existing` with the
    target's current or inherited type, as Ensure Next retypes already do.
- **Copy.** Delete the `TypeMarkerSubject::SubBullet` teaching copy, which names the
  retired `@<route>+<block-id>#<type>`. Global or forced destinations that still make a
  type marker inapplicable must teach only current grammar.
- **Execution.** Stage the `[type::]` edit with
  `CaptureTaskThemeContext::apply_type_marker_to_existing` before the child write.
  - Move a live Task Link with the existing D8 `relocate_for_retype`.
  - Choosing the inherited type clears an explicit override, as today.
  - No Ensure Next, no new Task Link, and no lane change. Only marker-only
    `@route+id #type` keeps its current lane behavior.
- **One transaction.** An unconfigured type, a closed target, a strict-cap refusal, a
  stale `--task-ref`, a drawer refusal, or a later batch-item failure leaves the vault
  byte-identical.
- **`^prj` targets.** The same applies there.
  `tests/cli/capture/prj_drawers.rs::capture_prj_drawer_separate_type_and_plain_child_still_work`
  currently never asserts the retype; make it assert the resulting `[type:: FIX]`, or
  the project typing rule the parent plan §4 names.

### 2. Clean drawer creation

- **The blank line.** Find and fix the extra empty line written between a newly created
  drawer header and its first entry. The create branch in `capture/sub_bullet.rs` stages
  the header, then re-resolves and inserts against the staged text; check
  `drawer_insertion` on an empty drawer and `insertion_text_preserving_line_endings`.
- **Simplify that branch while there.** Delete the dead `let _ = (insertion, drawer);`
  and the duplicated catalog load and second resolve. Keep header and entry in one
  staged edit sequence, and keep the batch rule: two rows into one new drawer create one
  header.
- **Tests.** Cover single and batch creation, tab and space indentation, CRLF, end of
  file, and creation before a managed log and after a Depends-On first child. Assert
  exact bytes, which no existing test does for creation.

### 3. Output contract (parent plan §3.3 and §3.7)

- **JSON object.** Replace `parent_section` with a per-item
  `drawer: {title, key, created}`, or `null` for a plain child, in `capture -f json`
  items. Mirror it on the top-level first-item fields the way other per-item fields are
  mirrored.
  - Update every `parent_section` construction site (`plan.rs`, `output.rs`, `batch.rs`,
    and the `pomodoro_*`, `project_note`, `task_complete`, and `dependencies` builders).
    Update the tests that read it, including `prj_drawers.rs`.
  - For `^prj` H2 targets, `drawer.title` is the H2 title, and `created` is true when
    the H2 was appended.
- **Type summary.** Drawer and plain-child items carry the existing type-summary or
  relocation fields when a `#type` retypes.
- **Human and dry-run output** state both effects in one block:

  ```text
  foo › ^bar Upgrade Postgres › REQUIREMENTS        new drawer
    - Postgres 17 minimum
  type  BUILD → FIX · Task Link moved under FIX
  ```

  Match the house styling of the existing `under …` block. The exact layout may adapt,
  but "new drawer" and the type row must both appear. Preview and commit share the
  planner.

- **Drawer spans.** `capture-parse` `drawer` spans carry `title` and `key` when the name
  is valid: a slug's key, or a literal's uppercased title. Parsing stays lexical. Spans
  still never overlap the block-ID span.
- **Refusal codes.** Every drawer refusal carries a JSON `code`, both in
  `capture -f json` and in any `capture-parse` diagnostic that can see it:
  `invalid_drawer`, `drawer_text`, `ambiguous_drawer`, `reserved_drawer`,
  `drawer_near_duplicate`, the `!`-toggle refusal (name it `drawer_toggle`), and the
  `^id#` R3 refusal. Reuse existing codes where one already fits.
- **Warning.** On create, emit `drawer_near_catalog`
  (`new drawer REQUIRMENTS · did you mean REQUIREMENTS?`) through the existing capture
  warnings channel, in both JSON and human output. It fires when the new key is within
  edit distance 2 of a catalog key used in at least two notes.

### 4. Docs and tests

- **Docs.** Update `docs/capture.md` (drawer section, retype, and output examples),
  `docs/reference/json-outputs.md` (the `drawer` object, codes, and warning),
  `docs/task-types.md` (separate `#type` on child captures), and any CLI help that
  mentions `parent_section` or the refusal. Then run `just docs-gen` and
  `just docs-check`.
- **Tests: the user example.**
  - REQUIREMENTS existing versus missing.
  - FIX explicit versus the inherited type.
  - A linked parent whose Task Link moves.
  - A closed target and an unconfigured `#type` each leave nothing written.
- **Tests: other cases.**
  - `Some note. @foo+bar #fix`.
  - A batch where a later failure rolls back the retype.
  - Codes for each refusal, and the near-catalog warning.
  - Span `title` and `key`.
  - The `drawer` object with `created` true and false.
- Rewrite the existing suites rather than keep parallel old and new cases.

**Done when:**

- `just check` and `just docs-check` pass.
- `git grep -n parent_section -- src tests docs` finds nothing.
- The scratch-vault smokes in §"Landing smokes" for writes behave as specified.

## Phase `drawer-menu` (bob-cli, medium)

### Trigger and top level

The menu opens when the caret is inside or at the end of a glued `#…` after `@route+id`,
with or without body text. Body-text triggering already works.

- `target: {route, block_id, text}` stays.
- Add `initial_selection: "none"` when the drawer query is empty. Handle it in
  `capture_complete/engine.rs` `initial_selection()`, which today returns early because
  `result.query` is `None` for drawers. The existing `#` and `%` behavior stays.
- `grammar` stays.

### Rows, in order

- **`task`.** The target's drawers in document order, filtered by the typed query. On a
  `^prj` target these are the eligible H2s; keep the current H2 behavior.
- **`recent`.** Catalog keys not on this task.
  - On an empty query, use `capture_drawers::drawer_catalog_recent`: at most 8 keys,
    each used in at least 2 notes.
  - On a typed query, use `drawer_catalog_search`: prefix before substring, by key or
    slug.
  - Load the catalog once per request with the cached `load_drawer_catalog`. The task's
    own rows read only the target note.
- **`new`.** One row, only for a non-empty typed query with no exact key among the other
  rows.

### Row fields

- `title`, `key`, and `group`.
- `entry_count` on task rows; `note_count` and `last_seen` on recent rows.
- `detail`: bob-authored with correct singular and plural, for example `3 notes`,
  `1 note`, `used in 12 notes · 3 days ago`.
- `creates`: true on recent and new rows.
- `refused`: the resolver's reason for reserved, near-duplicate, or ambiguous choices.
  It must come from the same `resolve_drawer_for_write` decision the commit uses, never
  a parallel rule.
- `consequence`, for example `Adds after "2 GB RAM" in REQUIREMENTS`,
  `Adds to REQUIREMENTS` for an empty drawer, and `Creates FUTURE WORK under ^bar`.
  `^prj` rows keep `Adds to ## Requirements`.
- `replacement`: covers the `#` and the name, with the completion envelope's
  `replacement.start` at the `#`. Use `#slug` only when the slug resolves back to
  exactly this title against the full target list; otherwise use `#"TITLE"`. No trailing
  space.
- `cursor_after`.

### Targets and timing

- **A missing, duplicate, or non-task target** returns its diagnostic and no rows. Never
  return a plausible create row.
- **Latency.** Re-time warm and cold `capture-complete` on the drawer menu against the
  `task_type` menu on the same draft. ey.2 recorded 19.6 ms cold and a 23.7 ms warm
  median on a 400-note fixture. Record the new numbers in the phase bead note; the
  target is warm within about 20 ms of `task_type`.

### Vocabulary sweep

In zsh drawer completion and `capture --drawer` value completion, change the group label
`task sections` to `drawers`:

- `capture_complete/shell.rs` (~258);
- `completion/providers.rs` (~238, ~270);
- `tests/cli/completion/{capture_text,vault}.rs`;
- `docs/completion.md` (~141, ~502).

Do not rename `task_status_hooks`' unrelated `grouped task sections`.

**Done when:**

- Unit and CLI coverage exists for each group, an empty versus typed query, singular and
  plural detail, refused rows (reserved, near-duplicate, ambiguous), consequences,
  quoted replacements, `initial_selection`, the missing-target diagnostic, and a `^prj`
  target.
- `docs/reference/json-outputs.md` and `docs/completion.md` describe the menu.
- `just check` and `just docs-check` pass.
- This grep finds only the hidden alias registration, its alias test, the justfile alias
  smoke, the `mkdocs.yml` redirect, `grouped_task_sections`, and generated
  decision-record text:

  ```bash
  git grep -n -i -e task_section -e task-section -e 'task section' -- ':!CHANGELOG.md' ':!sase/'
  ```

## Phase `mac-ci-regressions` (bob-mac-capture, medium)

These macOS-hosted XCTests fail on master `9a1876d` (CI run 41289488292, job
124797557661). They are not among **bob-cli-f3**'s seven pre-epic failures, so the
epic's Mac phases caused them:

| Test                                                                                                            | Failure                                                                                                                                       |
| --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `CaptureOpenHintTests.testChipShowsWhenDraftHasNoPreviewYet`                                                    | :13 XCTAssertTrue failed                                                                                                                      |
| `CaptureOpenHintTests.testLegacyPreviewResultsFillPlanWhenStoredPlanIsMissing`                                  | :93 XCTAssertTrue failed                                                                                                                      |
| `CapturePanelModelTests.testExplicitCompletionSubmitWaitsForCompatibleParse`                                    | :364 timeout; :367 `"Call @cash"` ≠ `"Call cash"`                                                                                             |
| `CapturePanelModelTests.testExplicitCompletionSubmitStopsWhenParseFails`                                        | :404 `"Call @cash"` ≠ `"Call cash"`                                                                                                           |
| `CapturePanelModelTests.testExplicitCompletionSubmitIsCanceledWhenPanelRetainsAndHidesDraft`                    | :478 timeout; :481 same                                                                                                                       |
| `CapturePanelModelTests.testExplicitCompletionSubmitIsCanceledWhenBobClientIsReplaced`                          | :523 timeout; :528 same                                                                                                                       |
| `CapturePanelModelTests.testNewSubmitGestureDoesNotInheritPendingExplicitContinuation`                          | :562 timeout                                                                                                                                  |
| `CaptureSavedTemplateShelfTests.testRefreshLoadsPopulatedListAndKeepsLastGoodOnFailure`                         | :23 `1` ≠ `3`                                                                                                                                 |
| `CaptureTaskTypeHostedTests.testUnselectedMenuTabTakesTheFirstRow`                                              | :61 `"jot idea #build"` ≠ `"jot idea #build "`                                                                                                |
| `CaptureTaskTypeHostedTests.testUnselectedMenuUpSelectsTheLastRow`                                              | :86 `Optional(5)` ≠ `Optional(2)`                                                                                                             |
| `CaptureTemplateShelfModelTests.testCurrentWireDeleteRemovesTheRequestedTemplateAndReportsMissingIDs`           | :263, :264 XCTAssertTrue failed                                                                                                               |
| `CaptureTemplateShelfModelTests.testCommandSPresentsBobsNonTemplateRefusalAndDoublePressDoesNotStartSecondSave` | :630 got bob's real copy "add `%t` to the draft's first item to save it as a template", expected "only `%t` drafts can be saved as templates" |

How to fix them:

- **Find each root cause separately.** For each test, decide whether the product code or
  the test's expectation or fixture is wrong.
  - Bob's copy is authoritative: the Mac shows bob's refusal text and never hard-codes
    it.
  - Never weaken an assertion or delete a test to go green.
  - Async tests must never await an untimed continuation.
- **Linux runs miss these.** Earlier repairs passed `just portable-check` on Linux, but
  these failures appear only in the hosted `BobMacCapture*` targets. Use
  `Scripts/macos-ci-check`; the Mac scratch typecheck is best-effort.
- **Done when:**
  - A macOS CI run on the phase's tree shows none of the twelve.
  - Only the seven tests **bob-cli-f3** tracks may still fail. If a fix incidentally
    repairs one of them, note it on bob-cli-f3; do not otherwise take on f3's scope.
  - `just portable-check` passes.

## Phase `mac-drawer-fixtures` (bob-mac-capture, medium)

### Drawer fixtures

- **Generate from real bob.** Build bob from bob-cli master that contains
  `drawer-writes` and `drawer-menu`; use `$CARGO_TARGET_DIR/debug/bob` and see the
  stale-binary trap above. Bump the revision pin in
  `Scripts/generate-drawer-fixtures.sh` from `e660c8d8` to that commit, then regenerate.
- **Retire the synthetic overlays.**
  - Turn the `drawer-contract-*` overlays into real generated fixtures (empty open,
    typed recent, typed new, refused, and the new-drawer-plus-retype capture), or delete
    them and point the tests at real equivalents.
  - Empty the README gap list.
  - Add a `^prj` target case, since `cli-project-drawers` has landed.
- **Drop the fallback.** Remove the Swift fallback from `drawer` to `parent_section`.
  Adapt any decoding where the real wire differs from the overlays, keeping the thin
  client.

### Task-type fixtures

- **Generator inputs.** In `Scripts/generate-task-type-fixtures.sh`, replace the retired
  attached inputs with current grammar:
  - drop the attached cases: `@dev^parser#`, `@dev^parser#fix`, `@dev+current#`,
    `@dev+current#fix`, and their completion cases;
  - add separate-token cases: `Fix the parser @dev^parser #fix`, `@dev+current #fix`,
    `note @dev+current #fix`, and `note @dev+current#requirements #fix`;
  - add a negative case where `@dev^parser#fix` reports the R3 refusal.
- **Regenerate** every `task-type-*` fixture at grammar 3, and update
  `CaptureTaskTypeTests` and the other decoders.

### Wiring, docs, and verification

- **fake-bob.** Update `Tests/Fixtures/fake-bob` for the regenerated fixtures.
- **Docs.** Update the Mac docs (capture-previews, task-pickers, protocol) wherever the
  real contract differs from what they say, for example the retype row on child
  captures.
- **Verification.**
  - `just portable-check` passes.
  - `Scripts/macos-ci-check`, or a master CI run, shows only bob-cli-f3's seven
    failures.
  - This grep finds no stale references:

    ```bash
    git grep -n -i -e task_section -e 'task section' -e parent_section -e 'drawer-contract'
    ```

## Landing smokes (for this epic's land agent)

Run these against a scratch vault with a freshly built `$CARGO_TARGET_DIR/debug/bob`,
using `capture --dry-run -f json` plus a real write:

- `Some note goes here. @foo+bar#requirements #fix`: one block with a drawer object,
  type `BUILD → FIX`, and no blank line.
- `Some note. @foo+bar #fix`
- `@foo+bar#requirements`: refused with code `drawer_text`.
- `n @foo+bar#motivaton` with MOTIVATION in two other notes: the `drawer_near_catalog`
  warning.
- `n @foo+bar#future-work` with both `FUTURE WORK` and `FUTURE-WORK`:
  `ambiguous_drawer`.
- `capture-complete` on `note @foo+bar#`, `note @foo+bar#mo`, and `note @foo+bar#perf`:
  the groups, `initial_selection`, and replacements.
- The parent plan's §9 smokes: `jot idea %n`, `Build output %c3`,
  `{0} @job+search#leads %t` with rows, `https://example.com/essay %l`, a bare `#`, and
  a bare `%`.
- Confirm no `Cargo.toml` version or `CHANGELOG.md` edits, and confirm the `!` subjects.

The child epic's land agent then resumes the interrupted **bob-cli-ey** landing through
the `parent_bead` link. That landing closes bob-cli-ey and sets `status: done` on
`plan:202610/capture_drawers_and_directives.md`. This epic has no closeout phase for
bob-cli-ey.
