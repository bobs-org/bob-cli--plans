---
tier: tale
title: Move the Property Card editor gesture from == to =
goal:
  A bare `=` in Live Preview Vim normal mode opens the note's Property Card editor
  wherever `==` does today.
size: medium
decisions:
  habit_guard:
    ask:
      Swallow a fast second `=` after a bare `=` opens a card editor, so the old `==`
      habit opens it once?
    default: true
    why:
      every species editor binds bare `=`, so an unguarded `==` habit misfires on the
      next key
    answer: true
  period_callouts:
    ask:
      Bump period callouts to v4 so `bob periods ensure -R` can rewrite the ~59 vault
      callouts that say `==`?
    default: false
    why:
      a parenthetical key hint does not justify flagging every unreflected period note
      stale
    answer: false
  memory_glossary_cards:
    ask:
      Update the nine Property Card glossary strands so they name `=` as the card editor
      gesture?
    memory:
      - glossary:property-card
      - glossary:area-card
      - glossary:project-card
      - glossary:bank-card
      - glossary:intention-card
      - glossary:record-card
      - glossary:notebook-card
      - glossary:prose-card
      - glossary:ref-card
    corrects: "`==` opens the species editor"
    default: true
    answer: true
proposed_by: bbugyi200.athena.1h6
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-27 10:39:44
status: wip
---

# Move the Property Card editor gesture from `==` to `=`

## Goal

In Live Preview Vim normal mode, a bare `=` opens the current note's Property Card
editor (the "species editor": album schema card, Bank Setup, Record Card, Area/Project
Settings, Intention Card editor, Notebook/Prose focus-or-add, the Reference editor) on
every note where `==` does today. `==` stops being the gesture. Every hint, wire value,
doc, and glossary entry that teaches `==` for this gesture says `=` afterwards. Nothing
else about `-`, `_`, or `+` changes.

## Verified: `=` is free as a normal-mode prefix

Planning checked every place a bare normal-mode `=` could already be claimed:

- **Vault vimrc.** obsidian-vimrc-support reads `obsidian_vimrc.md` (its
  `vimrcFileName`). That file maps nothing that starts with `=`. Its gate note lists
  only `-`, `_`, and `+`.
- **Obsidian hotkeys.** `.obsidian/hotkeys.json` binds only Ctrl+=
  (`bob-navigation-hotkeys:open-child-note`). The gate refuses modified keys, so that
  hotkey is unaffected.
- **Plugin Vim maps.** None of the `vim.mapCommand` / `defineAction` maps start with
  `=`. Those maps are task-status-cycler (`<CR>`, `<C-CR>`, `<C-]>`, `o`, `O`, `#`,
  `<BS>`, `<C-d>`, `<C-u>`), bob-mail (`<M-CR>`), bob-navigation-hotkeys (`\s`, `<C-o>`,
  `<C-i>`), and bob-ledger-tools (`\p`, `\P`, `\o`, `\O`).
- **Editor-level `=` listeners.** The only one is the capture-phase listener in
  bob-plugins `plugins/bob-ledger-tools/src/248m-plugin-schema-double-equals.js`. Every
  other `=` handler runs only inside an already-open modal:
  - Record Card `=` opens the album schema.
  - Intention Card `=` goes to the parent subject.
  - Property Editor `=` hands off to the card.
  - Bank setup and category cards treat `=`/`+` as add.
  - Image zoom treats `=`/`+` as zoom in.
- **Capture grammar.** The relation alias `=`, the fast `==` → literal `=`, `=x` close
  selections, and the `==` Pomodoro override are text grammar in bob-cli capture and Bob
  Mac Capture (`decisions:relation-equals-alias-and-flips`). They are not Vim keymaps,
  and this plan leaves them untouched.

**What is given up:** CodeMirror Vim's `=` reindent operator, but only on notes that
resolve a Property Card through the registry: album, `bank.md`, canonical record, and
the registered extensions (area, project, day/period, notebook, prose, ref). Vim keeps
`=` everywhere else:

- counted `N=` / `N==`
- register-prefixed `"a=`
- operator-pending input
- multi-key prefixes (`g=`, `z=`, `f=`, `t=`, `F=`, `T=`, `r=`, `[=`, `]=`)
- visual, insert, and replace modes
- Source mode
- every non-card note

Today's bare-key gate for `-`/`_`/`+` already relies on this
keyBuffer/operator/count/register refusal.

## Governing decisions

- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` version or
  `CHANGELOG.md` edits. The props `editor` wire value changes, so the bob-cli commit is
  a `!` contract break (`feat(props)!: …`), matching how `51a50913 docs(props)!:`
  treated the previous props wire change.
- `decisions:relation-equals-alias-and-flips`: capture's `=`/`==` stays as it is. Its
  350 ms key-to-key window is reused for `habit_guard` so Bob has one "fast double `=`"
  timing.

No decision record governs Property Card keymaps (the decisions web excludes keymaps),
so this plan writes none.

## Work: bob-plugins (bob-ledger-tools)

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins`) and read its
`AGENTS.md`. bob-ledger-tools uses the committed source build. Edit `src/` fragments
only, then `npm run build`. Never hand-edit `main.js`. Keep each fragment at or below
1000 lines.

1. **Gate: `=` becomes a bare key.** In `248m-plugin-schema-double-equals.js`:
   - Route `=` through `propertyCardBareKeyGate`, extending its key check to `=`. Keep
     the same registry-backed arm (`canAct: Boolean(entry)`, `run` opens
     `entry.open({ path, frontmatter })`). Content-target, Live Preview, normal-mode, no
     operator, empty key buffer, no count, no register, and arm-actionable are all
     required. A miss never consumes the key, so non-card notes keep Vim's `=`.
   - Delete the pending-`indentAuto` path and the injected `<Esc>`
     (`Vim.handleKey(cm, "<Esc>", "mapping")`): Vim never sees the key now. Delete
     `propertyCardSecondKeyCommonGate` / `propertyCardSecondKeyGate` and the
     `canHandleKey` requirement if nothing else uses them.
   - Update or delete the `schemaDoubleEqualsGate` compatibility helper and its test
     export in `369-exports-schema-double-equals.js` so no remaining helper describes a
     pending-`=` second key.
   - Keep the cabinet-pick disarm for `=` and the
     `registerPropertyCardSecondKeyArm("+")` API unchanged.
   - Keep the `schemaDoubleEquals*` identifiers and fragment filename. A rename is
     internal churn that is out of scope. Rewrite the header comment so it describes the
     bare `=` gate. The old comment says a vimrc `nnoremap ==` never fires because `=`
     is an operator. That is no longer why: the capture-phase gate now claims `=` before
     Vim on card notes.
2. **Habit guard.**

   > [!decision] habit_guard Implement this step and its tests.
   - After a bare `=` acts, arm a one-shot guard. The next keydown is consumed
     (preventDefault plus stopPropagation/stopImmediatePropagation, deduped across the
     window and document listeners through the existing `WeakSet`) if it is an
     unmodified `=`, repeat or not, within 350 ms of the opening `=`. This applies
     whatever the target, including the just-opened modal.
   - Any other keydown disarms the guard without being consumed, so compositions like
     `=[r` still work.
   - Teardown clears the guard.
   - Use the event timestamp or an injectable clock so the unit tests stay
     deterministic.

   > [!decision] habit_guard = no Skip step 2 and its tests. A habitual `==` then also
   > fires the open editor's own `=`: the Record Card opens the album schema, the
   > Intention Card travels to the parent subject, and the Property Editor hands off.

3. **UI hints and wire defaults.** Change `==` to `=` in each of these:
   - `296b-plugin-intention-card-dom.js`: the ghost keycap.
   - `242j-plugin-record-sheet-dom.js`: `Edit fields (==, Ctrl+Shift+P)`.
   - `240o-plugin-property-card-foot-dom.js`: the foot label suffix `  ==`.
   - `249m-plugin-prose-note-card-dom.js`: `press == to set one`.
   - `249r-plugin-ref-note-card-dom.js`: `press == to add a title`.
   - `298e-plugin-property-editor-model.js` and `298f-plugin-property-editor-modal.js`:
     the hand-off `editor` default `"=="`.

   Then rewrite the gesture comments that name `==` (298z, 298m, 298s, 298v, 298w, 249s,
   249v, 297d, 297e) and the intention compositions: `[n==r` → `[n=r`, `==[r` → `=[r`.
   Find the rest with
   `grep -rn --exclude=main.js -E "==\[|\[n==|N==|press ==|\(==|  ==|\"==\"|'=='|\`==\`"
   plugins/\*/src`. Do not touch JavaScript `==`/`===`comparisons or the task-split`.split("==")`
   highlight-markup strip.

4. **Tests.** Rework `scripts/test-ledger-tools-schema-double-equals.cjs` so that:
   - A single `=` with no pending operator opens album, bank, record, and registered
     extension kinds, and injects no `<Esc>`.
   - Non-card notes (including `bank/2026-11.md` months and mixed record types) fall
     through.
   - Counted, register, operator-pending, visual, insert, replace, Source,
     outside-content, modified, composing, and defaultPrevented input falls through, and
     so does a pending multi-key prefix (`f`, `t`, `r`, `g`, `z`, `[`).
   - With `habit_guard` on: a second `=` within 350 ms is swallowed and opens nothing
     twice, a second `=` after 350 ms is not swallowed, and `=` then `[` disarms.

   Then update every other test or e2e harness that drives or asserts `==` for this
   gesture. Find them with `grep -rln -E "'=='|\"==\"|\`==\`|==\[r|\[n==r|N=="
   scripts`. The likely ones are `test-ledger-tools-property-editor-entrypoints.cjs`, the intention card navigation/acceptance tests, and `obsidian-ref-card-acceptance-e2e-flows.cjs`. Also check `obsidian-cabinets-e2e-lib.cjs`and`test-obsidian-cabinet-absorption-spike.cjs`, then any harness that sends two `=`
   keydowns.

5. **Docs** (`docs/`, mounted into the Bob site). Teach `=` wherever `==` names this
   gesture:
   - `property-editor.md`: including the `=` row, "owning `==` card editor" → "owning
     `=` card editor".
   - `ref-card.md`, `prose-card.md`, `notebook-card.md`, `area-and-project-cards.md`.
   - `record-card.md`: its key table fall-through list becomes counted `N=`, register
     prefixes, Source mode, insert and visual modes, and non-record notes keep Vim's
     `=`.
   - `intention-card.md`: the keycap and the `[n=r` / `=[r` compositions.
   - `albums.md`: its counted `N==` sentence becomes `N=`.
   - `bob-ledger-tools.md`, `develop/architecture.md`, `develop/validation.md`.
   - `bob-navigation-hotkeys.md`: replace "`nnoremap ==` yields to Vim's `=`" with the
     fact that a vimrc `=` map never fires on card notes because the gate claims `=`
     first.

   Mention the habit guard (when `habit_guard` is on) once in `property-editor.md` and
   `develop/architecture.md`. Leave unrelated `===` code samples alone.

6. **Verify and deploy.** In bob-plugins, run:
   - `npm run build`
   - `npm test` (includes `build:check`)
   - `npm run validate`
   - `npm run docs:check`

   Then run `bob plugins sync`. If sync refuses an overwrite, follow the bob-plugins
   `AGENTS.md` procedure (never `-F` on your own).

## Work: bob-cli

1. **Props wire value.**
   - `src/native/props/roles.rs` `card()`: set `editor` to `"="`.
   - `src/native/props/plan.rs` `refuse_protected`: change the `prop_owned` fallback
     hint to `"="`.
   - Update the roles unit test (`Some("==")`) and `tests/cli/props.rs`
     (`rating["editor"] == "=="`).
   - The plugin ignores this value: its protocol accepts any nullable string, and
     hand-off routes by `owner`. So the immediate plugin deploy and the next train
     release can land in either order.
2. **`docs/props.md`.**
   - The role table's `editor ==` cells, the "`prop_owned` with the `==` hint" sentence,
     and the bank hub's "editor `==`" all become `=`.
   - Add one short contract-break note beside the existing one (line ~223): card rows
     and `prop_owned` hints now report `"="`; earlier releases reported `"=="`.
3. **Period-note callouts.** In `src/native/periods/instructions.rs`, change the three
   `(\`==\`, then \`r\`)`phrases (week, quarter, year) to`(\`=\`, then \`r\`)`. Then
   update:
   - the `flat.contains(...)` test
   - the six `tests/fixtures/periods/scaffold-*.md` fixtures
   - `docs/periods.md` (line ~267)

   Keep the callout line wrapping valid: the fixtures show the reflowed text.

   > [!decision] period_callouts Also raise `INSTRUCTIONS_VERSION` to 4. Rename the
   > `weekly v3` / `quarterly v3` / `yearly v3` / `all-time v3` labels and their
   > fixtures and tests. Update `docs/periods.md` §Agent instruction callouts for v4.
   > Then run `bob periods ensure -R --dry-run` against the vault and report the count,
   > without running the real refresh. Bryan runs it.

4. **Glossary memory.** Under an accepted `memory_glossary_cards`, change only the `==`
   that names this gesture to `=` in the listed strands, and leave capture-alias's `==`
   alone. Then run `sase memory init`, then `just docs-gen` to regenerate
   `docs/reference/glossary.md`, then `just docs-check`.
5. **Leave alone:** the capture-grammar `==`/`=` in:
   - `src/native/capture*`, `src/native/capture_language/*`
   - `docs/capture.md`, `docs/reference/capture-syntax.md`,
     `docs/reference/cli/capture*.md`
   - `docs/getting-started.md` (Dataview inline-code note)
   - `docs/task-split.md` (highlight markup)
   - `sase/memory/glossary/capture-alias.md`
   - the relation decision record

   Also leave the vault's `obsidian_vimrc.md` untouched (its gate note stays true), and
   touch no `Cargo.toml` version or `CHANGELOG.md`.

6. **Verify.** Run `just all` (fmt, lint, test) and `just docs-check`. Commit the
   bob-cli work as a `feat(props)!:` change whose body names the `editor` / `prop_owned`
   hint value change. Commit bob-plugins separately as `feat(ledger-tools): …`.

## Acceptance

- On album, bank hub, record, area, project, day/period, notebook, prose, and ref notes
  in Live Preview Vim normal mode, one `=` opens the same editor `==` opened before. No
  pending-operator state is left behind and no `<Esc>` is injected.
- On every other note, and for counted, register, prefixed, visual, insert, Source, and
  modified input, `=` behaves exactly as stock Vim.
- With `habit_guard` on, a fast `==` opens the editor once and never fires the editor's
  own `=`.
- `[n=r` and `=[r` land in yesterday's reflection without a literal `r`.
- No UI hint, doc, glossary strand, or bob-cli props output teaches `==` for this
  gesture. Capture-grammar `==` is unchanged.
- bob-plugins: `npm test`, `npm run validate`, and `npm run docs:check` pass, and
  `bob plugins sync` deployed the build. bob-cli: `just all` and `just docs-check` pass.
