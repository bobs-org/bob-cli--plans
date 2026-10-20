---
tier: tale
title: Fix Task Card relation labels saving as "true"
goal: A relation label typed in the Task Card relation field (⇧↵ on the Links to or
  Linked from stage) is written to the LINKS line exactly as typed, never as `true`.
size: small
decisions:
  mac_deploy:
    ask: Commit and push the fix mid-turn so the MacBook's own `bob plugins sync`
      deploys it in this turn?
    choices:
      push_then_sync: The Mac gets nav 2.50.1 this turn; Bryan only reloads Obsidian
      leave_for_bryan: No mid-turn commit; Bryan runs `bob plugins sync` on the Mac
        after the commit lands
    default: push_then_sync
    why: The bug was hit on the Mac, whose bare sync pulls origin, so only a pushed
      fix reaches it
    answer: push_then_sync
proposed_by: bbugyi200.athena.12k
proposed_with: claude/opus
decided_by: auto
status: done
---

# Fix Task Card relation labels saving as "true"

## Diagnosis (established; do not re-derive)

**Symptom.** Bryan linked `bar baz buz` to `foo bar baz` from the Task Card
(Ctrl+Shift+P → `l` → ⇧↵) and typed `son of`. Both chips show `TRUE`. The vault bytes in
`sase.md` explain why: the stored label really is `true`.

```
- [ ] #task foo bar baz … ^foo-bar-baz
	- 🪢 **LINKS:** [[mac_inbox#^buy-abby-bball-hoop]]
	- 🪢 **LINKED FROM:** [[#^bar]] (true)
- [*] #task bar baz buz [created::2026-10-20] ^bar
	- 🪢 **LINKS:** true [[#^foo-bar-baz]]
```

Chip rendering (bob-ledger-tools), the mirror reconcile, and the line writer all behave
correctly. The bad value enters before the writer.

**Root cause.** The bug is in the bob-plugins repo, in
`plugins/bob-navigation-hotkeys/src/426-picker-relation-stage.js`, inside
`commitRelationLabel` (about line 787):

```js
label =
  item.typed || item.label
    ? normalizeRelationLabel(item.typed || item.label || "")
    : null;
```

`getRelationLabelPreviewItems` builds the typed-input row as
`{ kind: "relation-label", label: "son of", typed: true, … }`. Here `typed` is a boolean
flag that also drives the row's `is-create` styling; it is not the text. So
`item.typed || item.label` evaluates to `true`, and `normalizeRelationLabel(true)` runs
`String(true)`, giving `"true"`. That string passes every Rule 2 check. The typed row
comes first and is selected by default, so every typed label commits as `true`. Picking
an existing suggestion row (`typed: false`) or the plain row works, which hid the bug.

**Reproduced.** A throwaway Node harness ran against both the repo build and the
deployed vault copy (nav 2.50.0). For raw input `"son of"`,
`getRelationLabelPreviewItems()[0]` is
`{"kind":"relation-label","label":"son of","typed":true,…}`, and `commitRelationLabel`
hands `commitRelationEdit` the label `"true"`.

**Scope.** Both directions (`l` Links to, `⇧L` Linked from) and both modes (add from
RESULTS, relabel from CURRENT) share `commitRelationLabel`, so all four are affected.
Capture `~` / `~~` is unaffected: it runs through bob-cli's Rust grammar, and Bob Mac
Capture goes through bob. No other `normalizeRelationLabel` caller passes a non-string,
and `item.typed ||` appears nowhere else.

**Why tests missed it.** The labelled-link tests (LG7, LG12) call `commitRelationEdit`
directly with a ready label. LG3 only checks refusal copy. No test drives
`getRelationLabelPreviewItems` → `commitRelationLabel`.

## Work (bob-plugins linked repo)

Open the repo with your `/sase_repo` skill (`sase repo open bob-plugins -r "<reason>"`)
and read its `AGENTS.md`. All paths below are relative to that checkout. Edit only
`src/` fragments. Never hand-edit the generated `main.js`, and keep each hand-edited
fragment at or below 1000 lines.

1. **Write the failing test first.** Add the LG3 "add, typed label" test from step 4a.
   Run `npm run build` and that test file. Confirm it fails with `true` in the LINKS
   line before you change any code.
2. **Fix `commitRelationLabel`** (`426-picker-relation-stage.js`). A non-plain
   `relation-label` row commits `normalizeRelationLabel(item.label)`. Never read `typed`
   as text; it stays a boolean styling flag. Keep the existing refusal Notice when the
   result is `null`, and leave the plain-row and no-row (raw input) branches as they
   are. Add at most a one-line comment saying the label text is always `item.label`.
3. **Harden `normalizeRelationLabel`** (`085-relation-line-grammar.js`). Return `null`
   for any non-string `value` before the trim and collapse work. As a result, the
   planner's `normalizeOptionalRelationLabel` (`086-relation-line-plan.js`) refuses a
   non-string label as `invalid-label` instead of writing `String(value)`.
   `null`/`undefined`/`""` still mean "no label" there, because that function checks for
   them first. Leave `normalizeRelationReaderLabel` unchanged.
4. **Regression tests.** Do not weaken any existing assertion.
   - In `scripts/test-navigation-relation-links-stage.cjs`, add `LG3:` tests that drive
     the real relation field.
     - Use `makePlugin` with open `TestEditor`s and `makeRelationPicker`.
     - Enter the field with `showRelationLabelStage([target], { mode })`. If its
       `applyOptions`/DOM calls do not run in this harness, set the three fields it sets
       instead: `stage = "relation-label"`, `relationLabelTargets`, and
       `relationLabelMode`.
     - Stub `getRawQuery` to return the typed text. Take the row from
       `getRelationLabelPreviewItems()` and `await picker.commitRelationLabel(row)`.

     Cover these cases:
     1. Add with typed `son of`. The source reads `son of [[Target#^target]]` and the
        target mirror reads `(son of)`. Assert that neither note contains `true`.
     2. Typed `son   of:` normalises to `son of`.
     3. Relabel mode, starting from an existing `see [[Target#^target]]` entry. The
        entry becomes `son of [[Target#^target]]` and the mirror is relabelled.
     4. The `"from"` (Linked from) direction with a typed label. The label lands on the
        other task's LINKS line.
     5. The other paths still work:
        - a suggestion row (`typed: false`, label `see also`) commits `see also`;
        - the plain row commits no label;
        - `commitRelationLabel(undefined)` with raw `son of` commits `son of`.

   - In `scripts/test-navigation-relation-lines.cjs`, assert that
     `normalizeRelationLabel(true)`, `(1)`, and `(null)` all return `null`.
   - In `scripts/test-navigation-relation-writer.cjs`, extend LW11: a `label: true` add
     gives `ok: false` with `reason: "invalid-label"` and leaves the content unchanged.

5. **Version and build.** Bump `plugins/bob-navigation-hotkeys/manifest.json` from
   `2.50.0` to `2.50.1`; earlier fixes also took patch bumps. Run `npm run build`, then
   `npm run docs:gen` to refresh the version in `docs/index.md` and
   `docs/bob-navigation-hotkeys.md`. No other doc text changes: `docs/task-relations.md`
   already describes LG3 correctly.
6. **Validate.** `npm run build:check`, `npm test`, `npm run validate`, and
   `npm run docs:check` must all pass.
7. **Deploy on athena** (bob-plugins `AGENTS.md` requires it). Run
   `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`, and
   confirm bob-navigation-hotkeys reads `2.50.1` and is synced.

Commit subject: `fix(navigation): save typed relation labels instead of "true"`. Name
the root cause (the boolean `typed` flag read as label text) in the body.

## MacBook rollout

> [!decision] mac_deploy = push_then_sync Commit the bob-plugins change now with
> `/sase_git_commit`. Bryan's approval of this plan is the explicit request to commit it
> before the Mac deploy. Confirm the commit is on `origin/master`. Then run
> `ssh mac 'bob plugins sync && bob plugins list'`; the bare sync pulls the Mac's
> bob-plugins checkout first. Confirm nav shows `2.50.1`. Do not kill or relaunch
> Obsidian, because Bryan is using it. In the final report, ask him to reload Obsidian
> (command palette → "Reload app without saving") so the new `main.js` loads.

> [!decision] mac_deploy = leave_for_bryan Do not commit mid-turn and do not touch the
> Mac. Finalization commits the change. In the final report, give Bryan the Mac steps:
> after the commit is on `origin/master`, run `bob plugins sync && bob plugins list`
> (expect nav `2.50.1`), then reload Obsidian.

**Reaching the Mac (push_then_sync only).** Read `tailnet.md` with `/sase_memory_read`
first; SSH to `mac` is best-effort. Retry
`ssh -o ConnectTimeout=20 -o BatchMode=yes mac true` in one foreground loop for up to
about 10 minutes (for example, 10 tries with `sleep 60` and a matching Bash timeout). If
it never answers, skip the Mac step and give Bryan the exact commands from the
`leave_for_bryan` branch.

## Left for Bryan (not in scope)

- **The existing bad entry.** The `bar baz buz` → `foo bar baz` relation still stores
  `true`. A vault scan found no other `true` label: only `sase.md` lines 29 and 31. Do
  not edit the vault. It is Bryan's scratch pair, and the note is open on the Mac. After
  reloading, Bryan repairs it with the fixed gesture, which doubles as the live
  acceptance check: Ctrl+Shift+P on `bar baz buz` → `l` → the CURRENT `foo bar baz` row
  → ⇧↵ → type `son of` → ↵. Both chips should then read `son of`, and the mirror should
  read `(son of)`.
- **Unrelated doc issues** already noted on the bob-cli-a2 epic are out of scope: the
  unquoted multi-word capture labels in `docs/guides/link-related-tasks.md`, and the
  stale `bob capture-parse --help` text.
