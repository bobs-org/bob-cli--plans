---
tier: tale
size: medium
title: Land annotation sections — comment declarations, linked page headings, and
  epic closeout
goal: 'A highlight, underline, or image comment whose first line is `# Name` declares
  a section that its own block joins, so Bryan''s real gesture groups instead of failing
  silently; master CI is green again because linked page headings always split pages;
  and epic bob-cli-bd is verified, closed, and its plan marked done.

  '
proposed_by: bbugyi200.apollo.bob-cli-bd.land
proposed_with: claude/opus
bead: bob-cli-bd
status: done
---

- **PARENT:**
  [202610/ref_note_annotation_sections.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_note_annotation_sections.md)
- **BEAD:**
  [bob-cli-bd](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bd/README.md)

# Land annotation sections: comment declarations, the CI fix, and closeout

This tale finishes the landing of epic `bob-cli-bd` (annotation sections for ref notes).
Read the epic with `sase bead read bob-cli-bd -r "<why>"`. Its plan file (the `PLAN`
path that command prints) holds the full design contract. This tale changes only what is
described here. The landing triage note on `bob-cli-bd` records every follow-up outcome;
nothing else needs triage.

There are three parts, in this order:

1. **CI fix.** Master CI is red because of phase `bd.1`.
2. **Bryan's note #2 on the epic.** The section gesture he actually used fails silently.
3. **Epic closeout.**

Do not edit the `Cargo.toml` version or `CHANGELOG.md` (release train). Every change
here is additive, so no commit needs `!`.

## 1. Linked page headings always split pages (CI fix)

**Failing tests:**
`tests/cli/highlights/continuations.rs::continuation_scan_reports_dangling_marker` and
`::continuation_scan_tucks_two_pages_and_is_idempotent`.

**Cause:** commit `6ec2f16a` made `parse_sidecar_markdown`
(`src/native/highlights_ref/sidecar.rs`) accept page headings only at the level of the
first page-shaped heading. These fixtures start with a simple `## Page 1` marker page
and then use linked `#### [Page 11](highlights://…)` headings, so every highlight now
lands under Page 1.

**Fix:** a linked page heading is a page break at any level. In the `next_page_heading`
match guard, change `if heading.level == *expected` to
`if heading.linked_page_style || heading.level == *expected`, and update the comment
above `page_heading_level` to say so.

**Test:** add `linked_page_headings_split_after_a_simple_marker_page` to
`src/native/highlights_ref/tests/sidecar.rs`, next to
`simple_sidecar_level_one_stickies_are_not_page_breaks`. The fixture is:

- `## Page 1` with a `- status: wip` line;
- `#### [Page 11](highlights://example#page=11)` with a quote;
- a `# P values` sticky;
- `#### [Page 12](…)` with a quote.

Assert the page labels are `[Page 1, Page 11, Page 11, Page 12]` and that the sticky is
a `StandaloneNote` containing its text.

If this exact change is already on the branch, keep it and move on. Verify with
`cargo test --test cli highlights::continuations` (3/3).

## 2. A comment whose first line is `# Name` declares a section

### What went wrong

Bryan tried the feature on `ref/blogs/the_ai_native_sdlc_playbook.md` (epic note #2):

- He highlighted the PDF's own heading "What is an AI-native SDLC?" and commented
  `# what is it`.
- He tagged two other highlights `@what-is-it`.

Highlights exported the line verbatim. The sidecar has `# what is it` as that
highlight's comment, after a blank line. But the epic contract lets only sticky notes
declare a section, so:

- nothing grouped;
- the note shows a literal `\# what is it` in Obsidian and in the Mac Refs panel;
- `plan_sections` (`src/native/highlights_ref/sections.rs`) returns before its
  `section_declaration_in_comment` check whenever no sticky declares a section, so no
  diagnostic was printed either.

Bryan's vault has exactly one real sticky note; everything he writes is a highlight
comment. The research deferred "highlight-anchored declarations" mainly because of the
`bob-cli-ba` hard-wrap false positives. Phase `bd.1` has since fixed those (wrapped `# `
quote lines stay quote text). Highlight-anchored declarations are now in scope.

### Behavior contract

- **Who may declare:** the comment of a displayed highlight, an underline (absorbed or
  standalone), or an image, as well as a sticky note.
- **The declaration line** is the first non-empty line of that annotation's comment
  source, after an optional `Comment:`/`Note:` label. It must be a line the sidecar
  author typed as `# Name`: exactly one `#`, whitespace, then text, using the same
  `scan_section_declaration` grammar (aliases, title sanitizing, keys, slug, tag,
  reserved and empty-title rejections).
- **Escapes:** a user-typed `\# Name` is the escape and never declares. Today the parser
  stores both forms as `\# Name` (`escape_markdown_heading` in `sidecar.rs`). So record
  the declaration explicitly at parse time instead of re-detecting it from the escaped
  text, e.g. a new `SidecarAnnotation.comment_section_heading: Option<String>` set in
  `parse_sidecar_chunk`'s highlight-comment and image-comment paths.
- **Not declarations:** a `# ` line anywhere other than the first comment line, and
  quoted PDF text.
- **Continuation followers** (`<`) are not declaration sites; at most the head of a
  continuation group declares.
- **Accepted declaration:**
  - The section is declared exactly like a sticky declaration.
  - The declaring block's **home is that section**. A declaration beats tags.
  - Within the block, same-section tags are consumed, and tags for other sections stay
    visible with `section_conflict` (kept: the declared section).
  - The declaration line, and one blank line after it, is removed from that member's
    display source through `SectionPlan.rewritten`. Any remaining comment text stays the
    annotation's own comment; an emptied comment means no Comment card.
- **The card:** a comment declaration contributes **no description**; descriptions come
  only from sticky declarations. The card is the usual `[!summary|section]` card, and
  its count includes the declaring block.
- **Several declarations in one block:** the first in member order (head, then absorbed
  underlines) is the home. A later one still declares its section and is consumed, and
  reports `section_conflict`.
- **Merging and order:** declarations that share a key merge into the first one in
  sidecar order, whether that is a sticky or a comment, with `section_duplicate` as
  today. Sections render in the sidecar order of their first declaration of either kind.
- **Rejected comment declarations** (empty or reserved title) render the literal escaped
  line as today and report `section_title_empty` / `section_reserved_title`.
- **Identity:**
  - The declaring highlight keeps its `^h-` ID, because its quote key is unchanged.
    `#task` lines in its comment stay that block's tasks, linked to its `^h-` ID.
  - Comment declarations never enter `SectionPlan.declarations`, the sticky
    `declaration_orders` used for `s-` task mapping, or the declaring-sticky redirect
    stubs. Keep them in a separate field, e.g. `PlannedSection.anchor_orders`, and leave
    sticky semantics byte-identical.
  - Comment-tiebreaker identity keys (`comment_identity_key` and the existing-region
    side in `sidecar_render.rs`) ignore a first-line section heading in both the `# X`
    and `\# X` spellings, on both sides. Twin quotes then keep their IDs when a
    declaration is added or removed.
- **Removing the declaration** restores the earlier layout on the next scan. Members
  fall back to Other highlights with their tags visible. An existing `^s-` ID keeps the
  removed-section stub rules from phase `edges`.
- **Diagnostic:** `section_declaration_in_comment` now covers only a `# ` line that is
  not the first comment line, which still renders literally. Reword it as
  `section_declaration_in_comment: Page 3 "# Memory" is literal; put "# Name" on the first line of a comment or sticky note`.
  Emit it even when the PDF declares no section, so the mistake is never silent. It
  never changes rendered bytes. The "no declarations → no section diagnostics" rule now
  has this one exception.
- **No other changes:** the region parser, `bob ref show`, and the Mac app need no
  change, because the rendered region shape is identical. Prove it with a round-trip
  test.

### Tests

Put these in `src/native/highlights_ref/tests/sections.rs`, `tests/sections_render.rs`
(or the existing section render test module), and `section_edges` as fits:

- A synthetic sidecar mirroring Bryan's note, with linked pages:
  - an uncommented page-2 highlight;
  - a page-4 highlight commented `# what is it`;
  - a page-4 highlight commented `@what-is-it`;
  - a page-5 highlight commented `This is an interesting foobar… @what-is-it`.

  It renders:
  - `### what is it`;
  - the card `@what-is-it · 3 highlights · Pages 4–5`;
  - `^s-what-is-it`;
  - `#### Page 4` with both page-4 quotes, with no Comment card on either;
  - `#### Page 5` with the comment `This is an interesting foobar…`;
  - then `### Other highlights` with `#### Page 2`.

  Every `^h-` ID equals the IDs the same sidecar gets without the declaration line. A
  second render against the new region is byte-identical.

- Removing the declaration restores the original render, IDs, and literal tags.
- A comment declaration plus remaining comment text: the text stays the comment, and a
  `- #task` line in it keeps linking to the block's `^h-` ID.
- A user-typed `\# Name` first line does not declare.
- A non-first-line `# Name` stays literal, and reports the reworded diagnostic both with
  and without other sections declared.
- Image and standalone-underline comment declarations.
- A `<` follower whose comment starts with `# Name` does not declare.
- Merging a comment declaration with a sticky declaration in both orders: title and slug
  come from the first, and the description from the sticky.
- A tag for another section on the declaring block, and two declarations in one block
  (`section_conflict`).
- Rejected comment declarations (`# !!!`, `# Other highlights`).
- `parse_managed_region` round trip and `bob ref show -f json` `sections`/`section` for
  the rendered fixture.
- Every existing highlights test stays byte-identical, and sticky-declaration behavior
  is unchanged.

### Docs

In `docs/highlights-ref-sync.md` § "Sections (`#` and `@`)":

- Rewrite the first rule of thumb: **"A sticky note or a comment whose first line is
  `# Name` starts a section."** For a sticky, the rest is the description. For a
  comment, the annotation joins the section, and the rest stays its comment.
- Replace "A `# ` line in a highlight, underline, or image comment is ordinary text, not
  a declaration" with the new rule: only the first comment line declares, later lines
  stay literal, and `\#` escapes.
- Add a comment-declaration row to the Joining examples table.
- Update the diagnostic wording.
- Replace "Highlights has not yet been verified to export this spelling unchanged" with:
  Highlights on the Mac exports a comment's `# Name` line verbatim (observed in a real
  sidecar on 2026-10-24); sticky notes and iPad exports are still unverified.

Then run `just docs-check`. Leave the CSS snippet and its doc mirror unchanged.

### Vault check (read-only)

Build this branch and its base commit. Run `bob ref scan --dry-run -f json` against
`~/bob` with each build. Compare `notes[]`, ignoring timestamps. Only
`ref/blogs/the_ai_native_sdlc_playbook.md` may differ (it gains the section). Record the
result in the closing note.

- Never run a non-dry-run sync or scan against the live vault. The Mac's normal scan
  re-renders Bryan's note once the release train ships.
- Do not edit the epic plan's design contract text.

## 3. Verification

- `just check`.
  - The only expected failure is
    `native::completion::kinds::tests::every_value_arg_has_a_decision` (themes
    `rename:old/new`). Epic `bob-cli-bc` owns it, as noted on that bead; do not fix it
    here.
  - Any other failure blocks the closeout.
- `just docs-check` passes.

## 4. Epic closeout (final step)

1. Run `sase bead epic-symbols bob-cli-bd`. It listed none at planning time. Resolve or
   re-key any entry that appears, per the Symvision epic-whitelist policy.
2. Close the epic: `sase bead close bob-cli-bd --note "<verification>"`.
   - Summarize what the lander verified: all 8 phases' code and notes were reviewed;
     macOS CI run 40105623623 at bob-mac-capture 9759771 is green, including the section
     render tests; the inspected light and dark renders look right; triage is on the
     epic.
   - Then add this tale's results: the linked-page-heading CI fix, comment declarations
     with test counts, `just check` and `docs-check` results, and the vault-check diff.
   - Never use `--force` to make the close succeed. If it is rejected for leftover epic
     symbols, clean those up and close again.
3. Run `just symvision` if the justfile has that recipe. It did not at planning time; if
   it is missing, say so in the report.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the `PLAN`
   path printed by `sase bead read bob-cli-bd -r "Need the plan path"`, under the plans
   repo (`202610/ref_note_annotation_sections.md`).
5. `bob-cli-bd` has no `parent_bead`, so no parent handling is needed. Re-check with
   `sase bead read bob-cli-bd -r "Need the parent link"`. If a parent appears, follow
   the parent-bead rules from the lander prompt: close a phase parent after verifying
   it; otherwise stop and note the blocker on it.

## Report to Bryan

In the final response:

- Explain that his `# what is it` was a highlight comment. Comment declarations now
  work, and the note will regroup on the Mac's next scan after the release ships and
  every host updates.
- Say that the Obsidian section-card CSS still needs his visual check: reading view and
  Live Preview, light and dark themes, and mobile.
