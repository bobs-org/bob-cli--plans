---
tier: epic
title: 'Configured Pomodoro themes: a vocabulary with colors and gentle warnings'
goal: 'A `themes:` block in bob''s config.yml declares Bryan''s Pomodoro theme vocabulary
  and each theme''s palette color. Every bob surface shows the configured colors,
  the menus suggest the configured themes, and using a name outside the vocabulary
  always works but produces a clear warning wherever a theme gets authored. Bryan''s
  six themes ship configured with distinct colors.

  '
decisions:
  unconfigured_hue:
    ask: How should a theme outside the configured vocabulary be colored?
    choices:
      mute: Muted gray with a dashed outline, so color means "one of your themes"
      keep: Keeps its pin/area/hash hue plus the dashed outline; may share a configured
        color
    default: mute
    why: A hashed stranger can wear PLAN's blue; muting makes color itself mean membership
    answer: mute
  memory_glossary_theme:
    ask: Update the Pomodoro Theme glossary term with the new hue order and configured
      themes?
    memory:
    - glossary:pomodoro-theme
    default: true
    corrects: Hue priority is pin, exempt mute, matching top-level area, then
    answer: true
  memory_hue_record:
    ask: Supersede the theme hue decision record with a config-first hue order record?
    memory:
    - decisions:theme-hue-is-pin-exempt-area-hash
    - decisions:theme-hue-is-config-pin-exempt-area-hash
    default: true
    corrects: 'Hue is resolved in order: a journal pin (`hue_source: pin`); else a'
    answer: true
  memory_vocabulary_record:
    ask: Add a decision record that configured themes are a soft vocabulary that warns,
      never refuses?
    memory:
    - decisions:configured-themes-are-a-soft-vocabulary
    default: false
    answer: false
phases:
- id: config
  title: bob-cli themes config, hue rule, and catalog reports
  depends_on: []
  size: medium
  description: 'config: parse and validate the `themes:` block, put config colors
    first in the hue rule, add `configured` to every ThemeRef, make `bob themes list`
    and `bob themes color` config-aware, extend the hue vectors, and document PT12.'
- id: warnings
  title: bob-cli suggestions and warnings for unconfigured themes
  depends_on:
  - config
  size: medium
  description: 'warnings: limit Recent suggestions to configured themes and emit `theme_not_configured`
    from capture, Name-it, bob plan, rename/split, and themes-list hints, plus menu
    vectors and docs.'
- id: obsidian
  title: bob-plugins configured-theme rendering, pickers, and lints
  depends_on:
  - config
  - warnings
  size: medium
  description: 'obsidian: decode the new report fields, mirror the hue rule and menu
    scoping in JS, badge unconfigured create rows, mark unconfigured ledger chips,
    update the theme manager, and add the plan-block lint.'
- id: mac
  title: Bob Mac Capture configured-theme rendering and warnings
  depends_on:
  - config
  - warnings
  size: medium
  description: 'mac: render `configured: false` themes with the dashed treatment,
    badge unconfigured create rows, show `theme_not_configured` in every preview and
    in Name-it without flicker, and scope Name-it chips to configured themes.'
- id: dotfiles
  title: Bryan's six configured themes in chezmoi
  depends_on:
  - config
  size: xsmall
  description: 'dotfiles: add the six-theme `themes:` block with the chosen colors
    to the chezmoi bob config, check it with the new parser, and apply it.'
proposed_by: bbugyi200.athena.151
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 07:47:08
status: done
bead_id: bob-cli-dk
---

- **PROMPT:** [prompts/202610/configured_pomodoro_themes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/configured_pomodoro_themes.md)
- **BEAD:** [bob-cli-dk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dk/README.md)

# Plan: Configured Pomodoro themes

## Context

Pomodoro themes are fully derived today (`docs/pomodoro-themes.md` PT1–PT11). The
catalog comes from canonical daily notes. Hide, pin, and rename state comes from the
per-host journal `.bob/themes/<host>.jsonl`. The hue rule is pin → `plan.exempt` mute →
top-level area stem → hash. Clients render the `hue` bob reports.

Bryan has narrowed his vocabulary by hand. He hid 71 of 76 known themes and pinned
colors on five of the six he still uses: PLAN, BUILD, READ, CHORE, WRITE, FIX. Nothing
stops a new stray name, menus can still offer old ones, and PLAN's hash color (aqua)
matches CHORE's pin.

This epic makes the vocabulary explicit in `~/.config/bob/config.yml`, which chezmoi
manages and the user owns. bob reads that file and never writes it. Rust stays the only
owner of meaning: clients render the new JSON fields and never parse `themes:` on their
own.

Repositories: `bob-cli` (phases `config` and `warnings`), `bob-plugins` (`obsidian`),
`bob-mac-capture` (`mac`), and `chezmoi` (`dotfiles`). Phases that work outside bob-cli
must open their repo with `/sase_repo` and read that repo's `AGENTS.md` first.

**Compatibility.** JSON stays additive at `schema_version: 1`. Never bump it: the Mac
app and the plugins reject any other version. Every new field is absent when no
`themes:` block is configured, so output without the block is byte-for-byte unchanged.
Both clients already accept `null` `first_used`/`last_used`, unknown keys, and unknown
`hue_source` strings. The Mac already renders structured top-level capture `warnings`
inside its Pomodoro start and close cards. So the bob-cli phases can ship before the
client phases without breaking anything.

## Design

### The config block

```yaml
themes:
  PLAN: blue
  BUILD: yellow
  READ: green
  CHORE: magenta
  WRITE: violet
  FIX: red
```

- **Shape.** A mapping from theme name to color. The value is a palette slot (`blue`,
  `orange`, `aqua`, `yellow`, `magenta`, `green`, `violet`, `red`; case-insensitive,
  trimmed), `auto`, or empty/null. `auto` and null both mean "configured, but bob picks
  the color" (pin, then exempt, then area, then hash). `auto` matches the existing
  `bob themes color NAME auto`.
- **Why palette slots only.** No hex values. Every surface already has a light/dark pair
  and an xterm-256 code for each slot: Obsidian CSS, Mac `PomodoroHomeHue`, the
  terminal, tmux, and Hammerspoon. A hex color would render gray on the Mac and need
  derived variants everywhere.
- **"Configured" means** the block is present, valid, and has at least one entry.
  Missing, `null`, or `{}` means no configured themes, which is exactly today's
  behavior.
- **Validation.** It is strict, and a failure drops the whole block, so behavior is as
  if no themes were configured:
  - `themes` must be a mapping.
  - Each key must be a YAML string. A number or boolean key fails with "quote theme
    names YAML reads as numbers or booleans".
  - Each name must be a valid single theme, using the same rule as rename's NEW: not
    blank after trimming, and no `+`.
  - Two names with the same `normalize_component` key fail, and the error names both
    spellings.
  - A value that is not a slot, `auto`, or null fails, and the error lists the valid
    colors.
  - Every message names the config path and the offending `themes.NAME`.
- **Where an invalid block is reported.** It is never fatal anywhere:
  - `bob themes list`: as a `warnings` entry, printed in yellow in human output.
  - `bob plan`: as the lint `themes_config_invalid`.
  - `bob capture`: as a plain warning string.

  `themes list` must not exit 2: the Obsidian plugins depend on it for every pill. A
  broken block degrades to "no configured themes"; it never breaks a surface.

- **Identity and display.** A configured theme's key is `normalize_component(NAME)`. The
  history catalog itself does not change: it is still derived only from daily notes plus
  the journal, per decision `pomodoro-theme-is-one-component`. Reports combine the
  catalog with the configured set. A configured theme with no history displays its
  config spelling. Otherwise the most recently authored spelling wins, as today.
- **One read.** One config read per catalog load provides both `plan.exempt` and
  `themes:`. This replaces today's separate `load_exempt()` read, so the +5 ms p95
  budget for warm `capture-complete`/`capture-parse` (PT3) still holds.

### Hue rule (PT5 revised)

1. **Config color**: `hue_source: "config"`.
2. **Journal pin**: `pin`.
3. **`plan.exempt` mute**: `hue: null`, `exempt`.
4. **Not configured**: only when themes are configured and the key is not one of them.

   > [!decision] unconfigured_hue = mute `hue: null`, `hue_source: "unconfigured"`.

   > [!decision] unconfigured_hue = keep No step 4. Unconfigured themes fall through to
   > area/hash and still carry `configured: false`.

5. **Top-level area stem**: `area`.
6. **Hash**: `hash`.

The order is explicit choices (config, pin), then mutes, then implicit sources (area,
hash). Config beats pin because the user-owned file is the source of truth: editing it
must always take effect. So a theme whose config sets a color refuses `bob themes color`
with `config_color` (below). Pins still apply to configured `auto` themes and to
unconfigured themes. Journal pins shadowed by a config color stay in the journal
untouched, and they apply again if that config color is removed. A config color also
beats the exempt mute, so PLAN can wear blue and still not count toward
`plan.max_themes`.

### The `configured` flag and `known`

- **`configured`.** When themes are configured, every `ThemeRef` and every `themes list`
  row carries `configured: true|false`. When none are configured, the field is absent.
  Clients treat `configured === false` as "not one of your themes", and absent as "no
  opinion".
- **`known`.** It also becomes true for configured themes that have no history. A
  configured name is tinted as soon as it is typed, because it is part of your
  vocabulary.
- **Inventory labels and `(unnamed)`.** `plan.inventory_labels` are storage, not themes,
  and already get `inventory_label_open`. They never get `theme_not_configured` warnings
  and are never muted as unconfigured. They report `configured: true`, and their
  `hue_source` follows the normal rule. `(unnamed)` is not a theme and gets neither.

### Where the warning shows

One code is used everywhere: `theme_not_configured`. The message is
`SNACK isn't one of your configured themes`, plus `; did you mean CHORE?` when the
similarity rule (PT7) matches a configured theme. Warnings never block anything, and no
`strict` mode exists for them.

**Active surfaces: the moment a theme is authored.**

1. **Theme-name menus** (`capture-complete`, Mac, zsh, Obsidian pickers). This is
   prevention, at the point of choice. With configured themes:
   - Recent rows come only from configured themes, minus hidden themes and Today
     components. That includes configured themes never used yet: they rank after used
     ones, in config order, with no `last_used`/`days_used`.
   - The empty-query cap rises from 5 to 12, so the whole vocabulary shows.
   - `similar_to` draws only from visible configured themes.
   - The create row's `themes` refs carry `configured: false`. Clients show a "Not in
     your themes" warning badge, and zsh shows `new · not in your themes`.

   Suggesting a name and then warning when it is used would contradict itself; this
   scoping removes the need for Bryan's 71 hides.

2. **`bob capture`**, including `--dry-run`, which is the Mac preview. bob diffs every
   staged canonical daily note before and after the batch. A name counts as "used by
   this batch" when its count of entries (open or closed) grew, which covers starts,
   "again" starts, deferred `$<N>` captures, and renames. Closing or adjusting an
   existing entry does not count.

   Each unconfigured, non-inventory component of such a name adds one structured
   top-level warning, deduplicated by key in first-seen order:
   `{code: "theme_not_configured", label: NAME, message}`. Human output prints it
   through `print_capture_warnings`.

3. **`bob capture-pomodoro-name`** (Mac Name-it). Its success JSON, including
   `--dry-run`, gains the same `warnings` array, and human output prints it.
4. **Obsidian pickers that write names in JS** (block-id-prompt link picker,
   navigation-hotkeys bullet-move/rename pickers). The create row shows a badge, and
   after commit the Notice gains a ` · SNACK isn't one of your themes` suffix.
5. **`bob themes rename` / `split`**:
   - A NEW name or split target that is not configured adds a `warnings` entry to the
     preview and result, and human output marks it `◌ … not configured`.
   - Renaming a configured OLD to an unconfigured NEW refuses with the existing
     `config_reference` code: "BUILD is a configured theme; add NEW under themes:
     first". This is the same rule as for `plan.exempt`.
   - A `--merge` of two configured themes notes in the preview that OLD stays in
     `themes:` until you remove it.

**Passive surfaces: detection, so names typed by hand or synced from elsewhere are
caught.**

6. **`bob plan`** adds lint `theme_not_configured`, once per key, at the first open
   entry that uses it. The Obsidian `bob-plan` block mirrors it.
7. **Pills and swatches everywhere** (PT6 addition). A theme with `configured: false`
   uses the **unconfigured pill**: a 1 px dashed outline in the platform warning color
   (Obsidian `--text-warning`, macOS `systemOrange`) at about 70 % opacity, and the
   tooltip/AX suffix "not in your themes". Exactly like the existing "Typing" rule, the
   mark appears only once the name is committed or the cursor leaves it, so `BUI` on the
   way to `BUILD` never flashes.

   > [!decision] unconfigured_hue = mute The pill underneath is the muted pill (exempt
   > styling).

   > [!decision] unconfigured_hue = keep The pill keeps its hue tint under the outline.

8. **CLI swatch.** `■` stays the colored theme swatch and `□` stays exempt. Unconfigured
   themes get `◌` (U+25CC, from the same Geometric Shapes block), dim under mute or in
   its hue under keep. Under `NO_COLOR` the glyph and the `not configured` text still
   carry the meaning.
9. **`bob themes list` hints**:
   - Hint `theme_not_configured` covers visible (not hidden), unconfigured,
     non-inventory themes used today or in the last 7 days. Hidden themes are excluded:
     hiding already dismisses them, and Bryan has about 40 hidden strays from before his
     consolidation.
   - Human output prints at most 3 hint lines, then `… and N more · bob themes -a`. A
     line reads:
     `SNACK isn't one of your configured themes → add it under themes: in ~/.config/bob/config.yml, or fold it in: bob themes rename SNACK CHORE`.
     The rename part appears only when a similar configured theme exists.

**Not warned:** Stats, the Day strip, history, hide/unhide, linking to an existing open
entry, and closing or adjusting entries. These display or touch existing entries; they
do not author a theme. Mac post-submit notifications are not warned either: the preview
already showed the warning before Return.

### `bob themes list` and `bob themes color`

- **Rows.** The default view always includes visible configured themes, even never-used
  ones, which show `never` and `0 days` and sort last. Other visible themes used in the
  window also appear. `-a`, `-H`, and `-q` are unchanged, except that they include
  never-used configured themes. A never-used configured row has `first_used: null`,
  `last_used: null`, `days_used: 0`, `entries: 0`, and no `today`.
- **Human output**, for the mute branch:

  ```text
  THEMES  6 configured · 76 known · 2 today · 71 hidden · last 90 days
    ■ PLAN      today     55 days   blue · config
    ■ BUILD     today     31 days   yellow · config
    ◌ SNACK     Oct 27    1 days    muted · not configured
    hint  SNACK isn't one of your configured themes → add it under themes: in ~/.config/bob/config.yml
  ```

  Configured `auto` themes keep today's labels: `aqua`, `red · pinned`,
  `muted · exempt`, `aqua · area`.

- **JSON additions**, each present only when themes are configured:
  - `counts.configured`.
  - Top-level `configured_themes: [{key, name, color}]` in config order, where `color`
    is a slot or `null`.
  - Per-row `configured`.
  - `hue_source` values `config` and, under mute, `unconfigured`.
- **Collision hints.** `theme_hue_collision` skips config-colored and muted themes, just
  as it skips pins.
- **`bob themes color NAME HUE|auto`** on a theme whose config sets a color refuses with
  exit 1, writes nothing to the journal, and returns
  `{ok: false, code: "config_color", message: "BUILD's color comes from themes: in ~/.config/bob/config.yml (yellow)", hint: "edit themes.BUILD there, or set it to auto to pin colors with bob themes color"}`.
  Display the config path with `~` when it is under home, and respect
  `BOB_CONFIG_FILE`/`XDG_CONFIG_HOME`.

### Bryan's palette

| Theme | Color   | Why                                                                                         |
| ----- | ------- | ------------------------------------------------------------------------------------------- |
| PLAN  | blue    | The daily anchor; it shares days with every other theme, so it takes the clearest cool slot |
| BUILD | yellow  | Hard-hat amber; keeps the current pin                                                       |
| READ  | green   | Library green; keeps the current pin                                                        |
| CHORE | magenta | Rubber-glove pink; far from the PLAN/BUILD/READ core it shares 14–16 days with              |
| WRITE | violet  | Ink; the classic creative hue                                                               |
| FIX   | red     | Bugs and alarms                                                                             |

**How the colors were chosen.**

- **Co-occurrence**, from the last 60 ledger days:
  - BUILD+PLAN on 26 days, PLAN+READ 24, BUILD+READ 21.
  - CHORE+PLAN 16, CHORE+READ 16, BUILD+CHORE 14.
  - WRITE appears on 9 days and FIX on 4.
- **Distance.** CIEDE2000 distances were computed for each slot's light/dark solid hex,
  its 16 %/24 % pill tints, and its xterm code. The four themes that share the most days
  got four different hue families (blue, yellow, green, pink).
- **Unused slots.** Orange sits between red and yellow, and aqua shadows green, so both
  are left out.
- **Closest remaining pairs.** These sit on the rarest overlaps: CHORE/FIX (pink vs red,
  3 shared days) and PLAN/WRITE (blue vs violet, 9).
- **Changes from today.** The block moves CHORE from aqua to magenta, WRITE from magenta
  to violet, and FIX from blue to red. PLAN gains blue. Config wins over the old pins.

## Phase `config` — bob-cli themes config, hue rule, and catalog reports

1. **Parser.** Add `src/native/config/themes.rs`:
   - `ThemesConfig { entries: Vec<ConfiguredTheme { key, name, color: Option<Hue> }> }`
     plus a loader that returns `Ok(None)` for missing, null, or empty. Validation
     follows "The config block".
   - Add `themes: Option<serde_yaml::Value>` to `RawConfig`, and re-export it from
     `config/mod.rs` like the other blocks.
   - Unit tests cover each validation rule and its exact message, along with shorthand
     slots, `auto`, null, case-folding, duplicate keys, non-string keys, a missing file,
     and an invalid file.
2. **One read.** Replace `pomodoro_themes::load_exempt()` with a single loader that
   returns `(exempt, Option<ThemesConfig>, Option<warning>)`. Thread it into `Catalog`
   and `ThemeLookup`.
   - Add accessors: `configured(key)`, `has_configured()`, `config_color(key)`, and the
     ordered list.
   - Every caller already goes through `theme_ref`: capture-parse, capture-complete,
     capture budget refs, `bob plan`, the tmux line, stats `theme_hues`, and task-status
     hooks. So the hue change reaches every surface.
3. **Hue rule and ThemeRef.**
   - Extend `resolve_hue` and `HueSource` per "Hue rule (PT5 revised)". Follow the
     `unconfigured_hue` decision for step 4.
   - Add `configured: Option<bool>` to `ThemeRef` with
     `skip_serializing_if = "Option::is_none"`.
   - Make `known` true for configured keys.
   - `swatch_name` renders `◌` for `configured == Some(false)`, ahead of the
     unknown-name rule.
4. **`bob themes list`.** Implement "`bob themes list` and `bob themes color`":
   never-used configured rows, the header count, meta labels, the `◌` swatch, the JSON
   additions, collision-hint exclusions, and the config warning. Leave
   `theme_not_configured` hints to phase `warnings`.
5. **`bob themes color`.** Add the `config_color` refusal and add it to the refusal-code
   list in help text and docs.
6. **Hue vectors.** In `tests/fixtures/pomodoro_themes/vectors.json`, `hue` cases gain
   an optional `configured` input (`{key: slot|null}`); absent means no configured
   themes. Add cases for:
   - config beats pin;
   - config beats exempt;
   - configured `auto` falls through to pin, to area, and to hash;
   - an unconfigured key, both muted and keep per the decision, and with a pin;
   - no `configured` input, unchanged.

   Run them in the Rust vector tests.

7. **Docs.**
   - In `docs/pomodoro-themes.md`, add **PT12 — Configured themes** (the config block,
     validation, identity, `configured`/`known`, the warning code, and the surface list
     above). Update PT5, PT6 (unconfigured pill and `◌`), the JSON contract, the CLI
     section, and the intro's "PT1–PT11" range.
   - In `docs/reference/configuration.md`, add a `### themes` key section and the
     example to the sample file.
   - Update `bob themes list`/`color` clap help, then run `just docs-gen` and
     `just docs-check`. Read `sase/memory/docs.md` and `sase/memory/cli_rules.md` with
     `/sase_memory_read` first.
8. **Memory.** Use `/sase_memory_write` and keep within the accepted memory decisions.
   Run `sase memory init` afterwards.

   > [!decision] memory_glossary_theme Rewrite the hue sentence in
   > `glossary:pomodoro-theme` to give config color first, then pin, exempt mute, the
   > unconfigured mute (if `mute`), area, and hash. Add one sentence: configured themes
   > live under `themes:` in config.yml and scope Recent suggestions.

   > [!decision] memory_hue_record Add the strand
   > `decisions/theme-hue-is-config-pin-exempt-area-hash.md` with the revised order, why
   > config beats pin, the rejected alternatives (pin beats config; hex colors; unique
   > colors), and the cost (the `config_color` refusal; shadowed pins). Mark
   > `theme-hue-is-pin-exempt-area-hash` superseded with `superseded_by` and a
   > back-link, per the decisions web rules.

   > [!decision] memory_vocabulary_record Add the strand
   > `decisions/configured-themes-are-a-soft-vocabulary.md`. Its claim: config-owned,
   > bob never writes it; warns, never refuses; scopes Recent and `similar_to`; an
   > invalid block degrades to none. Rejected alternatives: strict refusal, warn-only
   > without menu scoping, vault-stored vocabulary.

   > [!decision] memory_vocabulary_record = no A phase worker records a
   > `PROPOSED FOLLOW-UP:` note on its phase bead instead.

**Verify:** run `just all`. Then, with a temp config through `BOB_CONFIG_FILE`, run
`cargo run -q -- themes list` and `cargo run -q -- themes list -f json` against a
fixture vault: six configured rows, `configured_themes`, a never-used configured row,
and an unconfigured row with `◌`. Run `cargo run -q -- themes color BUILD red` and
confirm it refuses with `config_color` and leaves the journal unchanged. Finally,
confirm that removing the block reproduces today's JSON byte-for-byte; snapshot or
golden tests should cover this.

## Phase `warnings` — bob-cli suggestions and warnings for unconfigured themes

1. **Menus.** In `src/native/capture_complete/theme_menu.rs` `compose`, implement the
   menu rules from "Where the warning shows", item 1:
   - Recent scoping: rank the configured set, including never-used themes, through
     `Catalog`.
   - The empty-query cap of 12, and `similar_to` over configured themes in
     `Catalog::similar_to`.
   - zsh descriptions: `your theme · never used` for a never-used Recent row, and
     `new · not in your themes` on unconfigured create rows.
   - Theme-name shell completion for `bob themes` arguments
     (`src/native/completion/providers.rs`) also offers never-used configured themes.
   - Add `menu`-section vectors for configured scoping, including a never-used
     configured row, a hidden configured theme, and an unconfigured create row.
2. **Capture.** In `src/native/capture/budget.rs` (or a sibling module), add the
   staged-daily-note ledger diff and the structured `theme_not_configured` warnings
   (item 2). Reuse `StructuredCaptureWarning` with `label`. Cover with tests:
   - a start of an unconfigured name, an "again" start, and a `$<N>` deferral;
   - a rename via a Pomodoro-name field;
   - the negative cases: close, link to an existing entry, a configured name, an
     inventory label, and no themes configured.
3. **`capture-pomodoro-name`.** Add `warnings` to its JSON and human output, including
   `--dry-run`.
4. **`bob plan`.**
   - Add lint `theme_not_configured` in the CLI layer next to `with_theme_refs`, so the
     pure `compute_for_daily` keeps its signature. Use the first open-entry line.
   - Add `themes_config_invalid` for a broken block.
   - Add both to the Lints table in `docs/plan.md`, and add a conformance example with
     configured themes.
5. **Rename and split** (`rename.rs`, `split.rs`): add the warnings, the extended
   `config_reference` rule, and the merge note (item 5). Add JSON `warnings` and human
   preview lines.
6. **`themes list` hints.** Add the `theme_not_configured` hints (item 9) in `hints.rs`:
   - JSON is one hint per theme:
     `{code, theme: {key, name}, last_used, days_used, similar_to?}`.
   - Human output follows the 3-line cap.
7. **Docs.** Update PT7, PT10, PT11, PT12's surface list, and the JSON contract in
   `docs/pomodoro-themes.md`; capture warnings in `docs/capture.md` and
   `docs/reference/json-outputs.md`; and help text for `capture-pomodoro-name`. Then run
   `just docs-gen` and `just docs-check`.

**Verify:** run `just all`. With a temp config,
`cargo run -q -- capture --dry-run -f json -- '=SNACK'` against a fixture vault shows
the structured warning, and `=BUILD` shows none. `capture-complete` for an empty `=`
query lists all configured themes. `bob plan` lints an open SNACK entry.

## Phase `obsidian` — bob-plugins configured-theme rendering, pickers, and lints

Work in the linked `bob-plugins` repo. Edit `src/` fragments, which are capped at 1000
lines each, and never the deployed copies under `~/bob`.

1. **Decoder.** In `bob-ledger-tools/src/279-plugin-pomodoro-theme-mirrors.js`:
   - `sanitizeTheme` keeps `configured`, and keeps `hue_source` `config`/`unconfigured`
     without folding them to `hash`.
   - The report keeps `configured_themes` and `counts.configured`.
   - Expose both through `api.themes` (`snapshot`), plus a helper `isConfigured(key)`
     that returns `true`, `false`, or `null` when no themes are configured.
2. **Hue mirror.**
   - `pomodoroThemeResolveHue` gains the config step and the decision's unconfigured
     step, taking the configured map from the snapshot. The JS fallback is used only for
     keys missing from the report.
   - Copy `tests/fixtures/pomodoro_themes/vectors.json` byte-for-byte from bob-cli,
     update the fixture README's source commit, and run the hue, fold, rank, and menu
     vectors.
3. **Pickers.**
   - In `block-id-prompt/src/076-…/085-…` and `bob-navigation-hotkeys/src/165-…/201-…`,
     apply the Recent scoping, the 12-row empty cap, and `similar_to` over configured
     themes.
   - Add a "Not in your themes" warning badge on unconfigured create rows. Give it a new
     `is-warning` badge style with an amber outline, next to `like X`.
   - After committing an unconfigured name, add the Notice suffix.
4. **Ledger chips.** In `279-plugin-pomodoro-theme-chips.js` and `-reading.js`:
   - Add the `bob-pomo-theme-unconfigured` class and a `· not in your themes` tooltip
     suffix.
   - Live Preview skips the cursor line, following the existing `-unknown` rule.
   - Add CSS in `styles.css` for the dashed outline, plus the muted base under mute.
5. **Theme manager** (`202-…`, `203-…`):
   - Add a `config` hue chip and a `not configured` badge.
   - "Pin color" is disabled for `hue_source: "config"`, with the tooltip "Color set in
     ~/.config/bob/config.yml". If bob still refuses with `config_color`, show bob's
     message.
6. **Plan block.** `computePlanBudget` consumers add `theme_not_configured` lints, using
   the same Rust message, from the snapshot's configured set. When no snapshot exists
   (mobile), they emit nothing.
7. **API docs and tests.** Update `docs/develop/plugin-apis.md`. Extend the theme,
   picker, manager, and plan-budget test scripts.

**Verify:** run `npm run build`, `npm test`, and `npm run docs:check`. Then deploy with
`bob plugins sync`, following the repo's AGENTS.md. With an older bob that lacks the new
fields, every surface behaves as it does today.

## Phase `mac` — Bob Mac Capture configured-theme rendering and warnings

Work in the linked `bob-mac-capture` repo, and keep it a thin client.

1. **Models.**
   - `CapturePomodoroThemeRef` and `CapturePomodoroTheme` decode `configured: Bool?`.
   - The themes response decodes `configuredThemes`.
   - Spoken labels handle `hue_source` `config` and `unconfigured`.
   - Harden `PomodoroThemeSwatches` (`CompletionRowContent.swift`, around lines 317 and
     336). Replace `Dictionary(uniqueKeysWithValues:)` with `uniquingKeysWith`, so a
     refs array with a repeated key (`— FIX + fix`) cannot trap.
2. **Pills and swatches** (`PomodoroThemeViews.swift`, editor highlight in
   `CapturePanelModel.swift`):
   - Apply the unconfigured treatment: a dashed `systemOrange` outline at about 70 %,
     muted underneath per the decision, and the AX label "not in your themes".
   - The swatch becomes a dashed outline square.
   - Apply the typing rule: no mark while the caret is in the name.
3. **Completion rows** (`CompletionRowContent.swift`, `CompletionRow`): add a "Not in
   your themes" badge with a new warning style (orange outline) on create rows whose
   themes include `configured == false`.
4. **Preview warnings.**
   - Render structured `theme_not_configured` warnings in every capture preview, not
     just the Pomodoro start and close cards. Use one orange caption row with the
     unconfigured pill and bob's message, placed with the plan-budget captions.
   - Show it once the preview settles.
   - Suppress it while the caret sits at the end of that name and the name is a
     case-insensitive prefix of a visible configured theme. The menu is already offering
     that theme, so typing `BUILD` never flickers a warning, while `SNACK` warns even if
     Return follows immediately.
   - Do not add it to post-submit notifications.
5. **Name-it.** When `configuredThemes` is present, the "Recent themes" chips come from
   visible configured themes. `capture-pomodoro-name` warnings render in the prompt.
6. **Docs and tests.**
   - Update `docs/protocol.md`'s optional-fields list.
   - Extend `PomodoroThemeSwatchesTests`, `CompletionRowContentTests`,
     `CaptureModelTests`, `BobProcessClientTests`, and
     `CaptureEditorHighlightTests`/`CapturePanelModelTests`.

**Verify:** run `just portable-check` on athena. It cannot type-check the
`BobMacCapture*` targets, so state that in the phase notes, and run `just all` or
`just typecheck-tests` on the Mac when available. Finally, run `just docs-check`.

## Phase `dotfiles` — Bryan's six configured themes in chezmoi

Work in the linked `chezmoi` repo, in `home/dot_config/bob/config.yml`. After the
`plan:` block, add:

```yaml
# Pomodoro themes: your vocabulary and its colors (docs/pomodoro-themes.md PT12 in
# bob-cli). A name outside this list still works, but menus stop suggesting it and
# capture, bob plan, and the ledger mark it. Colors are palette slots (blue, orange,
# aqua, yellow, magenta, green, violet, red) or auto; they win over
# `bob themes color` pins. Colors were picked so the themes that share the most days
# sit in different hue families.
themes:
  PLAN: blue
  BUILD: yellow
  READ: green
  CHORE: magenta
  WRITE: violet
  FIX: red
```

Leave `plan.exempt: [PLAN]` as it is: PLAN still does not count toward
`plan.max_themes`, and its config color simply replaces the exempt mute.

From a bob-cli checkout that contains phase `config`, run
`BOB_CONFIG_FILE=<the edited chezmoi source file> cargo run -q -- themes list`. Confirm
the six rows are labeled `config` with the chosen colors, there are no config warnings,
and the header reads `6 configured`.

Older bob builds ignore the unknown `themes` key, so this is safe to land before the
release train ships. After the commit, run `chezmoi update -a --force`, as the chezmoi
repo's AGENTS.md requires.

## Rollout notes

- bob-cli ships through the daily release train. Never edit the `Cargo.toml` version or
  `CHANGELOG.md`.
- Use conventional commit types, for example `feat(themes): …`. Nothing here breaks a
  CLI or JSON contract, so no `!`.
- Live behavior arrives when the train installs the new bob, after `bob plugins sync`,
  and after a Mac app rebuild. Until then, each client keeps today's behavior.
