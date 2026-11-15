---
tier: tale
title: Allow punctuation in task drawer titles
goal: Drawers such as WHAT IS SASE? are scanned, offered by the bullet-start
size: medium
decisions:
  punctuation:
    ask: Which new punctuation should a drawer title accept?
    choices:
      sentence:
        "Adds ? ! ; anywhere plus a trailing : (OPTIONS:); LABEL: value notes stay plain
        notes"
      question_bang:
        "Adds only ? and !; colon-ended headers like OPTIONS: stay plain notes"
      broad:
        "Adds ? ! ; : anywhere except ::; ALL-CAPS LABEL: value notes also become
        drawers"
    default: sentence
    why:
      "Covers questions and the colon headers already in the vault, but not LABEL: value
      notes"
    answer: sentence
  slug_reach:
    ask:
      "Should typing the slug #what-is-sase also select the punctuated drawer WHAT IS
      SASE?"
    choices:
      exact:
        "Punctuation stays part of drawer identity, like Q&A; #what-is-sase gets a
        fixable refusal"
      punctuation_blind:
        "#what-is-sase selects WHAT IS SASE?; more resolver work and new ambiguous-slug
        cases"
    default: exact
    why: "One identity rule, like existing Q&A drawers; the - # menu works either way"
    answer: exact
proposed_by: bbugyi200.athena.1cq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 13:19:15
status: wip
---

# Allow punctuation in task drawer titles

## Goal

Bryan wants to rename the `WHAT IS SASE` drawer under `^section-field` in `sase_blog.md`
to `WHAT IS SASE?` and still use the bullet-start `#` drawer menu in Bob Mac Capture.
That means drafting `@sase_blog+section-field` and then `- #`, choosing the drawer, and
adding notes. The drawer must be listed under "On this task", and choosing it must
insert `- WHAT IS SASE?`. On commit, that bullet must merge into the existing drawer,
never write a second header.

Today none of this works. `capture_drawers::is_drawer_title` accepts only `A-Z`, `0-9`,
space, tab, and `& ' ( ) , . / -` (lead character `A-Z` or `0-9`, and at least one
letter). As a result:

- The vault scanner (`drawers()` → `parse_drawer_title`) does not see `WHAT IS SASE?` as
  a drawer. The completion menu never lists it, and the catalog never records it.
- `is_all_caps_drawer_title` rejects the bare `- WHAT IS SASE?` bullet, so capture would
  file it as a plain note with a nested note.

Probe on the current binary: `bob capture-parse -f json` on `@sase_blog+section-field` /
`- WHAT IS SASE?` / `  - a note` reports no `drawer_bullet` span. The same draft without
`?` does.

Bob Mac Capture is a thin client (`decisions:mac-capture-is-a-thin-client`). It has no
drawer-title logic of its own: it renders the rows, spans, and `replacement` strings
that bob returns. **No bob-mac-capture change is needed**, and the capture contract
keeps the same JSON shape.

## Every copy of the drawer-title rule

Change all of these together. They must agree.

| Where                                                                                                                                                                               | What                                                                                                                                                                                        | Action                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| bob-cli `src/native/capture_drawers.rs` `is_title` / `is_title_rest`                                                                                                                | `is_drawer_title` and `is_pomodoro_name` share one predicate                                                                                                                                | Split so that only drawer titles gain the new punctuation                                   |
| bob-cli `src/native/capture_project_note.rs` `is_project_section_title`                                                                                                             | Hand-copied duplicate of the drawer charset, used by project-note capture sections, notebook seeding (`capture_language/section_bullet.rs`), and record-note sections (`records/render.rs`) | Delegate to `is_drawer_title(body.trim())` and delete the duplicate charset                 |
| bob-plugins `plugins/bob-navigation-hotkeys/src/256-task-split-plan.js` `isTaskSplitDrawerTitle`                                                                                    | JS twin used by task split and by `buildProjectSeedFromChildBullets` (task → project promotion: drawer → `##`)                                                                              | Same new rule                                                                               |
| bob-plugins `plugins/bob-navigation-hotkeys/src/010-requires-and-config.js` `PROJECT_SECTION_TITLE_RE`, used only by `formatProjectReversalSectionTitle` in `100-yank-and-paths.js` | Project reversal (`##` → drawer)                                                                                                                                                            | Call `isTaskSplitDrawerTitle(title)` instead and remove the regex, leaving one JS predicate |

Everything else calls those predicates and picks up the change on its own:

- `normalize_drawer_literal`, which handles quoted `#"…"` selectors, the bullet menu,
  editor spans, transfer, and merge-in.
- `is_all_caps_drawer_title`, which handles capture bullets, the dry-run preview,
  commit, records, and the duplicate-header refusal in `capture/plan.rs`.
- `task_children::classify_task_child`, which handles task merge and task split.
- `gcal/description/bullets.rs`. It bolds drawer-shaped Google description lines so they
  never read as drawers under an Event Task.

Leave these alone; they are Pomodoro-name and theme grammar, not drawers:

- `is_pomodoro_name`
- plugin `075-pomodoro-link-targets.js`
- plugin `279-plugin-task-theme-model.js`
- plugin `315-task-card-theme-stage.js`

## Step 1: the predicate (bob-cli)

Rework `is_title(title, allow_plus)` into a grammar-aware predicate. For example, use an
enum `TitleGrammar { Drawer, PomodoroName }`, or a separate extra-character set:

- **Shared base:** lead character `A-Z` or `0-9`; the rest from `A-Z`, `0-9`, space,
  tab, and `& ' ( ) , . / -`; at least one `A-Z`.
- **Pomodoro names:** base plus `+` in a non-leading position. Unchanged.
- **Drawer titles:** base plus the punctuation chosen below. `+` stays out.

> [!decision] punctuation = sentence Allow `?`, `!`, and `;` anywhere after the lead
> character. Allow `:` only as the title's final character, so `OPTIONS:` and
> `WHAT NEXT:` qualify. `NOTE: X`, `PHONE: (609) 597-4111`, `KEY:: VALUE`, and
> `OPTIONS::` do not.

> [!decision] punctuation = question_bang Allow only `?` and `!` anywhere after the lead
> character. `:` and `;` stay refused.

> [!decision] punctuation = broad Allow `?`, `!`, `;`, and `:` anywhere after the lead
> character. Refuse any title that contains `::`, so a Dataview `KEY:: value` inline
> field never reads as a drawer.

Under every choice, these stay refused because they are Markdown or capture syntax: `"`,
`#`, `*`, `_`, `` ` ``, `[`, `]`, `^`, `@`, `$`, `%`, `=`, `~`, `+`, `<`, `>`, `|`, `{`,
`}`, `\`. Excluding `"` keeps the quoted `#"TITLE"` literal unambiguous.

Other Step 1 changes:

- Update the doc comments on `is_drawer_title`, `is_pomodoro_name`, and
  `parse_drawer_title`. They currently say a Pomodoro name is "the drawer title grammar
  plus `+`", and that is no longer true.
- Make `capture_project_note::is_project_section_title` return
  `capture_drawers::is_drawer_title(body.trim())`. Keep its existing callers and fix its
  doc comment.

## Step 2: catalog cache and near-duplicate copy (bob-cli)

- Bump `DRAWER_CATALOG_VERSION` in `capture_drawers.rs` from `1` to `2`. The per-note
  cache is keyed by mtime/size, so without the bump, unchanged notes would never be
  rescanned for newly qualifying drawers. Check that the existing stale-version cleanup
  removes `catalog-v1-*` files. Then grep for any other persisted cache that stores
  drawer classification and invalidate it the same way.
- Fix the slug-path near-duplicate refusal in `resolve_drawer_for_write`
  (`did you mean `#{near.slug}`? …`). It currently prints the drawer's `selector_slug`
  even when that slug is not a valid slug (`what-is-sase?`, `q&a`), which suggests a
  selector that cannot parse.
  - When `!is_drawer_slug(&near.slug)` and `normalize_drawer_literal(&near.title)` is
    `Some(key)`, suggest `#"{key}"` instead.
  - Keep the rest of the message (`write #"…" to create …`) as it is.

> [!decision] slug_reach = exact No resolver matching change. Slug selectors keep
> comparing exact keys, so `#what-is-sase` does not match `WHAT IS SASE?`. The
> near-duplicate guard refuses it with `did you mean `#"WHAT IS SASE?"`? write `#"WHAT
> IS SASE"` to create WHAT IS SASE`. Glued `@route+id#what` menu rows keep their
> existing fallback to the quoted replacement `#"WHAT IS SASE?"`, because the title's
> `selector_slug` is not a valid slug.

> [!decision] slug_reach = punctuation_blind Define `bare_drawer_key(key)` as the key
> with the newly allowed punctuation removed and whitespace re-collapsed.
>
> - On the slug path only: in `resolve_drawer_for_write` exact-key matching and in
>   `find_near_duplicate_drawer`, a drawer also matches when its bare key equals
>   `key_plain` or `key_spaced`. More than one match keeps the existing
>   `ambiguous_drawer` refusal ("use a quoted literal to pick one").
> - Quoted and ALL-CAPS (exact) selectors stay byte-exact on the key.
> - Slug-path creation never adds punctuation: `drawer_spelling_for_slug` stays
>   key-exact, so `#what-is-sase` with no match creates `WHAT IS SASE`.
> - In `capture_complete/drawers.rs` `choose()`, when `selector_slug(title)` is not a
>   valid slug, try the slug of the bare title. Use it as `#slug` if it resolves to the
>   same drawer (`matches_want`). Otherwise keep the quoted fallback.
> - Still apply the Step 2 suggestion fix for titles whose bare slug is invalid, such as
>   `Q&A`.

## Step 3: bob-cli tests

Add or adjust tests. Each one names the behavior it pins.

- **`capture_drawers.rs` unit tests.**
  - `parse_drawer_title` / `is_drawer_title` accept `WHAT IS SASE?`, `WOW!`, and the
    chosen `;`/`:` forms.
  - They refuse a punctuation lead (`?WHY`), `KEY:: VALUE`, `C++`, and the forms the
    chosen branch excludes.
  - `is_pomodoro_name("WHY?")` stays false.
  - A task with a `WHAT IS SASE?` child enumerates it from `drawers()` with key
    `WHAT IS SASE?`.
- **`capture_project_note.rs`.** `section_title_shape_rejects_mixed_case_and_banners`
  currently asserts that `WHAT'S NEXT?` is refused. Flip that assertion, rename the test
  if "banners" no longer fits, and add a refused excluded form.
- **`capture_language/tests/drawer_bullet.rs`.**
  - `@route+id` / `- WHAT IS SASE?` / `  - note` reports a `drawer_bullet` span titled
    `WHAT IS SASE?`.
  - `- #"what is sase?"` resolves to the title `WHAT IS SASE?`.
- **`capture_complete/tests/drawers.rs`.** Use a vault whose task has a `WHAT IS SASE?`
  drawer.
  - The bullet-start `- #` menu lists it under the task group, not refused, with
    `replacement` `WHAT IS SASE?`.
  - Typing `- #what` keeps it as a prefix match.
  - The glued `@route+id#what` row's replacement follows the `slug_reach` branch.
  - Mirror the existing test helpers (`vault`, `isolate_cache`, `menu`).
- **Capture commit or dry-run test.**
  - `@route+id` / `- WHAT IS SASE?` / `  - new note` lands as the last entry of the
    existing drawer, with no second header.
  - A childless `- WHAT IS SASE?` repeat gets the existing `empty_drawer_bullet` / "is
    already a drawer" refusal.
- **Near-duplicate copy.** Test `#what-is-sase` against an existing `WHAT IS SASE?`
  drawer for the branch's outcome: the quoted suggestion under `exact`, or a selection
  under `punctuation_blind`.
- **gcal description.** A Google description line `WHY?` renders bold as `- **WHY?**`,
  like other drawer-shaped lines. Existing synced descriptions whose blocks are
  untouched re-render once on the next sync. The recorded-hash window logic in
  `gcal/plan.rs` replaces them silently, with no "kept" report, which is acceptable.
  Confirm with the existing `tests/fixtures/gcal_description` suite.
- **Task-split twin vector.** Add `SP31` to bob-cli
  `tests/fixtures/task_split/vectors.json`.
  - Case: an owner task with a `WHAT IS SASE?` drawer holding one note.
  - Split that note, either with a cursor selection or with
    `section {title: "WHAT IS SASE?"}`.
  - The new task record carries `drawer: "WHAT IS SASE?"`.
  - Take minted IDs and postimages from the real planner output (run the vector test and
    read its diff); never guess them.
  - Update `tests/fixtures/task_split/README.md` (`SP1`–`SP31`) and the case table in
    `docs/task-split.md`.

Do **not** bump `capture_language::GRAMMAR_REVISION`. No client-visible shape changes:
the `drawer_bullet` span, the menu rows, and the refusal codes already exist, and
clients never evaluate the charset. Bumping it would also churn the Mac app's
grammar-gated fixtures for no gain.

## Step 4: bob-plugins

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins`) and read its
`AGENTS.md`.

- **The predicate.** Change `isTaskSplitDrawerTitleRest` and `isTaskSplitDrawerTitle` in
  `256-task-split-plan.js` to the exact rule chosen in Step 1. Include any position rule
  (such as trailing-only `:`) and the `::` refusal. Update its header comment.
- **Project reversal.** Make `formatProjectReversalSectionTitle` in
  `100-yank-and-paths.js` use `isTaskSplitDrawerTitle(title)` instead of
  `PROJECT_SECTION_TITLE_RE.test(title) || /[A-Z]/`. Delete `PROJECT_SECTION_TITLE_RE`
  from `010-requires-and-config.js` and fix the comment near line 886 that references
  it.
- **Fixture copy.** Copy bob-cli's `tests/fixtures/task_split/vectors.json`
  byte-for-byte to
  `plugins/bob-navigation-hotkeys/scripts/fixtures/task_split/vectors.json`. Never
  hand-edit the copy.
- **Tests.** In `scripts/test-navigation-hotkeys-project-from-task.cjs` and
  `scripts/test-navigation-hotkeys-project-reversal.cjs`, a `WHAT IS SASE?` drawer with
  a note promotes to `## What Is Sase?` and reverses to the `WHAT IS SASE?` drawer key.
  Add a refusal case for an excluded form.
- **Build and deploy.** Edit only the `src/` fragments. Run `npm run build` and
  `npm test`, which includes the build check and
  `test-navigation-task-split-vectors.cjs`. Then run `bob plugins sync`. If sync refuses
  an overwrite, follow that repo's `AGENTS.md` procedure. Never pass `-F` on your own.

## Step 5: docs (bob-cli)

Read reference memory `docs.md` first. This is a behavior change, so update the contract
text in `docs/capture.md`:

- **Drawer definition.** Where drawers are introduced (the `- FUTURE WORK` rows near the
  top and the first-level ALL-CAPS drawer paragraph near line 1025), state the drawer
  title grammar once and plainly. Use `WHAT IS SASE?` as the example.
- **Project-note sections (~line 5922).** Replace the "`& ' ( ) , . / -`" charset with
  the new grammar, or point to the definition above.
- **Pomodoro selector slug (~line 1623).** It says "That is the drawer selector
  character set plus `+`". Restate it as its own explicit set, because drawers now
  differ.
- **`capture-pomodoro-name` (~line 10823).** "requires the drawer title grammar plus
  `+`": same fix. List the Pomodoro charset explicitly.
- **Quoted literals.** Mention that punctuated drawers are reached through the menu or a
  quoted `#"WHAT IS SASE?"`, matching the `slug_reach` branch.

Run `just docs-check`. Do not edit `Cargo.toml`'s version or `CHANGELOG.md`; the release
train owns them (`decisions:bob-cli-releases-ride-a-daily-train`). This is not a
CLI/JSON contract break, so use a plain `feat(capture):` commit type with no `!`.

## Step 6: verification

1. In bob-cli, run `just` gates covering fmt, clippy, and the full `cargo test` suite,
   or the repo's documented equivalents. Both repos must be green.
2. End-to-end with the debug build against a throwaway vault, never Bryan's. Use
   `BOB_DIR` and a temp HOME/cache so the drawer catalog cache stays isolated.
   - The vault holds a note with `- [ ] #task Add section field ^section-field` and a
     child `- WHAT IS SASE?` with three notes.
   - `bob capture-complete -f json --cursor <end> -- $'@note+section-field\n- #'` lists
     `WHAT IS SASE?` in the task group, not refused, with `replacement` `WHAT IS SASE?`.
     This is the command the Mac app runs.
   - `bob capture --dry-run` of the draft that choosing the row produces
     (`- WHAT IS SASE?` / `  - fourth note`) previews one new entry inside the existing
     drawer.
   - `bob capture-parse -f json` on that draft reports the `drawer_bullet` span.
3. Read-only vault audit. Do not modify `~/bob`. Run a throwaway script or `rg` over the
   vault to list direct `#task` children that newly qualify as drawers, and report them
   in the final summary. A pre-plan probe of the vault found these:
   - Under `sase_better_llm_disable.md:24` and `needs_attn_tasks.md:28`, the bullets
     `OPTIONS:` and `NOTES:` qualify under `sentence` or `broad`.
   - `done/body_done.md:31` `PHONE: (609) 597-4111` qualifies under `broad` only.
   - The vault has no `?`/`!`/`;` titles yet.

   Leave Bryan's own rename of `WHAT IS SASE` to him.

## Out of scope

- Any bob-mac-capture change. It has no local drawer grammar.
- Pomodoro-name or theme grammar.
- Unicode or emoji drawer titles.
- Shipping. The daily release train delivers the new `bob` to the Mac.
- Memory notes. The `glossary:task-drawer` strand and
  `decisions:capture-bullets-mirror-the-task-block` say "ALL-CAPS" without enumerating
  characters, so they stay true.
