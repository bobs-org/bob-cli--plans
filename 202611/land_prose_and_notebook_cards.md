---
tier: tale
size: medium
title: Land the Prose and Notebook Cards epic
goal: 'Prove both cards in real Obsidian against the re-landed Bob Prose, finish the
  preview-gate removal, and close bob-cli-qm.

  '
proposed_by: bbugyi200.athena.bob-cli-qm.land
proposed_with: claude/opus
bead: bob-cli-qm
status: done
---

- **PARENT:**
  [202611/prose_and_notebook_cards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/prose_and_notebook_cards.md)
- **BEAD:**
  [bob-cli-qm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qm/README.md)

# Plan: Land the Prose and Notebook Cards epic (bob-cli-qm)

## Why

All seven phases of `bob-cli-qm` (`plan:202611/prose_and_notebook_cards.md`) are closed,
and the cards are live.

- **Deployed.** `bob plugins list` reports drift 0 across 12 plugins:
  - `bob-ledger-tools` 1.172.4, the Notebook and Prose Cards;
  - `bob-articles` 0.4.0, Bob Prose.
- **Live note.** The vault's `sase_blog_contents.md` is now `type: "[[prose]]"`.

The land agent verified the work and found three gaps that this epic caused. All three
must close before the epic can.

1. **The prose acceptance rows never ran against a real Bob Prose.**
   - When `bob-cli-qm.6` ran acceptance, the `bob-cli-qj.3` prose-plugins commit had
     never reached bob-plugins master. Bob Prose tracked only `[[article]]` notes and
     had no `toggle-section-numbers` command.
   - bob-plugins `c41e07e9` (`bob-cli-qj.4`) re-landed that work after `bob-cli-qm.7`
     went live. Both spellings now count, the plugin spawns `bob prose`, and
     `bob-articles:toggle-section-numbers` is registered. That is exactly the command id
     the Prose Card dispatches (`PROSE_NOTE_CARD_NUMBERING_COMMAND` in
     `plugins/bob-ledger-tools/src/249n-plugin-prose-note-card-host.js`).
   - `scripts/obsidian-note-cards-e2e.cjs` still encodes the old world, in three places:
     - `prose-identity-no-snapshot` expects `[[prose]]` notes to stay identity-only;
     - `prose-legacy-refusal-line` expects bob to refuse the legacy `[[article]]`
       fixture;
     - the numbering-chip block's comments say the chip is inert until `qj.3`.
   - So the epic acceptance items below were never shown in real Obsidian: "the measure
     row comes from Bob Prose", "the numbering chip toggles", and outline/finding jumps.
2. **Forced colors were never actually emulated.**
   - Around line 993, `scripts/obsidian-note-cards-e2e.cjs` sends
     `Emulation.setEmulatedMedia({ forcedColors: "active" })`. That is not a CDP
     parameter, so Chromium ignores it.
   - The runner still records `forced-colors-emulation` as passing because the
     screenshots were captured. `bob-cli-qm.6` noted the shots were full-color.
   - The family contract requires both cards to work in `forced-colors`.
3. **The preview gate is only half deleted.**
   - `propertyCardKindLive(plugin, kind)` in
     `plugins/bob-ledger-tools/src/240t-plugin-property-card-note-species.js` now only
     checks `PROPERTY_CARD_KINDS.includes(kind)`.
   - The Notebook and Prose hosts still call it with their own constant kinds, so it is
     always true: dead indirection named like a gate.
   - The epic's go-live step said to delete the gate.

Everything else was verified and needs no change: phase notes, source, commits,
docs-check in both repos, the focused unit suites, and deploy state. The follow-up
triage is already recorded on `bob-cli-qm` as the `LAND TRIAGE` note, with these
outcomes:

- `bob-cli-oc` closed;
- `bob-cli-qt` and `bob-cli-cd` corroborated with +1s;
- a `DISCOVERED ISSUE` recorded on `bob-cli-qq`;
- new feature task `bob-cli-qw`.

Do not redo that triage.

## Governing decisions

- `decisions:prose-notes-are-owned-prose`:
  - `[[prose]]` and the legacy `[[article]]` spelling both count forever.
  - Copy says "prose note".
  - Plugin ids and wire names keep their legacy names (`bob-articles`,
    `bob-articles:toggle-section-numbers`).
- `decisions:bob-cli-releases-ride-a-daily-train`: this tale changes no Rust or CLI
  code. Never edit `Cargo.toml` versions or `CHANGELOG.md`.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the existing Esc/`q`/Ctrl+`[` ladder rows
  stay in the runner unchanged.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture changes nothing.

## Rules

- **Opening bob-plugins.** Work in bob-plugins only through `/sase_repo`
  (`sase repo open bob-plugins`), use only the printed path, and read its `AGENTS.md`
  first.
  - Edit `src/` fragments and run `npm run build`; never hand-edit `main.js`.
  - Keep each fragment at or below 1000 lines.
  - On a bundle conflict, take upstream, rebuild, and re-bump.
  - Other agents land on bob-plugins master concurrently, in particular the cabinets
    go-live `bob-cli-qq.9`. Fetch and rebase before committing.
- **Before you start.** Read these with `/sase_memory_read`:
  - the `obsidian.md` reference memory, before anything touches the vault;
  - `sase_artifacts.md`, before retaining evidence;
  - `tailnet.md`, if you run on athena.
- **Live vault.** Never open the live vault in an automated Obsidian run. Use only the
  runner's disposable vault built from `scripts/fixtures/note-cards/`.
- **athena's `/tmp`.** The tmpfs is full (tracked by `bob-cli-qt`). Never delete other
  agents' `/tmp` leftovers; that belongs to Bryan. Put every scratch tree, cargo target,
  and Electron temp directory on disk. Keep the paths short (for example
  `/var/tmp/qmland-<suffix>`), because a long `TMPDIR` overflows Chromium's
  SingletonSocket path (`bob-cli-np`).
- **Long commands.** Run them in the foreground with a large explicit timeout; this
  session cannot be woken later. Long commands include cargo builds, `npm test`, and
  real-Obsidian runs.
- **Bob Prose.** Change nothing in `plugins/bob-articles`.

## Steps

### 1. Delete the vestigial gate shim (bob-plugins)

1. Remove `propertyCardKindLive` from `240t-plugin-property-card-note-species.js`.
   Remove its export from `src/374-exports-property-card.js`.
2. Drop every call site, keeping the surrounding logic intact. Re-grep first, because
   line numbers drift.
   - `249i-plugin-notebook-card-host.js`: the
     `propertyCardKindLive(this, NOTEBOOK_CARD_HOST_KIND) !== true` clause, about
     line 107.
   - `249n-plugin-prose-note-card-host.js`: the clauses at about lines 108 (`match:`),
     158, 389, 433, and 543.
   - Each removed clause was always true for these constant kinds, so mount behavior
     must not change.
3. `grep -rn propertyCardKindLive plugins/ scripts/ docs/` must return nothing outside
   generated `main.js`, and after the rebuild nothing at all.
4. Bump the `bob-ledger-tools` manifest patch version. Run `npm run build` and
   `npm run build:check`.

### 2. Refresh the note-cards acceptance runner (bob-plugins)

Edit `scripts/obsidian-note-cards-e2e.cjs` and its offline wrapper
`scripts/test-obsidian-note-cards-e2e.cjs`.

1. **Prose ready-snapshot rows.** Replace the identity-only assumptions with assertions
   for what Bob Prose now does.
   - **`[[prose]]` with a title.** `prose-with-title.md` reaches a ready snapshot. The
     measure row paints, from bob's numbers, parts such as `≈ N words`, `M min read`,
     `S sections`, and `K to write`. It never paints `0 words` or `0 sections`.
   - **Legacy spelling.** The legacy `[[article]]` fixture also reaches a ready
     snapshot, since both spellings count.
   - **Refused note.** The `^prj` fixture (`prose-with-prj.md`) shows bob's own refusal
     message as the one muted measure line, never zeros.
   - **When bob disagrees.** Before writing an assertion, run the runner's `BOB_BIN` on
     the fixture directly to learn what bob actually says, for example
     `bob prose format --stdin <path> -f json`. If bob's real behavior differs from
     `docs/prose.md`, stop and record a note on `bob-cli-qm`; do not loosen the
     assertion.
   - Rename or replace the stale check ids (`prose-identity-no-snapshot`,
     `prose-legacy-refusal-line`, `prose-prj-identity-no-crash`) so the report reads
     truthfully.
   - Delete every comment that says `qj.3` never reached master or that the chip is
     inert until `qj.3`.
2. **Outline and findings.**
   - After `_` (or the fold button) expands `prose-with-title.md`, the outline rows
     render. The `to write` badges follow the accepted `stub_chip = leaf` rule: stub
     sections with no deeper section; umbrella headings are not counted.
   - Clicking an outline row moves the Live Preview cursor to that section's heading
     line.
   - Clicking a finding row (use a fixture with a finding) moves the cursor to the
     reported line.
3. **Numbering chip, live path.**
   - With Bob Prose enabled, `toggleCommand` is true and the chip is a `BUTTON`.
   - One click flips `§ numbered` ↔ `§ unnumbered`, and the card repaints.
   - The note change is one undoable transaction: one editor `undo()` restores the note
     text byte-for-byte.
   - Toggle a fixture that is not in the protected-hash set. Otherwise exclude that one
     deliberate write from the passive hash check, and say so in the report detail.
   - Keep the inert branch (`SPAN`, an "unavailable" tooltip, a click that does nothing)
     and run it inside the existing "Bob Prose disabled" row, where the command really
     is absent.
4. **Forced colors.**
   - Replace the invalid call with
     `Emulation.setEmulatedMedia({ media: "screen", features: [{ name: "forced-colors", value: "active" }] })`.
     Reset with `features: []`.
   - Before each forced-colors shot, evaluate `matchMedia("(forced-colors: active)")` in
     the page.
   - Record `forced-colors-emulation` as passing only when it matched for every
     forced-colors shot. On a mismatch, fail with the reason; never pass merely because
     the shots were captured.
   - `scripts/obsidian-cabinets-e2e.cjs` (around line 812) shows the working call shape.
5. **Electron temp.** Replace the hard-coded `TMPDIR: "/tmp"` in the Obsidian spawn env
   with `process.env.BOB_OBSIDIAN_TMPDIR || "/tmp"`. That matches the takeover runner
   (`scripts/obsidian-frontmatter-takeover-e2e.cjs`, `electronTmpDir`). The scratch
   vault already honors `TMPDIR` through `os.tmpdir()`.
6. **Offline coverage.** Extend `--self-test` and the wrapper so they cover:
   - the new check ids;
   - the forced-colors call shape (assert the `features` form, never `forcedColors`);
   - the `BOB_OBSIDIAN_TMPDIR` env selection.

   `node --test scripts/test-obsidian-note-cards-e2e.cjs` must pass.

### 3. Run real-Obsidian acceptance until GO

1. **Build bob.** Build a debug bob from current bob-cli master (the primary checkout;
   no source change) into a disk-backed target, for example
   `CARGO_TARGET_DIR=/var/tmp/qmland-target cargo build`. Point `BOB_BIN` at it and
   follow the runner's provenance contract (`BOB_BIN` / `BOB_CLI_DIR`).
2. **Run the full matrix.** Run `scripts/obsidian-note-cards-e2e.cjs` with:
   - `TMPDIR` and `BOB_OBSIDIAN_TMPDIR` set to short disk-backed directories;
   - `-o <shots dir>` and `-r <report.json>`;
   - `xvfb-run -a` as the runner's docs describe. Check `scripts/` and
     `docs/develop/validation.md` for the exact invocation.

   If Xvfb cannot start because `/tmp` is full, record that blocker on `bob-cli-qm` with
   the `df -h /tmp` output and stop before step 6. Do not delete other agents' files,
   and do not close the epic.

3. **Required GO.**
   - Every check passes, including the new prose ready-snapshot, outline, finding-jump,
     live-toggle, and true forced-colors rows.
   - The pre-existing notebook, takeover, single-owner, fail-open, focus, and passive
     rows still pass.
   - Offline self-tests or a skipped real-app run are not acceptance.
4. **Screenshots.**
   - Open and inspect every new shot with the Read tool, at minimum:
     - both cards, folded and expanded, with a ready measure row;
     - the refusal line;
     - the forced-colors shots for both cards.
   - In forced colors, check that text stays legible, the card and tile edges stay
     visible, and the `spine` and `folio` tile forms stay distinct.
   - Record a verdict, size, and SHA-256 per shot in the report.
   - Repair any visual defect in `plugins/bob-ledger-tools/styles.css`, for example a
     `forced-colors` media block using system colors, or a missing border. Rebuild,
     re-bump, and rerun the affected rows.
5. **Retain evidence.** Attach the evidence to the epic with
   `sase artifact create --bead bob-cli-qm`:
   - `report.json`;
   - the run log;
   - a tar of the shots plus their `SHA256SUMS`.

   Record the artifact refs.

### 4. Docs (bob-plugins)

Read the `docs.md` reference memory first.

- **`docs/prose-card.md`.** The `§` chip is described twice, in the paragraph ending
  "otherwise it stays inert" and again in "A `§` chip reads local `section_numbers`…".
  Merge them into one statement:
  - the chip runs Bob Prose's **Toggle section numbers** command, which flips
    `section_numbers` through bob in one undoable edit;
  - the chip is inert only when Bob Prose is absent, disabled, or older.
- **Other pages.** Leave other pages unchanged unless step 3's repairs change documented
  behavior.
- **Checks.** Run `npm run docs:check` and keep the generated inventory current.

### 5. Gates and deploy (bob-plugins)

1. Run these:
   - `npm run build:check`;
   - the focused suites (`test-ledger-tools-note-card-species`,
     `test-ledger-tools-note-card-coordinator`, `test-ledger-tools-notebook-card`,
     `test-ledger-tools-notebook-card-readouts`,
     `test-ledger-tools-prose-note-card-stub`,
     `test-ledger-tools-property-editor-focus-or-add`,
     `test-ledger-tools-property-card-passive-rendering`, `test-bob-articles`, and
     `test-obsidian-note-cards-e2e`);
   - the full `npm test`.
2. A pre-existing failure that reproduces on clean master is recorded in the epic close
   note, not fixed. The known one is the `obsidian-block-marks-e2e --self-test` ENOSPC
   on `/tmp/bm-e2e`, tracked by `bob-cli-qt`.
3. Commit through the normal finalizer flow, then run `bob plugins sync` without `-F`.
   Follow the `AGENTS.md` refusal protocol exactly.
4. Run the isolated post-sync startup smoke from `docs/develop/deployment.md`.
   `bob plugins list -f json` must show `drift=0`. Record the deployed bob-plugins SHA
   and plugin versions.

### 6. Close out bob-cli-qm (final step)

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-qm`; the land agent saw none.
   For every listed entry:
   - resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it
     per the Symvision epic-whitelist policy;
   - or, only when a still-open later bead needs the exemption, re-key that Justfile
     line to the open bead.
2. **Close the epic.** Run `sase bead close bob-cli-qm --note "<verification>"`. Build
   the note from the land agent's verification below plus this tale's results:
   - the runner and shim commits;
   - the real-Obsidian GO count and artifact refs;
   - the forced-colors verdict;
   - the deployed SHA and versions;
   - any pre-existing failures.

   Never pass `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, clean them up and close again.

   > Land agent verification (bob-cli-qm.land, 2026-11-27): all 7 phases closed done.
   >
   > **Commits.**
   >
   > - bob-plugins: deee0f21, 5152f493, d190ba21, 175515f2, 3863bc46, f6458cb0,
   >   31024085, fb72a96c, eca733c2.
   > - bob-cli: ebc45fec (docs) and eef6cc07 (the three approved glossary strands:
   >   prose-card, notebook-card, property-card; no other memory edited).
   >
   > **Source.** The source matches the plan:
   >
   > - 240t predicates and the precedence function, with the exhaustive test;
   > - the T1, T2, and T3 kit fixes, plus focus-or-add on loose keys;
   > - the notebook readouts on one `apiNotebookSubscribe` subscription;
   > - the Bob Prose `api.prose` v1 seam and its consumption;
   > - the passive-proof extension;
   > - the preview gate and `cardPreviewKinds` removed from ledger-tools;
   > - the `stub_chip = leaf` count.
   >
   > **Checks.**
   >
   > - The focused card suites are green on 56baeb9f.
   > - `just docs-check` (bob-cli eef6cc07) and `npm run docs:check` (bob-plugins
   >   56baeb9f) pass.
   > - Deploy drift was 0 (bob-ledger-tools 1.172.4, bob-articles 0.4.0).
   > - The live `sase_blog_contents.md` is `[[prose]]`.
   >
   > **Integration.**
   >
   > - c41e07e9 (`bob-cli-qj.4`) re-landed Bob Prose both-spellings detection and
   >   `bob-articles:toggle-section-numbers`, matching the card's command id.
   > - c6fbeb36 (catalogs out of embeds/hover) does not conflict with header absorption.
   > - dc96ef4c fixed the `vault-format.md` budget overflow from ebc45fec; `bob-cli-oc`
   >   is closed.
   >
   > **Follow-ups.** Triage is recorded in the epic's `LAND TRIAGE` note:
   >
   > - `bob-cli-qw` filed;
   > - `bob-cli-qt` and `bob-cli-cd` +1;
   > - `DISCOVERED ISSUE` on `bob-cli-qq`;
   > - the rest declined as resolved.

3. **Symvision.** Run `just symvision` if the recipe exists. Neither repo had one when
   this plan was written; if it is absent, say so.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file
   `plan:202611/prose_and_notebook_cards.md`. Use the PLAN path that
   `sase bead read bob-cli-qm -r "<why>"` prints.
5. **Parent bead.** `bob-cli-qm` has no parent bead, so nothing further closes.
6. **Final report.** Remind Bryan to:
   - run `bob plugins sync` on the Mac and on apollo;
   - glance at a notebook and at `sase_blog_contents.md` in his live vault. It should
     show a folded Prose Card with a measure row from Bob Prose and a working `§`
     numbering chip.

## Out of scope

- Any bob-cli Rust/CLI change, or any change to `plugins/bob-articles`.
- The cabinets runner's stale `cardPreviewKinds` seed. It belongs to `bob-cli-qq.9` and
  is recorded on `bob-cli-qq`.
- Cleaning athena's `/tmp` (`bob-cli-qt`), the bob-emoji vault tracking (`bob-cli-cd`),
  and the Prose Card provenance crumb (`bob-cli-qw`).
- Deleting the linked notebook header fallback.
