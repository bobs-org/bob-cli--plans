---
tier: tale
title: 'Gate ^prj review on #hide, fix scheduled-project surfacing, add a REFERENCES
  review group'
goal: '`bob projects sync` unhides an empty, non-future-scheduled project''s ^prj
  (so sase_sites.md surfaces); the freshness walk reviews ^prj only when it has no
  #hide and reviews every ^ref (hidden or not) on a 7-day cadence in its own REFERENCES
  group right before ROTTEN, identically in `bob freshness` and the Obsidian `]s`
  walk.'
size: medium
proposed_by: bbugyi200.athena.0vp
status: done
---

# Gate `^prj` review on `#hide`, fix scheduled-project surfacing, and give `^ref` its own review group

## Outcome

Bryan asked for three corrections to how `^prj` and `^ref` trackers are handled:

1. **Simpler `^prj` review eligibility.** The freshness walk reviews a `^prj` task only
   when it does **not** carry `#hide`. `bob projects sync` already owns that tag: it
   removes `#hide` when a project has no unhidden open tasks and no open sub-projects,
   and adds it back otherwise. The evaluator stops counting open tasks in the project
   note and stops reading the note's `scheduled` frontmatter. `^ref` tasks are still
   reviewed whether or not they carry `#hide`.
2. **`bob projects sync` bug.** A project whose frontmatter `scheduled` date is today or
   in the past must follow the normal surfacing rule. Today the sync skips that rule for
   any project with a `scheduled` key, so `~/bob/sase_sites.md` (scheduled 2026-08-08,
   only closed tasks, hidden `^prj`) keeps its `#hide` forever. A project scheduled in
   the future stays hidden, as it does now.
3. **References.** Bryan's chezmoi config moves `freshness.reference_interval` from 3 to
   **7** days. The `]s` walk also reviews references as their own **REFERENCES** group,
   right before ROTTEN. The walk becomes **NEW → PROJECTS → PENDING → NEXT → RETURNED →
   REFERENCES → ROTTEN**.

This is one medium tale. The Rust evaluator and its JavaScript mirror must change
together under the same contract. The sync fix is small and is what makes the new `^prj`
rule correct. The chezmoi edit is one line. Splitting these by language would ship a
contract mismatch between `bob freshness` and the `]s` walk.

## Planning-time facts (re-verify; do not treat as a fixed mutation list)

- `bob projects sync --dry-run` on the live vault (installed binary) reports
  `0 ^prj edited`. `sase_sites.md` is the failing shape: `scheduled: 2026-08-08`, one
  open `^prj` with `#hide`, and one `[x]` task. In `src/native/projects/sync.rs`
  (`plan_project_sync_at`), the surfacing block runs only when
  `project.scheduled.is_none()`. `TaskSchedulePolicy::for_schedule` (`model.rs`) touches
  the `^prj` hide only when the date is in the future **or** `task_lines.len() == 1`. A
  closed task therefore turns off the "sole ^prj" unhide. The unit test
  `scheduled_tasks_precede_prj_surfacing_at_local_date_boundary` in
  `src/native/projects/tests/edits.rs` (the `Due.md` case) encodes this bug and must be
  changed.
- Other projects with past/today schedules and hidden `^prj` (for example
  `sase_better_installs`, `sase_polish_proc_lanes`, `sase_more_agent_clis`) still have
  unhidden open or `[?]` tasks, so they must stay hidden. `sase_research.md` is
  scheduled 2026-10-05 (future) and must stay hidden. Cron runs
  `~/.cargo/bin/bob projects sync` every 15 minutes (`docs/vault-git-sync.md`), so the
  fix goes live only after the binary is reinstalled.
- All 13 open `^ref` tasks in the vault are Pending `[/]`, never stamped. None are
  Ready. `docs/highlights-ref-sync.md` generates new references as Ready
  `- [ ] #task #ref [[…pdf]] #hide ^ref`. REFERENCES must therefore take lane rows as
  well as Ready rows.
- Visible open `^prj` today: `sase_decks.md` and `sase_message_boards.md`. The current
  PROJECTS rows `sase_blog.md` and `project.md` come from the hide bypass. `sase_blog`
  is hidden because of an open sub-project. `project.md` is the `type` documentation
  note, which contains an example `^prj` line. Both leave the walk under the new rule,
  as intended.
- The chezmoi source (`home/dot_config/bob/config.yml`) has `project_interval: 1` and
  `reference_interval: 3` (commit `855a98e`). That commit has not been applied to
  `~/.config/bob/config.yml` yet. The chezmoi repo's `sase/sase.yml` commit hook runs
  `chezmoi update -a --force`, which applies both.
- The CLI human header and the plugin status bar mix walk-tier counts (`projects_due`,
  `pending_due`, `next_due`) with Ready-state totals (`new`, `resurfaced`, `rotten`). A
  Ready tracker therefore shows up twice: once in its tracker tier and again in a state
  count. Adding REFERENCES would make this worse, so both surfaces switch to `by_tier`
  (below).

## Repositories

- **bob-cli** (this repo): `src/native/projects/{model,scan,sync,edits,output}.rs` and
  `src/native/projects/tests/`, `tests/cli/projects/schedule.rs`;
  `src/native/freshness/{state,scan,cli}.rs`, `state_tests.rs`,
  `tests/cli/freshness.rs`; docs `docs/freshness.md`, `docs/projects.md`,
  `docs/plan.md`, `docs/highlights-ref-sync.md`.
- **bob-plugins** (linked): open with `/sase_repo` (`sase repo open bob-plugins -r …`),
  read its `AGENTS.md`, and use only the printed path. Files:
  `plugins/bob-ledger-tools/main.js`, `plugins/bob-navigation-hotkeys/main.js`, their
  `manifest.json`, `README.md`, and the tests
  `scripts/test-ledger-tools-freshness*.cjs`, `scripts/test-navigation-freshness.cjs`,
  `scripts/test-navigation-keep-counting.cjs` (plus any other suite that asserts six
  tiers or `projectOpenCount`).
- **chezmoi** (linked): open with `/sase_repo`; edit `home/dot_config/bob/config.yml`.

## Decisions (made while planning)

1. **`^prj` eligibility is `#hide` only.** An exact `^prj` uses the ordinary
   lane-visible predicate with no `#hide` exemption. Hidden `^prj` rows are out of
   scope: null state/bucket/tier, the same as any hidden task. Remove project occupancy
   (`project_open_count`), the frontmatter schedule gate and resurfacing
   (`project_scheduled`, `project_schedule_invalid`), and the
   `project_scheduled_invalid` lint from **both** evaluators. A visible `^prj` keeps
   everything else it has now: the PROJECTS tier (any lane, actual lane retained, null
   state for lane rows), the `project_interval` override, otherwise the Ready interval
   chain in every lane, ordinary inline-`scheduled` RESURFACED handling for Ready rows,
   and never deciding. If a visible `^prj` sits in a note that has open tasks because
   sync hasn't run yet, it is reviewed. That is accepted, because sync runs every 15
   minutes.
2. **`^ref` keeps the `#hide` bypass.** Every other exclusion still applies: closed or
   unsupported status, dependency-blocked, recurring, `_templates`/`_conflicts`,
   canonical daily note, Today-linked, inline future `scheduled`.
3. **REFERENCES tier.** Every due in-scope `^ref`, in any lane, walks in `references`
   between `returned` and `rotten`. That includes never-confirmed Ready references,
   which no longer enter NEW. Lane rows keep their actual lane and a null state/bucket,
   like lane `^prj`. Ready references keep their Ready `state`/`bucket` unchanged (NEW,
   RESURFACED, ROTTEN, FRESH); only the tier changes.
   - Cadence is `reference_interval` when set, otherwise the Ready chain (task `refresh`
     → note `task_refresh` → `freshness.interval` → 7). It is **never** the lane
     interval. A lane reference is due when it has never been stamped or when
     `today ≥ fresh + interval`.
   - `pending_interval: false` / `next_interval: false` no longer disable reference
     review. This now matches PROJECTS.
   - Order within the tier is the same as PROJECTS: never-confirmed first, then `due_on`
     ↑, `created` ↑, path ↑, line ↑.
4. **REFERENCES is a commitment tier.** "Commitments" becomes NEW, PROJECTS, PENDING,
   NEXT, RETURNED, and REFERENCES. The "Commitments done — N ROTTEN left" / "ROTTEN next
   — N commitments still due" boundary stays at the ROTTEN edge, which is exactly "right
   before ROTTEN". The CLI divider `── commitments done above · upkeep below ──` stays
   before ROTTEN. The status-bar "due" mode covers outstanding references.
5. **Tracker tiers never decide and never count keeps.** The existing exact-eligibility
   rule (`lane === 'ready'` and `tier in {'rotten','returned'}`) already excludes
   `references`. Keep that code and document that REFERENCES behaves like PROJECTS: an
   explicit keep stamps uncounted and preserves the streak. In practice nothing is lost,
   because no Ready references exist and decay activates on 2026-10-19.
6. **Header and status bar use tier counts.** The `bob freshness` human header and the
   ledger-tools status-bar text, tooltip, and mode read the `by_tier` histogram, so the
   shown numbers sum to `walk`. The status bar keeps folding RETURNED into its "rotten"
   number, as before. JSON state totals (`due`, `new`, `resurfaced`, `rotten`, `fresh`)
   and the raw `budget_met` formula are unchanged. The plugin status view falls back to
   the old state counts when `byTier` is absent.
7. **Sync surfacing for scheduled projects.** For a non-terminal project with an open
   `^prj`:
   - **future** `scheduled` → force exactly one `#hide` (unchanged);
   - **today/past** `scheduled` or **no** `scheduled` → the normal rule: remove `#hide`
     when there are no unhidden open tasks and no open sub-projects, add it back
     otherwise.

   The past-date "sole ^prj" special case disappears into the normal rule.

8. **Chezmoi.** Set an explicit `reference_interval: 7` (the user asked for "every 7
   days", matching the 7-day Ready interval). Leave `project_interval: 1` alone.

## Work

### A. `bob projects sync` fix (bob-cli)

- `model.rs`: `TaskSchedulePolicy::include_prj` becomes `future` only. Drop the
  `task_count == 1` clause and, if it becomes unused, the `task_count` parameter of
  `for_schedule`. `prj_hide_needs_change` therefore acts only on future dates.
- `sync.rs` (`plan_project_sync_at`): run the surfacing block when
  `project.scheduled.is_none_or(|s| s.date <= today)`, not only when unscheduled.
- **Use the post-reconcile unhidden count.** In a scheduled project,
  `ReconcileTaskSchedules` strips `#hide` from ordinary tasks in the same run
  (`ordinary_hide_needs_removal`). `open_unhidden_count` is computed before that runs,
  so an open ordinary task that is hidden now but will be unhidden this run must count
  as unhidden. Otherwise the `^prj` would surface while a newly visible task exists.
  Implementation sketch: record on `ProjectTaskLine` whether the line is an open `#task`
  (the same predicate `scan.rs` uses for `open_task_count`). When a schedule policy
  applies, compute the effective count as open non-`^prj` lines where
  `hide_tag_count == 0 || policy.ordinary_hide_needs_removal(task)`. Otherwise keep
  `project.open_unhidden_count`.
- `output.rs`: remove the now-dead `"#hide from sole ^prj"` branch. Past-date unhiding
  is reported through the ordinary `RemoveHideTag` event
  (`no non-hidden open tasks or open sub-projects`).
- Make sure `RemoveHideTag`/`AddHideTag` and `ReconcileTaskSchedules` never both edit
  the `^prj` hide in one plan, and that a second run is a no-op.

### B. Freshness contract in Rust (bob-cli)

- `state.rs`:
  - Add `Tier::References` between `Returned` and `Rotten` (`as_str` = `"references"`).
  - Remove `project_open_count`, `project_scheduled`, and `project_schedule_invalid`
    from `FreshnessRow`. Remove `is_open` and the
    `is_open_project_row`/`project_open_count` helpers too, if they become dead; let
    clippy decide.
  - Rewrite the tracker branch of `evaluate` per decisions 1, 3, and 5:
    - interval: tracker override, then (any tracker) the Ready chain, then the lane
      interval for ordinary lane tasks, then the Ready chain;
    - tracker lane due/`due_on`/`days_overdue`: same as the current lane-`^prj`
      arithmetic, without the occupancy/schedule gates;
    - tier: PROJECTS for due `^prj`, REFERENCES for due `^ref`, both checked before NEW
      and the lane tiers;
    - `^ref` is excluded from the PENDING/NEXT lane tiers, like `^prj`.
  - Extend `queue` (REFERENCES uses the PROJECTS comparator), `ByTier` (`references`,
    included in `sum`), and `Counts` (`references_due` = `by_tier.references`).
  - Keep `decide_for` as is.
- `scan.rs`: delete the per-path open-count pass, the schedule cache,
  `fill_project_context`, and `parse_project_scheduled`. Restrict the hidden tracker
  candidate list to exact `^ref` rows: hidden `^prj` rows must no longer enter. Update
  the `RowCtx`/`Snapshot` docs. Remove `project_scheduled_invalid` from `lint_message`.
- `cli.rs`: bump `SCHEMA_VERSION` 6 → 7, with a doc comment explaining what changed: the
  `references` tier, the seven-key `by_tier`, `counts.references_due`, the removed
  `project_scheduled_invalid` lint, and the `^prj` hide gate. The seed envelope shares
  the constant; seed behavior is unchanged.
  - Update the tier order in `long_about` strings.
  - Header:
    `REVIEW {walk} due - {new} new - {projects} projects - {pending} pending - {next} next - {returned} returned - {references} references - {rotten} rotten - ✓ …`,
    with all counts taken from `by_tier`.
  - Add `("references", "REFERENCES")` to the tier list; count it on the commitment side
    of the divider.
  - Add a `references` row detail: `reference - never confirmed - created …` or
    `reference - {lead} - fresh {fresh} - every Nd (source)`, using the same `lead`
    logic as projects.
  - Change the PROJECTS detail lead from `No open tasks in this project` to
    `Empty project`. The evaluator no longer inspects occupancy; sync surfaced it.
  - JSON: add `references_due` and `by_tier.references`.

### C. JavaScript mirror (bob-plugins)

- `bob-ledger-tools/main.js`:
  - `freshnessEvaluate`: mirror the Rust changes exactly, and drop the
    `projectOpenCount`/`projectReadyCount`/`projectScheduled`/`projectScheduleInvalid`
    inputs.
  - `freshnessIntervalForLine`: lane `^ref` rows without a configured
    `reference_interval` show the Ready chain, not the lane interval, so the nav refresh
    row agrees with the queue.
  - `FRESHNESS_TIER_ORDER`: add `references: 5`, `rotten: 6`. `freshnessTierLabel`:
    `"REFERENCES"`. `freshnessQueue` comparator: `references` with
    projects/pending/next.
  - `freshnessCounts`: `byTier.references`, `referencesDue`, `walk`.
  - `freshnessStatusView`:
    - text: `⟳ N new · N projects · N pending · N next · N references · N rotten · ✓ …`;
    - tooltip: add `· REFERENCES n` after RETURNED;
    - counts come from `byTier` when present (rotten = returned + rotten), otherwise
      from the legacy state counts;
    - mode: `new` when the NEW tier > 0; `due` while any commitment tier, including
      references, remains.
  - `freshnessRowFromTask`: apply the hide-stripping re-check only for
    `tracker === "ref"`, and remove the project-occupancy/schedule block.
  - Memo/context: remove `projectOpenCountFor`, `projectReadyCountFor`,
    `projectScheduledFor`, `freshnessProjectOpenCounts`, `freshnessProjectReadyCounts`,
    and `freshnessProjectScheduledFor` where they become unused. Check note-ready and
    dashboard callers first; keep anything they still use. Remove any memo invalidation
    that existed only for note schedules.
  - Marks: `freshnessMarkReason` loses its project-occupancy/schedule branch.
    `freshnessMarkResolution` treats a NEW-state row in either tracker tier
    (`projects`/`references`) as resolved, not as unresolved NEW.
  - Lints: remove `project_scheduled_invalid` from `FRESHNESS_LINT_MESSAGES`.
  - Default/unavailable count objects (around the `projectsDue: 0` / `byTier` literals):
    add `references`/`referencesDue`.
  - Update the api comment: freshness namespace stays v5 and `trackerReview` stays true.
    Add an explicit `referenceReview: true` capability on `api.freshness` that tells
    consumers the queue may carry `references` entries. No namespace bump.
- `bob-navigation-hotkeys/main.js`:
  - `reviewEntryMachineTier` recognizes `references`, and `reviewIsCommitmentTier`
    includes it. Update the comment above `reviewWalkRemaining`.
  - `buildReviewJumpNotice`: add a `references` detail mirroring PROJECTS, i.e.
    `Reference · never confirmed · every Nd` or
    `Reference · {due today | Nd overdue | due <date>} · confirmed <date> · every Nd`.
    Change the PROJECTS lead to `Empty project`.
  - Keep `matchFreshStampExactEntry` unchanged; it already returns `reason: "tier"` for
    references. Add a test that proves it.
- Bump `bob-ledger-tools` 1.25.0 → 1.26.0 and `bob-navigation-hotkeys` 1.70.0 → 1.71.0
  (the manifests). Update the README table rows (versions, the tier order, the
  `referenceReview` capability) and the README `freshness` api line.

### D. Docs (bob-cli)

- `docs/freshness.md`, updated throughout to the seven-tier order:
  - the intro;
  - §2: the interval-precedence text (lane references no longer use the lane interval;
    the disabled-lane exception is gone) and the example comment
    `# reference_interval: 7`;
  - §2a: the event table and the `decide` note covering both tracker tiers;
  - §4: pseudo-code (`in_scope` hide bypass for `^ref` only, `interval`, `lane_due`,
    `due_on`, `tier`), the per-tier order table, the commitment-tier sentence, counts
    (`references_due`, seven `by_tier` keys), "Tracking review" rewritten for the hide
    gate and REFERENCES, the PR/RF conformance paragraph, and machine vocabulary schema
    7;
  - the lint list (drop `project_scheduled_invalid`);
  - §6 review ritual: REFERENCES before "Commitments done"; "confirm the reference still
    needs reading" moves to REFERENCES.
- `docs/projects.md`: the freshness-tracker paragraph after the `^prj` example (review
  is gated by sync's `#hide`, not a separate predicate). In the sync rules, a valid
  `scheduled` date now overrides surfacing only when it is in the future. Rewrite the
  `^prj` bullet and the `^prj, P due/past` table row as "normal surfacing rule".
- `docs/plan.md` (Lanes paragraph) and `docs/highlights-ref-sync.md` (the tracker
  paragraph: an unstamped reference walks in REFERENCES, not NEW, on the reference
  cadence in any lane).

### E. Chezmoi

- `home/dot_config/bob/config.yml`: `reference_interval: 7`, keeping the existing
  personal-setting comment style.
- Verify before committing: run the new build with
  `BOB_CONFIG_FILE=<opened chezmoi path>/home/dot_config/bob/config.yml bob freshness list`
  and confirm references show `every 7d (reference)`.
- The repo's commit hook (`chezmoi update -a --force`) applies the change, together with
  the pending `855a98e`, to `~/.config/bob/config.yml` when the implementer's final
  declaration commits the chezmoi repo.

## Tests

- **Projects (unit, `src/native/projects/tests/`):**
  - the `sase_sites` shape (past date, `^prj #hide`, only closed tasks) →
    `RemoveHideTag`;
  - the same shape dated today → `RemoveHideTag`;
  - future date → forced `#hide`, no surfacing change;
  - past date with an unhidden open or `[?]` task → stays or becomes hidden
    (`AddHideTag` when visible);
  - past date where the only open task is hidden but reconcile strips its `#hide` →
    `^prj` stays hidden;
  - past date with an open sub-project → hidden;
  - the old sole-`^prj` case still unhides, through `RemoveHideTag`;
  - plan idempotence;
  - update `scheduled_tasks_precede_prj_surfacing_at_local_date_boundary` to match.
- **Projects (CLI, `tests/cli/projects/schedule.rs`):** update
  `projects_sync_shows_sole_prj_task_when_schedule_is_due` for the new event text. Add
  an end-to-end `sase_sites`-shaped case with `BOB_NOW`: dry run, then apply, then a
  second run that changes nothing.
- **Freshness state (`state_tests.rs`):**
  - visible `^prj` NEW/stamped/rotten in Ready and lane, with and without
    `project_interval`;
  - hidden `^prj` → out of scope;
  - a visible `^prj` in a note with open tasks or a future frontmatter schedule is still
    reviewed, with no lint;
  - hidden Ready `^ref` NEW → `references` (state `new`, bucket `new`);
  - Pending `^ref` → `references` with null state, never `pending`;
  - lane `^ref` cadence: `reference_interval` 7 (stamped 6 days ago is not due, 7 days
    ago is due), and without the key the Ready chain applies rather than the lane's 1;
  - `pending_interval: false` still reviews lane references;
  - a RESURFACED Ready `^ref` → `references`, and `decide` is false at the keep limit
    after activation;
  - seven-tier ordering with stable ties, `walk = sum(by_tier)`, `references_due`.
  - Replace the occupancy tests, which are now obsolete.
- **Freshness CLI (`tests/cli/freshness.rs`):** schema 7, `by_tier.references`,
  `references_due`, a human header whose tier numbers sum to `walk`, the REFERENCES
  section above the divider, hidden `^prj` absent, and visible `^prj` present. Rework
  `list_tracker_intervals_and_open_occupancy` and
  `list_walks_projects_after_new_with_decoupled_counts`.
- **Plugins:** mirror the same vectors in `test-ledger-tools-freshness.cjs` (evaluate,
  queue, counts, status view text/tooltip/mode with and without `byTier`, the row
  adapter's hide bypass for `ref` only, mark resolution). In
  `test-navigation-freshness.cjs`, cover tier recognition, the commitment count
  including references, the boundary notice when stepping REFERENCES → ROTTEN, and the
  references/projects notice text. In the keep-counting suite, show that a
  references-tier entry is refused with `tier`. Update any suite that pins six tiers or
  project occupancy.

## Verification and deployment

1. bob-cli: `just all` (fmt, clippy, tests). bob-plugins: `npm test` and
   `npm run validate`.
2. Plugins: from the opened bob-plugins path, run
   `bob plugins sync -p bob-ledger-tools -r "$PWD" --dry-run`, then without `--dry-run`,
   and the same for `bob-navigation-hotkeys`. If a file is reported "dirty in vault",
   check the on-disk file against the vault baseline before considering `--force`. Tell
   Bryan to reload both plugins.
3. bob-cli: `just install`, so the 15-minute cron and `bob freshness` use the fix.
4. Vault: `bob projects sync --dry-run`, then `bob projects sync`. Confirm
   `sase_sites.md`'s `^prj` loses `#hide`. Confirm that every other reported edit
   follows the new rule (past/today-scheduled, empty, no open sub-projects) and that
   future-scheduled projects stay hidden. Report every changed file.
5. `bob freshness list` (with the chezmoi-source config as in E, if the hook hasn't run
   yet):
   - `sase_sites.md` is in PROJECTS;
   - the 13 Pending references are in REFERENCES, every 7d;
   - `sase_blog.md` and `project.md` are no longer listed;
   - the header numbers sum to `walk`.
6. Commit bob-cli, bob-plugins, and chezmoi through the final declaration. The vault
   edits from step 4 are ordinary sync output that vault-sync carries.

## Out of scope

- Changing what "unhidden open tasks" means in `bob projects sync` (hidden ordinary
  tasks still do not keep `^prj` hidden), `project_interval`, decay/keep rules for
  ordinary tasks, dashboard buckets and chips, the note-ready cap, and highlights-ref
  generation.
- Canonical SASE memory. The decision record `decisions:review-walk-is-tiered` (and its
  roster line) still lists NEW → PENDING → NEXT → RETURNED → ROTTEN, and it already
  predated the PROJECTS tier. Do not edit it under this plan. Instead, file a `memory`
  task bead through `/sase_new_task` proposing a new decision record that partly
  supersedes it for the tracker tiers (PROJECTS gated by sync's `#hide`; REFERENCES
  before ROTTEN).
