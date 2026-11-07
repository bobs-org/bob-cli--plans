---
tier: epic
title: Root the daily ^gtd row at gtd and give areas and projects configurable colors
goal: 'New daily notes root their ^gtd row at gtd.md and the built-in Review time
  keeps counting that time. Any area or project note can set its own color with a
  `color:` slot, either a palette hue or grey, and every Bob surface honors it, so
  the gtd area renders grey like the PLAN type in the Day and period stats strips.

  '
decisions:
  history:
    ask: Which existing daily notes should have their ^gtd row re-rooted at gtd?
    choices:
      recent: Notes dated 2026-11-07 or later; today's strip shows gtd now, older
        days keep gtd_daily
      template: Only notes created from the template after the change; today keeps
        gtd_daily
      all: Every daily note (about 143); past strips and Stats merge gtd_daily time
        into gtd
    default: recent
    why: '"Start using" means going forward, and old gtd_daily time turns grey anyway'
    answer: all
  gtd_link:
    ask: Also change the daily ^gtd row's visible link from [[gtd_daily]] to [[gtd]]?
    default: false
    why: '[[gtd_daily]] still opens the PRE/POST checklist the row stands for'
    answer: false
  memory_root_parent:
    ask: Correct the Root Parent glossary entry to say daily review time is rooted
      at gtd?
    memory:
    - glossary:root-parent
    corrects: Daily review time is now the time attributed to tasks rooted at `gtd_daily`.
    default: true
    answer: true
  memory_area_color_record:
    ask: Add a decisions record for nearest-note area/project colors and the grey
      mute?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: cli
  title: bob-cli nearest-note colors, grey mute, and Review time on gtd
  depends_on: []
  size: medium
  description: 'cli: in bob-cli, resolve each home''s hue from the nearest area or
    project `color:` slot (palette hue or grey), report `hue_source`, give root groups
    the root''s own hue, mute types named after a grey top-level area, point the built-in
    review_time at gtd, extend the shared vectors, and update docs and memory.'
- id: plugins
  title: bob-plugins mirrors and grey root lanes
  depends_on:
  - cli
  size: medium
  description: 'plugins: in bob-plugins, mirror nearest-note colors and the grey mute
    in the home-chip and type-hue mirrors, keep muted roots gray in every root lane,
    copy the shared vectors byte for byte, then test, build, and run `bob plugins
    sync`.'
- id: vault
  title: Vault adoption and live verification
  depends_on:
  - cli
  - plugins
  size: small
  description: 'vault: install bob from master, re-root the daily template''s ^gtd
    row at gtd, set gtd.md to `color: grey`, rewrite the daily notes the history decision
    picks, and verify the live stats JSON and strips.'
proposed_by: bbugyi200.athena.19s
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 17:26:26
status: wip
bead_id: bob-cli-ij
---

- **PROMPT:** [prompts/202611/gtd_root_and_area_colors.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gtd_root_and_area_colors.md)
- **BEAD:** [bob-cli-ij](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ij/README.md)

# Plan: Root the daily ^gtd row at gtd and give areas and projects configurable colors

## What exists today

- **The `^gtd` row comes from the vault template.** In `~/bob/_templates/daily.md`
  (Templater), the row reads
  `- [*] #task [[gtd_daily]] [root:: gtd_daily] [created::…] ^gtd`. No bob-cli,
  bob-plugins, or Mac code writes that row; they only test against copies of it.
- **Colors exist only partly.** `pomodoro_homes::resolve::resolve_note` reads `color:`
  only from the **top-level** area (parent `[[area]]`). It accepts only the eight
  palette slots (`Hue::parse`), so there is no grey. Sub-areas and projects always
  inherit the top area's hue, and `color:` on them is silently ignored. Today `gtd.md`
  has `color: orange`, so `gtd_daily` and everything under gtd is orange.
- **A home with no hue renders neutral almost everywhere.** `hue: None` already means a
  dim/neutral dot in the CLI (`plan_budget/roots.rs`, `capture_pomodoros.rs`,
  `gcal/ui.rs`), the Mac app (`PomodoroHomeHue?`), and the plugin home chips. The
  exception is the stats root lanes. bob-plugins `statsPanelDistinctHues`
  (`239-plugin-stats-panel-models.js`) gives every non-bucket root a palette hue, so a
  root with `hue: null` would be recolored. The Day strip, the period strip's lane and
  parts row, and the Stats page all go through that function.
- **Built-in `review_time` is `in: [gtd_daily]`** (`stats/registry.rs`). Once `^gtd` is
  rooted at `gtd`, the walk starts at `gtd.md` and never passes through `gtd_daily`, so
  Review time would drop to zero. Live data for the last 30 days: gtd_daily 1697m,
  gtd.md's own tasks 45m, and nothing on the inbox sub-areas. So `in: [gtd]` keeps the
  metric almost the same, and old days rooted at `gtd_daily` still count because
  `gtd_daily` sits under `gtd`.
- **JavaScript mirrors with shared vectors.** bob-plugins mirrors the homes resolver in
  `142-pomodoro-home-chips.js` (`pomodoroHomeResolve`, `pomodoroHomeHueForArea`,
  `pomodoroHomeRootGroups`). It mirrors the type hue rule in
  `279-plugin-pomodoro-theme-mirrors.js` (`pomodoroThemeResolveHue`) and builds the area
  map in `279-plugin-pomodoro-themes.js` (`pomodoroThemesAreasMap`). The bob-cli files
  `tests/fixtures/pomodoro_homes/vectors.json` and
  `tests/fixtures/pomodoro_themes/vectors.json` are copied byte for byte into
  bob-plugins: `plugins/bob-ledger-tools/scripts/fixtures/…`, plus
  `scripts/fixtures/pomodoro_themes/vectors.json` for the themes file.

## Design: `color:` on any area or project note

The general-purpose interface is the frontmatter key the vault already uses: seven
top-level areas already carry `color:`. This plan extends it rather than adding a second
mechanism.

1. **Values.** A `color:` value is one of the eight palette slots (`blue`, `orange`,
   `aqua`, `yellow`, `magenta`, `green`, `violet`, `red`) or `grey` (`gray` also
   accepted). Matching ignores case, and YAML quotes are trimmed as today. An absent or
   blank value inherits. Any other value is invalid. It produces one
   `area_color_invalid` lint per note and is then treated as absent, so the note
   inherits. The lint code stays the same for JSON compatibility, but it now covers any
   area or project note.
2. **Nearest note wins.** Walk the note's chain from its nearest home outward, over
   **area and project notes only**. Daily, reference, and `done/` wrappers and `area.md`
   never count. The first valid slot decides:
   - a palette hue: `hue: <slot>`, `hue_source: "color"`;
   - grey: `hue: null`, `hue_source: "muted"`;
   - no slot anywhere, with a top-level area: today's top-area stem hash,
     `hue_source: "hash"`;
   - no slot and no area: `hue: null` with no `hue_source` (today's `@unfiled`
     behaviour). Buckets (`@unlinked`, `@unfiled`) have no `hue_source`.
3. **A root wears its own hue.** A root group's hue (Pomodoro chips, the JS mirror,
   Today roots) is the hue resolved for the **root note itself**. Today it comes from
   the root's first part. That rule silently assumed the whole chain shares one hue,
   which stops being true once a sub-project can override. The chip, Today roots, and
   the stats root lane must agree.
4. **Grey is a choice, not a missing value.** Every surface already renders `hue: null`
   as neutral. The stats `homes` catalog adds `hue_source` (additive JSON) so the strips
   can tell a deliberately grey root from an uncolored one. A muted root stays gray, is
   never shaded, never takes a palette hue, and sorts after coloured roots and before
   the buckets. `statsPanelHomeAreaRows` already puts gray slot-8 rows there.
5. **Types named after a top-level area** (type rule PT5, step 5) wear that area's
   resolved hue. If the area is grey, the type is `hue: null`, `hue_source: "area"`.
   Config colors, pins, and the exempt and unconfigured mutes still come first, so PLAN
   and Bryan's configured types are unchanged.
6. **Collisions.** `area_hue_collision` still compares top-level areas only. A muted
   area never collides.

Nothing changes in the current vault until a note opts in, because only top-level areas
carry `color:` today. Rejected alternatives:

- **An `areas:` block in `config.yml`.** It separates a note's color from the note,
  breaks on renames, and the JS mirrors read frontmatter, not bob's config.
- **Deriving grey from an exempt `task_type:` such as PLAN.** It couples two independent
  keys (`meet.md` is MEET-typed but violet), and the JS home mirror cannot see
  `plan.exempt`.
- **A new `gray` `Hue` variant in JSON.** Strict decoders would break, while `hue: null`
  is already the neutral contract everywhere.

## Phase `cli`: bob-cli

Work in bob-cli only; the `plugins` phase owns the bob-plugins copies of the vectors.

1. **Parse.** In `src/native/pomodoro_homes/facts.rs`, parse `color:` into a `NoteColor`
   enum (`Hue(Hue)`, `Grey`, `Invalid(String)`; absent or blank is `None`). Keep the
   grey/gray spelling rule in one helper next to `Hue::parse` in `palette.rs`.
2. **Resolve.** In `resolve.rs::resolve_note`, replace the top-area-only hue block with
   the nearest-note walk from the Design section. Generalize `lint_color` to any area or
   project note (once per path). Add `hue_source: Option<HomeHueSource>` (`Color`,
   `Muted`, `Hash`; serialized lowercase) and `root_hue: Option<Hue>` to `Home` in
   `mod.rs`. Compute `root_hue` with the same walk, starting at the root parent's chain
   index; for an area home it equals `hue`. Buckets set both to `None`. Update every
   `Home { … }` literal (stats `bucket_home`, tests).
3. **Root groups.** In `session.rs::root_groups`, take a root's hue from
   `group.home.root_hue` and fix the doc comment. `plan_budget/roots.rs` already
   resolves the root note itself. Check `capture_pomodoros_agenda.rs`, `gcal/root.rs`,
   `gcal/list.rs`, and `gcal/doctor.rs`: `None` already renders dim, so they need no
   change.
4. **Stats JSON and CLI.** Add `hue_source` to `HomeCatalogEntry` (`stats/report.rs`),
   omitted when `None`, and fill it in `stats/homes.rs::catalog_entry`. In
   `stats/render.rs`, paint a muted home's `●` with `styler.dim`. Reword the
   `area_color_invalid` warning to say the note inherits, and list grey among the valid
   values. Keep the code.
5. **Type hues.** `pomodoro_themes/scan.rs::root_area_hue` returns
   `Option<Option<Hue>>`, where `Some(None)` means a grey top area; an invalid value
   still hashes. `hue.rs::resolve_hue` takes `&BTreeMap<String, Option<Hue>>` and
   returns `(None, HueSource::Area)` for a muted area. In `defaults.rs`, give the cached
   `area_hue` the same type and bump `VERSION` from 1 to 2. That discards caches from
   older bob, which hashed `grey` as an invalid value. Update every area-map builder and
   caller the compiler finds.
6. **Review time.** In `stats/registry.rs`, the built-in `review_time` becomes
   `in: [gtd]`, with the description "Minutes attributed to tasks in, or rooted at, gtd,
   the GTD area that roots each daily ^gtd row; redefine stats.metrics.review_time if
   your review lives elsewhere." Update the registry assertion (`["gtd"]`). Recompute
   the `stats/compute.rs` M10 and `stats/homes.rs` H15 expectations from the fixtures
   rather than flipping strings: extra gtd-area time in a fixture legitimately changes a
   value. Leave the fixture vaults' own `gtd_daily` rows alone; they test the mechanism,
   not Bryan's template.
7. **Vectors.**
   - In `tests/fixtures/pomodoro_homes/vectors.json`, add new uniquely named notes and
     one case, `nearest-color-and-grey`. It covers:
     - a grey top-level area, where the parts, root, and chip all have `hue: null`;
     - a sub-area that inherits that grey;
     - a project with `color: red` under the grey area;
     - a project with `color: Green` under `dev`, whose child project inherits green;
     - a sub-project with its own color under a root project, where the chip still shows
       the root's own hue;
     - the `Gray` spelling;
     - an invalid project color that inherits and adds one `area_color_invalid`.

     Existing cases must keep passing unchanged.

   - In `tests/fixtures/pomodoro_themes/vectors.json` `hue`, add `area-grey-mutes` with
     an area map entry `null` and the expected `{"hue": null, "hue_source": "area"}`.
     Make the Rust vector loaders accept `null` area values.

8. **Docs** (routing from the `docs.md` memory: behavior goes in contracts):
   - `docs/pomodoro-homes.md`: rewrite R8 to the Design rules and add `grey` to the
     palette table. Fix the R7 root-hue step ("A root's hue equals each of its parts'
     hue…"). Add the new vector to the vector table. Change prose that describes Bryan's
     canonical daily row to `[root:: gtd]`, but leave the tables that describe the test
     fixtures.
   - `docs/stats.md`: document `homes.<key>.hue_source`; update the Day strip root-lane
     colour rule (muted roots stay gray, unshaded, never claim a hue, and sort after
     coloured roots); update the `review_time` row.
   - `docs/task-types.md`: PT5 step 5 and the `[root:: gtd_daily]` example.
   - `docs/reference/vault-format.md`: the daily `^gtd` row.
   - `docs/develop/migration-notes.md`: the built-in `review_time` row.

   Then run `just docs-gen` and `just docs-check`.

9. **Memory.**

   > [!decision] memory_root_parent
   >
   > Use `/sase_memory_write`. In the glossary strand `root-parent`, replace the
   > sentence quoted in the decision with: "Daily review time is the time attributed to
   > tasks in, or rooted at, `gtd`; each daily note's `^gtd` row carries
   > `[root:: gtd]`." Run `sase memory init`, then `just docs-gen`, which regenerates
   > `docs/reference/glossary.md`.

   > [!decision] memory_area_color_record
   >
   > Use `/sase_memory_write` to add the decisions strand
   > `area-color-follows-nearest-note`, in the format of the existing records.
   >
   > - **Applies to:** bob-cli homes and stats JSON, CLI dots, and bob-plugins chips and
   >   strips.
   > - **Claim:** the nearest area or project `color:` slot decides a home's hue; grey
   >   mutes it (`hue: null`, `hue_source: muted`); otherwise the top-level area stem
   >   hash applies; a root wears its own hue.
   > - **Why:** the three rejected alternatives above, plus the old top-level-only rule.
   > - **Cost:** clients need `hue_source` to tell grey from uncolored; the distinct-hue
   >   pass can still move an explicit color when a bigger root claims the same hue.
   > - **Reopens when:** the palette grows, or the strips stop reassigning hues.
   > - **Evidence:** this plan and the `cli` commit.
   >
   > Then run `sase memory init`.

10. **Gate.** Run `just check` (fmt, clippy, and tests), or at least `just fmt`,
    `just lint`, and `just test`, plus `just docs-check`. Commit as `feat(homes): …`.
    The JSON change is additive, so no `!`. Do not touch the `Cargo.toml` version or
    `CHANGELOG.md`.

## Phase `plugins`: bob-plugins

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Edit `src/` fragments only,
keep each fragment at 1000 lines or fewer, and never hand-edit the generated `main.js`.

1. **Vectors.** Copy the two vector files from bob-cli master, byte for byte, over every
   bob-plugins copy: `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/`,
   `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/`, and
   `scripts/fixtures/pomodoro_themes/`. Check them with `cmp`.
2. **Home-chip mirror** (`142-pomodoro-home-chips.js`). Replace `pomodoroHomeHueForArea`
   with the nearest-note walk from `homeIndex` outward over area and project chain
   items. Grey and gray become `null`. An invalid value warns `area_color_invalid` once
   per path and inherits. Return `root_hue` from `pomodoroHomeResolve`, walking from the
   root's index. `pomodoroHomeRootGroups` and the lead hue use the root's own hue
   instead of the first part's. Bucket parts stay `null`.
3. **Type-hue mirror.** `pomodoroThemesAreasMap` records a grey top-level area as
   `null`. `pomodoroThemeResolveHue` returns `{hue: null, hue_source: "area"}` when
   `areas` owns the key with a `null` value. Make sure theme pills and the Stats type
   lane render that as the muted gray, the same as the exempt case.
4. **Grey root lanes** (`239-plugin-stats-panel-models.js`). In
   `statsPanelDistinctHues`, a row whose catalog entry has `hue_source === "muted"` is
   set to `"gray"` up front, like the buckets, and never claims a hue. The
   `timeLaneRootSegments` tone, the period-strip lane and parts row, and the Stats page
   home rows then follow automatically. Confirm that muted roots sort after coloured
   ones and before `@unlinked`/`@unfiled`, and are not shaded.
5. **Tests.** Extend `scripts/test-ledger-tools-day-strip-lanes.cjs`,
   `test-ledger-tools-stats-lanes.cjs`, and `test-ledger-tools-period-strip.cjs` with a
   report whose catalog marks one non-bucket root
   `{"hue": null, "hue_source": "muted"}`. Add the same to a stats fixture if the tests
   use one. Also extend the pomodoro-home-chips and pomodoro-themes vector tests if they
   do not pick up new cases automatically.
6. **Docs.** Update `docs/bob-ledger-tools.md` wherever it states the root-lane or
   area-hue rules, then run `npm run docs:check`.
7. **Gate and deploy.** Run `npm run build`, `npm test`, and `npm run validate`, then
   commit. Run `bob plugins sync`, which the bob-plugins `AGENTS.md` requires. Bob Mac
   Capture needs no change: it decodes `hue` as optional, and the new JSON key is
   additive.

## Phase `vault`: adoption and live verification

1. **Install bob.** Bring the bob-cli workspace to master, which now includes `cli`, and
   run `just install` (no `--force`). Confirm that `bob stats show worked -r 1d -f json`
   has `hue_source` in its `homes` catalog. Confirm with `bob plugins sync` that the
   `plugins` phase is deployed. apollo and the MacBook pick up the new bob on their
   usual update path. Until then they show gtd with a hash hue and one
   `area_color_invalid` warning; note that in the summary.
2. **Template.** In `~/bob/_templates/daily.md`, change the `^gtd` row to
   `[root:: gtd]`.

   > [!decision] gtd_link = no
   >
   > Keep `[[gtd_daily]]` as the row's link.

   > [!decision] gtd_link
   >
   > Also change the link to `[[gtd]]`.

3. **Color.** In `~/bob/gtd.md` frontmatter, change `color: orange` to `color: grey`.
   `gtd_daily.md`, `inbox.md`, and the other notes under gtd then inherit grey.
4. **Existing daily notes.** Edit only each daily note's `^gtd` row. Replace
   `[root:: gtd_daily]` with `[root:: gtd]`, and change the link too only if `gtd_link`
   is yes. Touch nothing else on the line or in the note.

   > [!decision] history = recent
   >
   > Rewrite `~/bob/2026/YYYYMMDD.md` notes dated 2026-11-07 or later.

   > [!decision] history = template
   >
   > Rewrite no existing notes.

   > [!decision] history = all
   >
   > Rewrite every `~/bob/<year>/YYYYMMDD.md` daily note whose `^gtd` row carries
   > `[root:: gtd_daily]` (about 143). Leave `done/`, `_conflicts/`, and `.bob/`
   > untouched.

   Let `bob-vault-sync.service` carry the edits, or run `bob vault-sync` once.

5. **Verify.**
   - `bob stats show worked -r 1d -f json`: `homes["gtd.md"]` has `hue: null` and
     `hue_source: "muted"`; today's `root_parent` parts list `gtd.md` (not
     `gtd_daily.md`) when today was rewritten; there is no `area_color_invalid` warning.
   - `bob stats show review_time -r 7d` still shows the review minutes.
   - `bob types list` still reports PLAN as exempt-muted.

   In the summary, report what Bryan should see: a grey `gtd` segment in the Day and
   period strips' root lane, matching PLAN's grey in the type lane.
