---
tier: epic
title: "Record Card: an editable value sheet under the H1 of every record note"
goal:
  "Every healthy record note shows its album-defined fields as a quiet, beautiful record
  sheet under the H1 and above ## Notes, in bob's vocabulary. Every value is editable
  from a keyboard-first Record Card (== , Ctrl+Shift+P, palette, Command Mode, or a
  click) through one guarded bob writer, with Undo. Native Properties hide only when the
  sheet shows every key and bob can write."
phases:
  - id: record-read
    title: Per-record read and record fingerprint in bob records show
    depends_on: []
    size: medium
    description:
      "record-read: add an optional STEM to bob records show that returns one record's
      typed properties with source lines, identity and other keys, findings, full ranked
      suggestions, and a semantic record fingerprint, read-only, with human and JSON
      output and docs."
  - id: record-envelope
    title: Guarded JSON envelope for bob records update
    depends_on:
      - record-read
    size: medium
    description:
      "record-envelope: add bob records update -i FILE, a guarded envelope of typed
      field.set/field.unset ops with an expect fingerprint, inverse ops, group
      consequence, and typed refusals (stale_record repaints in one call), plus an
      albums list capability flag, docs, tests, and just install."
  - id: record-sheet
    title: Read-only record sheet under the H1 (Live Preview and Reading)
    depends_on: []
    size: large
    description:
      "record-sheet: render the record sheet after the first H1 of canonical album
      records from metadataCache and the cached album report (no subprocess on open),
      covering values by kind, lint, ghost chips, a read-only fold, a created footer,
      collapse, subscription docking, light/dark/375px, and a tested hide gate that is
      not yet applied."
  - id: record-session
    title: Record update service and guarded session with direct apply
    depends_on:
      - record-sheet
      - record-envelope
    size: medium
    description:
      "record-session: plugin service for records show/update, a record session over the
      shared guarded session with an opt-in direct-apply path, inverse-op Undo, pane
      saves, auto-sync echo suppression, an optimistic sheet overlay, and a real-bob
      writer matrix."
  - id: record-card
    title: Record Card shell, browse keys, choices, and discrete actions
    depends_on:
      - record-session
    size: large
    description:
      "record-card: keyboard-first Record Card modal in the Task Card / Schema Card
      idiom with browse navigation, the choices pick list, digits, t, Ctrl+D, Undo, the
      YAML excursion, = to the album Schema Card, the status rule, hint strip, All
      actions menu, and live region."
  - id: record-stages
    title: Typed Record Card stages and Tab fill-out
    depends_on:
      - record-card
    size: large
    description:
      "record-stages: text, suggest combobox, date, number, URL, link, and list stages
      with debounced bob previews, a latched Enter, Tab and Shift+Tab save-and-advance,
      and taught refusals for required, invalid, and stale edits."
  - id: record-entry
    title: Entry points, == on records, and the Properties hide
    depends_on:
      - record-stages
    size: medium
    description:
      "record-entry: wire the sheet's Edit, row, chip, ghost, and missing affordances;
      == on records; the edit-record palette command; Ctrl+Shift+P in navigation
      hotkeys; the Command Mode row; and switch on the fail-open Properties hide (and
      the inline-title decision)."
decisions:
  inline_title:
    ask:
      Should record notes hide Obsidian's filename-derived inline title while the sheet
      replaces Properties?
    choices:
      hide:
        Pages open on the H1 and the sheet; rename via the tab header or file explorer
      keep:
        The snake_case inline title stays above the H1, like album notes; the title
        shows twice
    default: hide
    why:
      The H1 already titles the record; the stem above it is the loudest leftover noise
    answer: hide
  decision_record:
    ask:
      Should the land agent record a decisions strand that record values are edited only
      through bob?
    memory:
      - decisions:record-values-edited-through-bob
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.55.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 11:29:50
status: wip
---

- **PROMPT:**
  [prompts/202611/record_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_card.md)

# Record Card: an editable value sheet under the H1 of every record note

## Scope and completion owner

Record notes are the last Bob surface that shows record data in the wrong vocabulary.
The album index, the Mac Record Card, `bob records show`, and the `[!records]` callout
say `✍️ Draft` and `🔮 Future Work`. The note itself shows Obsidian's raw Properties
rows instead: `status: draft`, `section: future`, a linked `type: record`, and an ISO
`created` (see Bryan's screenshot `~/tmp/screenshots/20261117_101208.png`, the
`sase_blog_zettel/bead_lifecycle.md` record). Properties edits have no schema: no choice
picker, required marks, bounds, or URL checks. `+ Add property` invites
`record_unknown_field`, and every Properties save re-serializes the whole frontmatter.

This epic implements the user-endorsed consolidated research
`research:202611/record_card/record_card.md`. Bryan agreed with every requirement it
recommends. Read it with `sase artifact read` before starting any phase. It is the
design authority, and its "The design" section is normative wherever this plan does not
override it. The overrides are listed under [Settled design](#settled-design). Also read
`plan:202611/double_equals_schema_keymap.md`: `==` landed as bob-plugins `2b61e76` with
its `album_only` records decision, and this epic amends that shipped behavior.

Each phase is bounded to one coding agent. This epic's land agent owns the integrated
proof and closes only this epic's own beads. Phase workers never create task beads. They
append discoveries as `PROPOSED FOLLOW-UP:` notes on their assigned phase bead.

## Starting evidence and repository discipline

Inspected bases: bob-cli `f6924787`, bob-plugins `2b61e76`. Verified there:

- **bob-cli.** `bob records show` takes only `TYPE` (`src/native/records/cli.rs`,
  `run_show`) and caps suggestions at five. `bob records update` (`execute_update`)
  plans with `plan::plan_update`, edits only YAML spans, and rebuilds the album index in
  one `write_staged_files_guarded` batch. It takes no maintenance lock, makes no commit,
  and has no envelope, `expect`, or inverse ops. A no-input `plan_update` already
  returns every schema property with `state`, `current`, and `display`. Album
  `schema_fingerprint` comes from `albums::report::schema_fingerprint`. The
  `bob albums update` envelope (`src/native/albums/update/protocol.rs`) is the parsing
  precedent: duplicate-key refusal, an unknown-key/version refusal, and `-i FILE` with
  `-` meaning stdin.
- **bob-plugins `plugins/bob-ledger-tools`.**
  - Album header chrome: `241v-plugin-album-header.js` is a self-contained mixin with a
    Prec.highest ViewPlugin widget after the first H1 (`notebookHeaderLiveAnchor`), a
    Reading post-processor at priority 49, a scoped surface class, and refresh through
    `apiAlbumSubscribe`. Clone this pattern.
  - Subscription strip: `241hg-plugin-bank-subscription-header.js` mounts its own
    record-note strip at the same anchor.
  - Guarded session: `242aa-plugin-guarded-update-session.js` is writer-agnostic and
    preview-gated. Album and bank wrap it in about 60 lines each (`242b`, `248e`).
  - Entry points and YAML: `242d-plugin-album-entrypoints.js` holds the loose
    `albumSchemaEntryKindOf` (any unknown type counts as `"record"`) and the parent
    resolver. `242e` owns the YAML excursion and already takes a `kind`.
  - `==`: `248m-plugin-schema-double-equals.js` maps `==` to `"album"` or `"bank"` only.
  - Other plugins: `bob-navigation-hotkeys/src/551-plugin-album-schema-fallback.js` (the
    Ctrl+Shift+P album fallback) and `bob-command-mode/src/160-sheet-rank.js`
    (`CM_SHEET_SURFACE_ROWS`).
  - CSS hides only one property row per surface (`styles.css`
    `.bob-album-header-surface .metadata-property[data-property-key="record"]`).
  - The album report is fetched by subprocess only, never persisted, so mobile has no
    report.
- **Vault.** All 186 records use canonical `type: "[[record]]"`. Field kinds in use:
  choices, text, suggest text, date, number with bounds, URL, and list text
  (`subscriptions` `match`). `eat` records carry legacy keys (`lid`, `source_*`), and
  `subscriptions` declares `legacy: [recur_id]`. 91 `eat` records have no `created`, and
  some text values are wikilinks (`location: "[[nj_towns#^z-…|cranford]]"`).

Recheck history and bead notes when each worker starts; other work keeps landing.

**bob-plugins discipline.**

- Open the repo with `/sase_repo` (`sase repo open bob-plugins -r "…"`) and read its
  `AGENTS.md`.
- Edit `src/` fragments and the `src/fragments.json` order, then rebuild `main.js`;
  never hand-edit it. Every edited fragment stays at or below 1000 lines, so prefer new
  cohesive fragments. Fragment names below are suggestions; manifest order is what
  matters.
- Register new tests explicitly in `package.json` `test:unit`.
- Bump the touched plugins' minor versions. Read the current value at implementation
  time.
- Run `npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`, and
  `BOB_CLI_DIR=<the bob-cli checkout you run in> npm run docs:check`.
- Run `bob plugins sync` after changes. Follow the repo's deploy-conflict procedure, and
  never pass `-F` on your own.
- The plugin phases run strictly in sequence, so deploys never race each other.

**bob-cli discipline.**

- Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before CLI or docs
  changes.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`.
- Everything here is additive: JSON stays `schema_version: 1`, and no commit needs `!`.
- Run `just docs-gen` and the `just check` gate through `/sase_monitor`.

End every turn through `/sase_final`.

## Settled design

These points are decided. Bryan endorsed the research, and the plan resolves the
research's open questions as follows.

1. **Names.** The in-note chrome is the **record sheet**. The editing modal is the
   **Record Card**, the name the Mac Capture surface already uses for the same fields.
   "Schema Card" stays reserved for the album definition editor. The Record Card edits
   _values_. It never adds fields, reorders them, relabels choices, or touches the emoji
   slot.
2. **One writer.** All writes go through `bob records update`. There is no
   `processFrontMatter`, no JS YAML serializer, and no JS validation authority; the
   client checks values only to paint lint. Notes stay in `## Notes`, written by capture
   and `bob records update -n`.
3. **No commits per edit.** Record edits stay capture-like: one guarded staged write
   (record + album index), no lock, no commit, and vault-sync commits later. This
   deliberately differs from the album Schema Card (research resolution 4, unvetoed).
4. **Identity is not editable.** `parent` becomes the breadcrumb, `type` is implicit,
   `title` is the H1, and `created` is a quiet footer. Title rename is a v2 bob
   operation.
5. **Canonical gate.** The sheet, the hide, and `==` require the note's own type to be
   exactly `record` (`type: "[[record]]"`, read with `albumTypeNames`). The note must be
   a direct child of an album folder whose album appears in the report, and its `parent`
   must resolve to that same album. The gate never uses `albumSchemaEntryKindOf`'s loose
   fallback, which would classify `bank/2026-11.md`. This overrides the research: legacy
   noun types (`type: "[[restaurant]]"`) fail open with no sheet, since
   `bob records check` already warns about them.
6. **Fail open everywhere.** Mobile has no bob and therefore no report, so it shows
   native Properties and no sheet. So do these cases:
   - unresolved, mismatched, or cyclic parents;
   - records outside their album folder;
   - other types;
   - Source mode.

   A broken album schema shows a warning sheet and leaves Properties visible.

7. **Capability gate (new).** `record-envelope` adds
   `"capabilities": ["record_values_v1"]` to `bob albums list|check -f json`. The plugin
   enables editing and the hide only when the cached report carries that capability.
   Otherwise the sheet renders read-only with Properties visible and a quiet
   `Update bob to edit here` tooltip on Edit. This matters because the vault syncs the
   plugin to the Mac before the Mac's `bob` is reinstalled.
8. **The hide ships last (override).** `record-sheet` computes and tests the hide gate
   but applies no hiding CSS. `record-entry` switches the hide on, once every field kind
   is editable from the card. Properties therefore never disappear while the card cannot
   edit everything they could.
9. **`==` on a record opens the Record Card.** This amends the shipped `album_only`
   decision. Inside the card, a single `=` climbs to the parent album's Schema Card.
   These keep Vim's reindent: `N==`, register-prefixed `==`, Source mode, insert and
   visual modes, and non-record notes.
10. **One gesture is one write.** Discrete actions apply directly with `expect`: a
    choice pick, a digit, `t`, Ctrl+D, Undo, and picking an existing suggestion. Typed
    values first need a current debounced dry-run preview, and Enter stays latched until
    one arrives. There are no multi-field batch saves.
11. **Read on open, not on note open.** The sheet paints from `metadataCache` plus the
    cached album report and never spawns bob. The Record Card spawns one
    `bob records show TYPE STEM` when it opens. That read supplies the `expect`
    fingerprint, lines, findings, and full suggestions. The card renders rows
    immediately from the sheet model and enables writes when the read lands.
12. **Subscriptions dock.** On `subscriptions/*.md` the existing charge-timeline strip
    becomes the sheet's trailing band, and the standalone strip yields to it. Two
    widgets at the same anchor have no guaranteed order.
13. **Desktop-only editing.** The card and every write gesture follow the existing
    desktop gate (`albumHeaderDesktop`).

## Shared design contract

### The record sheet

The sheet is header chrome mounted at the end of the first H1 line in Live Preview, and
after the first non-code `h1` in Reading view (including embeds and hover popovers, like
the album header). It is never document bytes and never appears in Source mode. The
`bead_lifecycle` example:

```text
# Bead Lifecycle
┃ 📇 SASE Blog Zettel › Zettel                                  ⊞ Edit   ▾
┃
┃   Status *   ✍️ Draft
┃   Claim *    Bead triage/lifecycle needs to be thoroughly designed in v2.
┃   Section    🔮 Future Work
┃
┃   added today, 7:55 AM
## Notes
```

A sparse movie with empty optionals and legacy keys:

```text
# Equilibrium
┃ 🎬 Movies › Movie                                             ⊞ Edit   ▾
┃
┃   Status *   ✅ Watched
┃   Verdict    😍 Loved
┃   Year       2002
┃   ＋ Service  ＋ Recommended by  ＋ Watched on  ＋ URL
┃
┃   added Aug 8, 2025 · 2 other properties ▸
```

**Visual rules.**

- The sheet belongs to the album-header family: 8px radius, a 2px start border in the
  root-area hue (`notebookHostHueName`), a 1px border at 30% hue mix, and
  `font-interface`. Its background tint is about 4%, half the album's 8%, so it reads as
  a nameplate rather than a banner.
- Rows form a `dl` grid. Labels are muted UI-small text on the left; values sit on the
  right in the note's text font. That contrast does most of the beautiful work.
- A required `*` stays quiet until the field is empty.
- Fields render in album declaration order with bob's labels.

**Values by kind.** Only the selected value renders, never the choice census.

| Kind           | Rendering                                                                                    |
| -------------- | -------------------------------------------------------------------------------------------- |
| Choices        | A chip with bob's combined emoji + label, e.g. `✍️ Draft`.                                   |
| Text           | Literal text. Complete `[[target\|alias]]` spans become internal links with hover preview.   |
| Date           | `Aug 8, 2025 · 1y ago`, or `today` / `yesterday`.                                            |
| URL            | A real anchor, middle-truncated (`maps.google.com/…/ramen`), with the full URL in its title. |
| Link           | An internal link showing its display text.                                                   |
| Number         | Tabular numerals. `subscriptions`' `amount` uses the bank money formatter.                   |
| List           | One chip per item.                                                                           |
| Empty optional | One ghost-chip row of `＋ <Label>` buttons.                                                  |
| Empty required | A full row with an amber `missing` pill.                                                     |

**States that must never be confused.**

- **Valid value.**
- **Hand-typed value that fails its kind.** The raw value shows in monospace with an
  amber tag: `not a Status choice`, `not a date`, `outside 1888–2100`, `not a URL`,
  `expected one value`, or `expected a list`.
- **Missing required.**
- **Broken album schema.** A warning sheet shows the breadcrumb, the album's error
  message, and a `y` jump to the album YAML.
- **Saving.** After a Record Card apply, the changed rows flash the accent once for
  600ms; there is no flash under reduced motion.

An empty optional is distinguishable from a read failure. Lint is presentation only; bob
stays the authority.

**Footer and fold.**

- The footer shows `created` as `added today, 7:55 AM`, `added yesterday`, or
  `added Aug 8, 2025`. It is omitted when `created` is absent.
- A `N other properties ▸` disclosure lists every non-schema, non-identity key
  read-only: key, compact value, and a `legacy` tag for keys the album's `legacy`
  declares. Wikilinks render; `null` renders as `—`; lists join with `, `.
- When `title` differs from the first H1 text, `title` appears in the fold tagged
  `differs from heading`.
- Every frontmatter key has exactly one home: breadcrumb, implicit, H1, footer, row,
  ghost chip, or fold. A unit test asserts this coverage for every fixture.

**Collapse.**

- The sheet starts expanded. Collapse state is per session and per path (`▾`/`▸`).
- Collapsed, the sheet is one line: the breadcrumb, then the choice-field chips (group
  field first), then `· N fields`.

**Accessibility and narrow panes.**

- The sheet is `role="group"` with the label `<Title> record details`.
- Rows and chips are buttons named like `Status: Draft. Change status`, with
  `aria-haspopup="dialog"` once editing exists, visible focus rings, and textual state
  marks beside color.
- `y` on a focused row jumps to that key's YAML line, matching album rows.
- At 375px or narrower, labels stack above values. Verify light and dark.

**Live Preview mechanics.**

- The sheet is its own mixin and its own ViewPlugin with Prec.highest, cloned from
  `241v`.
- Compute the model and signature before building any DOM, and reuse the previous
  decoration when the signature and anchor are unchanged, so body typing never rebuilds
  the DOM.
- Refresh on these signals:
  - `apiAlbumSubscribe`;
  - `metadataCache` `changed` for mounted paths;
  - the subscription report (when docked);
  - the session's optimistic overlay.
- A missing H1 falls back to after the frontmatter, as in `241v`.

### Hiding native Properties

Hiding requires all of the following: the gate passes, the schema is healthy
(`schema_fingerprint` is non-null and the album has no error issue), the platform is
desktop, the report carries `record_values_v1`, and `record-entry` has switched the hide
on.

When it hides, it hides the whole Properties block. A class on the view's own
source-view or preview-view container drives this. CSS child combinators must match the
real Live Preview and Reading structures, verified by a CDP DOM probe, so embedded notes
keep their own Properties.

The class is removed on these events:

- file switch;
- type or parent change;
- mode switch;
- view destroy;
- plugin unload or disable.

The worst failure is raw rows reappearing, never data vanishing. With
`inline_title = hide`, the same class also hides that view's `.inline-title`.

### The Record Card

The Record Card follows the Task Card / Schema Card idiom:

- a desktop-gated modal with the `bob-key-card` chrome;
- the title `<album emoji> <record title>` and the subtitle `<Album> › <Noun> · <path>`;
- one selectable row per schema field (ghost rows for empty optionals), rendered with
  the sheet's value renderers;
- a status rule, a contextual hint strip, a `⋯ All actions` menu with a button for every
  key, and a polite live region.

The card opens on the field the entry point names, else the first missing required
field, else the first field. It stays open after saves. Typing in inputs never triggers
navigation letters, and IME composition never applies.

| Mode   | Key                                     | Action                                                                               |
| ------ | --------------------------------------- | ------------------------------------------------------------------------------------ |
| Browse | `j`/`k`, `↓`/`↑`                        | Move between fields, ghost rows included.                                            |
| Browse | `Home`/`End`                            | First or last field.                                                                 |
| Browse | `Tab`/`Shift+Tab`                       | Move like `j`/`k`, so keyboard users never get trapped.                              |
| Browse | `Enter`                                 | Open the selected field's stage.                                                     |
| Browse | `1`–`9`                                 | On a choices row, pick that choice and save.                                         |
| Browse | `t`                                     | On a date row, set today and save.                                                   |
| Browse | `Ctrl+D`                                | Clear the selected optional field. Required fields refuse with a teaching line.      |
| Browse | `u` / `Ctrl+Z`                          | Undo the last save through bob's inverse ops.                                        |
| Browse | `y` / `Ctrl+E`                          | YAML excursion to the field's line, with the Done guard.                             |
| Browse | `=`                                     | Close and open the parent album's Schema Card.                                       |
| Stage  | `Enter`                                 | Save. Discrete picks save at once; typed stages are latched until a current preview. |
| Stage  | `Tab`/`Shift+Tab`                       | Save if changed, then open the next or previous field's stage.                       |
| Any    | `Esc` / `Ctrl+[` / `q` (outside inputs) | Ladder: menu → stage → card.                                                         |

**Stages.** Stages expand inline under their row; there are no nested modals.

- **Choices:** a filterable pick list showing digit badges, emoji labels, muted album
  counts, and the current value ringed. Picking from the closed list applies directly.
- **Text:** an auto-growing single-value textarea; newlines are refused.
- **Suggest text:** a combobox over the full ranked folder values from the per-record
  read. Picking an existing value applies directly; free text previews and shows bob's
  `record_value_near` warning.
- **Date:** accepts `today`, `t`, `yesterday`, and ISO, with a parsed echo.
- **Number:** shows min/max hints.
- **URL:** validated by bob.
- **Link:** a vault note suggester accepting `[[wikilink]]` or a bare name.
- **List:** a chip editor. Enter on an empty input saves; Backspace on an empty input
  removes the last chip. Values go to bob as a JSON string array, never comma-joined.

**Empty values.** An explicitly empty optional clears the key. An empty required field
refuses: `Status is required — pick one of 6 choices`. An update never silently refills
a default.

**Status rule.** It shows one of these lines:

- the loading line `Reading record…`;
- the preview consequence, e.g.
  `Status ✍️ Draft → ✅ Ready · moves to the ✅ Ready group`;
- `✓ Saved · u undo`;
- a refusal, using bob's message and hint verbatim;
- the stale repaint:
  `This record changed elsewhere — refreshed; your draft is kept · ↵ retry`.

**Staleness and Undo.**

- On a vault modify of the record or its album, the card re-reads, rebases `expect`, and
  keeps any open draft.
- Undo memory is per record path for the plugin session, so reopening the card can still
  undo.
- `=` and `y` close the card first. The YAML excursion generalizes `242e` with kind
  `"record"`: Done saves, restores the prior view state, and reopens the card on the
  same field. Done never blocks on record findings; the card paints them.

### The bob contract

The contract is additive, `schema_version: 1`, and follows `cli_rules`.

```text
bob records show TYPE [STEM] [-b DIR] [-f human|json]
bob records update TYPE STEM [-b DIR] [-d] [-f human|json] [-h] [-i FILE] [-n TEXT]... [-o] [-s KEY=VALUE]...
```

**Record fingerprint.** `sha256:` plus 64 lowercase hex digits over a canonical
serialization of two inputs:

- the parsed frontmatter mapping (canonical JSON with sorted keys, typed scalars kept
  distinct);
- the album `schema_fingerprint`.

Body bytes are excluded, so typing under `## Notes` never makes a status change stale.
YAML formatting and key order do not matter. File-level races remain caught by the
existing preimage guards. Put the fingerprint in its own small module so `show` and
`update` share it.

**`show TYPE STEM`.** It resolves the record exactly as `update` does
(`resolve_update_target`). The output keeps today's schema payload and adds:

- `fields[].values`: the full ranked suggestions for suggest fields (`top_values` stays
  capped at five).
- a `record` object:

```json
{
  "stem": "bead_lifecycle",
  "path": "sase_blog_zettel/bead_lifecycle.md",
  "title": "Bead Lifecycle",
  "heading": "Bead Lifecycle",
  "created": "2026-11-17T07:55:39-0500",
  "fingerprint": "sha256:…",
  "schema_fingerprint": "sha256:…",
  "frontmatter_end_line": 9,
  "group": { "field": "status", "value": "draft", "label": "✍️ Draft" },
  "properties": [
    {
      "key": "status",
      "label": "Status",
      "kind": "choices",
      "required": true,
      "list": false,
      "state": "set",
      "value": "draft",
      "display": "✍️ Draft",
      "raw": "draft",
      "line": 5
    }
  ],
  "identity": [{ "key": "parent", "line": 2, "value": "[[sase_blog_zettel]]" }],
  "other": [{ "key": "lid", "line": 11, "value": null, "legacy": false }],
  "findings": [
    { "code": "…", "severity": "error", "message": "…", "field": "status", "line": 5 }
  ]
}
```

- `state` is one of:
  - `set`;
  - `missing` (empty required);
  - `none` (empty optional);
  - `invalid` (the stored value fails validation; `raw` keeps it).
- `line` is 1-based, or null when the key is absent.
- `findings` are this note's `records check` diagnostics.

`show TYPE STEM` is read-only: it never writes, repairs, locks, or commits. A missing
stem, a parent mismatch, or a record outside its folder refuses with exit 2 and a typed
JSON `error` (`code`, `message`). It never impersonates an empty record. Human output is
a small colored card: title, album crumb, label/value rows, missing marks, findings, and
the other-key count.

**`update -i FILE`.** `-` reads stdin. The flag conflicts with `-s` and `-n`; without
`-i`, today's behavior is byte-identical. The envelope:

```json
{
  "schema_version": 1,
  "expect": "sha256:…",
  "ops": [
    { "op": "field.set", "field": "status", "value": "ready" },
    {
      "op": "field.set",
      "field": "match",
      "value": ["ANTHROPIC* CLAUDE", "CLAUDE.AI SUBSCRIP"]
    },
    { "op": "field.unset", "field": "url" }
  ]
}
```

**Envelope validation.**

- Parsing happens once, with the album-update protocol rules: duplicate keys, unknown
  keys, unknown versions, unknown ops, and wrong types all refuse.
- Values are typed JSON: strings for text, choice keys, dates, links, and URLs; a number
  or numeric string for numbers; a string array for list fields. List elements are
  literal and never comma-split. An empty string in `field.set` refuses and points to
  `field.unset`.
- Each field appears at most once per envelope. An empty `ops` array is a valid no-op.
- Ops feed the existing `plan_update` validation, the span writer, and the in-batch
  index rebuild; no second validator is added. Defaults never apply.

**Envelope output.** Dry run, apply, no-op, and refusal all return one JSON document
with these fields:

- `ok`, `command: "update"`, `dry_run`, `type`, `stem`, `path`;
- `changed[]`;
- `properties[]` (today's shape);
- `changes[]` (`key`, `label`, `before_display`, `after_display`);
- `fingerprint_before`, `fingerprint_after`;
- `inverse_ops` (`field.set` with the typed prior value, or `field.unset`, in reverse
  order);
- `group_before` and `group_after` (`{field, value, label}` or null);
- `consequence` (one human sentence);
- `album_index` (when it changes) and `warnings[]`;
- a nullable typed `error` (`code`, `message`, `field`, `hint`).

**Error codes.** `stale_record` (it also returns the current `record` object from the
per-record read, so the card repaints in one call), `record_invalid_value`,
`record_required_empty`, `record_unknown_field`, `envelope_*`, and the existing
target-resolution codes. Exit codes: 0 for success or no-op, 2 for refusals, 1 for
runtime or guard failures. The `expect` check runs before planning. The preimage guards
stay in place.

### Reliability rules

| Situation                                                           | Behavior                                                                                                                                                           |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Unsaved edits in a pane showing the record or its album             | The session saves those panes first (`albumSchemaSaveOpenEditors`). bob edits only YAML spans, and the editor reloads in place.                                    |
| Concurrent edit (another host, Source mode, Bases cell, Properties) | Fingerprint mismatch: `stale_record`, nothing written, the card repaints, the draft is kept.                                                                       |
| `bob albums sync` races a save                                      | A preimage guard refuses one side; Enter retries.                                                                                                                  |
| The record write echoes through metadataCache                       | The record and album paths are registered as own writes, so no automatic `albums sync` is scheduled. The album report is invalidated so groups and counts refresh. |
| metadataCache lags after a save                                     | The session pushes the applied properties into a per-path sheet overlay until the cache matches, or for at most 5s.                                                |
| Old bob without `record_values_v1`                                  | The sheet is read-only, Properties stay visible, and the card explains `Update bob (just install) to edit records here`.                                           |
| Plugin disabled or unloaded mid-flow                                | Surface classes are removed and Properties return. A started apply settles and reports. Late async responses are ignored.                                          |
| Presentation                                                        | Rendering, navigating, expanding, folding, cancelling, and opening the card are byte-neutral. Acceptance verifies this by hash.                                    |
| Popout windows                                                      | Main-window listeners only, matching the shipped `248m` / bob-vim-surround scope.                                                                                  |

## record-read

**Goal.** bob-cli gains the per-record read and the shared record fingerprint, per the
contract above.

**Build.**

- Add a fingerprint module under `src/native/records/` and add the optional `STEM`
  positional to `build_cli_show` with a value hint. Update the help text and examples in
  clap.
- In `run_show`, branch when `STEM` is present:
  1. Resolve with `plan::resolve_update_target`.
  2. Read the file and run a no-input `plan_update` to get properties.
  3. Add the per-key 1-based frontmatter lines, identity and other keys (with the
     `legacy` flag from the album's legacy patterns), the H1 heading, `created`, this
     note's check findings (reuse `check.rs` logic filtered to the path), `group`, full
     ranked suggestions, and the fingerprint.
- Mark stored values that fail validation as `state: "invalid"` with `raw`.
- Typed JSON errors use the existing `report_error` path with an added `code`.
- Human output follows the colored-output convention.

**Tests.** Rust unit tests plus the existing records CLI test pattern. Cover:

- a healthy record and a sparse record (`none` states);
- missing required and an invalid stored choice;
- legacy keys, a list field, CRLF, and BOM;
- `title` that differs from the H1;
- an unknown stem, a parent mismatch, and a record outside its folder;
- read-only behavior: vault bytes and HEAD unchanged;
- fingerprint stability across YAML formatting, key order, and body edits, and change on
  any value or schema change.

**Docs.** Update `docs/records.md` (`bob records` table and a "Reading one record"
subsection), add a `bob records show` row to `docs/reference/json-outputs.md`, and run
`just docs-gen` for the CLI reference.

**Verify.** `just check` through `/sase_monitor`, plus a read-only spot check against
the live vault (`bob records show zettel bead_lifecycle -f json`, a `subscriptions`
record, and a legacy `eat` record).

## record-envelope

**Goal.** `bob records update -i FILE` is the guarded, typed writer the Record Card
needs, and `bob albums list|check -f json` advertises
`capabilities: ["record_values_v1"]`.

**Build.**

- Add a `protocol` submodule for records (or a sibling of `cli.rs`) modeled on
  `albums/update/protocol.rs` but much smaller. Ops are `field.set` and `field.unset`
  only.
- In `execute_update`'s envelope branch:
  1. Read the envelope; check `expect` against the fingerprint of the current bytes.
  2. Convert the ops into planner inputs without comma splitting. Add an input variant
     if the current `RecordFieldInput` path splits commas.
  3. Plan, then compute the changes, group before/after, consequence, and inverse ops.
  4. Dry-run or install through the unchanged `write_staged_files_guarded` batch.
- `stale_record` embeds the current per-record read.
- Add `-i/--input` alphabetically with help, an example, and `conflicts_with` for `set`
  and `note`.

**Tests.** Cover:

- set, unset, list replace, and a no-op;
- empty `ops`;
- stale `expect`, invalid choice, required empty, and unknown field;
- an empty-string set;
- duplicate field, duplicate JSON key, unknown op, and wrong version;
- a typed number versus a numeric string;
- a list element containing a comma;
- dry run writing nothing;
- index rebuilt in the same write, with all other bytes identical (CRLF and BOM
  preserved);
- inverse ops round-tripping to the original values;
- classic `-s`/`-n` behavior unchanged;
- the capability flag present in list and check JSON.

**Docs.** In `docs/records.md`, add an "Updating with an envelope" subsection with the
envelope, ops, output, error codes, and Undo replay (`expect` = `fingerprint_after`,
`ops` = `inverse_ops`). Add rows to `json-outputs.md` and mention the capability in the
album reports section. Run `just docs-gen`.

**Land and install.** After landing, run `just install` from the landed tree on apollo
so the plugin phases and their real-bob tests use the new verbs.

## record-sheet

**Goal.** The read-only record sheet renders on every gated record, per the sheet
contract. The hide gate is computed and tested, but no hiding CSS is applied. Properties
stay visible, so for one phase the sheet and Properties coexist.

**Build** in `plugins/bob-ledger-tools` (suggested fragments):

- `242i-plugin-record-sheet-model.js` (pure):
  - `recordSheetGate({ path, frontmatter, report, desktop, metadataCache, vault })` →
    `{ mount, hideEligible, editable, state, reason }`. Reuse `albumTypeNames` and
    `albumResolveSchemaTargetForFile`. Also check folder == album stem.
  - `recordSheetModel(...)`: rows, ghosts, fold, footer, crumbs, lint, and coverage.
  - Value formatters by kind, plus the signature function.
  - An overlay merge hook (`overlay` argument, unused until `record-session`).
- `242j-plugin-record-sheet-dom.js`: the DOM builder and value renderers. Every report
  or frontmatter string goes in as a text node; wikilinks become anchors with
  `data-href` and hover.
- `242k-plugin-record-sheet.js`: a `BobLedgerToolsRecordSheetMixin` cloned from `241v`.
  It owns setup/teardown, the Live Preview extension, the Reading post-processor, the
  refresh fan-out (report, metadataCache `changed`, subscription report), per-path
  collapse and fold maps, and a scoped `bob-record-sheet-surface` class used for styling
  only.
- Subscription docking: the sheet appends the `bankSubscriptionPaintHeader` band.
  `241hg` skips Live Preview and Reading paths the record sheet owns (expose
  `recordSheetOwnsPath(path)`).
- `styles.css`: the sheet family, states, narrow and dark variants, and reduced motion.
  The `.bob-record-sheet-hides-properties` rules (Properties block, and the inline title
  under `inline_title = hide`) are written now, but nothing applies that class yet. Unit
  tests cover the class lifecycle only; the land agent's CDP walkthrough verifies the
  selectors against real Obsidian DOM.
- Lifecycle wiring in `170-plugin-lifecycle*.js` and `310-install-methods.js`; exports
  in a new last `37x-exports-record-sheet.js`.
- `⊞ Edit` and row affordances render only once `record-entry` provides handlers. Ship
  no dead controls: rows are focusable for `y` only.

**Tests.** Add `scripts/test-ledger-tools-record-sheet.cjs`. Cover:

- the gate table: canonical record; `bank/2026-11.md`; a legacy type; mismatched,
  missing, and cyclic parents; nested outside the folder; a broken schema; mobile; no
  report; no capability;
- the model on fixtures shaped like the real `bead_lifecycle`, a sparse movie, a legacy
  `eat` restaurant with a wikilink location, and a `subscriptions` record (list `match`
  and legacy `recur_id`);
- coverage (every key exactly once) and the lint states;
- footer phrasing, with and without `created`;
- `title`-versus-H1 fold behavior;
- collapse and fold maps;
- the signature reusing the decoration on body edits;
- DOM text-node safety;
- docking ownership;
- teardown removing classes and nodes.

**Docs.** Add a new `docs/record-card.md` plugin reference page (sheet half) to
`docs/nav.yml` and cross-link it from `docs/albums.md`. Run `npm run docs:gen`.

**Deploy.** `bob plugins sync`.

## record-session

**Goal.** Build the plugin-side write machine that the Record Card drives.

**Build.**

- `242l-plugin-record-update-service.js`:
  - `bobSpawnJson` wrappers for `records show TYPE STEM` and
    `records update TYPE STEM [-d] -i -` with the envelope on stdin (follow `242a`'s
    spawn conventions and timeouts);
  - strict decoders for the new JSON;
  - the capability check against the cached album report;
  - own-path echo suppression, reusing `albumUpdateOwnPathsStore` semantics so the
    record write never schedules an automatic `albums sync`;
  - `apiAlbumInvalidate("record-update")` after an apply;
  - a per-record-path Undo store.
- In `242aa`, add an opt-in `applyOps(ops)` path to `createGuardedUpdateSession`,
  enabled only by `init.allowDirectApply === true`. It saves the editors, sends the
  envelope with the current `expect`, and shares `apply()`'s outcome handling, without
  requiring a preview. Album and bank never set the flag. Their behavior and every
  existing test stay unchanged.
- `242m-plugin-record-update-session.js`: `createRecordUpdateSession` wraps the shared
  session the way `242b` does. It sets `allowDirectApply`, the record copy strings, an
  `isStaleCode` for `stale_record`, and `saveEditors([recordPath, albumPath])`.
  - On stale, it rebases to the returned record fingerprint.
  - Undo applies `inverse_ops` directly with `expect = fingerprint_after`.
  - After a successful apply, it pushes the applied `properties` into the sheet overlay,
    marking changed keys for the one-shot flash, and refreshes the sheet.

**Tests.** Add `scripts/test-ledger-tools-record-update.cjs` with fake spawns, timers,
and workspace. Cover:

- decoders;
- the direct-apply and preview paths;
- the stale rebase keeping the draft;
- Undo replay;
- save-failure refusal;
- echo suppression and report invalidation;
- overlay set and expiry;
- capability gating;
- late responses after close ignored;
- album and bank session tests still green, unchanged.

**Writer matrix.** Add `scripts/obsidian-record-writer-e2e.cjs`, modeled on
`obsidian-album-schema-writer-e2e.cjs` with an offline `--self-test` wrapper registered
in `npm test`. It runs against real `BOB_BIN` in a scratch Git vault and covers:

- a status pick rewriting only the `status:` span plus the `[!records]` callout in one
  write;
- stale, invalid, and required-empty refusals writing nothing;
- Undo restoring the value and the regrouping;
- list round-trips;
- CRLF and BOM preserved.

**Deploy.** `bob plugins sync`.

## record-card

**Goal.** The Record Card modal with browse mode, the choices stage, and every discrete
action, per the Record Card contract.

**Build.**

- `242n-plugin-record-card-model.js` (pure): targets, selection, the hint-strip model
  per row kind and mode, the All actions list, the Esc ladder, the opening-field rule,
  and status-rule text.
- `242o-plugin-record-card-dom.js` and `242p-plugin-record-card.js`.
  - Reuse `albumSchemaCardModalBase`, `bob-key-card` chrome, and the sheet's value
    renderers. Do not fork the album card stack.
  - `openRecordCard({ filePath, field, stage })` owns the desktop gate, the capability
    gate, and the gate's teaching Notices (`not in the album report — run Sync albums`,
    `needs desktop Obsidian`, `Update bob …`). For a broken album schema it opens
    read-only with lint and the YAML path.
  - The card spawns the per-record read on open and repaints on vault modify of the
    record or album.
- Implement:
  - browse keys `j`/`k`/arrows/`Home`/`End`/`Tab`/`Enter`;
  - digits on choices;
  - `t` on dates;
  - `Ctrl+D` clear, with the required refusal;
  - `u`/`Ctrl+Z`;
  - `y`/`Ctrl+E` through `242e` generalized with kind `"record"`;
  - `=` → `openAlbumSchemaEntry({ filePath })`;
  - the choices pick list (filter, `j`/`k`, Enter, digits, current ring, counts);
  - the `⋯` menu, live-region announcements, and the Esc ladder;
  - an Enter latch against double presses;
  - composition guards.
- Add the `edit-record` palette command (`Edit record`). On non-records it shows the
  Notice `Open a record note to edit its fields.`

**Tests.** Add `scripts/test-ledger-tools-record-card.cjs`. Cover:

- the key table per mode;
- opening-field selection;
- digit and `t` direct applies;
- clear refusal on required fields;
- Undo;
- the `=` and `y` dispatch;
- the Esc ladder;
- the inputs-never-navigate rule;
- composition;
- the stale repaint keeping the selection;
- read-only mode for a broken schema and for no capability;
- teardown.

**Docs.** Fill in the Record Card half of `docs/record-card.md` with the key table and
entry points known so far.

**Deploy.** `bob plugins sync`.

## record-stages

**Goal.** Every typed stage, per the contract, so each schema kind in the vault is
editable from the card.

**Build.**

- `242q-plugin-record-card-stages.js` (split further if it nears 1000 lines):
  - text, suggest combobox, date, number, URL, link, and list stages;
  - a debounced dry-run preview through the session, with bob's message and consequence
    in the status rule and an Enter latch until the preview is current;
  - direct apply for an existing suggestion;
  - `Tab`/`Shift+Tab` save-and-advance (wait for the current preview, then save, then
    open the next or previous field's stage; an unchanged value just advances);
  - empty-input semantics (clear optional, refuse required);
  - a stale draft kept across rebase;
  - focus and caret restoration through repaints.

**Tests.** Add `scripts/test-ledger-tools-record-card-stages.cjs`. Cover:

- each kind's input, preview, and apply;
- the near-duplicate warning;
- list chip editing producing JSON arrays (including comma-containing items);
- link suggester resolution;
- the Tab chain across mixed kinds;
- the latch under slow previews;
- refusals that teach;
- IME composition;
- no navigation letters while typing.

**Docs.** Complete the stages section of `docs/record-card.md`.

**Deploy.** `bob plugins sync`.

## record-entry

**Goal.** Every entry point opens the Record Card, and the Properties hide switches on.

**Build.**

- **Sheet affordances.**
  - `⊞ Edit` (desktop plus capability; title `Edit fields (==, Ctrl+Shift+P)`) opens the
    card.
  - A row opens the card on that field.
  - A value chip, ghost chip, or missing pill opens that field's stage directly.
  - Keyboard activation works on every affordance.
- **`==`.** In `248m`, `schemaDoubleEqualsTargetKind` returns `"record"` for notes whose
  own type is exactly `record` (the canonical check, not the loose kind resolver).
  Dispatch `void this.openRecordCard({ filePath })` inside the existing try/catch. All
  other gate rules stay unchanged. Update the gate table tests, including a
  `bank/2026-11.md`-shaped note that keeps its reindent.
- **Ctrl+Shift+P.** In `bob-navigation-hotkeys/src/551-plugin-album-schema-fallback.js`,
  a resolved `record-parent` target calls `ledger.openRecordCard({ filePath })` when the
  ledger exposes it. Otherwise it keeps today's `not-direct-album` fall-through. Tasks,
  links, and dependency rows keep priority. Bump navigation-hotkeys' minor version.
- **Command Mode.** Add
  `{ plugin: "bob-ledger-tools", id: "edit-record", title: "Edit record", section: "vault" }`
  to `CM_SHEET_SURFACE_ROWS`, update the surface tests, and bump the minor version.
- **The hide.** Apply `bob-record-sheet-hides-properties` to the view's own container
  exactly when the gate's `hideEligible` holds, with every removal path from the hiding
  contract. Under `inline_title = hide`, hide that view's inline title too.

**Tests.** Extend:

- `test-ledger-tools-schema-double-equals.cjs` (the record acts; counted, register,
  Source, visual, and non-record cases keep reindent);
- `test-navigation-album-schema-fallback.cjs`;
- the command-mode surface tests;
- `test-ledger-tools-record-sheet.cjs` (hide applied and removed on every event; old-bob
  and broken-schema cases leave Properties visible).

**Docs.**

- `docs/albums.md`: a record's `==` now opens its Record Card; `=` inside climbs to the
  schema.
- `docs/bob-ledger-tools.md`: the record sheet, the Record Card, and `edit-record`.
- `docs/bob-navigation-hotkeys.md`: Ctrl+Shift+P on records.
- `docs/bob-command-mode.md`: the new row.
- `docs/record-card.md`: entry points and the hide rule.
- Run `npm run docs:gen`. Do not edit the archived `==` plan file; these docs and this
  plan record the amendment.

**Deploy.** `bob plugins sync`.

## Integration and land-agent acceptance

The land agent audits all seven deliveries against this plan, fixes integration defects,
and owns the final proof.

1. **Real-Obsidian CDP walkthrough.** Add `scripts/obsidian-record-card-e2e.cjs`,
   modeled on `obsidian-album-ui-e2e.cjs`, with an offline wrapper in `npm test`. It
   uses an isolated scratch vault and profile with real `BOB_BIN`, seeded with:
   - a zettel;
   - a sparse movie;
   - a legacy-keyed restaurant with a wikilink location;
   - a subscription (docked strip, list `match`);
   - a missing-required record;
   - a broken-schema album's record;
   - a `bank/2026-11.md` lookalike.

   Run it where Obsidian and xvfb exist (athena per `docs/develop/validation.md`; see
   the tailnet reference memory for SSH). If no host can run it, record the run as
   skipped with the reason. Inspect every screenshot. Verify:
   - The sheet sits between the H1 and `## Notes`. Values use bob's vocabulary, and
     every key is shown exactly once. Light and dark at 375px and wide. Collapse and
     fold work. Reduced motion is honored.
   - Properties hide only in the green branch of the research's hide figure.
     Old-bob/no-capability, broken schema, legacy type, outside folder, Source mode, and
     embeds all fail open. Disabling the plugin restores Properties.
   - `==` opens the card with no reindent and no undo entry, while `3==`, Source mode,
     and non-records reindent. Ctrl+Shift+P, the palette, Command Mode, `⊞ Edit`, and
     row/chip/ghost clicks open the right field.
   - Keyboard-only end-to-end:
     - a digit pick rewrites only the `status:` span and the `[!records]` callout in one
       write, with all other bytes identical;
     - `t`, Ctrl+D, and the Tab fill-out across kinds work;
     - Undo restores the value and the regrouping;
     - stale (a Source-mode edit in another pane), invalid, and required-empty each
       refuse with nothing written;
     - `y` → Done returns to the card;
     - `=` reaches the album Schema Card.
   - Rendering, navigating, folding, cancelling, and opening are byte-neutral by hash.
     The sheet updates immediately after a save, with no metadataCache flash of old
     values.

2. **Vault coverage check.** On the live vault, read-only: for every record,
   `bob records show <album> <stem> -f json` must account for every frontmatter key
   across `properties`, `identity`, and `other`, and report no unexpected findings.
   Record the counts.
3. **Gates.** In bob-plugins run `npm run build`, `npm run docs:gen`, `npm test`,
   `npm run validate`, and `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`; keep
   only intended generated changes. In bob-cli run `just check` through `/sase_monitor`.
   Run `bob plugins sync` and verify the deployment per `docs/develop/deployment.md`. A
   failure that reproduces identically on a clean base is recorded with evidence as a
   proposed follow-up.
4. **Hosts.** Confirm apollo's `bob` carries `record_values_v1`
   (`bob albums list -f json`). The final report tells Bryan that the Mac needs
   `just install-all` before editing turns on there; until then the capability gate
   keeps Properties visible. List Bryan's manual Mac checks: `==` on `bead_lifecycle`, a
   status digit pick, a claim edit, Undo, and how the page looks.
5. **Records and follow-ups.** Record the implemented features, exact checks,
   screenshots, and deploy outcome on this epic's bead through the SASE artifact
   workflow. Triage phase `PROPOSED FOLLOW-UP:` notes into task beads via
   `/sase_new_task`. File these deferred items if no bead exists yet:
   - title rename as a bob operation;
   - pointer quick-pick menus on sheet chips;
   - in-sheet staging;
   - a provenance-key cleanup (`lid`, `source_*`, `recur_id`, `zorg_id`).
6. **Close.** Run `sase bead epic-symbols` for this epic and resolve or re-key the
   remaining entries. Close only this epic's phase beads and epic bead. End every turn
   through `/sase_final`.

> [!decision] decision_record Only if answered yes, the land agent authors the
> `decisions:` web strand `record-values-edited-through-bob` through the
> `/sase_memory_write` flow, citing `research:202611/record_card/record_card.md` and
> this plan. Its claim: record values render in a plugin record sheet from metadataCache
> and bob's album report, and are edited only through `bob records update`'s guarded
> envelope; native Properties hides only when the sheet covers every key and bob can
> write, failing open otherwise; `==` on a record opens the Record Card, amending the
> `album_only` behavior. It reopens if Obsidian ships a schema-aware property-widget API
> or record edits need audit-grade per-edit commits. Then the land agent runs
> `sase memory init`. If answered no, no memory file changes in this epic.

The completion criterion is the whole promised behavior, deployed and verified:

- every gated record shows the sheet with Properties hidden;
- every field kind is editable from the Record Card through `bob records update -i`,
  with previews, refusals, and Undo;
- every entry point works;
- every fail-open path leaves Properties visible.

## Out of scope

- Editing `parent`, `type`, `title`, or `created`, and title or stem rename.
- Schema edits from the Record Card (`=` hands off to the Schema Card instead).
- Note appends from the card, and multi-field batch saves.
- Pointer quick-pick menus on sheet chips.
- In-sheet editing inside the CM6 widget.
- Mobile editing.
- Legacy noun record types.
- Provenance-key cleanup.
- Bases-cell write safety (it predates this feature).
- A JS YAML writer or JS-side validation authority.
