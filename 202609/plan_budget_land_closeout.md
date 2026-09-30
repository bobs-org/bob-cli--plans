---
tier: tale
title: 'Land bob-cli-2o: fix plan-budget and #now gaps, then close the epic'
goal: 'The plan-budget and #now surfaces in bob-cli, bob-plugins, the vault dash and
  the Bob Mac Capture README match the epic spec: a mistyped plan block never breaks
  other commands, the Rust and JS engines agree, and docs are final. Epic bob-cli-2o
  is closed and its plan file is marked done.'
size: medium
proposed_by: bbugyi200.apollo.bob-cli-2o.land
bead: bob-cli-2o
status: done
---

- **PARENT:**
  [202609/pomodoro_plan_budget_now_tag.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/pomodoro_plan_budget_now_tag.md)
- **BEAD:**
  [bob-cli-2o](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2o/README.md)

# Plan: Finish and close epic bob-cli-2o (plan budget, `#now`, ledger guardrails)

## Context

Epic `bob-cli-2o` ("Close the day, tag the week") has all 13 phases closed. Its spec is
the epic plan `plan:202609/pomodoro_plan_budget_now_tag.md`: read its "Design" section
and the phase sections this tale cites. `docs/plan.md` is the authoritative rule text.
The land agent checked every phase against the code in bob-cli, bob-plugins, Bob Mac
Capture, the vault and the config.

- **Passing:** `cargo test` (1267 lib + 596 cli), `cargo fmt --check`, bob-plugins
  `npm test` (800/800) and `npm run validate` (6/6).
- **Deployment:** the deployed plugins match the repo, and Mac CI is green on `b020df7`.
- **Clippy:** the only clippy error is the pre-existing deny at
  `tests/cli/capture/pomodoro_name.rs:808`, which epic `bob-cli-28` owns. Do not fix it
  here.
- **Follow-ups:** already triaged and recorded on the epic. Do not re-triage them.

That review found the defects and spec deviations below. All were caused by this epic,
so this tale fixes them and then lands the epic.

**Repos.**

- **bob-cli:** your workspace.
- **bob-plugins:** `sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md`.
- **Bob Mac Capture:** `sase repo open gh:bobs-org/bob-mac-capture -r "<why>"`. The
  linked primary checkout does not exist on this host.
- **Vault:** `~/bob`. Edit in place and never commit it by hand; `bob vault-sync` syncs
  it.

Follow `sase/memory/cli_rules.md` (read it with `/sase_memory_read`) for any help-text
change.

## Part A — bob-cli fixes

1. **A mistyped `plan:` block must not break other commands (regression).**
   - `RawConfig` (`src/native/config.rs`) now has `plan: Option<RawPlan>`, with typed
     fields. `load_priority_property`, `load_highlights_config` and `load_gkeep_config`
     all deserialize the whole `RawConfig`, so a value like `max_themes: many`,
     `strict: "yes"` or `exempt: GTD` now makes `bob capture … p:N`, `bob randomize`,
     highlights and `bob gkeep` fail with a parse error.
   - Fix: store `plan` in `RawConfig` as an untyped `serde_yaml::Value`, and do all
     typing and validation in `load_plan_config` / `parse_plan_config`. The same bad
     values must still produce `ConfigError::Invalid` there, so `bob plan` still exits 2
     and the other plan surfaces still fall back to the defaults.
   - Add tests showing that each of the other three loaders succeeds when the `plan:`
     block is mistyped.
   - `config.rs` is 1554 lines. Move the plan-config types, parsing and tests into a
     child module (for example `src/native/config/plan.rs`, turning `config.rs` into
     `config/mod.rs`, or an equivalent split). Re-export the same names so callers are
     unchanged, and bring the file under about 1500 lines.

2. **`bob plan` header.**
   - `src/native/plan_budget/cli.rs` (about line 286) prints `no Pomodoros section`
     whenever there are no entries and no themes. It prints that even when the section
     exists but holds only `()` placeholders or closed entries.
   - Decide this from the ledger's `has_section` instead. Add a CLI test with a
     placeholder-only section.

3. **Status follows rule 8.**
   - `assemble_report` (`src/native/plan_budget/mod.rs` about line 146) sets
     `status: over` when only NOW is over. Rule 8 (`docs/plan.md` "Status") says status
     depends only on themes and links.
   - Keep the `now_cap_exceeded` lint and `now.over`, but stop NOW from changing
     `status`.
   - Add engine tests:
     - `now_cap_exceeded` with status `ok`;
     - `plan_link_cap_exceeded` (the engine has no direct test for it today).

4. **An empty link target means the daily note (rule 6).**
   - Today `[[#^x]]` and `[[2026/20260930#^x]]` in the same daily note count as two
     links, in both Rust and the bob-ledger-tools JS mirror.
   - Rust: let `compute` learn the daily note's vault-relative path without `.md` (add
     an optional parameter or a sibling `compute_for_daily`). Treat these three targets
     as the same key: an empty target, that path, and its basename. Callers that know
     the day file must pass it: `bob plan`, task-status-hooks `sync.rs`, `pomodoro.rs`
     tmux, `capture/budget.rs` and `capture_complete.rs`.
   - Add a conformance example 8 to `docs/plan.md` and a matching unit test. Part B
     mirrors it in JS.

5. **The capture budget only when the ledger changed.**
   - `append_plan_budget` (`src/native/capture/budget.rs`) attaches `plan_budget`
     whenever the batch stages the daily note, and loads the config before it checks
     anything. Fix both:
     - attach `plan_budget` only when the `## Pomodoros…` section text differs between
       `original_target` and `updated_target`;
     - load the plan config only after that check, so an invalid config adds its one
       warning only to captures that touch the ledger.
   - Add CLI tests:
     - a capture that edits another section of today's note has no `plan_budget`;
     - with an invalid config, a non-ledger capture has no "plan budget unavailable"
       warning.

6. **Create-row theme preview.**
   - `plan_themes_after_for` (`src/native/capture_complete.rs` about line 1364) adds 1
     for every new name. Split the name into components the way `plan_budget` does.
     Count only the components that are not exempt and not already themes: `GTD` adds 0,
     and `A + B` with `A` already present adds 1.
   - Carry the exempt list in `PlanCreationHint`.
   - Use `.contains()`. This also removes the only clippy warning this epic introduced.
   - Test both `pomodoro_name` and `pomodoro_start_name` rows.

7. **`#now` completion where execution rejects it.**
   - The `now_tag` completion (`completion.rs` about line 349) is offered after `=x #n`
     or `@r:id #n`. Execution rejects `#now` on items with no body text, so offer the
     candidate only where `#now` would be accepted. Add a test.

8. **Human `bob plan` layout.**
   - When no entry is timed, the `exempt` cell overflows its column and shifts the links
     column. Size the column to the widest cell.
   - On a TTY, keep the whole exempt row dimmed after the inner cell's style reset.
   - Update the example in `docs/plan.md` ("`bob plan`"): exempt rows report 0 links,
     not `1 link`.

9. **Docs and help.**
   - `docs/plan.md` "Surfaces" must be final:
     - drop the "Phase" column;
     - use the real strings:
       - tmux: `<status> · plan T/Tc · L/Lc | `;
       - capture: `→ under GOALS (next up)`, `→ into running GOALS (0945-1015)`,
         `→ new Pomodoro BOB`;
       - `dash.md` chips.
     - Verify each string against the code.
   - `docs/capture.md` span-kind list (about line 2508): add `pomodoro_close_drop`, and
     `now_tag` if it is missing.
   - `src/native/capture_language/markers.rs` (about line 561): the `=x0` hint should
     also mention `=x0~2`.
   - Add a `~` example to the `bob capture` examples.
   - The `capture-parse --help` Needs list (`capture_parse.rs` about line 221): add
     `now_tag`.
   - `tmux-pomodoro --help`: list `BOB_CONFIG_FILE`, which the meter now reads.
   - `docs/task-status-hooks.md`:
     - the plan line prints right after the stats line (about line 763);
     - add `plan_budget` to the stable-fields JSON example.

10. **Missing tests.**
    - Strict-mode refusal for the project-note path (`@r^id+#NAME`) and the toggle path
      (`@r+id#NAME`).
    - Execution of `^r:id=x~K` and `<text> @r:id=x~K` closes.
    - An end-to-end capture that checks the written line
      `- [ ] #task <body> #now [created:: …] ^id`.

11. **File size.** `tests/cli/capture/parse_pomodoro.rs` (1627 lines) went past 1500
    because of close-drop. Move the close/drop tests into a new sibling test module and
    register it the way the other modules are.

**Gate:**

- `cargo fmt --check` and `cargo test` pass.
- `cargo clippy --all-targets --all-features` shows no new warnings, and no error
  besides the bob-cli-28 deny at `tests/cli/capture/pomodoro_name.rs:808`.
- Update `README.md` wherever behavior text changed.

**Mac fixtures.** Build `bob` from this tree (`cargo build`) and re-run the commands
behind each Bob Mac Capture fixture that has `plan_budget`, `role` or
`plan_themes_after` (see the fixture conventions in that repo's tests).

- If any output changed, regenerate those fixtures, push, and confirm macOS CI is green
  with `/sase_monitor` before the closeout.
- If nothing changed, say so in the close note.

## Part B — bob-plugins fixes

1. **`bob-ledger-tools` render child.**
   - `renderPlanBlock` (about line 2759) passes a plain `{unload}` object to
     `ctx.addChild`. Obsidian calls `child.load()` on it.
   - Use `new MarkdownRenderChild(el)` from `require("obsidian")`, overriding
     `onunload`, with a guarded fallback for test harnesses.

2. **Live re-render.**
   - Subscribe to the Tasks plugin's `obsidian-tasks-plugin:cache-update` event on
     `app.workspace`. The vault runs Tasks 8.4.0, which fires it.
   - Keep the 5 s interval only when that plugin isn't loaded.
   - Re-render on `metadataCache` "changed" only when the changed file is a rendered
     block's target note.
   - Fix the misleading comment.

3. **Invalid config.**
   - Today `coercePlanCaps`/`loadPlanCaps` fall back per field. Fall back to the full
     default block on any invalid value, as Rust does, and say so once.
   - Test it.

4. **Rust parity.** Add test vectors for each of these:
   - Entries are column-0 `- [c] ` with exactly one status character, as in Rust's
     `capture_pomodoros` scan: `- [] …`, `-\t[ ] …` and `- [ab] …` do not count.
   - Normalize CRLF before splitting lines.
   - Mirror conformance example 8 (the empty target means the daily note). Pass the
     target path from `planBudget({path, content})` and from the block.

5. **NOW predicate parity.**
   - Make `nowBudgetFromTasks` match the Rust native Tasks engine's semantics for the
     NOW query (see `src/native/dataview/tasks/`): the `tags do not include #hide`
     matching rule, case handling for `folder does not include _templates` and
     `path does not include _conflicts`, and NON_TASK statuses counted as done.
   - Add vectors for `#hide/x`, `#Hide`, the path case, and NON_TASK.

6. **`bob-navigation-hotkeys` `#now` removal.**
   - `removeNowTagFromLine` (about line 13300) collapses every double space on the line,
     including inside field values (`[why:: a  b]`). Collapse only the whitespace next
     to each removed token.
   - Test it.

7. **link-picker.**
   - When a counted `N<Ctrl+Shift+P>` is clamped to a single link, still show the clamp
     subtitle (`getLinkPickerSessionSubtitle`, about line 13173).
   - Make "via Task Links" visible in the priority Notice card, not just its aria text
     (about line 20372), for example `P2 → 3 tasks via Task Links · …`.
   - Add a link-mode priority test.
   - Show the pinned `#now` picker row's detail text once, not twice.

8. **Small cleanups.**
   - Replace the literal NUL byte in the link key in `bob-ledger-tools/main.js` (about
     line 2408) with the `\u0000` escape.
   - Update block-id-prompt's `manifest.json` description to mention the plan-budget
     suffix.

**Finish.**

- Bump the minor version of each touched plugin and update its README row.
- Pass `npm test` and `npm run validate`.
- Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>` for each
  touched plugin.

## Part C — Vault and Mac README

1. **`~/bob/dash.md` NOW chip.** It still displays and compares against the hard-coded
   `NOW_CAP = 15` (lines 35 and 109).
   - Use `nowBudget().cap` and `.over` when the API returns them.
   - Keep 15 and the inline count only as the fallback.
   - Keep the file's formatting.
   - Run `bob vault-sync`, then confirm with `bob vault-sync status --json`.
2. **Bob Mac Capture `README.md`** (about line 136). It says an older Bob without `role`
   gets no destination row. The app actually renders `→ NAME` (test
   `testMissingRoleFallsBackToBareName`). Correct the sentence.

## Part D — Install and close the epic (final step)

1. Run `cargo install --path . --locked --force` from this bob-cli tree, so apollo's
   hooks and tmux use the fixed `bob`.
   - Spot-check `bob plan`, `bob plan -f json` and `bob tmux-pomodoro`.
   - Use `BOB_DAY_FILE=~/bob/2026/<latest existing day>.md`, because the host clock is
     UTC (see bob-cli-2q).
2. Run `sase bead epic-symbols bob-cli-2o`.
   - For each listed `--epic-symbol` entry, resolve the symbol (wire it up, privatize
     it, or delete it) or re-key it to a still-open bead.
   - It currently lists none.
3. Close the epic:
   `sase bead close bob-cli-2o --note "<what you verified: the Part A–C fixes, gates run, deploys, Mac fixture outcome, and that follow-up triage is recorded in the epic notes>"`.
   - Never use `--force` to make the close succeed.
   - bob-cli-2o has no `parent_bead`, so nothing further up needs closing.
4. Run `just symvision` if the justfile has that recipe (today it doesn't; say so).
5. Set `status: done` in the frontmatter of the epic's plan file. That file is the PLAN
   path shown by `sase bead read bob-cli-2o -r "Need the plan path"`
   (`plan:202609/pomodoro_plan_budget_now_tag.md`).
6. **Final response.** Report the fixes and the gates. Also hand Bryan the rollout
   checklist from the epic plan's rollout phase, step 6. In addition, tell him:
   - the two new `gtd_daily.md` chores ("Pick today…", "Weekly review…") were flipped to
     `[?]` by a MacBook sync after creation, so confirm that was intended;
   - bob-cli-2q tracks the evening day-flip.
