---
tier: epic
title: Shorter review footer labels and the Returned → Tickler rename
goal: 'The morning GTD review footer shows WIP, TICKS, and REFS instead of PENDING,
  RETURNED, and REFERENCES; the Returned walk tier is renamed Tickler across bob-cli,
  bob-plugins, the Bob vault, and SASE memory; and a new glossary term defines the
  review footer.

  '
phases:
- id: cli-tickler
  title: bob-cli rename, schema 10, and docs
  depends_on: []
  size: medium
  description: 'cli-tickler: rename the Rust returned walk tier to tickler (TICKLER
    in human output), bump bob freshness JSON to schema 10, update tests and the D4
    fixture vector, and document the tickler tier, the WIP alias, and the footer short
    labels in bob-cli docs and README.'
- id: plugins-tickler-footer
  title: bob-plugins rename, footer short labels, and deploy
  depends_on: []
  size: medium
  description: 'plugins-tickler-footer: in linked bob-plugins, rename the returned
    tier to tickler in bob-ledger-tools and bob-navigation-hotkeys (freshness namespace
    v8, nav legacy read), add WIP/TICKS/REFS footer labels with a legend tooltip,
    update tests and README, bump both plugin versions, build, test, and run bob plugins
    sync.'
- id: vault-and-memory
  title: Vault notes, glossary term, memory updates, and final sweep
  depends_on:
  - cli-tickler
  - plugins-tickler-footer
  size: small
  description: 'vault-and-memory: rename RETURNED to TICKLER on live Bob vault surfaces
    (rotten.md heading and links included), add the Morning GTD Review Footer glossary
    term, update the task-freshness and keep-streak strands and four decision records
    in place, regenerate memory with sase memory init, and sweep all three repositories
    for leftover references.'
proposed_by: bbugyi200.apollo.5i
create_time: 2026-10-07 09:55:35
status: wip
bead_id: bob-cli-55
---

- **PROMPT:** [prompts/202610/review_footer_short_labels_tickler.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/review_footer_short_labels_tickler.md)
- **BEAD:** [bob-cli-55](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-55/README.md)

# Shorter review footer labels (WIP / TICKS / REFS) and Returned → Tickler

## Goal

Shorten the morning GTD review footer, the bob-ledger-tools item in Obsidian's bottom
status bar, by giving it compact group labels:

- `WIP` instead of `PENDING`. WIP becomes an official alias of the Pending (`[/]`, In
  Progress) task status.
- `TICKS` instead of `RETURNED`.
- `REFS` instead of `REFERENCES`.

Also rename the "Returned" walk tier to **Tickler** everywhere: bob-cli, bob-plugins,
the Bob vault, and SASE memory. Then add a glossary term for the review footer.

## Shared vocabulary (all phases follow this table exactly)

| Walk tier                              | Machine key                | Full label (CLI, `]s` notices, docs, vault) | Footer label       |
| -------------------------------------- | -------------------------- | ------------------------------------------- | ------------------ |
| Pending lane                           | `pending` (unchanged)      | `PENDING` (unchanged)                       | `WIP`              |
| Tickler (was Returned)                 | `tickler` (was `returned`) | `TICKLER` (was `RETURNED`)                  | `TICKS`            |
| References                             | `references` (unchanged)   | `REFERENCES` (unchanged)                    | `REFS`             |
| PRE, NEW, PROJECTS, NEXT, ROTTEN, POST | unchanged                  | unchanged                                   | same as full label |

Rules:

- **Tickler wording.** Use "tickler task(s)", "a tickler", or the TICKLER tier where
  text used to say "returned task", "returned deferral", or the RETURNED tier. The GTD
  tickler idea fits: a deferred task comes back when its `scheduled` date arrives.
  Detail text such as `back since Oct 7` stays as it is. Only the bare fallback word
  `returned` becomes `tickler`.
- **The machine Ready _state_ is unchanged.** Keep `resurfaced`,
  `FreshState::Resurfaced`, `counts.resurfaced`, the
  `api.freshness.state(task) === "resurfaced"` filters in `rotten.md`, and the
  freshness-mark text `Resurfaced {date}`. The state is a different axis from the tier,
  and docs already split them ("human vocabulary says X; the machine `state` stays
  `resurfaced`"). That doc sentence now says "tickler".
- **WIP is an alias, not a rename.** Pending stays the canonical status and lane name.
  Keep `pending_interval`, the `pending` machine key, PENDING dashboard chips/sections,
  `bob plan` output, CLI output, `]s` notices, and "Still pending?" hints as they are.
  No CLI option or config key takes lane names as input, so there is no parser to
  extend. "Official" means three things: the footer shows `WIP`, the status table in
  `docs/getting-started.md` lists it, and the glossary term names it. (Vault chips
  already say `next/wip`.)
- **REFS is footer-only.** REFERENCES stays everywhere else.
- **Breaking-version signals, not legacy aliases.** bob-cli bumps `bob freshness` JSON
  `schema_version` 9 → 10, with no `returned` alias. The bob-ledger-tools
  `api.freshness` namespace goes from version 7 to 8. bob-navigation-hotkeys reads a
  legacy `tier: "returned"` from a v7 ledger api as `tickler`, so a mixed deploy still
  walks correctly. That one read-side alias is the only legacy shim.
- **Out of scope:** renaming `resurfaced`; changing PENDING or REFERENCES outside the
  footer; dashboard chip labels; config keys; Bob Mac Capture, which is a thin client of
  `bob capture` and does not read freshness tiers; and closing Bryan's own `bob.md`
  tasks ("Make status row smaller!", "Rename scheduled/RETURNED to tickler/TICK?!"),
  which stay his to close.

## Review footer design (implemented in phase `plugins-tickler-footer`, documented in `cli-tickler`)

- Footer groups use the footer label of each nonempty tier, in walk order:
  `PRE 7 · NEW 1 · PROJECTS 2 · WIP 10 · NEXT 15 · TICKS 14 · REFS 3 · ROTTEN 26 · POST 1`.
- The current-row context also uses the footer label, for example
  `Review 5/76 · WIP 3/10 · confirmed yesterday`.
- `]s` jump notices and `api.freshness.reviewEntryView(...).label` keep full labels
  (`TICKLER 2/14`, `PENDING 3/10`). Notices have room, and the CLI and docs use the same
  full names.
- The tooltip gets one legend line, but only when the view shows an abbreviated label
  (groups or current context). It lists only the abbreviations shown, in walk order:
  `WIP = PENDING · TICKS = TICKLER · REFS = REFERENCES`.
- The existing tooltip line becomes
  `Footer splits TICKS from ROTTEN; dashboard ROTTEN chips still fold both.`
- Fit order, hide rules, click action, modes, and the meter are unchanged.

## Phases

### Phase `cli-tickler` — bob-cli rename, schema 10, and docs

- **id:** `cli-tickler`
- **size:** medium
- **depends on:** none
- **repo:** bob-cli (the primary workspace checkout)

Steps:

1. `src/native/freshness/state.rs`
   - Rename `Tier::Returned` to `Tier::Tickler`, with `as_str` returning `"tickler"`.
   - Rename `ByTier.returned` to `ByTier.tickler`, and update `sum()`, the
     `Some(Tier::Returned)` match arms, and the comparator arm.
   - In comments, change the RETURNED tier to TICKLER and "rotten/returned" to
     "rotten/tickler". Keep every `Resurfaced` state identifier.
2. `src/native/freshness/cli.rs`
   - Bump `SCHEMA_VERSION` to 10. Prepend to its doc comment: "Schema 10 renames the
     `returned` walk tier to `tickler` (`tier` values and `by_tier.tickler`); the Ready
     `state` stays `resurfaced` and `counts.resurfaced` is unchanged."
   - Update the long_about tier order in both help texts.
   - In the REVIEW summary, `{returned} returned` becomes `{tickler} tickler`.
   - The tier heading tuple becomes `("tickler", "TICKLER")`.
   - The row detail arm becomes
     `"tickler" => "tickler {sep} scheduled … {sep} fresh …"`.
   - JSON `by_tier` key `"tickler"`.
   - Check whether `src/native/completion/` mirrors any of this help text. Update it if
     it does.
3. Tests and fixtures
   - Rename `returned` tier variables and assertions to `tickler` / `TICKLER` /
     `1 tickler` in `src/native/freshness/state_tests.rs` and `tests/cli/freshness.rs`.
   - Every `schema_version` assertion becomes 10.
   - In `tests/fixtures/freshness_keeps/vectors.json`, the D4 vector becomes
     `"id": "D4-tickler"`, `"tier": "tickler"`, `"why": "tickler counts as due"`. Phase
     `plugins-tickler-footer` mirrors this verbatim.
4. `docs/freshness.md`
   - Rename every RETURNED/`returned` _tier_ reference to TICKLER/`tickler`. This covers
     the intro tier order, §4 tier definitions and comparator table
     (`| tickler | due_on (= scheduled) ↑ … |`), the commitment-tier lists, keep
     counting (`tier in {'rotten','tickler'}`), the example human output (`14 tickler`,
     a `TICKLER 14` heading, `tickler · scheduled …` rows), the JSON contract's `tier`
     enum, the `B = NEW ∪ TICKLER ∪ ROTTEN ∪ READY` partition ("TICKLER is bucket rotten
     with state resurfaced"), the R1/R2/CL8 and other vector descriptions, and the
     status table rows (the `rotten.md` row now says TICKLER and ROTTEN groups).
   - Change "Human vocabulary says 'returned'" to "Human vocabulary says 'tickler'".
   - Add a schema 10 sentence to the machine-vocabulary paragraph and change the
     contract to `schema_version: 10`.
   - Change the bob-ledger-tools namespace mentions to v8 ("v8 renames the `returned`
     tier and `byTier.returned` to `tickler`").
   - Rewrite the footer paragraphs (the "desktop footer shows only nonempty walk
     groups…" paragraph and the "Obsidian footer omits zero-count groups…" sentence) to
     match the _Review footer design_ section above: short footer labels WIP/TICKS/REFS
     with full labels elsewhere, short labels in the current-row context too, the legend
     tooltip line, and TICKS split from ROTTEN.
   - Keep every RESURFACED/`resurfaced` _state_ reference.
5. `docs/getting-started.md`
   - In the status table, the `[/]` row becomes "In Progress, also called Pending or
     WIP".
   - Change the tier order line (~170) to TICKLER.
6. `README.md` (~697, ~713, ~729)
   - Change "returned from a scheduled deferral" to tickler wording.
   - Tier order becomes TICKLER.
   - "`rotten.md` holds tickler tasks plus expired tasks".
7. Do not touch `sase/memory/`. Phase `vault-and-memory` owns it.

Verification:

- `just all` passes (fmt, clippy, tests).
- `cargo run -q -- freshness list` against the real vault prints `tickler` / `TICKLER`.
  `cargo run -q -- freshness list -f json | jq '.schema_version, .counts.by_tier'` shows
  10 and a `tickler` key.
- Run `rg -n -i -w 'returned' src tests docs README.md`. Only unrelated English should
  remain, such as "returned by/from", `docs/ref.md`'s `returned` count field, capture
  and dependency comments, and dataview's "This returned". Phase 3 handles memory
  references.

### Phase `plugins-tickler-footer` — bob-plugins rename, footer short labels, deploy

- **id:** `plugins-tickler-footer`
- **size:** medium
- **depends on:** none (runs in parallel with `cli-tickler`)
- **repo:** linked `bob-plugins`. Open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`), use only the printed path, and read its
  `AGENTS.md`. Edit `src/` fragments only, never generated `main.js`, and keep each
  fragment ≤ 1000 lines.

Steps, bob-ledger-tools (`plugins/bob-ledger-tools/src/`):

1. `100-freshness-evaluate.js`
   - `tier = "returned"` becomes `"tickler"`. Update the documented tier union and the
     `FRESHNESS_TIER_ORDER` key (`tickler: 5`).
   - `freshnessTierLabel("tickler")` returns `"TICKLER"`.
   - Add `freshnessTierFooterLabel(tier)`. It returns `"WIP"` for `pending`, `"TICKS"`
     for `tickler`, `"REFS"` for `references`, and `freshnessTierLabel(tier)` for every
     other tier. Add a one-line comment saying it is the compact status-bar label.
   - Export it from `350-exports.js`.
2. `110-freshness-queue.js`
   - Comparator `left.tier === "tickler"`.
   - `byTier` init key and the walk sum.
   - In `freshnessStatusView`, rename `tierReturned` to `tierTickler`, read
     `tierCount("tickler", safe.resurfaced)`, and change the tooltip to `· TICKLER `.
     Leave that legacy view's other wording alone.
   - Update the comments.
3. `120-freshness-footer.js`
   - `FRESHNESS_FOOTER_TIERS` / `FRESHNESS_FOOTER_COMMITMENT_TIERS` use `tickler`.
   - `freshnessReviewMachineTier` accepts `tickler`, and legacy `state: "resurfaced"`
     maps to `tickler`.
   - The `freshnessReviewEntryView` tickler branch falls back to `"tickler"`. It still
     returns the full label.
   - `freshnessFooterReadTiers` reads
     `tickler: freshnessFooterTierCount(safe, "tickler", safe.resurfaced)`.
   - `freshnessFooterGroups` uses `freshnessTierFooterLabel(key)`.
   - In `freshnessFooterView`, `current.label` becomes
     `freshnessTierFooterLabel(presentation.tier) || presentation.label`.
   - Add the legend tooltip line and change the TICKS/ROTTEN tooltip line, both as in
     _Review footer design_.
4. `130-freshness-marks.js`
   - The `reviewModel` field `returned` becomes `tickler`.
   - The tooltip reads `… = N tickler + M rotten …`.
   - Update the comments. The local `resurfaced` variable can stay.
5. Comments and other code
   - `090-freshness-config.js`: `dueTier` checks `"tickler"`.
   - `070-ready-and-review.js`: comment.
   - `230-plugin-freshness-api.js`: empty `byTier` key `tickler`.
   - `170-plugin-lifecycle.js`: the freshness namespace goes to `version: 8`. Update its
     comment (tier order with TICKLER; "v8 renames the `returned` tier,
     `byTier.returned`, and `reviewModel().returned` to `tickler`").
6. `manifest.json`: minor bump from the current version (1.32.1 → 1.33.0 at planning
   time).

Steps, bob-navigation-hotkeys (`plugins/bob-navigation-hotkeys/src/`):

7. `470-keydown-and-freshness.js`
   - `reviewEntryMachineTier` accepts `tickler`, maps legacy `tier: "returned"` and
     `state: "resurfaced"` to `tickler`, and gets a comment that the `returned` read is
     for a v7 ledger api.
   - `reviewIsCommitmentTier` uses `tickler`.
   - Update the comments (NEXT/TICKLER/REFERENCES).
8. `480-review-jump-and-nav-api.js`
   - The `tickler` branch detail stays (`back since …`, fallback `tickler`).
   - The keep-count gate becomes `tier !== "rotten" && tier !== "tickler"`.
   - Update the comments.
   - `520-plugin-lane-links-and-review.js`: comment.
9. `manifest.json`: minor bump (2.10.1 → 2.11.0 at planning time).

Tests (`scripts/*.cjs`):

10. Update every `returned` tier key, label, and expectation. This covers the
    freshness-footer, freshness-queue, freshness-states, freshness-tracking,
    freshness-review-model, freshness-namespace (version 8), freshness-keeps (mirror
    `D4-tickler` verbatim), navigation-freshness, and navigation-keep-counting tests.
    - Footer expectations become `"NEW 1 · WIP 2 · TICKS 1 · ROTTEN 3"`,
      `contextText === "WIP 1/2"`, and `["PROJECTS 1", "REFS 1"]`.
11. Add focused tests:
    - The `freshnessTierFooterLabel` map for all nine tiers.
    - The current-row footer context uses the short label while `reviewEntryView`
      returns the full `TICKLER` / `PENDING`.
    - The legend line appears only when an abbreviated label is shown and lists only the
      abbreviations shown.
    - Nav normalizes a legacy `tier: "returned"` entry to `tickler`.

Docs and deploy:

12. Update `README.md`:
    - The plugin table versions.
    - Tier order with TICKLER in both plugin rows.
    - Namespace v8.
    - The review-footer paragraph (~line 135): short labels WIP/TICKS/REFS, full labels
      in notices, the legend tooltip, and TICKS split from ROTTEN.
13. Run `npm run build`, `npm test`, and `npm run validate`. All must pass.
14. Run `bob plugins sync`, which AGENTS.md requires.

Verification:

- Run `rg -n -w -i 'returned' plugins/*/src scripts README.md`. Only unrelated English
  should remain ("always returned null", "returned summary", "returned in ascending file
  order", and so on), plus the single documented legacy read in nav.

### Phase `vault-and-memory` — vault notes, glossary term, memory updates, final sweep

- **id:** `vault-and-memory`
- **size:** small
- **depends on:** `cli-tickler`, `plugins-tickler-footer`
- **repos:** the Bob vault (`~/bob`, plain files kept in git by the `bob vault-sync`
  automation; do not commit or push it manually) and bob-cli `sase/memory/`. Use
  `/sase_memory_write` before the memory edits. This approved plan is the authorization
  for every memory change listed here.

Vault steps. Edit live surfaces only. Leave history alone: completed task lines, dated
daily/monthly notes, `ref/chat/` transcripts, `_generated/`, `_conflicts/`, `done/`, and
unrelated English such as novels and nvim notes.

1. `rotten.md`
   - Change the intro prose RETURNED → TICKLER (lines ~11–13).
   - The decision-table row `returned deferral, not now` becomes
     `tickler task, not now`. Realign the table columns.
   - Heading `### RETURNED Tasks` becomes `### TICKLER Tasks`.
   - Keep the `state … === "resurfaced"` / `!== "resurfaced"` filters.
2. `blocked.md` (~line 36)
   - "A returned deferral" becomes "A tickler task".
   - The link becomes `[[rotten#TICKLER Tasks|TICKLER]]`.
   - "an unconfirmed return" becomes "an unconfirmed tickler".
3. `crowded.md` (~line 10): `NEW → PENDING → NEXT → RETURNED` becomes
   `NEW → PENDING → NEXT → TICKLER`.
4. `gtd_daily.md`: on the open recurring `#gtd #post Morning review` line only,
   "returned not-now is a priority roll" becomes "tickler not-now is a priority roll".
   Leave completed instances unchanged.
5. `dash.md` (~line 170 comment): "returned or age-expired row" becomes "tickler or
   age-expired row".
6. Run a vault-wide sweep:
   `rg -n -i 'RETURNED|rotten#RETURNED' ~/bob --glob '*.md' --glob '!_conflicts/**' --glob '!ref/chat/**' --glob '!_generated/**'`.
   Fix any remaining live reference to the tier or to the old heading anchor.

Memory steps (bob-cli `sase/memory/`):

7. Create `sase/memory/glossary/morning-gtd-review-footer.md` with the text below. The
   implementer may tighten the wording, but every fact must stay.

   ```markdown
   ---
   keyword: Morning GTD Review Footer
   aliases:
     - "review footer"
   ---

   The bob-ledger-tools item in Obsidian's desktop status bar that keeps the morning GTD
   review — the tiered `]s` walk over the shared [[task-freshness]] queue — in view
   while you work through it. It answers three questions at a glance: how much is still
   due, where the cursor sits in the walk, and whether the commitment tiers are done
   (after them come ROTTEN upkeep, which is fine to stop partway, and the POST
   closeout). Off a queue row it reads `⟳ Review N due · K commitments` (or
   `Commitments done` / `Upkeep budget met`) with a `]s next` hint; on a queue row it
   reads `⟳ Review r/N · TIER i/M` plus that row's detail. The other nonempty tier
   groups follow in walk order, then the `✓ upkeep/budget today` meter. To fit the
   status bar it uses short tier labels — `WIP` for PENDING (WIP is an official alias of
   the Pending status), `TICKS` for TICKLER, `REFS` for REFERENCES — explained in its
   tooltip, and it drops groups, then detail, hint, and meter as space runs out.
   Clicking jumps to the next due task like `]s`. It is a read-only projection: it never
   stamps or changes tasks, its numbers are queue positions rather than progress, and it
   hides when nothing is due or Tasks is unavailable. `]s` notices and
   `bob freshness list` keep full tier names, and dashboard ROTTEN chips fold TICKLER
   into ROTTEN where the footer splits them; see [[decisions/review-walk-is-tiered]].
   ```

8. `sase/memory/glossary/task-freshness.md`
   - "resurfaced (its `scheduled` date arrived after the stamp)" becomes "a tickler (its
     `scheduled` date arrived after the stamp; machine state `resurfaced`)".
   - The tier list RETURNED becomes TICKLER.
   - "`rotten.md` holds RETURNED plus expired ROTTEN" becomes TICKLER.
   - "a Pending or Next task" becomes "a Pending (WIP) or Next task".
   - Add one clause linking [[morning-gtd-review-footer]] as the walk's status-bar view.
9. `sase/memory/glossary/keep-streak.md`: "rotten/returned target" becomes
   "rotten/tickler target".
10. Decision records, amended in place at Bryan's request, following the 2026-10-04
    "Amended in place … at Bryan's request" precedent. In each record, change only the
    tier name in the summary and body, then append the line
    `Amended in place 2026-10-07 at Bryan's request: the RETURNED walk tier is renamed TICKLER (machine tier \`tickler\`,
    footer label TICKS); the Ready state stays \`resurfaced\`.`
    - `decisions/review-walk-is-tiered.md`: the summary and claim tier order; the
      `B = NEW ∪ TICKLER ∪ ROTTEN ∪ READY` partition; the rejected alternatives "is the
      TICKLER tier instead", "(NEW → TICKLER → ROTTEN)", and "Deferred tickler tasks
      roll their P-level".
    - `decisions/ready-is-freshness-gated.md`: the partition, "`rotten.md` (TICKLER plus
      expired ROTTEN)", and "TICKLER/ROTTEN groups in the vault".
    - `decisions/rotten-keeps-use-priority-decay.md`: "rotten/tickler" (two places).
    - `decisions/decay-decisions-are-available-immediately.md`: "ROTTEN/TICKLER".
11. Run `sase memory init` to regenerate `AGENTS.md`, the provider shims, and the memory
    README. Never hand-edit those files. Confirm that
    `sase memory read glossary:"review footer" -r "verify new term"` prints the new term
    and pulls in Task Freshness. Confirm that the regenerated roster shows TICKLER.

Final sweep (record leftovers in the phase notes):

12. Across bob-cli (`src tests docs README.md sase/memory` plus regenerated shims),
    bob-plugins (`plugins/*/src scripts README.md`), and the live vault surfaces above,
    `rg -n -w -i 'returned'` and `rg -n 'RETURNED'` must leave only unrelated English,
    the nav legacy read, and historical records. Then run `just all` once more in
    bob-cli.

## Risks and notes

- **Decision records amended in place.** The decisions descriptor calls accepted records
  immutable. This plan amends four of them for a pure rename. That follows the existing
  "Amended in place … at Bryan's request" precedent and keeps the always-loaded roster
  accurate. Approving the plan approves those edits.
- **Mixed plugin deploys.** `bob plugins sync` ships both plugins together. Nav's legacy
  `returned` read covers a reload window in which an older ledger-tools is still loaded.
- **Headless consumers of `bob freshness list -f json`.** None were found in bob-cli,
  bob-plugins, or the vault. The schema 10 bump signals the change to any external
  script.
- **Rust/JS parity.** Both evaluators must emit the same `tickler` key. The D4 vector
  must stay byte-identical across `tests/fixtures/freshness_keeps/vectors.json` and the
  plugin keeps test.
