---
tier: tale
size: medium
title: 'Land copy chips: bring the Copy card C row in line with the plan, then close
  bob-cli-ni'
goal: The Mod+Y Copy card shows a C chip row only when a chip can be resolved at the
  cursor, in its own CHIP group after TASK. The row copies through the card's normal
  path and Notice, then flashes the chip. Its tests, e2e, and docs match. Epic bob-cli-ni
  is then closed.
proposed_by: bbugyi200.apollo.bob-cli-ni.land
proposed_with: claude/opus
bead: bob-cli-ni
status: done
---

- **PARENT:**
  [202611/copy_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/copy_chips.md)
- **BEAD:**
  [bob-cli-ni](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ni/README.md)

# Land copy chips: fix the Copy card C row and close bob-cli-ni

## Context

Epic `bob-cli-ni` (plan `plan:202611/copy_chips.md`) shipped copy chips. Its land agent
verified phases ni.1, ni.2, ni.3, ni.5, and ni.6 against the plan. It also triaged every
proposed follow-up (see the `LANDING TRIAGE` note on `bob-cli-ni`).

The one remaining piece of epic work is phase ni.4, commit `0a4a91f`
`feat(navigation): add copy-chip row to Copy card` in bob-plugins. It drifted from the
plan's "Phase: copycard" section:

| Plan (copycard phase)                                                                                                                      | What 0a4a91f shipped                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A `chip` group **after the TASK group**, one row with id `chip` and key `c`                                                                | Row id `copy-chip` in the **file** group, so the keys read `P A F N C I T`                                                                                                  |
| **Ready:** label "Chip · <face>"; the value is the payload previewed with `⏎` and middle truncation                                        | Label "Copy chip"; preview "label · payload"                                                                                                                                |
| **Ambiguous:** a dimmed row "N chips on this line · move onto one"; pressing `C` nudges it                                                 | Same (keep this)                                                                                                                                                            |
| **None, plugin absent, or API call failed:** **no group at all**                                                                           | Always a dimmed row ("No copy chip at the cursor" / "Bob Copy Chips is not enabled")                                                                                        |
| Feature-detect `api.copyChips.version >= 1` and `chipAt`                                                                                   | Requires `version === 1` **and** `copyChip`                                                                                                                                 |
| Copy through the existing `copyCardRow` path, keeping the card's "Copied …" Notice, then call `copyChips.flash(chip, {view})` when exposed | Bypasses the clipboard path via `api.copyChip` and suppresses the card Notice for this row                                                                                  |
| A template keyed `c` diagnoses "key c is Chip's"                                                                                           | Diagnoses "key c is Copy chip's"                                                                                                                                            |
| Minor manifest bump for the touched plugin                                                                                                 | No bump (`bob-navigation-hotkeys` is still 2.89.0, set by an unrelated commit)                                                                                              |
| Update the model, keys, and view tests; extend `scripts/obsidian-copy-card-e2e.cjs`                                                        | Two assertion tweaks; no e2e. `scripts/test-navigation-copy-card-model.cjs:547` ("row order is file builtins, file templates, then TASK group") now fails deterministically |
| Update `docs/copy-card.md`: Keys table, reserved list `P A F N I C`, and one cross-link                                                    | Not touched                                                                                                                                                                 |

The plan is the approved spec, so this tale restores it.

Reviewer decisions on the epic still bind:

- no memory note may be edited (the `glossary:copy-chip` strand already landed and stays
  accurate);
- no bob-cli code change;
- no Bob Mac Capture change.

## Where the work lands

**bob-plugins.** Open it with `sase repo open bob-plugins -r "<why>"` and work only in
the printed path. Read its `AGENTS.md` first. The rules that matter:

- Edit `src/` fragments only, never a generated `main.js`.
- Keep every hand-edited fragment at or under 1000 lines.
  `plugins/bob-navigation-hotkeys/src/101-copy-card-model.js` is already at about 970
  lines, so put new chip logic in `src/101b-copy-card-chips.js` (or a new `101c-…`
  fragment registered in `src/fragments.json`).
- Run `npm run build` after source edits.

Files involved:

- `plugins/bob-navigation-hotkeys/src/101-copy-card-model.js`: the
  `COPY_CARD_BUILTIN_ROWS` entry and the row assembly in `buildCopyCardModel`;
- `src/101b-copy-card-chips.js`: `buildCopyCardChipRow`;
- `src/103-copy-card-modal.js`: the dividers;
- `src/641-plugin-copy-card.js`: `getCopyCardCopyChipsApi`, `resolveCopyCardChip`,
  `openCopyCard`, and `copyCardRow`;
- `plugins/bob-navigation-hotkeys/manifest.json`;
- `scripts/test-navigation-copy-card-model.cjs` and
  `scripts/test-navigation-copy-card-view.cjs`;
- `scripts/obsidian-copy-card-e2e.cjs`, its `-lib.cjs`, and the
  `test-obsidian-copy-card-e2e.cjs` self-test wrapper;
- `docs/copy-card.md` and `docs/bob-copy-chips.md`.

The chip API is `app.plugins.plugins["bob-copy-chips"].api.copyChips`. It is implemented
in `plugins/bob-copy-chips/src/090-plugin.js` and documented in
`docs/develop/plugin-apis.md`.

- `chipAt({lines, line, ch})` returns
  `{state: "none" | "ambiguous" | "ready", chip?, count}`.
- A chip is `{form: "inline" | "block", label, text, line, from, to}`.
- `flash(chip, {view})` shows success feedback without writing the clipboard.
- The API is frozen and never throws.

## Steps

### 1. Model: a CHIP group after TASK, present only when there is something to show

- Rename the builtin spec to `{id: "chip", key: "c", label: "Chip", needs: []}`.
  - Keep `c` in `COPY_CARD_RESERVED_KEYS`, so it stays reserved even while the row is
    hidden.
  - With the label "Chip", the existing collision code yields "key c is Chip's".
- Rewrite `buildCopyCardChipRow(spec, result)` so it returns `null` (no row) unless
  `result.state` is `"ready"` with a chip whose `text` has a non-whitespace character,
  or `result.state` is `"ambiguous"`.
  - **Ready:**
    - `group: "chip"`, `state: "ready"`, `value` = `chip.text` exactly (never trimmed);
    - `copyChip: chip`;
    - `label` = `Chip · <face>`, where face is the trimmed `chip.label` when non-empty,
      otherwise the payload. Pass the face through `copyCardPreviewText` (newlines
      become `⏎`) and `copyCardMiddleTruncate` with a short limit (about 40 characters);
    - `preview` gets the same payload preview treatment the template rows use (`⏎` plus
      middle truncation).
  - **Ambiguous:**
    - `group: "chip"`, `state: "unavailable"`, `label: "Chip"`, `value: null`;
    - `reason` and `preview` are `` `${count} chips on this line · move onto one` ``.
- In `buildCopyCardModel`, collect the chip row separately and append it last. The row
  order becomes: file builtins, file templates, diagnostic, task builtins, task
  templates, chip.
  - `defaultRowId` logic is unchanged: Path stays the `↵` default.

### 2. Plugin wiring: feature-detect, resolve, and copy through the card path

- **`getCopyCardCopyChipsApi(app)`.** Accept the API when `Number(api.version) >= 1` and
  `typeof api.chipAt === "function"`. Do not require `copyChip`.
- **`resolveCopyCardChip`.** Return `null` (no group) when:
  - the API is absent;
  - there is not exactly one selection;
  - the head is invalid;
  - the call throws;
  - the result is `none`.

  Pass `ready` and `ambiguous` results through unchanged (with `count`). Keep calling it
  under the same guards as the task resolver.

- **`openCopyCard`.**
  - Drop the special case that suppresses the "Copied …" Notice for the chip row. Every
    successful row shows `formatCopiedNotice(row.value)`.
  - Keep capturing `editorView` (`editor.cm`) for the flash.
- **`copyCardRow(row, context)`.**
  - Remove the `api.copyChip` branch, so the chip row copies `row.value` through the
    existing clipboard path like every other row.
  - After a successful write of a row with `id === "chip"` and a `copyChip`, call
    `api.flash(row.copyChip, {view: editorView})` inside `try/catch`, only when
    `typeof api.flash === "function"`.
  - A flash failure never turns a successful copy into a failure.

### 3. Modal: a CHIP divider that never impersonates TASK

- In `103-copy-card-modal.js`, before the first row with `group === "chip"`, render a
  divider:
  - the same markup as the TASK divider (rule, label, rule), with label text `CHIP` and
    no title element;
  - an extra class `is-chip` on the divider root (`bob-copy-card-divider is-chip`).
- Give the existing TASK divider no new class. Update anything that finds "the" divider
  (the e2e `cardProbeExpression` uses `root.querySelector('.bob-copy-card-divider')` to
  infer the task state) so it selects `.bob-copy-card-divider:not(.is-chip)`.
- Check `101a-copy-card-keys.js`:
  - `C` on a ready chip row copies it;
  - `C` on the ambiguous row nudges and announces the reason, like the other dimmed
    rows;
  - `↑`/`↓`/`Ctrl+N`/`Ctrl+P` skip the dimmed row.

### 4. Version

Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version from whatever it
holds when you start (currently `2.89.0` → `2.90.0`). Leave the description alone.

### 5. Unit tests

Update `scripts/test-navigation-copy-card-model.cjs`:

- **Builtin table:** the ids end with `"chip"`, and the reserved keys are
  `["p","a","f","n","i","c"]`.
- **The failing row-order test:** with no `copyChip` input, the order is
  `["path","absolute","file","note","template-2","block-id","template-1"]` (no chip
  row). Add a sibling case with a ready chip whose chip row comes last, after
  `template-1`.
- **New cases:**
  - **Ready unlabeled chip:** label `Chip · <text>`, value byte-exact including
    backticks and trailing spaces, and `copyChip` carried.
  - **Ready labeled chip:** label `Chip · <label>`.
  - **Block-form chip with a newline:** the preview shows `⏎`, and the value keeps the
    LF.
  - **Ambiguous:** a dimmed row with the "N chips on this line · move onto one" reason,
    and `C` resolves to a nudge, not a copy.
  - **No row** for `none`, for `null` (plugin absent), and for a whitespace-only text.
  - **Template collision:** a `copy.templates` entry keyed `c` diagnoses
    `key c is Chip's` and stays copyable by `↑↓ ↵`.
  - **Default row:** Path is still the default when a chip row exists.

Update `scripts/test-navigation-copy-card-view.cjs`:

- the existing render test without a chip expects the keys `P A F N I T`;
- add a render test with a ready chip: keys `P A F N I T C`, and one `.is-chip` divider
  labelled `CHIP` after the TASK rows;
- add a render test with an ambiguous chip: the dimmed C row.

If the plugin-level copy path has unit coverage (search
`scripts/test-navigation-copy-card-*.cjs` and other `scripts/test-navigation-*.cjs` for
`copyCardRow`), add a case:

- a chip row copies `row.value` through the clipboard stub;
- the success Notice text is produced;
- `flash` is called once with the chip;
- a throwing `flash` still reports success.

### 6. Real-Obsidian e2e

Extend `scripts/obsidian-copy-card-e2e.cjs` (and `-lib.cjs`):

- **Fixture.** `makeFixture` also copies `plugins/bob-copy-chips` into the scratch vault
  and lists `bob-copy-chips` in `community-plugins.json`. The seed note gains:
  - one line with a single double-backtick chip whose payload contains backticks and a
    trailing space, e.g. ``` ``copy[deploy]: run `just check` now `` ```;
  - one line with two chips.
- **Cases:**
  1. Cursor on the single-chip line, `Mod+Y` then `C`:
     - the clipboard equals the payload byte for byte;
     - a `Copied …` Notice shows;
     - the row label starts with `Chip · deploy`.
  2. Cursor on the two-chip line, between the chips:
     - the C row is dimmed with `2 chips on this line · move onto one`;
     - pressing `C` leaves the clipboard unchanged.
  3. Disable the plugin over CDP (`app.plugins.disablePlugin("bob-copy-chips")`), then
     reopen the card: there is no C row and no CHIP divider.
  4. The existing cases still pass. Adjust any row-count or key-order expectations, and
     keep the task-state probe on the TASK divider only.
- **Self-test.** Keep `--self-test` (`scripts/test-obsidian-copy-card-e2e.cjs`) passing
  offline.
- **Running it.** Use `xvfb-run -a` in an isolated scratch vault, never the live vault.
  - This host may have no Obsidian or `xvfb-run`; apollo lacks both. In that case, run
    the e2e on athena against a snapshot of the built checkout, as bob-cli-ni.2 and
    bob-cli-ni.5 did. Read the `tailnet.md` reference memory with `/sase_memory_read`
    before using Tailscale SSH.
  - If neither is possible, say so plainly in the close note. Never claim the e2e
    passed.

### 7. Docs

Update `docs/copy-card.md`:

- **Intro.** Mention the optional `C` chip row.
- **Keys table.** Add a row: `C` | Chip (CHIP group; needs Bob Copy Chips) | the payload
  of the copy chip at the cursor, e.g. the text inside
  ``` ``copy[deploy]: run `just check` now `` ```.
- **Reserved list.** Change it to `P A F N I C`.
- **Row order.** The sentence becomes: file rows, file-only templates, the `TASK` group,
  then the `CHIP` group.
- **Resolution.** A short paragraph on how the chip is resolved:
  - it uses the same cursor rule as `gy` through `api.copyChips.chipAt`;
  - ready shows a live row, ambiguous a dimmed row, and none or a missing plugin no
    group;
  - the copy shows the card's Notice and flashes the chip.
- **Collision example.** Add `key c is Chip's` to the collision list.
- **Cross-link.** Exactly one sentence linking to `bob-copy-chips.md`: "Authored
  payloads live on copy chips; live context lives on the Copy card."

Update `docs/bob-copy-chips.md`:

- **Gestures table.** Add two rows:
  - `gy` (normal mode; mapped in the vault vimrc to `copy-on-cursor-line`): copy the
    chip at the cursor and flash it;
  - `Mod+Y` then `C` (Copy card): copy the chip at the cursor, previewed first.
- **Closing sentence.** Reword the sentence near the end of Authoring so it matches the
  Copy card doc ("…live context lives on the Copy card, whose `C` row copies the chip at
  the cursor.").

### 8. Verify and deploy

In the bob-plugins checkout, run:

- `npm run build`, `npm run build:check`;
- `npm test`: the only acceptable failures are the known load-sensitive ledger-tools
  budgets tracked as `bob-cli-km` (event-link-times) and `bob-cli-io` (frees-marks), and
  each must pass when rerun in isolation;
- `npm run validate`, `npm run docs:gen`, `npm run docs:check`, `git diff --check`.

Then deploy:

- Run
  `bob plugins sync --repo "<bob-plugins checkout>" --no-pull -p bob-navigation-hotkeys`.
  On a refusal, follow `AGENTS.md` and `docs/develop/deployment.md`; never pass `-F` on
  your own.
- Confirm that `bob plugins list` shows `bob-navigation-hotkeys` at the new version,
  synced and enabled.

There is no bob-cli source change, so `just check` is not required. Do not run
`just check-full`.

### 9. Close out epic bob-cli-ni (final step)

Do all of this in the same turn as the code. Do not wait for a commit SHA, a push, or
CI.

1. Run `sase bead epic-symbols bob-cli-ni`.
   - For every listed `--epic-symbol` entry, resolve the symbol: wire it up, privatize
     it, add a non-test pragma, or delete it per the Symvision epic-whitelist policy.
   - Re-key the Justfile line to a still-open bead only when that bead still needs the
     exemption.
   - At planning time it listed none.
2. Run `sase bead close bob-cli-ni --note "<verification>"`. The note summarizes:
   - the landing verification of ni.1–ni.6 (see the `LANDING TRIAGE` note);
   - the C row fixes from this tale, with the test, e2e, and deploy evidence;
   - the follow-up outcomes:
     - `bob-cli-nm` and `bob-cli-nn` created;
     - `bob-cli-km`, `bob-cli-io`, and `bob-cli-cd` corroborated;
     - the stale test and copy-card docs folded into this tale.

   If the close is rejected for leftover epic-symbol entries, finish that cleanup and
   close again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` in the bob-cli workspace and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path that
   `sase bead read bob-cli-ni -r "Need the plan path"` prints, i.e.
   `plan:202611/copy_chips.md`, currently `status: wip`.
5. Run `sase bead read bob-cli-ni -r "Need the parent link"` and check `parent_bead`.
   - At planning time `bob-cli-ni` had **no** parent, so normally nothing more is
     needed.
   - If a parent phase appears, verify this work completed it and close only that phase
     with `sase bead close <parent> --note "<what you verified>"`.
   - If a parent plan appears, recheck it fully before closing, and stop with a note on
     the first incomplete parent.
6. Tell Bryan to reload Obsidian on each host, and to run `bob plugins sync` on the Mac
   and athena so the new navigation build reaches them.
