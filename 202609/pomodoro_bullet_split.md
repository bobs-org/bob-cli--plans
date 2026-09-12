---
tier: tale
title: Split the last merged Pomodoro from selected bullets
goal:
  Typing ++ in the Pomodoro sub-bullet picker can atomically peel the last merge-derived
  name into a new placeholder Pomodoro containing exactly the selected bullet subtrees.
size: medium
proposed_by: bbugyi200.athena.0k5.f1
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0k5.f1](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.0k5.f1.md)
- **COMMITS:**
  - [6eb8e71](https://github.com/bobs-org/bob-plugins/commit/6eb8e7120a5986802d046124344cf06838146390)
    — feat(navigation-hotkeys): split merged pomodoro bullets

# Plan: Split the last merged Pomodoro from the Ctrl+Shift+M bullet picker

## Objective and scope

Extend the existing Pomodoro **sub-bullet** destination picker so that entering the
exact trimmed query `++` and confirming its dedicated row with Enter splits the last
component from a previously merged Pomodoro. The counted or bare Ctrl+Shift+M invocation
already selects the cursor bullet plus the requested following siblings; those selected
subtrees become the children of a new placeholder Pomodoro named for the last merged
component, while the source Pomodoro keeps all unselected content and is renamed to the
remaining components.

This is a `tale`, size `medium`: the work is a substantial but bounded extension of the
pure Pomodoro planner, one picker, its guarded same-file commit path, regression tests,
documentation, versioning, and deployment. It belongs entirely in
`bobs-org/bob-plugins`'s `bob-navigation-hotkeys`; no bob-cli capture/timer contract,
other plugin, memory note, or daily note needs to change.

Preserve all existing Ctrl+Shift+M routing and behavior except the exact `++` query in
sub-bullet mode. In particular, keep ordinary task moves, Pomodoro entry move/rename,
Ctrl+X entry merge, existing-destination bullet moves, typed new-name creation, the
single-`+` same-name shortcut, count/clamping semantics, duplicate suppression for
ordinary moves, and names containing non-delimiter plus characters such as `C++`.
Entry-mode `++` remains an ordinary rename/filter query; the split shortcut is confined
to `PomodoroBulletMovePickerModal`.

## Repository and implementation context

Before reading or editing the plugin repository, use `/sase_repo` to open `bob-plugins`;
if its linked short name remains unavailable, open `gh:bobs-org/bob-plugins` and use
only the returned checkout. Read that repository's `AGENTS.md`. Its CommonJS `main.js`
is source code loaded directly by Obsidian, and its instructions require
`bob plugins sync` after changes.

Read the Pomodoro glossary through `/sase_memory_read` before relying on ledger grammar,
and use an audited artifact read of `plan:202609/pomodoro_entry_merge.md` as the merge
contract being reversed. The current implementation baseline is commit `ae4e37e`:
`planPomodoroEntryMerge` joins normalized names with canonical `+` separators and
appends absorbed bullet subtrees, but it does not retain provenance relating a name
component to particular bullets. Therefore the user's selected sibling range is
authoritative for split content; the implementation must not guess a boundary from
bullet text, task links, count of names, or append position.

Relevant files and symbols in the opened plugin repository are:

- `plugins/bob-navigation-hotkeys/main.js`: `parsePomodoroEntryLine`,
  `normalizePomodoroName`, `discoverMovablePomodoroBulletTargets`,
  `planPomodoroBulletMove`, `planPomodoroEntryRename`,
  `createPomodoroBulletMovePickerRows`, `PomodoroBulletMovePickerModal`,
  `openPomodoroBulletMovePicker`, `commitPomodoroBulletMoveSession`, transaction/focus
  helpers, notice builders, and the exported `helpers` test surface.
- `scripts/test-navigation-hotkeys.cjs`: `TransactionEditor`, the Pomodoro picker
  harness, the existing `+`/`++` row assertions, bullet-move planner tests, and guarded
  picker commit tests.
- `plugins/bob-navigation-hotkeys/manifest.json`, `README.md`, and `package.json`:
  plugin version/docs and the existing test/validation commands. The explored version is
  `1.35.0`; choose the next feature version from the implementation checkout's actual
  version so intervening changes are not overwritten.

The focused exploration baseline
`node --test --test-name-pattern='Pomodoro|pomodoro|move picker' scripts/test-navigation-hotkeys.cjs`
passes 125 tests with 254 skipped and no failures. No product files were changed during
planning.

## Behavior contract

### Name decomposition and eligibility

Treat a name as merge-derived when its normalized value contains at least one
whitespace-delimited plus separator, matching what Ctrl+X merge writes. Split at the
**last** such separator only. Normalize and validate the whole source name and both
resulting names with the existing Pomodoro name rules; require nonempty prefix and
suffix. This gives these results:

| Source name              | Remaining source name | New Pomodoro name |
| ------------------------ | --------------------- | ----------------- |
| `BUILD + REVIEW`         | `BUILD`               | `REVIEW`          |
| `BUILD + REVIEW + EMAIL` | `BUILD + REVIEW`      | `EMAIL`           |
| `C++ + REVIEW`           | `C++`                 | `REVIEW`          |

Do not split on adjacent pluses inside a component: `C++`, `A+B`, and a lone name have
no merge boundary and must yield one non-actionable invalid row explaining that the
source has no final merged Pomodoro to split. Exact trimmed `++` is now reserved for
this shortcut in sub-bullet mode, so it must no longer create a Pomodoro literally named
`++`. A query such as `C++` remains an ordinary typed name. Repeated use peels one
component at a time from the right.

The source must be a parsable named Pomodoro entry in the note's `## Pomodoros` section,
its captured header and every selected target must still match, and the name must meet
the existing 48-character/em-dash/whitespace rules. Do not add a new checkbox state
restriction: a merged source may be current, future, or may have closed since the merge,
just as the existing sub-bullet picker can operate on closed entries. Always reconstruct
the split-off entry as `- [ ] () — LAST`, because Ctrl+X only absorbs a future
placeholder and deliberately keeps no second time range. Keep the source's checkbox
status, exact placeholder/time body, emphasis, and `[t:: ...]` metadata.

If parsing, normalization, stale identity, target capture, or content safety fails,
return the original input unchanged and show a split-specific explanation. Do not fall
through to creating a literal `++` Pomodoro or perform a partial rename/move.

### Content, placement, and atomicity

Move exactly the selected top-level sibling bullets at the invocation depth, each with
its full descendant/continuation subtree and in original order, using the existing count
and end-of-Pomodoro clamping behavior. Preserve duplicates: split is a structural
inverse operation and must never collapse two selected subtrees. Promote a selected
nested sibling to the new Pomodoro's child indent using the existing rebasing rule,
matching ordinary bullet moves.

The original source entry always survives with the remaining name and all unselected
content, even when the selection consumes every owned substantive bullet. Do not delete
its header or transfer its time range. Do not invent an empty child placeholder when
none remains, but preserve any unselected empty/other content according to the existing
subtree and seam rules. Create the split-off placeholder entry immediately after the
surviving source block, before the next Pomodoro; put the selected subtrees under it.
Preserve LF/CRLF, terminal-newline state, unrelated headings/entries, blank seams,
markers, indentation within descendants, task links, and surrounding bytes.

For example, selecting the two `EMAIL` bullets in this merged current entry and then
confirming `++`:

```markdown
- [ ] (**0920-0950** [t:: 30m]) — BUILD + REVIEW + EMAIL
  - build work
  - review work
  - email one
    - follow-up
  - email two
```

produces one atomic edit:

```markdown
- [ ] (**0920-0950** [t:: 30m]) — BUILD + REVIEW
  - build work
  - review work
- [ ] () — EMAIL
  - email one
    - follow-up
  - email two
```

The selection is trusted even when it is not a trailing range: the name carries only
component order, not bullet ownership. Repeated splitting is therefore deterministic but
user-directed.

Derive rename, source preservation, insertion, and bullet movement in one pure plan,
then apply the complete `before`/`after` once with `applyEditorContentTransaction`. The
edit must form one undo group. Revalidate active file path, editor identity, full
captured content, raw source header, and target lines before writing. Close the picker
before committing through the existing `opening` latch, restore/focus the source editor,
and place the cursor on the first moved bullet in the new Pomodoro with its column
clamped. Failed or stale commits write nothing and must not emit a success notice.

### Picker and feedback

For exact trimmed `++` in bullet mode, return one dedicated `split` row such as
`Split off EMAIL`, with metadata that the selected bullets will create a new Pomodoro
below and the source will become `BUILD + REVIEW`. Confirmation is still Enter/click;
typing by itself must not mutate the note. Invalid source names return one invalid row
instead of destination matches or new-name rows. Recompute the decomposition in the pure
commit planner rather than trusting row metadata.

Update the bullet picker title/placeholder/subtitle or footer hints concisely so `+`
means same-name copy and `++` means split last merged name, while Enter remains the
confirmation key. The success notice should state selected/clamped bullet count, the new
Pomodoro name, and remaining source name, for example
`Split 2 bullets into new Pomodoro EMAIL; source is now BUILD + REVIEW`. Failure notices
must say that nothing was split. Other pickers must not acquire a split action.

## Implementation steps

1. Add a small pure name-decomposition helper near the Pomodoro name/merge helpers. It
   should recognize the last canonical whitespace-delimited `+` boundary, normalize and
   validate the source/prefix/suffix, and return frozen structured metadata or a clear
   error. Export it through `helpers` for direct tests. Extend
   `createPomodoroBulletMovePickerRows` only in bullet mode so exact trimmed `++`
   produces the dedicated split/invalid row before ordinary new-name and destination
   filtering; leave `+`, `C++`, and entry mode unchanged.
2. Add a pure `planPomodoroBulletSplit` (or equivalently named) helper that accepts the
   frozen source entry identity and selected targets, re-discovers and validates the
   source, decomposes its name, and composes source rename plus fresh-placeholder
   insertion/movement entirely in memory. Reuse `planPomodoroBulletMove` by adding a
   narrowly scoped `preserveSourceEntry`/duplicate-preservation option if that keeps its
   line-shift and newline machinery authoritative; both options must default to the
   current ordinary-move behavior. Do not route through separate editor writes. Return
   `after`, remaining/split names, source and new-entry final lines, first moved line,
   selected/moved count, and source-preservation metadata; every invalid result returns
   the original text as `after`. Export the planner and split notice helper for tests.
3. Wire the split row through `PomodoroBulletMovePickerModal` and a guarded
   split-specific commit path (or an explicit branch in the current bullet commit
   method). Preserve close-before-commit and the shared reentrancy latch. Recompute the
   plan against the frozen session, execute one guarded transaction, focus the first
   moved bullet, and issue split-specific success/failure notices. Update the picker's
   discoverability text without changing global `FilteredPickerModal` semantics.
4. Add direct planner, row-model, and modal/commit regressions in
   `scripts/test-navigation-hotkeys.cjs`. Update the README navigation description and
   coverage paragraph, bump the navigation plugin's feature version in the manifest and
   README table, run all required checks, inspect the final diff, and deploy only
   `bob-navigation-hotkeys` from the opened checkout.

## Verification and acceptance

Use explicit before/after Markdown fixtures rather than expectations generated from the
planner under test.

- Test two-name and three-or-more-name splits, repeated right-to-left splits, varied
  whitespace around the canonical delimiter, repeated component names, and components
  containing adjacent pluses (`C++ + REVIEW` splits while `C++` and `A+B` refuse). Cover
  lowercase/manual names only to the extent allowed by existing normalization, exact
  48-character input, invalid/over-length source names, empty prefix/suffix, no
  delimiter, unnamed entries, and unsupported header tails. Invalid plans/rows retain
  the exact input.
- Exercise current timed, future placeholder, and closed/cancelled source headers;
  assert the source keeps its exact status/time body while every new entry is an open
  placeholder. Test first/middle/last ledger positions, neighboring entries/headings, LF
  and CRLF with and without terminal newlines, and no unrelated byte changes.
- Cover bare and counted selections, end clamping, one and multiple siblings, a
  non-trailing selected range, nested descendants/continuation lines, mixed indent
  styles, identical selected bullets, and selection from a nested depth. Assert exact
  subtree order and no duplicate collapse.
- Select all source bullets and assert the renamed source header survives without an
  invented child; select only some and assert its unselected content remains in place.
  Cover stale raw headers, stale target lines, missing source entries, and planner
  failures with no mutation.
- Drive the actual picker query/Enter path for `++`: assert exactly one split row,
  close-before-commit order, active-picker cleanup, one transaction/undo group, cursor
  placement, success notice, stale active file/editor/content refusal, and reentrant
  activation protection. Assert typing alone and Escape do not write.
- Keep explicit compatibility regressions showing `+` still creates a fresh same-name
  Pomodoro, `C++` remains a normal new name, ordinary typed/existing destinations and
  their duplicate policy are unchanged, entry-mode `++` is not a split row, and
  task/entry pickers do not gain the action.

From the opened plugin repository, run:

```sh
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

After those pass, deploy with bob-cli's documented interface, replacing
`<opened-plugin-repo>` with the exact path returned by `sase repo open`:

```sh
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo <opened-plugin-repo> --format json
```

Inspect both sync outputs. Confirm `manifest.json` and `main.js` are deployed, no dirty
vault files were skipped, and the list reports the new navigation version as synced and
enabled. Do not add `--force` if the dirty-file guard refuses; report that concrete
limitation instead.

If an interactive Obsidian session is available, reload the plugin and use a disposable
note to exercise bare and counted Ctrl+Shift+M followed by `++`, Enter, and one undo for
both a two-name current entry and a three-name future entry. Confirm only the selected
bullets move, the last name peels off, the time range stays on the source, and Escape is
inert. If interactive access is unavailable, report the automated coverage and
explicitly leave the smoke test unverified.

Completion requires the dedicated `++` split flow, atomic no-loss planner and guarded
commit, compatibility regressions, updated feature docs/version, passing validation, and
an accurately reported targeted deployment attempt.
