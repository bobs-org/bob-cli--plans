---
tier: tale
size: medium
title: Finish and land the task-types epic (bob-cli-e6)
goal:
  Fix the leftover task-types work the land agent found in bob-cli, bob-plugins, and
  bob-mac-capture. Deploy the plugins, then close epic bob-cli-e6 and mark its plan
  done.
decisions:
  type_mark_hash:
    ask:
      How should an explicit task type draw its hash so it sits flush with the type
      name?
    choices:
      text:
        '"#" and the name become one text run; same baseline, zero gap, matches the
        ledger #BUILD'
      glyph:
        Keeps the custom hash glyph but tightens its box; alignment depends on the SVG
        margins
    default: text
    why:
      One text run is the only way to guarantee a shared baseline and no gap on every
      host
    answer: text
proposed_by: bbugyi200.apollo.bob-cli-e6.land
proposed_with: claude/opus
bead: bob-cli-e6
decided_by: auto
create_time: 2026-10-30 02:34:37
status: wip
---

- **PARENT:**
  [202610/task_types.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_types.md)
- **BEAD:**
  [bob-cli-e6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e6/README.md)

# Finish and land the task-types epic (bob-cli-e6)

Epic `bob-cli-e6` (plan `plan:202610/task_types.md`, read it with
`sase bead read bob-cli-e6 -r "..."`) has all 16 phases closed. The land agent verified
the shared contract (D1–D12), the Rust planners and vectors, the `type_mismatch` lint,
the plugin mirror, and the docs gates. It found the leftovers below. All of them were
caused by this epic or are steps its plan required, so they are epic work. The land
agent already triaged the phases' `PROPOSED FOLLOW-UP:` notes and recorded them on the
epic (the "LAND TRIAGE" note). Do not refile them.

Read `cli_rules.md` and `docs.md` memory with `/sase_memory_read` before touching CLI
help or docs. Open the linked repos only with `/sase_repo` (`bob-plugins`,
`bob-mac-capture`) and read each repo's `AGENTS.md`. In bob-plugins, edit `src/`
fragments only, keep each fragment at or under 1000 lines, and run `npm run build`.
Register any new test file in `package.json`. Never edit the bob-cli `Cargo.toml`
version or `CHANGELOG.md`. Never write Bryan's live vault notes.

Wire names stay unchanged (D1). That covers JSON keys and codes, command IDs, CSS
classes, API member names other than the removed `planThemeChange`, the `.bob/themes`
journal, and config keys. Change only words a person reads.

## 1. bob-cli gates (caused by e6.12 and e6.14)

- Run `cargo fmt --all`. Today `cargo fmt --check` flags
  `src/native/capture_pomodoro_name.rs`, `src/native/task_move/command.rs`,
  `src/native/task_theme.rs`, `src/native/pomodoro_themes/tests.rs`, and
  `tests/cli/task_move.rs`. It also flags `tests/gcal_list.rs`, which belongs to epic
  `bob-cli-ea`. Formatting that file with the rest is fine.
- Fix the two clippy gate errors from e6.12:
  - `src/native/capture/pomodoro_link.rs:671` `needless_option_as_deref`: use
    `task_theme_context` directly.
  - `src/native/task_complete/successors.rs:2193` `collapsible_if`: collapse it into a
    let-chain.
- Fix the stale CLI test
  `tests/cli/ref_library/today.rs::ref_library::today::ledger_and_resolution_refusals`
  (around line 887). Its `empty` vault has only closed entries. Typed placement (D7) now
  creates a `— READ` placeholder for the reading task, so the command succeeds instead
  of failing with `no_open_pomodoro`.
  - Assert the new success: `capture.creates_pomodoro == true` and the READ name.
  - Keep `no_open_pomodoro` covered with a vault where the task has no type to place by:
    `plan.reading_type: off` and no `plan.default_type`. Confirm it still refuses; if it
    does not, find out why before changing the test.
  - Update the `no_open_pomodoro` row in `docs/ref.md`'s error table to match, for
    example "No open entry, no selector, and the reading task has no type to place by".

Expected leftovers that are not this epic's: clippy `clone_on_copy` at
`src/native/gcal/classify.rs:75`, the catalog-count tests (59 vs 58), and completion
kinds `gcal:day`. All come from `bob-cli-ea.5` and are recorded on `bob-cli-ea`
(`bob-cli-ea.6` reports fixing the last two). Leave them alone. When `just check` fails
only on those, the epic's gate work is done.

## 2. bob-plugins: successor-wiring tests (caused by e6.12)

`scripts/test-task-status-cycler-successor-wiring.cjs` has 4 failing tests, bisected to
`6562b63`:

- Ctrl+Enter carries per-task theme intents through both pick normalizers.
- Default placement and a changed suggestion still link selected successors without
  retargeting themes.
- A project-finished successor writes its selected theme with real task-theme
  transforms.
- The successor theme save stays truthful when the later daily-link write fails.

`resolveSuccessorThemeWrite` (`task-status-cycler/src/085c-successor-theme-write.js`)
now requires `api.themes.effectiveTypeFor` and `planRetype`.
`successorOfferTypedDestination` also uses `typeAgrees` and `planPlacement`. The test's
`installRealTaskThemeApi` still fakes the old surface (`suggestFor`, `taskThemeOf`,
`planThemeChange`), so every write silently bails out.

- Rebuild that harness from the real Ledger Tools helpers (`main.js` `.helpers` exports
  `planRetype`, `planPlacement`, `typeAgrees`, `typeComponents`). Mirror the argument
  mapping in `279-plugin-task-theme-api.js` (`taskTypePlanRetype` and friends) with
  sources `{vocab, exempt: ["PLAN"], inventory: []}`.
- Then check each of the 4 tests against D6 and D7 and against the successor section of
  `docs/develop/plugin-apis.md`. Change an assertion only where the epic's contract
  changed the behavior. If a test exposes a real production bug, fix the code.
- Check `scripts/test-task-status-cycler-successor-theme-write.cjs` for the same stale
  stub.

## 3. bob-plugins: Pomodoro rename count preview and tests (e6.14 #3)

`8b262bb` made the Ctrl+Shift+M entry rename apply D6 retypes, in
`bob-navigation-hotkeys/src/650-plugin-move-commit.js`. It also relocates the live link
on note moves (`655-plugin-inbox-route.js` `planNoteMoveLedgerRelocation`). It added no
preview and no tests.

- **Preview.** Show the D10 count line on the rename picker row before Enter, for
  example `3 tasks become BUILD`. Use the same fresh-preview helpers the commit uses
  (`resolvePomodoroBulletMoveTypePreview`, `pomodoroBulletMoveTypePlanRows`) and the
  footer pattern in `200a-pomodoro-bullet-type-move.js`
  (`pomodoroBulletMovePickerConsequence`). Container and label results retype nothing,
  and the preview must say so.
- **Toast copy.** The rename toast suffix still says `isn't one of your themes`. Change
  it to the D4 copy (`… isn't one of your types`).
- **Node tests** (new file registered in `package.json`, or extend
  `scripts/test-navigation-hotkeys-pomodoro-split-rename.cjs` /
  `scripts/test-navigation-hotkeys-task-move.cjs`):
  - rename retypes a no-field task, clears on inherited, and skips container, label, and
    repair cases;
  - the preview count line;
  - a buffer changed between preview and commit refuses;
  - a note move without a field relocates the live link when the inherited type changes;
  - a task with an explicit field does not relocate.
- Update `docs/bob-navigation-hotkeys.md` for the rename preview.

## 4. bob-plugins + bob-cli: #TYPE alignment (Bryan, epic note #3)

Bryan's screenshot shows `# FIX`: a small raised hash glyph, then a gap, then the name
on a lower baseline. He wants `#FIX`, aligned, with no whitespace. The geometry is in
`bob-ledger-tools/styles.css` (around lines 1779–1850):
`.bob-task-tag-mark[data-bob-type]` has `padding: … 1.28em`, the `::before` glyph box is
1.08em, and `::after` draws the name.

> [!decision] type_mark_hash = text Typed marks hide the `::before` mask glyph and draw
> `content: "#" attr(data-bob-type)` in `::after`, with the hash in the same ink. Set
> the host's padding to the capsule padding only. Repair draws `#?`. The inherited
> toggle draws `#BUILD` in its quieter ink.

> [!decision] type_mark_hash = glyph Keep the mask glyph. Shrink its box and the host's
> left padding until the name abuts the glyph with no visible gap. Center the glyph on
> the name's x-height instead of the 0.8em hash box.

- In either branch, keep:
  - the 📖 book and `#gtd #pre`/`#post` glyph variants: glyph plus name, also flush
    (`📖BUILD`);
  - the hover capsule, ink, and states;
  - the D11 contrast test;
  - Tasks query rows still hiding the hash.
- Add a Node assertion for the chosen geometry in the existing tag-mark or type-identity
  test.
- Update bob-cli `docs/task-tag-marks.md` (the `data-bob-type` paragraph near line 167,
  and the reading-view bullet near line 235) and bob-plugins `docs/bob-ledger-tools.md`.

## 5. Remaining user-facing "theme" copy (e6.15 #2, e6.16 #4)

D1: every person-facing surface says "type". Rename these strings:

- **bob-navigation-hotkeys**
  - `src/490-plugin-lifecycle.js:123`: the command name `Pomodoro themes` becomes
    `Task types`. Keep the command id.
  - `src/202-pomodoro-themes-manager.js` and
    `src/203-pomodoro-themes-manager-actions.js`: notices, input labels, results labels,
    and aria labels ("theme action", "theme color", "Theme occurrences", "New theme
    name", "Could not rename the theme", "Pomodoro themes need the desktop app", and so
    on).
  - `src/313-task-card-theme-projection.js:152` `inherited theme`.
- **bob-command-mode** `src/160-sheet-rank.js:55`: the title mirrors the renamed
  command.
- **bob-ledger-tools**
  - `src/239-plugin-day-strip-theme-model.js:75`: "Pomodoros with several types …".
  - `src/239-plugin-day-strip-lanes.js` lane titles and aria text: "themes only" and
    "and themes".
  - `src/239-plugin-time-lanes-model.js:35`: "not one of your types".
  - `src/279-plugin-pomodoro-theme-chips.js:77`: the command name, in type words.
  - `src/279-plugin-task-theme-marks.js` tooltips: `Task theme …` becomes `Task type …`.
  - `src/050-plan-budget.js:806`: `… is an inventory label, not a type`.
  - Keep the `bob themes returned …` messages in `279-plugin-pomodoro-theme-mirrors.js`.
    The plugin really spawns `bob themes`, the alias that the installed bob 0.4.0 still
    needs.
- Re-run the land agent's scan to catch stragglers: user-facing string literals
  containing `theme` under `plugins/*/src`. Fix them the same way and update any test
  that pins the old strings.
- Update bob-plugins docs that quote these names. Run `npm run docs:check`.
- **bob-mac-capture.** Keep it a thin client (`decisions:mac-capture-is-a-thin-client`).
  - `Sources/CaptureCore/CapturePlanBudgetPresentation.swift:58`: `Themes N/M` becomes
    `Types N/M`.
  - Accessibility labels in `Sources/BobMacCapture/PomodoroThemeViews.swift:145` and
    `Sources/BobMacCapture/CapturePlanBudgetMeterRow.swift:49`: `… theme` becomes
    `… type`.
  - `Sources/CaptureCore/CaptureAgendaPresentation.swift:706` agenda words.
  - Update the matching tests and any Mac docs that quote these strings.
  - Leave the `daily review suggests` arm alone; task `bob-cli-e5` owns it.
  - There is no Swift on apollo. Run `just portable-check` where Swift is available:
    athena over Tailscale SSH (`sase memory read tailnet.md -r "..."`), as e6.15 did. If
    that is impossible, say so in the epic close note and leave it to macOS CI.

Out of scope: bob-cli messages that name a config or note key (`themes:`, `theme:`). The
legacy keys are valid aliases, and Bryan's config still spells `themes:` until
`bob-cli-ec`.

## 6. Retire `api.themes.planThemeChange` (closeout, e6.16 #2)

The plan says to remove the deprecated wrapper once no plugin calls it, and no plugin
does. The only reference outside tests is a fallback option read in
`block-id-prompt/src/077-pomodoro-picker-intent.js:291` (`opts.planThemeChange`).

- Remove the `planThemeChange` member and `pomodoroThemePlanThemeChange` from
  `bob-ledger-tools/src/279-plugin-task-theme-api.js`, and drop the 077 fallback.
- Remove `pomodoroThemePlanChange` and other legacy write-rule helpers too, if `rg`
  shows no remaining caller.
- Update the tests that use them:
  - `scripts/test-ledger-tools-pomodoro-themes.cjs` (the member list near line 856, and
    the cases near lines 900 and 1017–1045);
  - the successor-theme-write and successor-wiring stubs.
- Remove `planThemeChange` from `docs/develop/plugin-apis.md` (around lines 204 and
  237–247).

## 7. Verify and deploy

- bob-cli:
  - `just check`: the only failures may be the `bob-cli-ea` items listed in step 1;
  - `just docs-gen`, then `just docs-check`.
- bob-plugins:
  - `npm run build`, `npm run build:check`, and `npm test`. The only failure that may
    remain is the grey-pill card-view test owned by `bob-cli-e8` ("theme menu stays grey
    when hueFor is missing or throws"). Do not fix it;
  - `npm run docs:check`.
- bob-mac-capture: `just portable-check` (see step 5).
- Deploy with `bob plugins sync --repo <bob-plugins checkout> --no-pull` for every
  plugin you changed.
  - bob-navigation-hotkeys is pinned by a dirty `bob-cli_16` build at `a42de4d`. That
    build is a strict subset of master `74d6da2`, so force it:
    `bob plugins sync --repo <checkout> --no-pull -p bob-navigation-hotkeys -F`.
  - Then run `bob plugins list`. If every plugin is in sync, add a note to `bob-cli-e4`
    and close it (`sase bead close bob-cli-e4 --note "..."`). Otherwise note what is
    still stale.

## 8. Close out epic bob-cli-e6 (final step)

1. Run `sase bead epic-symbols bob-cli-e6`. It was empty at landing. Resolve every
   listed `--epic-symbol` entry (wire it up, privatize it, add a non-test pragma, or
   delete it). Re-key one to a still-open bead only if that bead still needs it.
2. Close the epic: `sase bead close bob-cli-e6 --note "<verification>"`. The note should
   cover:
   - the land agent's verification (16 phases reviewed against D1–D12, contract vectors
     byte-identical in all three copies, docs gates green, concurrent commits
     integrated, follow-ups triaged in the LAND TRIAGE note);
   - what this tale fixed in steps 1–6;
   - the final gate results;
   - the deploy result.

   Never use `--force` merely to make the close succeed. If the close refuses because of
   leftover epic symbols, clean them up and close again.

3. Run `just symvision` if that recipe exists. It does not exist today; then say so.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read bob-cli-e6 -r "..."` prints (`plan:202610/task_types.md`, in the
   plans sidecar). Its frontmatter currently says `status: wip`.

The epic has no `parent_bead`, so nothing further cascades.
