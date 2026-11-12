---
tier: tale
title: Set up the sase_blog_zettel album and usage guide
goal:
  Create and validate the sase_blog_zettel album in the vault and commit a usage guide
  to the research sidecar.
size: medium
decisions:
  seed_records:
    ask:
      Should the implementing agent also pre-create one 🌱 seed record per current
      outline topic?
    choices:
      none:
        Album and guide only; Bryan seeds every claim himself via the guide's
        batch-capture template
      stubs:
        Also create one 🌱 record per outline topic with a placeholder claim Bryan must
        replace
    default: none
    why:
      Claim-authoring is the heart of Bryan's S0; placeholder claims hollow out the
      required-claim rule
    answer: none
proposed_by: bbugyi200.apollo.research.4f.linker.w0
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 13:48:19
status: wip
---

# Set Up The `sase_blog_zettel` Album And Its Usage Guide

## Context

Bryan is drafting a blog post about SASE (the `sase-org/sase` repo) and wants to draft
each topic as a record note under a new album, `~/bob/sase_blog_zettel.md`, so the final
writing pass is mostly arranging already-written material. A five-researcher
consolidated report evaluated and adjusted this idea; **Bryan has agreed to all of its
requirement adjustments (R1–R8)**, so this plan implements that report's "Recommended
solution" verbatim where it specifies mechanics. Required context reads for the
implementer:

```bash
sase artifact read research:202611/sase_blog_drafting_in_records/sase_blog_drafting_in_records.md "Implementing the blog-zettel album this research specifies"
sase bead read bob-cli-k3 -r "Album/record model context for the blog-zettel album setup"
```

Key adjusted requirements the deliverables must reflect:

- **R1**: one record per _topic/claim_, never per outline section; many-to-one with
  sections; some records never ship.
- **R2**: assembly in the spine **plus one real rewrite pass**; `^v1-draft` stays a
  writing task.
- **R3**: album is `~/bob/sase_blog_zettel.md`, parent `[[sase_blog]]` (a project note),
  noun `zettel`, emoji 📇.
- **R4**: assemble early after Wave 1; maturity (`ready`) ≠ inclusion.
- **R5**: `sase_blog_contents.md` (the article) is the spine and the only authority for
  order and transitions; the outline is unfinalized candidate coverage; **no
  `order`/`section`/`wave` fields** in the schema.
- **R6**: one home per kind of material (routing rule, reproduced in the guide).
- **R7**: zero new bob/plugin code; article-only writing is the explicit fallback.
- **R8**: the `ready` gate includes a name/fact check against current sase HEAD (Macros
  not Xprompts, Patches not ChangeSpec).

Relevant background (already verified by the research lead on 2026-11-12): the
`bob-cli-k3` album/record epic has all 11 phases closed, `bob albums create` ships in
the installed `bob`, the vault has four existing albums (`eat`, `podcasts`, `tv`,
`movies`), and `~/bob/sase_blog.md` is the project note that will own this album.
`docs/records.md` in the bob-cli repo is the authoritative album/record contract.

## Goal

1. Create and validate the `sase_blog_zettel` album in the live vault with the
   research-validated schema.
2. Author a practical usage guide for Bryan and commit it to the `bob-cli--research`
   sidecar repo.

## Non-Goals

- No edits to `~/bob/sase_blog_contents.md` (the spine) or hand edits to
  `~/bob/sase_blog.md` — the `## Albums` catalog row there is machine-owned and written
  by bob itself.
- No new bob features, plugins, or assembly/export scripts (R7; the research explicitly
  rejects an exporter for now).
- No fix for the cosmetic `1 zettels` singular-tally bug or the stale
  `bob capture --help` text — both are already recorded as `DISCOVERED ISSUE:` notes on
  bead `bob-cli-k3` for its land agent. Expect the tally artifact to appear on the first
  record; do not chase it.
- No SASE memory edits.
- No authoring of real claims or draft prose on Bryan's behalf — that is his S0 writing
  session (see `seed_records` decision for the only exception, and even `stubs` writes
  placeholders, never real claims).

## Step 1 — Preflight

From any directory on this host (bob operates on the live `~/bob` vault):

```bash
bob albums list
bob albums create --help
ls ~/bob/sase_blog.md
```

Confirm `bob albums create` exists, `sase_blog_zettel` is not already an album, and
`~/bob/sase_blog_zettel.md` / `~/bob/sase_blog_zettel/` / `~/bob/sase_blog_zettel.base`
do not exist. If `bob albums create` is missing from the installed bob, stop and report
— do not build from source or hand-write the album file.

The vault is live and synced by `bob-vault-sync`; mutate it only through `bob` commands
plus the one documented frontmatter edit in Step 3. Never run git commands in `~/bob`.

## Step 2 — Create The Album

Dry-run first, inspect the printed diffs, then run for real (adjust flag spellings only
if `--help` shows the research's flags have drifted):

```bash
bob albums create sase_blog sase_blog_zettel -n zettel -e 📇 -t 'SASE Blog Zettel' \
  -s 'Topic-sized pieces of my SASE posts, one claim per record, in my own voice.' -d
# inspect, then:
bob albums create sase_blog sase_blog_zettel -n zettel -e 📇 -t 'SASE Blog Zettel' \
  -s 'Topic-sized pieces of my SASE posts, one claim per record, in my own voice.'
```

Notes from the research (sandbox-verified): pass `-t` or the title renders as "Sase blog
zettel"; the emoji must be 📇 (🧩 is already used for sub-project rows; ✍️ was rejected
as generic). The parent is the **project** `sase_blog` — valid, since album parents may
be areas or projects.

## Step 3 — Add The Record Schema

`bob albums create` writes only the basics (`noun`, `label`); extend the album
frontmatter in `~/bob/sase_blog_zettel.md` by hand (this is the documented workflow —
create's success output prints a paste-ready `fields:` example). Target schema, already
validated by the research with `bob albums check` (0 errors):

```yaml
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
    part:
      values:
        hook: 🪝 Hook
        why: 🙋 Why I use it
        what: 🧱 What it is
        why_not: 🛑 Why not
        close: 🏁 Close
```

Also set `index_facets: [part, claim]` (top-level album key) so each generated index row
reads `part · claim` and the `[!records]` callout reads as the post's argument. Keep
whatever `summary`, `emoji`, and generated body sections `create` wrote. Do **not** add
`order`, `section`, `wave`, `target_words`, a `status` default (bob disallows one; the
research relies on capture always supplying it), or devil/angel CARD fields (R5; CARD
material is prose for `## Notes`).

## Step 4 — Validate The Album

```bash
bob albums check
bob albums list
bob albums sync
bob records check zettel
```

All checks must pass with zero errors/findings for the new album. Confirm:

- `~/bob/sase_blog_zettel.md` has the full schema, the intro teaching
  `Name @sase_blog_zettel/`, and the empty-state generated records index.
- `~/bob/sase_blog.md` now shows the machine-owned `## Albums` catalog row for 📇 SASE
  Blog Zettel (written by bob, not by hand).

Then verify the guide's capture examples work without writing: run each capture form
that will appear in the guide through `bob capture --dry-run` (add `--no-clip` if a clip
directive interferes) and confirm the planner previews the expected record
creation/update. A real throwaway record is allowed only if a dry run cannot exercise a
path, and must be deleted through `bob records`/documented means afterward, leaving
`bob records check zettel` clean.

> [!decision] seed_records = stubs After validation, create one 🌱 record per current
> top-level topic in `~/bob/sase_blog_contents.md` (the ~15 numbered headings:
> why-I-use, gates/plans/epics, beads/sidecars, agent CLIs,
> skills/memory/provider-interface, agent presentation, VCS/workspace ops,
> macros/control-flow, project management, routines/jobs, AI-slop definition, future
> work, hook, close — use current sase names: **Macros**, not Xprompts). Use
> `bob records new zettel '<Title>' ...` (check `--help` for field-set flags) with
> `status=seed`, the matching `part` value, and
> `claim='TODO — state the claim in one sentence'`. Rerun `bob records check zettel`
> afterward. The guide must then tell Bryan his first S0 act is replacing every TODO
> claim or cutting the record.

## Step 5 — Author The Usage Guide In The Research Sidecar

Open the sidecar with the `/sase_repo` skill
(`sase repo open bob-cli--research -r "..."`) and write the guide to
`202611/sase_blog_zettel_album_guide.md` (flat monthly note, matching existing guides
like `bob_gmail_everyday_guide`). Write it for Bryan in second person, practical and
compact (roughly 150–300 lines), distilling the consolidated research — do not restate
its evidence or panel debate. Required content, in this order:

1. **What this album is (and is not).** A workbench of claim-sized drafts; the article
   `sase_blog_contents.md` stays the manuscript, spine, and only authority for order and
   transitions; the album is never a second outline; one record per topic/claim, not per
   section (R1, R5). "Mostly re-arranging" is an illusion: budget the rewrite pass (R2).
2. **The schema, field by field.** `status` lifecycle with exit criteria: 🌱 title +
   claim → ✍️ ≥150 unedited words in `## Draft` → ✅ revised + current-names/facts check
   against sase HEAD + links/images resolved (R8) → 📌 pasted into the spine, after
   which **only the article is edited** (one-way fork, no back-syncing); ⏭️ later =
   future post; ✂️ cut = dropped but kept. `claim` required — if you cannot state the
   claim, it is not a topic yet; capture it as a note on an existing record. `part` is a
   display facet only.
3. **Record body convention.** Reproduce the research's skeleton: bob writes `# Title`
   and `## Notes`; only `## Draft` ever ships; capture appends dated bullets to
   `## Notes`; never a checkbox in a record — actions live as tasks in `sase_blog.md`
   linking to the record.
4. **Capture cheatsheet.** Dry-run-verified forms: create a zettel
   (`<Title> @sase_blog_zettel/ ...` with `status` and `claim`), append an idea to one
   (`idea @sase_blog_zettel/<stem>`), and the equivalent `bob records new zettel` /
   `bob records update zettel <stem>` commands. Include one paste-ready **batch-capture
   template** seeded with the current outline's topics (current sase names) and blank
   claim slots for Bryan to fill — labelled as candidate coverage only, since the
   outline is not final.
5. **Routing rule (R6).** The one-home-per-kind-of-material table: idea/quote/
   screenshot/proof → that record's `## Notes`; prose in Bryan's voice → that record's
   `## Draft`; order/headings/transitions/intro/promise → `sase_blog_contents.md`;
   post-wide style and production rules → `sase_blog.md ## Requirements`; work items →
   tasks in `sase_blog.md` linked to records. Plus: triage the ~44 existing Requirements
   bullets lazily, per topic, never as an up-front migration.
6. **Revised S0 checklist.** ≤ 1 🍅 total, then stop: album exists (done by this plan);
   seed one zettel per topic with a real claim via one batch capture; move the four Wave
   1 CARD bullets into their records' `## Notes` and point each `zd-*` task at its
   record (exit criteria unchanged, now targeting `## Draft`); add spine links under the
   article headings; write the promise sentence.
7. **Assembly and publishing from the spine.** Link records under article headings
   (`→ [[sase_blog_zettel/<stem>|Title]]`); reorder by moving headings and link lines
   (`bob articles` renumbers); write transitions directly in the spine; **assemble
   early** after Wave 1 instead of waiting for every record (R4); publish by copying
   into `docs/blog/posts/<slug>.md` in sase-org/sase, converting Obsidian syntax
   (wikilinks, callouts, embeds, footnotes), and running `mkdocs build --strict`. No
   exporter script until two posts ship this way.
8. **Fallback triggers (R7).** Article-only writing is fully viable: fall back if setup
   ever costs more than 1 🍅, or if after Wave 1 most pieces feel inseparable and
   file-switching hurts more than it helps; either exit is a successful experiment.
9. **Open questions for Bryan.** Does this post replace or accompany the July launch
   post (drives ⏭️/✂️)? Keep `claim` required, or relax to optional if it blocks capture
   flow? The "zettel" name over-promises slightly — keep or rename.
10. **Pointer** to the consolidated research report
    (`202611/sase_blog_drafting_in_records/`) for the full rationale, and a note that
    the `1 zettels` singular-tally cosmetic bug is known and tracked on `bob-cli-k3`.

After writing the guide, link it to its source:

```bash
sase artifact link add research:202611/sase_blog_zettel_album_guide.md derives-from research:202611/sase_blog_drafting_in_records/sase_blog_drafting_in_records.md "guide operationalizes the consolidated research recommendation"
```

The sidecar commit itself is handled by the normal `/sase_final` repository-obligation
flow; do not commit it any other way.

## Acceptance Criteria

- `bob albums list` shows the 📇 `sase_blog_zettel` album; `bob albums check` and
  `bob records check zettel` exit clean.
- The album note carries the exact Step 3 schema (including `index_facets` and
  `dated_notes: true`, excluding every rejected field) and a generated records index.
- `~/bob/sase_blog.md` shows the bob-written `## Albums` catalog entry.
- Every capture/CLI example in the guide was verified via `bob capture --dry-run` or the
  matching `bob records` command's dry-run/preview on this vault.
- The guide exists at `202611/sase_blog_zettel_album_guide.md` in the sidecar with all
  ten content sections, and the `derives-from` artifact link is recorded.
- `sase_blog_contents.md` is byte-identical to before the run; `sase_blog.md` differs
  only by machine-owned bob writes.
- Under `seed_records = stubs` only: one 🌱 record per current outline topic with a
  placeholder claim, and checks still clean.
