---
tier: epic
title: Demote bank.md to a single-owner bank hub, then finish the card epics
goal: '~/bob/bank.md is an untyped, path-canonical bank hub under cash. It has exactly
  one card (the Bank Card, gated on props_v1), no area semantics in any repo, and
  no tasks. Every piece of logic that supported the bank/area hybrid is gone. On top
  of that single-owner model, the unfinished work of the canceled bob-cli-pl and bob-cli-q6
  epics is completed: acceptance, polish, cleanup, docs, and deployment.

  '
phases:
- id: vault-hub
  title: Reparent the subscriptions album and untype bank.md in the vault
  depends_on: []
  size: small
  description: 'vault-hub: move subscriptions.md under cash and sync albums while
    bank.md is still an area. Then remove bank.md''s area type, fix its summary and
    prose, sync areas, and prove the albums, areas, bank, props, and capture-target
    outputs are clean through their structured output.'
- id: cli-hub
  title: Remove the bank area contract from bob and guard the no-tasks hub
  depends_on: []
  size: medium
  description: 'cli-hub: drop NoteKind::Bank from the props settings contract (a !
    commit), refuse @bank task captures with a preview diagnostic, and add the bank_hub_task
    and bank_hub_area_type findings to bob bank check. Also fix the starter text and
    the note-catalog order, invert the hybrid tests, untype the hub fixtures, and
    correct the bob-cli docs.'
- id: plugin-single-owner
  title: Delete the bank/area arbitration and restore the single-owner Bank Card
  depends_on: []
  size: medium
  description: 'plugin-single-owner: delete 249db and every bank/area arbitration
    hook in bob-ledger-tools, restore the bank.md home-card exclusion, and gate bank
    on props_v1. Bank becomes a no-settings Property Editor kind whatever bob reports.
    Settle the 240r foot lifecycle by a species-agnostic test, convert the hybrid
    unit and e2e cases, fix the baseline harness blockers, then build, test, and sync.'
- id: takeover-proof
  title: Real-Obsidian takeover acceptance and family screenshot review
  depends_on:
  - vault-hub
  - cli-hub
  - plugin-single-owner
  size: medium
  description: 'takeover-proof: run the production-adapter matrix in isolated real
    Obsidian against a master-built bob, covering area, inbox, project, and the single-owner
    bank card. Prove fail-open, focus, routing, and ^prj integrity, and repair defects
    this work caused. Review light/dark and narrow/wide screenshots across the card
    family and retain all evidence.'
- id: count-sweep
  title: Re-run the retired project-count cleanup in the vault
  depends_on:
  - vault-hub
  - cli-hub
  - plugin-single-owner
  size: small
  description: 'count-sweep: dry-run the landed retire-project-count-keys tool against
    the live vault with a current bob. If any host recreated task_count/open_task_count,
    attribute that host and apply the guarded cleanup, then prove a repeat dry-run
    is a no-op and retain the receipts.'
- id: finish
  title: Passive-rendering proof, docs sweep, approved memory, and clean deploy
  depends_on:
  - takeover-proof
  - count-sweep
  size: medium
  description: 'finish: add the unit proof that folds and refreshes never write the
    vault or spawn bob from paint, and sweep the bob-plugins and bob-cli docs onto
    the single-owner card contract. Apply only the memory decisions answered yes,
    then deploy cleanly from master with the isolated post-sync smoke check.'
decisions:
  hub_finding_severity:
    ask: How severe should bob bank check's new bank_hub_task and bank_hub_area_type
      findings be?
    choices:
      warning: Reported, but bank check still exits 0 when they are its only findings
      error: A stray hub task or an area type makes bank check fail
    default: warning
    why: The @bank refusal already blocks captures, so this finding only catches hand
      edits
    answer: warning
  memory_card_glossary:
    ask: Update glossary:property-card (area/project species) and glossary:bank-card
      (untyped, taskless hub)?
    memory:
    - glossary:property-card
    - glossary:bank-card
    default: true
    corrects: The shared family of note-top album, record, day, period, and bank cards
    answer: true
  memory_home_card_strands:
    ask: Add glossary:area-card and glossary:project-card strands (approved earlier
      for the canceled pl epic)?
    memory:
    - glossary:area-card
    - glossary:project-card
    default: false
    answer: true
  memory_bank_hub_record:
    ask: Add a decisions strand recording that the bank hub is path-canonical and
      is never an area note?
    memory:
    - decisions
    default: false
    answer: true
proposed_by: bbugyi200.athena.1gn
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 10:06:11
status: done
bead_id: bob-cli-qi
---

- **PROMPT:** [prompts/202611/bank_hub_demotion.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_hub_demotion.md)
- **BEAD:** [bob-cli-qi](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qi/README.md)

# Demote bank.md to a single-owner bank hub, then finish the card epics

## Why

Bryan canceled `bob-cli-pl` (area and project cards) and `bob-cli-q6` (bank as a proper
property card) because `~/bob/bank.md` should never have been an area note. It is a
child of `cash.md` used only for bank row tracking, and it never holds tasks. The two
epics gave `bank.md` two title-, foot-, and frontmatter-owning cards (Area and Bank).
Bryan's 2026-11-26 screencast shows the result: the cards flip on cursor movement and
stack on `_`, with duplicate `Properties · 5` feet. The `pl.9.1.2.x` arbitration layer
added to reconcile them never reached a GO.

Source of requirements: `research:202611/bank_hub_demotion/bank_hub_demotion.md`. Bryan
accepted all of its requirements except the memory changes, which are gated by this
plan's memory decisions. The sibling reports in the same directory hold the detailed
evidence:

- `__cld` §4: per-file removal inventory for both repos.
- `__gem` §2: the screencast frames and the phase-by-phase landed/canceled audit.
- `__gem` §5: per-file removal manifest and the vault edit sequence.
- `__cdx`: the disposable-vault experiments.

Read the consolidated report through `sase artifact read` before starting any phase.
Read a sibling report when your phase needs its manifest.

## Target state

- `bank.md` has no `type:`. It keeps `parent: "[[cash]]"`, which is only display
  affinity (the `BANK · Cash` kicker and cash's magenta hue), not an area-graph edge. It
  keeps its `bank:` mapping, queue fence, reports, and navigation. Monthly notes stay
  `type: "[[bank_month]]"` with `parent: "[[bank]]"`. Nothing is moved or renamed, and
  no ledger or journal is migrated.
- `subscriptions.md` (the subscription album) has `parent: "[[cash]]"`. Subscription
  matching is unchanged, because it binds by the `subscription` noun, never by parent.
- bob identifies the hub by path only (`props/roles.rs`, `bank/scan.rs`, and the
  plugin's `bankSetupIsHubPath`):
  - `bob props show bank.md` reports `note_kind: bank` with `props_v1`, has no contract
    block, and keeps `bank` card-owned (`==`).
  - `setting.*` ops on `bank.md` refuse.
  - `bob areas list`, the `## Areas` catalogs, and `bob capture-targets` have no bank
    row, and `//` classifies `bank.md` as Bank.
- A task capture routed at `@bank` is refused with a hint toward `@cash`.
  `bob bank check` reports a `#task` in the hub and an area/project `type:` on the hub.
- Exactly one card mounts on `bank.md`: the Bank Card, with one title, one foot, and
  Setup folded by `_`. Native Properties hide only behind the `props_v1` fail-open gate.
  `+` opens the generic Property Editor, where the `bank` row hands off to `==`, and
  shows no AREA SETTINGS.
- No code path in either repo names a Bank/Area owner, `suppressBankFoot`, or a bank
  area contract.

## Governing decisions

- `decisions:bob-cli-releases-ride-a-daily-train`: the props contract change and the new
  `@bank` capture refusal are CLI/JSON contract breaks, so mark those commits `!`. Never
  edit the `Cargo.toml` version or `CHANGELOG.md`.
- `decisions:mac-capture-is-a-thin-client`: the refusal and its preview diagnostic live
  only in bob, documented in `docs/capture.md`. Bob Mac Capture renders them unchanged.
- `decisions:capture-sigils-say-where-what-how`: `@` names a place, and `@bank` now
  names a place that holds no tasks, as albums and notebooks already do.
- `decisions:notebook-routes-write-notes`: its rejected "sub-areas drag in Ready caps,
  `## Tasks`, Stats roots, and tasks" is the precedent for not modelling a taskless note
  as an area.
- `decisions:note-ready-cap-counts-the-lane`,
  `decisions:off-clock-follows-tag-then-note-default` and
  `decisions:off-clock-note-defaults-filter-by-task-type`: `bank.md` stops being a
  Ready-cap and clock-default home. It declares none of those keys and holds no tasks,
  so no effective value changes. The area/project cards must keep these semantics.
- `decisions:projects-block-from-prj-line`, as amended by
  `decisions:subprojects-inherit-project-blocking`: the takeover never moves, hides,
  duplicates, or writes a `^prj` line.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the Property Editor keeps its shared close
  ladder.

## What already landed and stays

The work below landed on master before the cancel. Do not rebuild it and do not revert
it:

- pl.1–pl.8: one frontmatter writer, the props roles/ops/contract for area and project,
  the Area and Project cards, report-backed readouts, and the settings stages.
- pl.9.1.1: the home-card `props_contract_v1` ownership gate and native entry routing.
- q6.1–q6.3 and q6.5: the fold kit and `_`, the bank Setup fold, the title-slot Bank
  Card, and `_` on the record, area, and project cards.
- q6.4: the bank takeover code and its glossary.

Keep these as they are in bob-plugins:

- the title-slot Bank Card identity, the `BANK · Cash` crumbs, and the summary subtitle;
- the hue walk;
- `==` precedence (`248m`);
- the single-owner bank frontmatter takeover (the `248cc` core and the bank paths in
  `240q`);
- the bank foot's own `nested.bank` row;
- `248cd` lint deltas;
- `homeCardIsDesktopPlatform`;
- the kind-scoped releases;
- the Vim cursor redraw fixes;
- the generic e2e pane fixes.

Do not reopen, edit, or close the canceled beads `bob-cli-pl`, `bob-cli-q6`, or their
descendants. Read them with `sase bead read -r` as evidence only. Bryan's own vault GTD
tasks in `bob.md` (`bob__bank-not-area`, `bob__bank-card-first`) are his. Do not edit
them.

## Shared rules for every phase

- **Repos.** bob-cli is the primary checkout. Open bob-plugins with `/sase_repo`
  (`sase repo open bob-plugins`), use only the path it prints, and read its `AGENTS.md`.
  Before any vault work, read the `obsidian.md` reference memory with
  `/sase_memory_read`. Vault notes live in `~/bob/` and are edited in place, and
  vault-sync owns their commits. Never open the live vault in an automated Obsidian UI
  run.
- **Code anchors drift.** The file:line anchors below come from bob-cli `4dd787bc` and
  bob-plugins `1770259c`. Re-locate each symbol on current master before editing.
- **bob-plugins.** Edit fragment sources, never the generated `main.js`. Remove code
  with forward commits, never `git revert`: the arbitration commits also contain work to
  keep, and reverting would mean hand-merging the bundle. For each plugin change:
  1. Bump the manifest per the repo convention.
  2. Run `npm run build`, `npm run build:check`, the focused suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`. If sync refuses an overwrite, follow
     `AGENTS.md` and `docs/develop/deployment.md`.
- **bob-cli.** Run `just check`. If you touch docs, also run `just docs-gen` and
  `just docs-check`. If a doc you edit is already over its word budget on the base, your
  net change must not grow it. Budget failures in untouched docs that reproduce on the
  clean base are recorded, not fixed.
- **Long commands and waiting.** Use `/sase_monitor` for long builds, real-app runs, and
  sweeps. Finish through `/sase_final`.
- **Beads.** Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on
  your own phase bead instead, including any baseline failure that reproduces on clean
  master (with exact commands and SHA). Record memory corrections you discover that no
  accepted decision covers the same way; do not edit that memory. Run
  `sase bead epic-symbols <phase>` before closing.

## Version skew to design for

`bob plugins sync` deploys at once. bob-cli changes reach hosts only through the daily
train or a local build, and the Mac can lag further. So the plugin must never trust the
backend's contract for bank: it renders the generic editor and gates on `props_v1` for
`note_kind: bank` even when an older bob still sends the hybrid AREA SETTINGS contract.
The opposite skew is also safe: a new bob with the old plugin drops bank's
`props_contract_v1`, so the old `bankHub` gate fails open to native Properties.

The vault edit is safe to land first. With the currently deployed plugin, an untyped
`bank.md` already fails `homeCardEligible`, so the Area Card stops mounting at once.
Until the code phases deploy, the only leftover is `+` showing AREA SETTINGS.

## Phase `vault-hub`

Order matters, so that no point in the edit leaves an invalid album:

1. **Baseline.**
   - Confirm `bob vault-sync status --json` is healthy. Check `git -C ~/bob status` for
     unrelated in-flight edits to `bank.md`, `subscriptions.md`, `cash.md`, or
     `area.md`; stop and record if there are any.
   - Save JSON baselines of `bob albums list`, `bob albums check`, `bob areas list`,
     `bob areas check`, `bob bank check`, `bob bank subscriptions`, and
     `bob props show bank.md`, plus `bob capture-targets` output.
2. **Reparent the album.** In `subscriptions.md`, change only `parent: "[[bank]]"` →
   `parent: "[[cash]]"`.
3. **Sync albums.** Run `bob albums sync` while `bank.md` is still an area. It removes
   the bob-owned `## Albums` region from `bank.md` and creates one on `cash.md`.
   - Inspect the diff: only those regions should change.
   - Remove any heading left empty in `bank.md`.
4. **Untype `bank.md`.** Make byte-careful edits:
   - Delete the `type: "[[area]]"` line.
   - Change `summary:` to `Every transaction and deposit, categorized in monthly notes.`
   - Replace "Money tasks like disputes and cancellations live here too." with a
     pointer, for example:
     `Money tasks live in [[cash]]; recurring charges live in [[subscriptions|🔁 Subscriptions]].`
   - Leave `parent`, `created`, the `bank:` mapping, and the queue fence untouched.
5. **Sync areas.** Run `bob areas sync`. It drops `[[bank]]` from the `## Areas`
   catalogs on `cash.md` and `area.md`. `cash.md`'s hand-written `Related: [[bank]]`
   line stays.
6. **Verify through structured issues, not exit codes.** `bob albums list` can exit 0
   with errors present.
   - Albums: no `album_parent_kind`, and Subscriptions is grouped under cash.
   - Bank: `bob bank check` has no subscription-album-invalid finding, and
     `bob bank subscriptions` resolves the same records and totals as the baseline.
   - Areas: `bob areas list` has no bank row, and `bob areas check` is clean.
   - Props and capture: `bob props show bank.md` still reports `note_kind: bank`, and
     `capture-targets` no longer lists `bank`.
   - Diff every output against its baseline, and treat any unexplained change as a phase
     failure.
7. Let vault-sync commit, then confirm with `bob vault-sync status --json` that the
   change was published. Record the before/after summaries on the phase bead.

This phase changes no repository. Its evidence is the bead note.

## Phase `cli-hub`

1. **Props contract** (commit as `fix(props)!: …`).
   - In `src/native/props/contract/settings.rs` `for_kind`, change
     `NoteKind::Area | NoteKind::Bank` to `NoteKind::Area`. `bob props show bank.md`
     then reports `props_v1` with no contract block. `bank` stays `role: card`
     (`owner: bank`, `editor: ==`), and `parent`/`summary`/`created` stay loose.
   - `setting.*` ops on `bank.md` refuse through the existing refusal for a kind with no
     settings.
   - Update the doc comments in `contract/read.rs` and the `long_about` in
     `props/cli.rs`.
   - Invert the hybrid tests into assertions that bank has no contract, keeps a
     card-owned `bank` key, and refuses `setting.*` without writing:
     - `contract/tests.rs` (`home_and_bank_roles_match_the_editor_contract`,
       `bank_show_uses_the_area_contract`);
     - `tests/cli/props_contract.rs` (`bank_hybrid_and_refused_batches_never_write` and
       its sibling bank assertions);
     - the bank fixture in `props/roles.rs` tests.
   - Keep the path classification in `roles.rs` and `bank` → `card("bank")`.
2. **Refuse task captures into the hub** (commit as `feat(capture)!: …`).
   - Add a code constant `bank_holds_no_tasks` beside `ALBUM_HOLDS_NO_TASKS_CODE`. Add a
     message builder beside `album_holds_no_tasks_error` /
     `notebook_holds_no_tasks_error`, for example: "bank.md is the bank hub and holds no
     tasks — capture money tasks with @cash".
   - Return it as a `CaptureError::usage_code` wherever `album_task_home_refusal` runs
     in `src/native/capture/plan.rs`. That covers the `--route` / `@@` draft-wide path,
     the per-item path, and the snooze/Pomodoro-link path.
   - Gate the refusal with `is_album_guarded_task_kind`, so it covers the same task
     kinds as the album guard.
   - Identity is the vault-root route `bank` (`bank.md`), whatever its frontmatter says.
     Add one small shared helper for the hub path instead of another literal.
   - Add the matching live-preview diagnostic in
     `capture_parse.rs::collection_route_diagnostics`.
   - Tests:
     - `bob capture "@bank Buy stamps"` refuses with the code and writes nothing.
     - `--dry-run` and `capture-parse` report the same code.
     - An area-typed hub is still refused.
   - Existing protections need only regression assertions:
     - task move, `>` transfer, and inbox `--move-in` refuse non-area destinations;
     - merge sections require area/project;
     - project-note capture validates its parent.
   - Document the refusal in `docs/capture.md`. Then use `/sase_repo` to confirm, read
     only, that Bob Mac Capture renders an unknown diagnostic code generically (it
     special-cases only `album_holds_no_tasks`). Record a follow-up if it does not.
3. **`bob bank check` findings** (`src/native/bank/check.rs`).
   - `bank_hub_area_type`: the hub's `type:` resolves to `[[area]]` or `[[project]]`.
   - `bank_hub_task`: any `#task` checkbox line in `bank.md` outside fenced code. Reuse
     the existing task-line detection, and hint that money tasks live in `cash.md`
     (`@cash`).
   - Add unit and CLI tests, and list both codes wherever the docs list bank check
     codes.

   > [!decision] hub_finding_severity = warning Both findings use `Severity::Warning`.
   > `bob bank check` still exits 0 when they are its only findings.

   > [!decision] hub_finding_severity = error Both findings use `Severity::Error`.
   > `bob bank check` fails under its existing error contract. Document that a
   > hand-added hub task now fails the check.

4. **Starter text.** In `bank/output.rs` `print_teaching_empty`, remove the
   `type: "[[area]]"` line from the paste-ready start, and update its test or snapshot.
5. **Go To Notes order.** In `src/native/note_catalog/mod.rs`, check `is_bank_note`
   before `facts.is_area`. Daily and inbox stay first. Add a test that an area-typed
   `bank.md` still classifies as Bank.
6. **Fixture hygiene.** Remove `type: "[[area]]"` from the hub fixtures in the bank and
   walks suites. That covers `tests/cli/bank.rs`, `tests/cli/walks.rs`, and the
   `src/native/bank/` tests (`import_run`, `import_plan`, `config`, `naming`, `tests`,
   `charts_tests`, `config_spans_tests`, `config_update/tests`). This proves bank works
   untyped. The only area-typed hub kept is the regression test for
   `bank_hub_area_type`.
   - Rename the `bank` sub-area example in the `src/native/areas/{sync,report}.rs` tests
     and in `docs/areas.md` to a neutral area name.
7. **Docs.**
   - `docs/bank.md` `## bank.md`: an untyped hub under cash, no tasks, money tasks in
     `cash.md`, `@bank` refused, and the subscriptions album parented to cash. Fix the
     example frontmatter.
   - `docs/props.md`: remove the hybrid paragraph and the "bank uses the area order"
     lines.
   - `docs/reference/vault-format.md` and `docs/reference/json-outputs.md`: fix the bank
     contract text.
   - Grep `docs/` and `src/` help text for "sub-area" or "area" wording about bank and
     fix every hit.
8. Run `just check`, `just docs-gen`, and `just docs-check`.

## Phase `plugin-single-owner`

All paths are under `plugins/bob-ledger-tools/` unless noted. The full manifests are in
`__cld` §4.2 and `__gem` §5.2.

1. **Delete the arbitration.**
   - Delete `src/249db-plugin-bank-area-arbitration.js`, its `src/fragments.json` entry,
     and its `src/374g-exports-home-card.js` exports.
   - Remove every bank/area register, release, generation, and repaint hook from:
     - `249d` (including the `noteKind: "bank"` mapping; return `homeCardOpenSetting` to
       area|project);
     - `249da`;
     - `249e` (the nested `bank` row in the Area foot, and the Bank Setup repaint);
     - `248c` (`suppressBankFoot`, the arbitration options, the `arbitration.owner`
       anchor hack, and the `scheduleHomeCardRefresh` cross-call);
     - `248ca` (the suppression and the generation in the foot signature);
     - `248cb` (the title yield);
     - `248cc` (`bankSetupAreaFrontmatterEligible` and every
       `propertyCardHomeBankAreaCurrent` branch).
   - In `249ca`, delete `homeCardPropertyEditorNoteKind` and keep
     `homeCardIsDesktopPlatform`.
2. **Defensive single owner.** Put `"bank.md"` back in the `249b` home-card exclusion,
   with a comment explaining that `bank.md` is not an area. Even a stale deploy or an
   accidentally restored type then cannot mount a second card.
3. **Bank is a `props_v1` species again.**
   - In `240q`, drop the `bankHub` `props_contract_v1` requirement. Bank gates on
     `props_v1`, as album, record, and temporal do, and area/project keep
     `props_contract_v1`.
   - Bank is a no-settings kind whatever bob reports:
     - `298w`: remove bank from `PROPERTY_EDITOR_SETTING_ORDER` and from the fallback
       hint;
     - `298x`: remove `"bank"` from the setting kinds;
     - `298m`: remove the bank bare-`+` AREA SETTINGS options.
   - Bare `+` on `bank.md` opens the generic Property Editor, and its `bank` row hands
     off to `==`.
4. **Foot lifecycle (`240r` / `374f` `propertyCardFootAdapterPruneDisconnected`).**
   Write a species-agnostic test, for example two panes on one album or record note,
   with pane close and reopen. If the test proves the prune fixes a real non-bank bug,
   keep it and add a comment naming that bug. Otherwise revert to whole-scope release.
   Record the outcome on the phase bead.
5. **Tests.**
   - Unit suites:
     - Remove the arbitration cases from the home-card-host suite.
     - Invert the home-card-model case: an area-typed `bank.md` is not home-eligible.
     - Convert the bank cases in the Property Editor entrypoint and settings-entrypoint
       suites to the generic editor.
     - In the frontmatter-ownership suite, bank gates on `props_v1`. Include a case
       where bob reports the old hybrid contract and the editor still shows no settings.
   - In `scripts/obsidian-frontmatter-takeover-e2e*.cjs` and their offline wrapper:
     - Remove the `bank-area-*` checks and probes and `bankSettingsFailures`.
     - Untype the bank fixture. It keeps byte-copied live frontmatter shapes:
       flow-mapping categories, quoted digit hotkeys, bare dates, an offset timestamp,
       and `[[…]]` strings.
     - Convert the bank cases to single-card assertions: one title, one foot.
   - In the property-editor e2e settings, replace "bank plus keeps Area settings hybrid"
     with the generic-editor expectation.
   - Fix the baseline harness blockers:
     - the `modalLifecycleFailures` self-test fixture in
       `obsidian-frontmatter-takeover-e2e-lib.cjs`
       (`visible: true, activeInside: true`);
     - the bank Vim `k`/`gg` case, which must place the cursor on visible body text
       instead of inside the hidden YAML. Do not weaken the ownership or cursor
       assertions.
6. **Guard grep.**
   `rg -n "BankArea|bankArea|suppressBankFoot|bankSetupAreaArbitration|bankSetupAreaFrontmatterEligible|propertyCardHomeBankArea|homeCardPropertyEditorNoteKind"`
   over `plugins/`, `scripts/`, and `docs/` returns nothing.
7. **Docs edits for this phase only.**
   - `docs/property-editor.md`: bank lines.
   - `docs/develop/validation.md`: remove the bank/area arbitration text.
   - `docs/area-and-project-cards.md`: the exclusion and its reason.

   `finish` owns the full sweep.

8. Build, test, and sync as the shared rules describe. Syncing from clean master also
   clears the recorded `bob-ledger-tools` deploy drift.

## Phase `takeover-proof`

This phase replaces the canceled pl.9.1.3 and q6.4 acceptance and the q6.6 screenshot
review.

**Setup.**

- Use a bob built from current bob-cli master, passed through `BOB_BIN` and the runner's
  `BOB_CLI_DIR`/provenance contract.
- Use an isolated Obsidian profile and process with a disposable Git vault. If the run
  uses athena, first read the tailnet reference memory and follow its isolated route.
- Extend `scripts/obsidian-frontmatter-takeover-e2e.cjs` and
  `scripts/obsidian-home-cards-e2e.cjs` rather than manufacturing ownership in tests.

**Required evidence.**

| Concern             | Evidence                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Normal takeover     | LP and Reading for the top-level area, sub-area, inbox, active project, terminal project, and the bank hub. Each shows exactly one card, one title, one foot, and native Properties hidden.                                                                              |
| Bank single owner   | Bank stays single-owner even when the fixture hub carries `type: [[area]]`. `_` toggles Setup, `-` toggles the drawer, and `==` opens Bank Setup. `+` opens the generic editor with no AREA SETTINGS, and its `bank` row hands off to `==`.                              |
| Native restoration  | Native Properties return for Source, embed, hover, canvas, print, mobile emulation, bad/duplicate YAML, an unclosed fence, source/cache mismatch, a path change, a failed mount, and unload. Check each at the transaction/frame boundary.                               |
| Capability          | Pending, absent, failed, `props_v1`-only bob, an old bob that still sends bank the hybrid contract, the ordinary-note initial probe, a bob-path change, and stale or late replies.                                                                                       |
| Ownership stability | Autosave/reparse, replaced metadata containers, Reading reattachment, split panes (two panes on `bank.md`), and bank refresh. No duplicate title or foot.                                                                                                                |
| Focus and routing   | Vim `k`/`gg` and Up at body start, Tab/Shift+Tab, typing after navigation, Mod+`;`, a custom binding, command dispatch, the summary ghost, Settings and chips, opener failure, and the native File Properties sidebar. The editor's Esc/`q`/Ctrl+`[` ladder still works. |
| Project integrity   | Cursor, selection, and IME on `^prj`. Task Card and close gestures still target the real line.                                                                                                                                                                           |
| Exclusions          | Untyped `area.md`/`project.md`, templates, `done/`, `_conflicts/`, and ordinary notes stay unchanged.                                                                                                                                                                    |
| Passive behavior    | Protected fixture hashes stay unchanged. Lifecycle probes are allowed; paint-, drawer-, collapse-, or resize-triggered subprocesses are not.                                                                                                                             |

**Screenshots.** Capture light/dark × narrow/wide in LP and Reading for the bank, album,
record, area, inbox, and project (active and terminal) cards. Also capture
representative native-recovery and Source states.

- Inspect every image and record a verdict, size, and digest for each.
- Fix visual drift: row alignment, chip placement, keycap fade, narrow-split wrapping,
  and the folded bank card's height beside a folded album.

**Evidence and repairs.**

- Retain the raw and inspected reports, screenshots, console logs, source, binary, and
  plugin hashes, and the exact commands, with `sase artifact create --bead`.
- Offline self-tests or a skipped real-app run are not acceptance.
- Repair defects this work caused in bob-plugins (build, test, sync), then rerun the
  affected rows.
- The phase closes only on GO. Unrelated baseline failures become follow-ups with a
  reproduction on clean master.

## Phase `count-sweep`

This phase finishes pl.9.1.4's cleanup half. Read `obsidian.md` first. Then run the
landed tool from the opened bob-plugins checkout, using a current bob:

```sh
BOB_BIN=<current-bob> node scripts/retire-project-count-keys.cjs --dry-run --bob-dir ~/bob
BOB_BIN=<current-bob> node scripts/retire-project-count-keys.cjs --apply --bob-dir ~/bob
```

1. Run the dry-run first.
2. If keys came back:
   - Find the responsible host from deploy and sync evidence (`bob plugins list` drift
     on each host). The earlier recurrence came from a stale Mac writer; do not assume
     it is the same one.
   - Apply through the guarded props transactions. They remove only the two retired keys
     and preserve concurrent edits.
   - Handle stale fingerprints and lock-busy by re-reading and retrying, with a bound.
     Never overwrite.
3. Prove that a repeat dry-run is a no-op at a named vault snapshot.
4. Retain the preimages and receipts. The prior receipt is
   `file:explicit:7e981553181fd23c81d6b221`; read it with `sase artifact read`.
5. Report honestly: no permanent absence is promised while any host can still recreate
   the keys.

## Phase `finish`

1. **Passive-rendering proof (q6.6).** Add a unit test that toggles the bank, album,
   record, area, and project folds many times and forces repeated refreshes. It asserts
   zero vault writes and zero new paint-triggered bob subprocesses; bank config
   refreshes stay subscription-driven. Register the suite in `package.json`.
2. **Docs sweep.**
   - bob-plugins: `docs/area-and-project-cards.md`, `docs/property-editor.md`,
     `docs/bob-ledger-tools.md`, `docs/develop/architecture.md`,
     `docs/develop/validation.md`, and stale lines in `docs/bob-project-tasks.md`.
     Cover:
     - the capability gates (`props_v1` for bank, album, record, and temporal;
       `props_contract_v1` for area and project);
     - the exclusions and their reasons;
     - fail-open behavior;
     - the single foot;
     - one consistent key family: `-` drawer, `_` own section, `==` species editor, `+`
       Property Editor;
     - `takeover-proof`'s exact tested matrix and evidence refs.
   - Regenerate the mounted plugin docs in bob-cli with `just docs-gen`, and recheck
     `docs/bank.md`.
   - Nothing may still describe bank as a sub-area, hybrid, or Area/Bank split. Grep
     both repos to confirm.
3. **Memory.** Apply only the decisions answered yes. Use `/sase_memory_write`, link
   `docs/reference/plugins/area-and-project-cards.md` where relevant, then run
   `sase memory init`, `just docs-gen` (which regenerates `docs/reference/glossary.md`),
   and `just docs-check`.

   > [!decision] memory_card_glossary `glossary:property-card`: the family includes the
   > area and project cards, after the existing species and before extensions in the
   > registry. Area and project need `props_contract_v1` on top of `props_v1`. Bare `-`
   > and `+` stay. `bank.md` has one owner, the Bank Card on `props_v1`, with no AREA
   > SETTINGS. Link any new card strands. `glossary:bank-card`: describe the title-slot
   > card instead of "Under `# 🏦 Bank`". State that `bank.md` is an untyped hub under
   > cash that never holds tasks, and that `+` opens the generic Property Editor.

   > [!decision] memory_home_card_strands Add `glossary:area-card` and
   > `glossary:project-card`. Each covers its identity, crumbs, title, summary, authored
   > policies and live readouts, Settings, read-only paint, and native recovery.
   > `area-card` states that `bank.md` is excluded because it is not an area.
   > `project-card` covers lifecycle from the Project Task and the unchanged `^prj`
   > control.

   > [!decision] memory_bank_hub_record Add a decisions strand, "The Bank Hub Is
   > Path-Canonical And Never An Area Note", in the web's record format. Cover where it
   > applies, the claim, why it beat a typed hub, a `bank_hub` type, and keeping the
   > area type, the cost (`@bank` is gone and subscriptions moved under cash), and what
   > would reopen it. Cite the cancel reason on `bob-cli-pl` and `bob-cli-q6`, the
   > research report, and this epic's commits.

4. **Deploy.** Run `npm run build:check` and `npm test`. Then run `bob plugins sync`
   from clean master, followed by the isolated post-sync startup smoke check in
   `docs/develop/deployment.md`. `bob plugins list` must show no `bob-ledger-tools`
   drift. Record provenance on the bead.

## Epic acceptance (land agent)

The land agent verifies the whole target state above against live evidence from every
phase:

- **Vault.**
  - `bank.md` is untyped under cash, and `subscriptions.md` is under cash.
  - The albums, areas, and bank checks are structured-clean, and subscriptions still
    match.
  - No bank row appears in areas or capture targets, and `//` reports Bank.
- **bob.**
  - `props show bank.md` reports `props_v1` with no contract.
  - `@bank` task captures are refused, with the preview diagnostic.
  - Both hub findings are tested.
- **Plugins and acceptance.**
  - `takeover-proof` is GO, with one Bank Card on `bank.md` and no AREA SETTINGS from
    either repo.
  - The guard grep is empty.
  - The passive proof passes.
- **Area and project cards.** They meet the original pl acceptance: typed settings, zero
  paint writes or spawns, and fail-open.
- **Docs and memory.** Docs are consistent, memory matches the answered decisions, and
  the deploy is clean.

The land report reminds Bryan to:

- update bob on Mac and apollo once the train ships the `!` commits, and run
  `bob plugins sync` there;
- close or flip his `bob.md` tasks `bob__bank-not-area` and `bob__bank-card-first`.

## Out of scope

- The pre-existing nav-hotkeys `PARENT_RULE_NON_RECORD_TYPES` omission of `bank_month`
  is tracked separately.
- Moving `## Albums`/`## Notebooks` catalogs into the cards.
- A `bank_hub` type note.
- Any `bank.md` file move or rename.
- Any Bob Mac Capture code change.

## Risks

- **A half-landed vault edit could silently break subscriptions.** The reparent and
  albums sync come first, and acceptance asserts on structured issues and subscription
  totals.
- **The hybrid could regrow.** Four cheap guards cover it: the starter-text fix, the
  `bank_hub_area_type` finding, the restored plugin exclusion, and the optional
  decisions strand.
- **Changing `240r` could regress other species.** The phase decides it with a
  species-agnostic test, not a guess.
- **Real-app acceptance has failed before.** Most of those failures came from two owners
  on one note. Any repair stays inside `takeover-proof`, and the phase does not close
  without GO.
