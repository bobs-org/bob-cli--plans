---
tier: tale
title: De-zorg the area notes
goal:
  Every area note that ever held tasks (plus kelly and meet) matches the newer area-note
  format and carries no zorg frontmatter, so Gmail watching sees job.md with no code
  change.
size: medium
decisions:
  legacy_home:
    ask:
      Should zorg-era legacy sections move out of the area notes into new legacy_<area>
      notebooks?
    choices:
      notebooks:
        Move them verbatim into 8 legacy_<area> notebooks; rewrite ~320 links in ~136
        files
      in_place:
        Leave them at the bottom of each area note; no new notebooks, no moved-link
        rewrites
    default: notebooks
    why:
      Repeats your 2026-11-02 abby → legacy_dad move; area notes keep only live content
    answer: notebooks
  converter_keys:
    ask:
      Should the converter's other keys (id, aliases, tags, inbox's status and area) go
      too?
    choices:
      drop:
        Remove them from 5 notes; the Meetings alias and frontmatter-tag search hits go
        away
      keep: Remove only the 7 zorg keys; the converter keys stay
    default: drop
    why:
      No newer area note has them, and nothing in bob or the vault reads them on area
      notes
    answer: drop
  restore_job_legacy:
    ask:
      Should legacy_job.md get back the two zorg-era job entries cut from job.md on
      2026-10-12?
    choices:
      restore:
        Copy them from git history into legacy_job.md, so 5 dangling job.md links
        resolve
      skip: legacy_job.md stays a one-line stub, and those 5 links keep dangling
    default: restore
    why: legacy_job.md was made a minute before the cut to hold those notes, and says so
    answer: restore
proposed_by: bbugyi200.athena.1ax
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 09:45:22
status: wip
---

# De-zorg the area notes

## Problem

The `bob gmail` vault walk skips every note whose frontmatter has
`generated_from_zorg: true`. Twelve live area notes still carry that flag, including
`job.md`. As a result, job.md's mail tasks are never watched, and Dan Dowd's 2026-11-09
reply never lifts its `#awaiting_reply` hold. The rejected plan
`202611/gmail_watch_zorg_area_notes.md` would have changed the code instead.

This plan changes the vault instead. It makes the old zorg-converted area notes look
like Bryan's newer area notes, and it removes every zorg key from their frontmatter.
Once the flag is gone, the walk sees these notes with no code change.

This is vault-only work in `~/bob` (BOB_DIR). There are no bob-cli or bob-plugins source
changes.

## Safety audit (done while planning)

Every check below was run against the live vault and the current repos. Each one passed.

- **Readers of the zorg keys.** The only code that reads any `zorg_*` or
  `generated_from_zorg` key is `src/native/gmail/vault_walk.rs`, which skips flagged
  notes. That skip is the bug this plan fixes.
  - bob-plugins: no matches.
  - chezmoi: no matches.
  - Vault `.base` files, `.obsidian` config, and Dataview blocks: no matches.
    `podcasts.base` reads `zorg_id`, which is a different key that no area note has.
  - `periods migrate` keeps `zorg_*` keys only on year notes.
  - `ref` zorg coverage counts `status::` records, and these notes have none.
- **What the Gmail walk does once it sees these notes.**
  - The watch scan is read-only. It acts only on open `#task` rows that link a saved
    mail note.
  - `bob gmail link -a` and its list footer would also sweep these notes. They contain
    no `mail.google.com` URLs, so there is nothing for them to rewrite.
  - Expected effect: the next Mac `bob gmail sync` watches job.md's four mail tasks,
    folds the 2026-11-09 reply, and lifts that hold.
- **Converter keys.**
  - Frontmatter `status` is read only for project notes (bob-cli `ProjectStatus`, plugin
    project status). On area notes it is ignored.
  - `aliases` is read only for notebook titles.
  - Nothing reads `id`, `tags`, or `area` from area notes.
- **`theme:` → `task_type:`.** Both bob-cli (`pomodoro_homes/facts.rs`) and bob-plugins
  (`279-plugin-task-theme-home-facts.js`) prefer `task_type:` and fall back to `theme:`.
  `bob types migrate` performs the same rename. Same value, so behaviour does not
  change.
- **Tasks are untouched.**
  - No checkbox line, task child, `## Tasks` section, `[id::]`, or Depends-On line
    changes.
  - No open task, Depends-On line, or `[dependsOn::]` field points at a legacy block.
  - No checkbox row exists outside `## Tasks` in any in-scope note, so legacy content
    can become a notebook without a `notebook_checkbox` error.
- **Vault-sync guards.**
  - The clobber guard needs a displaced note plus a foreign identity: a `created:`
    value, a `^prj` line, or at least 3 new task IDs shared with another note. inbox.md,
    meet.md, kelly.md, and fun.md will be displaced, but none of them gains a `created:`
    value, a `^prj` line, or task IDs. So never add `created:` to an existing note.
  - The lost-task guard watches task block IDs only, and no task moves.
- **Mac concurrency.** The edits touch frontmatter, preambles, and trailing legacy
  sections. Bryan's Mac edits land in `## Tasks`. Git's three-way merge combines both,
  and any true conflict keeps a copy under `_conflicts/`.
- **Reversibility.** Everything lands in one single-purpose commit, so `git revert` is
  safe. It is not a mixed vault commit.

## Scope

All files are at the vault root.

| File           | Zorg keys | Change                                                                    |
| -------------- | --------- | ------------------------------------------------------------------------- |
| abby.md        | yes       | Frontmatter only, plus repairing 39 `[[abby#^z-…]]` links into legacy_dad |
| body.md        | yes       | Frontmatter, `Related:` line, legacy → legacy_body                        |
| cash.md        | yes       | Frontmatter, preamble, `^balance` → `## Notes`, one entry → legacy_cash   |
| dev.md         | yes       | Frontmatter (`theme`), `Related:` line, legacy → legacy_dev               |
| fun.md         | yes       | Frontmatter, `Related:` line, legacy → legacy_fun                         |
| gkeep_inbox.md | yes       | Frontmatter only; legacy_gkeep_notes.md becomes a typed notebook          |
| gtd.md         | yes       | Frontmatter, `Related:` line, three legacy sections → legacy_gtd          |
| inbox.md       | yes       | Frontmatter, preamble, Migrated Zorg Inbox → legacy_inbox                 |
| job.md         | yes       | Frontmatter only (`theme`); legacy_job.md becomes a typed notebook        |
| kelly.md       | yes       | Frontmatter, description, `## Tasks`, three entries → legacy_kelly        |
| love.md        | yes       | Frontmatter (`theme`); drop the empty `- ` under Future Work              |
| meet.md        | yes       | Frontmatter, `Related:` line, dated sections → legacy_meet                |
| gtd_daily.md   | no        | Frontmatter (`theme`, converter keys)                                     |
| mac_inbox.md   | no        | Frontmatter (converter keys); stale `inbox.zo` description                |
| recur.md       | no        | `# Kelly's Salary` → `## Kelly's Salary` (a stray H1 between two H2s)     |
| gmail_inbox.md | no        | Audited only; already in the newer format                                 |

Out of scope:

- family.md and vacation.md: they never held tasks and have no zorg keys.
- bob.md, sase.md, and needs_attn_tasks.md: these were once area notes but are now
  project notes.
- The other ~3,500 zorg-flagged notes. One of them, obsidian_ref.md, is a ref note with
  one task.

## Target format

This follows the newer area notes: job.md, abby.md, vacation.md, family.md,
gmail_inbox.md, and the June de-zorg of recur.md.

### Frontmatter

1. Remove these seven keys: `generated_from_zorg`, `zorg_source`, `zorg_source_abs`,
   `zorg_source_sha256`, `zorg_source_mtime`, `zorg_converter`, and
   `zorg_converter_version`.
2. Rename `theme:` to `task_type:`, keeping the value bytes:
   - dev: `BUILD`
   - job: `WRITE`
   - love: `CHORE`
   - gtd_daily: `PLAN`
3. Under `converter_keys = drop`, also remove these keys:
   - job: `aliases`, `tags`, `id`
   - inbox: `status`, `area`, `aliases`, `tags`, `id`
   - meet: `aliases`, `tags`, `id`
   - mac_inbox: `id`, `aliases: []`, `tags: []`
   - gtd_daily: `aliases: []`, `id`, `tags: []`
4. Keep every other key byte-for-byte: `parent`, `type`, `color`, `task_type`,
   `offclock`, `offclock_types`, `ready_cap`, `project_name_aliases`, and `done_tasks`.
   Order them `parent`, `type`, then the rest in their current relative order. This only
   matters for gkeep_inbox, whose flag currently comes first.
5. Never add `created:` (see the clobber guard above).

### Body

The body runs in this order:

1. The description paragraph(s).
2. One `Related:` line, as in job.md.
3. `## Notebooks`. Its content belongs to bob; `bob notebooks sync` places it before
   `## Tasks`.
4. `## Tasks`. Bob's status line and subsections stay byte-identical.
5. Optional live sections such as `## Future Work` and body.md's `## Notes`.

No zorg-era content stays in the area note. Keep existing H1s (inbox and mac_inbox): the
mac_inbox status-line anchors use `# Mac Inbox`.

## Per-file edits

### Legacy content and Related lines

The `Related:` lines below assume `legacy_home = notebooks`. Under `in_place`, omit
`[[legacy_<area>]]` from each one.

- **body.md**
  - Add `Related: [[dentist]] [[gym]] [[outlive]] [[body_ticks]] [[legacy_body]]` after
    the two preamble paragraphs.
  - `## Notes` stays.
  - The `## Legacy Notes` entries go to legacy_body.
- **cash.md**
  - Replace the zorg title line `#cash | Money Money Mooooney` with:
    ``Money Money Mooooney! [[project]]s / [[area]]s about money should link to this note file as their `parent`.``
  - Replace the "Related notes:" list with
    `Related: [[bank]] [[old_recur]] [[legacy_cash]]`. `[[taxes]]` is already in the
    catalog.
  - Delete `## Area Notes`; it holds only an empty `- `.
  - In `## Legacy Notes`, the `z-240625-06` entry (sell GSUs) goes to legacy_cash.
  - The live `^balance` block (recur.md's tasks link `[[cash#^balance]]`) moves,
    byte-identical, under a new `## Notes` heading after `## Tasks`. This follows the
    body.md precedent.
- **dev.md**
  - Add
    `Related: [[boltons]] [[genai]] [[job]] [[logging]] [[phone]] [[dev_ref]] [[legacy_dev]]`.
    This drops ai, dev_ideas, dev_tools, linux, and networking, which the catalog
    already lists.
  - `## Future Work` stays.
  - The zorg entries in `## Legacy Notes` go to legacy_dev.
- **fun.md**
  - Add `Related: [[music]] [[surf]] [[legacy_fun]]`.
  - The legacy entries go to legacy_fun.
  - Make sure a blank line separates the last task block from whatever follows.
- **gtd.md**
  - Add
    `Related: [[prj_gtd]] [[now_gtd]] [[project_checklists]] [[gtd_ref]] [[pomodoro]] [[legacy_gtd]]`.
    gtd_ideas is already in the catalog.
  - The `## Legacy Notes` entries, `## HABITS TO TRACK`, and `## @FOOTNOTE` go to
    legacy_gtd, keeping their `##` headings.
- **inbox.md**
  - Under `# Inbox`, write:
    `Root of the inbox files: [[mac_inbox]], [[gkeep_inbox]], and [[gmail_inbox]] hold new tasks awaiting triage.`
    Follow it with `Related: [[inbox_overflow]] [[legacy_inbox]]`.
  - The zorg-era `- Now: [[now_dev]] …` bullet and all of `## Migrated Zorg Inbox` go to
    legacy_inbox.
  - Keep `## Tasks`.
- **kelly.md**
  - Add the description
    `Tasks and notes about Kelly (see [[legacy_kelly]] for legacy notes).`
  - Delete `## Related notes` and its `[[kellys_salary_info]]` bullet; the catalog
    already lists that note.
  - The three zorg entries go to legacy_kelly.
  - Append an empty `## Tasks` after the catalog, as vacation.md and meet.md have.
- **meet.md**
  - Replace the three-line "Related notes:" list with
    `Related: [[the_monthly]] [[all_hands]] [[recur_meet]] [[legacy_meet]]`. The
    per-link captions are dropped.
  - Keep `## Tasks`.
  - Every dated `## 2024-…`/`## 2025-…` section goes to legacy_meet verbatim, including
    its `====` divider lines.
- **mac_inbox.md**
  - Replace the description with:
    ``Default destination for `bob capture` and Bob Mac Capture; triage these tasks in Obsidian.``
  - Keep `# Mac Inbox` and `## Future Work`.
- **legacy_gkeep_notes.md and legacy_job.md** (both decisions branches)
  - Add `type: "[[notebook]]"` and a one-line `summary:`.
  - Write `parent` as `"[[gkeep_inbox]]"` / `"[[job]]"`.
  - Keep their `created:` values.
  - Add an H1 (`# Legacy gkeep notes`, `# Legacy job notes`) right after the
    frontmatter. Their existing prose stays below it.
  - Under `restore_job_legacy = restore`, append the job.md `## Reference` and
    `## Meetings` blocks verbatim from commit `85fce7e7c`
    (`git -C ~/bob show 85fce7e7c:job.md`). These are the `^z-241210-0m` entry and the
    2025-10-20 Meta call with `^job_meet_251020` and `^z-251020-0k`.

### New legacy notebooks (`legacy_home = notebooks`)

There are eight new notebooks: legacy_body, legacy_cash, legacy_dev, legacy_fun,
legacy_gtd, legacy_inbox, legacy_kelly, and legacy_meet. None of these names exists yet.
The naming follows Bryan's legacy_dad, legacy_job, and legacy_gkeep_notes.

Each file looks like this:

```markdown
---
parent: "[[dev]]"
template: "[[new_notebook]]"
type: "[[notebook]]"
summary: Zorg-era dev notes, moved here from dev.md on <run date>.
created: <run timestamp, ISO 8601 with offset, distinct per file>
---

# Legacy dev notes

Legacy [[dev]] notes were moved here on <run date>.

<moved lines, byte-for-byte, in their original order>
```

Do not add `zorg_*` keys. Provenance lives in `summary`, as it does in legacy_dad.

> [!decision] legacy_home = in_place
>
> - Create no notebooks and run no moved-block link rewrites.
> - Each note's legacy section stays byte-for-byte where it is.
> - Preamble zorg leftovers (inbox's `- Now:` bullet; kelly's three entries under a
>   renamed `## Legacy Notes`) move to the top of that note's legacy section.
> - The frontmatter edits, descriptions, `Related:` lines (without `[[legacy_<area>]]`),
>   cash's `## Notes` move, and the link repairs below still happen.

## Link rewrites

Write one migration script that both rewrites links and checks invariants. Keep it in a
scratch directory outside the vault and outside any repo. Run it with `python3 -I`.

Scan every `.md` file in the vault except `.git/`, `.obsidian/`, `.trash/`, `.bob/`, and
`_conflicts/`. `.bob/` is skipped because ref sources are write-once. `done/`, the dated
folders, and `_generated/` are included, as Obsidian's own rename did on 2026-11-02.

Rewrite rules:

1. **Moved anchors** (`notebooks` only). For each area A with a new notebook L, M(A) is
   every trailing `^id` on a moved line, including heading lines such as meet's
   `^20240722`, plus every `ID::name` / `LID::name` token on a moved line.
   - Rewrite `[[A#^id…`, `[[A.md#^id…`, `![[A#^id…`, and caret-less `[[A#id…` (zorg
     heading links such as `[[meet#20240722]]`) whenever `id ∈ M(A)`.
   - Replace only the note name with L. Keep `.md`, `!`, `|alias`, and every other byte.
2. **Same-note links.**
   - Inside L, a `[[#^id]]` whose target stayed in A becomes `[[A#^id]]`.
   - Inside A, a `[[#^id]]` whose target moved becomes `[[L#^id]]`.
3. **Repairs** (both branches). A `[[A#^id…` link whose id is not defined in A but is
   defined in A's existing legacy note is rewritten to that note:
   - abby → legacy_dad: 39 links over 5 ids, including 5 inside legacy_dad.md itself.
   - gkeep_inbox → legacy_gkeep_notes: 1 link.
   - Under `restore`, job → legacy_job: 5 links.

Planning measured about 320 matching links in about 136 files, most of them inbox's 145
and body's 50. Print the per-note counts in the dry run.

## Procedure

1. **Baseline.**
   - Run `bob vault-sync`. Then check `bob vault-sync status --json`: no error, no
     pending clobbers.
   - Check that `git -C ~/bob status --porcelain` shows no in-scope path.
   - Save these outputs for comparison after the commit:
     - `bob task reconcile --dry-run`
     - `bob notebooks check -f json`
     - `bob gmail doctor` (athena reports `watching 2 · 0 awaiting` today)
     - `bob types migrate -d`
2. **Dry run.**
   - The script reads every input file and records each file's SHA-256.
   - It builds all outputs in memory and writes a unified diff of every changed or new
     file to the scratch directory.
   - It runs the invariants below.
   - Review the diff by eye, especially the moved/kept line classification for cash.md,
     body.md, and meet.md.
3. **Apply under the maintenance lock.**
   - `lock="${BOB_VAULT_SYNC_LOCK_FILE:-${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock}"`, then
     `flock -w 120 "$lock" bash -c '<apply> && git -C ~/bob add -- <paths> && git -C ~/bob commit -m "vault: de-zorg area notes and move legacy notes into notebooks" -- <paths>'`.
   - The apply step re-hashes every input and aborts without writing if any file changed
     since the dry run. If it aborts, rerun step 2.
   - It writes each file atomically (temp file plus rename) and reruns the invariants on
     the written bytes.
   - Commit exactly the touched paths. Follow `~/bob/AGENTS.md`: use plain `git add` and
     `git commit`, never `/sase_git_commit`.
   - Holding the lock makes athena's `bob-vault-sync.service` skip its cycles.
4. **Publish.**
   - Run `bob vault-sync`.
   - Run `bob notebooks sync` (`notebooks` branch). It inserts the new `## Notebooks`
     catalogs before `## Tasks` and makes its own scoped commit.
   - Then confirm that `bob notebooks sync -d` is a no-op.
5. **Settle.**
   - After about two minutes (several Mac LaunchAgent cycles), run `bob vault-sync`
     again.
   - Check that `bob vault-sync status --json` shows no new `clobbers`, `task_losses`,
     or conflict entries.
   - Check that `_conflicts/sync_conflicts.md` has no new entry that names an in-scope
     file.

## Invariants (the script must assert these)

1. **Tasks and frontmatter.**
   - Every checkbox line, and every line in each `## Tasks` section, is byte-identical
     before and after.
   - The vault-wide `#task` line count is unchanged.
   - Every kept frontmatter key keeps its value bytes.
2. **Nothing lost.** Every non-blank line removed from an area note appears in its
   legacy notebook (or in cash.md's `## Notes`), apart from link rewrites. The only
   exceptions are the lines this plan deletes on purpose:
   - zorg and converter keys
   - "Related notes:" headers, list items, and captions
   - cash's `#cash | …` title line and `## Area Notes`
   - kelly's `## Related notes` heading and its salary bullet
   - love's empty `- `
   - mac_inbox's old description
3. **Block IDs.** For each area, the trailing `^id` definitions in A before equal those
   in A after, plus those in its legacy notebook, minus that notebook's pre-existing
   IDs.
4. **Links.** No link into an in-scope note or legacy notebook dangles after the run
   unless it already dangled before. The repairs reduce the dangling count by at
   least 40.
5. **Notebooks.** `bob notebooks check` reports no `notebook_checkbox`,
   `notebook_parent_*`, `notebook_cycle`, `notebook_child_kind`, or
   `notebook_not_at_root` error for any new or retyped notebook.
   - `notebook_duplicate_block_id` errors are acceptable only when every flagged id has
     exactly one trailing definition in that file. The extra hits come from
     `extract_block_ids` counting carets inside wikilinks, a known bug already recorded
     on epic bob-cli-j8 (note #3) and in bob-cli-j8.9's proposed follow-up. Do not fix
     it here.
   - `notebook_shape` warnings, such as meet's `====` dividers, are acceptable.

## Verification

- `rg -n '^(generated_from_zorg|zorg_)' ~/bob/{abby,body,cash,dev,fun,gkeep_inbox,gtd,inbox,job,kelly,love,meet,gtd_daily,mac_inbox,recur,gmail_inbox}.md`
  prints nothing.
- `bob types migrate -d` no longer lists dev.md, gtd_daily.md, job.md, or love.md. It
  still lists dev_plex_hack_patch, job_2026, and sase_blog, and one sase.md field; those
  are out of scope.
- `bob gmail doctor` on athena now counts job.md's mail tasks: `watching` rises above 2,
  and `awaiting` is at least 2.
  - Optionally, run `ssh mac 'bob gmail sync -d -f json'`. It should show watch rows for
    job.md's threads, with Dan Dowd's thread ready to lift.
  - Do not run a live Gmail sync; the Mac's scheduled sync will do it.
- `bob task reconcile --dry-run` matches the baseline.
- Each new notebook appears in exactly one catalog (its area's). A second
  `bob notebooks sync -d` is a no-op.
- `bob vault-sync status --json` is clean after the settle step.

## Rollback

1. `git -C ~/bob revert --no-edit <migration commit>`, then the catalog commit, made
   under the lock as in step 3.
2. Then run `bob notebooks sync` and `bob vault-sync`. Both commits are single-purpose,
   so reverting them is safe.

## Out of scope

- Code changes. The Gmail walk keeps skipping flagged notes; once this lands, no live
  area note is flagged.
- The notebook check's caret bug (bob-cli-j8 owns it).
- Converting zorg syntax inside the moved legacy content: it stays verbatim, as in
  legacy_dad.
- Other dangling links into job.md that point at tasks deleted before this work
  (`^call-anthem`, `^ray-lyles`, `^update-linkedin`, and others).
- The other ~3,500 zorg-flagged non-area notes.
