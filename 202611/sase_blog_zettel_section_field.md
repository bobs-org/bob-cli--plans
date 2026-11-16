---
tier: tale
title: Add a section field to the SASE Blog Zettel album
goal:
  Every zettel can name the blog-post section it targets, from a fixed album-owned value
  set, with the spine and Base matching.
size: small
decisions:
  why_section:
    ask: Should the album and the spine keep a "Why I use SASE" section?
    choices:
      omit:
        Leave it out, matching ^section-field; the spine drops its "Why I use SASE" stub
      keep:
        "Add `why: 🙋 Why I use SASE` after `intro`; the spine keeps that stub as
        section 1"
    default: omit
    why:
      Every version of the ^section-field outline omits it; adding it later is one YAML
      line
    answer: omit
  close_section:
    ask: Should the album and the spine keep a separate Conclusion after Future Work?
    choices:
      omit: Future Work closes the post; the spine drops its "Conclusion" stub
      keep: "Add `close: 🏁 Conclusion` after `future`; the spine keeps that stub last"
    default: omit
    why:
      The ^section-field outline ends at Future Work; adding it later is one YAML line
    answer: keep
proposed_by: bbugyi200.apollo.ee
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 08:48:34
status: wip
---

# Add a `section` field to the SASE Blog Zettel album

## Goal

Give the `~/bob/sase_blog_zettel.md` album a `section` record field. It names the
section of the SASE blog post that a zettel (record note) is written for, and its fixed
value set comes from the `^section-field` task in `~/bob/sase_blog.md`. Bryan reviewed
the consolidated research report and **accepted every requirement it recommends**. Read
it before starting:

```bash
sase artifact read research:202611/sase_blog_zettel_section_schema/sase_blog_zettel_section_schema.md "Implementing the sase_blog_zettel section field"
```

Accepted requirements this plan implements (report adjustments A1–A6):

1. **A1:** add one flat, optional, scalar choices field. In Bob terms this is
   `record.fields.section`, and each record stores a top-level `section:` key. There is
   no `default`, `required`, `list`, or `suggest`.
2. **A2:** use 13 values. Twelve come from the `^section-field` outline, flattened, plus
   `intro`. Whether to also add "Why I use SASE" and "Conclusion" is decided by the
   `why_section` and `close_section` decisions.
3. **A3:** keys are short concept slugs that never change; labels are short emoji text
   and can be edited freely. The `families` label uses the current glossary term,
   "Sessions, clans & tribes".
4. **A4:** add no other record fields. Retire `part`, and set
   `index_facets: [section, claim]`.
5. **A5:** move the one existing record from `part: why` to `section: macros`. Add a
   section column and view to `sase_blog_zettel.base`. Bring the spine
   (`sase_blog_contents.md`) headings in line with the new outline.
6. **A6:** keep one outline vocabulary (the album) and one outline order and wording
   (the spine). Retire the stale `## Outline` in `sase_blog.md`. The `^section-field`
   task's bullet list retires when Bryan closes that task (see Out of scope).

Every candidate extra field (`order`, `subsection`, `post`, `proof`, `words`, task or
heading links, `required: true`, `list: true`, `index_by`) was rejected in the report's
audit, and this plan agrees. **Do not add any other field.**

## Ground truth (verified 2026-11-16)

- **No bob-cli code changes.** This is vault content plus existing `bob` commands. The
  album and record contract is in this repo's `docs/records.md`. Section numbering is
  `bob articles format`.
- `~/bob/sase_blog_zettel.md` has fields `status` (required, 6 choices), `claim`
  (required text), and `part` (optional choices `hook`/`why`/`what`/`why_not`/`close`).
  It has `index_facets: [part, claim]` and one record.
- `~/bob/sase_blog_zettel/prompt_management_macros.md` has `status: seed`, `part: why`,
  and the claim "Prompts should be managed like code…".
- `bob albums check` and `bob records check zettel` are both clean (0 errors).
- `bob capture --dry-run` already plans zettel records with `- field:: value` child
  bullets.
- The `index_facets` parser reads the inline form `[a, b]` and the indented block form
  (`  - a`). It silently ignores an **unindented** block list (`- a` at column 0), which
  would quietly fall back to default facets. Keep the inline form, and edit that line in
  place rather than re-serializing the frontmatter. (This parser bug is filed as
  `bob-cli-m9`; do not fix it here.)
- `bob albums update` (the Schema Card envelope) cannot write `index_facets`, derives
  long keys from labels, and refuses key renames. That is why the schema addition is a
  hand YAML edit.
- **Migration-order trap (scratch-verified in the report):** if `part` is removed from
  the schema while the record still carries `part: why`, `bob records check` errors.
  `bob records update … -s part=` is then refused as "not a Zettel field". Clear the
  record's `part` **before** removing the field.
- Nothing in the vault links to `sase_blog#^outline`, `sase_blog#Outline`, or any
  `sase_blog_contents#…` heading (only `.bob/ref-sources` copies mention them), so the
  spine and outline edits break no links.
- `sase_blog_contents.md` has a ~190-word lede and footnote-style TODO lines above its
  first `##`. Every numbered section is a TODO stub. The old `## 2. What is SASE?`
  subsection ideas are preserved verbatim in `sase_blog.md` `## Requirements` under
  "Things that sase generalizes:", so replacing those stub headings loses nothing. Its
  four `pseudo-footnote` warnings already exist and are not in scope.

## Target album frontmatter

Only `index_facets` and the `part` → `section` block change. Every other line stays
byte-identical. There are no YAML comments, `index_facets` stays inline, and choices are
declared in outline order.

```yaml
---
parent: "[[sase_blog]]"
type: "[[album]]"
summary: Topic-sized pieces of my SASE posts, one claim per record, in my own voice.
emoji: 📇
created: 2026-11-12T13:54:08-0500
index_facets: [section, claim]
record:
  noun: zettel
  label: Zettel
  dated_notes: true
  fields:
    status:
      required: true
      values:
        seed: 🌱 Seed
        draft: ✍️ Draft
        ready: ✅ Ready
        placed: 📌 Placed
        later: ⏭️ Later
        cut: ✂️ Cut
    claim:
      required: true
    section:
      values:
        intro: 🪝 Intro
        what: 🧱 What is SASE?
        macros: 📜 Prompt language (macros)
        turn: 🔁 Agent turn
        gate: 🚦 Gate
        families: 👪 Sessions, clans & tribes
        memory: 🧠 Memory
        plan: 🗺️ Plan & artifacts
        project: 📁 Project
        automation: ⚙️ Automation (service)
        demos: 🎬 The Demos
        not_ready: 🛑 Why SASE is still NOT ready
        future: 🔮 Future Work
---
```

> [!decision] why_section = keep Insert `why: 🙋 Why I use SASE` between `intro:` and
> `what:`.

> [!decision] close_section = keep Append `close: 🏁 Conclusion` after `future:`.

`what` holds overview and transition material about SASE as a whole. It is not a bucket
for "undecided" material; leave `section` empty for that. The task's "should include
artifacts!" NOTE lives in the `plan` label.

## Steps

1. **Preflight.** Run `bob vault-sync` so the vault matches its remote. Re-read the
   three vault files and compare them with Ground truth. If the album now has more than
   one record, list them with `bob records list zettel`. Backfill each one with the
   `section` key its `claim` targets, and leave `section` empty when no value clearly
   fits. Never guess from `part` alone. Record `git -C ~/bob log -1 --format=%H` as the
   starting point.
2. **Add `section`, keeping `part` for now (hand edit).** In
   `~/bob/sase_blog_zettel.md`, change the `index_facets:` line in place to
   `index_facets: [section, claim]`. Insert the `section:` block from the target
   frontmatter directly after `claim:`, and leave the `part:` block in place for this
   step. Run `bob albums check` (0 errors for this album). Then run
   `bob albums list -f json`: the `sase_blog_zettel` entry must list
   `index_facets: ["section", "claim"]` and a `section` field whose choices are in the
   order above.
3. **Migrate the record (guarded).** Preview first, then apply:

   ```bash
   bob records update zettel prompt_management_macros -d -s section=macros -s part=
   bob records update zettel prompt_management_macros -s section=macros -s part=
   ```

   The record must now carry `section: macros` and no `part:` key, with every other byte
   unchanged. This write also rebuilds the album's `[!records]` index in the same
   transaction, and its pre-sync commits the step-2 hand edit.

4. **Remove `part` (guarded).** Read the album's current `schema_fingerprint` (it
   changed in step 2), then preview and apply:

   ```bash
   FP=$(bob albums list -f json | python3 -I -c 'import json,sys; print(next(a["schema_fingerprint"] for a in json.load(sys.stdin)["albums"] if a["stem"]=="sase_blog_zettel"))')
   printf '{"schema_version":1,"expect":"%s","ops":[{"op":"field.remove","field":"part"}]}' "$FP" | bob albums update sase_blog_zettel -d
   printf '{"schema_version":1,"expect":"%s","ops":[{"op":"field.remove","field":"part"}]}' "$FP" | bob albums update sase_blog_zettel
   ```

   The preview must say the field was unused. If the guarded path refuses for an
   unexpected reason, stop and diagnose rather than forcing it. A hand removal of the
   `part:` block is acceptable only once step 3 succeeded and `filled` for `part` is 0.
   Afterwards the album frontmatter must equal the target frontmatter exactly (after
   applying the decisions).

5. **Validate the album.** `bob albums check` must show 0 errors for `sase_blog_zettel`,
   `bob records check zettel` must show 0 errors and 0 warnings, and
   `bob albums list -f json` must report `index.state: current`. The index row must read
   `… — 📜 Prompt language (macros) · Prompts should be managed like code…`. If the
   index is stale, run `bob albums sync`, preview with `-d` first, and stop if the diff
   touches unrelated albums or catalogs in a surprising way.
6. **Prove capture works (dry run only; never create a real test record).**

   ```bash
   printf 'A gate is plan mode, generalized @sase_blog_zettel/\n- status:: seed\n- claim:: Plan mode is one instance of a deeper primitive.\n- section:: gate\n' | bob capture --dry-run
   ```

   This must plan `section: gate`, and the index preview must show `🚦 Gate`. The same
   draft with `- section:: gates` must be refused (exit 2) with the valid keys listed. A
   draft with `- part:: why` must also be refused.

7. **Update the Base (hand edit).** In `~/bob/sase_blog_zettel.base`, add `section` and
   `claim` to the existing "📇 All Zettels" view's `order` after `status`. Append this
   second view, matching the file's existing block-list style:

   ```yaml
   - type: table
     name: "🧭 By section"
     filters:
       and:
         - note.status != "cut"
         - note.status != "later"
     groupBy:
       property: section
       direction: ASC
     order:
       - formula.name
       - status
       - claim
     sort:
       - property: formula.name
         direction: ASC
     summaries:
       formula.name: Count
   ```

   **Do not add `groupOrder`.** Bases hides any value missing from that list, so a
   section added later would silently disappear from the view. Groups sort by raw key,
   which is the accepted cost. Check that the file parses as YAML (for example with
   `python3 -I -c 'import sys, yaml; yaml.safe_load(open(sys.argv[1]))' ~/bob/sase_blog_zettel.base`).
   Leave the filters, formulas, and properties blocks unchanged.

8. **Reconcile the spine headings.** In `~/bob/sase_blog_contents.md`, keep the
   frontmatter and everything above the first `##` heading (the lede and the `[1]`,
   `[2]`, `[a]`, `[b]` TODO lines) byte-identical. Replace the numbered sections with
   this outline (shown with default decisions):

   ```markdown
   ## 1. What is SASE?

   ### 1.1 What is a "prompt language" (macros)?

   TODO

   ### 1.2 What is an "agent turn"?

   TODO

   ### 1.3 What is a "gate"?

   TODO

   ### 1.4 What are agent sessions, clans, and tribes?

   TODO

   ### 1.5 What is a "memory"?

   TODO

   ### 1.6 What is a "plan"?

   TODO: Include artifacts!

   ### 1.7 What is a "project"?

   TODO

   ### 1.8 What is an "automation" (service)?

   TODO

   ## 2. The Demos

   TODO

   ## 3. Why SASE is still NOT ready?

   TODO: Potential first line: "And might never be... At least in its current form and
   name."

   ### 3.1 My Definition of "AI Slop"

   TODO

   ## 4. Future Work

   TODO
   ```

   - Heading wording follows Bryan's `^section-field` bullets, except 1.4, which uses
     the current glossary term (A3).
   - "My Definition of 'AI Slop'" is an existing spine heading kept under its successor
     parent, because the `## Requirements` AI-slop bullets still target it.
   - The task's two NOTEs and its "POTENTIAL FIRST LINE" now live in stub bodies, which
     `bob articles` still counts as TODO stubs.

   > [!decision] why_section = keep Keep `## Why I use SASE` with body `TODO` as the
   > first section, before "What is SASE?". Every later number shifts by one.

   > [!decision] close_section = keep Keep `## Conclusion` with body `TODO` as the last
   > section, after "Future Work".

   Then run `bob articles format sase_blog_contents`, which owns numbering and the
   trailing newline. Afterwards `bob articles format -c sase_blog_contents` must exit 0
   (the four existing pseudo-footnote warnings may remain).
   `bob articles outline sase_blog_contents` must print exactly the outline above.
   Confirm with `git diff` that no lede or footnote byte changed.

9. **Retire the stale `## Outline` in `sase_blog.md`.**
   - Cut the whole `## Outline` section: the heading, the standalone `^outline` line,
     and every bullet down to (not including) `## Later Posts`.
   - Append it to `~/bob/sase_blog_0_legacy_notes.md` (the "Obsolete notes" file) under
     a new `## Retired sase_blog outline (XPrompts/ACE era, retired 2026-11-16)`
     heading. Copy the outline-tree bullets (Introduction, Overview, XPrompts, ACE, AXE,
     with all children) verbatim, and drop the `^outline` id.
   - The three dated action bullets that were in that section are live requirements, not
     outline. Move them verbatim, with their children and `[created::]` stamps, to the
     end of `sase_blog.md`'s `## Requirements` list (just before `## Screenshots`):
     "Move 15-minute install post…", "Read
     [[ref/chat/blog00_launch_post_review_consolidated…]]!" (with its embed child), and
     "Start the blog post off with a brief timeline…".
   - Do not touch the bob-owned `[!projects]` and `[!albums]` callouts, any task line,
     or the `^section-field` task and its children.
   - Re-run `bob albums check` to confirm the `## Albums` owner catalog is still
     `current`.
10. **Publish and verify.** Run `bob vault-sync`, then `bob vault-sync status --json`
    (it must report success). `git -C ~/bob status --short` must be clean, and
    `_conflicts/sync_conflicts.md` must have no new entries since step 1. Review
    `git -C ~/bob diff <step-1 SHA>..HEAD --stat`: only `sase_blog_zettel.md`,
    `sase_blog_zettel/prompt_management_macros.md`, `sase_blog_zettel.base`,
    `sase_blog_contents.md`, `sase_blog.md`, and `sase_blog_0_legacy_notes.md` may carry
    this work's changes. Other files changed by Bryan's other hosts in between are
    expected; do not revert them.

## Acceptance checks

- The album frontmatter equals the target (with decisions applied). `bob albums check`
  and `bob records check zettel` report 0 errors for this album, and
  `index.state: current`.
- The record has `section: macros` and no `part`. The index row shows
  `📜 Prompt language (macros) · <claim>`.
- The capture dry runs: `section:: gate` is accepted, while `section:: gates` and
  `part:: why` are refused.
- The Base parses as YAML, has `section` and `claim` columns, has the "🧭 By section"
  view, and has no `groupOrder`.
- `bob articles outline sase_blog_contents` matches step 8, `format -c` exits 0, and the
  lede and footnotes are unchanged.
- `sase_blog.md` has no `## Outline` section. The legacy notes file holds the retired
  tree, and the three action bullets end `## Requirements`.
- The vault is synced and clean, with no new sync conflicts.

## Out of scope

- **Closing `^section-field`.** That is Bryan's gesture after he reviews the result.
  Closing it (and the nightly `bob task archive`) retires its outline bullets into
  `done/`. Do not edit or close that task or any other task line.
- The `index_facets` unindented-block parser bug in bob-cli (task bead `bob-cli-m9`).
- Writing any spine prose, fixing the spine's pseudo-footnote warnings, or renumbering
  the stale `§` references in the blocked `zd-*` task titles.
- Mac Capture's unfocused choice-pill strip clipping past about six pills (cosmetic; the
  focused completion list scrolls and filters).
- Any `## Notes` or `## Draft` content in records, and any new records.

## Report back to Bryan

- One line per changed vault file, plus the commands that verified it.
- **Verify in Obsidian on the Mac:** open `sase_blog_zettel.base` and check how the "🧭
  By section" view renders records with an empty `section`. This is undocumented and
  cannot be checked from Linux.
- He can now close `^section-field`, and his next step is drafting (for example,
  dictating the `prompt_management_macros` draft) rather than more schema work.
- Adding a section later is one YAML line in the album. Rewording a label is free.
  Renaming or removing a used key means retagging its records first.
