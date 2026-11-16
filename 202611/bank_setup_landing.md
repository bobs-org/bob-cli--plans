---
tier: tale
size: medium
title:
  "Land bank setup (bob-cli-lt): Setup e2e, real-Obsidian proof, scoped hide, docs
  budget, closeout"
goal:
  Finish the remaining land-agent acceptance work for epic bob-cli-lt and close it. That
  means real-Obsidian e2e proof of the Setup header, the + New category flow, and the
  Bank Setup card; a truly scoped raw-row hide; the bob-ledger-tools reference page back
  under its word cap; green gates and deployment; and a recorded closeout.
proposed_by: bbugyi200.athena.bob-cli-lt.land
proposed_with: claude/opus
bead: bob-cli-lt
status: done
---

- **PARENT:**
  [202611/bank_setup_render_and_edit.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_setup_render_and_edit.md)
- **BEAD:**
  [bob-cli-lt](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lt/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-lt.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-lt.land.md)
- **COMMITS:**
  - [ae73186](https://github.com/bobs-org/bob-cli--plans/commit/ae731865b0bdf1f75068ee0f44760a27418cb942)
    — docs(plans): mark bank setup epic and landing tale done

# Land bank setup (bob-cli-lt): e2e, real-Obsidian proof, scoped hide, docs budget, closeout

## Context

Epic `bob-cli-lt` ("Bank setup: render the bank: mapping on the hub, edit it through
bob") has all eight phase beads closed. The land agent verified their code against the
epic plan `plan:202611/bank_setup_render_and_edit.md`:

- **bob-cli:** `bob bank config [show]`, `bob bank config update` with every op,
  refusal, inverse, and guarantee, and the shared `yaml_spans` / `config_spans` editors.
- **bob-plugins:** Setup header (`248a`–`248c`), New category stage (`248d`–`248g`,
  `242aa`), Bank Setup card (`248i`–`248l`), and the Command Mode row.

The land agent also committed these integration edits in bob-cli:

- `bob bank subscriptions` now emits located config findings.
- `category.remove` and its Undo guard now count subscription records that name the
  category.

It closed `bob-cli-lq` and triaged every phase follow-up (see the `LAND TRIAGE` note on
`bob-cli-lt`). Do not redo that work.

This tale finishes what is left. When the epic plan's "Shared design contract" and
"Integration and land-agent acceptance" sections disagree with this tale, they are
normative except where this tale narrows them. Read the plan first with
`sase artifact read plan:202611/bank_setup_render_and_edit.md "Need the normative design contract and land acceptance list"`.

**Repository discipline.** Open bob-plugins only with
`sase repo open bob-plugins -r "<reason>"`, use the path it prints, and read its
`AGENTS.md`.

- Edit `plugins/bob-ledger-tools/src/` fragments and `src/fragments.json`, then run
  `npm run build`. Never hand-edit `main.js`. Every hand-edited fragment stays at or
  below 1000 lines.
- Register every new test file explicitly in `package.json` `test:unit`.
- After changing bob-plugins, run `bob plugins sync` and follow `AGENTS.md` when sync
  refuses an overwrite. Never pass `-F` on your own.
- Read `docs.md` with `/sase_memory_read` before any docs edit, and `sase_artifacts.md`
  before recording artifacts.
- In bob-cli, never edit the `Cargo.toml` version or `CHANGELOG.md`.
- The epic's DECISIONS are final:
  - `mobile = degrade`;
  - no `decisions:` record;
  - no glossary strand;
  - no other memory edits.

## 1. Make the raw `bank` row hide truly scoped to the Setup surface

**Defect.** The setup-header phase added the scoped `bob-bank-setup-surface` hide, but
the legacy hide still applies to whole leaves:

- `plugins/bob-ledger-tools/src/241p-plugin-bank-marks.js` `bankMarkSyncHubProperty`
  still adds `bob-bank-hub-note` to every hub leaf's containers.
- `styles.css` still hides
  `.bob-bank-hub-note .metadata-property[data-property-key="bank"]` next to the
  `.bob-bank-setup-surface` selector.

So a hub pane hides the native row even where no Setup header is mounted, which is the
original "hidden with nothing in its place" problem.

**Fix.**

- Delete the `.bob-bank-hub-note …` selector, so only `.bob-bank-setup-surface` hides
  the row.
- `bob-bank-hub-note` has no other consumer. Remove the class writes in
  `bankMarkSyncHubProperty` and its unload cleanup, along with the method and its
  callers (`241p` lines ~132, ~396–403, ~529).
- Update `scripts/test-ledger-tools-bank-marks.cjs` (~489–499) and
  `scripts/test-ledger-tools-bank-setup.cjs` (~800–805) to assert that the hide is owned
  only by `bob-bank-setup-surface`.
- Update the `hubClass` probe in `scripts/obsidian-bank-ui-e2e-lib.cjs` (~989) to drop
  the body fallback.

## 2. Bring the bob-ledger-tools reference page back under its cap

`BOB_CLI_DIR=<this bob-cli checkout root> npm run docs:check` in bob-plugins fails with
`docs/reference/plugins/bob-ledger-tools.md: over budget at 6302 words (cap 6000)`.

The source is bob-plugins `docs/bob-ledger-tools.md`. The overage comes from this epic's
Setup header, Setup card, and New category paragraphs on top of the parallel
subscriptions work, and the epic notes require fixing it before landing.

- Trim existing prose across the page to at most 6000 words, aiming for about 5,950.
- Do not raise any budget. Do not drop any behavior, default, keybinding, refusal, or
  contract link.
- Condense duplicated wording first, especially in the bank sections. Preserve the
  section anchors that inbound links use, or update their callers.
- Run `npm run docs:gen` if the inventory changes.
- If you reach 5,700 or fewer words, add a note on `bob-cli-il` (the standing 300-word
  headroom task) with the count. Do not close it.

## 3. Setup scenarios in the real-Obsidian bank e2e

Add `scripts/obsidian-bank-ui-e2e-setup.cjs` so the existing 1741-line run and its 420 s
budget stay unchanged.

- Reuse `obsidian-startup-smoke-lib.cjs` and `obsidian-bank-ui-e2e-lib.cjs`: scratch
  vault and profile, owned Obsidian under `xvfb-run -a`, CDP probes, never the live
  vault. Resolve `bob` the same way (`BOB_BIN` or `~/.cargo/bin/bob`).
- Writer scenarios need a git scratch vault. Follow
  `scripts/obsidian-album-schema-writer-e2e.cjs`, the album-writer precedent.
- Add an offline `--self-test` with wrapper
  `scripts/test-obsidian-bank-ui-e2e-setup.cjs`, registered in `package.json`, modelled
  on `scripts/test-obsidian-bank-ui-e2e.cjs`.

Each scenario runs against real bob JSON:

1. **Live-config shape:** one account, flow-map categories with quoted digit hotkeys, no
   rules.
   - The header mounts once after the H1 in Reading and Live Preview.
   - Collapsed grammar is correct, and it expands on click.
   - Header tiles match the Category card tile for tile: built-ins, hidden, order,
     dashed derived keycaps, and the `hotkey: none` dot.
   - The native `bank` row is hidden only on the surface.
2. **Invalid config:** a duplicate hotkey plus a rule with an unknown `cat`.
   - The header auto-expands, both findings are located (line + key path), and clicks
     jump to Source mode at those lines.
   - Source-mode lint marks appear.
   - The queue card reads "bank: has N problems — see Setup above".
3. **Missing mapping:** `Setup · no bank: mapping yet`, with Copy starter and ↗ YAML;
   nothing is written.
4. **Unrelated `bank` property** on another note: its native row stays visible.
5. **Add-then-file:** from a waiting row, open the Category card and use `+` → label →
   Enter.
   - Exactly one `bank: config (+<key>)` commit.
   - The flow-style config gains exactly one flow-style line; every other byte is
     identical.
   - The row is filed with one editor undo step on the month note.
6. **Setup card:**
   - Open through the header's `⊞ Edit` and the "Edit bank setup" palette command.
   - Relabel and hide a category; each is a previewed consequence line, then one commit.
   - Ctrl+Z previews and applies the inverse, restoring the original bytes.
   - Ctrl+D on a used category paints bob's `category_in_use` refusal and writes
     nothing.
   - Add an account, then add a rule.
   - Ctrl+E opens the YAML excursion; Done returns only on a valid config.

Also automate these acceptance items as checks in the same script where CDP can drive
them:

- **Properties round trip.** Set `summary` through `app.fileManager.processFrontMatter`,
  the same re-serialization the Properties panel uses. bob still reports `state: valid`
  with an unchanged `fingerprint`, and the header still renders. Then a writer edit on
  that block-style config stays block style.
- **Two-pane race.** A Source-mode edit with a delayed save between preview and Enter
  refuses as `stale_config` and never corrupts the file.
- **Byte identity.** Cancelled, refused, and no-op flows leave `bank.md` byte-identical.
  Painting never writes (hash before and after).
- **Plugin disable.** Disabling bob-ledger-tools restores the native `bank` row.

## 4. Real-Obsidian acceptance and screenshots

1. Run the new script and the existing `scripts/obsidian-bank-ui-e2e.cjs`.
   - Capture screenshots of the collapsed, expanded, and invalid header in light and
     dark at about 375 px, the Category card's New category stage, and each Setup card
     section.
   - Open every screenshot with the Read tool and inspect it against the epic plan's
     "Visual language". Fix defects in this epic's surfaces.
2. Check keyboard-only operation and screen-reader names: `aria-expanded`, tile labels
   like "Coffee, key C, 5 rows", and a live region for the invalid banner.
3. The existing script's `chip-today` failure is pre-existing and tracked by
   `bob-cli-le` (hard-coded `FIXTURE_TODAY`). Do not fix it here; cite it.
4. File any other failure that reproduces identically on a clean base through
   `/sase_new_task`, naming `bob-cli-lt` as the proposer.

## 5. Gates, deployment, and records

- **bob-plugins:**
  - `npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`, and
    `BOB_CLI_DIR=<this bob-cli checkout root> npm run docs:check`, which must now pass.
  - Known load-only timing flakes `bob-cli-km`, `bob-cli-ki`, and `bob-cli-ju` may fail
    in the full suite. Rerun the failing file alone and cite the bead.
- **bob-cli:** run `just check` only if you changed bob-cli code. Do not run
  `just check-full`.
- **Deploy:** run `bob plugins sync` and verify the deployment per
  `docs/develop/deployment.md` in bob-plugins.
- **Record:** register the e2e reports and screenshots as SASE artifacts (per
  `sase_artifacts.md`) and reference them from a `sase bead note bob-cli-lt` summary of
  the checks, screenshots, and deployment.

## 6. Closeout (final step)

1. Run `sase bead epic-symbols bob-cli-lt`. It reported none at planning time; resolve
   or re-key any entry that appears.
2. Run `sase bead close bob-cli-lt --note "<what you verified>"`. The note covers:
   - phases verified;
   - integration edits;
   - bob-cli-lq closed;
   - follow-ups triaged (+1 on km/ki/ju, new m4/m5, m3 owns dismissal, declines);
   - the scoped-hide fix;
   - the docs page word count;
   - the e2e and acceptance results with artifact refs;
   - gates and deployment.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` if the recipe exists. It does not exist on current bob-cli
   master; if it is still missing, say so in the close note.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path shown by
   `sase bead read bob-cli-lt`, `plan:202611/bank_setup_render_and_edit.md`).
5. `bob-cli-lt` has no `parent_bead`, so finish normally after that.
