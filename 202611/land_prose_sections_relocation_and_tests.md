---
tier: tale
size: medium
title:
  Finish landing prose-section extract/inline — lossless link relocation, missing tests,
  closeout
goal:
  Make the round trip `inline(extract(S))` byte-identical for sections with in-note
  links, keep the link path form, add the extract and inline tests the epic plan
  required but never landed, then close epic bob-cli-qj.5.1.
proposed_by: bbugyi200.apollo.bob-cli-qj.5.1.land
proposed_with: claude/opus
bead: bob-cli-qj.5.1
status: done
---

- **PARENT:**
  [202611/prose_section_extract_inline.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/prose_section_extract_inline.md)
- **BEAD:**
  [bob-cli-qj.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qj/bob-cli-qj.5.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-qj.5.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-qj.5.1.land.md)
- **COMMITS:**
  - [3deb0bd](https://github.com/bobs-org/bob-cli/commit/3deb0bd3adab6014338682e56675733325ba1c24)
    — fix(prose): keep link form and shortest fragments across extract and inline

# Plan: Finish landing prose-section extract/inline

## Context

Epic `bob-cli-qj.5.1` (plan `plan:202611/prose_section_extract_inline.md`, the "epic
plan" below) shipped `bob prose extract` / `bob prose inline` (bob-cli commits
`85d2fb32`, `4f55514f`, `9f461d52`, `a558ce11`, `eff5d986`, `d856be45`) and the Bob
Prose gesture (bob-plugins `c2f73154`, `07c4eeb4`, both on `origin/master`). All eight
phases are closed. The land agent verified the epic and found the work below still
missing. Read the epic plan's **Conversion contract** section (Link relocation table,
Losslessness L1–L3) and its `extract` and `inline` phase **Tests** lists before you
start. They are the spec.

**Already done by the land agent** (committed before this tale starts; don't redo it):

- clippy: `filter_map_bool_then` and `useless_format` in
  `src/native/prose/convert/inline.rs`.
- `src/native/completion/kinds.rs`: added `FreeText` decisions for `prose extract`
  `expect` and `line`, and `prose inline` `expect`.
- `tests/fixtures/help/root-{short,long}.txt`: updated for the new `prose` root-help
  line.
- `just check` is green, `just docs-gen` leaves no diff, and `just docs-check` passes.
- Follow-up triage is recorded on the epic bead (`sase bead note`). It filed
  `bob-cli-rf` (install_ui flake) and `bob-cli-rg` (glossary memory) and declined the
  rest. Don't re-triage.

### Verified defects (reproduced with `-o -w` on a scratch git vault)

Area `owner.md` holds `## Intro` (with `[[#Detail]]`), then `## Topic` > `### Detail`
(with `[[#Detail]]`, `[[#Intro]]`, `^moved`), then `## Outside`. Third-party linkers sit
in `done/day.md` and `notes/other.md`. Extract writes are correct. Inline then writes:

1. **The shortest fragment is never chosen.** `[[owner_topic#Detail]]` comes back as
   `[[owner#Topic#Detail]]`, not `[[owner#Detail]]`. The contract says "the shortest
   fragment that resolves uniquely (`T#S` only when needed)".
2. **L1 breaks for in-note links.** The host is not byte-identical after the round trip.
   `[[#Detail]]` in the staying text and in the moved text both return as
   `[[#Topic#Detail]]`. The contract says links between two moved headings stay
   unchanged.
   - Root cause for both: `RelocationMap::canonical_fragment` in
     `src/native/prose/relocate.rs` tries `chain[start..]` for
     `start in 0..chain.len()`. That is the longest candidate first, so it returns
     `Topic#Detail` whenever that form resolves.
   - Fix: try the shortest suffix first (`(0..chain.len()).rev()`), keeping the "matches
     exactly `[index]`" check.
   - This affects extract too: a nested moved `#### Sub` under `### Detail` would be
     written `child#Detail#Sub` even when `Sub` is unique.
3. **The path form is not preserved.** From `done/day.md`, `[[areas/owner#Detail]]` (a
   vault-rooted wikilink) becomes `[[../areas/owner_topic#Detail]]`, and inline then
   keeps the `../` form. The contract preserves "stem versus path form".
   - Root cause: `render_note_path` renders every path-form target with
     `relative_path(current dir, …)`. But `lookup_target`, in the same file, treats a
     wikilink containing `/` as vault-rooted, unless it starts with `./` or `../`. Ref
     rename (`highlights_ref/rename/links.rs`) also writes vault paths.
   - Fix: a path-form **wikilink** that doesn't start with `.` renders as
     `slash_path(destination_without_extension)`.
   - A wikilink starting with `./` or `../` stays relative. Keep a leading `./` when the
     result doesn't start with `../`.
   - Markdown links keep today's relative rendering, which matches `lookup_target`'s
     Markdown rule and the existing
     `extract_relocates_host_and_vault_links_without_touching_protected_text` test.

### Missing coverage the epic plan required

- **Extract:** `src/native/prose/convert/extract.rs` (1543 lines) has **no tests**, and
  `tests/cli/prose.rs` has none for extract. The phase notes cite manual checks only.
- **Inline:** there are two CLI tests and four unit tests. Missing: `has-children`,
  `title-dropped`, trash plus rollback when verify fails, and recovery after an injected
  crash.
- **Proptest:** `extract_then_inline_preserves_eligible_sections` masks every wikilink
  with `strip_wikilinks` before comparing, so it never checks L1 for links. Its L3 check
  (`report.counts.layout == extracted.layout_count`) is a tautology.
- **Relocation:** `relocate.rs` has three tests. The epic plan asked for the `__cdx`
  link matrix in both directions.
- **Layout count:** a verbatim extraction of `### Detail` (body: one paragraph with a
  block id, then a bullet) reported `counts.layout: 2`, though the new note's body
  equals the promoted source body. Find out what the format pass counts there. If it
  counts lines extract itself synthesizes (the frontmatter seam or a trailing blank),
  exclude those so `counts.layout` reports only layout normalization of the moved text
  (L3). Otherwise record why the count is right on the epic bead.

## Governing decisions

- `decisions:prose-notes-are-owned-prose`: eligibility is derived. Bob never formats a
  host area or project note.
- `decisions:relation-label-rename-rewrites-history`: one previewed `vault_rewrite`
  transaction with one revertable commit.
- `decisions:reference-sources-are-preserved-write-once`: link scans never read or
  rewrite `.bob/`.
- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` version or
  `CHANGELOG.md` edits. These are bug fixes to an additive contract, so no `!`.

## Steps

1. **Fix relocation** in `src/native/prose/relocate.rs`, per root causes 1–3 above. Then
   update the unit tests that pin the old output in
   `src/native/prose/convert/inline.rs`:
   - `inline_restores_section_and_repairs_external_links` expects
     `Child prose [[#Topic#Idea]]` and `[[../area#Topic#Idea|Read]]`. They become
     `[[#Idea]]` and `[[area#Idea|Read]]`, because `notes/draft#Idea` was a vault-rooted
     path.
2. **Relocation matrix tests** (`relocate.rs` tests module). Read the research appendix
   with `sase artifact read` (the epic plan names
   `research:202611/prose_rename_and_section_extraction/…`, `__cdx`). Cover every row of
   the contract's Link relocation table in both directions:
   - Source-local links and moved-to-staying links.
   - Links within the move (unchanged both ways).
   - Third-party links in stem, vault-path, `./`/`../`, embed, and percent-encoded
     Markdown forms, from root, `done/`, and a daily-note subfolder.
   - Aliases, nested and duplicate headings (`T#S` only when needed), and links to the
     kept heading.
   - Protected regions left alone.
   - `inverse(forward(x))` resolving to the original targets, with spelling identical
     whenever the original was already the shortest form.
3. **Strengthen the proptest** in `convert/inline.rs`:
   - Drop `strip_wikilinks`.
   - Seed the host with in-note links from staying text to a moved heading and from
     moved text to a staying heading, in shortest form.
   - Seed third-party linkers in a subfolder using stem, vault-path, and Markdown forms.
   - Assert L1 byte for byte, apart from what L1 allows (owned number tokens in a prose
     host, seam blank lines).
   - Assert L2 for every linker by resolving each link before and after to the same
     (note, heading or block).
   - Add a layout-dirty variant asserting `counts.layout > 0` in the extract preview,
     and assert `0` for the layout-clean shapes.
   - Keep both stub forms.
4. **Extract tests** (`tests/cli/prose.rs`, scratch git vaults, as the existing inline
   CLI tests do):
   - Every extract refusal code through the CLI: `not-host-note`, `no-section`,
     `ambiguous-section`, `structural-section`, `holds-tasks`, `holds-anchor`,
     `conflict-markers`, `stub-section` with its `next`, `unclosed-fence`,
     `setext-multiline`, `owned-number-heading`, `definition-boundary`, `name-taken`,
     `bad-name`, and `ambiguous-link`. Check exit code 2 and the `refusal` object shape.
   - Preview versus write, the `-f json` ProseConversion v1 shape (`schema_version`,
     `contract`, `status`, `changes[].action`/`sha256`, `links`, `counts`, `focus`,
     `notice`, `commit`, `sync`), and human output.
   - `-e` with a stale fingerprint gives `plan-changed` and writes nothing.
   - A `name-taken` race: the target file appears between the preview and `-w -e`.
   - A drifted linker aborts the whole write (W4). Nothing is written, and HEAD is
     unchanged.
   - `-o` makes one commit with the W5 subject, and `.bob/` and `.trash/` are never read
     or touched.
   - A prose host with `section_numbers: true` renumbers the host, and the new note has
     numbering off.
   - The provenance frontmatter keys come out in X2 order.
   - The no-section `next` for a prose note that has a parent (the toggle dispatch
     case).
5. **Inline tests:**
   - `has-children` (lists up to five) and the `title-dropped` warning finding.
   - Verify failure rolls back and restores the trashed note.
   - Recovery after an injected crash. `vault_rewrite`'s fault point is `#[cfg(test)]`,
     so drive it from a lib unit test through inline's real write path (the `run_inline`
     → `vault_rewrite::run` flow). If it truly can't be reached, record why on the epic
     bead.
6. **Gates:** `just check` (don't run `just check-full`), `just docs-gen` with no diff,
   and `just docs-check`. If behavior a doc describes changes (shortest fragment, path
   form), update `docs/prose.md`'s Link relocation section to match the contract
   wording.
7. **Commit** with
   `fix(prose): keep link form and shortest fragments across extract and inline`. Use
   separate `test(prose): …` commits if that splits more cleanly.
8. **Close the epic.** Do this in the same turn as the final code:
   1. Run `sase bead epic-symbols bob-cli-qj.5.1`. For each listed `--epic-symbol`
      entry, resolve it (wire it up, privatize it, add a non-test pragma, or delete it).
      Re-key a Justfile line only to a still-open bead that needs it. At landing time
      the command reported none.
   2. Run `sase bead close bob-cli-qj.5.1 --note "<verification>"`. The note covers the
      land agent's checks plus this tale's: what you fixed, the test counts, and the
      gate results. Never use `--force` to make the close succeed. If the close is
      rejected for leftover epic symbols, clean them up and close again.
   3. Run `just symvision` and confirm the whitelist is clean.
   4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN
      path that `sase bead read bob-cli-qj.5.1 -r "<why>"` prints
      (`plan:202611/prose_section_extract_inline.md`, under the plans repo).
   5. The epic's `parent_bead` is phase `bob-cli-qj.5` ("prose-sections: author and
      propose the epic"). It is already closed (done) because its only job was authoring
      this epic, which it did. Run `sase bead read bob-cli-qj.5 -r "<why>"`. If it is
      still closed, leave it. If it was reopened, close it with
      `sase bead close bob-cli-qj.5 --note "child epic bob-cli-qj.5.1 landed: <summary>"`.
      Never close the containing epic `bob-cli-qj`; its own land agent does that.

## Out of scope

- The follow-ups the land agent triaged (`bob-cli-rf`, `bob-cli-rg`, the q5 snooze
  vector), the Prose Card provenance crumb (`bob-cli-qw`), and the report's Phase 3
  (reversible projects).
- Any bob-plugins change. The plugin decodes only bob's JSON, and these fixes don't
  change the contract shape.
