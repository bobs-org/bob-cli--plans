---
tier: tale
size: medium
title: Verify and close bob-cli-qq (Property Card Cabinets)
goal: "Finish the interrupted bob-cli-qq landing. Re-verify cabinets after the link-walk
  floor landed, wire the missing `#Records` and Outline-pane navigation, prove Find,
  Outline, and heading-link reveal in real Obsidian, then close bob-cli-qq and mark its
  plan done.

  "
proposed_by: bbugyi200.apollo.g7.f1
proposed_with: claude/opus
create_time: 2026-11-27 18:51:37
status: wip
---

# Verify and close bob-cli-qq

## Why

Bryan asked for epic **bob-cli-qq** (Property Card Cabinets) to be verified and closed.
Every child is closed, but the epic's own landing never finished. `bob-cli-qq.10.land`
landed the child epic, then ran the parent landing and left bob-cli-qq open behind a
`LANDING BLOCKER` note (bob-cli-qq note #4). This tale is the parent landing: it clears
that blocker, and its last step closes bob-cli-qq.

The binding design contract is the parent plan. `sase bead read bob-cli-qq -r "<why>"`
prints its path (`plan:202611/property_card_cabinets.md`). Its "Design → Reveal and
navigation", "Phase `acceptance`" matrix, and "Epic acceptance (land agent)" sections
are authoritative.

## Verified current state (2026-11-27, 23:25 UTC)

- **Beads.** Phases bob-cli-qq.1 through bob-cli-qq.9 and child epic bob-cli-qq.10 are
  all closed. bob-cli-qq is `IN_PROGRESS` (assignee `bob-cli-qq.land`).
  - The qq.10 close note lists the verification it already did: phase commits, gate
    removal, 191/191 focused suites at bob-plugins `f35848b4`, a clean
    `bob plugins sync -d`, and the GO and smoke evidence refs. It also lists the
    follow-ups it filed: bob-cli-ri, rj, rk, rl, rm, rn, ro, and rp, plus +1s on
    bob-cli-pg and bob-cli-q5.
  - bob-cli-qq note #3 records epic notes #1 and #2 as resolved.
  - bob-cli-qq note #4 (`LANDING BLOCKER`, from bob-cli-qq.10.land) rechecked every
    "Epic acceptance (land agent)" bullet at `f35848b4` and found them all OK. It kept
    bob-cli-qq open only for the navigation and live-coverage gap described below, and
    it independently matches this plan's finding.
- **Plan files.** `plan:202611/finish_property_card_cabinets.md` is already
  `status: done` (plans repo `9c7a669a`). `plan:202611/property_card_cabinets.md` is
  still `status: wip`.
- **Symbols.** `sase bead epic-symbols bob-cli-qq` and `bob-cli-qq.10` report no
  entries. Neither bob-cli nor bob-plugins has a `just symvision` recipe.
- **The lander.** `bob-cli-qq.10.land` was still running at 23:22 UTC. It was `DONE` by
  23:30 UTC, after writing note #4. No `bob-cli-qq.land` agent exists. Step 0 re-checks
  this.
- **Drift since the qq.10 verification.** bob-plugins `28b13c4e` (bob-cli-ra.4,
  bob-ledger-tools 1.174.0) docks the link-walk rail as every property card's floor. It
  edits `240wl-plugin-cabinet-stack.js`, `240o`, `240r`, `242k`, and `242kc`, and it
  landed after the qq.10 lander ran the suites. bob-cli-ra.5 (link-walk acceptance) and
  bob-cli-ra.6 (docs) are still in progress in bob-plugins.
- **Parent-phase follow-ups.**
  - qq.7 #3 (stray `console.log("DEBUG REST")`): gone. `64268af9` removed it.
  - qq.8 #1, #4, and #5: resolved per qq.8 note #8.
  - qq.1 #1 (protected bundles): resolved by qq.1 #2.
  - qq.7 #1 (unit-probe perf caveat): resolved by the real-Obsidian GO perf in qq.8 #7.
  - qq.3 #2 asked acceptance to confirm, live: Vim motions, heading-navigation reveal,
    Find reveal, and fail-open contexts. That was only partly done. The GO runner covers
    the Vim motions (`vim-motions-and-arrows-no-invisible-cursor`), the fail-open
    contexts, and one same-note `[[cash#Projects]]` link in Reading
    (`heading-navigation-reveals-cabinet`). Find reveal, the Outline pane, and
    `#Records` are never exercised live.
- **A defect this epic caused: `#Records` navigation is not wired.**
  - The only heading-navigation hook is `propertyCardCabinetHostRevealNavigation`
    (`241al-plugin-cabinet-host-catalogs.js:540`). It is registered as a capture-phase
    document click at `:624`.
  - It resolves headings only through `PROPERTY_CARD_HOST_CATALOG_HEADINGS`, which
    covers areas, projects, notebooks, and albums.
  - The Records claim carries `reveal: () => cabinetRecordsReveal(...)`
    (`241tb-plugin-cabinet-records.js:416` and `:656`), but no code path calls it.
  - The result: a `#Records` link or an Outline "Records" click never opens the Records
    cabinet. The parent plan requires both.
  - No unit test covers `propertyCardCabinetHostRevealNavigation`
    (`scripts/test-ledger-tools-cabinet-host-catalogs.cjs` has none).
  - The runner's scratch vault does not enable the core `outline` plugin.
- **qq.4 has no recorded resolution.** Its status is `closed`, but `closed_at` and
  `resolution` are null and `close_history` is empty.

## Governing decisions

- `decisions:cabinets-absorb-catalogs-at-render-time`: stored callouts stay
  byte-for-byte bob's, and nothing hides without a working cabinet. A navigation reveal
  only opens and scrolls a cabinet. It never writes, and it falls through (fail open)
  whenever no working cabinet claims the heading.
- `decisions:notebook-routes-write-notes`: bob owns catalog content. Fixture or runner
  changes never alter how stored catalogs are written.
- `decisions:prose-notes-are-owned-prose`: a prose note's own headings are never
  intercepted. Prose has no catalog claims, so the handler must fall through there.
- `decisions:bob-cli-releases-ride-a-daily-train`: no bob-cli code, `Cargo.toml`
  version, `CHANGELOG.md`, or JSON contract change. bob-cli gets only bead and plan-file
  updates.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture is untouched.
- `decisions:decision-rules-are-read-at-planning-time`: the roster was read, and no
  other record governs cabinet navigation.

## Rules

- **Repos.**
  - All plugin code is in the linked **bob-plugins** repo. Open it with
    `sase repo open bob-plugins -r "<why>"`, use only the printed path, and read its
    `AGENTS.md` first.
  - Read the `obsidian.md` and `tailnet.md` reference memory before using athena.
  - Real-Obsidian runs use scratch vaults only, never the live vault.
- **Fragments.**
  - Edit only `plugins/bob-ledger-tools/src/` fragments, and keep each at or below 1000
    lines (`241al` is at 634 lines and `241tb` at 861).
  - Run `npm run build`, and never hand-edit `main.js`.
  - Add test exports through the existing `374k`/`374l` export fragments.
- **Rebase discipline.** bob-cli-ra.5 and bob-cli-ra.6 may land in bob-plugins while you
  work. On a conflict in `main.js`, `styles.css`, `fragments.json`, `package.json`, a
  manifest version, or the docs inventory: take upstream, re-apply your additive lines,
  rebuild, and re-bump. Never hand-merge a bundle.
- **Per plugin change.**
  1. Bump the bob-ledger-tools manifest (patch).
  2. Run `npm run build` and `npm run build:check`.
  3. Run the focused `scripts/test-ledger-tools-cabinet*-*.cjs` suites and the
     property-card host and passive suites.
  4. Run `npm test` under `/sase_monitor`.
  5. Run `bob plugins sync` without `-F`, following the refusal protocol in bob-plugins
     `AGENTS.md`.

  A failure that also fails on clean master is recorded, not fixed. Known examples are
  the snooze vector (bob-cli-q5) and the taskLinks p95 check (bob-cli-pg).

- **Acceptance bars.** Never weaken or delete a runner check to get a pass. You may fix
  a measurement only when you can show it measures the wrong thing; record that evidence
  on bob-cli-qq.
- **Scope fence.** Leave bob-cli-rk's runner items alone: panel reset before rest
  captures, album head capture, and live-shaped fixtures. Leave bob-cli-rp (Ctrl+J/K
  Section Map jumps) and bob-cli-r5 alone too. No memory edits.
- **Long work.** Builds, `npm test`, and the athena run go through `/sase_monitor`.

## Steps

### 0. Preconditions

1. Run `sase agent list -j`. If `bob-cli-qq.10.land` or any `bob-cli-qq.land*` agent is
   still RUNNING, stop without changes. Note that on bob-cli-qq and report it.
2. Re-read bob-cli-qq and every note on it. Note #4 is the blocker this tale clears.
   Carry forward anything added after note #4.
3. Confirm that `plan:202611/finish_property_card_cabinets.md` (the PLAN path from
   `sase bead read bob-cli-qq.10`) says `status: done`. Set it if it does not.

### 1. Re-verify cabinets after the drift

1. On bob-plugins master, run:
   - `npm run build:check`, `npm run docs:check`, and `npm run docs:inventory:check`;
   - the focused cabinets, property-card host, passive-rendering, walker-footer, and
     card-floor suites.
2. Confirm that `cabinetsPreview`, `propertyCardCabinetsGateLive`, and the `240w` gate
   fragment are absent from `src/` and `main.js`.
3. Review `git log f35848b4..origin/master` for commits that touch `240w*`, `241al`,
   `241tb`, `240o`, `240r`, `122*`, or the Cabinets CSS region. Confirm that none
   duplicates, bypasses, or breaks cabinets. Fix any breakage in scope.

### 2. Wire `#Records` and Outline-pane navigation

**Contract.**

- On an album note whose Records region is absorbed in the active view, these all open
  the Records cabinet and scroll its head into view, with the default prevented:
  - a same-note `#Records` link;
  - an `[[<this note>#Records]]` link;
  - an Outline-pane "Records" item.
- The host kinds (Areas, Projects, Notebooks, Albums) keep today's behavior, and the
  Outline pane works for them too.
- Everything else falls through untouched, with the default not prevented:
  - links to another note's heading;
  - non-catalog headings;
  - prose notes;
  - Source mode, embeds, and hovers;
  - any view with no live claim or absorbed region for that kind.
- The reveal never writes the vault and never spawns bob.

**Design.** Keep one capture-phase navigation hook. Extend its heading-to-kind
resolution to cover `Records` → `records`, and route records through the active view's
claim reveal: `propertyCardCatalogRevealClaim` over the same candidate views. That
reaches `cabinetRecordsReveal`. Keep the host-panel fast path for the host kinds only,
unless it works unchanged for records. If records-specific logic would bloat `241al`,
put it in `241tb` behind the claim, not in the hook.

**Unit tests.** Add these to `scripts/test-ledger-tools-cabinet-host-catalogs.cjs`, or
to a new `scripts/test-ledger-tools-cabinets-navigation.cjs` registered wherever
`npm test` lists the cabinets suites:

- a same-note `#Projects` link and an Outline "Projects" item reveal the host cabinet;
- `#Records`, `eat#Records`, and an Outline "Records" item call the Records reveal and
  prevent the default;
- `[[other#Records]]`, a "Notes" heading, a prose note, and a view with no claim fall
  through without preventing the default.

### 3. Add live checks to the cabinets runner

1. **Vault.** Add `"outline"` to the scratch vault's `core-plugins.json` in
   `scripts/obsidian-cabinets-e2e.cjs`.
2. **Fixture.**
   - Add `[[eat#Records|Jump to Records]]` to the album fixture body, outside the
     callout.
   - If no unique token exists inside the cash Projects callout rows, add a unique Find
     sentinel there in a row label.
   - Update the lib self-test and the pins in `scripts/test-obsidian-cabinets-e2e.cjs`
     to match.
3. **Three mandatory checks.** Follow the surrounding `check`/`failOpenCheck` style.
   - `find-strict-inside-reveals-stored-source`:
     - In Live Preview on `cash.md`, use Obsidian's editor Find (`editor:open-search`)
       to go to the sentinel.
     - Assert that the selection is strictly inside the absorbed range, that the range's
       raw Markdown is visible, that the cursor is visible, and that no console errors
       occurred.
     - Then move the cursor out and assert the 240wg contract for leaving. The unit test
       at `scripts/test-ledger-tools-cabinets-absorption.cjs:487` defines it: boundary
       contact never reveals.
   - `outline-pane-reveals-cabinet`:
     - With Projects closed on `cash.md`, open the Outline view and click its "Projects"
       item, in both Reading and Live Preview.
     - Then do the same with "Records" on `eat.md`, with Records closed first.
     - Assert that each cabinet is expanded and its head is in the viewport.
     - Close the Outline leaf afterwards so the later captures are unchanged.
   - `records-link-reveals-cabinet`:
     - In Reading on `eat.md`, close Records and click "Jump to Records".
     - Assert that Records is expanded, its head is in the viewport, and the default was
       prevented.
4. **Screenshots.** Capture one screenshot after each new check and add it to the
   screenshot inventory.
5. **Docs.**
   - In bob-plugins `docs/develop/validation.md` (Cabinets runner section), name the
     Find, Outline-pane, and heading-link checks.
   - In `docs/cabinets.md` "Absorption and fail-open", add one sentence: heading links
     (`[[note#Projects]]`, `#Records`) and the Outline pane open the matching cabinet,
     and a Find hit inside a hidden range reveals the stored source.
   - Run `npm run docs:check`.

### 4. Real-Obsidian run on athena

1. Run the full runner once on athena under `xvfb-run` against the bob-plugins commit
   that has steps 2 and 3. Use the command in `docs/develop/validation.md` and run it
   through `/sase_monitor`.
2. **GO** requires all of the following:
   - every mandatory check passes, including the three new ones;
   - 0 runtime exceptions and 0 console errors;
   - 0 passive writes or spawns, with the scratch markdown unchanged;
   - every PNG is reviewed.
3. Write an `acceptance-review.md` with a verdict for each image. Look especially at the
   cabinet stack sitting above the link-walk floor that `28b13c4e` added.
4. If a check that passed in the GO runs fails, rerun once. If it fails twice, it is a
   NO-GO.
5. **On NO-GO:** fix in scope (`240w*`, `241al`, `241tb`, the Cabinets CSS region, or
   the runner) and rerun. If the fix is out of scope, stop:
   - leave bob-cli-qq open and its plan file `wip`;
   - note the blocker and the evidence on bob-cli-qq;
   - report it.
6. **On GO:**
   - Retain the report, the review, and the screenshots with `/sase_artifact` against
     bob-cli-qq.
   - Note `VERIFIED:` on bob-cli-qq with the evidence refs, the bob-plugins commit, and
     the bob-ledger-tools version.

### 5. Deploy

Apply the per-plugin-change rules above. The deploy ends with a clean `bob plugins sync`
of the final bob-plugins commit.

### 6. Follow-up triage

- Run any genuinely distinct issue you find that this epic did not cause through
  `/sase_new_task`, naming bob-cli-qq as the proposer.
- Do not refile bob-cli-ri, rj, rk, rl, rm, rn, ro, or rp, and do not re-corroborate pg
  or q5.
- Record the declines in the close note:
  - qq.7 #3 (fixed by `64268af9`);
  - qq.8 #1, #4, and #5;
  - qq.1 #1;
  - qq.7 #1;
  - qq.3 #2 (closed by steps 2–4).
- The parent plan's "Follow-ups to consider after go-live" are ideas, not defects:
  device-local persistence, a root `area.md` hub card, hiding absorbed headings from the
  Outline pane, and an explicit picker. Record that they are not filed.
- qq.4: confirm from the bob-plugins history that its chrome work landed (commits naming
  bob-cli-qq.4). Then record the missing resolution as evidence with
  `sase bead note bob-cli-qq.4 "..."`. Re-closing a closed bead is a no-op. Never
  hand-edit its status.

### 7. Close bob-cli-qq (final step)

Bryan explicitly asked for this close, and this tale stands in for the stopped land
agent.

1. Run `sase bead epic-symbols bob-cli-qq`. Resolve or re-key every entry; none existed
   at planning time.
2. Run
   `sase bead close bob-cli-qq --note "<what steps 0–6 verified, with evidence refs and every follow-up outcome>"`.
   The note must say that the note #4 blocker is resolved, and how. Never use `--force`.
3. Run `just symvision` if `just --list` shows it in bob-cli. Otherwise record that the
   recipe is absent.
4. Set `status: done` in the frontmatter of `plan:202611/property_card_cabinets.md`. Use
   the PLAN path from `sase bead read bob-cli-qq`; the line currently reads
   `status: wip`.
5. bob-cli-qq has no `parent_bead`, so the landing ends here. Do not order the close
   after this work's own commit, push, or CI.

## Verification

- The new unit tests pass, along with the focused cabinets, host, passive, and
  card-floor suites. `npm run build:check` and `npm run docs:check` are green.
- `npm test` passes except for known failures that also fail on clean master.
- The athena run is GO, and its evidence is retained against bob-cli-qq.
- `bob plugins sync` is clean.
- bob-cli-qq is closed, and both cabinets plan files are `status: done`.

## Out of scope

- Any bob-cli code, JSON contract, release, or Bob Mac Capture change.
- Changing stored catalog bytes or the vault format.
- Cross-note heading links (`[[other#Projects]]`), which open another note; Ctrl+J/K
  Section Map jumps (bob-cli-rp); and Live Preview link clicks. If a Live Preview
  same-note link click does not open the cabinet, file it through `/sase_new_task`
  rather than fixing it here.
- The work of bob-cli-rk, bob-cli-rj, bob-cli-rm, bob-cli-rn, and bob-cli-r5.
- Re-running the retired cabinet-vs-inline perf budget; the GO evidence on bob-cli-qq.8
  holds it.
