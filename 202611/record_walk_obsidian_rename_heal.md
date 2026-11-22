---
tier: tale
title: Heal Obsidian-rewritten record walk lines
goal:
  Fix the two damaged SASE Blog Zettel walk lines now, and teach bob albums sync (and
  its plugin mirror) to heal Obsidian rename rewrites automatically.
size: medium
decisions:
  arm_scope:
    ask:
      Should a walk line count when an arrow-labelled arm points outside the album, not
      just at a member's bare stem?
    choices:
      arrow:
        Any ‹/› arm beside the ↑ hub counts; renames, moves out, and every link format
        heal
      member:
        Only bare member-stem arms count; both classifiers need member lists; moved-out
        arms stay damaged
    default: arrow
    why:
      An all-link run around the album's ↑ hub is already bob's line; vectors stay
      self-contained
    answer: arrow
proposed_by: bbugyi200.athena.1f2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 06:40:58
status: wip
---

# Heal Obsidian-rewritten record walk lines in `bob albums sync`

## Problem

The SASE Blog Zettel album card warns:

> sase_blog_zettel.md has 2 records with damaged walk lines (first:
> sase_blog_zettel/layers_of_prompt_management.md); delete the offending lines and run
> `bob albums sync`

**Root cause (verified in the vault history).** Vault commit `eb58099b3` renamed
`sase_blog_zettel/the_prompt_input_widget.md` to
`sase_blog_zettel/the_prompt_composer.md` in Obsidian. Obsidian's link updater then
rewrote every link to that record in its "shortest path" form. Bob-owned bytes changed
in three places:

- the `[!records]` index in `sase_blog_zettel.md`. It is regenerated wholesale, so the
  next sync (`a1db2ce3f`) already restored
  `[[sase_blog_zettel/the_prompt_composer|The Prompt Composer]]`;
- `sase_blog_zettel/layers_of_prompt_management.md` walk line:
  `[[sase_blog_zettel/agent_control_flow|‹ Agent Control Flow]] · [[sase_blog_zettel|↑ SASE Blog Zettel]] · [[the_prompt_composer|The Prompt Input Widget ›]]`;
- `sase_blog_zettel/memory.md` walk line:
  `[[the_prompt_composer|‹ The Prompt Input Widget]] · [[sase_blog_zettel|↑ SASE Blog Zettel]] · [[sase_blog_zettel/usage_limits_and_api_costs|Usage Limits and API Costs ›]]`.

`src/native/albums/walk.rs` recognizes a walk line only when every link targets
`<album>` or `<album>/<child>` (`allowed_target`). The bare `the_prompt_composer` target
fails that test, so `is_walk_line` returns false. The `↑` hub link still makes
`has_nav_like_link` true, so `classify` returns `Damaged` with `extra_text`.
`plan_line_edit` never writes damaged lines. The plugin watcher already schedules an
automatic `bob albums sync` on album-member renames (`albumScheduleSync("album-rename")`
in bob-plugins `241m-plugin-album-data-service.js`). That sync ran, but because these
lines were classified as damaged it skipped them, so a routine rename leaves a permanent
warning.

`bob albums sync -d` currently reports everything else current, and these two records
are the only damage (`bob albums check`: 1 warning, 0 errors).

## Should `bob records tidy` auto-fix this? No: `bob albums sync` should

- `docs/records.md` ("Record walk" → "Owners and radius") says `bob albums sync` is the
  full repair owner, and that `bob records tidy` never touches walk lines and only sees
  past them. Adding walk repair to tidy would split ownership. Tidy would also have to
  rebuild the album ring projection to know the canonical line, duplicating
  `plan_album_walk`.
- Sync already runs automatically: from the plugin's rename/frontmatter watcher, the
  record walk rail's mismatch repair, and nightly. Once sync _recognizes_ an
  Obsidian-rewritten line, the line becomes `stale` rather than `damaged`, and that same
  automatic sync rewrites it in place. A rename then heals with no command at all, which
  a manual `bob records tidy` run cannot match.
- The damage reasons that remain (`extra_text` for authored prose around a `↑` link,
  `no_frontmatter`, `duplicate_lines`) involve bytes bob cannot safely claim, so they
  stay reported and are fixed by hand, as documented.

So the fix teaches the shared walk-line recognizer (Rust plus its plugin mirror) about
Obsidian's rewritten arms. Tidy stays as it is.

## Recognition rule

> [!decision] arm_scope = arrow

A line is a walk line when it is a run of at most three wikilinks joined by `·` with
exactly one hub link (`allowed_target == Some(true)`, label starting with `↑`). Every
other link must be either:

1. an album child (`allowed_target == Some(false)`), with any label, as today; or
2. **a foreign arm (new):** any other target, provided it sits _immediately before_ the
   hub with a label starting with `‹`, or _immediately after_ the hub with a label
   ending with `›`. Use only the canonical glyphs `‹`/`›`, not ASCII `<`/`>`.

A foreign link in any other position, or with any other label, still disqualifies the
line. The line then stays `extra_text` damage through `has_nav_like_link`.
`has_nav_like_link`, `allowed_target`, the `↑` hub rule, fence skipping, and the
`duplicate_lines` / `no_frontmatter` checks are unchanged. A recognized line whose bytes
differ from the canonical line is `stale`, and `plan_line_edit` already rewrites it in
place.

> [!decision] arm_scope = member
>
> In place of rule 2, accept a foreign arm only when its target, minus `.md`, has no `/`
> and equals the stem of a current album member. Pass the member stem set into
> `classify` (Rust: from the projected `slots` in `plan_album_walk`; JS:
> `recordWalkClassify(markdown, albumStem, canonical, memberStems)` from
> `recordWalkModel`'s `members`). Add a `members` array to every recognition vector, and
> update both vector consumers to pass it. Position and arrow-label rules from rule 2
> still apply.

## Changes

### bob-cli

1. **`src/native/albums/walk.rs`**: restructure `is_walk_line` so it parses the link run
   first, then finds the single hub index and validates each other link under the rule
   above. Keep `parse_link_run` (it already accepts any target) and `allowed_target` as
   they are. Add a small helper, for example `is_foreign_arm(index, hub, label)`.
   `is_walk_or_nav_line`, used by `records/notes_section.rs` (tidy), picks the change up
   automatically, which is the intended "see past walk lines" behavior. Update the
   `classify` and `plan_line_edit` doc comments to mention the rename case.
2. **Unit tests in `walk.rs`**: extend
   `parser_accepts_canonical_and_rejects_extra_text`, or add a sibling test, covering:
   - a shortest-path next arm (`[[eat/pho|‹ Pho]] · [[eat|↑ Eat]] · [[ramen|Ramen ›]]`),
     accepted;
   - a shortest-path prev arm, accepted;
   - a moved-out arm (`[[drinks/ramen|Ramen ›]]` after the hub), accepted;
   - a foreign link without an arrow (`[[ramen|Ramen]]`), rejected;
   - a foreign `‹` arm _after_ the hub, rejected;
   - a foreign arm not adjacent to the hub, rejected;
   - a foreign link carrying `↑` (a second hub-like link), rejected.

   Add one `plan_line_edit` test: the rewritten line (CRLF-preserving) is replaced in
   place with the canonical line, and a second plan is `None`.

3. **Shared vectors `tests/fixtures/record_walk/vectors.json`** (`recognition`): add
   cases named `obsidian-shortest-path-next-arm-is-stale`,
   `obsidian-shortest-path-prev-arm-is-stale`, `moved-out-arm-is-stale`,
   `foreign-link-without-arrow-is-extra-text`, and
   `foreign-arm-on-wrong-side-is-extra-text`, in the existing case shape
   (`album: "eat"`, canonical three-link line). Keep the file's existing formatting
   style; the plugin copy must stay byte-identical.
4. **CLI integration test `tests/cli/albums_sync.rs`**: add
   `walk_sync_heals_an_obsidian_rename_rewrite`, modelled on
   `walk_sync_leaves_a_damaged_line_and_warns`. Build a three-record `walk_fixture` and
   run sync once so the lines are canonical. Then simulate the rename: move one record
   file to a new stem and rewrite both neighbours' arms to the bare new stem, keeping
   the old labels. Commit, then assert:
   - `check -f json` reports `walk.state == "stale"` (not `damaged`) with no
     `album_walk_damaged` issue;
   - `sync -f json` rewrites exactly the expected walk lines to canonical (new stem path
     and new title label);
   - a second sync is a no-op.
5. **Docs**:
   - In `docs/records.md` → "Record walk", make the **States** bullet say a walk line
     may carry foreign arrow arms (the rule above, in one or two sentences, with the
     `[[the_prompt_composer|‹ …]]` example). State that an Obsidian rename or move
     therefore leaves neighbours `stale`, and that the next sync (including the plugin's
     automatic rename sync) rewrites them in place. Keep "Deleting the offending line
     heals a damaged record on the next sync" for the remaining reasons. Leave the
     **Owners** bullet's tidy sentence as is.
   - `docs/reference/plugins/record-card.md` is mounted from bob-plugins; do **not**
     edit it here.
   - No CLI flag, help text, JSON field, or reason value changes, so `just docs-gen` is
     not needed. Run `just docs-check`.
6. Do not touch `Cargo.toml`'s version or `CHANGELOG.md` (release train). Use a
   `fix(albums): …` commit subject; this is not a contract break.

### bob-plugins (open with `/sase_repo`; read its `AGENTS.md` first)

7. **`plugins/bob-ledger-tools/src/242t-plugin-record-walk-model.js`**: mirror the rule.
   `recordWalkParseLinkRun` currently returns `null` as soon as
   `recordWalkAllowedTarget` fails. Change it to keep such links as
   `{ target, label, root: false, foreign: true, stem }`, where `stem` is the target's
   last path segment without `.md` or any `#…`/`^…` suffix. Then make
   `recordWalkIsWalkLine` enforce the hub-adjacency and arrow-label rule for foreign
   links. `recordWalkHasNavLikeLink` is unchanged. In `recordWalkModel`, treat a foreign
   `writtenPrev` or `writtenNext` as a neighbour mismatch, so the state is `"syncing"`
   with reason `"neighbors_mismatch"`. This matters because `recordWalkOpen` resolves
   `<target>.md` by vault path, so a bare-stem arm is not a verified destination. The
   rail then shows `⟳` and requests its one album sync for the episode, which heals the
   line once a bob with this fix is installed. Keep the fragment under 1000 lines.
8. **Vectors copy**: copy bob-cli's updated `tests/fixtures/record_walk/vectors.json`
   byte-for-byte to `plugins/bob-ledger-tools/scripts/fixtures/record_walk/vectors.json`
   and verify with `cmp`.
9. **Tests `scripts/test-ledger-tools-record-walk.cjs`**: the vector loop covers
   recognition. Add a model test using the existing `recordWalkWorld()` helper. A source
   whose next arm is `[[c|Charlie ›]]` (bare stem) must yield `lineState: "stale"`,
   `state: "syncing"`, `reason: "neighbors_mismatch"`, a `writtenNext.target` of `"c"`,
   and a rail (`recordWalkHasRail`) that is still present. If a record-sheet
   title-recognizer test covers walk lines, add one case showing that a foreign-arm walk
   line is still hidden as the walk line.
10. **Docs `docs/record-card.md`** ("Record walk rail"): add one sentence. A line whose
    arm Obsidian rewrote after a rename or move is still recognized; the rail shows `⟳`
    and requests one album sync, which rewrites it.
11. Run `npm run build` (never hand-edit `main.js`) and `npm test` (includes
    `build:check`) or at least `npm run build:check` plus
    `node --test scripts/test-ledger-tools-record-walk.cjs scripts/test-ledger-tools-record-sheet.cjs`.
    Also run `npm run docs:check`, then **`bob plugins sync`** (follow the repo's
    AGENTS.md if it refuses an overwrite; never pass `-F` on your own).

**Deploy-order note.** The plugin deploys immediately; the bob-cli change rides the next
daily release. Until that release, an installed bob still calls these lines damaged. The
plugin's mismatch request is once per episode, so it cannot loop.

### Vault repair (the reported errors, today)

12. **Real-data check first (read-only).** From the bob-cli checkout, run
    `cargo run --quiet -- albums sync -d -b ~/bob`. With the new recognizer, the planned
    changes must be exactly two `kind: "walk"` rewrites, of
    `sase_blog_zettel/layers_of_prompt_management.md` and `sase_blog_zettel/memory.md`,
    now pointing at `[[sase_blog_zettel/the_prompt_composer|… The Prompt Composer …]]`.
    There must be no index or catalog changes. If anything else appears, stop and report
    instead of repairing.
13. **Repair with the installed `bob`** (do not write the vault with an unreleased
    build). In `~/bob/sase_blog_zettel/layers_of_prompt_management.md` and
    `~/bob/sase_blog_zettel/memory.md`, delete only the single damaged walk line (the
    one containing `[[sase_blog_zettel|↑ SASE Blog Zettel]]` and
    `[[the_prompt_composer|`), leaving every other byte alone. Then run
    `bob albums sync -d` and confirm it plans exactly those two walk-line insertions.
    Run `bob albums sync`, which takes the vault lock, pre-syncs, and commits
    `albums: sync (2 notes)`. Confirm that `bob albums check` reports 0 warnings for
    `sase_blog_zettel.md`, that `bob records check zettel` shows no
    `record_walk_damaged`, and that the two lines now equal the canonical lines from
    `bob records show zettel memory -f json` / `… layers_of_prompt_management`.

The user-authored task text in `sase.md` (`Rename the [[the_prompt_composer]] record!`)
is a normal Obsidian link and is not touched.

## Validation

- bob-cli: `cargo test --lib albums::walk`, `cargo test --test cli albums_sync`, then
  `just check` (fmt, clippy gate, full test suite) and `just docs-check`.
- bob-plugins: the commands in step 11, `cmp` of the two `vectors.json` files, and
  `bob plugins sync`.
- Vault: the step 12 dry run output and the step 13 post-repair checks.

## Out of scope

- Auto-repairing `duplicate_lines`, `extra_text`, or `no_frontmatter` damage, and any
  walk repair in `bob records tidy` (see the rationale above).
- Album _note_ renames (they change the album stem itself) and Obsidian rewrites of
  other bob-generated regions (`## Albums` catalogs, Depends-On lines). If a similar
  rename failure turns up there, file a separate task bead via `/sase_new_task`.
- Changing the `album_walk_damaged` / `record_walk_damaged` message text.
