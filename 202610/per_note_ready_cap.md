---
tier: epic
title: 'Per-note Ready cap: crowded notes in the CLI, dash, and notes'
goal: 'Every area/project note has a soft cap on its Ready lane (plan.max_ready_per_note,
  default 5, per-note ready_cap override). Crowded notes are named, counted, and easy
  to act on: in a new `bob ready` CLI view, a dash CROWDED chip that opens crowded.md,
  and a live chip on each note''s `## Tasks` heading. All surfaces share one read-time
  contract, implemented in Rust and in bob-ledger-tools and pinned by shared vectors.
  The keymap-notice work is fully specified and filed to start after the freshness
  trial.

  '
phases:
- id: core
  title: Ready-lane-per-note contract, config, and Rust evaluator
  depends_on: []
  size: medium
  description: 'core: add plan.max_ready_per_note and the ready_cap frontmatter. Extend
    the area/project classifier to list forms. Build the Rust note_ready evaluator
    (counts, states, make-up, lints) on the freshness snapshot. Document the contract
    and vectors R1–R14 in docs/plan.md, and record the new decision memory.'
- id: cli
  title: bob ready command
  depends_on:
  - core
  size: medium
  description: 'cli: add the read-only top-level `bob ready [NOTE]`: a colored CROWDED/FULL/ROOM
    bar view, a per-note worklist, schema-1 JSON, --check (exit 3), --cap preview,
    and --all. Includes help, README, justfile smoke, and integration tests.'
- id: ledger-api
  title: bob-ledger-tools noteReady API
  depends_on:
  - core
  size: medium
  description: 'ledger-api: add api.noteReady v1 (snapshot, forNote, counted, inCrowdedNote,
    groupLabel), built in one memoized O(tasks) pass with live invalidation. Also
    a stat-cached loadPlanCaps, a shared getFileCache frontmatter reader (fixes bob-cli-3e),
    and the R1–R14 JS vectors. Deploys the plugin.'
- id: ledger-views
  title: CROWDED chip, bob-ready-notes block, and Tasks heading chip
  depends_on:
  - ledger-api
  size: medium
  description: 'ledger-views: add the live renderCrowdedChip, the bob-ready-notes
    ranked-bar code block, and the `## Tasks` heading chip (CM6 widget plus Reading
    view). Adds theme-safe CSS, the bob-cli-3d anchor-class fix, and one consolidated
    live-refresh fan-out. Deploys the plugin.'
- id: rollout
  title: Vault rollout, docs, and live verification
  depends_on:
  - cli
  - ledger-views
  size: medium
  description: 'rollout: add the dash CROWDED chip and the new crowded.md page. Set
    ready_cap: off on the three inboxes, add CROWDED to the morning ritual and `bob
    ready -a` to the weekly prune, add a sase.md triage task, and log the trial. Updates
    the plan and freshness docs, cross-checks the CLI against the plugin, and records
    verification evidence.'
proposed_by: bbugyi200.athena.0v5
create_time: 2026-10-01 17:55:46
status: wip
bead_id: bob-cli-3f
---

- **PROMPT:** [prompts/202610/per_note_ready_cap.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/per_note_ready_cap.md)
- **BEAD:** [bob-cli-3f](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3f/README.md)

# Per-note Ready cap: crowded notes in the CLI, dash, and notes

## Outcome and scope

During the morning GTD review, Bryan wants no area/project note to hold more than N
ready tasks, with N configurable and defaulting to 5. When a note goes over, the fix is
to split the work into a new project or to de-prioritize some of it. Bryan accepted
every recommendation in `research:202610/per_note_ready_cap/per_note_ready_cap.md`. Read
it with `sase artifact read` before starting any phase. This epic implements that
report's design, with the refinements called out under "Design decisions beyond the
report".

The phases deliver these surfaces:

- **CLI:** a new `bob ready`.
- **Dash:** a `CROWDED k ↗` chip, between READY and BLOCKED, that opens a new
  `crowded.md`.
- **Area/project notes:** a live `ready n/cap` chip on each note's `## Tasks` heading.
- **Shared contract:** one read-time contract with shared conformance vectors, in Rust
  and in bob-ledger-tools.
- **Vault rollout:** inbox exemptions and the new morning-ritual step.

Keymap and command feedback is the Ctrl+Shift+M picker pills and the capacity chips in
Bob command notices. It is fully specified below under "Deferred: gesture feedback after
the freshness trial", and the epic's land agent files it as a snoozed task bead. The
reason is timing:

- The report recommends shipping it only after the 2026-10-05 → 2026-10-18 freshness
  trial, and after one `sase.md` triage session. Bryan accepted that.
- SASE cannot defer an epic phase to a date.
- Landing the notice code in bob-plugins now would let any routine `bob plugins sync`
  deploy it early.

Out of scope:

- A `bob plan` CROWDED line, and a `note_ready` field in capture or capture-targets JSON
  for Bob Mac Capture.
- An IDLE-projects list.
- Changes to the hooks badge row or to the lane semantics.
- Storing counts in frontmatter.
- A separate area default, or an Obsidian setting for N.
- Hard refusal of moves, auto-deferral, auto-splitting, and parent rollups.

The report rejects all of these, or makes them optional later.

## Context verified during planning

bob-cli at `5d24c98`:

- `src/native/dataview/tasks/mod.rs`:
  - `READY_QUERY` (:94) is the Ready lane predicate.
  - `RichTask` (:104) carries `path`, `line`, `status_symbol`/`status_type`,
    `scheduled`, `is_recurring`, `is_blocked`, `tags`, and `block_id`. A `^prj`
    lifecycle row is `block_id == Some("prj")`.
- `src/native/freshness/scan.rs` `scan()` returns a `Snapshot`. It holds READY_QUERY
  rows (`ready`), open rows, all rows, freshness config, and file contents.
  `freshness/state.rs` `evaluate()` and `FreshState::bucket()` give `new`, `rotten`, or
  null. Recurring, daily-note, and Today rows are out of freshness scope (null bucket).
  `BOB_NOW` is honored through `env::current_datetime()`.
- `src/native/config/plan.rs` `PlanConfig` and `parse_plan_config`: caps are integers ≥
  1 and unknown keys are ignored. An invalid value makes `bob plan` exit 2. Every other
  caller falls back to defaults.
- `src/native/projects/scan.rs`:
  - `frontmatter_has_type` compares the trimmed scalar exactly against `[[project]]` and
    `[[area]]`. Quoted and bare forms work; block-list and flow-list `type` forms do
    not.
  - `ProjectStatus::parse` (model.rs:146) treats `done`, `canceled`, and `cancelled` as
    terminal.
  - `is_excluded_directory` skips `.git`, `.obsidian`, `_conflicts`, `_generated`,
    `_templates`, and `done`.
  - `read_note_refresh` (freshness/scan.rs:355) is the per-note frontmatter pattern to
    copy.
- Note-level lints currently repeat once per task row (`collect_warnings`). The new note
  lints must be emitted once per note.
- CLI wiring:
  - `src/runner.rs` `SUBCOMMANDS` must stay alphabetical; `ready` goes right after
    `randomize`. A test enforces the order.
  - `src/native.rs` holds the `mod` list and `NativeCommand`.
  - Per-command clap builder pattern: `plan_budget/cli.rs` and `freshness/cli.rs`, with
    the shared `bob_dir_arg`, `format_arg`, and `help_arg` helpers.
  - `src/native/style.rs` `Styler`: color only on a TTY with `NO_COLOR` unset, plus the
    `display_width`, `pad_right`, `terminal_width`, and `truncate` helpers.
  - No command uses exit code 3 yet.
- Tests:
  - Integration tests live in `tests/cli/` (`main.rs` module list, `support.rs`
    helpers). `freshness.rs` and `plan.rs` show the fixture-vault pattern.
  - The help-surface lists are in `tests/cli/help.rs` and `help_options.rs`.
  - The `justfile` targets are `fmt`, `lint`, `test`, and `install-smoke`.
- Docs:
  - `docs/plan.md`: Lints (:85), Lanes (:138), READY backlog (:188), Config (:225),
    `bob plan` (:254), Surfaces (:346, dash chip order row), and conformance (:359,
    :488).
  - `docs/freshness.md`: §6 ritual (:245), §8 surfaces, §13 trial.

bob-plugins at `4744607`:

- bob-ledger-tools:
  - Version 1.14.1; top-level api `version: 3`, built as a frozen object in `onload()`
    (main.js ~5958). Namespaces carry their own version (`freshness` is v3).
  - `readyTaskVisible` and `planTaskIsBlocked` are the JS lane predicate.
  - `freshnessRowFromTask` detects recurrence. `planTaskBlockId` reads `blockLink`.
  - The freshness memo (`freshnessEnsureMemo`, `freshnessBuildMemo`) is keyed on the
    tasks array, tasksGen, date, frontGen, config, and the Today stamp. It exposes
    `evaluatedByIndex`.
- `loadPlanCaps()` (~10460):
  - It re-reads and re-parses `config.yml` on every call (13+ callers), and returns the
    defaults on mobile.
  - `coercePlanCaps` replaces the whole block with defaults when any value is invalid.
- Live widgets:
  - The lifecycle widget-Set pattern lives in `renderReadyBadge`,
    `renderDashboardLaneBadge`, and `renderReviewChip`.
  - The `.text` setter gotcha is handled by `setReadyAnchorContent`.
  - A new widget family has to be added at four refresh fan-out sites: the
    `cache-update` handler, `refreshFreshnessForChangedFile`,
    `refreshFreshnessForRollover`, and the review-changed branch in
    `freshnessEnsureMemo`.
- Live Preview and Reading view:
  - The CM6 model is freshness marks: `createFreshnessMarkExtension`,
    `buildFreshnessMarkDecorations`, and the `freshnessMarksRefresh` effect.
  - No plugin decorates headings yet. Test stubs lack `Decoration.widget`.
  - The only code-block processor is `bob-plan` (`renderPlanBlock`, with a
    `MarkdownRenderChild` for cleanup).
- Tests are `scripts/test-ledger-tools-*.cjs`. The `npm test` file list in
  `package.json` is explicit. `npm run validate` checks the manifests.
- Two open small bugs sit on code this epic touches:
  - **`bob-cli-3d`:** `setReadyAnchorContent` overwrites the PENDING/NEXT lane class
    with `bob-plan-ready`.
  - **`bob-cli-3e`:** `getCache({path})` / `getCache(file)` return null, so the
    `task_refresh` override never applies in Obsidian.
  - Both were found by this feature's research, as gotchas for it.
- bob-navigation-hotkeys 1.49.0 and task-status-cycler 1.19.0 are only touched by the
  deferred gesture work. Their seams are listed in that section.

Vault (`gh:bobs-org/bob`):

- `dash.md` chips:
  - The chip lists are `chipsBeforeReady`, the lane badges, the READY badge, then
    `chipsAfterReady` (BLOCKED, ROTTEN, TODAY), defined around lines 217–233.
  - The generic `renderChip` defaults an external chip's destination to "Blocked Tasks".
    Its aria label says "tasks".
  - `counts` starts every key at `"–"`.
  - The chip CSS is inline in dash.
- `rotten.md` is the review-page template: `parent: "[[gtd]]"`, aliases, a live summary,
  and Tasks queries with guarded `filter by function` calls. Its trial tally is at
  :75–83.
- Inboxes:
  - `inbox.md`, `gkeep_inbox.md`, and `mac_inbox.md` are all `type: "[[area]]"`.
  - Only `gkeep_inbox.md` has a `## Tasks` heading.
- Type values and headings:
  - Every type value in the vault is in quoted form. There are 86 projects (counting the
    template and two `_conflicts` copies) and 13 areas.
  - 85 projects and 8 areas have `## Tasks`.
- `gtd_daily.md`:
  - Line 18 is the daily morning-review task (NEW → ROTTEN → PENDING → NEXT).
  - Line 19 is the weekly prune task.
- `bob_gtd.md:22` is `^prj-task-count-warn`, the vault task this epic implements. It has
  no children.
- New notes need a `parent` frontmatter link; this comes from the athena `obsidian.md`
  reference memory. The vault syncs through its Git remote and `bob vault-sync`;
  Obsidian Sync is retired.

## Design decisions beyond the report

1. **Gesture feedback is deferred mechanically.** See "Outcome and scope". The ledger
   API built now is read-only. The notice API (`countedRows`, the change rule, `watch`)
   ships with its only consumer, the deferred task, so no speculative API lands.
2. **Fold in `bob-cli-3d` and `bob-cli-3e`.** The new chip reuses the shared anchor
   helper, and `ready_cap` reads go through the same changed-file frontmatter path.
   Fixing both once avoids two half-right copies.
   - If either bead is already closed or in progress when its phase starts, integrate
     with that work instead of redoing it.
   - The land agent closes them with the phase's evidence.
3. **Noise rule for the deferred notices.** A standalone notice appears only for a
   worsening (warn) or for crossing back to the cap (`✓ back to 5/5`). Smaller `▼`
   progress steps only piggyback on a notice the command already shows. This matters
   because the cycler shows no success notices today, and completions inside `sase.md`
   would otherwise toast every time.
4. **One live-refresh fan-out.** ledger-views replaces the four hand-maintained fan-out
   sites with one `scheduleLiveWidgetRefresh()`. New widget families then cannot miss a
   site.
5. **crowded.md groups are ranked and linked.** Each group heading reads
   `[[sase]] · 61/5 · +56`, sorted by excess with a hidden `%%NNN%%` prefix.
6. **`ready_cap` and the global key share one range, 1–999.** `off` (case-insensitive)
   and YAML `false` mean exempt. YAML 1.1 parsers read a bare `off` as `false`.
7. **`bob ready NOTE` is forgiving.** A path, a stem, or a case-insensitive stem all
   resolve. An unknown note exits 2 with up to three closest-name suggestions.

## Shared contract (authoritative)

Phase core writes this contract into `docs/plan.md` as "## Ready cap per note", between
"READY backlog" and "Config". Rust and bob-ledger-tools implement it identically.

```text
counted(t)  = lane(t) ∧ ¬recurring(t) ∧ block_id(t) ≠ "prj"
lane(t)     = the existing READY_QUERY / readyTaskVisible ∧ ¬planTaskIsBlocked predicate:
              TODO-type status, not done, not dependency-blocked, no #hide tag, not under
              _templates/ or _conflicts/, no scheduled date or scheduled ≤ today.
              Freshness bucket and Today are NOT filters (whole lane).
note(t)     = the vault file that holds t (residence; never parent, heading, embed, backlink)
eligible(n) = type(n) ∈ {[[area]], [[project]]} (quoted, single-quoted, bare, flow-list,
              block-list forms) ∧ (area ∨ status ∉ {done, canceled, cancelled})
              ∧ path not under .git/ .obsidian/ _templates/ _conflicts/ _generated/ done/
cap(n)      = ready_cap integer 1–999 → (cap, source "note")
              | ready_cap off / false → exempt
              | ready_cap present but invalid → lint note_ready_cap_invalid, then default
              | absent → plan.max_ready_per_note (source "config") else 5 (source "default")
count(n)    = |{ t : note(t) = n ∧ counted(t) }|   (each row counts; nested child tasks count)
make_up(n)  = { new, rotten, ready = count − new − rotten } from freshness buckets
              (rotten includes resurfaced; null when freshness is unavailable)
recurring(n)= lane rows in n excluded only because they recur (shown as ↻ k)
state(n)    = exempt | crowded (count > cap) | full (count = cap) | room (0 < count < cap)
              | empty (count = 0); surfaces add "unavailable" when no snapshot exists
totals      = notes (eligible, not exempt) with areas/projects split, crowded, full, room,
              empty, exempt, counted (Σ count over capped notes),
              excess (Σ max(0, count − cap)), recurring
lint note_ready_in_terminal_project: a done/canceled project still holding counted rows
              (not capped; listed under "not capped")
```

The note-entry field names are shared by CLI JSON and `api.noteReady`: `path`, `name`
(stem), `kind` (`area`|`project`), `status`, `parent`, `count`, `cap`, `cap_source`
(`note`|`config`|`default`|`preview`), `state`, `over_by`, `make_up`
(`{ready,new,rotten}` or null), and `recurring`. Lints are `{code, path, message}`,
emitted once per note, never once per row.

### Vocabulary and visual language (every surface)

- **Words:**
  - A note is **crowded**, **full**, or has **room**. An exempt note has **no cap**.
  - The verbs are always **split · sequence · defer · drop**.
  - The UI never suggests "promote to Next".
- **States:**

  | State       | CLI                           | Obsidian                            |
  | ----------- | ----------------------------- | ----------------------------------- |
  | room        | dim cells                     | muted chip or pill, `ready 3/5`     |
  | full        | cells up to `│`, `full` label | muted, `· full` text. **No amber.** |
  | crowded     | red overflow cells, `+k`      | `bob-plan-over` red, `+k` pill      |
  | exempt      | dim, `no cap`                 | muted, `ready 65 · no cap`          |
  | unavailable | n/a (the command errors)      | `–`, never `0`                      |
  | fixed       | `CROWDED 0 ✓` green           | calm/muted `CROWDED 0 ✓`            |

- **Accessibility:**
  - Color always comes with text.
  - Numbers use tabular numerals with a stable width.
  - Obsidian styling uses theme variables only (no hex), and motion respects
    `prefers-reduced-motion`.
  - Chips and rows carry aria labels and tooltips that spell out the count, for example
    `11 ready-lane tasks = 11 ready + 0 new + 0 rotten · cap 5 (Bob config) · 6 over · split, sequence, defer, or drop`.

### Failure behavior

- **CLI:**
  - An invalid global `plan.max_ready_per_note` exits 2, with a JSON error envelope
    under `-f json`.
  - An invalid per-note `ready_cap` produces a lint and falls back to the default.
  - A vault I/O failure exits 1. A non-Dataview task format exits 2 (inherited from the
    freshness scan).
  - Crowded notes never fail the report. Only `--check` maps them to exit 3.
- **Plugin:**
  - A missing Tasks plugin or a non-`Warm` Tasks cache gives `available: false`, and
    every surface shows `–`.
  - Unavailable freshness gives `make_up: null`; counts are still shown.
  - A missing or old `noteReady` gives the dash fallback chip `CROWDED –` and empty
    crowded.md queries with a `–` summary. The heading chips are simply absent.
  - API members are synchronous and never throw. Consumers still wrap calls in try/catch
    plus optional chaining.
- **Mobile:** `config.yml` is unreadable there, so the default cap applies and the
  tooltip says so. Per-note `ready_cap` works everywhere.
- **Config validity:** An invalid global key follows the existing whole-block fallback
  in both languages: Rust callers other than `bob plan`/`bob ready` use defaults, and JS
  `coercePlanCaps` marks the block invalid. Tooltips then say
  `plan config invalid, using defaults`.

### Conformance vectors (copied verbatim into Rust and JS tests)

| #   | Fixture                                                                                                                                                                                | Expected                                                                                                     |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| R1  | 6 `[ ]` tasks, default cap                                                                                                                                                             | `count 6`, crowded, `over_by 1`, `excess 1`                                                                  |
| R2  | 5 `[ ]` tasks                                                                                                                                                                          | full, no lint                                                                                                |
| R3  | `#hide`, `#Hide`, `#hide/x`, future `scheduled`, dependency-blocked, `[?]`, `[*]`, `[/]`, `[x]`, `[-]`                                                                                 | none count                                                                                                   |
| R4  | 3 tasks: one unstamped (NEW), one stamped 8 days ago (ROTTEN), one fresh                                                                                                               | `count 3`, make-up `1 ready + 1 new + 1 rotten`                                                              |
| R5  | `[repeat:: every day]` and `🔁` tasks                                                                                                                                                  | not counted; `recurring 2`                                                                                   |
| R6  | a visible `^prj` task                                                                                                                                                                  | not counted                                                                                                  |
| R7  | tasks in a daily note, an untyped note, `_templates/`, `done/`, `dash.md`                                                                                                              | absent from the per-note list                                                                                |
| R8  | `ready_cap: 8` / `off` / `OFF` / `false` / `0` / `lots` / `1000`                                                                                                                       | cap 8 (source `note`) / exempt / exempt / exempt / lint then default / lint then default / lint then default |
| R9  | a parent `[ ]` with 2 child `[ ]` tasks                                                                                                                                                | `count 3`                                                                                                    |
| R10 | `status: done` (and `cancelled`) project with 2 `[ ]` tasks                                                                                                                            | not capped; lint `note_ready_in_terminal_project`                                                            |
| R11 | `type: "[[project]]"`, `type: '[[area]]'`, bare `type: [[project]]` (Obsidian parses it as nested array `[["project"]]`), flow list `["[[area]]"]`, block list, nested folder `x/y.md` | all eligible; Rust and JS agree                                                                              |
| R12 | stamping `[fresh::]` on a NEW task (Alt+Shift+F) in a note at 5                                                                                                                        | count stays 5 (full); make-up moves 1 from new to ready                                                      |
| R13 | a Today-linked `[ ]` task                                                                                                                                                              | counted (whole lane)                                                                                         |
| R14 | `plan.max_ready_per_note: 3`; one note with `ready_cap: 8`                                                                                                                             | other notes cap 3 (source `config`); that note cap 8 (source `note`)                                         |

## Phase core

Repo: bob-cli. Read the research artifact and `cli_rules.md` first.

1. **Config.**
   - Add `max_ready_per_note: u32`, default 5, to `PlanConfig`, with a getter.
   - Parse it in `parse_plan_config` with the same `cap` validation plus an upper bound
     of 999. Use the message pattern
     `plan.max_ready_per_note in <path> must be an integer from 1 to 999; got …`.
   - Unit-test valid, invalid, and absent values, and check that a mistyped block still
     leaves other loaders working.
   - Document the key in the `docs/plan.md` Config YAML block, with the comment
     `# soft cap per area/project note; frontmatter ready_cap overrides`.
2. **Classifier.** Extend `frontmatter_has_type` (which backs `frontmatter_is_project`
   and `frontmatter_is_area`) so the flow-list (`type: ["[[project]]"]`) and block-list
   (`type:` followed by `  - "[[project]]"`) forms also match.
   - This is an intentional improvement for every caller of those helpers. Keep the
     other independent classifiers (`task_status_hooks/sync.rs note_kind`,
     `capture_targets.rs`) unchanged.
   - Add unit tests for every R11 form.
   - Expose a `pub(crate)` helper that walks the vault for typed notes, reusing
     `is_excluded_directory`. It returns each note's path, stem, kind, `ProjectStatus`,
     parent stem, and raw `ready_cap`.
3. **Evaluator.** Create `src/native/note_ready/` with:
   - `mod.rs`: pure types (`NoteReady`, `NoteState`, `CapSource`, `Totals`, `Report`,
     lints) and a pure `evaluate(notes, rows, buckets, default_cap, today)`. It must
     take no I/O so it is unit-testable.
   - `scan.rs`: one `freshness::scan::scan` snapshot plus the typed-note walk. Expose a
     small `pub(crate)` freshness helper that returns each ready row's bucket and its
     confirmation date (or evaluated result), so `RowCtx` logic is not duplicated.
   - `ready_cap` parsing per the contract: digits in 1–999; `off`/`false`
     case-insensitive; empty or missing means absent.
   - Constants `LINT_NOTE_READY_CAP_INVALID` and `LINT_NOTE_READY_IN_TERMINAL_PROJECT`,
     with messages such as `ready_cap "lots" in sase.md is not 1–999 or off; using 5`.
   - Deterministic ordering: state rank (crowded, full, room, empty, exempt), then
     `over_by` descending, `count` descending, and name.
4. **Tests.** Encode R1–R14 as unit tests: pure evaluator plus temp-vault scan tests
   that use the existing support helpers, `BOB_NOW`, and Dataview task settings with `?`
   and `*` set as ON_HOLD. Keep `bob plan` behavior and its tests unchanged.
5. **Docs.** Write the "## Ready cap per note" section from the shared contract:
   - the definition, with the reason the lane (not gated READY) is counted, citing the
     rot-cliff evidence in one sentence;
   - the states and vocabulary;
   - `ready_cap`;
   - the failure behavior;
   - the mobile caveat;
   - the vectors.

   Also add the two lint codes to the "## Lints" table.

6. **Decision memory.** Bryan's acceptance of the research covers this, because the
   report's phase 1 names the record. Plan approval makes it explicit. Use
   `/sase_memory_write` before writing, and read existing records only through
   `/sase_memory_read`.
   - Add `sase/memory/decisions/note-ready-cap-counts-the-lane.md`. Its claim: the
     per-note soft cap (`plan.max_ready_per_note`, default 5; `ready_cap: N|off`) counts
     each area/project note's whole Ready lane by residence, whatever its freshness.
     Recurring and `^prj` rows are excluded. Crowded means strictly over; full is
     neutral. Nothing is refused, written, or stored. The dash chip order becomes NEW,
     PENDING, NEXT, READY, CROWDED, BLOCKED, ROTTEN, TODAY.
   - Rejected alternatives: a gated per-note count (rot cliff and review inversion),
     OPEN/SHOWN, hard refusal, auto-defer or auto-split, stored counts, the hooks badge
     row, parent rollups, an area default, an amber at-limit color, and folding into the
     READY chip.
   - Cost: per-note counts don't sum to READY; two implementations; the mobile default
     cap.
   - Reopens when: the post-trial keep rule fails after tuning N and the notice scope.
   - Cite the research ref and this plan. Invent no other reasons.
   - Mark `decisions:ready-is-freshness-gated` with
     `metadata.status: superseded-in-part`, `superseded_by` pointing at the new record,
     and a one-line back-link saying only its chip list is superseded. Do not edit its
     accepted body otherwise.
   - Run `sase memory init`. Never hand-edit generated shims.

Validation: `cargo fmt --check`, focused `cargo test` for `note_ready`, `config::plan`,
and `projects`, then the full `cargo test` and
`cargo clippy --all-targets --all-features`. Use `/sase_monitor` for anything long.
Report any known baseline failures (e.g. `bob-cli-v` clippy warnings) accurately.

## Phase cli

Repo: bob-cli. Follow `cli_rules.md`: alphabetical options, a short alias for every long
option, excellent help, and color on a TTY.

1. **Wiring.**
   - Add `NativeCommand::Ready` and `note_ready::cli::run`.
   - Add a `SUBCOMMANDS` entry right after `randomize` with the summary
     `Show ready tasks per area/project note against the per-note cap`, plus a top-level
     `AFTER_HELP` example.
   - Update `tests/cli/help.rs`, `help_options.rs`, and the `justfile` `install-smoke`
     list.
2. **Usage.**

   ```text
   Usage: bob ready [OPTIONS] [NOTE]

   Show each area/project note's Ready lane against the per-note cap. The Ready lane is
   visible, pullable [ ] tasks whatever their freshness (dash READY also hides NEW and
   ROTTEN). A note is crowded above its cap, full at it, and has room below it.

   Arguments:
     [NOTE]  List one note's lane tasks (vault path, or note name)

   Options:
     -a, --all              Also list room notes in full, empty, exempt, and done projects
     -b, --bob-dir <DIR>    Bob vault root; defaults to BOB_DIR or ~/bob
     -n, --cap <N>          Preview a different default cap (1–999) for this run only
     -c, --check            Exit 3 when any note is crowded (for scripts and tmux)
     -f, --format <FORMAT>  human or json [default: human]
     -h, --help             Print help
   ```

   After-help covers:
   - examples: `bob ready`, `bob ready sase_remote`, `bob ready -a`, `bob ready -n 7`,
     `bob ready -c -f json`;
   - environment: `BOB_DIR`, `BOB_NOW`, `BOB_CONFIG_FILE`, `NO_COLOR`;
   - exit codes 0/1/2/3;
   - the `ready_cap:` frontmatter override.

   `--cap` replaces only the configured default. Note overrides and exemptions still
   apply, and the source shows as `preview`.

3. **Overview (human).** Build this in a separate `render.rs`, unit-testable with color
   forced on and off. Target look, using today's live numbers:

   ```text
   bob ready · Thu 2026-10-01 · cap 5 per note

     CROWDED 4 · 66 over · 3 full · 51 notes (10 areas · 41 projects)

     CROWDED
       sase            61/5  ■■■■■│■■■■■■■■■■■■■■■■■■■■■■■■…  +56   project · dev · 1 new
       sase_remote     11/5  ■■■■■│■■■■■■                      +6   project · sase
       sase_pager       7/5  ■■■■■│■■                          +2   project · sase
     FULL
       bob              5/5  ■■■■■│                                 project · gtd
       cash             5/5  ■■■■■│                                 area · ↻ 1
     ROOM
       dev 4 · sase_memory 4 · sase_art 3 · love 2 · … 12 more
     27 empty · not capped: gkeep_inbox 65 (ready_cap: off) · ↻ 11 recurring   (-a for all)

     Make room → bob ready sase_pager
     split Ctrl+Shift+N · sequence / defer / drop Ctrl+Shift+P
   ```

   - **Bar glyphs:**
     - One `■` per task, at most 30 cells; longer bars end in `…`.
     - A dim `│` sits right after the cap-th cell. It is omitted when the cap is at or
       past the bar width, and the `n/cap` column then carries the information.
     - Cells past the cap are red. Cells within it are cyan. FULL rows stop at the
       marker. The `+k` excess is red.
     - Meta is dim: kind · parent · non-zero make-up (`1 new`, `3 rotten`) · `↻ k` ·
       `cap 8 (note)` when the source isn't the default.
   - **Columns:**
     - Pad to the longest displayed name (at most 24 chars, truncated with `…`); show
       the vault path when two stems collide.
     - Right-align `n/cap`.
     - Fit `terminal_width()`: drop the meta column first, then shorten the bar to at
       least 10 cells.
   - **Header and footer:**
     - `CROWDED k` is bold red when k > 0. With none crowded the line reads
       `CROWDED 0 ✓ · every note has room` in green, and the CROWDED group is omitted.
     - `Make room → bob ready <note>` names the note with the smallest positive excess,
       the quickest win (ties broken by name). Omit it when nothing is crowded.
     - FULL uses no warning color.
   - **`-a`:** expands ROOM into bar rows, lists EMPTY names, and lists "not capped"
     with the exempt notes (dimmed, count and `ready_cap: off`) and done/canceled
     projects that still hold lane rows (each with its lint). It also prints a LINTS
     block in the `freshness` style.
   - **Without a TTY or with `NO_COLOR`:** no ANSI. Identical glyphs and text, so state
     is never color-only.

4. **Worklist (`bob ready NOTE`).** Resolve NOTE in this order: vault path (with or
   without `.md`), then exact stem, then case-insensitive stem.
   - Ambiguous → exit 2 listing candidates. Unknown or not an area/project → exit 2 with
     up to three closest names.
   - Print rows in file order:

     ```text
     bob ready · sase_remote · 11/5 · CROWDED +6 · project · parent sase · 11 ready + 0 new + 0 rotten

         1  sase_remote.md:21  Add full support for machines across the TUI and CLI!   fresh 1d  ^machines
         …
        11  sase_remote.md:38  Share prompt history and PIW / LSP completions…          new

       also here: 1 next · 1 pending · 2 blocked · ↻ 0 recurring
       Make room for 6: split Ctrl+Shift+N · sequence / defer / drop Ctrl+Shift+P
     ```

   - The `path:line` is jumpable. Text is truncated to the width. The freshness label is
     `new`, `rotten Nd`, or `fresh Nd`, and is blank when out of scope.
   - Room notes end with `room for k more`. Full notes end with `full — at cap`. Exempt
     notes say `no cap (ready_cap: off)`.

5. **JSON (`schema_version: 1`).**
   - Overview:
     `{ok, schema_version, date, definition: "ready_lane", cap: {default, source}, totals, notes: [...all eligible and exempt notes, regardless of -a], warnings: [{code, path, message}]}`.
   - Worklist: the same envelope with `note` instead of `notes`, plus
     `tasks: [{path, line, text, block_id, bucket, fresh_on}]` and
     `also: {next, pending, blocked, recurring}`.
   - Error: `{ok: false, schema_version, error: {code, message}}`.
   - `--check` with `-f json` still prints JSON and exits 3 when crowded.
6. **Integration tests** in `tests/cli/ready.rs`:
   - JSON totals and order on a fixture vault covering crowded, full, room, empty,
     exempt, terminal, recurring, `^prj`, and nested rows;
   - human sections, order, and no ANSI;
   - all-clear output;
   - `-a`;
   - the `--cap` preview;
   - `--check` exit 0 and 3;
   - worklist resolution, ambiguity, and suggestions;
   - invalid config exits 2 (JSON envelope);
   - invalid `ready_cap` lint emitted once per note;
   - `BOB_NOW` date handling;
   - help ordering, plus the top-level help listing `ready`.

   Measure wall time against `bob freshness list` on the live vault (read-only) and
   record it. The target is comparable time, with no per-note rescans.

7. **Docs.**
   - Add a "## `bob ready`" section to `docs/plan.md` after "## `bob plan`" covering
     usage, human/JSON/worklist contracts, and exit codes.
   - Add a README Commands-table row and a short section with a sample output.
   - Add `bob ready` to the docs/plan.md Surfaces table.

Validation: same as core. Also run `just install-smoke` if cheap, or the equivalent
`cargo run -- ready --help`.

## Phase ledger-api

Repo: bob-plugins (`sase repo open bob-plugins`; read its AGENTS.md). Plugin:
bob-ledger-tools. Copy the vectors verbatim from bob-cli `docs/plan.md`.

1. **Caps.**
   - Add `maxReadyPerNote` (default 5, integer 1–999) to `defaultPlanCaps`,
     `coercePlanCaps` (snake and camel keys), and `effectivePlanCaps`.
   - Cache `loadPlanCaps()` on a stat key (path, mtimeMs, size). Parse only when the key
     changes. Handle creation, deletion, invalid edits and recovery, env path overrides,
     and the mobile defaults. Keep the injectable test seams.
   - `checkReadyCapsAndDay` keeps detecting changes within the 60 s tick.
2. **Frontmatter reader (fixes `bob-cli-3e`).**
   - Add one `noteFrontmatter(pathOrFile)` helper. It resolves a TFile through
     `vault.getAbstractFileByPath` and reads `metadataCache.getFileCache(file)`.
   - Route `noteFreshnessRawFor` and `refreshFreshnessForChangedFile` through it.
   - Make the test stub's `getCache` accept only strings, and `getFileCache` only file
     objects. Add a regression test showing `task_refresh: 30` now applies.
3. **Eligibility cache.**
   - Keep a per-path fingerprint of `type`, `status`, and `ready_cap`, built from
     `app.vault.getMarkdownFiles()` with the contract's directory exclusions.
   - Update it on metadataCache `changed` and `deleted` and on vault `rename`. Bump a
     `noteReadyFrontGen` only when the fingerprint changes.
   - Type forms: a string, an array, and the nested `[["project"]]` array that bare YAML
     wikilinks parse to.
   - `ready_cap`: a number, a digit string, `off` in any case, or `false`.
4. **Snapshot.** Build the snapshot in one O(tasks) pass:
   - **Memo key:** tasks array identity, tasksGen, local date, noteReadyFrontGen, caps
     key, and freshness memo identity (for make-up).
   - **Rows:** count rows with `readyTaskVisible` ∧ `¬planTaskIsBlocked` ∧ ¬recurring
     (the `freshnessRowFromTask` rule) ∧ `planTaskBlockId(task) !== "prj"`, grouped by
     `task.path`.
   - **Make-up:** take buckets from the freshness memo's evaluated rows. If freshness is
     unavailable, make-up is null.
   - **Availability:** availability requires a Tasks array and a `Warm` state. Today
     readiness is not required, because Today is not a filter.
   - **Identity index:** keep a counted-identity index, by task object and path+line, so
     `counted(task)` is O(1).
5. **`api.noteReady`.** Add it inside the frozen api as
   `Object.freeze({version: 1, ...})`; the top-level api stays v3 (additive). Members
   are synchronous and never throw:
   - `snapshot(now?)` →
     `{available, reason?, date, cap: {default, source, invalid}, totals, notes, lints}`.
     Field names follow the contract.
   - `forNote(path)` → a note entry (exempt notes included, with state `exempt`), or
     null when the note is neither eligible nor exempt.
   - `counted(task)` → boolean, or null when unavailable.
   - `inCrowdedNote(task)` → `counted(task) && forNote(path).state === "crowded"`.
   - `groupLabel(task)` → `%%NNN%%[[name]] · count/cap · +k`, where NNN ranks notes by
     excess descending, for crowded.md grouping.
6. **Query refresh.** When the snapshot's crowded set, caps, or labels change without a
   Tasks cache change (for example a `ready_cap` edit), trigger the existing
   `TODAY_RELOAD_EVENT` path so Tasks queries re-read. Avoid recursive reload loops.
7. **Tests** in `scripts/test-ledger-tools-note-ready.cjs`, appended to the `npm test`
   list:
   - R1–R14 verbatim;
   - unavailable states;
   - memo reuse counters (no per-call config parse, no re-walk without a fingerprint
     change);
   - invalidation on task edits, frontmatter edits, rename, delete, day rollover, and
     config edits;
   - throwing-Tasks guards.

   Existing ready-badge, freshness, Today, and plan-budget suites must stay green.

8. **Release.**
   - Bump ledger-tools to 1.15.0.
   - Update the README plugin row and the api prose. Also fix the stale freshness
     `version: 2` mention there.
   - Run `npm test` and `npm run validate`.
   - Deploy with
     `bob plugins sync --no-pull --repo <opened bob-plugins> -p bob-ledger-tools`
     (dry-run first; never `--force` past dirty-file guards). This change is API-only
     and invisible.

## Phase ledger-views

Repo: bob-plugins, plugin bob-ledger-tools.

1. **Shared anchor and fan-out.**
   - Fix `bob-cli-3d`: let `setReadyAnchorContent` keep the caller's chip-kind class (a
     `kind` option) instead of hard-coding `bob-plan-ready`. Add class assertions for
     the READY, PENDING, NEXT, and CROWDED chips across refreshes.
   - Introduce `scheduleLiveWidgetRefresh()` that fans out to READY, lanes, review
     chips, status bar, marks, and the new noteReady widgets. Make the four existing
     fan-out sites call it, with no behavior change for existing widgets.
2. **`api.noteReady.renderCrowdedChip(el, {sourcePath, component})`.**
   - Lifecycle-owned, following the widget-Set pattern with unload cleanup.
   - `<a class="bob-plan-chip bob-plan-crowded …">`, with a label span `CROWDED`, a
     value span, and an `↗` arrow span with `aria-hidden`.
   - Values:
     - `4` with `bob-plan-over` red;
     - `0 ✓`, calm/muted;
     - `–` with `bob-plan-unavailable`.
   - Tooltip and aria-label:
     `4 notes over their ready cap: sase 61/5, sase_remote 11/5, … · 3 full · 66 over. Open Crowded Notes.`
     Name at most five notes.
   - Click, Enter, and Space call `openLinkText("crowded", sourcePath, newLeaf)`;
     Ctrl/Meta opens a new leaf. Mouseover fires `hover-link`.
   - Refresh in place without flicker. Respect the `.text` setter gotcha.
3. **`bob-ready-notes` code block** (`registerMarkdownCodeBlockProcessor`, with a
   `MarkdownRenderChild` and a live widget Set). It renders the same model as the CLI
   overview:
   - a summary line (`CROWDED 4 · 66 over · 3 full · 51 notes`, or
     `CROWDED 0 ✓ · every note has room`);
   - CROWDED and FULL rows as a CSS grid: internal-link name with hover preview, tabular
     `n/cap`, a bar of up to 30 cells with a cap tick and red overflow, a `+k` pill, and
     muted meta;
   - ROOM compressed to `name n` pills, and exempt notes dimmed;
   - a footer with the four remedies and their gestures.

   Unavailable renders a `–` line. It needs no source options in v1.

4. **`## Tasks` heading chip.** This targets eligible and exempt notes only: the first
   heading matching `^ {0,3}##[ \t]+Tasks(?:[ \t].*)?$` (case-insensitive) outside code
   fences.
   - **Live Preview:** a `Decoration.widget({side: 1})` at the heading line's end. A
     `WidgetType` whose `eq` compares a model key. A ViewPlugin that rebuilds on
     docChanged, viewport, file-path, and Live Preview toggles, and on a new
     `noteReadyRefresh` StateEffect dispatched to markdown leaves by the refresh
     fan-out.
   - **Source mode:** no chip.
   - **Reading view:** a markdown post-processor appends the chip to the matching `h2`
     for `ctx.sourcePath`. It is registered for live refresh and cleaned up through
     `ctx.addChild`.
   - **Copy:** `ready 3/5` (room), `ready 5/5 · full`, `ready 11/5 · +6` (red accent and
     excess pill), `ready 0/5`, `ready 65 · no cap`, `ready –`.
   - **Tooltip:** spells out the make-up, cap source, over-by, and remedies, plus the
     invalid-cap lint and the mobile default note when relevant.
   - **Click:** opens `crowded`; mousedown is prevented so the editor cursor doesn't
     jump. It never edits the note.
   - **Test stubs:** add `Decoration.widget` to the test stubs.
5. **CSS** (ledger-tools `styles.css`):
   - accents through theme variables (`--color-red` and `--task-status-blocked` for
     over; muted text for room, full, and exempt);
   - tabular numerals;
   - reduced-motion blocks;
   - no hex colors;
   - light/dark-safe `color-mix` backgrounds matching the existing `.bob-plan-chip`
     family.
6. **Tests:**
   - chip states, classes, text, and aria;
   - refresh and unload;
   - code-block model and render;
   - the heading matcher, including fences, `### Tasks`, two `## Tasks` headings, and
     Reading-view h2 detection;
   - widget `eq` stability;
   - the fan-out helper reaching every family.
7. **Release.** Bump to 1.16.0. Update the README api prose (render members and the
   code-block name). Run `npm test` and `npm run validate`, then deploy as in
   ledger-api. Heading chips appear in notes immediately; the dash chip waits for
   rollout.

## Phase rollout

Repos:

- the vault: `sase repo open gh:bobs-org/bob`; read AGENTS.md and run `git status`
  first;
- bob-cli, for docs.

Read the athena `obsidian.md` reference memory with `/sase_memory_read` before editing
vault notes. Preserve unrelated changes, and stage only your files.

1. **`dash.md`.**
   - Render CROWDED between READY and BLOCKED through
     `ledgerApi.noteReady.renderCrowdedChip.call(…, bar, {sourcePath, component: dv.component})`
     inside try/catch.
   - Fallback: the generic `renderChip` with
     `{key: "crowded", target: "crowded", label: "CROWDED", external: true, destination: "Crowded Notes", unit: "notes"}`.
   - Add `counts.crowded = "–"` and a `.task-count-crowded` accent.
   - Teach `renderChip` a `unit` (aria "notes", not "tasks").
   - Update the chip-order comment.
   - Don't change sections, queries, or other chips.
2. **New `crowded.md`.**
   - Frontmatter: `parent: "[[gtd]]"`, aliases `Crowded Notes` and `Crowded`.
   - Heading `# Crowded Notes`, then an intro: "Back to the [[dash|Dashboard]]. After
     NEW and ROTTEN, clear CROWDED to 0 by splitting, sequencing, deferring, or dropping
     work."
   - A ` ```bob-ready-notes``` ` block.
   - A remedies table:
     - Split: Ctrl+Shift+N then `N`Ctrl+Shift+M, or Ctrl+Alt+Shift+N.
     - Sequence: Ctrl+Shift+P → dependsOn, or `!`.
     - Defer: Ctrl+Shift+P → P1–P4.
     - Drop: Ctrl+Shift+P → cancel.
   - `### CROWDED Tasks`, a Tasks query:
     - `not done`, `status.type is TODO`, and the `_templates`/`_conflicts` path
       filters;
     - a guarded `filter by function … api?.noteReady?.inCrowdedNote?.(task) === true`;
     - a guarded `group by function … groupLabel?.(task) ?? "–"`;
     - `sort by function task.lineNumber`, `short mode`, `hide toolbar`.
   - Verify that the `%%NNN%%` prefix stays hidden. If it renders visibly, drop it and
     accept alphabetical groups.
3. **Inbox exemptions.** Add `ready_cap: off` to the frontmatter of `inbox.md`,
   `gkeep_inbox.md`, and `mac_inbox.md`.
4. **Ritual.** Edit only the text of the `gtd_daily.md` task lines:
   - Daily: insert "then clear [[crowded|CROWDED]] to 0 (split, sequence, defer, drop)"
     after ROTTEN and before PENDING → NEXT.
   - Weekly prune: add "check `bob ready -a`".
   - Keep their repeat, created, and scheduled fields and their statuses.
5. **Triage task.** Add one child task under `bob_gtd.md` `^prj-task-count-warn`:
   `Triage sase.md to its cap with bob ready sase (split, sequence, defer, drop) before gesture notices ship ~2026-10-19`,
   with `[created:: <today>]`. Leave the parent task open; the deferred gesture task
   closes it.
6. **Trial log.** In `rotten.md`'s trial section and in bob-cli `docs/freshness.md` §13,
   add one dated line: per-note CROWDED surfaces (`bob ready`, the dash chip,
   crowded.md, the heading chips) went live on <date>; gesture notices are deferred
   until after the trial.
7. **Docs (bob-cli).**
   - `docs/plan.md` Surfaces: the dash chip order NEW, PENDING, NEXT, READY, CROWDED,
     BLOCKED, ROTTEN, TODAY; and rows for the heading chip, crowded.md, and
     `bob-ready-notes`.
   - `docs/freshness.md` §6: ritual step 4 "clear CROWDED to 0". Note that step 4 does
     not depend on how far ROTTEN review got.
   - `docs/freshness.md` §8: the dash chip list.
8. **Land the vault changes.**
   - Commit in the opened vault clone, as its AGENTS.md says, and push.
   - Make sure `~/bob` receives them through `bob vault-sync`. Run it, or confirm with
     `bob vault-sync status --json`. An edited clone alone is not completion.
   - Confirm the deployed ledger-tools version in `~/bob/.obsidian/plugins`.
   - Other machines need their own `bob plugins sync`. Read `tailnet.md` first if remote
     access is needed.
9. **Verification.**
   - Cross-check `bob ready -f json` against
     `app.plugins.plugins["bob-ledger-tools"].api.noteReady.snapshot()` on the live
     vault at the same date: per-note `count`, `cap`, `state`, and the totals. Do not
     hardcode the research's historical numbers.
   - In Obsidian on athena or apollo, and the Mac where available, verify:
     - the dash chip's text, color, tooltip, keyboard and click, and hover;
     - the crowded.md bars, groups, and click-through to source lines;
     - heading chips in Live Preview and Reading view;
     - `ready 65 · no cap` on gkeep_inbox;
     - live update after completing a task in a crowded note, and after editing
       `ready_cap`;
     - day rollover.
   - Compare `bob task-status-hooks --dry-run` before and after: there must be no
     feature-induced rewrites.
   - If a GUI or machine is unreachable, record exactly what remains unverified and how
     to reproduce it. Never fabricate a pass.

## Deferred: gesture feedback after the freshness trial

The land agent files this as a task bead (see "Epic landing"). Its worker plans first
(size `large`) against the code as it stands then, using this section as the accepted
design.

- **Ledger-tools API additions** (additive; the namespace stays v1):
  - `countedRows(path, rows)`, where rows is `[{line, text}]`. It counts the rows the
    snapshot counted at that path and line whose markdown still matches. When they don't
    match, it falls back to ledger-tools' own syntactic lane test: TODO symbol per the
    Tasks status configuration, no `#hide`, not future-scheduled, not recurring, not
    `^prj`, and no `dependsOn` naming an open task id in the snapshot. Nav therefore
    never duplicates the predicate.
  - `change(before, after)`, applying the rule:

    ```text
    warn     if after > cap ∧ after > before        (crossing or worsening)
    progress if before > cap ∧ after < before       (▼k, or ✓ back to cap/cap when after ≤ cap)
    ```

    Exempt, ineligible, and unavailable notes yield nothing.

  - `chips(results)` → `[{text, tone}]` plus plain text.
  - `watch(paths)` → a receipt that resolves with the before and after counts after the
    first post-write snapshot for those paths, and times out silently.

- **Prevention in the Ctrl+Shift+M picker:**
  - Every area/project row in `TaskMoveDestinationPickerModal` (through
    `renderTypedNotePickerRow`) gets a right-aligned tabular pill: `3/5`, `5/5 full`,
    red `11/5 +6`, or `no cap`.
  - The highlighted row adds `5/5 → 6/5`, red when over. The increment is
    `countedRows(source, session ranges)`.
  - Counts come from one snapshot taken when the picker opens.
  - Order, filtering, and ranking are unchanged.
- **Notices:**
  - **Ctrl+Shift+M:** `commitTaskMoveSession`'s plain `Moved N tasks to X` becomes a
    `.bob-nh-notice` card, `⤵ Moved 2 tasks → sase_pager`, with chips
    `sase_pager ready 9/5 · +4 ▲2` (new `is-over` tone) and `sase ready 59/5 ▼2`, plus a
    remedies line on warn. It keeps a plain-text fallback.
  - **Alt+N release and the lane row:** append to `buildLaneToggleNotice`, for example
    `· 🔴 sase_remote ready 12/5`.
  - **Ctrl+Shift+P:** priority and scheduled rolls, cancel, dependsOn add, Ctrl+D
    clears, and Blocked recovery append chips to their existing cards and texts.
  - **Create project from task and convert note to task:** warn if the new note starts
    over its cap.
  - **Cycler** (Alt+]/Alt+[, Ctrl+Enter reopen and completion recovery, Ctrl+Shift+]
    promote): a `watch` receipt. It shows a standalone notice only for warn or
    `✓ back to cap`.
- **Noise rules:**
  - Each command yields one consolidated chip group.
  - Review stamps (Alt+F, Alt+Shift+F, `]s`) never notify.
  - Hand edits, sync, hooks, the CLI, and mobile changes stay silent.
  - Nothing appears after a cancel or rollback. A partial apply counts only what was
    applied.
  - There is no daily dedupe.
  - An over card stays 8 s.
  - Never suggest promote-to-Next.
- **Tests:**
  - rule vectors N1–N8: 5→6 warn; 7→8 warn; 7→6 ▼; 6→5 ✓; rewording at 7 gives none; a
    review stamp gives none; exempt gives none; unavailable gives none;
  - picker pill and projection rendering through helpers;
  - notice models;
  - cycler receipts against fake snapshots;
  - the nav stylesheet rule (no hex after the priority-notice marker).
- **Release:**
  - Bump nav, cycler, and ledger-tools.
  - Update the README keymap and api prose.
  - Deploy each plugin with `bob plugins sync`.
  - Close vault task `bob_gtd#^prj-task-count-warn` once verified.
- **Keep rule** (two weeks after shipping):
  - crowded notes other than `sase` reach 0 on at least 10 of 14 mornings;
  - `sase`'s excess falls by at least 50%;
  - at most 1 noise notice per day;
  - at most 3 numeric `ready_cap` overrides.

  If it fails, tune N first, then the notice scope.

## Epic landing

The land agent:

1. Verifies all phase evidence.
2. Closes `bob-cli-3d` and `bob-cli-3e` with that evidence, if they are fixed and not
   already closed.
3. Triages any `PROPOSED FOLLOW-UP` notes.
4. Uses `/sase_new_task` to file one `feature` task bead, size `large`, titled "Per-note
   ready cap: gesture feedback (picker pills and capacity notices)". Its description
   points at this plan's "Deferred: gesture feedback after the freshness trial" section,
   and its reason names the trial and the `sase.md` triage.
5. Marks it ready, then snoozes it with
   `sase bead snooze <id> -u 2026-10-19T08:00:00-04:00 -r "Freshness trial runs through 2026-10-18; ship notices after it and after the sase.md triage"`.

The landing report must list any live checks that are still gated, with exact
reproduction steps.

Rollback:

- Revert the dash chip block and crowded.md links together.
- The plugin API is additive and can stay installed.
- Remove `ready_cap: off` only if the feature itself is withdrawn.
- Never roll back by editing task lines.
