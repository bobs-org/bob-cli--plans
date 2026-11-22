---
tier: tale
title: Group the album Records index by an album-chosen choices field
goal:
  The sase_blog_zettel Records index groups zettels under their section labels in
  declared outline order.
size: medium
decisions:
  index_key:
    ask: Which album setting should choose the Records index headings?
    choices:
      own_key: New top-level `index_group_by` key; albums without it never change
      follow_walk_by:
        Headings follow `walk_by`; no new key, but reverses the documented independence
    default: own_key
    why: Opt-in, and keeps ]n navigation separate from index display
    answer: own_key
  vault_rollout:
    ask: How should the coder roll this out to `~/bob/sase_blog_zettel.md`?
    choices:
      edit_now:
        Edit its frontmatter now; rows show status, not section, until `just
        install-all`
      defer:
        Leave the vault alone; the summary lists the frontmatter to add after install
    default: edit_now
    why: You asked for this album; the edit only swaps row facets under the older bob
    answer: edit_now
  decision_record:
    ask: Add a decisions-web record for the accepted Records-heading rule?
    memory:
      - decisions:records-index-headings-are-opt-in
    default: false
    answer: false
proposed_by: bbugyi200.apollo.fk
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 10:24:04
status: wip
---

# Group the album Records index by an album-chosen choices field

## Goal

Bryan wants the `## Records` callout in `~/bob/sase_blog_zettel.md` grouped by each
zettel's `section` field, in outline order, instead of one big `🌱 Seed` group. Today
the `[!records]` index always groups by `status`, or by the first choices field when
there is no `status`. No album key changes that. `index_by: section` only adds raw-key
subheadings, sorted A→Z, inside status groups of more than 12, so it does not help.

Background and mockups:
`research:202611/album_index_group_by_section/album_index_group_by_section.md` (read it
with `sase artifact read`). This plan follows its option C. Decision `index_key` keeps
option B available.

Target output for the album today (24 zettels, all Seed, facets `[status, claim]`):

```markdown
> [!records] 24 zettels · 🌱 24 seed
>
> - **🪝 Intro** · 1
>   - [[sase_blog_zettel/what_came_before|What Came Before]] — 🌱 Seed · I went through
>     …
> - **🧱 What is SASE?** · 2
>   - [[sase_blog_zettel/sase_is_many_abstractions|sase is many abstractions]] — 🌱 Seed
>     · …
>   - [[sase_blog_zettel/why_a_tui|Why a TUI?]] — 🌱 Seed · …
> - **📜 Prompt language (macros)** · 4 …
> - **🔮 Future Work** · 6 …
```

## Contract

> [!decision] index_key = own_key

- **Key.** `index_group_by: <field>` sits at the top level of the album frontmatter,
  beside `walk_by`, `index_by`, and `index_facets`. It affects only the `[!records]`
  index.
- **Eligibility.** The field must be declared with kind `Choices` and `list: false`.
  This is the same test as `walk_by` (`walk::walk_field`). Invalid values produce a
  **warning** and fall back to today's heading field; they never damage the index:
  - `album_index_group_by_invalid`: the key is blank, multi-line, or not a string.
  - `album_index_group_field_unknown`: the key names an undeclared field.
  - `album_index_group_by_not_choices`: the field is not a single-value choices field.
- **Headings.** Choice labels appear in declared choice order, so the YAML order is the
  outline order. Empty choices are omitted. Members sort by `title_order`, so when
  `walk_by` names the same field, the index order is exactly the `]n` walk order.
  Missing, blank, malformed, list-shaped, or undeclared values go last under
  `⚠️ Needs <field>` (`needs_group_label`).
- **Nesting.** `index_by` still nests: a heading group with more than 12 members can
  render `index_by` subgroups, through the existing `finish_group`.
- **No-op equivalence.** If the effective `index_group_by` equals the auto-picked
  heading field, the index stays byte-identical to today. Today the auto field is
  `grouping::group_field`, so for example `index_group_by: status`. The non-status auto
  field keeps its `⚠️ Needs a status` wording. Albums without the key never change.
- **Status keeps its other jobs.** These do not change:
  - the callout title tally (`24 zettels · 🌱 24 seed`, `⚠️ k need a status`)
  - the report's `group_field`, `groups`, `needs_status`, and peeks
  - the owner `## Albums` catalog chip
  - the `group_before`/`group_after` fields in `bob albums update`
  - record `group` JSON in `bob records show` and `update`
- **Facets.** When `index_group_by` is in effect, the default facet rule excludes the
  heading field instead of the auto group field. `status` therefore becomes an eligible
  default facet. An explicit `index_facets` list is honored exactly as written, the same
  precedent as `index_by` subgroups.
- **JSON.** Each album entry in `bob albums list|check|sync -f json` gains an additive
  `index_group_by` key: the effective field key, or `null`. `index_facets` already
  reports the effective facets. Nothing else in the JSON changes, so there is no schema
  bump and no `!`.
- **Update protocol.** `field.remove` refuses a field named by the effective
  `index_group_by`. The refusal uses code `index_reference`, `key_path`
  `index_group_by`, the message
  `field "<f>" is named by index_group_by; remove the index reference in YAML first`,
  and `MIGRATION_HINT`. There is no new update op; Bryan edits the key in YAML.

> [!decision] index_key = follow_walk_by
>
> Do not add a key, a parse step, a JSON field, or an update guard. In steps 3 and 4
> below, pass `declaration.walk_by` as the heading selector. The `field.remove` guard
> for `walk_by` already exists. All the other rules above still apply: no-op
> equivalence, status keeping its jobs, facets, and `index_by` nesting.
>
> These parts of the contract reverse, so update them too:
>
> - `walk_by.set` now also regroups the index. Its preview and commit already include
>   the album edit.
> - Change the `bob albums update` help in `src/native/albums/cli.rs`, which says "the
>   index keeps its independent ordering". Then run `just docs-gen`.
> - Rewrite every "independent" sentence in `docs/records.md`, near the update section
>   (about lines 427–431), the record-walk intro (about 509–513), and the walk order
>   bullet (about 538–545). Also update the `bob albums list` row in
>   `docs/reference/json-outputs.md`.
> - Rework `walk_by_groups_the_report_independently_of_the_index` in
>   `tests/cli/albums.rs`: the index now follows `walk_by`, while `groups` JSON stays on
>   status.
> - The vault step needs only the `index_facets` swap.

## Implementation

1. **Parse (`src/native/albums/mod.rs`).**
   - Generalize `frontmatter_walk_by`/`FrontmatterWalkBy` into one decoder that takes
     the key name, for example `frontmatter_selector(frontmatter, key)` returning
     `FrontmatterSelector::{Absent, Valid, Invalid}` and the line. Use it for both
     `walk_by` and `index_group_by`. Update the "keep this repair local to `walk_by`"
     comment so it names both selectors. `index_by` keeps `top_level_string`.
   - Move the effective-selection block (undeclared, not choices, else `Some`) into a
     helper. The helper takes the key name and the finding codes, so `walk_by` and
     `index_group_by` share it. The `walk_by` codes and messages must stay
     byte-identical.
   - Add `index_group_by: Option<String>` (the effective key) to `AlbumDeclaration`,
     with a doc comment. Update the struct literals in the tests of `write.rs`,
     `tidy.rs`, `walk.rs`, and `mod.rs`.
2. **Group (`src/native/albums/grouping.rs`).**
   - Give `group_records` an `index_group_by: Option<&str>` parameter.
   - Add `index_headings: Option<IndexHeadings>` to `RecordGrouping`, where
     `IndexHeadings { field: String, groups: Vec<RecordGroup>, needs: Option<RecordGroup> }`.
   - Build it only when the key resolves to a non-list choices field that differs from
     `group_field(fields)`. Use `bucket_choice_members`, `finish_group(…, index_by)`,
     and `needs_group_label`. Otherwise use `None`.
   - Pass the heading key, instead of the group key, as the `facets(…)` exclusion
     whenever `index_headings` is `Some`.
   - Leave `groups`, `needs`, and `needs_status` exactly as they are.
3. **Render (`src/native/albums/index.rs`).** In `render_callout`, iterate
   `index_headings.groups` and then `index_headings.needs` when the value is `Some`.
   Otherwise iterate `groups` and `needs` as today. `render_group`, `row_facets`, and
   `tally_text` stay unchanged; the tally keeps reading `groups`.
4. **Thread the key.**
   - Pass `declaration.index_group_by.as_deref()` in `write.rs` (`plan_album_write`,
     which also serves record writes and `bob albums update`) and in `report.rs`.
   - Pass `None` in `create.rs` and in the `walk.rs` test call.
   - Add `index_group_by` to `AlbumReportEntry` and its `to_json`, next to `index_by`
     and `walk_by`.
5. **Update guard (`src/native/albums/update/`).**
   - Add `index_group_by: Option<String>` to `ops::IndexRefs`: the struct, the static
     `EMPTY_REFS`, the test literals, and `spans.rs` `refs_from`.
   - Fill it in `plan.rs` from `entry.declaration.index_group_by`.
   - Refuse it in `apply_field_remove`, next to the `index_by` check.
   - Update the `IndexRefs` line in the `update/mod.rs` module doc.
6. **Docs** (behavior change, so update the contract and reference pages):
   - `docs/records.md`, Albums section:
     - Add an `index_group_by` bullet after **Grouping**, stating the contract above.
     - Say in **Grouping** that this auto rule picks the tally and `groups`, and also
       the index headings unless `index_group_by` overrides them.
     - Say in **`index_facets`** that the default excludes the heading field.
     - Add `index_group_by` to the JSON key list, about line 271.
     - Name `index_group_by` in the `field.remove` refusal sentence, about line 427.
     - Change the record-walk sentence "The generated `[!records]` index keeps its
       independent group field and ordering." so it points to `index_group_by`. Show
       `walk_by: section` with `index_group_by: section` as the outline-album pairing.
   - `docs/reference/vault-format.md`, Album notes: name `index_group_by`.
   - `docs/reference/json-outputs.md`, `bob albums list` row: mention `index_group_by`.
   - Plugin docs are mounted from bob-plugins; leave them alone.
7. **Tests.**
   - `grouping.rs` unit tests:
     - Section headings follow declared order, omit empty choices, and keep members in
       title order.
     - A trailing `⚠️ Needs section` group collects missing and unknown values.
     - `groups` and `needs_status` stay on status.
     - The default facets include `status` and exclude the heading field.
     - An explicit `index_facets` list is honored exactly.
     - A heading group over 12 members with two `index_by` values still subgroups.
     - `index_group_by` equal to the auto field gives `index_headings == None`.
   - `index.rs` render tests:
     - The exact callout bytes for a section-grouped album, including a status-based
       title tally.
     - Byte identity with today's output when the key is absent, and when it names
       `status`.
   - `mod.rs` parse tests:
     - The valid key, plus its quoted and commented spellings.
     - Each warning code falls back to `None` and the album has no error findings.
   - `tests/cli/albums.rs`:
     - The list JSON carries `index_group_by`, or `null` when the key is absent or
       invalid.
     - Each invalid shape produces its warning code.
   - `tests/cli/albums_sync.rs`: extend or add next to
     `grouped_sync_orders_walk_lines_and_is_idempotent`. With `walk_by: section` and
     `index_group_by: section`:
     - the committed `[!records]` callout has section headings in declared order and the
       status tally;
     - the index order equals `walk.order`;
     - a second sync is a no-op.
   - `tests/cli/albums_update.rs`: `field.remove` of the `index_group_by` field refuses
     with `index_reference` and changes no bytes.
8. **Vault rollout.** Decision `vault_rollout`, below.

> [!decision] vault_rollout = edit_now
>
> Edit only the frontmatter of `~/bob/sase_blog_zettel.md` and keep every other byte:
>
> - Add `index_group_by: section` on the line after `walk_by: section`.
> - Change the `index_facets` items from `section, claim` to `status, claim`, keeping
>   the block-list style.
>
> Then dry-run the workspace build against the real vault:
>
> ```
> cargo run -q -- albums sync -d -f json
> ```
>
> Confirm three things:
>
> - The only index change is the `[!records]` callout of `sase_blog_zettel.md`.
> - Its headings run 🪝 Intro → … → 🔮 Future Work, in declared order.
> - Every other album's index is untouched.
>
> **Never** run a non-dry `albums sync` or a record write with the workspace build. The
> Mac plugin re-syncs albums with its installed bob whenever album frontmatter changes,
> so an older bob would revert the index anyway. Let the vault's git sync publish the
> edit. The final summary must tell Bryan two things:
>
> - The index regroups on the first album sync after `just install-all` on each host (or
>   after a manual `bob albums sync`).
> - Until then, the older bob shows `🌱 Seed` instead of the section on each row.

> [!decision] vault_rollout = defer
>
> Do not touch `~/bob`. The final summary gives the frontmatter to add after
> `just install-all`: `index_group_by: section` after `walk_by`, and `index_facets` set
> to `[status, claim]`. Under `follow_walk_by`, only the facet change.

> [!decision] decision_record
>
> Use `/sase_memory_write` to add the strand
> `sase/memory/decisions/records-index-headings-are-opt-in.md`, in the format of the
> existing decision strands, with `metadata.status: accepted` and the decided date.
>
> - **Applies to:** bob-cli albums and the `[!records]` index.
> - **Claim:** the accepted `index_key` rule. Status keeps the tally, the catalog chip,
>   and the `groups` JSON.
> - **Why:** list the rejected alternatives from the research report: follow `walk_by`,
>   or `index_by` with raw keys sorted A→Z under the 12-member gate, or the Base
>   `🧭 By section` view.
> - **Evidence:** the research ref and `docs/records.md`.
> - **Cost:** one more key to set on outline albums.
> - **Reopens when:** every walk-grouped album sets the same `index_group_by`.
>
> If `index_key = follow_walk_by`, retitle the strand to match. Then run
> `sase memory init`.

## Non-goals

- Listing empty sections as `· 0` (a later `index_show_empty`).
- An `index_group_by.set` update op.
- Plugin changes. The records fold parses any `**label** · N` heading and falls back to
  the label for fold identity. The catalog chip keeps reading status `groups`.
- Bob Mac Capture changes, and the `.base` view.
- Cargo.toml version or CHANGELOG edits: the release train owns those.

## Verification

- Run `just check` (fmt, clippy gate, and every test binary) and `just docs-check`.
- Run the dry run against the real vault from the `vault_rollout` step (also useful
  under `defer`). No index other than `sase_blog_zettel.md` may change.
- Commit subject: `feat(albums): group the records index by index_group_by`. Under
  `follow_walk_by`, use `feat(albums): head the records index by walk_by`.
