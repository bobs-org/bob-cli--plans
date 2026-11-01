---
tier: epic
title: Retire the daily type cap and show Today's root parents
goal: 'No Bob surface counts, caps, warns about, or refuses daily task types any more
  (plan.max_themes and plan.strict are gone; the Task Link cap stays), and the daily
  bob-plan block plus `bob plan` show which root parents today''s planned work goes
  to, as a concise, hue-ringed line after the ★ type line.

  '
decisions:
  root_counts:
    ask: Should each root on the new roots line show how many Today tasks it holds?
    choices:
      counts: Faint tabular count after each label (○ bob 5), like the Day strip legend
      names: Bare labels (○ bob) like the ★ type line; counts move to the tooltip
    default: counts
    why: The balance between roots is the line's most useful fact, for one faint digit.
    answer: counts
  root_ring_hue:
    ask: Which colour should each root's ring wear?
    choices:
      area: Stable area hue, like the ledger home chips; same-area roots share a colour
      strip: The Day strip's distinct bar colour when worked today; may change mid-day
    default: area
    why: Labels already identify roots; a stable identity colour beats one that shifts
      all day.
    answer: area
  memory_decision_record:
    ask: Add a decisions record that the daily plan caps Task Links only, never types?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: mac-drop-type-meter
  title: Bob Mac Capture stops showing the type cap
  depends_on: []
  size: small
  description: 'mac-drop-type-meter: in bob-mac-capture, drop the Types capsule, the
    +N type delta chip, and the red after/cap completion badge, decode old and new
    bob budget JSON, update tests and docs, and compile per AGENTS.md.'
- id: cli-drop-type-cap
  title: bob-cli retires plan.max_themes and plan.strict
  depends_on: []
  size: medium
  description: 'cli-drop-type-cap: remove the type cap from bob-cli config, plan budget,
    capture, routing contract, every strict refusal path, completion previews, reconcile
    output, tests, fixtures, and docs; bump bob plan JSON to schema 3; clean Bryan''s
    chezmoi config; optionally add the decision record.'
- id: cli-today-roots
  title: bob plan reports and prints Today roots
  depends_on:
  - cli-drop-type-cap
  size: medium
  description: 'cli-today-roots: write the Today roots contract in docs/plan.md, compute
    today_roots in Rust from today_tasks via the Pomodoro home resolver, print the
    ROOTS line, and add shared today_roots conformance vectors.'
- id: plugins-drop-type-cap
  title: bob-plugins retires the type cap
  depends_on:
  - cli-drop-type-cap
  size: medium
  description: 'plugins-drop-type-cap: mirror the cap removal across bob-ledger-tools,
    block-id-prompt, bob-navigation-hotkeys, and task-status-cycler (TODAY L/Lc chip,
    caps API, routing, pickers, notices), copy the routing vectors, update tests and
    docs, and sync.'
- id: plugins-today-roots
  title: The roots line in the bob-plan block
  depends_on:
  - cli-today-roots
  - plugins-drop-type-cap
  size: medium
  description: 'plugins-today-roots: add a pure planTodayRoots model, paint the ringed
    roots line after the ★ type line with click-to-open, styles, and repaint wiring,
    and run the shared today_roots vectors plus DOM tests, then sync.'
proposed_by: bbugyi200.apollo.by
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 12:12:33
status: done
bead_id: bob-cli-f9
---

- **PROMPT:** [prompts/202611/retire_type_cap_today_roots.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/retire_type_cap_today_roots.md)
- **BEAD:** [bob-cli-f9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-f9/README.md)

# Retire the daily type cap and show Today's root parents

## Context

The plan budget for today's daily note has two caps: types (`plan.max_themes`, default
3; Bryan's config uses 4) and distinct Task Links (`plan.max_links`). Bryan has never
found the type cap useful (the link cap is useful, so it stays). The type cap is not one
setting. It shows up in:

- **bob-cli**: `plan.max_themes`, plus `plan.strict`. `strict` exists only to refuse
  captures that would add a type past the cap. Also the `themes` meter in `bob plan`,
  `bob task reconcile`, capture's `plan_budget`, and the `capture-pomodoros --tasks`
  agenda; the `plan_theme_cap_exceeded` lint and the `over` status rule; every
  `strict_cap` refusal path (capture link/start/retype, Task Card retype, task move,
  ref-today, successors, typed placement); the `plan_themes_after`/`plan_themes_cap`
  completion preview fields; the routing contract's `themes_before`/`cap`/`strict`
  inputs and their shared vectors; docs; tests; a dev benchmark script.
- **bob-plugins**: the JS mirror of all of the above. The `TODAY 4/4 · 8/13` chip (types
  first). The Work badge rule that forces TODAY red when types are over. The link
  picker's `+1 type · 4/3` hint, the ` · plan 4/4 · 8/13` notice suffixes, the `caps()`
  API's `maxThemes`/`strict`, Task Card strict retype refusals, and the routing mirror.
- **bob-mac-capture**: the `Types 3/3` capsule, the `+1 BOB` type-delta chip, and the
  red `4/3` cap badge on Pomodoro completion rows.
- **chezmoi**: Bryan's Bob config, which still sets `max_themes: 4` and `strict: false`.

In its place, Bryan wants the daily note to show which **root parents** today's planned
work is going to (his task `bob.md#^remove-daily-type-limits`: "Remove daily type limits
in favor of showing daily root parents after themes!"). The model is the
`★ BUILD · READ · WRITE · FIX` type line that the `bob-plan` block already paints after
its chips.

Existing pieces this work reuses: Pomodoro homes (`docs/pomodoro-homes.md`: R2 root
parents, R8 area hues, R11 `[root::]` fields, R7 `○`/`●` home chips), Today
(`docs/plan.md` §Today), and the PT6 visual rule in bob-ledger-tools ("names wear a
tint, places wear a dot").

## Design

### 1. The type cap is gone, not hidden

- Remove `plan.max_themes` and `plan.strict`. A stale key in an old config is silently
  ignored, as the retired `max_now` already is. bob never fails on it and never warns.
- The plan budget keeps one cap: Task Links. `status` is `over` iff
  `links.count > links.cap`. The `plan_theme_cap_exceeded` lint is removed, along with
  the `strict_cap` error code and the `plan_theme_cap_exceeded` capture refusal code. No
  capture, Task Card, move, ref-today, or successor gesture is ever refused for adding a
  type.
- Types themselves stay. Kept: the names, `theme_names` / `theme_refs`, the ★ line,
  `(unnamed)`, `plan.exempt` (it still exempts links, chooses the highlight, and mutes
  the hue), `plan.inventory_labels`, and the `duplicate_open_pomodoro_name` lint.
  Wording that says a type "counts toward" anything is reworded.
- Capture keeps `plan_budget.added_themes` and `added_theme_refs`. Bob Mac Capture uses
  those refs to tint destination, start, toggle, and Refs Today pills, so they are hue
  data, not budget data. Removed: only the `themes` meter, its warning and refusal, and
  the human `(+1 type: BOB)` suffix.
- Chip and meter text: `TODAY 4/4 · 8/13` becomes `TODAY 8/13`; `bob plan` prints
  `PLAN  8/13 links`. The `plan …` lines of `bob task reconcile` and capture drop their
  types meter and keep everything else. The Mac shows only `Links 8/13`.
- Contract versions: `bob plan --json` goes to `schema_version` 3 because it drops
  `themes`, `caps.max_themes`, and `caps.strict`. The capture, `capture-complete`, and
  `capture-pomodoros --tasks` envelopes keep their versions. Bob Mac Capture
  hard-requires agenda `schema_version == 1`, and it already decodes a missing meter.
  Removing fields there is safe for the only client. bob-cli commits that change
  CLI/JSON contracts use a `!` subject, per the release-train decision. Never edit
  `Cargo.toml`'s version or `CHANGELOG.md`.

### 2. Today roots: where today's planned work goes

**Definition (Rust owns it; JS mirrors it).** Today roots are the root parents of
Today's tasks. Today's tasks are exactly the `today_tasks` rows (§Today): open `#task`
lines behind dedicated Task Links under the note's **open** Pomodoros, including exempt
PLAN entries and unnamed placeholders, deduplicated by (path, block id).

- Each task's root is resolved by Pomodoro homes R1/R2/R11 against the current note
  tree. A valid `[root:: X]` on the task line wins. Archive and ref notes are walked
  through. A task with no area or project lands in the `@unfiled` bucket.
- Group by root key (the root note's vault path with `.md`, or `@unfiled`). `tasks` =
  distinct Today tasks under that root.
- Order: `tasks` descending, ties by the Today order of each root's first task, and
  `@unfiled` always last.
- Label: lowercase root stem, or `unfiled` for the bucket. Hue: per decision
  `root_ring_hue`, the root's R8 area hue by default; `null` for the bucket.
- Invariant: the sum of `tasks` equals `today.count`, so the roots line partitions
  TODAY.
- It is a view, never a budget. It has no cap, no lint, no status effect, and nothing
  written to the vault.
- Unlike the per-entry home chips (R3), which forecast every linked task including done
  ones, Today roots count only open Today tasks. A task completed mid-session leaves the
  line.

**Obsidian `bob-plan` block (the primary surface).** The line sits directly after the ★
type line, as its own flex item in the chip row. It wraps as a unit when the row is
full. The grammar is PT6: types wear a tinted pill, places wear a dot.

```
[TODAY 8/13] [PENDING 9/8] [NEXT 12/13] [READY 15/21]  ★ BUILD · READ · WRITE · FIX   ○ bob 5   ○ sase 2   ○ sase_blog 1
```

- **Anatomy, per root**: a hollow 7 px ring (1.5 px border in `--bob-area-<hue>`), a
  `--text-muted` label, and (decision `root_counts`) a `--text-faint` tabular count
  after a no-break space.
  - The hollow ring is the existing forecast mark of the ledger's home chips: planned,
    not yet worked. The Day strip's filled `●` above it shows worked time, so the two
    read as "worked so far" vs "still planned".
  - The `unfiled` bucket uses a dashed `--text-faint` ring. That echoes the Day strip's
    hatched "unattributed" tone.
  - There is no leading glyph: the rings are the glyph. Roots are separated by `0.85em`
    gaps, with no `·` separators. Font size and colour match `.bob-plan-themes` (0.9em,
    muted).
- **Interaction**:
  - Each real root is a quiet link. Click or Enter opens the root note (Mod-click opens
    it in a new tab).
  - On hover or focus, the label goes `--text-normal` and the ring fills with a 35% tint
    of its hue. Focus-visible uses the plan chips' outline.
  - `unfiled` is not interactive.
  - There are no animations, and no Obsidian hover-preview popover.
- **Text**:
  - Item `title`: `bob · 5 scheduled tasks` (singular `task`) plus a second line,
    `Open bob`.
  - `unfiled` title: `unfiled · 1 scheduled task` and
    `File the task in an area or project, or give it a root field` (the R6 hint).
  - Group: `role="group"` with
    `aria-label="Root parents scheduled: bob, 5 tasks; sase, 2 tasks; sase_blog, 1 task"`.
    Rings are `aria-hidden`.
- **Truthful or neutral**:
  - With no Today tasks, or no Pomodoros section, there is no line and no placeholder.
  - Until the Tasks cache is available, the line is not painted rather than painted with
    guessed statuses. It appears on the next repaint, using the triggers that already
    repaint the lane badges.
  - Older daily notes compute from their own content, like the ★ line.
- **Colour note** (decision `root_ring_hue` = `area`): roots in one area share a hue.
  The Day strip may re-hue same-area bar segments for legibility. The roots line keeps
  each root's stable identity colour.

> [!decision] root_ring_hue = strip Only the Obsidian ring changes; Rust, JSON, and the
> CLI keep the area hue. When the Day strip has a cached report for this note's day, the
> plugin colours each root it lists with that strip's colour (the
> `statsPanelDistinctHues` map the roots lane already paints). Roots the strip does not
> list keep their area hue. The roots line repaints when the strip's report refreshes.

> [!decision] root_counts = names Each item is ring and label only, on both surfaces:
> the Obsidian line and the CLI `ROOTS` line drop the count. JSON keeps `tasks`, and the
> item `title` and group `aria-label` still state the count.

**`bob plan` (CLI parity).** Add a `ROOTS` line under the `PLAN` meter line, only when
there is at least one root:

```
  PLAN  8/13 links      TODAY 8 · PENDING 12/8 · NEXT 12/13
  ROOTS ○ bob 5 · ○ sase 2 · ○ sase_blog 1
```

- Each `○` is painted with `styler.xterm256(hue.xterm(), "○")`, the same dot painting as
  `capture_pomodoros::homes_chip_suffix`. `unfiled` is dim.
- Labels are normal and counts are dim. Separators use `styler.separator()`, and
  `NO_COLOR` yields plain text.
- JSON adds an always-present `today_roots` array next to `today_tasks`:
  `[{"key": "bob.md", "label": "bob", "hue": "blue", "tasks": 5}, …]`. It is `[]`
  without a daily note or section.

**Conformance.** A new top-level `today_roots` case list lives in
`tests/fixtures/pomodoro_homes/vectors.json`, reusing that file's `notes` vault plus
task notes. bob-plugins copies the file byte-for-byte and runs the same cases.

**Not in scope**: a Mac agenda roots row, any root cap, and toggling the line off. The
block has no per-line toggles for the ★ line either.

## Phases

### Phase `mac-drop-type-meter` — Bob Mac Capture stops showing the type cap

- Size: small
- Depends on: none

Work in the `bob-mac-capture` linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "…"`) and read its `AGENTS.md` first: Swift compile
rules, `just portable-check`, and the best-effort Mac typecheck. bob is dropping the
type cap (`plan.max_themes`/`plan.strict`) in phase `cli-drop-type-cap`. This app must
stop rendering it, and must work against both older bob (which still sends the fields)
and newer bob (which omits them).

1. `Sources/CaptureCore/CaptureModels.swift`: remove `CapturePlanBudget.themes` (field,
   init parameter, decode, coding key). Decoding must simply ignore a `themes` key from
   older bob. Keep `links`, `status`, `addedThemes`, `addedThemeRefs`, and `warnings`.
   `addedThemeRefs` still tints destination/preview pills.
   `CapturePlanBudgetMeter.before` stays optional.
2. `Sources/CaptureCore/CapturePlanBudgetPresentation.swift`: remove
   `themesCapsuleText`, `themesOverCap`, `deltaChipText`, and `deltaThemeSwatches` (the
   `+1 BOB` chip only explained type-cap growth). The accessibility summary becomes
   destination, `Links L/Lc`, and the warnings. Keep `destinationThemeSwatches` and
   `swatches(forName:budget:)`. Update the doc comment's wording mirror
   (`plan L/Lc links`).
3. `Sources/BobMacCapture/CapturePlanBudgetMeterRow.swift`: render only the Links
   capsule plus the warnings. Delete the delta chip, `deltaTint`, and `deltaSpokenHue`.
4. `Sources/CaptureCore/CaptureAgendaPresentation.swift` (around the
   `themesCapsuleText, linksCapsuleText` summary) and any agenda fit/measure code
   (`CaptureAgendaFitPlanner.swift`, `CaptureAgendaRowMeasurer.swift`) that budgets for
   the types capsule.
5. `Sources/CaptureCore/CompletionRowContent.swift`: remove the `planCapBadge` red
   `after/cap` badge and the `planThemesAfter`/`planThemesCap` candidate fields (model
   and decode), wherever they are declared.
6. Tests in `Tests/CaptureCoreTests/` (`CapturePlanBudgetPresentationTests.swift`,
   `CaptureAgendaPresentationTests.swift`, completion-row tests): delete type-cap
   assertions. Add one test that a budget JSON _with_ an old `themes` meter and
   `plan_themes_after`/`plan_themes_cap` still decodes and renders only `Links …`.
7. `docs/capture-previews.md`: drop `Types 3/3` and the `+1` chip from the description.
   Run `just docs-check`.

Verify with `just portable-check` (or `just test` where Swift/Xcode exists). For
`Sources/BobMacCapture*`, run the Mac typecheck per `AGENTS.md`, and say so if the Mac
was offline.

### Phase `cli-drop-type-cap` — bob-cli retires `plan.max_themes` and `plan.strict`

- Size: medium
- Depends on: none

Implement Design §1 in bob-cli: remove every trace of the daily type cap while keeping
types, the ★ data, and the link cap.

1. **Config** (`src/native/config/plan.rs`, plus tests in `config/mod.rs` and
   `config/stats.rs`): delete the `max_themes`/`strict` fields, defaults, getters,
   parsing, and tests. Stale keys are ignored like `max_now`; widen that code comment
   and add a test that a config with `max_themes: 0`/`strict: true` loads cleanly.
2. **Plan budget** (`src/native/plan_budget/{mod,cli,today}.rs`):
   - Drop `PlanCaps.max_themes`/`strict`, `LedgerBudget.themes`, `PlanReport.themes`,
     and `LINT_THEME_CAP`. `status` depends on links only.
   - Keep `theme_names`, `theme_refs`, `UNNAMED_THEME`, exempt handling, and the other
     lints. Reword the `inventory_label_open` message so it no longer says the label
     "counts".
   - `bob plan` human meter: `PLAN  L/Lc links      TODAY …`. `report_without_ledger`
     follows.
   - JSON `SCHEMA_VERSION` 2 → 3. Update help text.
3. **Capture**
   (`src/native/capture/{budget,output,cli,mod,pomodoro_link,pomodoro_start,task_theme}.rs`):
   - Remove the `themes` meter from `CapturePlanBudget`, the type-cap warning, the
     strict refusal, `theme_warning`, `theme_refusal`, `CaptureError::strict`,
     `strict_type_cap`, and the `strict_cap` and `plan_theme_cap_exceeded` error codes.
   - Keep `added_themes`/`added_theme_refs`.
   - The human meter line becomes `plan L/Lc links` with no `(+N type: …)` suffix.
4. **Shared routing contract**
   (`src/native/pomodoro_themes/{routing,relocation_apply}.rs`):
   - Drop `themes_before`, `cap`, and `strict` from `PlaceInput`/`RelocateInput`. Drop
     `ledger_keys` too if it then only fed `adds_theme`.
   - Delete the `Refused` variants (`strict_cap` is their only code) and
     `StrictCapRefusal`. The applier returns its `Option` directly.
   - Update `tests/fixtures/pomodoro_themes/vectors.json` `place`/`relocate` cases to
     match: remove the inputs and the strict cases, and leave every other section
     byte-identical. bob-plugins copies this file in phase `plugins-drop-type-cap`.
5. **Other strict/cap callers**:
   - `src/native/capture_task_toggle/{links,relocation,ledger,tests}.rs` (`TypeCap`).
   - `src/native/task_move/command.rs` (the `plan.strict: … would be type N of M`
     refusal).
   - `src/native/ref_library/today.rs` and `src/native/task_complete/successors.rs`.
   - `src/native/capture_complete/{pomodoros,model,cli}.rs`: delete
     `plan_themes_after`/`plan_themes_cap`, `plan_themes_after_for`, and the cap parts
     of `PlanCreationHint`. Delete the hint entirely if nothing else uses it.
   - `src/native/capture_pomodoros_agenda.rs`: the agenda's `plan_budget` drops
     `themes`, but agenda `schema_version` stays 1 (Mac hard-checks it).
   - `src/native/task_status_hooks/output.rs`: the human `plan …` line and JSON caps.
6. **Tests and fixtures**:
   - Delete cap/strict tests:
     `tests/cli/capture/{plan_budget,typed_placement,pomodoros_agenda,…}.rs`,
     `tests/cli/{plan,task_move}.rs`, `tests/cli/task_status_hooks/sync.rs`, and src
     unit tests.
   - Remove `max_themes: 30` config lines that existed only to dodge the cap. Find them
     with `rg -n "max_themes" tests src`.
   - Update `tests/fixtures/capture_pomodoros/agenda-*.json` and the help fixtures.
     Update `scripts/dev/benchmark_theme_defaults.py`.
7. **Docs**:
   - `docs/plan.md`: the intro sentence, §Types (types are listed, not budgeted),
     §Status, the §Lints table, the TODAY badge paragraph (`TODAY L/Lc`, no type red),
     §Config sample and validation sentence, the `bob plan` JSON sample (schema 3), and
     the conformance examples.
   - `docs/capture.md`: rename §"Plan budget and strict mode" to "Plan budget", keep its
     anchor working via an explicit `{#plan-budget-and-strict-mode}` or update every
     in-repo link. Also remove the `strict_cap` and refusal JSON, the `code` sentence
     (keep `pull_in_choice_required`), the completion `plan_themes_*` text, and the
     refusal mentions near lines 543, 1290, and 3270.
   - Also update `docs/task-types.md`, `docs/task-status-hooks.md`,
     `docs/task-dependencies.md`, `docs/reference/configuration.md`,
     `docs/reference/json-outputs.md`, and `docs/_templates/how-to.md`.
   - Regenerate with `just docs-gen`. Read the `docs.md` reference memory first. Mounted
     plugin docs belong to phase `plugins-drop-type-cap`.
8. **Bryan's config**: open the `chezmoi` linked repo with `/sase_repo`, then find the
   Bob config source that holds the `plan:` block (`rg -n "max_themes" .`). Delete the
   `max_themes:` and `strict:` lines and their comments, and leave everything else
   untouched. Do not run `chezmoi apply`; bob ignores the stale live keys until Bryan
   applies.
9. **Memory** (only if decision `memory_decision_record` is true): use
   `/sase_memory_write` to add `sase/memory/decisions/plan-caps-links-not-types.md`.
   - **Applies to**: bob-cli plan/capture, the bob-plugins `bob-plan` block and pickers,
     and Bob Mac Capture.
   - **Claim**: the daily plan caps distinct Task Links only. There is no type cap and
     no strict mode. Today roots are a view, never a budget.
   - **Why**: Bryan never found the type cap useful, unlike the link cap
     (`bob.md#^remove-daily-type-limits`, 2026-11-01).
   - **Rejected**: "keep the cap but default it off" keeps dead config and refusal paths
     alive; "cap root parents instead" has no evidence of value.
   - **Cost**: no guard against many-type days, and stale keys are silent.
   - **Reopens when**: Bryan asks for a type or root cap.
   - Then run `sase memory init` and `just docs-gen` (regenerates
     `docs/develop/decisions.md`).
   - Leave the accepted `theme-hue-is-config-pin-exempt-area-hash` record untouched. Its
     historical rationale mentions `plan.max_themes`, and accepted records are
     immutable.

> [!decision] memory_decision_record Step 9 runs only when this is accepted. Declined:
> no record is written, and a phase worker records nothing about it.

Gate: `just all` (fmt, lint, test) and `just docs-check` pass. Then this may print only
these hits: the stale-key comment and test in `src/native/config/plan.rs` (step 1),
records under `sase/memory/decisions/`, and the generated `docs/develop/decisions.md`:

```
rg -n "max_themes|plan\.strict|strict_cap|plan_theme_cap_exceeded|plan_themes_(after|cap)|maxThemes|type cap|types past" src tests docs scripts sase/memory
```

Use a `!` commit subject, for example
`feat(plan)!: retire the daily type cap and plan.strict`.

### Phase `cli-today-roots` — `bob plan` reports and prints Today roots

- Size: medium
- Depends on: cli-drop-type-cap

Implement Design §2's definition and CLI surface in bob-cli, and author the contract and
conformance vectors that bob-plugins will mirror.

1. **Contract**: add `## Today roots` to `docs/plan.md`, right after `## Today`.
   - It states the definition, order, label/hue rule (as decided in `root_ring_hue`),
     the partition invariant, the difference from home chips, and "a view, never a
     budget".
   - It also gives the Obsidian line's display contract (anatomy, text, interaction,
     degrade rules, exactly as in Design §2) and the CLI line.
   - Update §Surfaces. Add one cross-link sentence to `docs/pomodoro-homes.md` R7.
2. **Rust**: add a small module, for example `src/native/plan_budget/roots.rs`, with:
   - `TodayRoot { key, label, hue: Option<Hue>, tasks }` (serde: `hue` nullable).
   - `with_today_roots(report, bob_dir, daily_path)`.
   - It uses one `pomodoro_homes::HomeResolver<DiskHomeSource>`. For each `TodayTask`,
     call `resolver.home_for_link(daily_path, LinkResolution::Found(path), &block_id)`,
     which honors block `[root::]` facts (R11), and read `home.root_parent`.
   - Label: lowercase stem, or `unfiled`. Hue: the root note's resolved area hue
     (`resolve_note(root).hue`), or `None` for `@unfiled`.
   - Group, count, and order per Design §2. Call it from `show_plan` after
     `today_tasks`.
   - Add `today_roots: Vec<TodayRoot>` to `PlanReport` (always serialized;
     `report_without_ledger` gives `[]`).
3. **Human output**: add the `ROOTS` line under `PLAN` in `human_success`, painted as in
   Design §2. Omit it when empty.
4. **Vectors**: add a top-level `today_roots` array to
   `tests/fixtures/pomodoro_homes/vectors.json`. Each case is
   `{name, daily_path, daily, tasks_notes?, expected}`; reuse the shared `notes` and add
   task notes. Expected rows are `{key,label,hue,tasks}`. Cover:
   - tie order, and a `[root::]` field;
   - a `root: true` project, and a sub-project grouping under its top project;
   - an archive note;
   - a done task and a cancelled task (excluded);
   - an exempt PLAN entry and an unnamed placeholder (included);
   - a closed entry, a struck link, and an unresolved link (excluded);
   - a duplicate link (counted once);
   - `@unfiled` last, and no section → `[]`.

   The existing `cases` stay byte-identical. Drive them from a Rust test that writes the
   notes into a temp vault, using the existing pomodoro_homes vector harness's patterns.
   Assert `sum(tasks) == today.count` in every case.

5. Update `docs/reference/json-outputs.md` (`today_roots`, schema 3) and the `bob plan`
   section of `docs/plan.md`, then run `just docs-gen`.

Gate: `just all` and `just docs-check`. `bob plan` and `bob plan -f json` against
Bryan's vault show roots whose counts sum to TODAY. This is an additive JSON field, so a
`feat(plan): …` subject is enough.

### Phase `plugins-drop-type-cap` — bob-plugins retires the type cap

- Size: medium
- Depends on: cli-drop-type-cap

Work in the `bob-plugins` linked repo. Open it with `/sase_repo` and read its
`AGENTS.md`:

- Edit `src/` fragments, then run `npm run build`; never hand-edit generated `main.js`.
- Each hand-edited fragment must stay at or below 1000 lines.
- After changes, run `bob plugins sync`.

Mirror bob-cli phase `cli-drop-type-cap` (read its diff in bob-cli and the updated
`docs/plan.md`). All in-repo consumers of the changed ledger-tools APIs change in this
same commit, so nothing ever validates a missing `maxThemes`/`strict`.

1. **bob-ledger-tools**:
   - `src/050-plan-budget.js`: drop `maxThemes`/`strict` from defaults, config parsing
     (stale `max_themes`/`strict` keys ignored, never "config invalid"), the `themes`
     meter, the status rule, and the lint.
   - `src/075-work-badge-tone.js`: drop the themes validation and the red forcing.
   - `src/340-dashboard-views-and-plan-block.js`: `planText` becomes `TODAY L/Lc`. Keep
     `planTitle`, the ★ `themesText`, and `themePills`.
   - `src/279-plugin-task-theme-api.js` (`cap`/`strict` plumbing) and
     `src/279-plugin-task-type-routing.js`: `place`/`relocate` mirror the new Rust
     inputs, with no `refused` outcome.
   - `src/350-exports.js` and the `caps()` API shape.
2. **block-id-prompt**:
   - `src/076-pomodoro-theme-picker.js`: remove the strict fallback and the `caps()`
     validation of `maxThemes`/`strict`.
   - `src/085-pomodoro-link-picker-modal.js`: the meter text becomes `plan L/Lc`, with
     no `+1 type · T/Tc` hint and no themes-over state.
   - `src/130-plugin-task-link-open-and-notices.js`: the notice suffix becomes
     ` · plan L/Lc` plus ` 🔴` when links are over.
3. **bob-navigation-hotkeys** (`src/280-notices-and-dates.js` meter,
   `src/316-task-card-type-relocation.js`, `src/655-plugin-inbox-route.js`) and
   **task-status-cycler** (`src/085c-successor-theme-write.js`): remove strict/cap
   handling.
4. **Vectors**: copy bob-cli's `tests/fixtures/pomodoro_themes/vectors.json`
   byte-for-byte to `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/`. Update
   that README's source commit.
5. **Tests**: update or delete cap/strict cases in `scripts/test-*.cjs`. Find them with
   `rg -n "maxThemes|strict_cap|themes\.(over|cap)|plan_theme_cap" scripts`. Key files:
   `test-ledger-tools-plan-budget`, `-work-badge-tone`,
   `test-block-id-prompt-pomodoro-*`, `-plan-budget-and-freshness`,
   `test-navigation-task-card-theme-edit`, and `test-task-status-cycler-successor-*`.
6. **Docs**: `docs/bob-ledger-tools.md` (`TODAY 3/3 · 7/10` → `TODAY 7/10`),
   `docs/task-card.md` (strict refusal paragraph), and `docs/develop/plugin-apis.md`
   (`caps()` shape).

Gate:

- `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` pass.
- This prints nothing except a stale-key config test, in the style of the existing
  `max_now` coverage:
  `rg -n "maxThemes|max_themes|strict_cap|plan_theme_cap|themes\.over|themes\.cap|\+1 type" plugins/*/src scripts docs`
- Then run `bob plugins sync`.

### Phase `plugins-today-roots` — the roots line in the `bob-plan` block

- Size: medium
- Depends on: cli-today-roots, plugins-drop-type-cap

Work in `bob-plugins` (open it with `/sase_repo`; same `AGENTS.md` rules as above).
Build the Obsidian surface from Design §2, mirroring the contract that phase
`cli-today-roots` added to bob-cli `docs/plan.md` §Today roots. That contract is the
source of truth, and decisions `root_counts`/`root_ring_hue` are final.

1. **Pure model** in a new fragment, for example
   `plugins/bob-ledger-tools/src/142b-plan-today-roots.js`, registered in
   `src/fragments.json` after its dependencies. `340-…` is already at about 995 lines,
   so add nothing there beyond the call.
   - Signature:
     `planTodayRoots({ content, dailyPath, metadataCache, vault, tasks, taskLineForBlock })`
     returns `null` (Tasks unavailable) or `[{ key, label, hue, tasks }]`.
   - Reuse `computeTodayLinks`, `resolveTodayLinkKeys`, and the Pomodoro-home resolver
     for root parents, including `[root::]` lines via `taskLineForBlock`. The same link
     and resolution rules feed the home chips, so roots and chips cannot disagree.
   - A key counts only when the Tasks cache holds an open task for it (not `x`, `X`, or
     `-`).
   - Never throws.
2. **Model and paint**:
   - `planBlockModel` gains `todayRoots`, computed only when `budget.hasSection`.
   - In `src/290-plugin-today-and-location.js`, add
     `paintPlanTodayRoots(container, roots, sourcePath)` right after the ★ line.
   - DOM: `span.bob-plan-roots[role=group]` holding `span.bob-plan-root` items, each
     with a `[data-root-hue]` ring, a label, and a count. Text, `title`s, the group
     `aria-label`, and keyboard/click behaviour follow Design §2.
   - Open notes with
     `app.workspace.openLinkText(stem, sourcePath, Keymap.isModEvent(evt))`. Swallow
     `mousedown` so Live Preview never moves the cursor into the fence.
3. **CSS** (`plugins/bob-ledger-tools/styles.css`, beside `.bob-plan-themes`): rings use
   the existing `--bob-area-<hue>` tokens (as `.bob-pomodoro-home-dot` does), with the
   dashed faint ring for `unfiled` and the hover/focus fill. It must work in both light
   and dark themes, with no motion.
4. **Repaint**: verify the plan block repaints when a linked task's status changes and
   when Tasks data first arrives; the lane badges already need both. If a parent-chain
   frontmatter edit does not repaint, hook the existing debounced plan refresh to
   `metadataCache` `changed`.
5. **Conformance**: copy bob-cli's `tests/fixtures/pomodoro_homes/vectors.json`
   byte-for-byte to `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/` and
   update that README.
   - Add `scripts/test-ledger-tools-plan-today-roots.cjs` (register it in `package.json`
     `test`). It runs every `today_roots` case through `planTodayRoots`, extending the
     existing fake `metadataCache` and building a fake `tasks` list from the notes.
   - Add DOM tests: ring hue attribute, the count text per `root_counts`, `unfiled` last
     and non-interactive, the aria-label sentence, no line when `tasks` is unavailable
     or there are no Today tasks, and placement after `.bob-plan-themes`.
6. **Docs**: describe the line in `docs/bob-ledger-tools.md`, next to the ★ type line.

Gate:

- `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` pass.
- Run `bob plugins sync`.
- Leave this checklist in the final report for Bryan's live check:
  - today's note shows `○ bob 5  ○ sase 2 …` after the ★ line, and the counts sum to
    `bob plan`'s TODAY;
  - completing a linked task drops its count;
  - clicking a root opens its note;
  - dark and light themes both look right.
