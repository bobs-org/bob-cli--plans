---
tier: epic
title: Prose and Notebook Cards
goal: 'Prose notes and notebooks open on beautiful, reliable title cards in the Property
  Card family. Native Properties hide under the family gate, and the cards paint rich
  derived data without spawning bob or writing the vault. The work is built beside
  bob-cli-qi and bob-cli-qj and touches them only at four explicit gates.

  '
decisions:
  stub_chip:
    ask: How should the Prose Card count the sections that still need writing?
    choices:
      leaf: '''12 to write'': stub sections without subsections; umbrella headings
        are excluded'
      literal: '''13 TODO sections'': every bob stub, including umbrellas whose subsections
        are written'
    default: leaf
    why: An umbrella heading whose subsections are written is not unwritten
    answer: leaf
  memory_prose_card:
    ask: Add the glossary:prose-card strand?
    memory:
    - glossary:prose-card
    requested: As a part of this change, we should add excellent new glossary terms
      for "prose card" and "notebook card".
    default: true
    answer: true
  memory_notebook_card:
    ask: Add the glossary:notebook-card strand?
    memory:
    - glossary:notebook-card
    requested: As a part of this change, we should add excellent new glossary terms
      for "prose card" and "notebook card".
    default: true
    answer: true
  memory_property_card:
    ask: Update glossary:property-card so the family lists the Notebook and Prose
      Cards?
    memory:
    - glossary:property-card
    corrects: The shared family of note-top album, record, day, period, and bank cards,
      built on the bob-ledger-tools kit.
    default: true
    answer: true
phases:
- id: kit
  title: Kit species, eligibility, focus-or-add, the preview gate, and scaffolding
  depends_on: []
  size: medium
  description: 'kit: register the notebook and prose species, fix the known kit traps
    (silent created, coordinator kinds, editor focus on a missing loose key), add
    the exhaustively tested precedence and dual-spelling predicates, add the preview
    gate that keeps both cards off in the live vault, and pre-create every shared
    file region the two species phases will fill.'
- id: notebook-card
  title: Notebook Card shell with header absorption
  depends_on:
  - kit
  size: medium
  description: 'notebook-card: paint the Notebook Card''s identity, title adoption,
    summary or Add-summary ghost, foot, drawer, fold, and editor key from local data,
    and replace the linked notebook header wherever the card mounts.'
- id: prose-card
  title: Prose Card shell, local-first
  depends_on:
  - kit
  size: medium
  description: 'prose-card: paint the quiet title-page Prose Card from local data
    (folio initial tile, title, dek, crumbs, numbering chip, foot), owning only the
    inline title and coupling its typography to the Bob Prose look.'
- id: notebook-readouts
  title: Notebook Card report-backed readouts
  depends_on:
  - notebook-card
  size: medium
  description: 'notebook-readouts: add the note count, newest date, section chips
    that jump to their headings, child notebooks, an error pill, and lint from the
    existing shared notebook report, with zero new spawns and no layout shift.'
- id: prose-seam
  title: Bob Prose outline seam and Prose Card measures
  depends_on:
  - prose-card
  size: medium
  description: 'prose-seam: after bob-cli-qj.3 closes, add a read-only versioned api.prose
    snapshot API to Bob Prose, then paint the card''s measure row, outline, and findings
    from it.'
- id: acceptance
  title: Real-Obsidian acceptance and screenshot review for both cards
  depends_on:
  - notebook-readouts
  - prose-seam
  size: medium
  description: 'acceptance: after bob-cli-qi.4 closes, run both cards with the preview
    gate on in isolated real Obsidian across the takeover, single-owner, fail-open,
    data, focus, and version-skew matrix. Review the screenshot matrix and repair
    defects until GO.'
- id: go-live
  title: Switch the cards on, docs, approved memory, and clean deploy
  depends_on:
  - acceptance
  size: medium
  description: 'go-live: after bob-cli-qi.6 and bob-cli-qj.4 close, delete the preview
    gate, extend the passive-rendering proof, finish the docs in both repos, apply
    the accepted memory decisions, and deploy cleanly.'
proposed_by: bbugyi200.athena.1gr
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 12:37:56
status: done
bead_id: bob-cli-qm
---

- **PROMPT:** [prompts/202611/prose_and_notebook_cards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/prose_and_notebook_cards.md)
- **BEAD:** [bob-cli-qm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qm/README.md)

# Plan: Prose and Notebook Cards

## Why

Bryan wants a property card at the top of every prose note (formerly article note) and
every notebook, behaving like the rest of the [[property-card]] family. The cards hide
native Properties, use the shared key family, and show rich, useful data about the note.
He also wants excellent glossary terms for both cards, and the work must run in parallel
with the in-progress `bob-cli-qi` (bank hub demotion and card acceptance) and
`bob-cli-qj` (article → prose rename) epics.

**Required reading for every phase:**
`research:202611/prose_and_notebook_title_cards/prose_and_notebook_title_cards.md` (read
it with `sase artifact read`). Bryan agreed with every requirement it recommends, and
this plan adopts its eight recommended defaults. Memory is the exception: it changes
only under this plan's accepted memory decisions. This plan carries the report's design
and makes these changes as lead designer:

1. **Preview gate (new).** Every commit lands directly on bob-plugins master, and any
   agent's `bob plugins sync` (`qi.6` and `qj.4` both deploy) ships master to the live
   vault. So "develop now, land through gates" needs a switch. Both cards stay off in
   the live vault until `acceptance` says GO; see [Preview gate](#preview-gate).
2. **One gate moved earlier.** The Bob Prose seam waits for `qj.3`, not `qj.4`. `qj.3`
   is the phase that rewrites `bob-articles`; `qj.4` only swaps a spawn argv, and a
   rebase absorbs that.
3. **Leaf-stub "to write" count.** This replaces the report's literal TODO count, under
   the `stub_chip` decision.
4. **Concrete choices** for the prose tile (a folio carrying the title's initial),
   reserved readout rows (no layout shift), honest word rounding, and the drawer's
   Managed fold for importer keys.

## Verified current state

Verified at bob-cli `cf414ce1` and bob-plugins `e36ddc68`. Anchors drift: re-locate
every symbol on current master before editing.

- **Kit.** The kit lives in `plugins/bob-ledger-tools/src/240*` and has no `notebook` or
  `prose` species.
  - Species registry: `240f`.
  - Shared Reading/Live Preview host: `240c` `createPropertyCardHost`.
  - Title slot and adoption: `240e`–`240k`, with `propertyCardNeverTitleRecognizer` in
    `240h`.
  - Created stamp: `240m`.
  - Drawer inventory: `240n`. Adapters can mark keys `managed`, which folds them into a
    closed `Managed · N` group.
  - Ownership coordinator: `240q`.
  - Fold kit: `240s`.
  - `==` registry: `248m` `registerPropertyCard`.
- **Trap T1.** `240m`: `const silent = temporal || kind === "area"`. A notebook or prose
  card would print "no created date" on 28 of 39 notebooks.
- **Trap T2.** The notebook catalog's **add summary** calls
  `openPropertyEditor({ setting: "summary" })`. Notebooks have no settings contract, and
  the `298z` fallback selects only existing rows, so a missing `summary:` strands focus.
  This is a live bug today.
- **Trap T3.** `240q` hardcodes three things: the eligible kind list, a per-kind
  mount-source attribute, and the area/project-only `props_contract_v1` rule.
- **Notebooks.** The live vault has 39 notebooks: 13 roots, 15 with `##` sections (at
  most 6), and 7 with children. 25 of 39 carry seven zorg-importer keys, all 39 have a
  `summary:`, 28 of 39 lack `created:`, and only 1 has a non-null `last_note`.
  - The shared report service is `241a` (`apiNotebookSubscribe`), which reads
    `bob notebooks list -f json` (`docs/notebooks.md` §JSON). Sections there are
    `{title, notes}` with no line numbers.
  - The linked notebook header (`241e`–`241g`) paints the crumbs and child chips today.
- **Prose.** The one live prose note is `sase_blog_contents.md`, still typed
  `[[article]]` until `qj.4`.
  - It has 14 sections: an umbrella `## 1.` with eight TODO subsections, plus mostly
    TODO leaves.
  - Its frontmatter is `parent`, `type`, and `created` only.
- **Bob Prose outline cache.** Bob Prose (`plugins/bob-articles`) already keeps a
  per-path outline cache (`articleOutlines`, `articleRememberPlan` in
  `035-plugin-look.js`). It fills the cache off-paint on open, after format, and from
  the stale-hint timer. It stores only `plan.outline`; findings and refusals are
  dropped.
  - Each ArticlePlan outline section carries `number`, `depth`, `title`, `words`,
    `stub`, `line`, `end_line`, `output_line`, `written`, and `stale`
    (`docs/articles.md`, ArticlePlan v1).

## Governing decisions

- `decisions:notebook-routes-write-notes`: notebooks hold note bullets only. The card
  never adds tasks, checkboxes, ready caps, or review pressure. Its `@stem` token is the
  notebook's existing capture route.
- `decisions:capture-sigils-say-where-what-how`: `@` says where. The route token and the
  section-chip tooltips show exactly the routes capture already accepts.
- `decisions:q-and-ctrl-bracket-mirror-esc`: anything the cards open (the Property
  Editor, focus-or-add, the `+N` expansion) closes with Esc, `q`, and Ctrl+`[` through
  the shared ladder.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture changes nothing.
- `decisions:bob-cli-releases-ride-a-daily-train`: this plan has no Rust or CLI change.
  Never edit `Cargo.toml` versions or `CHANGELOG.md`. No commit here needs `!`.
- `decisions:decision-rules-are-read-at-planning-time`: this plan read the roster. No
  record governs property-card species, and none is added.
- `decisions:prose-notes-are-owned-prose` lands with `qj.4`. Its rules bind the Prose
  Card: both spellings count forever, the copy says "prose note", and wire and plugin
  ids keep their legacy names.

## Design

### Family contract, applied

Both cards are ordinary kit title-block cards. They copy the family's **contract**, not
the Area/Project card's **contents**, because neither note kind is a task home.

- **Takeover.** In desktop Live Preview and Reading, a mounted card and foot replace
  native Properties only under the existing fail-open gate:
  - valid source matches the cache;
  - `props_v1` is confirmed for the current bob (both kinds gate on `props_v1` alone,
    with no `props_contract_v1`);
  - editor routing works.

  Source mode, embeds, hover, canvas, print, mobile, uncertain evidence, failed mounts,
  and unload keep or restore native Properties. The File Properties sidebar stays
  native.

- **Keys.** In Live Preview Vim normal mode:
  - `-` toggles the drawer.
  - `_` folds the card's own section.
  - `==` opens the Property Editor focused on the species key: `summary` for notebooks,
    `title` for prose. A missing key opens Add mode prefilled (kit focus-or-add).
  - `+` opens the Property Editor.

  Commands mirror the keys outside Vim, as the family does.

- **Hue says where; form and word say what.** The crumbs carry the root-area hue pip,
  and the tile form plus the kicker word name the species. Neither species gets its own
  hue. Colors derive from the accent and theme tokens. Both cards must work in light and
  dark, in `forced-colors`, and in grayscale (the tile forms must stay distinct there).
- **Missing data is not zero.** Every report-backed part is content-gated. A missing or
  stale report omits the part and never paints `0 notes` or `0 words`. A real empty
  value from bob is a fact; even then, chips show no `0` count.
- **No layout shift.** When a report or snapshot is expected (the service or API is
  present), reserve the readout row's height. When it arrives, patch in place. If none
  comes, collapse the row once. Never paint placeholder numbers.
- **Card shell and data freshness are checked separately.** Stale derived data hides or
  quietly updates chips. It never unmounts the title or flashes native Properties while
  Bryan types.
- **Passive painting.** Painting, folding, resizing, cursor movement, and drawer toggles
  never spawn bob and never write the vault. Note text renders as text, never as HTML.
  Subscriptions are released on unload. Each view gets one card, with generation-bound
  handles across split panes.

### Notebook Card: expanded by default

The Notebook Card is the notebook's front door and navigation surface. Its closest
analog is the Album Card. It **replaces** the linked notebook header wherever it mounts.

```text
╭────────────────────────────────────────────────────────────────────────╮
│ ┃▤  NOTEBOOK · ● Dev › Developer tools        30 notes · newest Nov 25  │
│ ┃   Developer tools                                                    │
│     Command-line tools and dotfile setup I rely on: chezmoi, tmux,     │
│     fzf, and friends.                                                  │
│     § Shell 6   § Git 4   § Terminal 9   § Dotfiles 11                 │
│     ↳ Git 6                                                            │
│ ────────────────────────────────────────────────────────────────────── │
│   created Oct 12, 2024 · @dev_tools                ▸ Properties · 9  − + │
╰────────────────────────────────────────────────────────────────────────╯
```

Folded (`_`), and with errors and no summary:

```text
│ ┃▤  NOTEBOOK · ● Dev        30 notes · 4 sections · newest Nov 25       │
│ ┃   Developer tools                                                    │
│     Command-line tools and dotfile setup I rely on: chezmoi, tmux, …   │

│ ┃▤  NOTEBOOK · ● Dev › Languages               18 notes   ✗ 2 errors   │
│ ┃   Java                                                               │
│     ＋ Add summary                                                     │
```

| Element         | Source and rule                                                                                                                                                                                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tile            | `spine` form: a tile with an inset accent bar on its left edge, like a bound spine, and the `notebook-tabs` icon the pickers already use.                                                                                                                                             |
| Kicker          | `NOTEBOOK` · crumbs from the header's ancestry model (`241e`), without the self crumb. At most four crumbs, with the middle collapsed to `…`. Crumbs navigate like links; Mod opens a new tab.                                                                                        |
| Title           | The existing notebook rule: eligible opening H1 → first alias → humanized stem. A matching H1 hides at render time, and a Live Preview cursor reveals it. `role="heading"`.                                                                                                           |
| Summary         | `summary:`, clamped to two lines, with the full text in a tooltip when clamped. Missing or blank shows a quiet `＋ Add summary` ghost that opens focus-or-add on `summary`.                                                                                                           |
| Readout (right) | `N notes` (direct notes; the tooltip says that nested detail and child notebooks are excluded). `· newest <date>` appears when `last_note` is non-null; its tooltip says it is the newest authored `[created::]` date, never file time. When folded, `· M sections` is added.         |
| Error pill      | `✗ N errors` for hard findings only. Click jumps to the first error line when the issue carries one; the tooltip lists every message.                                                                                                                                                 |
| Section chips   | One chip per `##` section in document order: `§ Title N`, with no count for an empty section. Click jumps to the heading. The tooltip names the count and the capture route (`@stem#Section`). With more than 8 chips, show 7 plus a `+N` chip that reveals the rest for the session. |
| Children        | `↳` plus direct child-notebook chips (`Title N`), shown only when children exist. They open like crumbs.                                                                                                                                                                              |
| Foot            | The kit foot. `created` is silent when missing. Then a quiet monospace `@stem` route token (click copies it, with a short notice), the lint chip (warnings, whose tooltip lists each code and message), `▸ Properties · N`, and `−` / `+`.                                            |
| Drawer          | Every key in file order. `generated_from_zorg`, `zorg_*`, and `template` are marked `managed`, so they fold into the closed **Managed** group.                                                                                                                                        |

- **Narrow panes (~420 px).** The readout drops below the title as a muted line, and
  chips wrap to at most two lines before `+N`.
- **Not on the card:** tasks, ready counts, freshness, WIP badges, mtime, word counts,
  and activity pressure. A quiet notebook is a healthy reference collection.
- **Notebook chips setting.** The setting keeps governing the `[!notebooks]` catalogs
  and the fallback header. The card's own chips are part of the card.

### Prose Card: folded by default

The Prose Card is a title page in the publishing sense, and the quietest card in the
family.

- Its wash is about 3–4% of the hue.
- When the Bob Prose look (`.bob-article`) is active, its width aligns to the 40rem text
  column and its title uses the prose heading face.
- Elsewhere it keeps the family's typography.

```text
╭────────────────────────────────────────────────────────────────╮
│ ╭─╮◥ PROSE · ● SASE › SASE blog                     § numbered  │
│ │W│  Why I built SASE                                           │
│ ╰─╯  A short account of the ideas behind the tool.              │
│      ≈ 310 words · 2 min read · 14 sections · 12 to write       │
│ ─────────────────────────────────────────────────────────────── │
│   created Oct 15, 2026 · ⚠ 4             ▸ Properties · 3  − +  │
╰────────────────────────────────────────────────────────────────╯
```

Expanded (`_`) adds a bounded outline and the findings:

```text
│      ≈ 310 words · 2 min read · 14 sections · 12 to write       │
│      1     What is SASE?                                        │
│        1.1 What is a "prompt language" (macros)?     to write   │
│        1.2 What is an "agent turn"?                  to write   │
│      3     Why SASE is still NOT ready?              ≈ 40       │
│      Findings                                                   │
│      ⚠ line 8   pseudo-footnote: …                              │
```

| Element                | Source and rule                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tile                   | `folio` form: a page with a folded top-right corner. It holds the title's first letter or digit, uppercased, in the prose heading face. When the title starts with anything else (an emoji or punctuation), it holds the `pilcrow` icon. If `acceptance` finds that the initial does not read as a species in grayscale, it switches to the pilcrow everywhere.                                                                                                                                                            |
| Kicker                 | `PROSE` · crumbs and hue from the home-card parent resolver. A parent that is not a home is shown as a plain link with no pip. The word is always PROSE, including for legacy `[[article]]` notes.                                                                                                                                                                                                                                                                                                                         |
| Numbering chip (right) | `§ numbered` or `§ unnumbered`, read from local `section_numbers` (absent or `true` means numbered). An invalid value shows `§ ⚠`, with a tooltip naming the allowed values. Click runs `bob-articles:toggle-section-numbers` when that command is registered (from `qj.3`); otherwise the chip is inert text. The card never writes YAML itself.                                                                                                                                                                          |
| Title                  | `title:` (one line) → first alias → the **verbatim** file stem. On the stem fallback, the tooltip reads "Title from the file name — press == to set one". The card owns only the inline title (see [Prose title ownership](#prose-title-ownership)).                                                                                                                                                                                                                                                                       |
| Dek                    | Optional `summary:`, in italics, clamped to two lines. Absence is normal: no ghost and no nag.                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Measure row            | Built from the Bob Prose snapshot. **Words:** `≈ N words`, rounded to the nearest 10 below 1,000 and the nearest 100 above, with grouping. The tooltip says the count is approximate because it includes heading text. **Reading time:** `M min read`, where M = round(words / 230), shown as `< 1 min read` when M is 0. **Sections:** `S sections`, exact. **Unwritten:** see [Unwritten sections](#unwritten-sections). Each part is omitted when its data is absent. Clicking the unwritten count expands the outline. |
| Refusal                | When bob refuses the note, the measure slot shows bob's own message as one muted line, never a fake zero report.                                                                                                                                                                                                                                                                                                                                                                                                           |
| Outline (expanded)     | Rows in document order: bob's `number` (only when numbering is on), the de-numbered title, an indent by `depth`, and a right column with `≈ words` or a `to write` badge. Click jumps to the section. The outline is capped at about 16 rows, then scrolls. **Findings** follow, one row each (severity, `line N`, message), and click jumps to the line.                                                                                                                                                                  |
| Foot                   | The kit foot: `created` (silent when missing), the lint chip with the findings count (click expands the card at Findings), `▸ Properties · N`, and `−` / `+`.                                                                                                                                                                                                                                                                                                                                                              |

- **Unwritten sections.**

  > [!decision] stub_chip = leaf Count bob's `stub` sections that have no deeper section
  > before the next section at the same or a shallower depth. Label the count
  > `N to write`. Its tooltip says: "Sections whose own body is empty or only TODO/TBD;
  > headings whose subsections hold the writing are not counted". The outline badges
  > exactly those rows.

  > [!decision] stub_chip = literal Count every `stub: true` section. Label the count
  > `N TODO sections`. Its tooltip quotes bob's definition: the section's own body is
  > empty or only TODO/TBD. The outline badges every stub row `TODO`.

  Either way, never show a percentage or a completion claim.

- **Prose title ownership.** In a prose note, every ATX heading, including `#`, is a
  numbered section. So the Prose Card hides the Obsidian inline title only, through
  `propertyCardNeverTitleRecognizer`, and never hides a body heading. This is the
  family's one species exception.
- **Jump safety.** Jump to bob's `line` only when that buffer line is still a heading
  whose text contains the section title. Otherwise, pair by order with the metadata
  cache's headings. If neither matches, do nothing.

### Precedence and eligibility

One kit function resolves a note's top species in bob's props order:

1. bank path;
2. album;
3. record;
4. area/project;
5. temporal;
6. notebook;
7. prose;
8. none.

The new species claim a note only when its `type` is exactly `[[notebook]]`, or is
`[[prose]]` / `[[article]]` (scalar, or a list holding only those two spellings). Any
other mixed type mounts neither new card.

- **One shared dual-spelling predicate.** Use it everywhere in ledger-tools; it mirrors
  `qj`'s rule. Writers never retype a note.
- **Exclusions match the home card:** `_templates/`, `done/`, `_conflicts/`, the untyped
  type notes, and embeds.
- **Test.** An exhaustive unit test runs the existing eligibility predicates over every
  single-type, empty, and two-type combination of {area, project, album, record,
  notebook, prose, article, bank_month, ref, a custom record type}, plus the bank and
  daily paths. It asserts that at most one species claims each note. The existing
  species' code is not changed.

### Data sources

- **Notebook data.** The shared notebook report service (`241a`), with one plugin-level
  subscription. Section jumps pair each report section with the local metadata cache's
  `##` headings in document order, and duplicate titles are told apart by position. No
  bob-cli change is needed.
- **Prose data.** Bob Prose's read-only `api.prose` v1 (`prose-seam`). The card's
  identity parts (kicker, crumbs, title, dek, numbering, created) paint synchronously
  from local YAML. A missing, disabled, or older Bob Prose therefore still gives a
  correct quiet card, with the measure row omitted. **The JS never recomputes word
  counts, numbers, or stubs.** It only filters and rounds bob's values.

### Preview gate

A kit gate keeps both new species from mounting unless bob-ledger-tools plugin data
lists them: `cardPreviewKinds: ["notebook", "prose"]`.

- **While the gate is closed:** no card, the old notebook header, and native Properties.
  Nothing changes in the live vault.
- **Opening the gate:** the `acceptance` harness writes the key into its disposable
  vault. Bryan can also add it by hand to preview a card early.
- **Removal:** `go-live` deletes the gate, the key, and their tests, so no permanent
  setting remains.

This lets every phase commit to master and run `bob plugins sync`, as the bob-plugins
`AGENTS.md` requires, while unaccepted cards never reach the live vault. The pl/q6
history (Bryan's screencast of flipping cards) is what this prevents.

## Parallel delivery with bob-cli-qi and bob-cli-qj

Coordination is needed at exactly four seams. Everything else is new fragment files,
plus additive edits to shared kit files that rebase cleanly.

| In-flight phase       | What it touches                                                              | Overlap with this epic                              | Handling                                                                                           |
| --------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `qi.4` takeover-proof | Takeover e2e harness, kit repairs, family screenshots                        | The harness and kit files                           | `acceptance` waits for `qi.4`. Development edits to kit files are additive; rebase and rebuild.    |
| `qi.6` finish         | Passive-proof suite, the shared docs sweep, `glossary:property-card`, deploy | Shared docs, the family glossary, the passive suite | `go-live` waits for `qi.6`.                                                                        |
| `qj.2` prose-cli      | bob-cli Rust, `docs/articles.md` → `docs/prose.md`                           | `docs/prose.md`                                     | Only `go-live` edits it, after `qj.4`, which implies `qj.2`.                                       |
| `qj.3` prose-plugins  | `bob-articles`, command-mode, nav-hotkeys, ledger-tools `242d`               | `bob-articles`; the ledger-tools bundle             | `prose-seam` waits for `qj.3`. The bundle: rebase and rebuild.                                     |
| `qj.4` prose-vault    | Vault cutover, `bob-articles` argv, `glossary:prose-note`                    | The glossary link target; the live note's type      | `go-live` waits. The predicate reads both spellings from `kit` on, so cutover order never matters. |
| `qj.5` prose-sections | Proposes the extract/inline child epic                                       | Provenance frontmatter                              | Out of scope. `go-live` records the provenance-crumb follow-up.                                    |

**Gate procedure.** A gated phase runs a step 0 before changing anything:

1. Poll the non-audited `sase bead list --status closed --type phase -f json` for the
   gate bead under `/sase_monitor`. Allow 36 h for `qi`/`qj.3` and 48 h for `go-live`.
2. When the bead shows closed, run `sase bead read <id> -r "<why>"` once. Confirm it
   closed done and not canceled.
3. If it was canceled, or the wait timed out, stop without changes. Record a note on
   your phase bead for the land agent.

Launch-time `-w bead=` is not used, because it would hold every phase.

## Rules every phase follows

- **Repos.**
  - Work in bob-plugins through `/sase_repo` (`sase repo open bob-plugins`), use only
    the printed path, and read its `AGENTS.md`.
  - bob-cli is the primary checkout; only `go-live` edits it (docs).
  - Before any vault read, read the `obsidian.md` reference memory. Never open the live
    vault in an automated Obsidian run.
- **Fragments.** Edit `src/` fragments. Keep each fragment at or below 1000 lines, run
  `npm run build`, and never hand-edit generated `main.js`.
  - Prefixes: `notebookCard*` / `proseNoteCard*` in JS and `.bob-notebook-card*` /
    `.bob-prose-card*` in CSS. A bare `prose*` collides with the `proseField` namespace
    (T6).
  - Fill only the regions that `kit` pre-created for your species.
- **Rebase discipline.**
  - On a conflict in `main.js`, `styles.css`, `fragments.json`, `package.json`, or a
    manifest version, take upstream, rebuild, and re-bump. Never hand-merge a bundle.
  - Do not land shared-kit edits while a `qi.4` real-app run is in flight; check its
    bead notes.
- **Per plugin change.**
  1. Bump the manifest per repo convention.
  2. Run `npm run build`, `npm run build:check`, the focused suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`, following the `AGENTS.md` refusal protocol.

  The preview gate makes these syncs safe. Pre-existing `npm test` failures that
  reproduce on clean master (see `bob-cli-qi.4` note #2) are recorded, not fixed.

- **Tests and evidence.** Cover each model and DOM with unit suites against the shared
  fixtures. Painting tests spy on spawns and vault writes and expect zero. Retain
  real-app evidence with `sase artifact create --bead`.
- **Beads.** Phase workers never create beads. Record a `PROPOSED FOLLOW-UP:` note on
  your phase bead instead. Run `sase bead epic-symbols <phase>` before closing.
- **Long work.** Use `/sase_monitor` for builds, real-app runs, and gate waits. Finish
  through `/sase_final`.

## Phase `kit`

Owner of every shared kit file. Later phases fill regions this phase creates.

1. **Species.**
   - In `240f`, add `notebook` (word Notebook, tile `spine`, icon `notebook-tabs`,
     expanded by default) and `prose` (word Prose, tile `folio`, icon `pilcrow`, folded
     by default) to the registry and `PROPERTY_CARD_KINDS`. Map the fold defaults onto
     the kit's fold model.
   - In `styles.css`, add the `spine` and `folio` tile forms in a new "Note cards"
     region, plus empty `Notebook Card` and `Prose Card` sub-regions. The folio holds
     either text (the initial) or an icon.
   - Verify that light, dark, forced-colors, and grayscale distinction hold against the
     existing tiles.
2. **T1.** `240m` keeps missing `created` silent for `notebook` and `prose`.
3. **T3.** `240q` gains:
   - the two kinds;
   - mount-source attributes `data-notebook-source` and `data-prose-source`;
   - the `props_v1`-only gate (no contract requirement).

   Add coordinator tests for both kinds.

4. **Predicates and precedence.** Add a new kit fragment, for example
   `240t-plugin-property-card-note-species.js`, holding:
   - the dual-spelling prose predicate;
   - the exact notebook predicate;
   - the exclusions;
   - the precedence function from
     [Precedence and eligibility](#precedence-and-eligibility), with its exhaustive
     two-type test.
5. **Focus-or-add (T2).** Add a loose-key request to the Property Editor:
   `openPropertyEditor({ filePath, key })`.
   - It selects the key's row when present.
   - Otherwise it opens Add mode with the key prefilled and the value focused.
   - The existing `setting:` path is unchanged.

   Rewire the notebook catalog's **add summary** onto it. That fixes today's stranded
   focus.

   Tests: present key, absent key, a duplicate key, the degraded report, and the
   Esc/`q`/Ctrl+`[` ladder.

6. **Preview gate.** Add a kit function such as `propertyCardKindLive(plugin, kind)`.
   - It reads `cardPreviewKinds` from ledger-tools plugin data once at load.
   - Existing kinds are always live; `notebook` and `prose` are live only when listed.

   Test both states.

7. **Scaffolding for parallel species work.** Create the fragment stubs and their
   `fragments.json` entries after the home-card fragments:
   - notebook: model, DOM, host, foot, readouts;
   - prose: model, DOM, host, foot, outline;
   - one exports fragment for test hooks.

   Also create:
   - skeleton test files for each species, registered in `package.json` `test:unit`,
     each with one smoke assertion;
   - `docs/notebook-card.md` and `docs/prose-card.md`, each with a "Preview — not yet
     enabled" admonition, registered in `docs/nav.yml` and the docs inventory.

   Record the exact file names on this phase bead for the species phases.

8. **Shared fixtures.** Add a fixture vault (for example `scripts/fixtures/note-cards/`)
   that byte-copies live shapes. Unit and e2e tests both use it.
   - **Notebooks:**
     - zorg-imported, with seven provenance keys and no `created`;
     - an `about_abby`-style notebook with sections and `created`;
     - a missing summary;
     - duplicate `##` titles;
     - a parent/child pair;
     - a notebook with a checkbox (an error);
     - more than 8 sections.
   - **Prose notes:**
     - a copy of the live `[[article]]` note;
     - `[[prose]]` with `title:` and `summary:`;
     - `section_numbers: false`;
     - an invalid `section_numbers`;
     - a list holding both spellings;
     - a mixed `[prose, area]` type;
     - no headings;
     - an emoji-leading title;
     - one holding a `^prj` anchor (bob refuses it).
9. Build, test, and sync. The only visible change is the T2 fix.

## Phase `notebook-card`

1. **Model (pure).** Derive the view model from local frontmatter, the metadata cache,
   and the header's ancestry model:
   - the crumbs and hue;
   - the title and its H1 adoption range;
   - the summary or ghost;
   - `created`;
   - the `@stem` route;
   - the managed key set (`generated_from_zorg`, `zorg_*`, `template`).

   Test it against the fixtures.

2. **DOM.** Build the kit title block per the
   [Notebook Card](#notebook-card-expanded-by-default) table:
   - the spine tile, kicker, title, and summary or ghost;
   - a reserved readout slot, chip rows, and children rows for `notebook-readouts`.

   Text only. Everything must be reachable by accessible name, for example "Notebook:
   Developer tools".

3. **Host.**
   - Mount on `createPropertyCardHost` for Reading and Live Preview, behind the preview
     gate and the precedence function.
   - Register with the coordinator (`notebook`, `data-notebook-source`).
   - Own the title with matching-H1 adoption.
   - Wire the foot adapter: created, the route token that copies, the lint slot, and the
     drawer with the managed group.
   - Add the `_` fold (default expanded, per note for the session).
   - Register `==` with `registerPropertyCard` so it runs focus-or-add on `summary`.
   - `+` and `-` come from the kit.
4. **Header absorption.**
   - The linked notebook header never renders in a scope where the card has mounted,
     whichever mounts first.
   - Unmounting the card (gate closed, takeover denied, failure, unload) restores the
     header.
   - Mobile, embeds, hover, and a closed gate keep today's header unchanged.

   Test both mount orders and every restore path.

5. **Docs.** Fill `docs/notebook-card.md` (behavior, keys, mount rules, fallback),
   keeping the preview admonition.
6. Build, test, and sync.

## Phase `prose-card`

1. **Model (pure).** Derive:
   - the dual-spelling detection (from `kit`);
   - the title precedence and stem-fallback flag;
   - the tile initial or the pilcrow;
   - the dek;
   - the numbering state from local YAML (numbered, unnumbered, or invalid);
   - `created`;
   - the crumbs and hue from the home-card resolver.

   Test it against the fixtures.

2. **DOM.** Build per the [Prose Card](#prose-card-folded-by-default) table:
   - the folio tile with the initial;
   - the kicker, numbering chip, and title;
   - the dek;
   - a reserved measure row and an empty outline region for `prose-seam`.
3. **Host.**
   - Mount on `createPropertyCardHost` behind the gate and precedence, with coordinator
     kind `prose` and `data-prose-source`.
   - Own the inline title only (`propertyCardNeverTitleRecognizer`); every body heading
     stays visible.
   - Wire the foot, the `_` fold (default folded), and `==` for focus-or-add on `title`.
   - Apply the typography coupling only when the Bob Prose look class is present on the
     view.
4. **Numbering chip.** Dispatch the Bob Prose command id when it is registered; it is
   inert otherwise. Test both states with a stubbed command registry.
5. **Docs.** Fill `docs/prose-card.md`, keeping the preview admonition.
6. Build, test, and sync.

## Phase `notebook-readouts`

1. **Subscribe.** Use one plugin-level `apiNotebookSubscribe` subscription that patches
   every mounted Notebook Card in place and is released on unload. Add zero new spawns;
   a spy test proves none on paint, fold, resize, or drawer.
2. **Readout and health.**
   - Show `N notes`, `newest <date>` (with the kit date formatter), and the folded
     `· M sections`.
   - Show the error pill for `severity: error`, jumping to the first line when present.
   - Warnings join the foot lint chip with per-issue tooltips.
   - Paint none of these without a fresh report row for this path.
3. **Section chips.**
   - Pair report sections to local `##` headings in order. When titles disagree (stale
     report), pair by title in order.
   - A section with no pairing keeps its chip, but the chip is inert, with the tooltip
     "Heading moved; the report will catch up".
   - Add the `+N` overflow (session state).
   - Clicks move the Live Preview cursor or scroll Reading view to the heading.
   - Test duplicates, renames, a stale report, and overflow.
4. **Children.** Show direct child chips with counts from the report.
5. Update the docs page. Build, test, and sync.

## Phase `prose-seam`

0. **Gate.** Wait for `bob-cli-qj.3` per the
   [gate procedure](#parallel-delivery-with-bob-cli-qi-and-bob-cli-qj). Then read
   `docs/bob-articles.md` and the plugin as `qj.3` left them.
1. **Bob Prose `api.prose` v1** (in `plugins/bob-articles`).
   - **Shape.** `plugin.api.prose = { version: 1, snapshot(path), subscribe(fn) }`.
     `subscribe` returns an unsubscribe function.
   - **Snapshot.** `snapshot(path)` returns `null` or a frozen copy of:
     - `path`;
     - `generation`;
     - `status: "ready" | "refused"`;
     - bob's `outline` and `findings`, verbatim;
     - `refusal` (`{code, message}` or `null`);
     - `numbering`.
   - **Plans.** Remember findings and refusals beside the existing outline cache. Leave
     decorations, the status bar, and the formatter's own behavior byte-for-byte
     unchanged.
   - **Notify.** `fn(path)` runs after each remembered plan or forget.
   - **Reads and order.** A read never spawns, because Bob Prose keeps owning every
     request. Late responses never overwrite newer generations.
   - **Tests.** Add them to `scripts/test-bob-articles.cjs`. Document the seam in
     `docs/develop/plugin-apis.md` and `docs/bob-articles.md`. Bump the manifest.
2. **Card consumption.**
   - Look up `app.plugins.getPlugin("bob-articles")?.api?.prose` and require
     `version === 1`. Otherwise the card stays identity-only.
   - Use one plugin-level subscription. Paint:
     - the measure row;
     - the unwritten count per `stub_chip`;
     - the refusal line;
     - the foot lint findings;
     - the expanded outline and findings list with safe jumps.
   - Test absent, old, refused, late, stale, and numbering-off snapshots, plus the
     no-spawn spy.
3. Update the docs page. Build, test, and sync both plugins.

## Phase `acceptance`

0. **Gate.** Wait for `bob-cli-qi.4` (GO). Its harness and screenshot review are then
   settled.
1. **Setup.**
   - Use a bob built from current bob-cli master, through `BOB_BIN` and the runner's
     provenance contract.
   - Use an isolated Obsidian profile with a disposable Git vault built from the shared
     fixtures, with `cardPreviewKinds` set. On athena, read the `tailnet.md` reference
     memory first.
   - Extend the takeover harness (`scripts/obsidian-frontmatter-takeover-e2e*.cjs`), or
     add a sibling runner that imports its lib. Never fork it.
2. **Required evidence.**

   | Concern            | Evidence                                                                                                                                                                                                                                        |
   | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Takeover           | In LP and Reading, every fixture shows one card, one title, one foot, native Properties hidden, and no notebook header.                                                                                                                         |
   | Single owner       | Mixed types mount no new card. Area, project, album, record, and bank notes are unchanged. A closed gate gives exactly today's rendering, with the header and native Properties.                                                                |
   | Native restoration | Source, embed, hover, canvas, print, mobile emulation, bad YAML, a source/cache mismatch, rename, a failed mount, unload, and a disabled ledger-tools.                                                                                          |
   | Capability         | `props_v1` pending, absent, or failed. Bob Prose absent, disabled, or without `api.prose` gives an identity-only card. Late and stale snapshots.                                                                                                |
   | Data               | A missing report never shows a zero; arrival patches without layout shift. Duplicate-H2 jumps, child chips, route-token copy, the error-pill jump, outline and finding jumps, and the refusal line.                                             |
   | Focus and routing  | `==`, `+`, `-`, and `_` in LP Vim normal mode. Vim `k`/`gg` and Up at body start. Tab. Focus-or-add on present and missing keys. Esc, `q`, and Ctrl+`[`. The numbering chip toggles through Bob Prose in one undoable transaction and repaints. |
   | Typing             | Editing under a mounted card never unmounts it or flashes native Properties. A stale-hint refresh patches the measures.                                                                                                                         |
   | Passive            | No paint-, fold-, drawer-, or resize-triggered spawns or writes. Protected fixture hashes are unchanged.                                                                                                                                        |

3. **Screenshots.**
   - Capture light/dark × narrow (~420 px)/wide × LP/Reading for both cards, folded and
     expanded, beside an album card and an area card for family consistency, plus the
     native-recovery states.
   - Check forced-colors and grayscale species distinction; this settles the prose
     initial or pilcrow.
   - Inspect every image and record a verdict, size, and digest.
   - Fix visual drift: row alignment, chip wrap, reserved-row height, wash strength, and
     the prose column alignment.
4. **Repairs.** Repair defects in bob-plugins (build, test, sync), then rerun the
   affected rows.
   - Offline self-tests or a skipped real-app run are not acceptance.
   - The phase closes only on GO.

## Phase `go-live`

0. **Gates.** Wait for `bob-cli-qi.6` and `bob-cli-qj.4` per the
   [gate procedure](#parallel-delivery-with-bob-cli-qi-and-bob-cli-qj). Re-read
   `glossary:property-card`, `glossary:prose-note`, and `docs/prose.md` as those phases
   left them.
1. **Switch on.** Delete the preview gate, `cardPreviewKinds` handling, and their tests.
   Both cards are now live.
2. **Passive proof.** Extend `qi.6`'s passive-rendering suite: toggle the notebook and
   prose folds and drawers many times and force refreshes, asserting zero writes and
   zero paint-triggered spawns.
3. **Docs (bob-plugins).** Read the `docs.md` reference memory first.
   - Remove both preview admonitions.
   - Update `docs/property-editor.md` (focus-or-add; `==` per species).
   - Update `docs/bob-ledger-tools.md`: the notebook header is now the fallback, and the
     Notebook chips setting text changes. Update the setting's own description to match.
   - Update `docs/develop/architecture.md` (species list, gates, and the precedence
     function).
   - Update `docs/area-and-project-cards.md` (the related-cards links).
   - Run `npm run docs:check`.
4. **Docs (bob-cli).**
   - `docs/prose.md`: document `summary:` as an optional one-line dek and `title:` as
     the display title. bob ignores both. Point to the Prose Card page.
   - `docs/reference/vault-format.md`: add the same optional keys to the prose section.
   - `docs/notebooks.md`: point to the Notebook Card page.
   - Run `just docs-gen` and `just docs-check`. A doc already over its word budget must
     not grow.
5. **Memory.** Use `/sase_memory_write`, apply only the accepted decisions, then run
   `sase memory init`, `just docs-gen`, and `just docs-check`.

   > [!decision] memory_prose_card Add `glossary:prose-card` (keyword "Prose Card",
   > alias "prose title card"). Adjust this draft to the shipped behavior:
   >
   > The note-top [[property-card]] for a [[prose-note]], including the legacy
   > `[[article]]` spelling. Bob Ledger Tools paints it as a quiet title page in desktop
   > Live Preview and Reading view.
   >
   > - **Identity.** A folio tile bearing the title's initial and a `PROSE` kicker, with
   >   the parent's crumbs in the root hue, lead the title (`title:`, else the first
   >   alias, else the file stem verbatim) and the optional one-line `summary:` dek.
   > - **Data.** From Bob Prose's read-only outline snapshot it adds approximate words
   >   and reading time, the section count, the sections still to write, and findings.
   >   Parts without a snapshot are omitted, never zeroed. A `§` chip shows section
   >   numbering and toggles it through Bob Prose.
   > - **Fold and title.** It is folded by default; `_` opens an outline whose rows jump
   >   to their sections. It owns only the inline title, because every body heading is a
   >   section and stays visible.
   > - **Takeover and keys.** It replaces native Properties only under the family gate.
   >   `==` focuses `title`, `+` opens the Property Editor, and `-` the drawer.
   > - **Passive.** Painting never writes, formats, or spawns bob.
   >
   > Not Bob Prose's formatter or hanging numbers, a publishing status, or a completion
   > score. See `docs/reference/plugins/prose-card.md`.

   > [!decision] memory_notebook_card Add `glossary:notebook-card` (keyword "Notebook
   > Card", alias "notebook title card"). Adjust this draft to the shipped behavior:
   >
   > The note-top [[property-card]] for a [[notebook]]. Bob Ledger Tools paints it in
   > desktop Live Preview and Reading view, and it replaces the linked notebook header
   > wherever it mounts.
   >
   > - **Identity.** A spine tile and a `NOTEBOOK` kicker, with the owner's crumbs in
   >   the root hue, lead the notebook title and its `summary:` scope line (a quiet
   >   `＋ Add summary` when missing).
   > - **Data.** From the shared `bob notebooks list` report it adds the direct note
   >   count, the newest dated note (an authored `[created::]` date, never file time),
   >   section chips with counts that jump to their `##` headings, child notebooks, and
   >   findings: hard errors as a visible pill, warnings in the foot's lint chip.
   >   Missing report data is omitted, never zeroed.
   > - **Foot.** It carries `created`, the `@stem` capture route (click copies), and the
   >   Properties drawer, where importer keys fold under Managed.
   > - **Keys.** `==` focuses `summary`, `+` opens the Property Editor, `-` the drawer,
   >   and `_` folds the sections.
   > - **Passive.** Painting writes nothing, creates no tasks, and spawns no bob.
   >
   > Not the `[!notebooks]` catalog on an owner note, a task home, or a review queue.
   > See `docs/reference/plugins/notebook-card.md`.

   > [!decision] memory_property_card Edit `glossary:property-card` as `qi.6` left it:
   >
   > - Add the Notebook and Prose Cards to the family list, linking both new strands.
   > - State that both gate on `props_v1` alone and register `==` as extensions.
   > - Name the Prose Card as the inline-title-only exception to recognized-H1 hiding.
   >
   > Change nothing else.

6. **Deploy and provenance.**
   - Run `npm run build:check` and `npm test`.
   - Run `bob plugins sync` from clean master, then the isolated post-sync startup smoke
     in `docs/develop/deployment.md`. `bob plugins list` must show no drift.
   - Record on the bead: the bob-cli and bob-plugins SHAs, the plugin versions, and the
     acceptance evidence refs.
7. **Follow-up.** Record `PROPOSED FOLLOW-UP:` on this bead: once `qj.5`'s
   extract/inline child epic defines provenance fields, the Prose Card gains a quiet
   `↰ from <owner> § <heading>` crumb.

## Epic acceptance (land agent)

- **Notebooks.** Every live notebook opens on one Notebook Card in desktop LP and
  Reading:
  - native Properties are hidden and there is no header;
  - sections jump, children open, and the route copies;
  - errors are visible;
  - zorg keys are folded under Managed.
- **Prose notes.** `sase_blog_contents.md` (now `[[prose]]`) and both spellings in the
  fixtures open on a folded Prose Card:
  - the measure row comes from Bob Prose;
  - the numbering chip toggles;
  - body headings stay intact.
- **Fail-open.** Fail-open, single-owner, and passive evidence is GO.
- **Clean state.** The preview gate is gone, the docs in both repos are consistent,
  memory matches the accepted decisions, and the deploy is clean.
- **Land report.** It reminds Bryan to run `bob plugins sync` on the Mac and apollo, and
  to glance at a notebook and the blog note in his live vault.

## Out of scope

- Any bob-cli Rust or CLI change, a new props wire kind, or an ArticlePlan change.
- Deleting the linked notebook header. It stays as the fallback; revisit after the card
  has soaked.
- A per-section draft ribbon or completion visualization.
- The extract/inline provenance crumb (follow-up above).
- Cards on mobile.
- Moving the `## Notebooks` catalogs.
- Any Bob Mac Capture change.
- Retyping or editing any vault note's content.
