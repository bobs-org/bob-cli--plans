---
tier: tale
size: small
title:
  "Finish bob-cli-mb: document and test living-first [[ suggestions, then close the epic"
goal:
  bob-cli-mb.2 shipped the living/archive/hidden partition for vault-wide `[[`
  suggestions but skipped the help sentence, the docs/capture.md rule, docs-gen, and
  most of the required tests. Add those, then close epic bob-cli-mb.
proposed_by: bbugyi200.athena.bob-cli-mb.land
proposed_with: claude/opus
bead: bob-cli-mb
status: done
---

- **PARENT:**
  [202611/note_goto_followups.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/note_goto_followups.md)
- **BEAD:**
  [bob-cli-mb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mb/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-mb.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-mb.land.md)
- **COMMITS:**
  - [157934c](https://github.com/bobs-org/bob-cli--plans/commit/157934c2c2faa6c31ad5e0ac53b7fafb9518044a)
    — docs(plans): mark 202611/note_goto_followups as done

# Finish bob-cli-mb: living-first `[[` help, docs, and tests, then close the epic

## Why

Epic `bob-cli-mb` (plan `plan:202611/note_goto_followups.md`) has all five phases
closed. The bob-cli-mb land agent verified the work:

- `bob-cli-mb.1` (430162f5): shared `path_tier` predicate, `capture-complete -A` archive
  catalog, help, docs, and tests. Complete.
- `bob-cli-mb.3` (bob-mac-capture 61d29a6f) and `bob-cli-mb.4` (bob-mac-capture
  0f430144): Recent group and Command-X Archive scope. Both landed with green exact-SHA
  macOS CI. Complete.
- `bob-cli-mb.5` (bob-plugins c68ff50): line-less `bob-open-notes` opens record a Vim
  jump. Complete and deployed.
- `bob-cli-mb.2` (b73cf56f, "feat(capture): prioritize living vault link suggestions"):
  incomplete. It touched only `src/native/capture_links.rs`. The code is correct: a
  stable Living, Archive, Hidden partition before `RESULT_LIMIT` truncation in
  `note_candidates`, the `Vault` branches of `heading_candidates` and
  `block_candidates`, and `NoteIndex::matching_note_inserts`. That follows the epic's
  final decision, `link_tiers = demote`. It is missing three things the epic plan
  required:
  1. the help sentence in the `capture-complete` long help;
  2. the matching rule in `docs/capture.md`, plus `just docs-gen`;
  3. most of the planned tests. Only one partition test,
     `vault_wide_link_suggestions_partition_living_archive_and_hidden`, was added.

No commits since the epic started conflict with it or duplicate it. The only non-epic
bob-cli commit is a glossary docs edit. The follow-up triage is already recorded on
`bob-cli-mb`.

## Ground rules

- Work only in this project's bob-cli checkout. Do not touch bob-mac-capture or
  bob-plugins.
- Before editing help or docs, read the reference memory:
  `sase memory read docs.md cli_rules.md -r "<why>"`.
- Do not edit `docs/reference/capture-syntax.md` (it is at its word cap), the
  `Cargo.toml` version, `CHANGELOG.md`, or any memory file.
- Do not change ranking behavior. The partition code is correct. This tale adds only
  documentation and tests.
- Nothing here breaks a contract, so no commit subject carries `!`.

## Step 1: help sentence

In `src/native/capture_complete/cli.rs`, find the long-help paragraph that begins "When
the cursor is inside an Obsidian wikilink, wikilink completion takes precedence over
capture-marker completion." Directly after the sentence that ends with "`[[##Head` and
`[[^^block`.", add this sentence:

> Vault-wide suggestions (`[[note`, `[[##Head`, `[[^^block`, and record link values)
> list living notes first; archived and hidden notes follow.

Keep the existing `\` line-continuation style of the surrounding string. The wording
"list living notes first; archived and hidden notes follow" is fixed by the epic
decision.

## Step 2: docs rule

In `docs/capture.md`, find the paragraph that begins "When the cursor is inside a valid
Obsidian wikilink component, link completion takes precedence over marker completion".
After the sentence about `wikilink_block` scopes (it ends "…while note and heading
completion keeps working."), add a short rule, about 40 words. It should say:

- Vault-wide suggestions list living notes first; archived and hidden notes follow. This
  covers `[[note`, `[[##…`, `[[^^…`, and record link-value suggestions.
- Tiers come from the same path rules as `//`. Link to the tier table with
  `[Going to a note with '//'](#going-to-a-note-with)`.
- Target-scoped completion (`[[note#`, `[[note#^`, `[[#`, `[[#^`) and link resolution
  are unchanged, so an archived note stays linkable.

## Step 3: generated docs

Run `just docs-gen`. It refreshes `docs/reference/cli/capture-complete.md` and any help
golden. Commit what it changes. Then run `just docs-check`.

## Step 4: tests

Add focused tests to the `mod tests` block of `src/native/capture_links.rs`. Reuse its
`TempDir` and `write_file` helpers and the style of the existing
`vault_wide_link_suggestions_partition_living_archive_and_hidden`. Each test builds a
`TempDir` vault and calls `NoteIndex::read`. Cover these cases:

1. **Library sidecar.** `papers/bob-paper.md` beside `papers/bob-paper.pdf` (Hidden),
   plus `lib/bob-lib.md` (Hidden; the default Highlights `lib` root, as the
   `note_catalog` tier test relies on), plus living `bob-living.md`. `[[bob` lists
   `bob-living.md` before both hidden notes.
2. **Date query.** Living daily `2025/20251111.md` and satellites
   `2025/20251110_{habit,done,poms,events}.md` (Archive). `[[2025111` returns
   `2025/20251111.md` first, then the satellites. Without tiers, path order could put
   the satellites first.
3. **Truncation.** `done/bob.md` (an exact-stem match, so the best score) plus 25 living
   notes `notes/bob-NN.md`. `note_candidates` for `[[bob` returns exactly `RESULT_LIMIT`
   rows, all living, and `done/bob.md` is absent. Also check `[[##` and `[[^^`
   truncation with the same shape. Give every note a matching heading and a matching
   block ID, so the 20 visible vault-scope rows are all living.
4. **Empty query.** Living `zeta.md`, archive `done/alpha.md`, and hidden
   `_meta/beta.md`. `[[` with an empty query lists `zeta.md` first, then
   `done/alpha.md`, then `_meta/beta.md`, even though path order alone would differ.
5. **Resolution and target scope unchanged.** In the same vault as case 4, plus
   `done/a_done.md` with a heading:
   - `index.resolve_target("done/a_done")` returns that note. The tests module can call
     the private method.
   - Stem-only resolution `resolve_target("a_done")` also returns it.
   - `[[done/a_done#` completes its headings.
6. **Record link values.** `matching_note_inserts` keeps the Living, Archive, Hidden
   order when the archive twin sorts first by path. For example, use `done/cash.md` and
   living `money/cash.md`: `matching_note_inserts("cash")` lists `money/cash` first.
   Also add a test at the `records.rs` suggestion path, or a CLI test, showing that the
   stem dedup there now keeps the living note. Add it only if a fixture is cheap; the
   unit test is required.

Update no existing golden unless the new order is the intended cause.

## Step 5: gate

- Run `just check` (fmt, clippy, and every test binary) and `just docs-check`. Both must
  pass.
- Do not run `just check-full`.

Suggested commit subject:
`docs(capture): document living-first [[ suggestions and cover the tier partition`.

## Step 6: close out epic bob-cli-mb (final step)

Do this after Step 5 passes, in the same turn. Do not wait for this tale's own commit
SHA, push, or CI.

1. Run `sase bead epic-symbols bob-cli-mb`. The land agent found none. If any entry is
   listed now, resolve it per the Symvision epic-whitelist policy: wire it up, privatize
   it, add a non-test pragma, or delete it. Re-key it only when a still-open later bead
   needs the exemption.
2. Close the epic. Fill in the bracketed part with what you actually did:

   ```bash
   sase bead close bob-cli-mb --note "Verified all five phases against plan:202611/note_goto_followups.md and the link_tiers=demote decision: mb.1 tiers + capture-complete -A (430162f5); mb.2 living-first [[ partition (b73cf56f), completed by the landing tale with the help sentence, docs/capture.md rule, docs-gen, and the missing partition/truncation/date/empty-query/resolution/record-value tests [summarize]; mb.3 Recent group (bob-mac-capture 61d29a6f, CI run 43899389310 green); mb.4 Command-X Archive scope (bob-mac-capture 0f430144, exact-SHA CI green); mb.5 <C-o> jump on line-less opens (bob-plugins c68ff50, deployed). No post-start commits conflicted or duplicated the feature. Follow-ups: flake corroborated on bob-cli-ko; vague bob-plugins flake mention declined. just check and just docs-check passed; no epic-symbol entries."
   ```

   Never use `--force`. If the close is rejected, fix the cause and close again.

3. Run `just symvision` if the recipe exists, and confirm the whitelist is clean.
4. Run `sase bead read bob-cli-mb -r "Need the PLAN path to mark the epic plan done"`.
   In the plan file at the PLAN path it prints, change the frontmatter `status: wip` to
   `status: done`. Change nothing else in that file.
5. Epic `bob-cli-mb` has no `parent_bead`, so there is no ancestor to close.
