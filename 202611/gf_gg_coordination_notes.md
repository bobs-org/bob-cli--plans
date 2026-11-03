---
tier: tale
title: Cross-epic coordination notes for bob-cli-gf and bob-cli-gg
goal:
  The gf and gg epic beads carry a checklist their land agents (and unfinished phases)
  use to land both features without one breaking the other.
size: small
decisions:
  phase_notes:
    ask:
      Also post a short coordination note on each unfinished phase bead of both epics?
    default: true
    why:
      Phase workers read only their own bead, and the conflicts surface when they commit
    answer: true
proposed_by: bbugyi200.athena.17b
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 10:36:05
status: wip
---

# Cross-epic coordination notes for bob-cli-gf and bob-cli-gg

## Goal

Two epics are in flight at the same time and both change the same ledger rows in
bob-ledger-tools (bob-plugins):

- `bob-cli-gf`: off-clock link chips (plan `plan:202611/off_clock_link_chips.md`). This
  epic adds an `off` chip after ledger Task Links whose task is off the clock.
- `bob-cli-gg`: Event link times (plan `plan:202611/event_link_times.md`). This epic
  adds the Event Task's time pill before ledger Task Links that point at an Event Task.

Record what the plans leave out by appending notes to the two epic beads, so each land
agent can check the notes item by item. gg's plan already has a "Coordination with epic
`bob-cli-gf`" section. gf's plan predates gg and has none. Neither plan covers the
regenerated `main.js`, the double manifest bump, the `bob plugins sync` refusal, or
resolver parity.

This is bead-note work only. Do not edit repo files, plans, memory, or bead
statuses/dependencies. Do not add a dependency edge between the epics.

## Facts this plan relies on (verified while planning)

- `bob-cli-gf.1` is closed. Its contract landed in bob-cli as `5f35f0b3`
  (`docs/off-clock-link-chips.md`, registered at contracts order 36, and a rule-5
  sentence in `docs/task-link-marks.md`). `gf.2` and `gf.3` are `in_progress`.
- `bob-cli-gg.1` is `open`. `gg.2` and `gg.3` are `in_progress`, preassigned by the epic
  launch. Both epics' agents run on apollo.
- bob-plugins master was `175aced` with no gf or gg commits. bob-ledger-tools
  `manifest.json` was `1.95.0`. `main.js` is generated and committed, and there is no
  `.gitattributes` merge driver.
- The SASE prompt templates (`bd/work_phase_bead`, `bd/land_epic` in sase
  `default_config.yml`) work like this. A phase worker reads only its own phase bead,
  which shows the epic plan and DECISIONS but not the epic's notes. A land agent must
  "review the epic bead's own notes" and confirm each one was addressed. A phase worker
  also checks its own notes for remaining work left by an earlier agent, so any phase
  note must say it is not remaining work.
- `bob plugins sync` has a deploy guard (`src/native/plugins/deploy.rs`). It refuses to
  overwrite a deploy that came from another checkout's uncommitted build, or from a
  clean revision that is not an ancestor. The refusal message itself suggests `-F`.
- Both epics resolve the clock through `taskEffectiveClockResolution`
  (`src/036-pomodoro-clock.js`). gf goes through `pomodoroClockLinkFacts`, gg through
  `gcalEventsApplyClockToRow` in `src/291c-plugin-gcal-event-clock.js`.

## Step 1: Refresh the state before writing

1. Read both epics and all six phases with `sase bead read <id> -r "<why>"`. Note which
   phases are now closed and whether either epic bead already has notes.
2. Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`). Run
   `git log --oneline -15` and grep commit messages for `bob-cli-gf` / `bob-cli-gg`.
   Then read `plugins/bob-ledger-tools/manifest.json`'s version.
3. In bob-cli, run
   `git log --oneline -5 -- docs/event-link-times.md docs/task-link-marks.md` to see
   whether gg.1 has landed.
4. Update the "State when written" line of the shared note below to match. If one epic's
   bob-plugins commit has already landed, name its short SHA and the version it took.
   Phrase item 3 so the other epic bumps past that version. If both have landed, rewrite
   items 1–4 as checks for the land agents, not warnings for the phases.

## Step 2: Post the epic notes

Post the shared body below to **both** epic beads. On `bob-cli-gf`, put header A first;
on `bob-cli-gg`, put header B first. Write each note to a file in a fresh `mktemp -d`
directory outside every repo. Run `grep -n '@' <file>` and make sure it finds nothing,
because an `@` would turn into an attachment. Then post with
`sase bead note <id> "$(cat <file>)"`. Keep the Markdown backticks.

Header A (`bob-cli-gf`):

```text
CROSS-EPIC COORDINATION with bob-cli-gg (Event link times). Not remaining work for any phase. Checklist for whichever of gf/gg commits to bob-plugins second, and for this land agent (confirm each item). gf's plan predates gg and has no coordination section; gg's plan has one, and these items add what both plans lack.
```

Header B (`bob-cli-gg`):

```text
CROSS-EPIC COORDINATION with bob-cli-gf (off-clock link chips). Not remaining work for any phase. Checklist extending this plan's "Coordination with epic bob-cli-gf" section, for whichever of gf/gg commits to bob-plugins second, and for this land agent (confirm each item). gf's plan has no matching section, so gf's workers do not know about gg.
```

Shared body:

```text
State when written (2026-11-03): gf.1 landed in bob-cli (5f35f0b3); no gf or gg commit in bob-plugins yet (master 175aced, bob-ledger-tools 1.95.0); gf.2 and gg's phases in flight in parallel.

1. Same hunks; expect textual conflicts and resolve by keeping both features:
   - `src/269-plugin-progress-marks-reading.js` `ledgerMarkClass` list must hold BOTH `bob-off-clock-link-chip` and `bob-event-link-time`. Dropping either one makes rows wearing that host stop counting as ledger rows: ◐/🔓/🔑 vanish and a status ring appears.
   - `src/279-plugin-task-theme-api.js` `taskThemeRequestSourceText` fill callback: keep both refresh calls (`scheduleOffClockLinkChipsRefresh`, `scheduleEventLinkTimesRefresh`).
   - `styles.css` calendar block (about lines 5240-5370): gf adds `.bob-off-clock-link-chip` to the Event off-clock chip rules and renames `--bob-calendar-off-clock-icon` to a body-level `--bob-off-clock-icon`; gg appends its host rule after the wing rules, right next to that variable. Nothing may keep using the old variable name.
   - `src/350-exports.js` tail, the `package.json` "test" line, the `manifest.json` description, and `docs/bob-ledger-tools.md` ("Marks and rendered blocks", "Session toggles", "Calendar Events").
   - Separate spots that should auto-merge: `src/fragments.json`, `src/310-install-methods.js`.
2. Generated output: never hand-merge `main.js` or `npm run docs:gen` output. Resolve the `src/` and docs sources, then rerun `npm run build`, `npm run docs:gen`, and `npm run build:check`.
3. `manifest.json` version: both plans bump the minor from 1.95.0. The second to land takes the next minor past master at commit time (1.97.0 if the first took 1.96.0).
4. `bob plugins sync`: both epics' phase 2 and phase 3 run it. Its deploy guard refuses to overwrite another checkout's uncommitted or non-ancestor deploy, and the refusal text suggests `-F`. If the refusal names the other epic's checkout, do NOT use `-F`; that rolls the other feature out of Bryan's vault. Rebase onto a bob-plugins master that contains the other epic's commit, rebuild, and sync again. If that commit is not on master yet, record the sync as pending in a phase note.
5. Intended overlap, keep both: a ledger link to an off-clock Event Task wears gg's dashed time pill before the link and gf's `off` chip after it (`pill link 🔓 🔑 N off`). Neither epic should hide the other's mark.
6. Parity: both decide the clock with `taskEffectiveClockResolution` (gf through `pomodoroClockLinkFacts`, gg through `gcalEventsApplyClockToRow`). Once both have landed, a row's pill has `data-off-clock="true"` exactly when the row wears the chip. Known transient gap: a full-path same-note link (`[[2026/20261103#^id]]`), where gf reads the source-text cache and gg reads the live document. A disagreement anywhere else is a bug.
7. gf's source-text freshness fix (refresh an already-cached entry on `metadataCache` `changed`) also serves gg's Reading-view pills. Keep exactly one such fix; do not add a second invalidation path.
8. Once both have landed, run both unit suites (`test-ledger-tools-off-clock-link-chips.cjs`, `test-ledger-tools-event-link-times.cjs`). Confirm gg's EP3 (`pill link off`) and gf's OP5 (◐/🔓/🔑 still paint, no ring) with both hosts on one row.
```

## Step 3: Post phase notes

> [!decision] phase_notes
>
> Post one note on each phase bead that is still not closed after Step 1. Skip any phase
> that has closed. Start every note with this exact prefix:
> `COORDINATION NOTE (from a planning agent; not remaining work and not an earlier attempt at this phase):`.
> Then add the phase-specific text below. Use the same file and `grep '@'` procedure as
> in Step 2.
>
> - `bob-cli-gf.2`: Epic bob-cli-gg (Event link times) edits the same bob-plugins files
>   in parallel, and gf's plan does not mention it. Before committing, read the
>   CROSS-EPIC COORDINATION note on epic bob-cli-gf (`sase bead read bob-cli-gf`) and
>   fetch bob-plugins master. If gg's commit is already there, apply items 1–4: rebase,
>   keep both features, rebuild `main.js` instead of hand-merging it, bump
>   `manifest.json` past master, and never sync with `-F` over gg's deploy.
> - `bob-cli-gf.3`: Read the CROSS-EPIC COORDINATION note on epic bob-cli-gf. Item 4
>   applies to every `bob plugins sync` in this phase. If gg has landed, a time pill may
>   sit before a chipped Event Task link in the scratch vault; that is intended (item
>   5), not a regression.
> - `bob-cli-gg.1`: `docs/task-link-marks.md` rule 5 already ends with gf.1's sentence:
>   "Ledger rows wear no ring; when their task is off-clock they wear the off-clock link
>   chip instead". Add the Event-pill clause so the two read as additive, since one row
>   can wear both. Do not add a second "instead". Register `event-link-times` after
>   `off-clock-link-chips` (order 36) in `docs/_meta/contracts.yml`.
>   `tools/docs/tests/test_contracts.py` currently expects 36 entries.
> - `bob-cli-gg.2`: Read the CROSS-EPIC COORDINATION note on epic bob-cli-gg (items 1–7)
>   before committing; it extends this plan's coordination section. gf's workers do not
>   know about gg, so if gg commits second, gg owns the merge. Use the renamed
>   `--bob-off-clock-icon` if gf has landed. Rely on gf's source-text freshness fix
>   instead of adding another.
> - `bob-cli-gg.3`: Read the CROSS-EPIC COORDINATION note on epic bob-cli-gg. Item 4
>   applies to every `bob plugins sync` in this phase. If gf has landed, cover EP3 and
>   item 6 (pill dashed exactly when the chip shows) in the live run.

> [!decision] phase_notes = no
>
> Skip this step. Only the two epic notes are posted.

## Step 4: Verify

1. `sase bead read bob-cli-gf -r "<why>"` and `sase bead read bob-cli-gg -r "<why>"`
   show the new note with intact backticks and no attachment line. Read each noted phase
   bead the same way.
2. If a note posted wrong, fix it with `sase bead note --edit <n>` (re-read after, since
   ordinals shift). Never post a second copy.
3. Confirm `git status` is clean in the bob-cli workspace and in the opened bob-plugins
   checkout. This work changes only the bead store.

In the final report, list each bead that received a note, any phase that was skipped
because it had closed, and any "State when written" change made in Step 1.
