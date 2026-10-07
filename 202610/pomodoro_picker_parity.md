---
tier: tale
title: Pomodoro picker parity fixes
size: small
goal:
  "Fix three capture-parity bugs in the Link to today picker, then close epic
  bob-cli-54. Per-cap plan meters stay independent, an exact canonical Pomodoro name
  never offers a duplicate, and a #N query does not match a shorter position token."
proposed_by: bbugyi200.apollo.bob-cli-54.land
bead: bob-cli-54
status: done
---

- **PARENT:**
  [202610/ctrl_shift_enter_pomodoro_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ctrl_shift_enter_pomodoro_picker.md)
- **BEAD:**
  [bob-cli-54](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-54/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-54.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-54.land.md)
- **COMMITS:**
  - [1b09a2f](https://github.com/bobs-org/bob-cli--plans/commit/1b09a2fa8c2bee72c39a4c6024a18df460e16eef)
    — docs(plans): mark pomodoro picker epic plan done

# Pomodoro picker parity fixes

Epic `bob-cli-54` is otherwise complete. Phases `bob-cli-54.1`, `bob-cli-54.2`, and
`bob-cli-54.3` are closed. Their notes were implemented: pure target helpers in
`plugins/block-id-prompt/src/075-pomodoro-link-targets.js`, the
`PomodoroLinkPickerModal` and `bid-ppk` styles, and the Ctrl+Shift+Enter wiring through
`122-plugin-pomodoro-link-picker.js` and `124-plugin-pomodoro-inbox-route.js`, with
notices, harness stubs, version `1.24.0`, and docs. Fifty-seven picker tests passed on
the tree that already contains the later freshness rename. No child note contains
`PROPOSED FOLLOW-UP:`. The plan's out-of-scope list was declined as task beads in a note
already on `bob-cli-54`. This tale is only the three bugs below, plus this epic's
closeout.

Open `bob-plugins` with
`sase repo open bob-plugins -r "Fix Link to today picker parity"` and edit only the path
it prints. Read that repo's `AGENTS.md` first. Edit fragments, never hand-edit
`main.js`. Keep every hand-edited fragment at or below 1000 lines.
`075-pomodoro-link-targets.js` is 949 lines, so keep the new helper short.

## 1. Keep per-cap over flags independent

In `plugins/block-id-prompt/src/130-plugin-task-link-open-and-notices.js`,
`readPlanBudgetMeter` sets each of `themes.over` and `links.over` to
`meter.over === true || budget.status === "over"`. Ledger-tools sets `status: "over"`
when either cap is exceeded and sets each meter's `over` from that meter alone
(`count > cap`). Copying `status` onto both meters makes
`PomodoroLinkPickerModal.createThemeOver` turn the create-row badge red
(`+1 theme · 2/3`) when only the link cap is over. The badge is red only when the theme
cap is over. The footer meter stays red when the plan is over either cap.

Return `themes.over` and `links.over` from those booleans only. Keep the combined `over`
field true when `budget.status === "over"` or either per-cap `over` is true, so the
footer and the Notice suffix still redden for either cap.

Add a test in `scripts/test-block-id-prompt-plan-budget-and-freshness.cjs` using the
existing `stubPlanBudgetApi` helper. Stub a budget whose themes are
`{ count: 2, cap: 3, over: false }`, links are `{ count: 11, cap: 10, over: true }`, and
`status` is `"over"`. Call `plugin.readPlanBudgetMeter` on a string and assert
`themes.over` is false, `links.over` is true, and `over` is true. Add the mirror case
(themes over, links not) and assert `themes.over` is true and `links.over` is false.

## 2. Match exact canonical names

Capture stores the em-dash tail as written and selects an open name by slug, but this
epic's contract suppresses a create row only for an exact canonical name (a prefix slug
still keeps the create row). These four comparisons use `entry.name === canonical.name`,
so a stored `Deep Work` does not match canonical `DEEP WORK` and the picker offers a
duplicate:

- `buildPomodoroLinkPickerRows` exact-open suppression and the completed `againOf`
  lookup
- `resolvePomodoroLinkCreateIntent` exact-open result
- `planNewPomodoroLinkInsertion` race lookup that must link into an open entry of that
  canonical name instead of creating another

Add a small helper next to `canonicalizePomodoroLinkName` that returns the canonical
name for an entry's `name`, or `""` when the entry has no name or the name is not a
valid Pomodoro name. Use it for those four comparisons. Do not change slug ranking:
exact slug still beats prefix, and a prefix such as `mem` against `MEMORY` still appends
the create row.

In `scripts/test-block-id-prompt-pomodoro-targets.cjs`, add coverage that:

- an open `- [ ] () — Deep Work` plus query `deep work` returns only the existing row
  and `resolvePomodoroLinkCreateIntent` returns that entry
- a completed `- [x] () — Taxes` plus query `taxes` still yields `againOf` `Taxes`
- planning `{ kind: "new", name: "DEEP WORK" }` while `- [ ] () — Deep Work` is already
  open returns `matchedExisting: true` and does not insert a second entry

## 3. Bound #N position matches

`rankPomodoroLinkEntry` treats a query as a position hit when
`query.includes("#" + position)`. Query `#10` therefore also matches Pomodoro `#1`, and
query `#1!` matches `#1` (the existing test requires that second case: an invalid name
that contains a position token shows the match and not the invalid row).

Match `#N` only when the character after the token is missing or is not a digit. `#10`
matches position 10 and not position 1. `#1!` still matches position 1. `#2` still
matches position 2. Add those assertions beside the existing position test in
`scripts/test-block-id-prompt-pomodoro-targets.cjs`.

## 4. Verify the plugin

From the bob-plugins checkout:

1. `npm run build`
2. `npm test`
3. `npm run validate`
4. `bob plugins sync`

Do not run `just check-full`. Do not run `just check` unless a bob-cli source file
changes. This tale does not change bob-cli source.

## 5. Close epic bob-cli-54

This closeout is the last step. Do not wait for this tale's own commit, SHA, push, or CI
result. Do not use `sase bead close --force`.

1. Run `sase bead epic-symbols bob-cli-54`. There are no `--epic-symbol` entries today.
   If any appear, resolve each one (wire it, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy) or re-key that Justfile line to a
   still-open later bead that still needs the exemption. Do not leave an entry keyed to
   `bob-cli-54` or a closed phase.
2. Close the epic:

```bash
sase bead close bob-cli-54 --note "Verified phases bob-cli-54.1, bob-cli-54.2, and bob-cli-54.3 against source and commits 881b9ad, c4b42a0, and d0680d1 in bob-plugins plus 9c825ca docs in bob-cli. The pure target model, Link to today modal, Ctrl+Shift+Enter wiring, destination notices, version 1.24.0, and docs match the epic plan. Fifty-seven picker tests passed before these parity fixes; npm test, npm run validate, and bob plugins sync passed after them. No child note had a PROPOSED FOLLOW-UP. The plan's out-of-scope items were not filed. Commits after the epic started outside its own commits (ledger date-mark repair, date-marks docs, gkeep pull, capture ref grammar, and the tickler freshness rename) do not touch the picker and needed no integration edit. Landed three parity fixes: per-cap meter over flags stay independent, exact canonical names suppress duplicate creates including mixed-case stored names, and a #N query no longer matches a shorter position. epic-symbols clean."
```

3. Run `just symvision` from the bob-cli checkout. The recipe is absent from this repo's
   justfile today. If it is still absent, record that in the final response and
   continue. If it exists, it must pass.
4. Set `status: done` in the frontmatter of the epic plan
   `plan:202610/ctrl_shift_enter_pomodoro_picker.md`. Open the plans sidecar with
   `sase repo open plans -r "Mark the pomodoro picker epic plan done"` and edit only the
   path it prints. Change the existing `status: wip` line to `status: done`. Do not
   change other frontmatter.
5. Run `sase bead read bob-cli-54 -r "Need the parent link"`. The epic has no
   `parent_bead`. If that is still true, finish. If a parent phase or plan bead is
   present, follow the landing parent rules: close a completed parent phase normally,
   or, for a still-complete parent plan, retire its `--epic-symbol` entries, close it
   with a recheck note, run `just symvision` when available, mark its plan file done,
   and repeat through directly parented plan ancestors. Stop at the first incomplete
   parent, note the blocker on that bead, and report it.
