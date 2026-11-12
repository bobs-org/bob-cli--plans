---
tier: tale
size: small
title: Album counts say "1 restaurant", then close the album-notes epic bob-cli-k3
goal: An album with exactly one record reads "1 restaurant" (not "1 restaurants")
  in its generated `[!records]` tally and on its owner `[!albums]` card, and "⚠️ 1
  needs a status" (not "need"). bob reports the display singular additively in album
  JSON. The retired config-records parser stops compiling into production code. Then
  the epic bob-cli-k3 is closed and its plan is marked done.
proposed_by: bbugyi200.athena.bob-cli-k3.land
proposed_with: claude/opus
bead: bob-cli-k3
status: done
---

- **PARENT:**
  [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)
- **BEAD:**
  [bob-cli-k3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/README.md)

# Album counts say "1 restaurant", then close the album-notes epic

This is the remaining landing work for epic **bob-cli-k3** ("Album notes — every record
collection hangs under the area it serves"). The epic's lander verified all 11 phases
and triaged every `PROPOSED FOLLOW-UP`. The outcomes are recorded as a `LANDING TRIAGE`
note on bob-cli-k3, so do not re-triage. One defect the epic caused is still open, and
this tale fixes it, along with a small leftover from the epic's config-retirement phase.
Then it closes the epic.

## Why

- **Singular tally bug** (bob-cli-k3 note #3, introduced by phase k3.5's generated
  index). `tally_text` in `src/native/albums/index.rs` always writes the plural, so any
  album with exactly one record renders `> [!records] 1 restaurants` (or `1 zettels`,
  `1 TV shows`). The needs-status fragment always says `need`, so a single offender
  reads `⚠️ 1 need a status`. Choice-group fragments (`🔖 1 to try`, `💚 1 liked`) are
  choice labels, not nouns. They are correct and must not change.
- **The same bug on the owner card.** Phase k3.7 added `albumRowModel` in bob-plugins
  `plugins/bob-ledger-tools/src/241o-plugin-album-catalog-model.js`. It renders
  `` `${album.records} ${album.plural || "records"}` ``, so the owner `[!albums]` card
  also shows `1 restaurants`.
- **Retired reader still compiled.** Phase k3.11 removed the `records:` config reader
  from every production path. `parse_records_config` in `src/native/records/schema.rs`
  is still non-test code, though, and it is now called only from tests. It stays silent
  because `src/native/records/mod.rs` carries a module-wide `#![allow(dead_code)]`.

Already done, so leave it alone: the stale `bob capture --help` sentence (k3 note #4)
was fixed by k3.11 in `5e82fc8e`. The album header's record-count phrase
(`albumHeaderRecordCountPhrase` in `241w-plugin-album-schema-model.js`) belongs to the
**active** epic bob-cli-kr (kr.3 / kr.4.1 Schema Card). Do **not** edit `241w`. A
`DISCOVERED ISSUE` note on bob-cli-kr already tells kr to adopt the new `singular`
field.

## Design

### One display-singular rule, owned by bob

- In `src/native/records/schema.rs`, split the display rule:
  - Add `pub(crate) fn album_singular(label: &str) -> String`. It contains today's
    lowering logic from `album_plural`: keep the label as-is when its first two bytes
    are ASCII uppercase (acronym, e.g. `TV show`), otherwise lowercase the first
    character.
  - Re-express `album_plural(label)` as `format!("{}s", album_singular(label))`, so the
    two can never drift. Existing plural tests must still pass unchanged.
- In `src/native/albums/report.rs`, add
  `pub(crate) fn resolved_singular(declaration: &AlbumDeclaration) -> String` next to
  `resolved_plural`:
  - `declaration.label` through `album_singular`;
  - else `noun_claim` through `album_label`, then `album_singular` (the label default);
  - else `"record"`, mirroring `resolved_plural`'s `"records"` fallback.

  An explicit irregular `plural` (for example `people`) does not change the singular,
  which still comes from the label (`Person` → `person`).

### The `[!records]` tally (bob-cli)

- Thread the singular into the index renderer so callers cannot pass one without the
  other. Replace the `plural: &str` parameter of `render_callout`, `plan_index`, and
  `tally_text` with a small borrowed pair: a struct such as
  `RecordNouns<'a> { singular: &'a str, plural: &'a str }`, or two parameters.
  - Update every production caller: `albums/create.rs` (~line 607), `albums/report.rs`
    (~line 355), and `albums/write.rs` (~line 118).
  - Callers build the pair from `resolved_singular` and `resolved_plural`.
- `tally_text` rules:
  - `0` keeps `No <plural> yet`.
  - `1` writes `1 <singular>`.
  - `N > 1` writes `N <plural>`.
  - The needs-status fragment writes `⚠️ 1 needs a status` when the count is 1, and
    `⚠️ k need a status` otherwise.
  - Group fragments are unchanged.
  - The singular passes through the same `inert_text` escaping as the plural.
- The index stays a pure function of the declaration and records. The only bytes that
  change are tallies with a total of 1 or a needs-status count of 1. The live vault's
  albums (104 restaurants, 15 podcasts, 10 movies, 23 shows, no needs-status) are not
  affected. Any one-record album elsewhere becomes `stale` once, and `bob albums sync`
  or the nightly step repairs it.

### Additive JSON field (bob-cli)

- `AlbumReportEntry` (`src/native/albums/report.rs`) gains `singular: Option<String>`.
  It serializes as `"singular"` right after `"plural"` in `to_json`, and is set from
  `resolved_singular` whenever `plural` is set (the same `Option` shape: `null` when the
  schema cannot be parsed and no label or noun is recoverable; match whatever `plural`
  does today).
- This is additive: `schema_version` stays 1 and nothing is renamed or removed. It must
  **not** feed `schema_fingerprint`; the label already does.
- Check any typed or JSON-shape test that pins the album entry's key list, such as
  `tests/cli/albums.rs` JSON assertions, and extend it.

### Owner `[!albums]` card (bob-plugins, bob-ledger-tools)

- In `plugins/bob-ledger-tools/src/241o-plugin-album-catalog-model.js` `albumRowModel`,
  the count reads `${records} ${noun}`, where noun is:
  - `album.singular` when `records === 1` and `album.singular` is a non-empty string;
  - otherwise `album.plural || "records"`.
- The fallback keeps today's text when the installed bob predates `singular`, because
  the vault plugin calls whatever `bob` is installed and bob-cli ships on the daily
  release train.
- If `241l-plugin-album-data.js` normalizes report entries through an allowlist that
  would drop unknown keys, carry `singular` through it the same way it carries `plural`.
  Check the `albumNullableString(entry.plural)` validation near line 107: an optional,
  nullable `singular` must not make an older report invalid.
- Rebuild `main.js` with the repo's normal build. Then run `bob plugins sync` **without
  `--force`**, which the project instructions require after a plugin change. If sync
  refuses because of another workspace's uncommitted deploy, record that in your close
  note rather than forcing.

### `parse_records_config` becomes test-only (bob-cli)

- Mark `parse_records_config` `#[cfg(test)]`. Do the same for any helper reachable
  **only** from it, but only if gating it is mechanical. Do not hunt for other dead
  code, and do not remove `#![allow(dead_code)]` from `records/mod.rs`.
- Callers that stay legal are in test modules only: `records/schema.rs` tests,
  `records/plan.rs` tests, `capture_record.rs` tests, and
  `capture_language/tests/record.rs`. Any non-test caller is a surprise: stop and
  re-check rather than gating it.
- No runtime behavior changes.

## Steps

1. **bob-cli code.** Make the `album_singular` / `album_plural` split, add
   `resolved_singular`, change the index renderer signature and `tally_text` rules, add
   the `singular` JSON field, and apply the `#[cfg(test)]` gate.
2. **bob-cli tests.**
   - Unit tests in `src/native/albums/index.rs`:
     - `tv_shaped_index_uses_acronym_plural` (total 1) now expects
       `> [!records] 1 TV show · 1 to watch`. Rename it to say the acronym singular and
       plural are both kept, and add a 2-record case that still asserts `2 TV shows`.
     - The needs-status cases (`⚠️ 1 need a status` around lines 587 and 706) become
       `⚠️ 1 needs a status`. Keep or add one case with a count of 2 that still says
       `need`.
     - Add a direct `tally_text` table test covering 0, 1, and 2 records, needs-status
       counts of 1 and 2, an acronym label, and an irregular explicit plural.
   - Unit tests in `src/native/records/schema.rs` / `albums/mod.rs`: `album_singular`
     for `Restaurant` → `restaurant` and `TV show` → `TV show`; existing plural
     expectations unchanged.
   - CLI tests, updated from `1 restaurants` to `1 restaurant`:
     `tests/cli/albums_sync.rs` (~lines 123 and 214), `tests/cli/albums_update.rs`
     (~line 852), and `tests/cli/albums_create.rs` (~line 1123).
   - `tests/cli/albums.rs` (~line 1253) has `⚠️ 1 need a status`; update it to `needs`.
     Assert `"singular"` in one albums list JSON test.
   - Run `rg -n "1 restaurants|1 TV shows|1 need a status"` over `src/`, `tests/`, and
     `docs/` to catch stragglers.
3. **bob-cli docs** (keep the additions short; `docs/reference/json-outputs.md` is near
   its word cap, so do not grow it):
   - `docs/records.md` ~line 64: note that the display singular is the same lowered
     label without the `s`.
   - `docs/records.md` ~line 83: the tally is `N <plural>` (`1 <singular>`), then
     nonzero group counts, then `⚠️ k need a status` (`needs` when k is 1).
   - `docs/records.md` album JSON paragraph (near "List and check share one JSON
     envelope"): add one sentence saying each album carries `noun`, `label`, `plural`,
     and the additive display `singular`.
   - Run `just docs-gen` if the CLI reference or generated pages change, then
     `just docs-check`. Two failures were triaged before this tale: the strict Zensical
     anchors on bob-cli-kk and mount staleness on bob-cli-it. If either reproduces,
     record it as unchanged. Any new failure is yours to fix.
4. **bob-plugins.** Open the linked repo with `/sase_repo` (`bob-plugins`). Make the
   `241o` change (plus `241l` passthrough if needed) and extend
   `scripts/test-ledger-tools-album-catalog.cjs`:
   - `records: 1` with `singular: 'restaurant'` gives `1 restaurant`;
   - `records: 1` without `singular` gives `1 restaurants`, the old-bob fallback;
   - `records: 4` still gives `4 restaurants`;
   - `records: 0` still gives `0 restaurants`.

   Then build, run that test script and the repo's unit suite (`npm run test:unit`, or
   the documented equivalent), and `npm run validate` if present. Known clean-base
   flakes: bob-cli-ki, bob-cli-ju, bob-cli-km, bob-cli-io, bob-cli-ks, and bob-cli-kw. A
   failure matching one of these is not yours, but name it in your note. Deploy with
   `bob plugins sync` (no `--force`).

5. **Gate.** Run `just check` in bob-cli. Do **not** run `just check-full`.
6. **Close the epic bob-cli-k3.** This is the final step and must not wait for this
   work's commit SHA, push, or CI:
   1. Run `sase bead epic-symbols bob-cli-k3`. For each listed `--epic-symbol` entry,
      either resolve the symbol (wire it up, privatize it, add a non-test pragma, or
      delete it per the Symvision epic-whitelist policy) or, only when a still-open
      later bead needs the exemption, re-key that Justfile line to the open bead. At
      landing time the list was empty.
   2. Close the epic:

      ```
      sase bead close bob-cli-k3 --note "<verification>"
      ```

      The note must say:
      - all 11 phases were verified closed and their work confirmed in source and vault:
        152 records in 4 albums, zero `bob records check` / `bob albums check` errors,
        and `source=album` everywhere;
      - integration since the epic started was clean: album writers already use the
        literal-pathspec `ob::commit_paths` and the configured-timezone `created` stamp,
        and kr's `albums update` rides the same `plan_index`;
      - epic note #4 was fixed by `5e82fc8e`; note #3 is fixed here (name the bob-cli
        and bob-plugins changes and the test results);
      - `parse_records_config` is now test-only;
      - the follow-up triage outcomes in the LANDING TRIAGE note: new bob-cli-kz,
        bob-cli-l0, and bob-cli-l1; declined items with reasons; and the kr header note;
      - the vault task `bob.md ^record-h2-sections` is reported for Bryan to close, not
        closed, and `^sub-project-h2-sections` can now reuse `host_catalog`;
      - the plugin sync outcome.

      Never use `--force` merely to make the close succeed. If the close is rejected for
      leftover `--epic-symbol` entries, finish that cleanup and close again.

   3. Run `just symvision` if the recipe exists. At landing time it did not; say so in
      the note if it is still absent.
   4. Set `status: done` in the frontmatter of the epic's plan file,
      `202611/album_and_record_notes.md` in the plans sidecar (the PLAN path printed by
      `sase bead read bob-cli-k3`). It currently reads `status: wip`.
   5. bob-cli-k3 has no `parent_bead`, so nothing above it needs closing.

## Out of scope

- `241w` / the album header phrase (bob-cli-kr owns it).
- Glossary strand aliases. Any change to group-fragment wording.
- Removing `#![allow(dead_code)]`.
- Editing `Cargo.toml` version or `CHANGELOG.md` (release train).
- Running `bob albums sync` against the live vault, unless `bob albums check` on the
  live vault reports a stale index after you install the new bob. Normally the nightly
  step handles that.

## Acceptance

- A one-record album renders `> [!records] 1 <singular>`. Its owner card says
  `1 <singular>` once bob reports `singular`.
- A single needs-status record reads `⚠️ 1 needs a status`.
- Multi-record tallies are byte-identical to before.
- `bob albums list -f json` includes `singular` for every parsable album, and
  `schema_fingerprint` values are unchanged.
- `parse_records_config` is `#[cfg(test)]`.
- `just check` passes. bob-plugins unit tests for the album catalog pass.
- bob-cli-k3 is closed with the verification note, and its plan file says
  `status: done`.
