---
tier: tale
size: medium
title:
  Finish bob-cli-qi's Area, inbox, and Project takeover acceptance, then close the epic
goal:
  The real-Obsidian takeover runner proves single-owner frontmatter takeover for area,
  sub-area, inbox, and active and terminal project notes. It also proves their
  exclusions, fail-open behavior, gestures, and ^prj integrity, with reviewed
  screenshots. The validation doc states that true matrix, and epic bob-cli-qi is closed
  with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-qi.land
proposed_with: claude/opus
bead: bob-cli-qi
create_time: 2026-11-26 16:22:25
status: done
---

- **PARENT:**
  [202611/bank_hub_demotion.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_hub_demotion.md)
- **BEAD:**
  [bob-cli-qi](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qi/README.md)

# Finish bob-cli-qi's Area, inbox, and Project takeover acceptance, then close the epic

## Why

Epic `bob-cli-qi` ("Demote bank.md to a single-owner bank hub, then finish the card
epics") is otherwise verified. Its land agent confirmed every other item:

- the vault, the bob, and the plugin target state;
- three Bank/album/record/day/week takeover runs, all **GO 63/63**;
- the guard grep and the passive-rendering proof;
- the docs and the approved memory.

One gap blocks the close. The `takeover-proof` phase (`bob-cli-qi.4`) had to prove the
real-Obsidian frontmatter takeover for the **top-level area, sub-area, inbox, active
project, and terminal project**, as well as the bank hub. It also had to capture **area,
inbox, and project screenshots**. That phase closed without either:

- `scripts/obsidian-frontmatter-takeover-e2e.cjs` in bob-plugins has no area, inbox, or
  project fixtures or checks.
- `scripts/obsidian-home-cards-e2e.cjs` expects native Properties to stay visible, so it
  never exercises the takeover. It also aborts before screenshots on a pre-existing
  failure, now tracked as `bob-cli-qr`.
- `docs/develop/validation.md` § "Bank and home-card takeover evidence" says the
  takeover runner covers Area, inbox, active/terminal Project, and `^prj` cursor
  integrity. That claim is false today.

These Area and Project home cards already take over native Properties in Bryan's vault
whenever bob advertises `props_contract_v1`, and the installed bob 0.6.0 does. This tale
adds the missing real-app evidence, repairs any takeover defects it finds, corrects the
doc, and then closes the epic.

Read the epic first: `sase bead read bob-cli-qi -r "<why>"`, including its notes. The
LAND TRIAGE note records every follow-up outcome. Then read its plan,
`plan:202611/bank_hub_demotion.md` (the PLAN path that `sase bead read` prints),
especially the `takeover-proof` evidence table and "Epic acceptance (land agent)".

## Governing decisions

- `decisions:bank-hub-is-path-canonical-never-area`: `bank.md` stays excluded from Area
  ownership. The new area fixtures must not weaken the bank checks.
- `decisions:projects-block-from-prj-line`, as amended by
  `decisions:subprojects-inherit-project-blocking`: the takeover never moves, hides,
  duplicates, or writes a `^prj` line.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the Property Editor and Settings keep the
  shared Esc/`q`/Ctrl+`[` close ladder.
- `decisions:bob-cli-releases-ride-a-daily-train`: this tale should not need bob-cli
  changes. If it does, never edit `Cargo.toml` versions or `CHANGELOG.md`.

## Setup

- Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`). Use only
  the path it prints, and read its `AGENTS.md`. Edit fragment sources, never a generated
  `main.js`. Pull first, because other epics (`bob-cli-qm`, `bob-cli-qp`) land in the
  same files.
- Build bob from current bob-cli master (`cargo build`) and pass it as `BOB_BIN`.
  `BOB_CLI_DIR` must be a **clean** bob-cli checkout, or the runner refuses its
  provenance prerequisite. If your checkout is dirty, point it at a
  `git worktree add --detach <tmp> HEAD`.
- The runner command takes about 2 minutes per run:

  ```sh
  BOB_BIN=<debug bob> BOB_CLI_DIR=<clean bob-cli checkout> \
    xvfb-run -a node scripts/obsidian-frontmatter-takeover-e2e.cjs \
    -e /snap/obsidian/current/app/obsidian -o <shots dir> -r <report.json> --profile full
  ```

  It runs on synthetic notes in an isolated vault and profile. Never point any automated
  Obsidian run at the live vault.

- Baseline before you change anything: the current master run is GO 63/63 with 46
  screenshots. Keep it that way.

## Steps

1. **Fixtures.** In `scripts/obsidian-frontmatter-takeover-e2e-lib.cjs`, extend `PATHS`
   and `fixtureNotes()` with synthetic notes that use live-shaped frontmatter (quoted
   `[[…]]` links, bare dates, an offset `created` timestamp):
   - a top-level area (`type: "[[area]]"`, `parent: "[[area]]"`);
   - a sub-area whose parent is that area;
   - an inbox area that the home-card model reports with the INBOX pill (see
     `homeCardInbox` in `plugins/bob-ledger-tools/src/249b-plugin-home-card-model.js`);
   - an active project with a WIP `- [ ] #task #prj … ^prj` line;
   - a terminal project (done or canceled `^prj`);
   - exclusion notes that must stay native:
     - an untyped `area.md` and an untyped `project.md`;
     - an area-typed note under `_templates/`, one under `done/`, and one under
       `_conflicts/` (the `249b` path exclusions).

   Add each new note to the self-test assertions in the lib. Keep `bytes-unchanged`
   covering every fixture.

2. **Ownership checks.** In `scripts/obsidian-frontmatter-takeover-e2e.cjs`, add Live
   Preview and Reading checks for every new eligible fixture (for example
   `own-area-live`/`-reading`, `own-subarea-*`, `own-inbox-*`, `own-project-*`,
   `own-project-terminal-*`). Each one asserts:
   - exactly one card;
   - one title;
   - one Properties foot;
   - native Properties hidden while the bob advertises `props_contract_v1`.

   Reuse the runner's existing probe and `waitOwned` helpers; do not manufacture
   ownership. Add the matching negative checks:
   - Each exclusion stays native with no home card.
   - With a `props_v1`-only bob, area and project stay native (fail open) while bank
     stays owned. The runner already strips `props_contract_v1` for one case, so extend
     that case instead of adding a new mechanism.

3. **Gestures and focus on area and project.**
   - `-` toggles the drawer.
   - `_` folds the card's own details.
   - `==` opens Area or Project Settings.
   - Bare `+` opens the generic Property Editor.
   - The editor's Esc/`q`/Ctrl+`[` ladder closes it.
   - Vim `k`/`gg` and Up at body start land on visible body text, never inside the
     hidden YAML or the hidden H1.
4. **`^prj` integrity.** On the active project in Live Preview:
   - Cursor placement and a selection on the `^prj` line work.
   - The card never moves, hides, duplicates, or writes that line. Use a byte hash
     before and after.
   - Typing on an adjacent scratch line leaves the `^prj` line byte-identical.

   If the harness can drive IME composition (`Input.imeSetComposition` over CDP), cover
   IME on that line too. If not, record why it is not covered.

5. **Screenshots.** Add the area, inbox, active project, and terminal project to
   `screenshotCards`, giving light/dark × narrow/wide × Live Preview/Reading.
   - Open every new PNG and inspect it. Record a verdict, size, and sha256 for each.
   - Check that each shows one card, one title, and one foot, that rows align, that
     chips sit correctly, and that narrow splits wrap cleanly.
   - Fix any visual drift in bob-plugins.
6. **Repairs.** Fix in bob-plugins any area, inbox, or project takeover defect the new
   checks expose. This epic adopted the canceled `bob-cli-pl` acceptance, so those
   defects are in scope.
   - Do not weaken an ownership, cursor, or spawn assertion just to get green.
   - Leave the seven `obsidian-home-cards-e2e.cjs` failures to `bob-cli-qr`. If you
     learn something new about them, add it to that bead with `sase bead note`.
   - For any unrelated failure that also reproduces on clean master, use
     `/sase_new_task`.
7. **Validate and deploy.**
   - Run `npm run build`, `npm run build:check`, the focused suites (the
     `test-ledger-tools-property-card-*`, `home-card-*`, and `property-editor-*`
     suites), and `npm test`. Record any pre-existing failures with the exact command
     and SHA.
   - Run the full takeover profile until it is **GO** with zero failures, then run it
     once more to show stability.
   - Commit, then run `bob plugins sync -p bob-ledger-tools` without `-F`. Another
     checkout may hold an unrelated uncommitted deploy of a different plugin, and `-p`
     leaves it alone. If sync refuses, follow `AGENTS.md`.
8. **Docs.** Rewrite `docs/develop/validation.md` § "Bank and home-card takeover
   evidence" so it states the true tested matrix: kinds, modes, gestures, exclusions,
   capability fail-open, and `^prj` integrity. Give the new pass counts and the new
   evidence refs. Also mention the land agent's three GO runs:
   - `file:explicit:660cabc9cf32530dcf480887` (report);
   - `file:explicit:d254ed707483200bbaaca14f` (bundle with shots, home-cards
     attribution, and pre-epic comparison).

   Do not grow the doc past its word budget. Then run
   `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check` in bob-plugins, and
   `just docs-gen` plus `just docs-check` in bob-cli. The bob-cli site mounts the plugin
   docs.

9. **Evidence.** Store the final takeover report, log, screenshot archive, and the PNG
   verdict list with `sase artifact create -b bob-cli-qi -l <label> -p <path>`. Run it
   from the bob-cli checkout so the bead store resolves. Each call takes about 80s, so
   bundle files into one archive where you can.
10. **Close out epic `bob-cli-qi`.** This is the final step.
    1. Run `sase bead epic-symbols bob-cli-qi`. Resolve every listed `--epic-symbol`
       entry by wiring it up, privatizing it, adding a non-test pragma, or deleting it,
       per the Symvision epic-whitelist policy. The land agent saw none.
    2. Run `just check` in bob-cli if you changed bob-cli. Its only known failure is
       `bob-cli-q9` (the meetings PDF test).
    3. Close the epic:

       ```sh
       sase bead close bob-cli-qi --note "<verification>"
       ```

       In the note, combine the land agent's verified summary below with your new area,
       inbox, and project results, evidence refs, and repair commits.

    4. Run `just symvision` in bob-cli to confirm the whitelist is clean.
    5. Set `status: done` in the frontmatter of the epic's plan file
       (`plan:202611/bank_hub_demotion.md`, the PLAN path that
       `sase bead read bob-cli-qi` prints).
    6. The epic has no `parent_bead`, so stop there.

    Never use `--force` merely to make the close succeed.

## Land agent's verified summary (paste into the close note)

- **Vault.**
  - `bank.md` is untyped, under cash, and taskless; `subscriptions.md` is under cash.
  - `bob bank check` reports 0 errors and 0 warnings; albums, areas, and props are
    clean.
  - No bank row appears in `bob areas list` or `bob capture-targets`.
  - `bob props show bank.md` reports `note_kind: bank` with `["props_v1"]`, no contract,
    and a card-owned `bank` (`==`).
  - `@bank` captures are refused with `bank_holds_no_tasks`, and the preview diagnostic
    is wired.
  - The note catalog checks bank before area.
- **bob-cli.** Commits 478d5233 and c89ba1d4, plus the land fix of the qi.2 props test
  assertion (the update report never carried `note_kind`).
- **bob-plugins.** Commits 8e6a5a61, 19e895f0, and 17a05670, plus the land fix of the
  stale 240q comment.
  - The guard grep is empty, and 210/210 epic suites pass, including the passive proof.
  - Takeover is GO 63/63 in 3 of 3 runs on master. The pre-epic 27447919 run is NO-GO
    with 18 bank/area-hybrid failures.
- **Memory.** `glossary:property-card`, `bank-card`, `area-card`, and `project-card`,
  plus `decisions:bank-hub-is-path-canonical-never-area`, are present.
- **Follow-ups.** Every outcome is in the LAND TRIAGE note on `bob-cli-qi`. The new task
  is `bob-cli-qr`, and `bob-cli-q9` received a +1.
- **Release note gap.** 478d5233 is a CLI/JSON contract break, but it lacks its `!`
  marker. History is not rewritten. The next release is already a breaking bump, but the
  changelog will file it under Added.

## Final report must remind Bryan to

- After the train ships the bank-hub changes, update bob on the Mac and apollo and run
  `bob plugins sync` there. The 478d5233 break will appear under "Added", not
  "Breaking", in the release notes, so consider editing that GitHub Release.
- Close or flip his `bob.md` tasks `bob__bank-not-area` and `bob__bank-card-first`. Do
  not edit them yourself.

## Out of scope

- The home-cards runner baseline (`bob-cli-qr`).
- The meetings PDF test (`bob-cli-q9`).
- Any vault edit, any Bob Mac Capture change, and any change to `bank.md` identity or
  behavior.
- Reopening or editing the canceled `bob-cli-pl` or `bob-cli-q6` beads.
