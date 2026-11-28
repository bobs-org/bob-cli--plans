---
tier: tale
title: Cabinets start closed, configurable per card type and cabinet
goal:
  Every property-card cabinet starts closed on a note's first paint in a session unless
  Bob's config.yml `cabinets:` block opens it for that card type, and Bryan's chezmoi
  config opens only the album Schema cabinet.
size: medium
decisions:
  problem_reveal:
    ask:
      Keep cabinets closed on first load even when their report names a problem (stale
      catalog, bad schema)?
    choices:
      header: Problems show only in the closed header's chip, summary, and Sync action
      reveal:
        A new or changed problem still opens its cabinet until you close it, as today
    default: header
    why:
      You asked for all cabinets collapsed; today a stale catalog alone opens Areas or
      Projects
    answer: header
  memory_glossary:
    ask:
      Correct glossary:cabinet (and bank-card's Setup line) to say cabinets start closed
      unless config opens them?
    memory:
      - glossary
    default: true
    corrects:
      Records starts open, the other content cabinets start closed, and Outline, Schema,
      and Setup open themselves for a new problem.
    answer: true
  memory_decision_record:
    ask:
      Add a decisions record that cabinet first-load state lives in config.yml per card
      type and cabinet?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.1hk
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-27 19:56:44
status: wip
---

# Plan: Cabinets start closed, configurable per card type and cabinet

## Goal

Every property-card cabinet starts **closed** the first time a note paints in a session.
A new `cabinets:` block in `~/.config/bob/config.yml` opens (or closes) chosen cabinets
per **card type** and per **cabinet**. Bryan's chezmoi config opens only the Schema
cabinet on album cards.

Open or closed stays session-only view state. It is never frontmatter, never a vault
write, and never a bob spawn. This plan changes only which state a cabinet starts in
when the session has no choice for that note and cabinet. Absorption is untouched
(`decisions:cabinets-absorb-catalogs-at-render-time`).

## Repositories

Open each repository with `/sase_repo` (`bob-plugins`, `chezmoi`) and use the printed
path. bob-cli is the primary checkout.

| Repo        | What changes                                                                                                                                                                                   |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob-plugins | `bob-ledger-tools` config loader, default resolution, Properties foot state, tests, docs, manifest version                                                                                     |
| bob-cli     | `docs/reference/configuration.md`, memory notes, regenerated glossary page. No Rust changes, no `Cargo.toml` version or `CHANGELOG.md` edits (`decisions:bob-cli-releases-ride-a-daily-train`) |
| chezmoi     | `home/dot_config/bob/config.yml` gains the `cabinets:` block                                                                                                                                   |

## Where defaults come from today

Five different state owners resolve a cabinet's first-load state. They live in
`plugins/bob-ledger-tools/src/`:

| Cabinet(s)                                  | State owner                                                                                                                                                                      | Default today                                                                                                                                                                                                                             |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Areas, Projects, Notebooks, Albums, Records | `240wb-plugin-cabinet-state.js` `propertyCardCabinetIsOpen`, called from `240wl-plugin-cabinet-stack.js` `propertyCardCabinetBuildProviderPanel` (which already knows `species`) | `PROPERTY_CARD_CABINET_KINDS[kind].defaultOpen` in `240wa-plugin-cabinet-model.js`: Records `true`, the rest `false`; provider `problems` (for example `stale`, `report-stale`, `damaged`) resurface it through `propertyCardFoldResolve` |
| Schema (album)                              | `241v-plugin-album-header.js` `albumHeaderIsExpanded`                                                                                                                            | `propertyCardCabinetDefaultOpen("schema", { problems })`, which is healthy-closed                                                                                                                                                         |
| Setup (bank)                                | `248c-plugin-bank-setup-hosts.js` (the `bankSetup…IsExpanded` path near the `foldDefault` lookup)                                                                                | `propertyCardSpecies("bank").foldDefault` (`false`) plus problem resurfacing                                                                                                                                                              |
| Outline (prose)                             | `249n-plugin-prose-note-card-host.js` `proseNoteCardIsExpanded`                                                                                                                  | `propertyCardSpecies("prose").foldDefault` (`false`); problems are always `""`                                                                                                                                                            |
| Properties (every card)                     | `240p-plugin-property-card-foot-state.js` (`propertyCardFootIsOpen`, `propertyCardFootToggle`, `propertyCardFootSetOpen`); stored as "present = open", deleted when closed       | always closed; path-only, so it cannot know the card type                                                                                                                                                                                 |

Problem resurfacing is inside `propertyCardFoldResolve` in
`240s-plugin-property-card-fold.js`. The card details folds (record sheet, home card,
notebook, ref) use the same function. Those details folds are not cabinets, so they stay
as they are.

## Config contract

```yaml
# Property-card cabinets (bob-ledger-tools; "Defaults" in bob-plugins
# docs/cabinets.md). Every cabinet starts closed the first time a note shows
# in a session. Map a card type (or `all`) to cabinets (or `all`) and `open`
# or `closed`. The most specific entry wins, and a card-type entry beats an
# `all` one. bob-cli ignores this block.
cabinets:
  album:
    schema: open
```

- **Card-type keys** are the property-card kinds from `PROPERTY_CARD_KINDS`
  (`240f-plugin-property-card-species.js`): `album`, `record`, `day`, `week`, `quarter`,
  `year`, `bank`, `area`, `project`, `notebook`, `prose`, `ref`. Add the reserved key
  `all`. Accept `article` as an alias of `prose`
  (`decisions:prose-notes-are-owned-prose`).
- **Cabinet keys** are the `PROPERTY_CARD_CABINET_KINDS` keys: `areas`, `projects`,
  `notebooks`, `albums`, `records`, `outline`, `schema`, `setup`, `properties`. Add the
  reserved key `all`.
- **Values** are `open` or `closed`. Trim and compare case-insensitively.
- **Precedence**, first match wins: (type, cabinet), then (type, `all`), then (`all`,
  cabinet), then (`all`, `all`), then the built-in `closed`.
- **Validation.** If `cabinets` is not a mapping, ignore the whole block. If a type
  value is not a mapping, or a key or value is unknown, ignore only that entry and keep
  the rest. Log one `console.warn` per config fingerprint that names the bad paths (for
  example `cabinets.album.schma`). Never throw or show a Notice. A valid pair that a
  card never shows (for example `area.records`) is harmless and does not warn.
- **Platforms.** Mobile (`Platform.isDesktopApp === false`), a missing `fs`, a missing
  file, or a YAML parse error all fall back to the built-in `closed`.
- **Live edits.** Cache by the config's stat key, the same way `loadPlanCaps` does
  (`planConfigPath`, `planRequireOptionalNodeModule`, `planCapsStatKey`, and `parseYaml`
  in `320-plan-lane-and-config.js`). An edit reaches a cabinet with no session choice on
  its next paint. A session choice always wins over config.

## Implementation (bob-plugins, `plugins/bob-ledger-tools`)

Edit `src/` fragments only. Add any new fragment to `src/fragments.json`. Keep each
hand-edited fragment at or below 1000 lines. Run `npm run build`; never hand-edit
`main.js`.

1. **Config loader, new fragment.** Suggested name: `240wn-plugin-cabinet-defaults.js`,
   listed after `240wa` in the manifest. Add:
   - `loadPropertyCardCabinetDefaults(options = {})`. It returns
     `{ rules, invalid, warnings, configPath }` and takes the same injectable
     `configPath`, `env`, `osModule`, `fsModule`, `parseYaml`, and `Platform` options as
     `loadPlanCaps`. Add a stat-keyed cache plus
     `resetPropertyCardCabinetDefaultsCache()` for tests.
   - `propertyCardCabinetConfiguredOpen(species, kind, options = {})`. It returns a
     boolean through the precedence ladder. An unknown or empty `species` uses only the
     `all` rows.
2. **Built-in defaults** (`240wa-plugin-cabinet-model.js`). Every kind defaults to
   closed. Remove the Records `true` and the `"healthy-closed"` sentinel (or set every
   `defaultOpen` to `false`). `propertyCardCabinetDefaultOpen(kind, options)` reads
   `options.species` and returns
   `propertyCardCabinetConfiguredOpen(options.species, kind, options)`.
3. **Content cabinets.** In `240wl-plugin-cabinet-stack.js`, pass `species` into every
   `propertyCardCabinetIsOpen` and `propertyCardCabinetSetOpen` call in
   `propertyCardCabinetBuildProviderPanel`. In `240wb`, `propertyCardCabinetIsOpen`
   passes `options.species` to `propertyCardCabinetDefaultOpen`. Grep for any other
   `IsOpen` callers and for exports in `374k-exports-cabinets.js`.
4. **Schema.** In `241v-plugin-album-header.js` `albumHeaderIsExpanded`, use
   `foldDefault: propertyCardCabinetDefaultOpen("schema", { species: "album", problems })`.
5. **Setup.** In `248c-plugin-bank-setup-hosts.js`, replace the
   `propertyCardSpecies("bank").foldDefault` lookup with
   `propertyCardCabinetDefaultOpen("setup", { species: "bank" })`.
6. **Outline.** In `249n-plugin-prose-note-card-host.js` `proseNoteCardIsExpanded`,
   replace the prose species `foldDefault` with
   `propertyCardCabinetDefaultOpen("outline", { species: "prose" })`. First confirm that
   the prose fold is always the Outline cabinet: `prose` maps to `outline` in
   `PROPERTY_CARD_CABINET_FOLD_KINDS`. If a species `foldDefault` for album, bank, or
   prose then has no readers, delete it. Keep the ones the details folds still read.
7. **Properties per card type.** Change `240p-plugin-property-card-foot-state.js` to
   store explicit booleans; a stored `false` now means "closed by choice".
   - `propertyCardFootIsOpen(path, species)`: a stored boolean wins. Otherwise use
     `propertyCardCabinetConfiguredOpen(species, "properties")`.
   - `propertyCardFootToggle(path, species)` flips the effective state and stores it.
     `propertyCardFootSetOpen` stores what it is given.
   - Rename carries the stored boolean; delete drops it.
   - Mounts record their `species` (through `propertyCardFootRegisterMount` /
     `propertyCardFootAdapterTrack`). That lets `propertyCardToggleProperties(view)` and
     `-` toggle against the right default.
   - Thread `species` through the shared adapter helpers in
     `240r-plugin-property-card-foot-adapter.js` (`…Extra`, `…Handlers` `onToggle`,
     `…Append`). Also thread it through every adapter: `241va` album, `242kc` record
     sheet, `248ca` bank, `249e` home card (area or project, from the model), `249j`
     notebook, `249o` prose, `249t` ref, and `296da` intention card (day, week, quarter,
     or year from the period kind).
   - A missing species uses the `all` rows only. Each foot digest or signature that
     includes `footOpen` must still change when the effective state changes.

> [!decision] problem_reveal = header Cabinets stop resurfacing on problems. In `240wb`
> (`propertyCardCabinetIsOpen`), `241v` (`albumHeaderIsExpanded`), and `248c` (bank
> Setup), resolve open state from the session choice, else the configured default, and
> ignore the problem signature. One way to do this is to call `propertyCardFoldResolve`
> with `problems: ""` after the restamp. Problems still feed each header's attention
> chip, summary, `data-cabinet-problems`, and actions such as Sync. Leave
> `propertyCardFoldResolve`'s resurfacing in place for the non-cabinet details folds
> (record sheet, home card, notebook, ref). An invalid or empty schema stays closed
> unless it is configured open, and its header summary still names the first diagnostic.

> [!decision] problem_reveal = reveal Keep `propertyCardFoldResolve`'s resurfacing for
> every cabinet. Only the no-problem default changes to the configured value. Docs keep
> the "a new problem opens it until you close it" wording, now stated for every cabinet
> whose report names problems.

8. **Version.** Bump the `bob-ledger-tools` manifest minor version and regenerate the
   docs inventory as the repo's recent commits do (for example `0a1b1170`, which also
   touched `package.json` and `docs/index.md`). `npm run docs:inventory:check` must
   pass.

## Tests (bob-plugins `scripts/`)

- Add `scripts/test-ledger-tools-cabinet-defaults.cjs` and register it in `package.json`
  `test:unit`. Cover these cases with injected `fsModule` and `parseYaml`, never Bryan's
  real config:
  - missing file, mobile, non-mapping block, and parse error all give closed everywhere
  - each rung of the precedence ladder
  - the `article` alias
  - trim and case on values
  - a bad key or value is ignored on its own while its siblings still apply, and logs
    one warning per fingerprint
  - the stat cache refreshes on an mtime or size change
  - `propertyCardCabinetDefaultOpen` returns `false` for every kind when no config is
    present
- Update the tests that pin today's defaults:
  - `test-ledger-tools-cabinets-kit.cjs` (around lines 143–146)
  - `test-ledger-tools-album-header.cjs` (schema default; invalid or empty schema per
    `problem_reveal`)
  - `test-ledger-tools-cabinets-records.cjs` (Records starts closed)
  - `test-ledger-tools-cabinet-host-catalogs.cjs` (stale or damaged catalogs per
    `problem_reveal`)
  - `test-ledger-tools-bank-setup*.cjs` (attention per `problem_reveal`)
  - the prose card tests
  - `test-ledger-tools-property-card-foot*.cjs`: a configured-open Properties cabinet,
    an explicit close beating a configured open, rename carrying a stored `false`, and
    `-` toggling against the species default
- Confirm that `scripts/hermetic-bob-env.cjs` points `XDG_CONFIG_HOME` away from the
  real home, so the unit suite never reads `~/.config/bob/config.yml`. If it does not,
  make the new loader's tests inject paths.
- Update the expectations in the real-Obsidian cabinets runner
  (`scripts/obsidian-cabinets-e2e*.cjs`, `scripts/test-obsidian-cabinets-e2e.cjs`) that
  assume Records starts open. Follow `docs/develop/validation.md` on whether this change
  needs a real-Obsidian run.
- Gate: `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`.

## Docs

- **bob-plugins `docs/cabinets.md`.** Rewrite "Defaults": every cabinet starts closed.
  Add a "Configuring defaults" subsection with the YAML above, the key vocabulary,
  precedence, validation, live-edit behavior, and the fact that a session choice wins.
  Remove "healthy-closed" wording per `problem_reveal`.
- **bob-plugins `docs/albums.md`.** Fix "Records starts open over a closed Schema"
  (around line 97), and the invalid and empty schema "opens itself" sentences (around
  lines 109–112) per `problem_reveal`.
- **bob-plugins `docs/bob-ledger-tools.md`.** Fix "Healthy Setup stays closed; attention
  opens it" (around line 121) per `problem_reveal`.
- **bob-plugins `docs/prose-card.md`.** Line 36 says Outline is closed by default. Add
  "unless `cabinets:` opens it" and link the cabinets page.
- **bob-cli `docs/reference/configuration.md`.** Add a `### cabinets` section after
  `### copy`. Use the same table shape: key `cabinets.<type>.<cabinet>`, type `open` or
  `closed`, default closed, effect "first-load state of property-card cabinets; the CLI
  ignores it". Add the YAML example and a link to `plugins/cabinets.md`. Run
  `just docs-check`.

## Chezmoi

In `home/dot_config/bob/config.yml`, add the commented `cabinets:` block from "Config
contract" with only `album: { schema: open }`. Put it next to the other plugin-only
block, `copy:`. Follow the chezmoi repo's gotcha: once the commit lands, run
`chezmoi update -a --force` so `~/.config/bob/config.yml` updates. bob-cli already
tolerates unknown top-level keys (`copy:` proves it), and older ledger-tools builds
ignore the block.

## Memory (bob-cli `sase/memory/`)

Use `/sase_memory_write`, then `sase memory init`, then `just docs-gen`. That
regenerates `docs/reference/glossary.md` from the glossary web. Finish with
`just docs-check`.

- `memory_glossary`: in `glossary/cabinet.md`, replace "Records starts open, the other
  content cabinets start closed, and Outline, Schema, and Setup open themselves for a
  new problem." with this: every cabinet starts closed on a note's first paint in a
  session unless the `cabinets:` block in Bob's config opens it for that card type. Add
  the problem clause that matches `problem_reveal`.
- Same decision, only under `problem_reveal = header`: in `glossary/bank-card.md`,
  reword "Setup folds when healthy" so it no longer implies attention opens Setup. Under
  `reveal`, leave it unchanged.
- `memory_decision_record` (only if yes): add a decisions strand. Claim: first-load
  cabinet state comes from `cabinets:` in Bob's config.yml, per card type and cabinet,
  and is closed unless configured; a session choice wins. Rejected alternatives:
  - Obsidian plugin settings: the vault's `data.json` is not chezmoi-managed.
  - Frontmatter: cabinet state is never frontmatter.
  - Persisting state across sessions. Evidence: this plan.

## Deploy and verify

1. After the bob-plugins commit, run `bob plugins sync`. Follow bob-plugins `AGENTS.md`
   if the sync refuses to overwrite, and never pass `-F` on your own.
2. Once Bryan's config is applied, check in Obsidian Live Preview:
   - An album note shows Schema open and Records closed.
   - Area, project, notebook, prose, bank, ref, record, and day notes show every
     cabinet, including Properties, closed.
   - Toggling a cabinet holds for the session and across revisits.
   - Editing `cabinets:` changes a cabinet with no session choice on its next paint.

## Out of scope

- Card details folds (record sheet, home card, notebook, and ref `_` folds when no
  cabinet sits above Properties). They are not cabinets.
- The Managed sub-fold inside Properties.
- Persisting open state across sessions.
- Changes to the `_` pick, heading-link reveal, or absorption.
