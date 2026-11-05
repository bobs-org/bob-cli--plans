---
tier: epic
title: 'offclock_types: off-clock by area and task type'
goal: 'Area and project notes accept an `offclock_types` list that sends only tasks
  of the listed effective types off the clock, with the same result in bob-cli and
  the Obsidian plugins, and the vault''s love.md and body.md are migrated so that
  only their MEET tasks are off-clock.

  '
decisions:
  cutover:
    ask: 'When should love.md and body.md drop `offclock: true` after the dual-key
      step?'
    choices:
      gate: A staged gate removes it once you confirm every host has run just install-all
      followup: Both keys stay for now; the phase records a follow-up and you remove
        it later
      now: Removed right away; hosts not yet upgraded record love/body MEETs as Worked
    default: gate
    why: Hosts not yet upgraded keep today's rule until you confirm every host runs
      the new code
    answer: gate
  memory_decision_record:
    ask: Add an offclock_types decision record and mark off-clock-follows-tag-then-note-default
      superseded in part?
    memory:
    - decisions
    default: true
    corrects: 'project `offclock: true|false` on the task''s PT13 root walk; else
      on-clock.'
    answer: true
  memory_glossary:
    ask: Update the off-clock-pomodoro, area-note, project-note, and event-task glossary
      terms for offclock_types?
    memory:
    - glossary:off-clock-pomodoro
    - glossary:area-note
    - glossary:project-note
    - glossary:event-task
    default: true
    corrects: 'A session goes off the clock exactly when it keeps at least one [[task-link]]
      and every one resolves off-clock: `#off` (deprecated `#offclock`, `#off-clock`,
      `#untracked`) or an inherited area/project `offclock: true`, with `#on` and
      `offclock: false` as overrides'
    answer: true
phases:
- id: clock-type-filter
  title: bob-cli resolver, lints, shared vectors, and contract docs
  depends_on: []
  size: medium
  description: 'clock-type-filter: parse the `offclock_types` key, extract a catalog-free
    effective-type helper shared with type suggestions, give HomeResolver type rules,
    apply the nearest-declaration rule in the clock walk, add the shadowed lint and
    key-naming messages, extend the shared clock vectors with a `config` member and
    the new cases, and update every bob-cli doc that states the clock rule.

    '
- id: clock-surfaces
  title: gcal list preview and running-entry re-stamps on naming
  depends_on:
  - clock-type-filter
  size: medium
  description: 'clock-surfaces: give the `bob gcal list` probe line `[type:: MEET]`
    so preview matches sync; make `bob capture-pomodoro-name` and capture''s `#type`
    relocations that rename or move a running entry recompute that entry''s clock
    key in the same write; document that every other retype is corrected at close.

    '
- id: type-rename-coverage
  title: bob types rename and split cover offclock_types lists
  depends_on:
  - clock-type-filter
  size: medium
  description: 'type-rename-coverage: `bob types rename` rewrites matching `offclock_types`
    items in flow, block, and scalar form inside its existing previewed transaction,
    removing duplicates when two types merge; `split` leaves the lists alone and counts
    them in its leftover warning; update docs/task-types.md.

    '
- id: plugin-mirror
  title: bob-plugins clock mirror and vector copies
  depends_on:
  - clock-type-filter
  size: medium
  description: 'plugin-mirror: in the bob-plugins linked repo, teach bob-ledger-tools''
    `taskEffectiveClockResolution` the same `offclock_types` rules with a type helper
    that also handles closed tasks, copy the shared vectors byte for byte into both
    plugin fixture directories, make the test harnesses honor `config`, update plugin
    docs, rebuild, test, and run `bob plugins sync`.

    '
- id: memory-records
  title: Decision record and glossary updates
  depends_on:
  - clock-type-filter
  - clock-surfaces
  - type-rename-coverage
  size: small
  description: 'memory-records: through /sase_memory_write, add the new decisions
    strand, mark off-clock-follows-tag-then-note-default as superseded in part, update
    the four glossary terms, then run `sase memory init`, `just docs-gen`, and `just
    docs-check`.

    '
- id: vault-migration
  title: Dual-key love.md and body.md, then the cutover
  depends_on:
  - clock-surfaces
  - plugin-mirror
  size: small
  description: 'vault-migration: record the drift baseline, add a block-list `offclock_types:
    [MEET]` beside `offclock: true` in love.md and body.md, register the property
    as multitext, verify with a workspace-built bob, and then handle removing `offclock:
    true` as the cutover decision says.'
proposed_by: bbugyi200.apollo.research.3c.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 10:03:51
status: done
bead_id: bob-cli-hc
---

- **PROMPT:** [prompts/202611/offclock_types.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/offclock_types.md)
- **BEAD:** [bob-cli-hc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hc/README.md)

# Plan: `offclock_types`, off-clock by area and task type

Research:
`research:202611/offclock_types_area_type_clock/offclock_types_area_type_clock.md` (read
it with `sase artifact read`). Bryan accepted all of its requirement adjustments (A1–A6)
and answered its open questions as follows: MEET is the only listed type, re-stamps use
the narrow scope, and hand-written untyped events stay on-clock until they are typed
MEET. The sibling link-budget work (`1a5c9b00`, `plan_budget/off_clock.rs`) already
resolves links through `effective_clock`, so it picks up this feature with no further
change.

## Contract (every phase implements or documents this)

A task's clock is decided by the first match:

1. **`#on`** on the task line: on-clock (unchanged, including the conflict rule).
2. **`#off`** or a deprecated alias: off-clock (unchanged). It beats `offclock: false`
   and beats an `offclock_types` list that does not include the task's type.
3. **The nearest area or project note on the task's root walk that declares a clock
   policy.** The walk is unchanged: it starts at a valid `[root::]` target, otherwise at
   the residence note, and a daily note with no `[root::]` gets no note default. For
   each note on `residence_chain`:
   - **Neither area nor project:** if it carries `offclock:` or a non-empty
     `offclock_types:`, lint `note_offclock_ignored` (naming the key) and keep walking.
   - **`offclock_types` present (valid or invalid):** this note decides, whatever its
     `offclock:` says. If `offclock:` is also present, raise the stats-only
     `note_offclock_shadowed` lint.
     - **Valid list:** off-clock if and only if the task's effective type key is listed.
       Any other type, or no type at all, is on-clock and the walk **stops** here.
       Source `note`, with `clock_note` set to this note.
     - **Invalid list:** on-clock, the walk stops, source `invalid_note`, and
       `note_offclock_invalid` names `offclock_types`.
   - **Otherwise `offclock: true|false|invalid`:** behaves exactly as it does today.
   - **Neither key, or `offclock_types` in an empty shape:** keep walking.
4. **No declaration:** on-clock.

**Effective type key (A1).** This is `normalize_component` of the first match: the
task's own `[type::]` / `(type::)` / legacy `[theme::]` field (closed `[x]`/`[-]` rows
included; a field in repair state falls through to inheritance); then the reading rule
(`plan.reading_type` for reading tasks); then the nearest `task_type:` / `theme:`
default on the task's **own** walk (same start rules as type suggestions;
`task_type: off` stops note inheritance); then `plan.default_type`. It is resolved only
when the walk reaches a valid list, at most once per resolution. It never consults the
`types:` catalog, so types that are not configured still match. Tag checks and
boolean-only walks cost what they cost today.

**Value rules (A4).** The same table applies in Rust and in JS:

| `offclock_types` value                                                                                                                                                                                                               | Meaning                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| absent, blank (`offclock_types:`), `null` / `~`, `[]`, or a list whose items are all blank or null (including Obsidian's lone `-`)                                                                                                   | No declaration: inherit, no lint. A same-note `offclock:` still applies.                                                                                        |
| Scalar `offclock_types: MEET`                                                                                                                                                                                                        | A one-item list.                                                                                                                                                |
| Flow list `[MEET, CALL]` or block list                                                                                                                                                                                               | Each item is canonicalized with `canonical_theme_name` and deduplicated by key. Blank and null items are skipped. Quotes and trailing `#` comments are handled. |
| Any non-blank item that is a composite (`A + B`), fails `canonical_theme_name`, is a reserved clock spelling (`OFF`, `ON`, `OFFCLOCK`, `OFF-CLOCK`, `UNTRACKED`), is a mapping or nested list, or is an unquoted YAML bool or number | The whole key is invalid.                                                                                                                                       |
| A mapping value                                                                                                                                                                                                                      | Invalid.                                                                                                                                                        |

Writers and docs show the block-list form. Reuse the existing reserved-clock-name check;
find it by grepping for `reserved for clock tags`.

**Lints and messages.**

- `note_offclock_invalid` and `note_offclock_ignored`: the stats messages and the
  capture/close notices name the key that caused them (`offclock` or `offclock_types`).
  The stats `HomeLintReport` gains an additive optional `key`.
- `note_offclock_shadowed` (new): stats health and warnings only, never a per-capture
  warning. Suggested message:
  `<path>: offclock ignored; offclock_types decides this note's clock`.

**Unchanged.** The counted-links session rule (mixed sessions are Worked, with
`pomodoro_clock_mixed`), frozen closed rows, settling of ended off-clock Event Tasks,
`clock_source` / `clock_note` JSON, and bob-mac-capture.

**Out of scope.** No `clock_type` JSON field or tooltip. No global "MEET is off-clock"
rule. Membership in `## Events` does not imply MEET. Retypes are not re-stamped beyond
the naming paths. No Cargo.toml version or CHANGELOG edits, and no `!` (the change is
additive).

## clock-type-filter

Everything here is in bob-cli.

- `src/native/projects/scan.rs`: add a strict frontmatter list reader beside
  `project_name_aliases`. That reader drops bad entries one at a time; do not reuse that
  leniency. The new reader returns absent / empty / items / invalid(raw) according to
  the value table. Leave `frontmatter_scalar` alone.
- `src/native/pomodoro_homes/facts.rs`: add
  `NoteOffClockTypes { Keys(..canonical display names unique by key..), Invalid(String) }`
  and `NoteFacts.offclock_types: Option<NoteOffClockTypes>`, where empty shapes map to
  `None`. Export it from `pomodoro_homes/mod.rs`.
- `src/native/pomodoro_themes/suggest.rs`: extract one catalog-free resolution core
  (field → reading → note default → `plan.default_type`) that returns the effective type
  key, and reads the field on closed rows too. Rebuild `suggest_for_task` and
  `resolve_without_task_field` on that core so the type pill and the clock cannot drift
  apart. Suggestion output must stay identical: the existing suggest, theme, and capture
  tests stay green unchanged, `suggest_for_task` still returns `closed_task` for closed
  rows, and `task_type: off` plus the global default keeps its current behavior.
- `src/native/pomodoro_homes/resolve.rs`: `HomeResolver` holds optional
  `TypeRules { reading: Option<String>, default: Option<String> }`.
  - Add an explicit setter for tests, and for commands that already hold a `PlanConfig`
    (injection there is optional).
  - Otherwise load the rules lazily, once, on the first list hit via
    `config::load_plan_config(&config::config_path())` with `unwrap_or_default()`, the
    same as `gcal::render::needs_meet_type`. Do not touch the ~44 `HomeResolver::new`
    call sites.
  - Unit tests that reach a list must inject rules and never read the developer's real
    config.
  - Add a `note_offclock_shadowed` health list with a dedupe set that `refresh()`
    clears, and record the key on lint entries.
- `src/native/pomodoro_homes/clock.rs`: `resolve_note_default` receives the line and
  residence so it can resolve the type lazily, and implements step 3 of the contract.
  `ClockResolution` may carry the deciding key for notices. `ClockSource` is unchanged.
- `src/native/stats/homes.rs` and `report.rs`: add shadowed warnings and health entries,
  and key-naming messages for invalid and ignored.
- `src/native/capture_pomodoro_close/linked_tasks.rs` and
  `src/native/capture/meet_event_run.rs`: the `note_offclock_invalid` notice names the
  key.
- Unit tests (`pomodoro_homes/tests.rs`, plus parser tests beside the reader) cover the
  whole contract matrix, every empty shape, and every invalid shape.
- **Shared vectors** (`tests/fixtures/pomodoro_clock/vectors.json`, owned by this
  phase). Later phases must not edit this file.
  - Add an optional `config` member: a JSON object holding the bob config document, for
    example `{"plan": {"default_type": "CHORE"}}`. When `config` is absent, no config
    file is written.
  - Both runners in `tests/cli/capture/pomodoro_clock.rs` (capture and dry-run) write it
    as the config file and point `BOB_CONFIG_FILE` at it.
  - Add cases for:
    - a love MEET Event Task rooted at a love child (off, `clock_note` love.md);
    - a love CHORE task (on);
    - MEET + CHORE (Worked, `pomodoro_clock_mixed`);
    - a child list narrowing a parent `offclock: true`;
    - a child `offclock: false` under a parent list;
    - both keys on one note (the list wins, with no capture warning);
    - a scalar value;
    - each empty shape (`[]`, blank, `null`, a lone `-`) inheriting a parent `true`;
    - an invalid item (on, `note_offclock_invalid`);
    - a list on a non-area note (ignored; the parent decides);
    - `#on` on a MEET task and `#off` on a CHORE task;
    - MEET inherited from a `task_type: MEET` note on the walk with no field;
    - a closed `[x]` linked task with no field;
    - CHORE coming only from `config.plan.default_type`;
    - a reading task with `[READ]`;
    - `task_type: off` plus a configured `default_type`.
  - Update `tests/fixtures/pomodoro_clock/README.md`: document the `config` member, and
    pin bob-plugins copies to this phase's commit.
- **Docs.** Sweep every bob-cli doc that states the clock rule
  (`grep -rn offclock docs`), skipping the generated `docs/develop/decisions.md` and
  `docs/reference/glossary.md` and the mounted `docs/reference/plugins/`:
  - `docs/reference/vault-format.md`: key paragraph, clock table, and resolution order,
    using the contract wording above;
  - `docs/capture.md`: "Which clock a session is on";
  - `docs/stats.md`: the new lint, the `key` field, and drift after narrowing a note;
  - `docs/off-clock-link-chips.md`: contract rows;
  - `docs/task-types.md`: effective type now feeds the clock;
  - also `docs/pomodoro-homes.md`, `docs/task-status-hooks.md`,
    `docs/event-link-times.md`, `docs/plan.md`, `docs/periods.md`, and
    `docs/reference/capture-syntax.md` wherever a sentence becomes incomplete.
- Gate: `just check` and `just docs-check`.

## clock-surfaces

Everything here is in bob-cli.

- **gcal (A6).** `src/native/gcal/clock.rs::probe_line` adds ` [type:: MEET]` to both
  probes, because sync always leaves an Event Task with effective type MEET.
  Effective-clock previews and the deprecated-tag drop advice ("already off-clock via
  love") then match sync. Add CLI tests showing that `bob gcal list` and `bob gcal sync`
  agree for a love-rooted event under `offclock_types: [MEET]`, both when the root
  inherits MEET and when it does not.
- **Naming re-stamp (A6).** `bob capture-pomodoro-name`
  (`src/native/capture_pomodoro_name.rs`) retypes linked tasks before writing the day
  note. When the named entry is open with a timed span, recompute its `[t::]` /
  `[off::]` key from its counted lineup after those retypes, in the same write.
  `--dry-run` must report the same post-image.
  - Reuse the existing lineup clock machinery
    (`capture::meet_event_run::restamp_session_clock` / `apply_meet_rule`, or
    `capture_pomodoro_start::resolve_session_clock_links`) instead of writing a second
    clock rule.
- **Capture `#type` relocation.** When a retype's `type_relocation` renames or moves a
  running entry (`src/native/capture/task_theme.rs` →
  `project_deps::restamp_changed_timed_entries`), make sure that entry's key is
  recomputed even when its child lines did not change. Write a failing test first to
  check the current behavior.
- Every other change to a linked task's effective type (hand edits, `#type` that leaves
  the link in place, note-default or `[root::]` edits) leaves the running key until
  close re-stamps it, the same way tag and root edits work today. Document that.
- Tests: naming a running love entry `MEET` flips `[t::]` → `[off::]`, renaming it back
  flips it back, and dry-run matches the real write.
- Docs: `docs/gcal.md`; `docs/capture.md` (the "What counts" writer table, the naming
  section near "Naming an entry that holds Task Links", and the type relocation
  section). Gate: `just check`, `just docs-check`.

## type-rename-coverage

Everything here is in bob-cli: `src/native/pomodoro_themes/coverage.rs`, plus
`rename.rs` and `split.rs` as needed.

- **Rename (A5).** Rename rewrites `offclock_types` items that name the old type on area
  and project notes (the same note filter as `task_type:` defaults).
  - It handles flow, block, and scalar form, and keeps quote style and every unrelated
    byte.
  - If the new name is already listed, the renamed item is removed instead,
    deduplicating by key: `[MEET, CALL]` with CALL→MEET becomes `[MEET]`, and a block
    list loses that item's line.
  - Matching items are rewritten even when another item makes the key invalid.
    Non-matching items are never touched.
  - These edits join the existing previewed transaction, its human and JSON preview, and
    its single commit. The current line-span `LineEdit` model may need a line-removal
    variant.
- **Split.** Split leaves lists untouched. `LeftoverThemeUses` and the
  `split_leaves_task_themes` warning also count notes whose `offclock_types` still names
  the old type.
- Tests: unit tests for each form, quoting, dedupe, and mixed-validity lists; CLI tests
  for rename preview plus commit, and for the split warning.
- Docs: `docs/task-types.md` rename and split coverage. Gate: `just check`,
  `just docs-check`.

## plugin-mirror

This phase works in the linked `bob-plugins` repo; open it with `/sase_repo` and read
its `AGENTS.md`. Edit `src/` fragments only, keep each fragment at or under 1000 lines,
and never hand-edit the generated `main.js`.

- `plugins/bob-ledger-tools/src/036-pomodoro-clock.js::taskEffectiveClockResolution`
  implements step 3 of the contract. It reads `frontmatter.offclock_types`, which
  Obsidian's metadata cache gives as an array, a string, or null, with `null` items for
  blank entries, and applies the value table.
  - Canonicalize with `pomodoroThemeCanonicalName` and compare with
    `pomodoroThemeNormalizeKey`.
  - Invalid: `{offClock: false, source: "invalid_note", note, ...}`, naming the key for
    notices.
  - No shadowed lint in JS.
- Add a type-key helper that also handles closed tasks, built on
  `pomodoroThemeTaskFieldState` and `pomodoroThemeResolveWithoutTaskField` and fed
  `loadPlanCaps().caps` (`defaultType`, reading type) the same way
  `taskTypeEffectiveTypeFor` already does. Parse the task line for its type. Never use
  Dataview's `task.type`, which falls back to the daily note's own `type:`.
  `pomodoroThemeSuggestWithDiagnostics` itself keeps returning `closed_task`.
- `pomodoroClockResolutionNotices`: the invalid notice names the key.
- Off-clock link chips, home chips, the gcal event clock, the plan-budget off-clock
  count, and task-status-cycler all reach this one function through
  `taskEffectiveOffClock`. task-status-cycler's tags-only fallback is unchanged.
- Copy `tests/fixtures/pomodoro_clock/vectors.json` from the clock-type-filter commit
  byte for byte into both `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_clock/`
  and `plugins/task-status-cycler/scripts/fixtures/pomodoro_clock/`, and update their
  READMEs. Make every harness that reads the vectors honor `config` by feeding the plan
  block into the caps:
  - `scripts/test-ledger-tools-pomodoro-clock.cjs`
  - `scripts/test-task-status-cycler-pomodoro-clock.cjs`
  - `scripts/test-ledger-tools-plan-budget-off-clock.cjs`
  - `scripts/test-ledger-tools-off-clock-link-chips.cjs`
- The Obsidian naming path (block-id-prompt `078-pomodoro-theme-write.js`) has no clock
  writer today. Leave it alone, and record a `PROPOSED FOLLOW-UP:` on this phase's bead
  for re-stamping on Obsidian naming.
- Docs: `docs/bob-ledger-tools.md`, `docs/task-status-cycler.md`, and any other plugin
  doc that states the clock rule.
- Gate: `npm run build`, `npm test` (which includes `build:check`), and
  `npm run docs:check`. Then run `bob plugins sync` as AGENTS.md requires.

## memory-records

Use `/sase_memory_write` first and edit only what the memory decisions accept.

> [!decision] memory_decision_record
>
> - Add a new strand to the `decisions` memory web, for example "Off-Clock Note Defaults
>   Filter By Task Type", with these sections:
>   - **Applies to:** bob-cli writers, bob-plugins, gcal, the vault.
>   - **Claim:** the contract's step 3, plus "the list wins on a dual-key note".
>   - **Why:** the research's alternatives table: global rule, list on meet.md,
>     bool-or-list `offclock:`, `offclock: events`, `onclock_types`, and gcal writing
>     `#off`.
>   - **Cost:** type becomes an accounting input; non-MEET off-clock time in love/body
>     becomes Worked; mixed sessions are more common.
>   - **Reopens when:** per-type exceptions under a list are needed, or Events-section
>     membership should imply MEET.
>   - **Evidence:** this plan, the research ref, and the phase commits.
> - Mark `decisions:off-clock-follows-tag-then-note-default` with
>   `metadata.status: superseded-in-part`, `superseded_by` pointing at the new strand,
>   and a one-line `[[...]]` back-link in its body saying what was retired. Do not
>   reword the accepted body.

> [!decision] memory_glossary
>
> - `off-clock-pomodoro`: an inherited `offclock: true` or a matching `offclock_types`
>   list.
> - `area-note` and `project-note`: `offclock_types` alongside `offclock:`, and the list
>   wins.
> - `event-task`: the effective type (MEET for synced rows) can decide the clock through
>   `offclock_types`.

Then run `sase memory init` and `just docs-gen`, which regenerates
`docs/develop/decisions.md` and `docs/reference/glossary.md`, and finally
`just docs-check`.

## vault-migration

This phase edits Bryan's vault at `~/bob`, outside the repo. Change only the files named
here.

1. Build bob from this workspace and use **that** binary for every check. Do not
   `just install`; Bryan upgrades hosts with `just install-all`.
2. **Baseline.** Record the `bob stats` `clock_drift` count (the research saw 1) and the
   current `note_offclock_*` lints. Run `bob gcal list` for today and note how love and
   body events read.
3. **Dual-key.** In `~/bob/love.md` and `~/bob/body.md`, add this directly under
   `offclock: true` and change nothing else:

   ```yaml
   offclock_types:
     - MEET
   ```

   Register `"offclock_types": "multitext"` in `~/bob/.obsidian/types.json`, keeping its
   formatting. Hosts that are not upgraded ignore the new key; upgraded ones narrow to
   MEET.

4. **Verify** with the workspace binary:
   - stats shows `note_offclock_shadowed` for both notes and no new
     `note_offclock_invalid`;
   - the drift count is unchanged;
   - love/body Event Tasks still read `off-clock via love/body` in `bob gcal list`;
   - a `bob capture --dry-run` start linking an existing untyped love task previews
     `[t::]`, and one linking a love MEET Event Task previews `[off::]`;
   - re-scan open love- and body-rooted Event Tasks and report any whose effective type
     is not MEET.
5. Review `git -C ~/bob diff`; only the three files should change. Let vault sync carry
   it, and confirm with `bob vault-sync status --json`.
6. **Cutover**, which deletes exactly the `offclock: true` line from love.md and
   body.md. Rollback is putting that line back; sessions recorded in the meantime keep
   their keys.

> [!decision] cutover = gate Use `/sase_gate` to stage the cutover as a confirmable
> command. The gate text tells Bryan to first run `just install-all` on athena, apollo,
> and the Mac; that pulls bob-cli, syncs bob-plugins, and restarts Obsidian. On
> confirmation the command:
>
> - records the drift count;
> - reports open love/body-rooted Event Tasks that lack effective type MEET;
> - deletes the two `offclock: true` lines;
> - re-runs the step-4 checks with the installed `bob`, expecting no
>   `note_offclock_shadowed` and an unchanged drift count.
>
> Do not wait for the gate.

> [!decision] cutover = followup Leave both keys in place. Record a
> `PROPOSED FOLLOW-UP:` on this phase's bead that holds the step-6 cutover and its
> checks, to run once every host has run `just install-all`.

> [!decision] cutover = now Delete the two `offclock: true` lines right after step 5,
> re-run the step-4 checks (expecting no shadowed lint), and say in the phase result
> that any host not yet upgraded counts love/body MEETs as Worked until it runs
> `just install-all`.

## Verification (epic level)

- bob-cli `just check` and `just docs-check` are green, and the shared vectors pass in
  Rust and in both plugin harnesses from byte-identical copies.
- On the live vault after the dual-key step: love/body MEET sessions are off-clock via
  their note, love/body CHORE sessions are Worked, job meetings are unchanged, and the
  drift count is unchanged.
