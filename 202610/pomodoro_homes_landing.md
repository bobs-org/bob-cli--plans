---
tier: epic
title: Finish Pomodoro homes — contract parity, review time, override labels, and
  panel fixes
goal: 'Close the gaps the bob-cli-9k landing review found between the shipped Pomodoro
  homes feature and its approved plan (plan:202610/pomodoro_homes.md). Rust and the
  bob-ledger-tools mirror agree on every shared vector. The review_time tile always
  equals the GTD review row. Stats theme parts are unchanged when no home filter applies.
  `bob ref today`, human `bob plan`, and the `=` override surfaces that landed during
  the epic label unnamed sessions by their derived homes. The Stats panel follows
  chart pins. Bob Mac Capture never routes by a derived label and draws forecast dots
  hollow.

  '
parent_bead: bob-cli-9k
decisions:
  review_row_time:
    ask: Count a work session's linked daily ^gtd share as review time, so the review_time
      tile always equals the GTD review row?
    default: true
    why: You asked for exactly the GTD review row's minutes; 4w today is row 1729m
      vs tile 1660m
    answer: true
phases:
- id: contract
  title: 'bob-cli: homes contract parity, review time, and Stats fixes'
  depends_on: []
  size: medium
  description: 'contract: give the R5 remainder to the first task, mark every closed
    entry attributed, re-split the shared vectors so only the ambiguous-basename case
    is rust_only and add the missing cases, put review_time on the GTD review row
    (per decision), stop re-splitting theme parts when no home filter drops a task,
    prune the homes catalog to used keys, align `bob projects list`, print the R10
    caveat and review hue in `bob stats show`, delete the stale `allow(unused)`, fill
    the H-vector gaps, and update docs/pomodoro-homes.md and docs/stats.md.

    '
- id: surfaces
  title: 'bob-cli: homes on bob ref today, human bob plan, and the = override surfaces'
  depends_on:
  - contract
  size: medium
  description: 'surfaces: add the R9 `homes` object to `bob ref today -f json` Pomodoro
    rows and use the derived label in its human output, print the dim chip for unnamed
    entries in human `bob plan`, and add `homes` to the bob-cli-9j override JSON (`override.previous`,
    `override.demoted`, capture-complete `override.running`) with the chip in the
    human warning, demoted, and `--refuse-override` lines for unnamed sessions. Update
    docs and tests.

    '
- id: plugins
  title: 'bob-plugins: JS mirror parity and Stats panel fixes'
  depends_on:
  - contract
  size: medium
  description: 'plugins: copy the updated vectors, bring the bob-ledger-tools homes
    mirror to parity with Rust (share rounding, lead home, any-day ^gtd home, review
    label, `[t::]` minutes, frontmatter shapes, closed-entry state, health), keep
    chips to Live Preview and Reading view with embed support, make chart pins refresh
    the Where the time went panel, fix the Themes pinned span, review hue, count/task
    marks, saved view choice, fix-it copy, and drill-down labels, update the README
    table, then test, validate, and sync.

    '
- id: mac
  title: 'Bob Mac Capture and chezmoi: no derived routing, hollow forecast dots, override
    labels'
  depends_on:
  - surfaces
  size: medium
  description: 'mac: stop Refs Today Link from selecting or routing by `homes.label`,
    name the derived label in its toast, draw forecast dots hollow, add the Later
    strip "for", fix the HACK fixture, show the derived label in the override banner
    from `override.previous.homes`, refresh fixtures from real bob output, compile
    on the Mac and check CI; in chezmoi add the unnamed-derived tmux status sample.

    '
proposed_by: bbugyi200.athena.bob-cli-9k.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 20:20:11
status: wip
bead_id: bob-cli-9k.8
---

- **PROMPT:** [prompts/202610/pomodoro_homes_landing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/pomodoro_homes_landing.md)
- **PARENT:** [202610/pomodoro_homes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/pomodoro_homes.md)
- **BEAD:** [bob-cli-9k.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9k/bob-cli-9k.8.md)

# Context

Epic bob-cli-9k shipped derived Pomodoro homes in bob-cli (commits 9ec332c, 196af9b,
81103c2, 930b349, 1933d54), bob-plugins (0ffb3e0 ledger chips, fdc4528 Stats panel), Bob
Mac Capture (e5cd4e1), and chezmoi (6259f0b). The parent plan is
plan:202610/pomodoro_homes.md. Its normative rules R1–R10, its DECISIONS (review_bridge
= yes, review_time_predicate = yes, memory_mac_pom = yes, memory_decision_record = no,
memory_home_term = no), and its cross-cutting rules stay the contract for everything
below unless this plan says otherwise. The normative bob-cli doc is
`docs/pomodoro-homes.md`.

The landing review confirmed that the feature works end to end.

- bob-cli `just check` is green: fmt, the clippy gate, and 4998 tests.
- `bob capture-pomodoros --tasks` reproduces the research example: BLOG gets `sase` and
  HACK gets `bob · dev +1`.
- Over the live 4-week window, every Stats dimension sums to its total.
- The bob-plugins vectors are byte-identical to bob-cli's, the homes and Stats tests
  pass, and the vault plugin copy matches the build at 1.57.0.
- Mac `just portable-check` passes, and the Swift palette matches R8 exactly.

The review also found these deviations, all introduced by bob-cli-9k:

1. **R5 remainder.** `split_minutes` (`src/native/pomodoro_homes/session.rs`) gives one
   leftover minute to each of the first tasks: 8m over 3 tasks gives 3/3/2. R5 says the
   whole remainder goes to the first task (4/2/2), and so do the theme split
   (`theme_minute_parts` in `src/native/stats/metrics.rs`) and the JS mirror. The doc's
   R5 was reworded to match the code, and its R4 sentence ("remainder going to the first
   component") contradicts it.
2. **Over-marked rust_only vectors.** `tests/fixtures/pomodoro_homes/vectors.json` puts
   the done/ archive, ref→sub-project, closed project, cycle, `[[love.md|love]]`,
   `rollup: yes please`, invalid `color:`, and area-resident cases into the same
   `rust_only` case as the ambiguous basename. The JS mirror therefore skips most
   resolution, lint, and hue vectors. The vectors also never cover:
   - an unresolved link;
   - a list-form or self parent;
   - the depth cap;
   - a direct rolled-up child;
   - an uneven three-task split;
   - share rounding;
   - a `[t::]` duration that differs from the range;
   - a `^gtd` link to another day's note;
   - a closed entry with no minutes.
3. **The review_time tile disagrees with the GTD review row.** On the live 4-week window
   `review_time` is 1660m, but the GTD review row is 1729m. The 69m gap is `^gtd` links
   inside work sessions, which R1.4 homes to `@review:GTD`. The accepted decision asked
   for "exactly the GTD review row's minutes" so that "the tile and the row always
   agree". `review: only` selects whole R4 sessions only, so the two never agree.
4. **A closed entry without minutes is a forecast.** `state` comes from whether minutes
   are present (`session.rs` around line 117, `capture_pomodoros.rs` around 413,
   `stats/homes.rs` around 182). So `- [x] ()` gets `state: forecast` and a hollow dot.
   This happens live: 2026-06-04 line 52 in `bob stats -f json -r all`.
5. **Theme parts are re-split without a home filter.** Every work session takes the
   kept-tasks path (`metrics.rs` around line 312), which re-splits theme minutes even
   when no `in:`/`exclude_in` filter applies. Example: `A + B + C` at 50m with
   `themes: [A, C]` now gives 17/17 instead of the pre-epic 18/16, and linked and
   unlinked sessions now split differently.
6. **The catalog holds unused keys.** It adds every vault project and area, so 13 of the
   31 live entries are used nowhere in the report. The plan keys it by keys used.
7. **CLI polish.**
   - The `rollup` tag in `bob projects list` (`src/native/projects/output.rs`) is
     inserted after the padded name, which shifts STATUS right by 7 columns.
   - `bob stats show` never prints R10's caveat.
   - `GTD review` prints dim with no dot, though review takes its area's hue.
8. **Hygiene.**
   - `src/native/pomodoro_homes/mod.rs` still carries `#![allow(unused)]`, and its
     comment says callers are still to land.
   - `NoteFacts.daily` is dead.
   - Rollup is parsed twice (`facts.rs` `parse_rollup` and `resolve.rs` `rollup_value`).
   - The sessions and done H-vectors check sum-to-total only in the `project` dimension,
     and no H-vector covers count mode with `in:`.
9. **Refs Today never receives homes.** Mac's `RefsTodayLinkPomodoro` decodes `homes`
   from `bob ref today -f json`, but bob-cli's `TodayPomodoroRow`
   (`src/native/capture/existing_task_link.rs`) has no such field. Unnamed rows
   therefore always show "Untitled Pomodoro", and the human output prints `(untitled)`
   (`src/native/ref_library/today.rs` around 638). The parent plan wrongly assumed these
   rows come from `bob plan -f json`.
10. **Human `bob plan` has no chip.** It prints `★ (unnamed)` with no chip
    (`src/native/plan_budget/cli.rs` around 403), while `docs/plan.md` promises that "an
    unnamed entry still shows its derived homes".
11. **Override surfaces show no homes.** The `=` override surfaces landed during this
    epic (bob-cli-9j: 5ec1390, b9788c4, Mac 4c391eb). For an unnamed running session
    they say only "the running session":
    - the human warning `⚠ overrides the running session 0920-0945 …`;
    - the swap demoted line `session → first future …`;
    - the `--refuse-override` error;
    - `override.previous` / `override.demoted` and capture-complete `override.running`,
      which carry no label.

    Their whole job is to say which session gets replaced. Integrating homes there is
    part of this epic's integration work.

12. **JS mirror drift** (`plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js`).
    Labels, chips, hues, and home keys matched Rust on all 269 live 4-week sessions, but
    minutes or shares differed on 15. The mirror:
    - rounds shares by giving the leftover percent to the last part, where Rust uses the
      largest fractional remainder;
    - picks the lead home by minutes, not task count;
    - homes only same-day `^gtd` links to review;
    - hardcodes `GTD review`;
    - ignores `[t:: Nm]`;
    - reads only string `type`, exact-case `rollup`, and string `parent`;
    - leaves ambiguous and unresolved links out of `unfiled_links`.
13. **Ledger chip view gaps.** Chips also draw in Source mode. Reading view reads only
    `data-href`/`href`, so embeds may be skipped. Reading view chips were never checked
    live.
14. **Stats panel bugs** (bob-plugins).
    - Clicking or pressing Enter on a Worked chart bar (`statsPinChartPoint` in
      `239-plugin-stats-chart-render.js`) never refreshes "Where the time went".
    - When a day is pinned, the Themes view keeps the whole-range rows.
    - `@review:GTD` rows are gray.
    - Count and task metrics drop the hue dots and mini-bars, though only the tint
      should go.
    - A report without homes overwrites the saved "areas" view choice.
    - Bucket fix-it copy is paraphrased.
    - The drill-down shows `(unnamed)` instead of the derived label.
    - The README table row for bob-navigation-hotkeys is stale (2.45.2 vs 2.46.0), and
      the parent plan asked for that table to be brought current.
15. **Mac.**
    - `RefsTodayLinkCard.swift` (around 113–131 and 298) matches the search box against
      `homes.label` and pre-selects that row. Typing `sase` then links under an unnamed
      session instead of creating `SASE`, which breaks R7 ("derived homes are never
      capture `#name` targets"). `RefsTodayLinkTests.swift` around 135 locks this in.
    - The link toast still says "Linked under Untitled Pomodoro"
      (`RefsTodayLinkPresentation.swift` around 169/177).
    - `PomodoroHomeColor.swift` (around 48–50) always draws a filled dot, though R7 says
      hollow for forecast.
    - The Later strip's spoken text omits "for".
    - The HACK fixture's `bob · dev +1` label has only two parts.
    - The override banner says "Overrides the running session" for an unnamed session.
16. **chezmoi** never got the tmux status sample that the parent plan asked for.

Already tracked elsewhere, so not in this plan:

- The six clean-base bob-plugins failures: bob-cli-6v, bob-cli-6w, bob-cli-7i.
- The stage ranker timing flake: bob-cli-3w.
- The macOS CI Test-step compile error: bob-cli-9m.
- Pre-existing unnamed-session placeholders the parent plan never scoped: bare-`x` close
  preview, capture-complete `pomodoro_name` candidates, successor `entry_name`, Mac
  pickers, and plugin pickers. These are bob-cli-9n.

# Shared rules for every phase

- Rust is the authority. The JS mirror re-derives only what the ledger chip needs, under
  the shared vectors. Swift decodes and draws, and never derives
  (`decisions:mac-capture-is-a-thin-client`).
- Contracts stay additive. There are no schema bumps, and every new field is optional
  for consumers.
- Derived homes are display-only. They never route captures, never become `#name`
  targets, and never feed the plan budget.
- A named session prints exactly as it does today and does no home resolution on live
  surfaces. A derived label is printed only when the chip is non-null.
- No memory note may be edited, because the parent's memory decisions stay final.
- Each bob-cli phase finishes with `just check` and then `just install`. Do not run
  `just check-full`.
- Each bob-plugins phase finishes with `npm test` (only the six known failures above are
  acceptable), `npm run validate`, and `bob plugins sync`.
- Open linked repos (bob-plugins, bob-mac-capture, chezmoi) only through `/sase_repo`,
  and follow each repo's AGENTS.md.

# Phase contract — bob-cli homes contract parity, review time, and Stats fixes

**Splitting (R4/R5).**

- Change `split_minutes` to the floor of the even split, with the whole remainder on
  index 0: 8 over 3 gives 4/2/2, and 5 over 3 gives 3/1/1. It is used for task splits
  and for composite exempt names, so both now match `theme_minute_parts`.
- Update the unit test in `pomodoro_homes/tests.rs`.
- Fix the R4 and R5 wording in `docs/pomodoro-homes.md`.
- Re-derive any H or M vector that a 3+-way uneven split touches. The current H1–H13
  rows use only 3-way splits of 30m and 2-way splits, so they should not move. Confirm
  this.

**Closed means attributed (R9).**

- Pass the entry's closed state into attribution instead of inferring state from
  `minutes.is_some()`. Do this in `attribute_session` / `attribute_ledger_entry` and
  their callers in `capture_pomodoros.rs`, `plan_budget`, `pomodoro.rs`, and
  `stats/homes.rs`.
- A closed entry with no recorded or range duration is `attributed`. Its parts carry
  `minutes: 0`, with task-based shares.
- Keep the current closed-unlinked shape: one `@unlinked` bucket part with share 100 and
  the session's minutes. Document it in R9. An unlinked forecast still has empty
  `parts`.

**Vectors.**

- Restructure `tests/fixtures/pomodoro_homes/vectors.json` so only the
  ambiguous-basename case carries `"rust_only": true`. Move every other R1/R2/R8 case
  into shared cases.
- Add hand-computed cases for every gap listed in deviation 2:
  - a 3-task 50m split (18/16/16);
  - share rounding where the largest remainder is not the last part;
  - a work session linking another day's `^gtd`, homed to `@review:GTD`, with the bridge
    still same-day only;
  - `[t:: 40m]` on a 50m range;
  - a direct rolled-up child and a grandchild;
  - an unresolved link;
  - list-form and self `parent:`;
  - the depth cap of 16;
  - a closed `- [x] ()` entry;
  - an unquoted `parent: [[x]]`.
- Keep expected values computed by hand. The Rust test must run every case through the
  indexed source, and through the disk source where cheap.
- Replace the prose vector description in `docs/pomodoro-homes.md` with an index table
  of case ids and what each proves.
- Document the review label as `<NAME> review`.

> [!decision] review_row_time = yes `review:` selects review-bucket time per task, not
> whole sessions. A review session (R4) is entirely review. In a work session, a task is
> review when its home is a review bucket (R1.4 `^gtd`).
>
> - `review: only` keeps the review shares and `review: exclude` keeps the rest.
> - Minutes mode contributes the kept shares. Count mode counts a session once if any
>   share is kept, choosing the lead home among kept tasks, the same as `in:`.
> - `review: include` is unchanged, and `only + exclude = worked` in minutes mode.
> - The built-in `review_time` stays `{kind: pomodoro, measure: minutes, review: only}`.
>   It now equals the sum of the `@review:*` project parts of `worked` for any window.
>
> Add an H-vector: a named work session linking `[[#^gtd]]` plus a project task, so its
> `^gtd` share lands in `review_time`. Assert that `review_time` equals worked's
> `@review:GTD` project part. Add this case without moving H1–H13: put it on a new day
> outside their window with its own range assertion, or recompute every affected row by
> hand. Re-derive M10 by hand. Update the `review:` paragraph, the built-in catalog
> table, and the H table in `docs/stats.md`. On the live vault, confirm that
> `bob stats -f json -r 4w` gives `review_time` equal to the GTD review row.

> [!decision] review_row_time = no Keep `review:` on the session-level R4 predicate. In
> `docs/stats.md`, document that the GTD review row also includes `^gtd` shares from
> work sessions, so it can exceed `review_time`. Add an H-vector that pins that gap.

**Theme parts.**

- When no `in:` / `exclude_in` filter applies (and, under review_row_time = yes, no
  `review:` filter drops a share), keep the session's original theme-selected shares.
  Re-split theme minutes only when a filter actually removes a task share.
- Add a unit test for `A + B + C` at 50m with `themes: [A, C]`. It expects A 18 and C 16
  with no home filter.

**Catalog.**

- Emit catalog entries only for keys that appear in some part, point, apportioned map,
  or `sessions[].homes`, plus the area and subarea entries those entries reference.
- `rolled_up` and `project_label` are unchanged.

**CLI.**

- `bob projects list`: pad the name and the optional `rollup` tag as one cell, so STATUS
  stays aligned. Extend `tests/cli/projects/list.rs` to assert the column.
- `bob stats show`: when area or project parts print, add one dim line,
  `grouped by today's note tree · from Task Links`. Review buckets get a `●` in their
  area's xterm hue. `@unlinked` / `@unfiled` stay dim with no dot. Plain output drops
  the glyph.

**Hygiene.**

- Delete the stale `#![allow(unused)]` and its comment in `pomodoro_homes/mod.rs`.
- Remove dead `NoteFacts.daily`. Keep one rollup parser.
- Privatize or delete any re-export or helper the compiler then flags. No new
  allow-attributes.

**H-vector gaps.**

- Assert sum-to-total in every dimension for the sessions and done metrics.
- Add a count-mode `in:` case.

**Done when:**

- `just check` passes, and every vector and H-vector passes.
- Repeated computes are byte-identical.
- Release `bob stats -f json -r 4w` stays within about 15% of the 174 ms baseline.
  Record the number in the docs' Performance section.
- `just install` has run.

# Phase surfaces — bob-cli homes on bob ref today, human bob plan, and override surfaces

**`bob ref today`.**

- Give `TodayPomodoroRow` an optional `homes` field (R9; forecast for open rows,
  attributed for closed) through the disk source and `attribute_ledger_entry`. The
  callers around `src/native/ref_library/today.rs` lines 481, 501, and 520 need
  `bob_dir` and the daily path.
- Named rows do no resolution.
- Human output prints the derived label in place of `(untitled)` when the chip is
  non-null.
- Add a CLI test and a `docs/ref.md` line. The JSON stays additive.

**Human `bob plan`.**

- An unnamed entry prints the same dim chip with its dot that
  `capture-pomodoros --tasks` uses. Make `homes_chip_suffix` in
  `src/native/capture_pomodoros.rs` `pub(crate)`, or move it next to the homes JSON.
- The `(unnamed)` theme key stays for the budget.
- Add a test, and update the human-output sample in `docs/plan.md`.

**`=` override surfaces (integration with bob-cli-9j).**

- In `bob capture` start, dry-run, and preview JSON, `override.previous` and
  `override.demoted` gain an optional `homes` for the session they describe. Unnamed
  sessions only; named ones keep today's fields.
- Capture-complete `override.running` gains the same optional `homes`.
- In human output, the override warning, the swap demoted line, and the
  `--refuse-override` error print the chip after the range for an unnamed session with a
  non-null chip. Unlinked unnamed sessions print exactly as today, so the existing test
  at `tests/cli/capture/pomodoro_override.rs` (around line 1282) keeps passing.
- Relevant code: `src/native/capture/start_output.rs`, `src/native/capture/batch.rs`,
  `src/native/capture/output.rs`, `src/native/capture_complete/model.rs`, and
  `src/native/capture_complete/engine.rs`.
- Before starting, check `sase bead read bob-cli-9j` and rebase onto its landing tale if
  that tale has changed `pomodoro_start_override_at`.
- Add tests next to `pomodoro_override.rs` and the capture-complete output tests. Add
  the field to the override JSON section of `docs/capture.md`.

**Done when:** `just check` passes, `bob capture-pomodoros --tasks -f json` still runs
within 15 ms on athena, and `just install` has run.

# Phase plugins — bob-plugins JS mirror parity and Stats panel fixes

Open bob-plugins with `/sase_repo` and follow its AGENTS.md. Edit `src/` fragments,
build, never hand-edit `main.js`, and keep fragments at or under 1000 lines.

**Vectors.** Copy bob-cli's updated `tests/fixtures/pomodoro_homes/vectors.json`
verbatim into `scripts/fixtures/pomodoro_homes/`, and update the "copied from bob-cli
<rev>" note. The node test's fake `metadataCache` must produce Obsidian's frontmatter
shapes:

- unquoted `parent: [[x]]` is a nested array;
- `type` may be a string or a list;
- keys keep their case.

Every non-`rust_only` case must pass.

**Mirror parity** (`142-pomodoro-home-chips.js`). Match Rust on all of these:

- share rounding (largest fractional remainder, ties in ledger order);
- the lead home by task count, with ties going to the first task in ledger order;
- any canonical daily note's `^gtd` homed to the review bucket, with the bridge same-day
  only;
- the review label `<NAME> review`;
- `[t:: Nm]` preferred over the range;
- string or list `type`;
- `rollup` key and value case handling as in `resolve.rs`;
- string or nested-array `parent`;
- closed entries always attributed;
- the closed-unlinked `@unlinked` part;
- ambiguous and unresolved links counted in `unfiled_links`.

Then rerun the 4-week live comparison against `bob stats -f json -r 4w`
`sessions[].homes`. Expect zero differences in minutes or shares, and report any
remaining differences with their cause.

**Chips.**

- Draw only in Live Preview and Reading view, following the progress marks' Source-mode
  check.
- The Reading view counterpart (`274-…`) must also find embedded `![[t#^id]]` links.
- Check live in Obsidian that Reading view chips actually appear on a daily note, in
  light and dark mode. Look at a screenshot or the rendered DOM.

**Stats panel.**

- Chart pin refresh: `statsPinChartPoint` (click and Enter on a chart bar) must refresh
  the "Where the time went" panel to the pinned span, exactly like the panel's own pin.
  Add a test that drives the chart pin.
- The Themes view honors the pinned span.
- `@review:*` rows use their area hue; `@unlinked` / `@unfiled` stay gray.
- Count and task metrics keep the hue dots and mini-bars, dropping only the apportioned
  tint.
- A report without homes shows the Themes view and the "Update bob" notice without
  overwriting the saved view choice.
- Bucket fix-it copy reads exactly "link the task to attribute it" and "file the task in
  an area or project".
- Drill-down rows show the session's derived `homes.label` for unnamed sessions instead
  of `(unnamed)`.
- Extend `test-ledger-tools-stats-panels.cjs`.

**Release.**

- Bump the bob-ledger-tools minor version.
- Bring the README table current, including bob-navigation-hotkeys 2.46.0.
- Run `npm test` and `npm run validate`, then `bob plugins sync`.
- Check `bob plugins sync --dry-run` afterwards and confirm nothing is left to copy.

**Done when:** the tests pass apart from the six known failures, the live comparison
shows no minute or share drift, a live Reading view and Live Preview check shows chips,
and `dash_stats.md` repins from a chart click.

# Phase mac — Bob Mac Capture and chezmoi

Open bob-mac-capture with `/sase_repo` and follow its AGENTS.md. Never land Swift you
have not compiled.

**Refs Today Link.**

- In `RefsTodayLinkCard.swift`, the search query matches authored names and ranges only.
  It must never match `homes.label`, so a derived label never pre-selects a row or takes
  Return. The Create row keeps its place when no authored name matches. Rows still
  display the derived label.
- Update the test around `RefsTodayLinkTests.swift:135` to assert this.
- The toast (`RefsTodayLinkPresentation.swift`) names the derived label when the target
  row is unnamed and has one, for example "Linked under sase".

**Agenda.**

- `PomodoroHomeColor.swift` draws a ring for `forecast` and a filled dot for
  `attributed`.
- The Later strip's spoken text adds "for <label>".
- Fix the HACK fixture in `agenda-homes.json` so its label matches its parts.

**Override banner.** Decode the optional `homes` that the surfaces phase adds to
`override.previous` (`decodeIfPresent`, safe defaults). An unnamed running session's
banner and restart rows show the derived label (`CapturePomodoroStartPresentation.swift`
around 237, 472, and 642). When the label is absent they fall back to today's text.

**Fixtures.** Regenerate the Refs Today and override fixtures from real
`bob ref today -f json` and `bob capture --dry-run -f json` output against a scratch
vault. Keep the legacy payloads that have no homes.

**Verify.**

- Run `just portable-check` on athena.
- Compile the changed `Sources/` on the Mac with a scratch copy and run
  `just typecheck-tests`.
- Push, then check macOS CI. If bob-cli-9m has not landed, the Test step still fails on
  its pre-existing compile error. In that case confirm Lint and Build are green, and say
  in the final report that the macOS XCTests are unverified on CI.
- If the Mac is offline, say that the macOS targets are unverified.

**chezmoi.** Add an unnamed derived-label sample (`[<13m] 0950-1015 sase`) to
`tests/bash/tmux_status_right_test.sh`, then run that suite and the Hammerspoon suite.
If busted's runner is not on the path, use `eval "$(luarocks path --lua-version 5.1)"`.

**Done when:** portable-check passes, the Mac compile check passes, CI is green or its
only red is bob-cli-9m's pre-existing Test-step compile error, and the chezmoi tests
pass.

# Out of scope

- bob-cli-9n's pre-existing placeholder surfaces, the tracked bob-plugins failures and
  flake (bob-cli-6v, bob-cli-6w, bob-cli-7i, bob-cli-3w), and the Mac CI compile fix
  (bob-cli-9m).
- Everything in the parent plan's "Later" list.
- Closing `bob-cli-9k` itself. This epic's `parent_bead` link hands the landing back to
  bob-cli-9k's land agent once this epic lands.
