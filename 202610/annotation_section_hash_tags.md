---
tier: tale
title: "#name tags join annotation sections"
goal:
  A Highlights annotation joins a declared annotation section when it carries `#name` as
  well as `@name` at a tag position, without changing any published ID or any note whose
  PDF declares no section, and the glossary gains an `annotation-section` term.
size: medium
decisions:
  unknown_hash_tags:
    ask:
      Should an unknown positional `#name` report `section_tag_unknown` only when it is
      a near miss of a section tag?
    choices:
      near_miss:
        Only typo-like near misses report, with a did-you-mean hint; other hashtags stay
        quiet
      always:
        "Every unknown positional #name reports, so #a-style tags in sectioned PDFs add
        noise"
    default: near_miss
    why:
      "#word is already an ordinary Obsidian tag in Bryan's comments (#a, #b); flag
      likely typos only"
    answer: near_miss
  glossary_term:
    ask: Add the glossary:annotation-section memory strand?
    memory:
      - glossary:annotation-section
    default: true
    requested: Also, add a great glossary memory web entry for "annotation section".
    answer: true
proposed_by: bbugyi200.athena.13v
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 15:34:36
status: wip
---

# Plan: `#name` tags join annotation sections

## What Bryan gets

In Highlights, either spelling now files an annotation under a declared section:

```text
Comment on page 4, first line:   # What is it
Comment on page 4, next one:     #what-is-it
Comment on page 5:               This is an interesting foobar… #what-is-it
Comment on page 6:               @what-is-it
```

All three tagged annotations land under `### What is it`, exactly as `@what-is-it` does
today. A matched `#name` is consumed like a matched `@name`, so it never becomes a stray
Obsidian tag on the ref note. A `#name` that matches no section stays as typed, which
means it stays an ordinary Obsidian tag.

The rules of thumb become:

- **A sticky note or a comment whose first line is `# Name` starts a section.**
- **Tag a note with `@name` or `#name` at the start or end of a line to file it there.**
  If you can still see the tag, it did not group.
- **The space matters:** `# Memory` declares, and `#memory` joins.

Rendering, `bob ref show`, the Mac Refs panel, `^h-` IDs, tasks, and every note whose
PDF declares no section stay exactly as they are.

## Context

Annotation sections shipped in epic `bob-cli-bd`
(`plan:202610/ref_note_annotation_sections.md`; read the bead with
`sase bead read bob-cli-bd -r "<why>"`). Its landing tale (commit 8a7b02e1) added
first-line `# Name` declarations on highlight, underline, and image comments. Bryan's
real note `ref/blogs/the_ai_native_sdlc_playbook.md` already groups through a comment
declaration plus `@what-is-it` tags.

All grammar lives in `src/native/highlights_ref/sections.rs`:

- `scan_section_tags` / `token_at` / `positional_region` / `positional_hit_indices` find
  tags.
- `display_source` consumes them.
- `trailing_aliases` / `valid_alias` read declaration aliases.
- `strip_identity_tags` (behind `section_match_key`) keeps tag edits from changing IDs.
- `plan_sections` emits the diagnostics.

`sidecar_render.rs` calls `display_source(…, &BTreeSet::new())` for every PDF, including
PDFs with no sections. Any new escape handling therefore reaches every note.

Facts checked against the live vault (`~/bob/lib`, read-only):

- Highlights exports `#word` comment text verbatim. 19 `#task` lines exist, plus
  line-start `#a` / `#b` comments in `lib/docs/sase_AGENTS_v2.md`,
  `lib/docs/sase_AGENTS_v3.md`, and `lib/chat/tailnet_agent_fleet.md`.
- There is one `#git:home` in a chat file.
- No sidecar contains `\#word` or `\@word`.
- The sidecar parser never treats `#memory` as a heading, because
  `markdown_heading_level` requires whitespace after the hashes. So `#memory` lines
  already reach comment and sticky text unchanged.

## Contract

### The `#name` token

Everything about `@name` stays as it is. A `#name` token is:

- `#`, then a Unicode letter or digit, then letters, digits, `_`, or `-`, then
  optionally one of `.,;:!?`, which is part of the token. Same as `@name`.
- At the start of the line or after whitespace. The escaped form is `\#name`, where the
  `\` starts the line or follows whitespace. Same as `@name`.
- Followed by whitespace or the end of the line. This rule is new and applies to `#`
  only. Because of it, Obsidian nested tags (`#area/memory`), `#git:home`, and
  `#memory)` are never section tags.
- Made of more than digits, which is Obsidian's own tag rule. So `#1` and `#2024` are
  never section tags.
- Never keyed `task`. `#task`, `#Task`, and `#task.` are never section tags, and `#task`
  keeps meaning a task line.

Things that are not tokens at all: `# Memory` (a declaration), `## Memory`, `C#`,
`page#anchor`, `[[note#Heading]]`, `[x](#anchor)`, and `#` inside inline code or fenced
code.

### Position, matching, and consumption

These are identical for both sigils:

- **Position.** A tag must be in the maximal run at the start or end of its line, after
  stripping a first-line `Comment:` / `Note:` label and one list marker.
  - A run may mix sigils: `@memory #visuals` is a two-tag run.
  - Tags on a line containing `#task`, in code, on fenced lines, or on image-note path
    lines are never positional.
- **Matching.** `section_key` works the same way and the allowlist is the same. The same
  keys match: `#OpenQuestions`, `#open-questions`, and `@open_questions` all join "Open
  questions". Forward references work.
- **One home per displayed block.** The rules are unchanged, and a comment declaration
  still beats tags.
  - `@memory #memory` on one block joins Memory, and both tags are consumed.
  - `#memory @visuals` reports `section_conflict` and keeps Memory.
- **Consumption.** Once a home is chosen, every positional `@` or `#` tag whose key
  belongs to it is removed, together with its punctuation. Empty lines, list items, and
  comments disappear exactly as they do today. Every other tag stays as typed.
- **Escapes.**
  - `\#name` at a tag position never resolves, and it renders exactly as typed,
    backslash included. Obsidian already displays `\#name` as plain `#name` text without
    creating a tag. Stripping the backslash (what `display_source` does for `\@name`)
    would turn it into a real tag, and would change notes with no sections.
  - `\@name` keeps today's behavior.
- **Tags-only stickies.** A sticky whose only content is tags, in either sigil, does not
  join. It reports `section_tag_only_note`, as today.
- **Routes.** `#task` routes stay `@name` only. A `#name` at the end of a `#task` line
  is prose: it is not a route, not positional, and not a `section_tag_on_task`.
- **No declarations.** A PDF with no declarations still gets no consumption and no
  section diagnostics, and its render is byte-identical to today's.

### Declarations

- **Aliases.** The trailing alias run on a `# ` heading line accepts `#alias` as well as
  `@alias`, so `# Open questions #q` means the same as `# Open questions @q`.
  - A `#alias` follows the token rules above: more than digits, never keyed `task`.
  - A rejected trailing `#word` stays in the title source as today, and
    `sanitize_section_title` drops its `#`. So `# Chapter #1` is still titled "Chapter
    1".
  - Today a trailing `#q` ends up glued onto the title ("Open questions q"), so this
    only improves titles.
- **The canonical tag stays `@`-spelled** on the card, in region parsing, and in
  `bob ref show` JSON (`tag: "@q"`), whichever sigil Bryan typed. Here is why:
  - a `#tag` in every card header would add an Obsidian tag to every sectioned note;
  - `region.rs` reads a card's tag only when its first token starts with `@`;
  - the JSON contract and the Mac app depend on that spelling.

### Diagnostics

- **`section_tag_unknown` for `#`** follows the `unknown_hash_tags` decision.
  - With `near_miss`, emit it only when `closest_section_tag` finds a key within edit
    distance 2. Otherwise emit nothing, so the scan's `section_tags_unknown` count also
    stays quiet.
  - With `always`, emit it for every unknown positional `#name`, exactly as for `@name`.
  - `@name` behavior is unchanged under either choice.
- **Hints use the typed sigil:**
  `section_tag_unknown: Page 4 #memry (did you mean #memory?)`. The hint text for `@` is
  unchanged.
- **Unchanged:** the conflict, duplicate, tag-only, on-task, and route-hint texts still
  name sections by their canonical `@` tag.

### Identity (no churn)

`section_match_key` also strips `#name` tokens wherever they appear, escaped or not,
together with their punctuation and the whitespace before them.

- **One predicate.** It uses exactly the token rules above (start or whitespace before,
  boundary after, more than digits, never keyed `task`). Share that predicate with the
  scanner, so `C#`, URL fragments, wikilink headings, nested tags, and `#task` stay part
  of every key.
- **Both sides change together.** The key changes at the same moment for the sidecar
  head and for the parsed existing region, so published IDs keep matching through the
  primary and tiebreaker paths, just as the epic designed for `@word`.
- **Only unpublished IDs move.** Fresh IDs for not-yet-published sticky notes that
  contain a `#word` hash the new key. The vault check below must prove that no published
  ID moves.
- **Tag edits never churn.** Adding a `#memory`, removing it, or swapping it for
  `@memory` never changes a `^h-` ID, creates a tombstone, or re-creates a task.

## Implementation

1. **`sections.rs` tokens.** Put the token shape in one place, for example a sigil-aware
   `token_at` that returns the sigil, plus a small predicate the identity normalizer
   reuses.
   - `scan_section_tags` looks for both `@` and `#`.
   - `RawTagHit` and `SectionTagHit` gain the sigil, for example `sigil: char`.
   - `positional_hit_indices` keeps working on spans and escapes unchanged.
   - Keep `tag_name = hit.raw[1..]`, which works because both sigils are one byte.
2. **`display_source`.** Remove the escape slash only for `@` hits. `#` escapes stay
   verbatim. Positional consumption covers both sigils.
3. **`trailing_aliases` / `valid_alias`.** Accept `#alias` tokens under the rules above.
4. **`strip_identity_tags`.** Also remove scanner-shaped `#name` tokens. Keep the
   empty-result fallback in `section_match_key`.
5. **`plan_sections` and `closest_section_tag`.** Implement the selected
   `unknown_hash_tags` branch and the typed-sigil hint.
   - `source_is_only_tags` and the OnTask and Escaped kinds need no new logic beyond the
     scanner changes. Confirm that a `#name` never yields `OnTask`.
6. **Comments and docs.** Update the module doc and comments that say "`@name`" or
   "@-insensitive" where they now mean either sigil. This includes the
   `note_identity_key` comment in `sidecar_render.rs`.
7. **Leave these unchanged** and confirm by reading:
   - `sidecar.rs`;
   - `region.rs`;
   - `annotation_tasks.rs` (routes and `section_for_route`);
   - `show.rs` and `report.rs` formatting;
   - `scan_json.rs`, whose counts derive from the diagnostics.

   No bob-mac-capture change is needed: it renders `bob ref show` JSON, whose `tag`
   stays `@…`.

## Tests

`src/native/highlights_ref/tests/sections.rs`:

- Scanner:
  - `Comment: #Memory.`, `- #open_questions`, `#memory` alone, and `Notes #memory` are
    positional;
  - `Compare #memory mid-sentence` is not;
  - `C#`, `page#anchor`, `[[n#H]]`, `#area/memory`, `#git:home`, `#2024`, `#task`, and
    `#Task` yield no hit;
  - `` `#memory` `` and fenced `#memory` yield no hit;
  - `- #task Draft #memory` yields no positional hit;
  - `\#memory` is Escaped;
  - mixed runs `@memory #visuals` and `#a Text #memory` work.
- `display_source`:
  - `#memory` and `- #memory` are consumed to nothing;
  - `A note #memory!` becomes `A note`;
  - `#memory @memory` is fully consumed;
  - `#other` stays;
  - `\#memory` stays `\#memory`;
  - `\@memory` still becomes `@memory`.
- Planner:
  - `#memory` joins;
  - a `#` forward reference joins;
  - `#memory @visuals` conflicts and keeps Memory;
  - `#memry` reports with a `#memory` hint;
  - a far unknown `#a` follows the decision (silent under `near_miss`);
  - a `#memory`-only sticky reports `section_tag_only_note` and does not join;
  - a comment declaration beats a `#name` tag for another section exactly as it beats an
    `@name` tag;
  - with no declarations, `#memory` yields an empty plan and no diagnostics.
- Declarations:
  - `# Open questions #q` gives the title "Open questions", keys `{openquestions, q}`,
    and tag `@q`;
  - `# Chapter #1` stays "Chapter 1";
  - `# Notes #task` keeps `task` in the title.
- Identity:
  - `section_match_key` strips `#Memory,` the same way it strips `@Memory,`;
  - it keeps `#task`, `C#`, `page#x`, and `#area/memory`;
  - it strips `\#memory`, with the all-tag fallback.

`src/native/highlights_ref/tests/sections_render.rs`:

- **Hash tag toggles.** Do the same as
  `identity_survives_tag_toggles_and_declaration_changes`, using `#memory`: add it,
  remove it, swap `@memory` ↔ `#memory`, and delete the declaration. Expect the same
  live IDs, no `Removed highlights`, and identical region bytes for the `@` and `#`
  spellings.
- **Playbook shape.** Do the same as
  `comment_declaration_groups_bryans_note_and_keeps_ids`, with `#what-is-it` in place of
  `@what-is-it`. Expect identical output.
- **No declarations.** A PDF without declarations whose comments contain `#memory`,
  `\#memory`, and `#a Text` renders byte-identical to the current output.

Every existing highlights test and fixture must pass unchanged.

## Docs

The Sections subsection of `docs/highlights-ref-sync.md` keeps its heading, "Sections
(`#` and `@`)". Make these changes:

- **Rules of thumb.** Use the three bullets from "What Bryan gets". The second rule says
  "`@name` or `#name`", and a new bullet covers the space.
- **Aliases sentence.** Allow `# Open questions @q` or `# Open questions #q`, and say
  that the card always shows the `@` spelling.
- **Positional-tag paragraph.**
  - A tag is a whole `@name` or `#name` token, and one line may mix them.
  - A `#name` must end at whitespace or the end of the line, and must contain more than
    digits.
  - `#task` is never a section tag.
  - Nested Obsidian tags never group.
  - A matched `#name` is consumed, so it never becomes an Obsidian tag on the ref note.
    An unmatched one stays an ordinary Obsidian tag.
  - `\#name` renders exactly as typed, which Obsidian shows as plain `#name` text.
- **Examples table rows:**
  - `#memory`: joins Memory, and the tag is consumed;
  - `Compare with the intro #memory`: joins, and the trailing tag is consumed;
  - `#memory` versus `# Memory`: a tag versus a declaration;
  - `#memry`: stays visible, and reports `section_tag_unknown` with a `#memory` hint
    under the selected `unknown_hash_tags` branch (spell out what a far-off hashtag does
    under that branch);
  - `#area/memory`: a nested Obsidian tag, never grouped;
  - `\#memory`: shown as plain `#memory` text, never grouped;
  - `- #task Draft #memory`: a task, not a section tag.
- **Accepted costs.** Keep them, and remove nothing.

Make no other docs changes. `docs/ref.md` (`bob ref show`),
`docs/reference/json-outputs.md`, and the mounted mac-capture docs are unaffected,
because the card tag, JSON, and Mac feed do not change. Run `just docs-check`.

## Memory

> [!decision] glossary_term
>
> Use `/sase_memory_write`, then create `sase/memory/glossary/annotation-section.md`
> with exactly this frontmatter and body. You may wrap the lines differently, and
> tighten the wording only to match the shipped behavior, for example the selected
> `unknown_hash_tags` branch.
>
> ```md
> ---
> keyword: Annotation Section
> aliases:
>   - section tag
>   - section card
> ---
>
> A named group of Highlights annotations inside a [[reference-note]]'s managed
> `## Highlights` region. A `# Name` line declares it: the first line of a sticky note
> (its later lines become the section's description) or of a highlight, underline, or
> image comment (that annotation joins the section). Trailing `@alias` or `#alias`
> tokens add match keys. An annotation joins by carrying `@name` or `#name` as a whole
> token at the start or end of a line. Matching ignores case and punctuation, and a
> matched tag is consumed, so a tag that still shows did not group. Mid-sentence tokens,
> `#task` lines, and `\` escapes never join. The space matters: `# Memory` declares,
> `#memory` joins. Each displayed block has one home, and a comment declaration beats
> tags. Each section renders as `### Name`, then a `[!summary|section]` card captioned
> `@tag · N highlights · Pages`, its `^s-<slug>` block, and its members under `####`
> page captions, ahead of Other highlights and Removed highlights. `bob ref show` and
> the Mac Refs panel group the same way. Membership is re-derived from the sidecar on
> every scan and nothing new is stored. Tag edits never change `^h-` IDs, and a PDF with
> no declaration renders unchanged. Not a Markdown `##` heading or a `bob capture`
> target section. See `docs/highlights-ref-sync.md`.
> ```
>
> Then run `sase memory init`. Confirm that the glossary roster lists "Annotation
> Section (section tag, section card)" and that
> `sase memory read glossary:annotation-section -r "verify new strand"` prints the body
> with Reference Note pulled in. Do not hand-edit `AGENTS.md` or the provider shims.

> [!decision] glossary_term = no
>
> Write no memory. If no human reviewed this plan, file the skipped strand through
> `/sase_new_task` as a `memory` task bead.

No other memory note changes. No existing note states that sections are `@`-only (the
epic declined both its decisions strand and its glossary term).

## Verification

1. `cargo test --lib highlights_ref::` while iterating, then `just check` (fmt, clippy,
   lib and CLI tests) and `just docs-check`.
   - Unrelated failures already owned elsewhere (for example the `bob-cli-bc` themes
     rename test the bd landing saw) are not this tale's work. Route them under the
     normal bead rules, citing node IDs.
2. **Vault zero-change check (read-only).** Build `bob` from this change and from its
   base commit. Run `bob ref scan --dry-run -f json` against `~/bob` with each build.
   `notes[]` must be identical; the bd landing recorded 522 unchanged and 0 updated.
   This proves that `#a`/`#b` comments, `#task` lines, and the playbook note keep their
   bytes and IDs.
3. **Scratch end-to-end check.** Never write to the real vault.
   - Copy into a temp vault (`-b <tmp>`):
     - `lib/blogs/the_ai_native_sdlc_playbook.pdf`;
     - `lib/blogs/the_ai_native_sdlc_playbook.md` (the sidecar);
     - `ref/blogs/the_ai_native_sdlc_playbook.md`.
   - In the copied sidecar, replace both `@what-is-it` tags with `#what-is-it`, then run
     `bob ref sync <tmp pdf> -b <tmp> --dry-run`, then a real sync.
   - The managed region must be byte-identical to the original.
   - A second sync must be a no-op.
   - Restoring `@what-is-it` must keep every `^h-` ID.
   - If the scratch sync cannot run outside the real vault (hooks, config), record why
     in the final report and rely on the playbook-shape render test.
4. **Release train.** Commit as `feat(highlights-ref): …` without `!`, because no CLI or
   JSON contract breaks. Never edit the `Cargo.toml` version or `CHANGELOG.md`
   (`decisions:bob-cli-releases-ride-a-daily-train`).

## Out of scope

- Unknown-tag diagnostics for PDFs that declare no section.
- A "missing space" hint for a `#Name` line that was meant as a declaration.
- Changing `@name` token rules (for example the end boundary `#` now enforces).
- Showing `#tag` on cards.
- Route suffixes spelled with `#`.
- A decisions strand for annotation sections.
