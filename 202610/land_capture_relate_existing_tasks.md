---
tier: tale
size: medium
title: Finish and land epic bob-cli-7r (capture `&`/`*` on existing tasks)
goal:
  Fix the defects that epic bob-cli-7r's own commits left in bob-cli and Bob Mac
  Capture. Afterwards, parse and execute name the same offending token, every ⛓ row
  reports its own "already" state and the prerequisite's real post-write status, a `*`
  sub-bullet's `under` row shows its arrow, the `+` owner picker disables only rows the
  writer refuses, the Mac preview never repeats an unchanged owner line, and the epic's
  macOS CI regressions are gone. Then close the epic.
proposed_by: bbugyi200.athena.bob-cli-7r.land
bead: bob-cli-7r
status: done
---

- **PARENT:**
  [202610/capture_relate_existing_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_relate_existing_tasks.md)
- **BEAD:**
  [bob-cli-7r](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7r/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-7r.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-7r.land.md)
- **COMMITS:**
  - [14505e5](https://github.com/bobs-org/bob-cli/commit/14505e5751dc4fea1c15e8ed87add9c89ceb574b)
    — fix(capture): land remaining existing-owner relationship defects (A1-A7)

# Finish and land epic bob-cli-7r

Epic **bob-cli-7r** ("Capture `&` and `*` relate existing `@route+id` tasks,
first-class") has all seven phases closed. Its design and contract is
`plan:202610/capture_relate_existing_tasks.md`; read it with
`sase artifact read plan:202610/capture_relate_existing_tasks.md "<why>"`. Its section
numbers (§2.4 and so on) are used below. The landing review found the epic-caused
defects listed here. Everything else the phases reported was verified, and `just check`
passes on bob-cli `master` at `75a1809`. Fix only the items below, then perform the
closeout in Part C **in this same turn**. Nothing resumes the landing after you.

Do not edit any SASE memory file. The epic's DECISIONS forbid it, and the decisions
strand is tracked separately as bob-cli-7t. Do not run `just check-full`.

## Reproduction vault

Most items reproduce in a scratch vault outside the repo, for example `/tmp/7r-tale`:

- `foo.md`:
  - `- [ ] #task Buy shoes ^bar`
  - `- [ ] #task Buy socks ^socks`
- `body.md`: `- [/] #task Exercise ^excercise`
- `job.md`: `- [*] #task Do cleanup tasks ^cleanup`
- `plain.md`: `- [ ] #task Plain ^plain`
- `old.md`: `- [x] #task Old thing ^old`
- `project.md`:
  - frontmatter `type: [[project]]`, `status: wip`
  - `- [ ] #task Project root ^prj`
  - `- [ ] #task Other resident ^other`
- `resident.md`: frontmatter `parent: [[project]]`, then
  `- [ ] #task Resident ^resident`

Copy the Tasks plugin status settings, so statuses read Ready / In Progress / Next /
Blocked rather than Todo. Take
`tests/fixtures/tasks_parity/vault/.obsidian/plugins/obsidian-tasks-plugin/data.json`,
or copy (read-only) from `~/bob/.obsidian/plugins/obsidian-tasks-plugin/data.json`. Run
`bob` with `BOB_NOW="2026-10-14 10:20:00"`, `NO_COLOR=1`, and
`BOB_DAY_FILE=<scratch>/day.md`. A day file whose `## Pomodoros` section links Exercise
in a running Pomodoro reproduces RX14.

## Part A: bob-cli

**A1. Parse and execute must name the same `relationship_needs_text` token (§2.3).**

- `bob capture-parse -f json -- '@body+excercise &foo:bar p:2 s:3'` reports
  `` `s:3` needs task text ``, but `bob capture` reports `` `p:2` needs task text ``.
  `% p:2` disagrees the same way.
- The editor side (`src/native/capture_language/editor_parse.rs`, the
  `relationship_needs_text` block near `spans.iter().find(...)`) takes the first span in
  vector order, which is the last token in the source. Execution
  (`first_relationship_text_token` in `capture_language/dependencies.rs`) takes the
  first token in the source.
- Make the editor pick the earliest span by `start`, and keep the diagnostic range on
  that token.
- In `tests/cli/capture/relate_existing.rs` (RX9), assert the exact identical message
  from parse and execute for `p:2`, `s:3`, `%`, and the multi-token drafts.

**A2. Per-prerequisite "already" (§2.4 Repeats).**

- `print_human_dependency_line` (`src/native/capture/output.rs`) derives `already` from
  the whole update (`added == 0`). After `@body+excercise &foo:bar` has landed,
  `@body+excercise &foo:bar &foo:socks` prints `⛓ now depends on "Buy shoes"`, which is
  wrong.
- Add an additive per-prerequisite JSON field to `PrerequisiteJson`:
  `already_present: true`, omitted when false (`skip_serializing_if`). Set it where
  prerequisites are resolved against the pre-existing links.
- Human rows use it per row: `⛓ already depends on "Buy shoes" …` and
  `⛓ now depends on "Buy socks" …`.

**A3. ⛓ rows report the prerequisite's post-write status (§2.4: "each relationship row
ends with the status of the task on its far side").**

- The problem: commitment transfer (`plan_promotions`, called near "Promote open
  prerequisites" in `src/native/capture/dependencies.rs`) can raise a prerequisite's
  status. With the In Progress owner Exercise, `@body+excercise &foo:bar` writes Buy
  shoes `[ ]` → `[/]`. Yet the row and `prerequisites[].status_symbol` still say
  `[ ] Ready`, because `ResolvedPrerequisite` snapshots the status before the write.
- JSON:
  - Make `PrerequisiteJson.status_symbol`/`status_name` the final staged status.
  - Add additive `previous_status` and `previous_status_name` on the prerequisite only
    when the write changed it. This mirrors the update-level names.
  - The JSON shape is otherwise unchanged.
- Human row: `⛓ now depends on "Buy shoes"  foo.md ^bar  [ ] → [/] In Progress`. This
  uses the same arrow rhythm as `🧱` rows; with no change the row stays `[ ] Ready`.
- Cover it in `relate_existing.rs`:
  - RX14 (linked `[/]` owner): human output and JSON.
  - A new-task case still shows no arrow when nothing transfers.

**A4. The `under` arrow for `*` sub-bullets (§2.4 Sub-bullet).**

- `parent_under_marker` (`output.rs`) only looks at `Dependent` updates.
  `Note @plain+plain *job:cleanup` moves Plain `[ ]` → `[*]` but prints
  `under [*] Plain`.
- Derive the arrow from the parent's pre-capture status, whichever relationship changed
  it.
- Add an RX12-style test with `*`, covering live and `--dry-run`.

**A5. The `+`/`@route+` owner-picker project guard over-disables (§2.6
`a project cannot depend on its own resident`).**

- The problem: `project_resident_relation` and its caller in
  `src/native/capture_complete/candidates.rs` disable on note membership only. The
  writer refuses only when the **dependent is the project's own open `^prj` line**
  (`prj_context_for_dependent` plus `refuse_internal_prj_targets` in
  `src/native/capture/project_deps.rs`). Today:
  - `&resident:resident +` disables `@project+other`, yet
    `@project+other &resident:resident` succeeds.
  - `*project:other +` disables `@project+prj`, yet `@project+prj *project:other`
    succeeds.
- The fix:
  - For `&` modifiers the candidate is the dependent: disable only when the candidate is
    `^prj` of a recognized project and the prerequisite is in that note or a descendant.
  - For `*` modifiers the typed blocked task is the dependent: disable only when that
    task is such a `^prj` and the candidate is in its note or a descendant.
  - Fold the duplicated `recognized_project_paths` logic into one helper.
- `blocked_task_candidates` (the `*` picker) has the same pre-existing flaw, tracked as
  **bob-cli-7s**. If your shared helper fixes it with a trivial change, do it, add a
  test, and close bob-cli-7s with
  `sase bead close bob-cli-7s --note "<what you verified>"`. Otherwise leave bob-cli-7s
  untouched.
- Tests (`tests/cli/capture/complete_relationship_owner.rs`):
  - The two cases above are enabled.
  - `&resident:resident` beside `@project+prj` still disables `^prj`.
  - `*project:prj +` still disables `project.md`/`resident.md` rows.

**A6. Test holes the review found** (in `relate_existing.rs` /
`complete_relationship_owner.rs`):

- RX7 through the forced path: `--route old --task old -- 'Note *job:cleanup'` is
  refused with the §2.3 message, and the vault is byte-identical.
- RXC2 for `^body:excercise *`: the Exercise row is disabled with
  `the owner task itself`.

**A7. Docs and help.**

- `docs/capture.md`: two owner JSON examples still show
  `"owner": { "kind": "new_task" }` without the `text` field that RXC3 added. Regenerate
  both from live output:
  - the `capture-complete … 'Buy Groceries! &foo'` example (around line 1276);
  - the `'Ask Kelly *jo'` example (around line 1503).
- Document the A2/A3 additive prerequisite fields in the capture JSON wire section.
- Re-verify byte-for-byte that the "Relating existing tasks" terminal examples match
  live output after A2–A4. The combined linked-owner example will now show the Buy shoes
  transition.
- Update the `bob capture --help` text only if a quoted example changed.

Run `just check` (fmt, clippy, full `cargo test`) until it is green.

## Part B: bob-mac-capture

Open the linked repo with your `/sase_repo` skill
(`sase repo open bob-mac-capture -r "<why>"`), use only the printed path, and read its
`AGENTS.md` first.

**B1. The epic's macOS CI regression.**

- In `Tests/BobMacCaptureTests/CaptureDependencyPanelTests.swift`,
  `installBlocksPicker()` uses draft `"@mac_inbox+ask-kelly *"` (22 bytes, `*` at
  offset 21) with `replacement: CaptureRange(start: 22, end: 23)`.
- Change it to `CaptureRange(start: 21, end: 22)`. This fixes the three failures on CI
  runs 38647350710 and 38656909109, which never failed before e41dee2:
  - `testBlocksPickerInsertsStarReplacement`
  - `testBlocksPickerIDlessPromptReturnsToBlocksPicker`
  - `testRemoveBlocksPickerTriggerDeletesStarToken`
- Recompute every other `CaptureRange` that e41dee2 or e5b2ef3 touched in tests.

**B2. Do not repeat an unchanged owner line (plan phase mac_preview step 3).**

- `CaptureTaskBlockPresentation.covers`
  (`Sources/CaptureCore/CaptureTaskBlockPresentation.swift`) only counts
  `added`/`changed` lines. A `task_dependency` whose owner line comes back `unchanged`
  therefore re-renders the verbatim `previewBlockLines` stack and the `relativeTarget`
  line under the owner header and card (`CapturePanelView.swift`, where
  `taskDependencyCovered` is computed). Two cases trigger it:
  - an owner that already has `[id::]` and keeps its status, for example `*` onto an
    owner that stays Ready;
  - a repeat with `added == 0`.
- Make `covers` treat the owner's block as covering it for `task_dependency`, whatever
  the line's change kind.
- Add a fixture recorded from such a dry run, plus a presentation test asserting it is
  covered.

**B3. Relationship rows consume A2/A3**
(`Sources/CaptureCore/CaptureRelationshipPresentation.swift`, the `rows(for:)` ⛓
branch):

- Decode the new prerequisite fields additively with `decodeIfPresent`.
- Use per-prerequisite `already_present` for the "already depends on" wording. For an
  older Bob that omits it, fall back to the update's `added == 0`.
- Render `Ready → In Progress` when `previous_status` is present, using the same
  `statusName` and arrow logic as the 🧱 branch. The accessibility label follows the
  text.
- Update the README preview paragraph if its example row changes.

**B4. Re-record the relationship fixtures from the bob built in Part A, in the
Tasks-settings scratch vault.**

- The existing `Tests/Fixtures/task-dependency-preview-*.json` fixtures were recorded
  without Tasks plugin settings, so they freeze `status_name` "Todo". That was
  bob-cli-7r.7's follow-up.
- Never hand-edit them. Regenerate with `target/debug/bob capture --dry-run -f json` for
  the same drafts the fake-bob routes key on.
- Update the presentation and model tests and the README text that quote "Todo" or the
  old prerequisite status.

**B5. Verify.**

- Run `just portable-check` on athena.
- Then best-effort compile the macOS targets as `AGENTS.md` describes: rsync a scratch
  copy without `.build`/`.git` to `mac`, run `bundle.sh` and `just typecheck-tests`
  there, then delete the copy.
- The Mac (`ssh mac`) was offline during the landing. If it still is, say in your final
  report that the macOS targets are unverified. The macOS CI run of your pushed commit
  is the backstop, and you must not wait for it.
- Known unrelated reds:
  - `RefsInspectorScrollTests` (bob-cli-7h);
  - `RefsPanelModelTests` (bob-cli-7p);
  - a midnight-UTC `NSCalendarDayChanged` crash, recorded on bob-cli-66.

## Part C: close the epic (same turn, after A and B are green)

1. Run `sase bead epic-symbols bob-cli-7r`. It listed no entries at landing time. If any
   appear, resolve each: wire it up, privatize it, add a non-test pragma, or delete it
   per the Symvision epic-whitelist policy. Re-key an entry only when a still-open later
   bead needs it.
2. Run
   `sase bead close bob-cli-7r --note "<what you verified: A1–A7 and B1–B5 results, just check / portable-check status, macOS verification status>"`.
   Never use `--force` to make the close succeed. If it is rejected, fix the cause
   (leftover epic symbols, or unclosed phases, which should not exist) and close again.
3. Run `just symvision` if the recipe exists. bob-cli's `justfile` had none at landing
   time; if it is still absent, note that in the close and skip it.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. That is the PLAN
   path that `sase bead read bob-cli-7r -r "<why>"` prints for
   `plan:202610/capture_relate_existing_tasks.md`; it is currently `status: wip`. That
   plans repo is then part of your commit obligations.
5. bob-cli-7r has no `parent_bead`, so nothing further cascades.
