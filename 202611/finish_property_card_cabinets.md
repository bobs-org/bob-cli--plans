---
tier: epic
title: Finish Property Card Cabinets — acceptance GO, go-live, and close bob-cli-qq
goal: 'Cabinets pass real-Obsidian acceptance with a GO verdict: the Reading perf
  budget is met and the intermittent Reading card miss is gone. They then go live.
  The `cabinetsPreview` gate is deleted, the docs and the approved memory are updated,
  and the deploy and smoke are clean. That lets bob-cli-qq.8, bob-cli-qq.9, and epic
  bob-cli-qq all close.

  '
parent_bead: bob-cli-qq
decisions:
  memory_cabinet_term:
    ask: Add the glossary:cabinet strand that bob-cli-qq's reviewer already approved?
    memory:
    - glossary:cabinet
    default: true
    requested: Can you help me finish whatever work remains on the bob-cli-qq bead
      and then close it?
    answer: true
  memory_card_strands:
    ask: Update the property-card, area-card, and project-card strands for cabinets
      and `_`?
    memory:
    - glossary:property-card
    - glossary:area-card
    - glossary:project-card
    default: true
    corrects: '`_` folds the card''s own section'
    answer: true
  memory_decision_record:
    ask: Add decisions:cabinets-absorb-catalogs-at-render-time, as approved on bob-cli-qq?
    memory:
    - decisions:cabinets-absorb-catalogs-at-render-time
    default: true
    requested: Can you help me finish whatever work remains on the bob-cli-qq bead
      and then close it?
    answer: true
phases:
- id: perf
  title: Bring the Reading cabinet paint within the 2x inline budget
  depends_on: []
  size: medium
  description: 'perf: profile the gate-open Reading paint of the 106-record album
    and the 64-row Projects host. Remove the double render and paint the breakdown
    confirms, without weakening fail-open or parity. Prove both perf checks pass in
    two athena runner runs.'
- id: reading-miss
  title: Fix the intermittent Reading card miss in the shared property-card host
  depends_on: []
  size: medium
  description: 'reading-miss: in the 240c host, resolve a Reading surface from the
    live preview leaf when the postprocessed element is detached or its retries are
    exhausted. Re-render only an eligible note that no host has carded, keep the absorption
    tags, and show zero misses in a reload loop.'
- id: polish
  title: Footer overlay narrow clipping, Areas title row, and runner console errors
  depends_on: []
  size: small
  description: 'polish: make the armed footer overlay shed `esc` and truncate nouns
    at 420 px without clipping either end. Drop the duplicated callout title row in
    the Areas cabinet body. Make the cabinets runner fail on console.error, including
    during Live Preview cursor motions.'
- id: acceptance
  title: Rerun real-Obsidian Cabinets acceptance to GO and close bob-cli-qq.8
  depends_on:
  - perf
  - reading-miss
  - polish
  size: medium
  description: 'acceptance: run the full gate-open cabinets runner on athena twice,
    review every PNG, and fix in-scope defects in place. Retain the evidence and record
    VERDICT: GO on bob-cli-qq.8, then close it.'
- id: switch-on
  title: Delete the cabinets gate, convert the tests, deploy, and smoke
  depends_on:
  - acceptance
  size: medium
  description: 'switch-on: delete 240w and every gate branch, and drop the dead preview
    keys from the runner. Convert the closed-gate tests, extend the passive suite,
    and give the runner a gate-free mode. Then build, test, run bob plugins sync,
    and run the post-switch smoke with prose Outline and a ref card.'
- id: docs-memory
  title: Cabinets docs in both repos, approved memory, and close bob-cli-qq.9
  depends_on:
  - switch-on
  size: medium
  description: 'docs-memory: add docs/cabinets.md and update the family pages, bringing
    bob-ledger-tools.md under its word budget. Fix stale bob-cli docs, write the approved
    glossary and decision memory, regenerate and check the docs, sync, and close bob-cli-qq.9.'
proposed_by: bbugyi200.apollo.g7
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 09:12:14
status: done
bead_id: bob-cli-qq.10
---

- **PROMPT:** [prompts/202611/finish_property_card_cabinets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_property_card_cabinets.md)
- **BEAD:** [bob-cli-qq.10](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qq/bob-cli-qq.10.md)

# Finish Property Card Cabinets

## Why

Epic **bob-cli-qq** (Property Card Cabinets) has seven closed phases. Two are stuck
`IN_PROGRESS`, and no agent is running for either:

- **bob-cli-qq.8 (acceptance) is NO-GO.** Its last run (attempt 28, bob-plugins
  `28e75739`, bob-ledger-tools 1.172.5) passed 94 of 97 checks. The note says the rest
  "does not fit one agent". Two blockers remain:
  1. **Perf.** In Reading, the cabinet median is over 2× the inline median. The 64-row
     Projects host measured 146–147 ms against 65–70 ms (2.1–2.25×). The 106-record
     album measured 127–138 ms against 65–69 ms (1.84–2.12×). Long tasks topped out at
     51–58 ms, well under the 100 ms limit.
  2. **The Reading card miss.** In the shared `240c` host, a Reading card sometimes
     fails to appear: attempts 10, 12, 20, 21, 24, 27, and 28
     (`area-reading-ready-after-plugin-reload`, and through it
     `area-content-parity-gate-closed`). Every tracked widget has a detached
     postprocessed element, no surface, and exhausted retries, while the live preview
     has no card. A blanket `rerender` (attempt 29) wiped the Reading absorption tags
     and was reverted.
  3. Two spec defects were filed as follow-ups. At 420 px the footer overlay clips at
     both ends (qq.8 note #4). The Areas cabinet body repeats the callout title row
     (qq.8 note #5).
- **bob-cli-qq.9 (go-live) has not started.** Its gate holds no longer: `bob-cli-qm.7`
  and `bob-cli-qp.8` are both closed. Two bob-cli-qq epic notes add work to it. Note #1
  asks it to drop the dead `cardPreviewKinds` and `refCardPreview` runner keys. Note #2
  says the runner never counts `console.error`.
- **The docs check fails on master today.** bob-plugins `npm run docs:check` (which runs
  bob-cli `docs-check`) reports `docs/reference/plugins/bob-ledger-tools.md` at 6049
  words, over the 6000-word reference cap. Go-live edits that page anyway, so it has to
  fix this. The qq.8 follow-up #1 overrun (`vault-format.md`) is no longer flagged.

This child epic finishes both phases, closes them, and hands bob-cli-qq to its landing.
**The binding design contract is the parent plan.** Read it before you start:
`sase bead read bob-cli-qq -r "<why>"` prints its path
(`plan:202611/property_card_cabinets.md`). Its sections "Design", "Phase `acceptance`",
"Phase `go-live`", and "Epic acceptance (land agent)" stay authoritative. This plan only
adds the repairs and the order. The reviewer decisions on bob-cli-qq are final:
`golive_waits_for_qp = yes` plus the three memory approvals, which this plan's own
memory decisions restate.

## Governing decisions

- `decisions:notebook-routes-write-notes`: bob owns catalog bytes. Cabinets never move,
  rewrite, or remove stored callouts. A perf fix must not touch them either.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the armed pick still cancels with Esc, `q`,
  and Ctrl+`[`.
- `decisions:bob-cli-releases-ride-a-daily-train`: no bob-cli code or JSON contract
  changes. bob-cli gets only memory and docs edits, never a `Cargo.toml` version or
  `CHANGELOG.md` edit.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture is untouched.
- `decisions:bank-hub-is-path-canonical-never-area`: the bank hub keeps only its Setup
  and Properties cabinets.
- `decisions:prose-notes-are-owned-prose`: the prose Outline cabinet stays the prose
  card's own fold, and cabinets never write prose.
- `decisions:decision-rules-are-read-at-planning-time`: the roster was read, and no
  other record governs card folds or `_`.

## Rules every phase follows

- **Repos.**
  - All plugin code is in the linked **bob-plugins** repo. Open it with
    `sase repo open bob-plugins -r "<why>"`, use only the printed path, and read its
    `AGENTS.md` first.
  - bob-cli (this project) changes only in `docs-memory` (memory and docs).
  - Read the `obsidian.md` reference memory before any vault read. Read `tailnet.md`
    before using athena.
  - Real-Obsidian runs use scratch vaults only (bob-plugins
    `docs/develop/validation.md`, "Real-Obsidian Cabinets acceptance"). Never open the
    live vault in an automated run.
- **Fragments.**
  - Edit `plugins/bob-ledger-tools/src/` fragments only, and keep each at or below 1000
    lines.
  - Run `npm run build`, and never hand-edit `main.js`.
  - Prefixes are `propertyCardCabinet*` in JS and `.bob-cabinet*` /
    `.bob-property-card-cabinets` in CSS.
  - Add test exports through `374k-exports-cabinets.js`.
- **Rebase discipline.** `perf`, `reading-miss`, and `polish` run in parallel, and each
  bumps the manifest. On a conflict in `main.js`, `styles.css`, `fragments.json`,
  `package.json`, a manifest version, or the docs inventory: take upstream, re-apply
  your additive lines, rebuild, and re-bump. Never hand-merge a bundle.
- **Per plugin change.**
  1. Bump the manifest.
  2. Run `npm run build`, `npm run build:check`, the focused
     `scripts/test-ledger-tools-cabinets-*.cjs` and host suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`, following the refusal protocol in bob-plugins
     `AGENTS.md`.

  A pre-existing `npm test` failure that also fails on clean master is recorded, not
  fixed. attempt 28 saw two: `test-prose-field-roles` and the
  `links-to row follows blocks …` relation-stage test.

- **Invariants.**
  - **Fail-open.** A valid stored catalog is never hidden without a working cabinet in
    that view. Source, embeds, hover, canvas, print, mobile, a failed mount, malformed
    or duplicate regions, and unload show the stored section in the same frame.
  - **Passive.** Painting, toggling, arming, picking, cancelling, and the footer overlay
    never write the vault or spawn bob.
  - **Gate-closed parity.** Until `switch-on`, the closed gate keeps every card
    byte-identical to today's.
- **Fixed acceptance bars.** Do not loosen the perf budget (cabinet median ≤ 2× the
  inline median, and no added long task over 100 ms). Do not weaken a runner check to
  get a pass. You may fix a measurement only when you can show it measures the wrong
  thing; record that evidence on your bead. A phase that cannot meet its bar stops and
  records evidence and a `PROPOSED FOLLOW-UP:`, so Bryan can decide.
- **Evidence and beads.**
  - Retain real-app evidence with `/sase_artifact` against your phase bead, and against
    `bob-cli-qq.8` or `bob-cli-qq.9` where noted.
  - Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on your own
    phase bead instead.
  - Run `sase bead epic-symbols <phase>` before closing.
- **Long work.** Builds, `npm test`, and athena runner runs go through `/sase_monitor`.
  Finish every turn through `/sase_final`.

## Phase `perf`

**Context.** The runner's `measureCatalogPaint` (`scripts/obsidian-cabinets-e2e.cjs`)
opens a neutral note in a Reading leaf, then the target. It times how long until the
expected `li` rows exist: in the cabinet body
(`.bob-property-card-cabinets [data-cabinet-kind=…] .bob-cabinet-provider-content`) with
the gate open, or in the native callout with it closed. `perfWithinBudget` applies the
budget. The suspected cost is double work. Obsidian renders the stored callout and the
inline painter decorates it before absorption hides it. Then
`propertyCardCabinetRenderCatalog` (`240wj`) runs a second `MarkdownRenderer.render`
plus the painter for the cabinet body. The body cache never helps across opens, because
a body inside another mount's container is not orphaned.

1. **Profile before fixing.** Break down the gate-open Reading path for both fixtures
   with `performance.mark`/`measure` (removed or test-flagged before commit), or with a
   CDP `Profiler`/`Tracing` capture driven from the runner. Cover these stages:
   - the native section render and inline painter;
   - the `240c` card mount (`renderInReading` batching, `refreshReading`);
   - stack assembly (`240wl`) and provider `applies`;
   - `buildBody` → render → painter;
   - Reading absorption (`240wh`).

   Record the breakdown in a bead note before you choose a fix.

2. **Remove the cost the breakdown confirms.** Candidates, most suspected first:
   - **Defer the inline Reading painter for a kind a cabinet will own.** This applies
     when the gate is open and that kind's provider applies to the note's card: host
     kinds on area and project cards, Records on album cards. The stored callout stays
     native until the claim hides it. If the claim never comes or is released (a failed
     body, unload, any fail-open context), the painter runs on the native callout in
     that same path, so fail-open still shows today's painted catalog. Contexts that
     never get a cabinet (embeds, hover, canvas, print, Source) never defer.
   - **Build the Reading body from the already-rendered callout** instead of a second
     `MarkdownRenderer.render`. Do this only if the render stage is material. Live
     Preview keeps `MarkdownRenderer`. Re-bind links and hover as `240wj` does today.
   - **Start a default-open body (Records) as soon as its provider applies**, rather
     than after the whole stack assembles. Batch layout reads and writes in the painters
     the profile flags.
   - **Do not change the surface-resolution or retry logic in `240c`.** `reading-miss`
     owns it. If card-mount latency dominates, rebase onto `reading-miss`'s landed
     commit before touching `240c`, or record the finding and stop.
3. **Tests.** Add unit cases in `scripts/test-ledger-tools-cabinets-*.cjs`:
   - the deferral path;
   - the release path that repaints the native callout;
   - no deferral in a fail-open context;
   - cabinet-body row parity unchanged;
   - zero writes and spawns.

   Also run the closed-gate parity tests.

4. **Verify on athena.** Run the full runner twice. Each run must pass both
   `perf-106-record-album-within-budget` and `perf-64-line-projects-host-within-budget`.
   Record the medians and long tasks on the bead, and retain both reports.

## Phase `reading-miss`

**Context.** `createPropertyCardHost` in `240c-plugin-property-card-host.js` keeps one
widget per postprocessed `el`. `scheduleReadingSurfaceRetry` gives up after 8 attempts,
and `refreshReading` derives the surface only from `readingRoot(widget.el)`. The
runner's `readingHostDiagnostics` show the failure state: after a plugin reload every
widget for `cash.md` holds a detached `el` (`elConnected: false`) with no surface, while
the active preview is connected and has no card. Every card species shares this host, so
the fix is family-wide.

1. **Reproduce first.** Add a focused runner mode (or a small sibling script) that puts
   a Reading leaf on the area fixture. It reloads `bob-ledger-tools` at least 20 times,
   counting card misses and capturing the diagnostics. Record the baseline miss rate on
   the bead.
2. **Fix.** When a widget's `el` is detached, or its retries are exhausted:
   - Resolve the surface from the live workspace, using markdown leaves in preview mode
     whose file path is `widget.path`, through `readingRoot`/`readingScope`. Rebind and
     paint.
   - Drop stale detached widgets once a live surface for the same path is bound, so none
     leak.
   - Request a targeted re-render of that one preview **only** when all of these hold:
     no surface is found; no host has mounted a card for that note; and the note is
     card-eligible under its species descriptor. Never re-render a card-less note or a
     note another host has already carded.
   - After any re-render, run the Reading absorption re-scan (`240wh`) so the
     `bob-cabinet-absorbed` tags return.
3. **Tests** in `scripts/test-ledger-tools-property-card-host.cjs`:
   - a detached `el` plus a live preview leaf mounts the card;
   - exhausted retries recover through the workspace;
   - a card-less note never re-renders;
   - a second host's card blocks the re-render;
   - absorption tags survive.
4. **Verify.**
   - The reload loop shows 0 misses in 20 or more reloads.
   - A full runner run passes `area-reading-ready-after-plugin-reload` and the Reading
     render matrix for every species.
   - `npm test` is green apart from recorded pre-existing failures.

## Phase `polish`

1. **The footer overlay at 420 px.** The `footer-*-narrow-armed` shots in
   `file:explicit:4e093423e907fab57e92e9db` show "abinets 1 Albums …" and "- Properties
   …" with the leading `▤ Cabinets` and the trailing `esc` cut off.
   - Per the parent contract, a narrow bar sheds `esc` first, then truncates nouns with
     an ellipsis. The label and the keycaps never clip.
   - Fix it in `122a-walker-footer-overlay.js` and the `Walker footer overlay` CSS
     region.
   - Add a narrow fixture to `scripts/preview-walker-footer.cjs`, and add a unit test
     for the shedding state.
2. **The Areas body repeats its title row.** The body shows "Areas 1 area" with its icon
   under a header that already reads "Areas · 1 area".
   - The CSS already hides `.callout-title` in cabinet bodies (the `Cabinets` region in
     `styles.css`), so find the row the Areas painter (`241aj`) emits and suppress it
     inside `.bob-property-card-cabinets`.
   - Check Projects, Notebooks, Albums, and Records for the same leak.
   - Row parity must not change.
3. **The runner counts console errors** (bob-cli-qq note #2, part 1).
   - Subscribe to `Runtime.consoleAPICalled`, following
     `scripts/obsidian-ref-card-acceptance-e2e.cjs:523`, and collect `consoleErrors`.
   - Add a mandatory `no-console-errors` check. Allowlist only documented environment
     noise, with a reason for each pattern.
   - Wrap the Live Preview Vim and arrow cursor-motion checks around absorbed ranges
     with a before/after error delta, following
     `scripts/obsidian-ref-card-acceptance-e2e-flows.cjs:602`.
   - Update `scripts/test-obsidian-cabinets-e2e.cjs` to match.
   - Leave the gate keys alone, because `switch-on` removes them.

## Phase `acceptance`

1. Read bob-cli-qq.8 and its notes, the parent plan's "Phase `acceptance`", and the
   attempt-28 `acceptance-review.md` in `file:explicit:4e093423e907fab57e92e9db`.
2. Run the full runner on athena under `xvfb-run` against current bob-plugins master,
   which includes the three repair phases. Keep the gate open, using the runner's
   existing gate data. Run it **twice**, so the old intermittent miss gets two chances
   to reappear.
3. Copy the screenshots back and inspect **every** PNG. Write an `acceptance-review.md`
   like attempt 28's, with a verdict for each image. Re-check the two polish defects and
   the earlier visual defects table.
4. **GO** requires all of the following in both runs:
   - every mandatory check passes, including both `perf-*` checks and
     `no-console-errors`;
   - 0 runtime exceptions;
   - 0 passive writes or spawns;
   - every capture is reviewed.

   **On NO-GO:** fix in place if the fix is in cabinets scope (`240w*`, `241al`,
   `241tb`, `122a`, the CSS regions, or the runner), then rerun. Otherwise record the
   evidence and a `PROPOSED FOLLOW-UP:`, then stop.

5. On GO:
   - Retain the reports, the review, and the screenshots with `/sase_artifact` against
     `bob-cli-qq.8` and this phase bead.
   - Note `VERDICT: GO` on bob-cli-qq.8 with the evidence refs, the source commit, and
     the perf medians.
   - Note on bob-cli-qq.8 that the `polish` commit fixed its follow-ups #4 and #5, and
     that follow-up #1 no longer reproduces.
   - Then run `sase bead close bob-cli-qq.8 --note "<what you verified>"`.

## Phase `switch-on`

0. **Gate.**
   - Confirm with `sase bead read` that bob-cli-qq.8 is closed on GO, and that
     `bob-cli-qm.7` and `bob-cli-qp.8` are closed. Both were closed at planning time.
   - Re-read the parent plan's "Phase `go-live`" steps 1, 2, and 6.
1. **Delete the gate.**
   - Remove `240w-plugin-cabinet-gate.js` and its `fragments.json` entry, along with the
     `cabinetsPreview` plugin-data handling.
   - Remove every `propertyCardCabinetsGateLive()` branch, keeping only the open path.
     There are 26 call sites, in `240o`, `240r`, `240s`, `240wd`, `240we`, `240wl`,
     `240wm`, `241q`, `241tb`, `241v`, `242kc`, `248m`, and `374k`.
   - Keep the fold-role and delegation behavior as the only path.
   - Do not hand-edit the live vault's `data.json`. A leftover key there is simply
     ignored.
2. **Convert the tests.**
   - Turn the closed-gate parity tests into regression tests of the live structure.
   - Replace the gate-reader tests in `scripts/test-ledger-tools-cabinets-kit.cjs`
     (around lines 100–110) with one test: stale plugin-data keys
     (`cabinetsPreview: false`, `cardPreviewKinds`, `refCardPreview`) neither disable
     cabinets nor throw.
   - Extend `scripts/test-ledger-tools-property-card-passive-rendering.cjs` to cover
     cabinet paint, toggle, pick, and the overlay, with zero writes and spawns.
3. **Give the runner a gate-free mode.**
   - Drop `cabinetsPreview`, `cardPreviewKinds`, and `refCardPreview` from `GATE_DATA`
     and the `gateClosed` object. Drop the matching `assert.match` lines in
     `scripts/test-obsidian-cabinets-e2e.cjs` (bob-cli-qq notes #1 and #2).
   - Retire the checks that need a closed gate: the gate-closed baseline, the `perf-*`
     inline comparison, and gate-closed fail-open. Acceptance's retained GO evidence
     covers them.
   - Make content parity compare cabinet rows and text against the stored regions
     (`propertyCardCatalogRegions` over the fixture source, or Source mode).
   - Keep plugin-disable as the fail-open case.
   - Update the Cabinets section of `docs/develop/validation.md` to match.
4. Run the build, `build:check`, the focused suites, and `npm test`, then
   `bob plugins sync` under the refusal protocol.
5. **Post-switch smoke on athena,** with no gate key. Confirm from the shots every item
   in the parent plan's "Epic acceptance (land agent)" list:
   - the area host's four closed content cabinets with honest summaries;
   - Records open over a closed Schema;
   - the bank's Setup and Properties in one chrome;
   - `_` arming with digits on the card and in the footer;
   - `__`, `-`, and Esc/`q`/Ctrl+`[`;
   - stored catalogs visible in Source mode;
   - the prose Outline cabinet over qm.5's landed body;
   - a ref card with zero extras, where `_` falls to its details fold.

   Retain the evidence against `bob-cli-qq.9` and this phase bead.

## Phase `docs-memory`

0. Confirm that `switch-on` closed and its smoke evidence is retained. Read the
   `docs.md` reference memory. Re-read `glossary:property-card` and the family docs as
   they stand now.
1. **bob-plugins docs.**
   - Add the reference page `docs/cabinets.md`. It covers anatomy, kinds and order,
     defaults, the `_` pick, the footer overlay, absorption and fail-open, and the
     passive guarantees.
   - Add it to `docs/nav.yml` and regenerate the inventory.
   - Update the pages that still say "fold" or "Properties drawer":
     `area-and-project-cards.md`, `albums.md`, `prose-card.md`, `notebook-card.md`,
     `record-card.md`, `ref-card.md`, `property-editor.md`, `bob-ledger-tools.md`, and
     `develop/architecture.md`.
   - Say "Properties cabinet", and use "drawer" only for the Task Drawer.
   - **Bring `docs/bob-ledger-tools.md` under the 6000-word reference cap** by moving
     cabinet detail to `cabinets.md` and tightening.
   - `npm run docs:check` must pass. Then run `bob plugins sync`.
2. **bob-cli docs.**
   - Per `docs.md`, `docs/reference/plugins/*` and `docs/reference/glossary.md` are
     generated, so fix their sources (bob-plugins docs and memory), not the outputs.
   - Grep the bob-cli-owned pages (for example `docs/bank.md`, `areas.md`,
     `projects.md`, `notebooks.md`, `records.md`) for stale `_`-fold or "Properties
     drawer" wording, and fix only what is now wrong.
3. **Memory.**
   - Use `/sase_memory_write` before you edit, and stay inside this plan's accepted
     memory decisions.
   - Then run `sase memory init`, `just docs-gen`, and `just docs-check`.
   - Record a `PROPOSED FOLLOW-UP:` on this phase bead for "Records cabinet", "Schema
     cabinet", "Setup cabinet", and "Properties cabinet" wording in
     `glossary:album-card` and `glossary:bank-card`. Both are still true as written.
4. Close `bob-cli-qq.9` with `sase bead close bob-cli-qq.9 --note "…"`. The note names
   the `switch-on` and `docs-memory` commits and the smoke evidence refs.

> [!decision] memory_cabinet_term Add `glossary:cabinet` from the parent plan's
> `memory_cabinet_term` draft, refined to the behavior that shipped. Link
> `[[property-card]]`, `[[area-card]]`, `[[project-card]]`, `[[album-card]]`,
> `[[bank-card]]`, and `[[task-drawer]]`.

> [!decision] memory_card_strands
>
> - **`glossary:property-card`:** replace "`_` folds the card's own section" and the
>   foot's "Properties drawer" with the cabinet stack, `-`, and the `_` routing (0, 1,
>   or 2+ extras).
> - **`glossary:area-card`:** replace "`_` folds the Area's own details" and the
>   Properties-drawer wording. Child catalogs render as cabinets above the Properties
>   cabinet, and `_` toggles or picks them, falling back to the details fold only when
>   none exist.
> - **`glossary:project-card`:** make the same change, replacing "`_` folds the card's
>   own details".
>
> Link `[[cabinet]]` from all three.

> [!decision] memory_decision_record Add
> `decisions:cabinets-absorb-catalogs-at-render-time`, per the parent plan's
> `memory_decision_record` callout: its claim, why, rejected alternatives, cost, reopen
> conditions, and evidence (the research report and this epic's commits).

## Landing (this epic's land agent)

- Verify the parent plan's "Epic acceptance (land agent)" list:
  - every phase of both epics is closed;
  - `cabinetsPreview` is gone, and `bob plugins sync` is clean on master;
  - GO and smoke evidence are retained;
  - the passive suites are green;
  - no bob-cli code changed.
- Triage the `PROPOSED FOLLOW-UP:` notes on this epic's phases and on bob-cli-qq.8:
  - #4 and #5 were fixed by `polish`;
  - #1 no longer reproduces;
  - the album-card and bank-card wording proposal from `docs-memory` is still open.

  bob-cli-qq epic notes #1 and #2 are resolved by `polish` and `switch-on`, so record
  that on bob-cli-qq. Task bead `bob-cli-r5` was waiting for qq to close. Leave it
  alone; it is READY for its own triage.

- Then resume the **bob-cli-qq** landing through the `parent_bead` link. That landing
  closes bob-cli-qq (Bryan asked for it to close) and sets `status: done` on
  `plan:202611/property_card_cabinets.md`. This epic has no closeout phase for
  bob-cli-qq.

## Out of scope

- Loosening the perf budget, or any other acceptance bar.
- Any bob-cli code, JSON contract, or Bob Mac Capture change.
- Changing the stored catalog bytes or the vault format.
- The parent plan's own out-of-scope list and its post-go-live follow-ups.
- `bob-cli-r5` (descriptor-driven kit species lists). It lands after this.
