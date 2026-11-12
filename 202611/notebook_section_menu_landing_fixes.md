---
tier: tale
size: medium
title: Finish notebook section menu round-trips and close epic bob-cli-ka
goal:
  Every notebook section menu row (bullet and glued, existing and create, typed capitals
  included) splices, parses, and dry-runs to exactly the section and action it
  advertises; empty notebook menus keep Bob's place_menu chrome; and epic bob-cli-ka is
  closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-ka.land
proposed_with: claude/opus
bead: bob-cli-ka
status: done
---

- **PARENT:**
  [202611/notebook_section_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/notebook_section_bullets.md)
- **BEAD:**
  [bob-cli-ka](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ka/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ka.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ka.land.md)
- **COMMITS:**
  - [f04f277](https://github.com/bobs-org/bob-cli--plans/commit/f04f277ede3a093b25217d3e621cf1786d0d4293)
    — docs(plans): mark notebook section bullets complete

# Finish notebook section menu round-trips and close epic bob-cli-ka

## Why

Epic `bob-cli-ka` ("Notebook section bullets in capture") shipped all six phases, and
its land agent verified most of the contract in a scratch vault on bob-cli master
`300a6e81`. Four completion defects remain, and the epic caused all four. Each breaks a
promise from the epic plan (`plan:202611/notebook_section_bullets.md`) or from
`docs/capture.md` (the "bob-capture-complete" notebook paragraphs, around the
`notebook_section_bullet` section). Fix them in bob-cli, then close the epic. No
bob-mac-capture change is needed: the Mac splices Bob's `replacement` verbatim and
already renders `place_menu.empty_hint` for empty menus (`CapturePlaceMenuView.swift`,
`CapturePanelModel.swift` near `response.placeMenu?.emptyHint`).

The epic's reviewer DECISIONS are final and still apply:
`create_row_return = preselect_clean` and `typed_capitals = yes`. Do not edit memory
(decision records were declined, and the glossary is already done).

Reproduce each defect with a scratch vault
(`bob capture-complete -b <vault> -c <cursor> -f json -- '<draft>'`,
`BOB_DIR=<vault> bob capture-parse -f json -- '<draft>'`). Use a notebook such as
`dev_tools.md` (`type: "[[notebook]]"`, `parent: "[[dev]]"`) with `## Git` and
`## API Design` sections, plus a notebook with no `##` sections.

## Defect 1: empty notebook section menus lose their chrome

This is the proposed follow-up from `bob-cli-ka.5`. `Note @empty_nb#` returns
`place_menu: null`, `query: null`, and `initial_selection: null` when the notebook has
no `##` sections. The same happens for a `%n`-staged notebook with no sections
(`Nvim @dev^nvim@ %n` / `- top` / blank line / `More @dev_nvim#`). With at least one
section, the same draft returns the notebook `place_menu`, `query: ""`, and
`initial_selection: "none"`.

Root cause is in `src/native/capture_complete/engine.rs`. `is_notebook_section_menu`
infers notebook-ness from row kinds, so it is false for zero rows. Three call sites use
it: the `query` override for `CompletionContext::Section`, `place_menu_for`, and the
notebook branch of the initial-selection function.

Fix: decide notebook-ness for the `section` context from the resolved
`super::candidates::notebook_menu_target(...)`, which is `Some` only for policy-visible
notebook routes or a staged `%n` create. Keep the row-kind check as a fallback. Compute
the target once, before `query`, and reuse it for `place_menu_for`. In the
initial-selection function, test `result.place_menu` kind `notebook_section` instead of
the rows.

Leave the read-failure path unchanged: when the notebook cannot be read, there is no
target, so there is still no create row, only the bounded warning. Record, area, and
project section menus must stay chrome-free.

## Defect 2: bullet rows spell multi-word titles as slugs

In `@dev_tools` / `- #`, the `API Design` row's replacement is `api-design\n  - `. The
contract says bullet rows use a title-preferring selector: the bare title when it opens
the bullet scanner and round-trips through the resolver to that heading, otherwise the
quoted exact title (`"API Design"`). Glued `@nb#` rows keep today's bare title → slug →
quoted order.

Root cause is in `src/native/capture_complete/candidates.rs`. `notebook_section_rows` →
`notebook_section_row` → `notebook_section_replacement` always uses the glued order.
Thread a spelling mode (bullet vs glued) from `notebook_bullet_candidates` (bullet) and
`section_candidates_with_raw` / `notebook_section_candidates` (glued) down to
`notebook_section_replacement`. In bullet mode, skip the slug step. Refused (ambiguous)
rows stay as they are.

## Defect 3: create rows drop typed capitals, which breaks the round-trip

`@dev_tools` / `- #API-keys` offers a create row titled `API Keys`
(`Creates ## API Keys …`), but its replacement is `api-keys\n  - `. Splicing it and
running `capture-parse` reports a section titled `Api Keys` with
`section_action: create`, a different heading from the one the row advertised. The glued
`Note @has_nb#API-keys` has the same problem. This violates the epic's decisive
round-trip invariant: every enabled row must parse and dry-run to exactly its advertised
section and action.

Root cause is in the unquoted `NotebookSectionResolution::Create` arm of
`notebook_section_rows`. It replaces the title with `selector_slug(&title)`, which
lowercases the title.

Fix: choose the create replacement by round-trip. Use the bare typed query when
`resolve_notebook_section(query, false, &headings)` creates exactly the advertised
title. Otherwise use the slug if it creates that same title. Otherwise use the quoted
title. Keep the existing near-miss quoted spelling and the authored-quote preservation.
Update the sentence in `docs/capture.md` that says the create row's replacement "is the
bare slug, or the quoted selector when the bare slug would be a near miss" so it
describes the round-trip choice.

## Defect 4: the glued menu lists the item under the caret as a draft section

`Note @has_nb#Zeta` (no `Zeta` heading) shows a `draft` row:
`Zeta · new · Adds to ## Zeta, new in this draft`. It should show the
`notebook_section_create` row (`＋ New section “Zeta”`,
`Creates ## Zeta at the end of Has`, group `new`, `creates: true`). The caret item is
treated as an earlier operation that creates the section. The contract says later
operations, failed items, and the selector under the caret never appear in the staged
view.

`capture-parse` and the earlier-item case (`Earlier @has_nb#Zeta` / blank line /
`Later @has_nb#Ze` correctly lists `Zeta` as a draft row) already behave. The header
form `@has_nb#Ze` / `- x` with the caret after `Ze` also behaves.

Root cause, likely: in `staged_notebook_sections`
(`src/native/capture_language/ notebook.rs`), the glued operation from
`notebook_section_operations` uses the section range start (`range.0`, at or before the
`#`) as its offset. The menu passes the selector start (after the `#`) as `at_offset`,
so `operation.offset < at_offset` keeps the caret's own operation.

Fix: exclude any operation whose selector range contains or touches `at_offset`. Earlier
bullets in the same list or `%n` item must stay included; the existing staged-view tests
cover that. Do not re-derive syntax in a second lexer.

## Tests

Add CLI regressions in `tests/cli/capture/complete_notebook_sections.rs` (reuse its
`vault`, `complete_at`, `parse`, and `dry_run` helpers; add an empty notebook to the
fixture vault):

- Empty notebook: `Note @<empty>#`, the header `@<empty>#` with the caret after `#`, and
  a `%n`-staged empty notebook each return `place_menu.kind == "notebook_section"` with
  `empty_hint == "Type a section name"`, `query == ""`, `initial_selection == "none"`,
  and no rows. A plain area-note `@dev#` section menu still has no `place_menu`.
- Bullet spelling: the `API Design` bullet row's replacement is `"API Design"\n  - `,
  and the glued `Note @dev_tools#` row for the same heading keeps its glued spelling.
  Splicing the bullet row parses and dry-runs to `API Design` / `existing`.
- Typed-capitals create round-trip, in both the bullet and glued contexts: the
  `#API-keys` create row splices, parses, and dry-runs to `API Keys` / `create`. Extend
  the existing round-trip test so every enabled row, create rows included, is checked
  for a mixed-case query, not only lowercase queries.
- Glued self-exclusion: `Note @dev_tools#Zeta` returns one `notebook_section_create` row
  (group `new`, `creates: true`) and no `draft` row. The earlier-item case still lists
  the draft row.

Run `cargo test --test cli capture::` and `cargo test --lib capture`, then `just check`
(the file-change gate; do not run `just check-full`). If help text or docs change, run
`just docs-gen` and `just docs-check`.

On master `300a6e81`, `just docs-check` already fails its strict Zensical build on three
broken links that this epic did not cause. They are tracked as task `bob-cli-kk` and as
a note on epic `bob-cli-kb`. Do not treat them as this tale's work. Confirm only that
your change adds no new docs-check findings.

Commit as `fix(capture): …` (no grammar bump, no `Cargo.toml` version or `CHANGELOG.md`
edits; releases ride the daily train).

## Epic closeout (final step)

After the fixes are committed in this same turn:

1. Run `sase bead epic-symbols bob-cli-ka`. At planning time it printed "No
   --epic-symbol entries for bob-cli-ka." If any entries appear, resolve each one: wire
   it up, privatize it, add a non-test pragma, or delete it, per the Symvision
   epic-whitelist policy. No later open bead needs this epic's exemptions.
2. Close the epic: `sase bead close bob-cli-ka --note "<verification>"`. The note must
   summarize the land audit: all six phases verified against source and a scratch vault
   (resolver ambiguity and typed capitals, staged view, grammar-26 section bullets with
   placement/merge/refusals/`\#`, the menu and `place_menu`, the `- /` alias, the
   glossary, and the Mac place menu green on exact-SHA macOS CI run 43117124821 for
   bob-mac-capture 7fceb52). It must also list the four defects this tale fixed with
   their commit SHA, and the follow-up triage already recorded on the epic (the "LAND
   TRIAGE" note: the bob-cli-jy +1, filed bob-cli-kk, the bob-cli-kb note, and the
   resolved or declined proposals). Never use `--force`.
3. Run `just symvision` if the recipe exists. It was absent from the justfile at
   planning time; if so, say so in your final response.
4. Set `status: done` in the frontmatter of the epic's plan file. That file is
   `plan:202611/notebook_section_bullets.md`; use the PLAN path that
   `sase bead read bob-cli-ka -r "<why>"` prints. It currently says `status: wip`.
5. Epic `bob-cli-ka` has no `parent_bead`, so the landing ends there.
