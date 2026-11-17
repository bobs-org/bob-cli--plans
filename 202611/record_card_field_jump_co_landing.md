---
tier: tale
size: medium
title: Land the Record Card and Schema Card field-jump epics safely together
goal: "bob-cli-ms (Record Card) and bob-cli-mu (Schema Card field jumps) both land with
  their work live in the vault, green docs gates, one coherent key story across the two
  cards, and no agent undoing the other epic's behavior.

  "
decisions:
  record_card_digits:
    ask:
      What should digits and Shift+J/K do on the Record Card, given the Schema Card's
      field jumps?
    choices:
      pick:
        Digits pick a choice and save; Shift+J/K step fields, clamped like the Schema
        Card
      pick_only:
        Digits pick a choice and save; Shift+J/K stay unbound on the Record Card
      field_numbers:
        Digits jump to field N like the Schema Card; pick digits only in the open list
    default: pick
    why: Keeps the one-key pick and carries the Schema Card's Shift+J/K habit over
    answer: pick
  climb_keeps_field:
    ask:
      Should = on the Record Card open the album Schema Card already on the same field?
    default: true
    why: You climb to change that field's schema; landing on Album details wastes a jump
    answer: true
  scoped_force:
    ask:
      May the remaining ms/mu agents pass -F to clear a sync refusal when the refused
      build has provably landed?
    default: true
    why:
      Three phases already left the vault stale; the checks and sync backups keep -F
      safe
    answer: true
proposed_by: bbugyi200.apollo.eo.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 16:48:40
status: wip
---

# Plan: Land the Record Card and Schema Card field-jump epics safely together

## Why this plan exists

The earlier conflict report (`research:202611/record_card_vs_schema_field_jumps.md`)
found one design fork and some merge friction. Re-checking live state on 2026-11-17
turned up **two landing blockers** and **one false acceptance line**. All three hit both
epics:

| #   | Finding                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Effect                                                                                                                                                                                                  |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **The vault deploy is wedged.** `~/bob/.obsidian/plugins/bob-ledger-tools/.bob-deploy.json` names a dirty build from `bob-cli_15` at `b0a9c89` (v1.129.0, 18:48Z). That was bob-cli-ms.3's pre-rebase build. ms.3 then landed a rebased commit (`60b5602`) and never redeployed. No commit holds the deployed bytes, so every `bob plugins sync` is refused. ms.4, mu.1, and mu.2 each recorded the refusal and moved on. Their work is on bob-plugins master but **not live in the vault**. `bob-cli_15` is now clean at `60b5602`, which is an ancestor of master. The other plugins are up to date. | mu.3, mu.land, ms.5–7, and ms.land will all hit the same refusal. With two epics landing on one plugin, a phase's rebased landing commit no longer byte-matches its dirty deploy. The wedge will recur. |
| 2   | **The docs gate is red on bob-cli master.** `docs/reference/json-outputs.md` is at 6089 words against a 6000 cap. It went over at ms.1 (`99005975`) and ms.2 (`48a3116d`). The bob-cli-ms epic's note #2 assigns the fix to that epic. bob-plugins' `npm run docs:check` runs bob-cli's gate, so every remaining phase of both epics fails it. The red gate also blocks docs publication. CI and the release train do not run it.                                                                                                                                                                      | Each phase records it as "pre-existing", and agents may race to fix it. em--code (`plan:202611/record_notes_on_demand.md`) is about to add a `bob records tidy` row to the same page.                   |
| 3   | **mu's acceptance line is false today.** It says "Record note: `==` opens the same card with the same keys." But `248m`'s `schemaDoubleEqualsTargetKind` returns `"album"` or `"bank"` only (the shipped `album_only` decision). On a record, `==` reindents. Records reach the Schema Card through the **Edit album schema** palette command. bob-cli-ms.7 will make a record's `==` open the Record Card.                                                                                                                                                                                            | mu.land may "fix" `248m` to open the Schema Card on records, which collides head-on with ms.7.                                                                                                          |
| 4   | **Docs pages are at their caps.** `bob-ledger-tools.md` and `bob-navigation-hotkeys.md` both sit at exactly 6000 words. ms.7 must add Record Card text to both. `record-card.md` (641) has room.                                                                                                                                                                                                                                                                                                                                                                                                       | ms.7 trips the budget unless it puts the detail in `record-card.md`.                                                                                                                                    |

**Where things stand.** mu.1 (`137c77f`) and mu.2 (`e32deb6`) have landed. mu.3 (bank
group jumps) is running, and mu.land waits for it. ms.1–ms.4 have landed. ms.5 (Record
Card shell) just started on base `f49e175`. ms.6, ms.7, and ms.land are waiting. **mu
will finish long before ms**, so ms.land is the last agent to touch both epics. Epic
phases cannot depend on outside beads, so the after-both-land proof rides on ms.land
through bead notes. Both epic plans already tell workers to recheck bead notes when they
start.

## Ground rules

- Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Work only in the opened
  checkout, and never in another agent's workspace.
- Do not edit either epic's plan, phase code, or any memory file. This plan changes
  bob-plugins **only through the deploy** (no commits there). It changes bob-cli only in
  `docs/reference/json-outputs.md`, plus prose moved to the pages that page links to.
- Each note cites this approved plan as Bryan's authority, using its archived
  `plan:202611/<name>.md` ref, so phase agents treat it as instruction rather than
  advice. Never author a note as Bryan.
- Read beads with `sase bead read`, and post notes with `sase bead note <id> "<text>"`.
- Do the steps in order. Steps 1 and 2 are time-sensitive: mu.3 is close to done, and
  mu.land starts right after it.

## Step 1: Unwedge the bob-ledger-tools deploy

Bryan's approval of this plan authorizes **exactly one** `-F` sync of
`bob-ledger-tools`, after every check below passes. If any check fails, do not force.
Ask Bryan with `/sase_questions`, and name what `-F` would drop (the deployed version
and the features it carries).

1. Re-read the stamp. If it no longer names the `bob-cli_15` dirty `b0a9c89` build,
   re-run `bob plugins sync --repo "$PWD" --no-pull --dry-run -p bob-ledger-tools` and
   follow `docs/develop/deployment.md`. The wedge may already be gone.
2. Prove the deployed build has landed:
   - the stamped `source_root` checkout has no uncommitted changes;
   - its HEAD is an ancestor of `origin/master` (`git merge-base --is-ancestor`);
   - `bob-cli-ms.3` is CLOSED.
3. Containment spot check. List the top-level `function NAME(` and `class NAME`
   identifiers in the vault's `main.js` that do not appear in master's
   `plugins/bob-ledger-tools/main.js`. Expect none, or only names ms.3 renamed before it
   landed (check `git show 60b5602`). Stop and ask if anything looks like unlanded
   feature work.
4. From your clean, fast-forwarded bob-plugins checkout:
   1. Run `npm run build:check`.
   2. Run `bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools -F`.
   3. Note the backup directory that sync prints.
5. Verify:
   - a dry-run sync of all plugins reports everything up to date;
   - the new stamp is clean at the master HEAD you deployed;
   - the vault `manifest.json` version matches master.
6. Record the old stamp (root, head, digest), the new head, and the backup directory.
   You will put them in the Step 2 notes.

## Step 2: Post the cross-epic notes

Post these in order: `bob-cli-mu` first, then `bob-cli-ms.5`, `bob-cli-ms.6`,
`bob-cli-ms.7`, and `bob-cli-ms` last. Keep each note tight, using the numbered items
below. Fill in the branch each decision selected, and do not mention branches that were
not chosen.

### Note on `bob-cli-mu` (read by mu.land)

1. **Acceptance correction.** Replace "Record note: `==` opens the same card" with:
   - From a record note, **Edit album schema** (palette) opens the Schema Card, and
     `⇧J`/`3` work there.
   - A record's `==` is album-only today. `bob-cli-ms.7` owns that route: it will open
     the Record Card, which is expected and not a regression.
   - **Do not change `248m`.**
   - If ms.7 has already landed, also check that `=` from the Record Card reaches the
     Schema Card with field jumps working.
2. **Deploy.** The vault was unwedged at `<new head>` (old stamp: abandoned dirty ms.3
   build). Before each sync, fetch and merge `origin/master`, then run `npm run build`.
   Sync may refuse because a running bob-cli-ms agent's dirty deploy is in place. If
   that stamp's `source_head` already contains every bob-cli-mu commit, mu's work is
   live: record that, then continue without waiting and without forcing. Add the refusal
   rule (below).
3. **Docs.** The json-outputs budget is fixed by this plan's bob-cli commit (Step 3).

### Notes on the bob-cli-ms beads

`bob-cli-ms.5` gets items 1–7. `bob-cli-ms.6` gets this line: "if ms.5 closed without
items 1–4, implement them before the stages", plus items 5–7. `bob-cli-ms.7` gets items
6–7. The `bob-cli-ms` epic note gets all items, for ms.land.

1. **Digits.**

   > [!decision] record_card_digits = pick
   >
   > Keep the plan's key table: on a choices row, `1`–`9` pick that choice and save. On
   > any other row a digit does nothing and the status line teaches
   > `Digits pick a choice on choice fields`. It never moves the selection.

   > [!decision] record_card_digits = pick_only
   >
   > Same digit behavior as `pick` (pick and save on choices rows, a teaching line
   > elsewhere).

   > [!decision] record_card_digits = field_numbers
   >
   > This overrides the Record Card key table. In browse mode, `1`–`9` select field N in
   > painted order, never write, and on a miss bump and announce
   > `No field 7 · this record has 3 fields`. Digits pick only inside the open choices
   > pick list (Enter, then a digit). Rows 1–9 show a field-number keycap: reuse mu's
   > keycap builder with **Record-Card-scoped** CSS. The hint strip shows `1–N field`.

2. **`⇧J`/`⇧K`.**

   > [!decision] record_card_digits = pick
   >
   > In browse mode, `⇧J`/`⇧K` step to the next or previous field and clamp at the ends
   > (no wrap). On a miss they bump through mu's `schemaCardBump` (`241xb`) and announce
   > `No next field` / `No previous field`. On a hit they announce
   > `<Label>, field N of M` through the card's live region. They are silent on
   > `event.repeat`, follow reduced motion, and are inert in stages, inputs, and IME
   > composition. They never write. Leave them out of the hint strip, and list them in
   > `docs/record-card.md`'s key table.

   > [!decision] record_card_digits = pick_only
   >
   > Leave `⇧J`/`⇧K` unbound on the Record Card.

   > [!decision] record_card_digits = field_numbers
   >
   > The same `⇧J`/`⇧K` behavior as under `pick` (clamped steps, bump and announce,
   > inert outside browse, never write). Also show them in the hint strip as
   > `⇧J/K 1–N field`.

3. **`=` climbs to the same field.**

   > [!decision] climb_keeps_field
   >
   > `=` calls `openAlbumSchemaEntry({ filePath, field: <selected field key> })`. Thread
   > the field through `openAlbumSchemaCard` into the modal's starting `selectedKey`
   > (`field:<key>`), falling back to `stats` when that target is missing. `241y` is 973
   > lines, so keep the change to a few lines. Test it: `=` on Status opens the Schema
   > Card on Status, `⇧J` then moves to the next field, and an unknown key opens on
   > Album details.

   > [!decision] climb_keeps_field = no
   >
   > `=` calls `openAlbumSchemaEntry({ filePath })` exactly as the epic plan says.

4. **CSS isolation.** Never set `bob-album-schema-dialog` or
   `bob-album-schema-dialog-body` on the Record Card. mu.2's transparent-row, gutter,
   keycap, and group-break rules key off those classes. Use `bob-record-card-*`. The
   field-jump research line "a future Record Card should call the same field-jump
   helpers" is superseded by item 1. Do reuse mu's bump and announcer helpers for
   feedback.
5. **Docs.** In `docs/record-card.md`, state the contrast in one sentence. Under `pick`
   or `pick_only`: "Digits here pick a choice; on the album Schema Card they jump to
   field N." Under `field_numbers`, say both cards number fields. `bob-ledger-tools.md`
   and `bob-navigation-hotkeys.md` are at their 6000-word caps. Put the detail in
   `record-card.md`, link to it from those pages, and trim to fit. Never raise a budget.
6. **Deploy discipline.** Right before every `bob plugins sync`, fetch and merge
   `origin/master` and run `npm run build`. Sync your final tree, so the vault build is
   what lands. Add the refusal rule (below).
7. **Merge friction.** The `test:unit` list in `package.json` is a single line, so keep
   both epics' entries. Rebuild `main.js` and never hand-merge it. ms bumps the
   bob-ledger-tools minor version, and mu does not, so take the next free minor.
8. **ms.land acceptance additions** (epic note only):
   - A record's `==` opens the Record Card. From there, `=` opens the Schema Card on the
     same field (under `climb_keeps_field`), and `⇧J` and `3` navigate there and never
     write.
   - The Record Card shows no Schema Card keycaps, unless `field_numbers` was chosen.
   - At close, the vault holds a clean `origin/master` deploy of bob-ledger-tools,
     bob-navigation-hotkeys, and bob-command-mode.
9. **Housekeeping** (epic note only). Epic note #2, the json-outputs budget, is resolved
   by this plan's bob-cli commit; do not redo it.

### The refusal rule (added to the mu note, item 2, and the ms notes, item 6)

> [!decision] scoped_force
>
> Bryan pre-authorizes `-F` for bob-ledger-tools, bob-navigation-hotkeys, and
> bob-command-mode, for the rest of these two epics only. Use it only when **all** of
> these hold:
>
> - the stamped `source_root` exists and has no uncommitted changes;
> - its HEAD is an ancestor of `origin/master`;
> - the author's phase bead is closed;
> - you deploy a clean, up-to-date `origin/master` (stash, fast-forward, run
>   `npm run build:check`, sync `-F`, pop, rebuild, then sync again without `-F`).
>
> Afterwards, note the old stamp and sync's backup directory on your bead. In every
> other case, follow `AGENTS.md`.

> [!decision] scoped_force = no
>
> When the documented recipe cannot clear a refusal, ask Bryan with `/sase_questions`.
> Never close a phase with only a "sync refused" follow-up.

## Step 3: Bring json-outputs.md back under its budget (bob-cli)

1. Trim `docs/reference/json-outputs.md` to **≤ 5850 words** by bob-docs' own counter.
   The headroom absorbs em--code's in-flight `bob records tidy` row and the next few
   rows.
   - Keep every table row, every `{#anchor}`, every link target, and both records
     contracts (`show TYPE STEM`, `update -i`).
   - Prefer cutting prose that repeats its linked contract page. Move irreplaceable
     detail to that page, and keep it within its own budget.
   - Never raise a budget or add an exemption.
2. Gates:
   - `just docs-check` passes;
   - from bob-plugins, `BOB_CLI_DIR=<this bob-cli checkout> npm run docs:check` passes;
   - `rg` finds no broken in-repo links to moved text.

   This is a docs-only change, and no test pins this page, so `just check` is not
   required.

3. Rebase onto `origin/master` before the final count. em--code may have landed edits to
   the same page; keep both sides, then recount.
4. Commit through the normal final flow as
   `docs(reference): trim json-outputs.md back under its word budget`.
5. Post a note on `bob-cli-ms`: "Note #2 (json-outputs budget) is resolved by this
   plan's docs commit: N → M words." Name the commit subject and this agent.

## Step 4: File the recurring deploy-wedge task

Use `/sase_new_task`. Its duplicate search should cover `bob-cli-lm` note #2 (the same
wedge from lj.4) and the three refusals in this epic pair.

- **Problem.** `evaluate_guard` in `src/native/plugins/deploy.rs` accepts another
  checkout's dirty deploy only through a byte-identical commit. A phase that rebases
  over parallel work never produces one, so the guard refuses and agents leave the vault
  stale.
- **Direction for its planner.** Treat a stamped checkout that is now clean, whose HEAD
  is an ancestor of the incoming HEAD and has moved past `source_head`, as landed. Print
  the proving checkout, rely on sync's backups, and keep refusing truly unlanded work.
- **Size:** `large`, because it changes deploy-safety semantics and needs a design pass.

## Step 5: Report

Summarize for Bryan:

- the deploy (old stamp, new head, backup directory, live versions);
- each note posted (bead and note number);
- the json-outputs word counts and gate results;
- the new task bead;
- anything that blocked a step.

Remind him that Obsidian must reload bob-ledger-tools to pick up the new build.

## Out of scope

- Implementing any Record Card or Schema Card code. The ms phases own it, and the notes
  steer them.
- Editing either epic's plan, decisions, or memory, and authoring a decisions strand.
  Bryan declined both `field_number_record` and `decision_record`.
- Changing the deploy guard now (Step 4 files it).
- em--code's `bob records tidy` work, apart from leaving it word headroom.
