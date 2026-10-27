---
tier: epic
title: Add the WRITE theme and split JOB blocks by work kind
goal: 'Every past and open JOB Pomodoro carries the work-kind theme its content supports
  (READ, WRITE, CHORE, PLAN, or BUILD) in one verified, revertable vault commit. WRITE
  is a live, pinned theme that shows in Recent. JOB is hidden. `bob themes list` reports
  no false rename reappearances.

  '
phases:
- id: split_cmd
  title: '`bob themes split` and the reappearance fix'
  depends_on: []
  size: medium
  description: 'split_cmd: add the template and map-driven per-entry theme split on
    the shared vault-rewrite transaction, and stop renamed_reappeared from reporting
    re-adopted names, with tests and docs.'
- id: vault_migrate
  title: Split the live JOB blocks and add WRITE
  depends_on:
  - split_cmd
  size: small
  description: 'vault_migrate: hide JOB, split its 89 blocks with the planned mapping
    in one vault commit, pin WRITE (and the standing set), and verify list, stats,
    and sync.'
decisions:
  planning_blocks:
    ask: Should the 10 JOB blocks spent organizing the search and the blog project
      become PLAN?
    choices:
      plan: PLAN, per the codebook; about 2h moves from the job and sase_blog homes
        to the review bucket
      chore: CHORE, so those minutes stay in the job and sase_blog homes
    default: plan
    why: The codebook routes "a choice" to PLAN, and the research accepts losing the
      home
    answer: plan
  pin_standing:
    ask: Should the migration also pin BUILD yellow, READ green, FIX blue, and CHORE
      aqua?
    default: true
    why: Research step 7 pins the whole set so no area note or rename can shift a
      color
    answer: true
proposed_by: bbugyi200.apollo.ar.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 17:11:09
status: wip
bead_id: bob-cli-db
---

- **PROMPT:** [prompts/202610/write_theme_and_job_split.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/write_theme_and_job_split.md)
- **BEAD:** [bob-cli-db](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-db/README.md)

# Plan: Add the WRITE theme and split JOB blocks by work kind

## Context

The research report `work_kind_pomodoro_theme_set` (bob-cli research sidecar,
2026-10-27) recommends six work-kind themes: PLAN (exempt), BUILD, FIX, READ, WRITE, and
CHORE. Its JOB row says JOB "becomes READ, WRITE, CHORE, or BUILD, depending on the
block". It says to hide JOB and not to rename or merge it, because a rename would carry
89 mixed JOB tails into one new name. The user asked to add WRITE and to migrate the JOB
blocks as recommended. This plan does that block by block, so each entry gets its own
kind. That is the only migration that avoids the mixed history the research warned
about. It departs from the research's "start WRITE with no history" step on purpose:
WRITE receives only the 17 blocks that were actually prose for people, not BLOG or JOB
history in bulk.

What already exists and what this plan builds on:

- GTD → PLAN and HOME → CHORE are done. `plan.exempt` is `[PLAN]` and the root `plan.md`
  exists (vault commits `d3ffa8d1`, `4b07dc29`; bob-cli `f6287847`).
- Live catalog (2026-10-27): JOB has 89 entries on 20 days (Oct 7–27), aqua from the
  `job` area. WRITE has zero entries. Its last journal event is the rename `WRITE → JOB`
  (2026-10-27 15:37, kellys-macbook-pro-local).
- `bob themes rename` is global by key, so it cannot split one theme across several. No
  bob command rewrites selected entries. Hand edits would bypass the lock, the pre-sync,
  the stale check, verification, and the single revertable commit that every other theme
  rewrite gets.
- `crate::native::vault_rewrite::run` (`src/native/vault_rewrite/mod.rs`) is the shared
  rewrite transaction that rename already uses (lock, pre-sync, plan, snapshot, stale
  check, atomic writes, post-write hook, verify-or-restore, one commit, post-sync).
  `capture_pomodoros::PomodoroRef` (`LINE:DIGEST`, where the digest hashes the entry
  line) and `PomodoroScan::by_ref` already give each entry a stale-safe identity.

There are two bugs in `renamed_reappeared` (`src/native/pomodoro_themes/rename.rs`):

- **Today's false report.** It reports every rename-source key that still has catalog
  entries, using only the latest rename _from_ that key. Today it reports
  `JOB renamed_to WRITE`, even though `WRITE → JOB` came later and made JOB the live
  name again.
- **WRITE would be reported next.** After the split, WRITE has entries again, so it
  would be reported as `renamed_to JOB`, even though reusing it is deliberate.

Phase `split_cmd` adds `bob themes split` and fixes the reappearance rule. Phase
`vault_migrate` uses them on the live vault.

## Phase split_cmd: `bob themes split` and the reappearance fix

### Command shape

```text
bob themes split OLD (-m/--map FILE | -t/--template) [-b/--bob-dir DIR] [-d/--dry-run]
                     [-f/--format human|json] [-o/--offline]
```

- Give it its own module, `src/native/pomodoro_themes/split.rs`, next to `rename.rs`.
  Reuse rename's helpers rather than copying them: `rewrite_name`, `replace_name_tail`,
  the bob-dir/format/offline arg builders, and the failure printing. Widen their
  visibility to `pub(super)` as needed.
- Wire it up wherever `themes rename` is wired: `src/runner.rs` / `src/native.rs`
  dispatch, `src/native/pomodoro_themes/mod.rs` exports, and
  `src/native/completion/tree.rs`. Add it to the `bob themes` members in help.
- The `bob themes` help text and about string become "List, hide, unhide, color, rename,
  and split Pomodoro themes". Add a group example: `bob themes split JOB -t > job.tsv`
  followed by `bob themes split JOB -m job.tsv -d`.
- Help, flags, and order follow the CLI rules memory: every long option has a short
  alias, options are alphabetical, members are alphabetical (split goes between rename
  and unhide). Give split a long_about and after_help as complete as rename's, with
  examples, the map format, exit codes, and a pointer to `docs/pomodoro-themes.md`.
- Exactly one of `-m` and `-t` is required. Otherwise it is a clap usage error. `-m -`
  reads the map from stdin.
- OLD must be a known theme (`unknown_theme` otherwise). Like rename, the preview and
  the template always rescan from source.

### Template mode (`-t`)

The template is read-only and never takes the lock. It lists every entry in canonical
daily notes (`scan::list_daily_notes`, past, today, and future) where some `+` component
normalizes to OLD's key. `_templates/daily.md` is out of scope for split because it has
no date. Order the entries by date, then line.

- **Human/default output is TSV** that the map parser accepts unchanged.
  - It starts with `#` comment lines naming the columns and the fill-in rule.
  - Each row is `DATE<TAB>REF<TAB>THEME<TAB>NOTE`:
    - DATE is `YYYY-MM-DD`.
    - REF is the entry's `PomodoroRef` (`LINE:DIGEST`).
    - THEME is prefilled with OLD's display name, so an untouched row is a keep.
    - NOTE is `[x]`/`[ ]`, the time range or `()`, the full current name, and the Task
      Link targets from the entry's child lines (for example
      `done/job_done#^apply-warp`), joined with `, `.
  - Plain-text children with no link appear in NOTE as their first 60 characters.
- **`-f json`** emits
  `{schema_version: 1, ok: true, mode: "template", theme: {key, name}, entries: [{date, path, ref, line, state, time_range, name, links, notes}]}`.

### Map mode (`-m FILE`)

**Parsing.** Map rows are tab-separated: `DATE<TAB>REF<TAB>THEME[<TAB>NOTE]`. Blank
lines and lines starting with `#` are skipped, and NOTE is ignored. Refuse with
`invalid_map`, citing the map line number, when:

- a row has fewer than three columns,
- the date is malformed or has no canonical daily note,
- the ref is malformed, or
- two rows resolve to the same entry.

THEME must be a valid single theme. A blank THEME or one containing `+` is
`invalid_name`, cited the same way.

**Resolving rows.** Resolve each row with `capture_pomodoros::scan` and
`PomodoroScan::by_ref` on that day's note. If the lookup is `Stale` or `Ambiguous`, or
the resolved entry has no OLD component, refuse with the new code `stale_entry`. Its
message names the date and ref, and its hint says "regenerate with -t".

**The rewrite.** A row whose THEME key equals OLD's key is a keep (no change). Otherwise
replace the OLD component with THEME's trimmed spelling, using `rewrite_name`. That
preserves the `+` separators and collapses a resulting duplicate (`JOB + BUILD` with
THEME BUILD becomes `BUILD`). Splice the new tail in with `replace_name_tail`. Every
other byte stays the same.

- If every row is a keep, refuse with `unchanged` and write nothing.
- OLD entries with no row are left alone and counted as `unmapped`.
- THEME may be a theme that does not exist yet. Report such targets as `new` so typos
  surface in the preview.
- Mark targets in normalized `plan.exempt` as `exempt`. Human output adds the note
  "exempt: these sessions count as review time in Stats".

**Writing.** `-d` prints the plan and changes nothing. A write runs through
`vault_rewrite::run` with `command_name` `bob themes split` and worktree subject "theme
splits", so `vault_mismatch`, `lock_busy`, `not_a_worktree`, `sync_failed`,
`stale_note`, `dirty_paths`, and `verify_failed` all behave as they do for rename.

- Split writes **no journal event**. OLD keeps its hidden state and pin, and every
  target keeps its own.
- `post_write` returns no extra paths.
- `on_committed` calls `cache::invalidate_vault`.
- The commit subject is `bob themes split: OLD → T1, T2, …`, with the changed target
  display names in alphabetical order.

**The verify hook.** For each touched file, check all of these:

- The reread contents equal the planned after-image.
- The line count is unchanged.
- The set of changed lines equals the planned set.
- Every changed line differs only in its name tail (`without_name_tail` is equal).
- Each planned line now carries its planned name.
- The scanned entry count is unchanged.

**Output.**

- _Human:_ reuse rename's styling.
  - A header gives OLD, the mode, and totals: one entry count per target, plus kept and
    unmapped.
  - Then one line per change, grouped by file:
    `2026/20261013.md:36  JOB → CHORE  0655-0735`.
  - Then the commit and sync lines, and warnings.
  - If no OLD entries remain after a write, add the line "OLD no longer appears in any
    daily note".
- _JSON:_
  `{schema_version: 1, ok, mode: "preview"|"write", theme: {key, name}, targets: [{key, name, entries, new, exempt}], kept, unmapped, remaining, files: [{path, changes: [{line, ref, before, after}]}], commit, sync: {pre, post}, warnings}`.
- _Failures:_ use rename's `{ok:false, code, message, hint?}` shape and exit-code
  mapping. The new code is `stale_entry`. `invalid_map` exits as a usage error.

### Reappearance fix (themes only)

In `renamed_reappeared` (`rename.rs`), keep choosing the latest rename _from_ each key,
using the fold's `(ts, host, line)` writer order. Then skip that key when a later event,
in the same order, re-adopts it:

- a `theme.rename` whose `to` is the key (the JOB → WRITE → JOB cycle), or
- a `theme.color` (pin or unpin) or `theme.unhide` naming the key.

A later `theme.hide` does not re-adopt a name. Hiding tucks a stale name away; it does
not mean the name is meant to be used.

Relation labels have a sibling `renamed_reappeared` in
`src/native/relation_labels/rename.rs` with the same cycle false positive. Leave it
unchanged. Record a PROPOSED FOLLOW-UP note on this phase's bead.

### Tests

- **New `tests/cli/themes_split.rs`**, registered with the other CLI test modules and
  modeled on `tests/cli/themes_rename.rs`: a git-initialized temp vault, `BOB_NOW`,
  `XDG_CACHE_HOME`, `HOSTNAME`, and a missing-config env. Cover:
  - The template lists only OLD entries, including composites and future-dated notes,
    and excludes `_templates/`. Two identical `- [ ] () — JOB` placeholders in one note
    get the same digest on different lines. The TSV round-trips through `-m` as an
    all-keep map, which gives `unchanged`.
  - A dry run reports per-target counts and the kept, unmapped, `new`, and `exempt`
    flags, and the vault bytes and git HEAD are unchanged.
  - A write produces one commit with the expected subject. Only the planned tails
    change, the composite collapses, and no `.bob/themes/*.jsonl` line is appended.
    `bob themes list -f json` reflects the new counts right away (the cache was
    invalidated).
  - Refusals: `unknown_theme`, `invalid_map` (short row, bad date, missing note,
    duplicate entry), `invalid_name` (`+` or blank), `stale_entry` (digest mismatch, and
    an entry without OLD), `unchanged`, and the vault-override refusal through
    `assert_vault_override_refused`.
  - The `-m -` stdin map works.
- **`renamed_reappeared` unit or CLI tests:**
  - An A → B then B → A cycle does not report A.
  - A later color or unhide on the old key clears its report.
  - A later hide does not clear it.
  - The existing `themes_rename_apply_verifies_commits_and_reappears` still passes.
- **Unit tests in `split.rs`** for map parsing (comments, blank lines, NOTE column, CRLF
  tolerance).

### Docs

- **`docs/pomodoro-themes.md`:**
  - Add a section "PT11 — Split by entry" covering scope, the template and map formats,
    the flow, verification, refusals, and the fact that split writes no journal event.
    Give the runbook: `-t` → edit THEME → `-m … -d` → `-m …`, and undo with
    `git revert`.
  - Update PT10 "Reappearance" with the re-adoption rule.
  - Add split to the `bob themes` CLI member list, the writers list, and the refusal
    codes (`invalid_map`, `stale_entry`).
- **`docs/reference/json-outputs.md`:** add a `bob themes split` row.
- **`docs/vault-git-sync.md`:** next to the rename commit note, add the
  `bob themes split: OLD → …` commit subject.
- **Generated docs:** run `just docs-gen` to regenerate `docs/reference/cli/themes.md`
  and the related CLI pages, then run `just docs-check`.

### Done when

- `cargo test` passes for the new and existing themes tests, plus clippy and fmt as CI
  runs them.
- `just docs-check` passes.
- `bob themes split --help` reads well.
- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`; the release train owns those.
  This is a new subcommand, so the commit is `feat(themes): …` with no `!`.

## Phase vault_migrate: Split the live JOB blocks and add WRITE

This phase runs against the live vault `~/bob` on apollo, using the bob binary built
from this phase's checkout, which contains `split_cmd`. Do not reinstall
`~/.cargo/bin/bob`; releases ride the train. Do not hand-edit daily notes or journal
files. Every vault write goes through `bob`. This phase changes no bob-cli files.

### Steps

1. **Build and snapshot.**
   - Run `cargo build --release` and set `BOB=<checkout>/target/release/bob`.
   - Confirm that `$BOB themes split --help` works.
   - In a fresh `mktemp -d` scratch directory outside the repo and the vault, save the
     output of `$BOB themes list -a -f json` and
     `$BOB stats show worked -r 2026-10-01..2026-10-27 -f json` as "before".
2. **Hide JOB first.** Run `$BOB themes hide JOB` while JOB is still a known theme.
   After a full split, JOB has no entries, and `hide` would refuse with `unknown_theme`.
3. **Generate the template.** Run `$BOB themes split JOB -t > job_split.tsv` in the
   scratch directory.
4. **Fill THEME from the mapping below.**
   - Match template rows to the table by date, time range or `()`, and child links. Line
     numbers are as of planning and may have moved; refs in the template are
     authoritative.
   - Entries in the template that the table does not list are JOB blocks added after
     planning. Classify each one with the codebook below and list it in the final
     report.
   - Table rows whose entry no longer exists or no longer says JOB (Bryan may have
     renamed today's open row) are dropped. List those too.
5. **Preview.** Run `$BOB themes split JOB -m job_split.tsv -d`.
   - Expect `unmapped: 0`, `kept: 0`, WRITE flagged `new`, and PLAN flagged `exempt`.
   - Expected target totals for the table as written:
     - > [!decision] planning_blocks = plan READ 37 · CHORE 23 · WRITE 17 · PLAN 10 ·
       > BUILD 2.
     - > [!decision] planning_blocks = chore READ 37 · CHORE 33 · WRITE 17 · BUILD 2.
       > Map every PLAN★ row to CHORE.
   - Explain any difference using step 4's added and dropped rows.
6. **Apply.** Run `$BOB themes split JOB -m job_split.tsv` (online, not `-o`).
   - On `stale_entry` or `stale_note`, regenerate the template, re-fill only the changed
     rows, then preview and apply again.
   - On `lock_busy`, wait about 30 s and retry, up to five times.
   - Record the commit sha.
7. **Pin colors.**
   - Run `$BOB themes color WRITE magenta`. Magenta is WRITE's hash hue, so nothing
     visibly changes, and it is a slot none of the other standing themes uses. This
     color event after the `WRITE → JOB` rename is also what marks WRITE as re-adopted
     under the new reappearance rule.
   - > [!decision] pin_standing Also run `$BOB themes color BUILD yellow`, `READ green`,
     > `FIX blue`, and `CHORE aqua`. These are the current hash hues, now frozen. Keep
     > the colors Bryan already knows.
8. **Verify and report.**
   - `$BOB themes list -a -f json`:
     - JOB is absent, or every remaining JOB is hidden and accounted for.
     - WRITE is listed, magenta, pinned, and visible.
     - `renamed_reappeared` is empty, and no `theme_hue_collision` hint involves WRITE.
   - The worked-minutes total for the window equals the "before" snapshot. Only the
     theme split may differ.
   - `bob vault-sync status --json` shows the post-sync succeeded.
   - Report the commit sha, the per-theme counts, any added or dropped rows with the
     theme chosen for each, and the one-line undo: `git -C ~/bob revert <sha>` then
     `bob vault-sync`. The hide and pin journal events stay after a revert. Undo them
     with `bob themes unhide JOB` and `bob themes color NAME auto`.

### Codebook (for rows the table does not cover)

Decide by the main result of the block and stop at the first yes:

- a choice of what to do (PLAN; see `planning_blocks`)
- something failed and was restored (FIX)
- routine upkeep or submitting an already-prepared application (CHORE)
- prose for people: CV, cover letters, LinkedIn, blog drafting, proofreading those, or
  personal job email (WRITE)
- mainly reading, or launching research (READ)
- otherwise BUILD

### Mapping (89 entries, classified 2026-10-27)

- "★" marks the `planning_blocks` rows.
- A short id like `^apply-warp` names the child Task Link's block id.
- Open rows are `[ ]`. Closed rows show their time range.

**2026-10-07**

- L45 · [ ] 1840-1905 · `^intro-proposal` (review launch-post outline and intro) → READ

**2026-10-08**

- L36 · 0845-0910 · `^ref-databricks-omnigent-job-fit`,
  `^ref-sase-launch-post-introduction`, `^ref-sase-launch-post-outline` → READ
- L81 · [ ] () · same three refs → READ

**2026-10-10**

- L44 · 1315-1340 · `^update-linkedin` → WRITE
- L47 · 1430-1520 · `^ref-databricks-omnigent-job-fit` (agent updating CV) → READ
- L50 · [ ] () · job-fit and launch-post refs → READ

**2026-10-11**

- L39 · 0755-0820 · `^update-linkedin` → WRITE
- L49 · [ ] () · `^update-linkedin`, Databricks CV/pitch refs, launch-post refs,
  `^job-sase-project` → READ

**2026-10-12**

- L29 · 0515-0530 · `^re-organize-job` → PLAN★
- L34 · 0605-0615 · `^re-organize-job` → PLAN★
- L39 · 0655-0720 · `^re-organize-job`, `^ref-software-engineer-cv-redesign` → READ
- L46 · 1235-1320 · `^ref-bryanbugyi-cv-extended` (reviewing own CV) → WRITE
- L49 · 1410-1425 · `^ref-software-engineer-cv-redesign` → READ
- L51 · 1505-1530 · `^final-proofread-cv` → WRITE
- L54 · 1700-1725 · `^final-proofread-cv` → WRITE
- L56 · 1730-1745 · `^apply-baeston` → CHORE
- L58 · [ ] () · launch-post refs → READ

**2026-10-13**

- L36 · 0655-0735 · `^ref-agent-platform-and-devtools-job-openings` ("Applied to Ramp,
  Clay, and LangChain") → CHORE
- L48 · 1010-1035 · `^ref-2026-goals-as-first-class-abstractions`, job-openings ref →
  READ
- L54 · 1200-1300 · goals ref, `^fix-core-dump` (epic launch) → READ
- L58 · 1315-1340 · goals ref, `^slash-capture`, `^equals-at-syntax` → BUILD
- L62 · 1420-1430 · `^apply-warp` → CHORE
- L64 · 1435-1445 · `^apply-warp` → CHORE
- L66 · 1530-1545 · job-openings ref ("Applied to last tier A role") → CHORE
- L75 · 1640-1655 · `^job-search-2026-prj` → PLAN★
- L77 · [ ] () · `^job-search-2026-prj` → PLAN★
- L85 · [ ] () · launch-post refs → READ

**2026-10-14**

- L29 · 0710-0735 · `^ref-second-wave-job-applications`,
  `^ref-instruction-bundle-provider-delivery`, `^apply-agent-quality` → CHORE
- L33 · 0735-0815 · wave-2 ref, `^ctrl-y-ref-panel`, `^apply-openrouter` → CHORE
- L37 · 0820-0825 · wave-2 ref ("Saved cover letter") → WRITE
- L40 · 0905-0930 · wave-2 ref, `^apply-schonfeld` → CHORE
- L43 · 0935-1000 · `^do-cleanup-tasks` ("fixed CV location") → CHORE
- L49 · 1010-1035 · `^do-cleanup-tasks` ("GitHub update + cover letter updates"),
  `^ctrl-shift-b`, `^gemini-4-argon` → WRITE
- L59 · 1245-1250 · `^apply-tower-tailscale` → CHORE
- L61 · 1325-1415 · `^apply-tower-tailscale` ("wrote draft Tailscale cover letter") →
  WRITE
- L64 · 1515-1535 · `^apply-tower-tailscale` → WRITE
- L66 · 1840-1850 · wave-2 ref, `^apply-decagon` → CHORE
- L69 · 1855-1900 · wave-2 ref, `^apply-harvey` → CHORE
- L72 · [ ] () · wave-2 ref → READ
- L74 · [ ] () · launch-post refs, `^no-more-0-prj` → READ

**2026-10-15**

- L31 · 0555-0620 · wave-2 ref → READ
- L33 · 0640-0650 · `^apply-coinbase`, `^fix-sase-sh-image` → CHORE
- L37 · 0650-0710 · `^apply-figma` → CHORE
- L39 · 0725-0735 · `^apply-two-sigma`, `^apply-datadog` → CHORE
- L42 · 0735-0745 · `job_2026_apply_29#^prj`, `^apply-gitlab`, `^apply-docker` → CHORE
- L46 · 0745-0755 · `^prj`, `^apply-optiver` → CHORE
- L49 · 0815-0820 · `^prj`, `^apply-de-shaw` → CHORE
- L52 · 0925-0940 · `^prj`, `^rename-prj`, `^research-timeline` → CHORE
- L59 · 1220-1220 · `^rename-prj` → CHORE
- L61 · 1220-1230 · `^collect-7-roles` → PLAN★
- L63 · 1305-1310 · `^collect-7-roles` → PLAN★
- L68 · 1505-1530 · `^no-more-0-prj` (research swarm) → PLAN★
- L71 · 1600-1605 · `^no-more-0-prj`, launch-post intro ref → PLAN★
- L75 · 1605-1620 · no link ("Launched research swarm to make sase.sh simpler") → READ
- L80 · 1650-1655 · `^apply-pydantic`, `^apply-stripe` → CHORE
- L84 · [ ] () · `^apply-pydantic`, `^apply-voleon`, `^apply-close`, `^apply-hightouch`,
  `^apply-doordash`, `^apply-susquehanna` → CHORE
- L91 · [ ] () · launch-post outline ref → READ

**2026-10-16**

- L33 · 0755-0805 · `^write-introduction` → WRITE
- L35 · 0910-0935 · `^write-introduction` → WRITE
- L65 · 1500-1525 · outline ref → READ
- L67 · 1530-1545 · outline ref → READ
- L69 · 1630-1705 · outline ref → READ
- L71 · 1715-1740 · outline ref → READ
- L79 · [ ] () · outline ref → READ
- L81 · [ ] () · `^apply-voleon`, `^apply-close` → CHORE

**2026-10-17**

- L67 · [ ] 1750-1815 · `^ref-gas-town-emergency-user-manual`, `^ref-sase-blog-261017`,
  outline ref → READ

**2026-10-18**

- L54 · [ ] 1745-1815 · Gas Town and blog-notes refs → READ

**2026-10-19**

- L59 · 1825-1900 · `^read-blog-notes`, `^github-io-docs` → READ
- L63 · [ ] () · `^write-introduction` → WRITE

**2026-10-20**

- L40 · 1310-1335 · no link ("Testing task relations!") → BUILD
- L53 · 1620-1645 · `^ref-redesigning-source-control`, `^cmd-t` → READ

**2026-10-21**

- L43 · [ ] () · `^write-introduction`, `^ref-agent-factory-merge-discipline-reading`,
  `^ref-solving-code-review-at-anthropic` → READ

**2026-10-22**

- L33 · [ ] 1315-1340 · code-review ref, `^write-introduction`,
  `^ref-structured-agentic-software-engineering` → READ

**2026-10-23**

- L54 · 1550-1600 · `^write-introduction` ("Research on how to split up into tasks") →
  READ
- L57 · 1645-1710 · `^ref-sase-blog-writers-block-task-plan` → READ
- L59 · 1750-1800 · writers-block ref → READ
- L61 · [ ] 1805-1830 · writers-block ref, `^write-introduction` → READ

**2026-10-24**

- L72 · [ ] () · `sase#^fill-four-cards` → WRITE

**2026-10-25**

- L54 · 1405-1420 · writers-block ref → READ
- L58 · 1625-1635 · writers-block ref → READ
- L60 · [ ] () · writers-block ref → READ

**2026-10-26**

- L29 · 0620-0645 · writers-block ref → READ
- L35 · 0800-0820 · writers-block ref → READ
- L49 · 1155-1220 · `sase_blog_done#^review-tasks` → PLAN★
- L54 · 1610-1625 · `^review-tasks` → PLAN★
- L62 · 1840-1850 · `sase_blog#^s0-spike` → WRITE
- L64 · 1910-1935 · `^s0-spike` → WRITE
- L66 · [ ] () · `^s0-spike`, `^ref-gmail-gog-reauth-apollo-athena` → WRITE

**2026-10-27**

- L64 · [ ] () · `job#^email-dan-d` → WRITE

The closed time involved is about 5h 15m for WRITE and 2h 5m for PLAN★.

## Out of scope

The research's other switch steps are not part of this plan:

- `plan.max_themes: 3`
- the TALK, PRACTICE, and VERIFY themes
- `^theme-on-task`
- the relation-label reappearance fix (a follow-up)
- Mac Capture and bob-plugins (both only consume `bob` output)
- memory notes (none become false)
