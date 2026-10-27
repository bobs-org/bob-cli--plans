---
tier: tale
title: Rename the GTD Pomodoro theme to PLAN and label daily-note time <daily>
goal:
  The Day Strip theme bar reads PLAN and its root-parent bar reads <daily> where both
  read GTD, with the vault, config, bob, and plugins renamed consistently.
size: medium
decisions:
  daily_label:
    ask:
      What should the root-parent bar call time spent on tasks defined in daily notes?
    choices:
      angle_daily:
        "`<daily>`: brackets mark a synthetic root, like `<stdin>`; matches the `#daily`
        tag"
      angle_day: "`<day>`: shorter, but reads like a unit of time (`<day> 55m`)"
      bare_daily:
        "`daily`: matches `unlinked`/`unfiled`, but looks like a real note stem"
    default: angle_daily
    why:
      True over multi-day ranges, never mistaken for a note, names where the tasks live
    answer: angle_daily
  daily_scope:
    ask:
      Should links to any task defined in a daily note (not just `^gtd`) count under
      `<daily>` instead of `unfiled`?
    choices:
      all:
        Every daily-note block attributes to `<daily>`; one more branch in bob and the
        plugin chip mirror
      review_only:
        Only review buckets roll up to `<daily>`; other daily-note tasks stay `unfiled`
    default: all
    why:
      The label then means exactly what it says; only `^gtd` is linked today, so history
      is unchanged
    answer: all
  review_area:
    ask:
      Should PLAN review keep the orange gtd area via a new root vault note `plan.md`
      whose parent is gtd?
    choices:
      plan_note:
        Create `~/bob/plan.md`; review keeps the gtd area, orange chip dot, and area
        history
      bucket:
        No vault note; review becomes its own grey `PLAN review` area, since `plan` is
        ambiguous
    default: plan_note
    why:
      The research verified it in a scratch vault and it keeps six months of area
      history in place
    answer: plan_note
  memory_glossary:
    ask:
      Should the glossary strands for Pomodoro Theme and Root Parent state the PLAN
      default and `<daily>` root?
    memory:
      - glossary:pomodoro-theme
      - glossary:root-parent
    default: true
    corrects: Buckets `GTD review`, `unlinked`, and `unfiled` stay buckets.
    answer: true
proposed_by: bbugyi200.apollo.ar
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 15:33:16
status: wip
---

# Rename the GTD Pomodoro theme to PLAN and label daily-note time `<daily>`

## Goal

Bryan is moving Pomodoro themes from _what the work is for_ to _what kind of work it
is_. The review theme `GTD` becomes `PLAN`. In the daily note's Day Strip (the
`bob-day-stats` block):

- The **theme bar** shows `PLAN 55m` where it showed `GTD 55m`. The colour stays muted
  grey because PLAN is exempt.
- The **root-parent bar** shows `<daily> 55m` where it showed `GTD 55m`. This root
  parent means "the task worked in this block was defined in an Obsidian daily note".
  That is exactly what the review row is: the `^gtd` task line in each daily note.

Context:

- The research report
  `research:202610/work_kind_pomodoro_theme_set/work_kind_pomodoro_theme_set.md` covers
  this; see its "Switching safely" section.
- The screenshot that motivated the change shows both bars with `GTD 55m`.

## Why `<daily>`

I considered several names. `<daily>` wins on four counts:

- **Synthetic marker.** Angle brackets are the usual way to name something that isn't a
  file, as in `<stdin>`, `<module>`, and `<unknown>`. Real root parents are lowercase
  note stems (`bob`, `sase`, `job`), so the label can't be mistaken for a note.
- **Matches the vault.** "daily" is the word the vault already uses: every daily note
  has `tags: [daily]`, the template is `_templates/daily.md`, and Obsidian's core plugin
  is called Daily notes.
- **True over ranges.** The Stats dash groups by root parent over ranges such as 4
  weeks. `<today>` would be wrong there.
- **Short.** It fits the one-line legend.

Rejected names:

- **`<today>`:** wrong for ranges.
- **`<day>`:** reads like a unit of time.
- **`daily` (bare):** looks like a real note; the vault even links a `daily` template.
- **`<journal>`:** `journal` is a real note, the parent of the year notes.
- **`<review>` or `PLAN review`:** names a purpose, which is the vocabulary being
  retired, and it repeats the theme bar.

The internal key is `@daily`, following the existing `@unlinked`, `@unfiled`, and
`@review:<NAME>` keys.

## Current behavior (verified in source)

- **Exempt defaults.** `plan.exempt` defaults to `[GTD]` in four places:
  - `PlanConfig::default` in `src/native/config/plan.rs`
  - `default_exempt()` in `src/native/stats/homes.rs`
  - a `"GTD"` fallback in `HomeResolver::home_for_link` in
    `src/native/pomodoro_homes/resolve.rs`
  - bob-plugins: `050-plan-budget.js` defaults, plus `["GTD"]` fallbacks in
    `142-pomodoro-home-chips.js`, `273-plugin-pomodoro-home-chips.js`, and
    `279-plugin-pomodoro-themes.js`
- **Review bucket.** `attribute_session` in `src/native/pomodoro_homes/session.rs` sends
  two kinds of session to `review_home(name)`:
  - all-exempt named sessions;
  - unnamed sessions linking that day's `[[#^gtd]]`, which bridge to `exempt[0]`.

  `^gtd` links inside work sessions also map to `review_home(exempt[0])`. `review_home`
  builds:
  - key `@review:<NAME>`
  - label `<NAME> review`
  - **`root_parent: @review:<NAME>`**
  - an area found by resolving the lowercase name as a note (`gtd` → `gtd.md`, orange)

- **Catalog.** `catalog_entry` in `src/native/stats/homes.rs` gives review buckets
  `short_label: <NAME>`. The plugin's `dayStripLegendLabel` (in
  `239-plugin-day-strip-model.js`) paints `short_label` first, which is why the
  root-parent legend says `GTD`.
- **Theme bar.** It reads the Stats `theme` dimension, which holds the raw ` — NAME`
  components. Unnamed sessions are `(unnamed)`.
- **Other daily-note tasks.** A link to any other block in a canonical daily note
  resolves the daily note's frontmatter: `type: [[day]]`, and a parent `[[2026/2026MM]]`
  that does not exist. It ends up `@unfiled` and adds to `unfiled_links` health. In the
  vault, `^gtd` is the only daily-note block ever linked from a Pomodoro (297 links).
- **Vault rename.** `bob themes rename GTD PLAN -d` currently refuses: _"GTD is a plan
  config reference; add PLAN to plan.exempt or plan.inventory_labels first."_ The vault
  has 142 GTD entries over 54 days, plus `_templates/daily.md`. The real config is the
  chezmoi source `home/dot_config/bob/config.yml`, which sets `exempt: [GTD]`.
- **No `plan` note.** The vault has no root `plan.md`. Two generated tag pages make the
  bare name `plan` ambiguous, so `review_home("PLAN")` would find no area.
- **Plugin fix copy.** The plugin's Stats fix copy hard-codes "GTD review stays grouped
  here…" (`239-plugin-stats-homes-render.js`).

Do **not** touch any `gtd` _identifier_. That covers:

- the `^gtd` block id and the bridge keyed on it
- `#gtd` / `#pre` / `#post` checklist tags and the review walk
- `gtd.md` and `gtd_daily.md`
- the `gtd_daily` task line
- "Morning GTD Review Footer"
- the README's "GTD workflow" wording
- write-once `.bob/ref-sources/**` snapshots
- immutable decision records

## Design

1. **PLAN is the default exempt theme.** Every `["GTD"]` default and fallback above
   becomes `["PLAN"]` in both repos. Config-driven behavior does not change.
2. **One synthetic root parent, `@daily`, labelled per the `daily_label` decision
   (`<daily>` by default).**
   - `review_home` keeps its key `@review:<NAME>`, label `<NAME> review`, and area
     lookup. Only `root_parent` changes, to `@daily`.
   - Every review bucket, including composites and multiple exempt names, therefore
     groups under one root, and the Day Strip and Stats dash root-parent views both say
     `<daily>`.
   - The project and area dimensions keep `PLAN review`.
   - `review_time`, `review_count`, and `review: only` are unchanged, because they key
     on session kind and home, not on the root.
3. **`@daily` is a grey bucket** like `@unlinked` and `@unfiled`:
   - `kind: bucket`, no hue, label and area label `<daily>`, area `@daily`, empty
     residence chain.
   - The plugin already paints every `@` key grey except `@review:*`, and sorts buckets
     after real roots. The root-parent segment moves from the gtd area's orange to grey,
     which matches the muted PLAN segment in the theme bar.
   - The legend shows the catalog `label`, since `@daily` has no `short_label`, so no
     plugin legend code changes.
4. > [!decision] daily_scope = all `HomeResolver::resolve_note(path)` returns
   > `daily_home()` (cached) when `task_status_hooks::canonical_daily_date(path)` is
   > `Some`. The existing `^gtd` branch in `home_for_link` stays first, so the review
   > mapping is unchanged. Any other daily-note block now becomes home `@daily` in every
   > dimension:
   >
   > - It no longer counts as unfiled health, and no unfiled warning fires.
   > - `home_for_note` gives tasks stored in daily notes the same home in the task-event
   >   metrics (done/created).
   > - The plugin's chip mirror gets the matching branch. A new shared homes vector pins
   >   this behavior.
   >
   > With `review_only`, skip this step; the vector and mirror branch are not added.
5. > [!decision] review_area = plan_note Create the vault note `~/bob/plan.md` with
   > frontmatter `parent: "[[gtd]]"` (no `type`) and one body line: "Anchors the PLAN
   > Pomodoro theme's review bucket to the gtd area." An exact root path beats basename
   > lookup, so `review_home("PLAN")` resolves to the gtd area (orange). No vault note
   > links `[[plan]]` today, so nothing re-resolves. With `bucket`, skip the note and
   > document that review is its own `PLAN review` area.
6. **Not changed:** R4 itself. A named all-PLAN session that links project tasks still
   counts as review, and so as `<daily>`. The research report accepted this (about 10
   hours in six months). Say so in the docs.

## Implementation steps

### A. bob-cli (this repo)

1. **`src/native/pomodoro_homes/resolve.rs`:**
   - Add `pub(crate) const DAILY_KEY: &str = "@daily";`.
   - Add `pub(crate) const DAILY_LABEL: &str = "<daily>";`, or the `daily_label` choice.
   - Add `pub(crate) fn daily_home() -> Home`, shaped like `unlinked_home()`.
   - In `review_home`, set `root_parent: DAILY_KEY.to_string()`.
   - Change the `bridge_name.unwrap_or("GTD")` fallback to `"PLAN"`.
   - Make the `daily_scope=all` branch in `resolve_note`.
   - Update the `Home::root_parent` doc comment: buckets are their own root, except
     review buckets, which roll up to `@daily`.
   - Re-export the new items from `pomodoro_homes/mod.rs`.
2. **`src/native/stats/homes.rs`:**
   - Make `default_exempt()` return `["PLAN"]`, preferably derived from
     `PlanConfig::default().exempt`, so the two never drift.
   - Add a `"@daily" => daily_home()` arm to `bucket_home`.
   - Make `ensure_root` (or `remember`) insert a `daily_home()` catalog entry when
     `home.root_parent == DAILY_KEY`. Today `ensure_root` returns early for every `@`
     root, so without this `@daily` would have no catalog entry or label.
   - Keep `short_label` on `@review:*` entries unchanged.
   - Confirm that `prune_home_catalog` in `stats/compute.rs` keeps `@daily`. It should,
     because it is a used `root_parent` key.
3. **`src/native/config/plan.rs`:**
   - Set `PlanConfig::default().exempt` to `["PLAN"]`.
   - Update the default-value test.
   - Tests that parse an explicit `exempt: [GTD, ...]` YAML may stay.
4. **`src/native/stats/registry.rs`:** reword the `review_time` description: "Minutes
   attributed to review rows (sessions named only with `plan.exempt` themes, PLAN by
   default), including linked ^gtd task shares inside work sessions."
5. **Tests:**
   - Update review-session `root_parent` assertions from `@review:GTD` to `@daily`. They
     are in `stats/homes.rs`, `stats/render.rs`, `stats/compute.rs`, and
     `pomodoro_homes/tests.rs`.
   - Assert the `@daily` catalog entry: label `<daily>`, kind `bucket`, no hue, no
     `short_label`.
   - Assert that `@review:<NAME>` entries carry `root_parent: "@daily"` and keep
     `short_label`.
   - Where a test relies on the _default_ exempt list, rename its ledger names from
     `GTD` to `PLAN`. Leave tests that pass `exempt: ["GTD"]` explicitly alone. The
     shared vectors pass `exempt` explicitly, so existing cases keep `GTD`.
   - `daily_scope=all`:
     - Add a resolver unit test: a canonical daily path gives `daily_home()`; a
       non-canonical path is unaffected.
     - Add one case to `tests/fixtures/pomodoro_homes/vectors.json`: a closed session
       linking `[[2026/20261017#^errand]]` plus a project task gives a `<daily>` part
       with key `@daily`, and the health shows no unfiled link.
   - Run `just all` (fmt, lint, test).
6. **Docs.** Per the docs reference memory, update the contract and reference pages,
   then run `just docs-gen` and `just docs-check`.
   - **`docs/pomodoro-homes.md`:**
     - R2: the review/bridge root becomes `@daily`. Add a daily-note row if
       `daily_scope=all`. Labels: `<daily>` is a bucket.
     - R4: default `[PLAN]`, root `@daily`, and the R4 caveat from Design §6.
     - R6: daily-note tasks are not unfiled (`all`).
     - R7: review buckets keep `short_label`; `@daily` uses its label.
   - **`docs/stats.md`:**
     - the `review_time` row
     - the H3/H5/H12/H15/H16 vector prose, where root_parent rows read `@daily`
     - the legend paragraph (around "review buckets carry `GTD`"), which now says the
       root-parent legend paints `<daily>`
   - **`docs/plan.md`:** the exempt default, the "GTD is exempt" prose, and the sample
     config and JSON that illustrate the default.
   - **`docs/reference/configuration.md`:** the `plan.exempt` default.
   - **`docs/pomodoro-themes.md`:** "GTD by default" and the muted examples.
   - **Exempt-entry examples:** `docs/concepts/today-and-the-pomodoro-ledger.md`,
     `docs/today-marks.md`, and `docs/task-status-hooks.md`, only where they describe
     the default exempt entry.
   - Leave `#gtd` checklist docs (`freshness.md`, `task-tag-marks.md`) alone.
7. **Memory**, only if `memory_glossary` is accepted, via `/sase_memory_write`:
   - In `glossary:pomodoro-theme`, the `plan.exempt` default becomes `[PLAN]`. The
     current text says "default `[GTD]`".
   - In `glossary:root-parent`, review buckets (and, under `daily_scope=all`, daily-note
     tasks) roll up to the `<daily>` bucket; `unlinked` and `unfiled` stay buckets.

   Then run `sase memory init` and `just docs-gen`, which regenerates
   `docs/reference/glossary.md`.

### B. bob-plugins (`sase repo open bob-plugins`; read its AGENTS.md)

1. **`["GTD"]` defaults become `["PLAN"]`:**
   - `src/050-plan-budget.js` defaults
   - `142-pomodoro-home-chips.js`: the `exemptNames` fallback, `firstExemptName`, and
     `bucketName`
   - `273-plugin-pomodoro-home-chips.js`: both returns
   - `279-plugin-pomodoro-themes.js`: the comment and the fallback
2. **`239-plugin-stats-homes-render.js` `statsPanelHomeFixCopy`:**
   - For `@review:*`, build the copy from the key's name:
     `${name} review stays grouped here; name mixed work to attribute it separately.`
   - Add an `@daily` branch: "Tasks defined in daily notes, including review, group
     here."
3. **`daily_scope=all`:** in `pomodoroHomeShareParts` (142), when a non-`^gtd` link
   resolves to a path matching `PLAN_DAILY_PATH_RE`:
   - push `pomodoroHomeBucketPart("@daily", "<daily>", "@daily", "<daily>", null)`;
   - add no unfiled health or warning.

   Copy bob-cli's updated `tests/fixtures/pomodoro_homes/vectors.json` verbatim to
   `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/vectors.json`.

4. **Tests:**
   - The Day Strip test: a `root_parent` part keyed `@daily` with catalog label
     `<daily>` paints legend `<daily>`, grey, after the real roots.
   - The fix copy for `@daily` and `@review:PLAN`.
   - Update tests that assert the GTD default.
   - Existing synthetic fixtures with explicit `@review:GTD` keys may stay.
5. **Ship it:**
   - Update `docs/bob-ledger-tools.md` where it states the default.
   - Run `npm run build`, then `npm test`, then `npm run docs:check`.
   - Run `bob plugins sync`.

### C. Config and vault rollout (after A and B pass their tests)

1. **chezmoi.** Open it with `sase repo open chezmoi`. In
   `home/dot_config/bob/config.yml`, set `exempt: [PLAN, GTD]`. PLAN must come first,
   because the unnamed `^gtd` bridge uses `exempt[0]`. Deploy this host's copy from the
   edited checkout with `chezmoi -S <chezmoi checkout> apply ~/.config/bob/config.yml`,
   then confirm with `bob themes list -a -f json` that PLAN is exempt.
2. **`review_area=plan_note`:** write `~/bob/plan.md` as in Design §5, then run
   `bob vault-sync`.
3. **Rename.** Run `bob themes rename GTD PLAN -d` first. Check the preview: about 142
   tails over about 54 canonical daily notes, plus `_templates/daily.md`; no prose, task
   text, `^gtd` links, or `.bob/ref-sources` changed. Then run
   `bob themes rename GTD PLAN`. It takes `bob_sync.lock`, writes one verified commit,
   and syncs. Undo is `git -C ~/bob revert <sha>`.
4. **Trim the config** to `exempt: [PLAN]` with the comment kept, then redeploy as in
   step 1.
5. **Verify with the workspace build**
   (`cargo run --release -- stats show worked -r <today> -f json`, or the equivalent
   read):
   - today's `root_parent` parts contain `@daily` and no `@review:*` key;
   - `homes["@daily"].label == "<daily>"`;
   - `theme` parts contain `PLAN` and not `GTD`;
   - `project` parts show `@review:PLAN` (area `gtd.md` under `plan_note`).

   Also run `bob themes list -a`: `plan` is muted/exempt and `gtd` is gone.

## Rollout notes for the final report

Commit subjects:

- **bob-cli:** a breaking `!` subject, because Stats JSON `root_parent` keys and the
  `plan.exempt` default change. For example:
  `feat(stats)!: roll review time up to the <daily> root parent; default plan.exempt to PLAN`.
- **bob-plugins:** `feat(ledger-tools): …`.
- **chezmoi:** `chore(bob): rename exempt theme GTD to PLAN`.

Never edit `Cargo.toml` versions or `CHANGELOG.md`.

Tell Bryan what is left: run `chezmoi update` on athena and the Mac, and
`just install-all` from bob-cli on the Mac. That pulls and installs bob-cli and
bob-plugins and restarts Obsidian. Until a host updates, its old config treats PLAN as
an ordinary theme: the theme is hashed, counts toward `max_themes`, and the Day Strip
keeps its old labels there.

## Out of scope

These are research recommendations Bryan did not ask for here:

- hiding JOB
- pinning theme colors
- `plan.max_themes: 3`
- creating WRITE
- rewording the Morning review checklist
- `^theme-on-task`
- changing R4 so that named PLAN sessions attribute by their links
- adding angle brackets to `unlinked` and `unfiled`
- bob-mac-capture, which renders bob's JSON and only mentions GTD in pinned test
  fixtures
