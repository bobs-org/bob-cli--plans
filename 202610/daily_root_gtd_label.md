---
tier: tale
title: Label the daily root parent GTD again
goal:
  The Day strip's root-parent bar paints GTD instead of <daily> for the synthetic daily
  root.
size: small
decisions:
  install:
    ask:
      Should the coder reinstall bob on this host (`just install`) so the Day strip
      shows GTD right away?
    default: true
    why:
      The strip reads the installed bob, so nothing visible changes until bob is
      reinstalled.
    answer: true
  memory_root_parent_glossary:
    ask: Should glossary:root-parent say the daily bucket is labeled GTD?
    memory:
      - glossary:root-parent
    default: true
    corrects: defined in canonical daily notes group under the `<daily>` bucket
    answer: true
proposed_by: bbugyi200.athena.14y
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 07:00:15
status: wip
---

# Label the daily root parent `GTD` again

## Goal

The Day strip's root-parent working-time bar should paint `GTD` for the daily root,
where it now paints `<daily>`. Bryan's goal is the Day strip, but the plan renames the
bucket's single display label everywhere it appears, because Bob owns every label and
clients only choose among the labels Bob sends.

## Background

- Commit `f6287847` (`feat(plan)!: make PLAN the review theme and add the <daily> root`)
  changed where review time rolls up. Before it, review time had its own root,
  `@review:GTD`. That root's catalog `short_label` was `GTD`, and the strip painted it.
  Commit `f6287847` made review buckets and tasks defined in daily notes roll up to a
  new synthetic root, `@daily`, labeled by `DAILY_LABEL = "<daily>"` in
  `src/native/pomodoro_homes/resolve.rs`.
- bob-plugins picks the strip's label in `dayStripLegendLabel`
  (`plugins/bob-ledger-tools/src/239-plugin-day-strip-model.js`). It uses a non-empty
  `short_label` first, then the bare catalog `label` for `root_parent`. `@daily` has no
  `short_label`, so the strip paints `<daily>`.
- Live vault check (`bob stats show worked -f json -r 7d`): every day since Oct 21 has
  10–150m on `@daily`. The `gtd.md` area root (label `gtd`) also logged time on Oct 23.
  The user's `plan.exempt` is `[GTD]`, so `@review:GTD` still exists, but as a
  project-dimension bucket whose root is `@daily`.

## Approach

Change only the display label of the synthetic daily bucket, from `<daily>` to `GTD`.
Everything else about `@daily` stays: its key, `kind: bucket`, gray/null hue, the
absence of a `short_label`, and all grouping rules. The Day strip (legend, tooltips,
aria sentences), the Stats dash, CLI human output, and the Pomodoro home labels and
chips then all read `GTD`. bob-plugins has a JavaScript mirror of the home resolver that
hardcodes the same label, plus a byte-for-byte copy of the shared
`pomodoro_homes/vectors.json`. Update both in the same change so live Obsidian chips
match bob.

Why not add `short_label: "GTD"` to `@daily` and keep `<daily>` as the label: the strip
has a rule for legend labels that match ignoring case. On a day when the `gtd` area root
also logs time (as on Oct 23), `GTD` and `gtd` match, and both items fall back to their
full labels. The strip would then show `<daily>` again, and the Stats dash and chips
would keep showing `<daily>` every day.

Expected side effects, which need no code changes:

- On a day with both roots, the label-collision rule makes the strip paint `GTD` and
  `gtd · area tasks`. `@daily`'s full label is itself `GTD`.
- The chip-suppression rule compares lowercased home labels with lowercased name
  components. A work session named `GTD + BOB` that links one daily-note task and one
  `bob` task now hides its chip, because its name already says `gtd`/`bob`.
- Ordering does not change. Session labels sort by share with ledger-order ties, and the
  byte order of bucket labels is unchanged (`GTD` still sorts before `PLAN review`,
  `unfiled`, and `unlinked`).
- This is not a CLI or JSON contract break, because keys and shapes are unchanged. Use a
  commit subject without `!`, e.g. `fix(homes): label the daily root parent GTD`. Do not
  touch the `Cargo.toml` version or `CHANGELOG.md`.
- Out of scope: the daily root's hue stays bucket gray. Review time painted orange
  before `f6287847`, but changing the hue would touch the hue rules in both repos.

## Steps

### 1. bob-cli source, tests, and shared vectors

- `src/native/pomodoro_homes/resolve.rs`: set `DAILY_LABEL` to `"GTD"`. Both `label` and
  `area_label` in `daily_home()` already read it.
- `src/native/stats/homes.rs` tests: change the three `"<daily>"` expectations
  (`review_root.label`, `daily.label`, `catalog["@daily"]["label"]`) to `"GTD"`.
- `tests/fixtures/pomodoro_homes/vectors.json`, case `daily-note-task-home`:
  `label`/`chip` `"<daily> · bob"` → `"GTD · bob"`; the `@daily` part's
  `label`/`area_label` → `"GTD"`. Edit these four strings in place and keep the file's
  formatting and `·` escapes (do not reserialize).
- Confirm `grep -rn '<daily>' src tests` is empty.

### 2. bob-cli docs

- `docs/pomodoro-homes.md`: in R2's labels sentence (`buckets keep \`PLAN review\`,
  \`<daily>\`, …`) and in R7 (`Use \`<NAME> review\`, \`<daily>\`,
  …`), replace `<daily>`with`GTD`. Name the key once, e.g. `` `GTD`(key`@daily`) ``.
- `docs/stats.md`, Day strip legend paragraph: change "the synthetic daily root paints
  `<daily>`" to `GTD`. Add one short clause saying that on a day when the `gtd` area
  also has time, the collision rule paints `GTD` and `gtd · area tasks`.
- `docs/develop/migration-notes.md`, the "Pomodoro root parent" row: "labeled `<daily>`"
  → "labeled `GTD`".
- `docs/reference/glossary.md` is generated from the glossary memory web, so never
  hand-edit it. Step 3 regenerates it.
- Run `just docs-check` after step 3.

### 3. Memory (decision `memory_root_parent_glossary`)

> [!decision] memory_root_parent_glossary Use `/sase_memory_write`. In
> `sase/memory/glossary/root-parent.md`, change "group under the `<daily>` bucket" to
> "group under the `GTD` bucket (key `@daily`)" and keep the rest of the strand. Run
> `sase memory init`, then `just docs-gen` to regenerate `docs/reference/glossary.md`.

> [!decision] memory_root_parent_glossary = no Leave the strand and the generated
> glossary alone. Follow `/sase_memory_write`'s declined-change routing.

### 4. bob-plugins (linked repo; open it with `/sase_repo` and read its AGENTS.md)

- `plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js`: in the daily-note branch,
  change `pomodoroHomeBucketPart("@daily", "<daily>", "@daily", "<daily>", null)` to
  pass `"GTD"` for the label and the area label.
- `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/vectors.json`: copy it
  again, byte for byte, from bob-cli's `tests/fixtures/pomodoro_homes/vectors.json`
  (`cmp` must report no difference). In that directory's `README.md`, update the
  "working tree based on `<sha>`" hash to the bob-cli HEAD the copy came from.
- `plugins/bob-ledger-tools/scripts/fixtures/stats/day-strip-fixtures.cjs`: set the
  `homes["@daily"]` label to `"GTD"`.
- `scripts/test-ledger-tools-day-strip.cjs`:
  - In "daily root paints its label in gray after real roots", expect `GTD` for
    `legendLabel`, `label`, and the painted legend text.
  - In "case-insensitive legend collisions fall back to full labels", assert that
    `@daily` paints `GTD`. That fixture (`sameHueRoots`) also has `@review:GTD` (short
    label `GTD`) and `gtd.md` (`gtd`), so this pins the collision case.
- Never hand-edit `main.js`. Run `npm run build`, `npm test`, and `npm run docs:check`.
  Then run `bob plugins sync`, which bob-plugins' AGENTS.md requires after any change.

### 5. Verify and deploy

- In bob-cli, `just check` must pass (fmt, clippy, every test binary).
- `grep -rn '<daily>'` over both repos (excluding `.git`, `target`, `node_modules`, and
  generated `main.js` checks) must find nothing.
- Read-only live check: from the vault directory (`~/bob`), run the freshly built bob
  (`cargo build`, then that debug binary) as `stats show worked -f json -r 7d`. Confirm
  `homes["@daily"].label == "GTD"` and that `parts.root_parent` keys are unchanged.

> [!decision] install After `just check` passes, run `just install` in bob-cli so the
> host's `~/.cargo/bin/bob` carries the change. Do not run `just install-all`, because
> it pulls the sibling repos and restarts Obsidian. Then confirm the installed
> `bob stats show worked -f json -r today` (run from `~/bob`) reports `GTD` for
> `@daily`.

> [!decision] install = no Leave the installed bob alone. Report that the strip shows
> `GTD` after the next `just install-all`.

Commit each repo separately with the non-breaking subject above, plus a matching
bob-plugins subject such as `fix(ledger-tools): mirror the GTD daily root label`.
