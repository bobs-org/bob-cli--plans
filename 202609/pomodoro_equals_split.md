---
tier: tale
title: Use `=` for same-name Pomodoro moves and `=NAME` to split a merged name
goal:
  In the Ctrl+Shift+M Pomodoro sub-bullet picker, replace the `+` same-name shortcut
  with `=` and add `=NAME` so selected bullets move into a new Pomodoro named NAME while
  NAME is removed from the source's merged ` + `-separated name.
size: medium
proposed_by: bbugyi200.apollo.0p
create_time: 2026-09-19 09:13:40
status: wip
---

# Use `=` for same-name moves and `=NAME` to split merged Pomodoros

## Objective and scope

Change the Pomodoro **sub-bullet** destination picker opened by bare or counted
`Ctrl+Shift+M` so that:

1. Exact trimmed `=` (replacing exact trimmed `+`) creates a fresh destination with the
   source Pomodoro's normalized name and moves the selected bullets into it. Source-name
   copying, counted moves, source cleanup, and duplicate-name handling stay the current
   `+` contract.
2. A query of the form `=NAME` splits the selected bullets into a **new** Pomodoro named
   `NAME` and removes every matching `NAME` component, plus the associated canonical `+`
   delimiter, from the source Pomodoro's name.

This is a `tale`, size `medium`: one implementation agent can retarget the existing
same-name shortcut, generalize the current last-component split planner to a named
component peel (including an unnamed remainder), update picker copy and regressions, and
deploy `bob-navigation-hotkeys`. The work belongs entirely in `bobs-org/bob-plugins`. No
bob-cli capture/timer contract, other plugin, SASE memory note, or live daily note needs
to change.

Preserve all existing `Ctrl+Shift+M` routing except the special queries in sub-bullet
mode. Keep ordinary task moves, Pomodoro entry move/rename, Ctrl+X entry merge, existing
destination bullet moves, typed new-name creation, count/clamping, duplicate suppression
for ordinary moves, and names that contain non-delimiter plus characters such as `C++`.
Entry-mode `=`, `=NAME`, `+`, and `++` remain ordinary rename/filter queries.

## Why this replaces `+` and `++`

Today, in `createPomodoroBulletMovePickerRows` bullet mode:

- Exact trimmed `+` returns one `kind: "new"` row whose name is the source's normalized
  name, even when another open entry already has that name.
- Exact trimmed `++` returns one `kind: "split"` row that peels only the **last**
  canonical `+` component via `decomposePomodoroMergedName` / `planPomodoroBulletSplit`.
- `commitPomodoroBulletSplitSession` recomputes that last-component peel and does not
  take a requested name from the row.

`+` is being vacated so merged names can keep using `+` as a visible name character
without colliding with the same-name shortcut. `=` is the new reserved prefix. The
requested `=NAME` syntax is a strict generalization of `++`: the user names the
component to peel instead of always taking the rightmost one. After this change, exact
`+` and `++` are no longer reserved in bullet mode; they fall through to ordinary name
handling. Do **not** add an `==` last-component alias. Peeling the last component is
`=EMAIL` (or whichever last name the source actually has).

Predecessor contracts, for implementer orientation:

- `plan:202609/pomodoro_same_name_shortcut.md` — current `+` same-name copy
- `plan:202609/pomodoro_bullet_split.md` — current `++` last-component split
- `plan:202609/pomodoro_entry_merge.md` — Ctrl+X joins names with canonical `+`

Read the Pomodoro glossary through `/sase_memory_read` before relying on ledger grammar.

## Repository access and implementation context

Before reading or editing plugin files, use `/sase_repo`. Try the linked short name
first, then the GitHub fallback if the linked checkout is missing (that fallback was
necessary during planning):

```sh
sase repo open bob-plugins -r "Retarget Pomodoro same-name moves to = and add =NAME split"
# if that fails:
sase repo open gh:bobs-org/bob-plugins -r "Retarget Pomodoro same-name moves to = and add =NAME split"
```

Use only the returned checkout path and read that repository's `AGENTS.md`. Plugins are
plain CommonJS; `main.js` is source, with no bundle/build step. Edit the source repo and
deploy with `bob plugins sync`. Do not edit deployed files under the vault.

Relevant files and symbols, all relative to the opened plugin repository:

- `plugins/bob-navigation-hotkeys/main.js`: `normalizePomodoroName`,
  `decomposePomodoroMergedName`, `formatPomodoroEntryLine`, `parsePomodoroEntryLine`,
  `planPomodoroEntryRename`, `planPomodoroBulletMove` (`preserveSourceEntry`,
  `preserveDuplicates`), `planPomodoroBulletSplit`,
  `createPomodoroBulletMovePickerRows`, `PomodoroBulletMovePickerModal` (placeholder and
  subtitle currently advertise `+` / `++`), `commitPomodoroBulletMoveSession`,
  `commitPomodoroBulletSplitSession`, `buildPomodoroBulletSplitNotice`, and the exported
  `helpers` test surface.
- `scripts/test-navigation-hotkeys.cjs`: same-name `+` picker/session tests, `++` split
  row/planner/commit tests, and `decomposePomodoroMergedName` cases.
- `plugins/bob-navigation-hotkeys/manifest.json` and `README.md`: plugin version and the
  long navigation-plugin description. Planning saw version `1.36.0`; choose the next
  feature version from the implementation checkout so intervening changes are not
  overwritten.

The plugin-side name normalizer strips em dashes, collapses whitespace, uppercases, and
rejects empty or over-length results. It does **not** reject `+` or `=` as characters.
Special query handling must therefore stay in the bullet-mode row builder, before
ordinary `normalizePomodoroName(queryText)` creation. Merge/split delimiters are the
canonical whitespace-delimited `+` sequence produced by Ctrl+X (`join(" + ")` after
normalization). Do not split on adjacent pluses inside a component: `C++` and `A+B` are
single components.

## Behavior contract

Parse the trimmed query in bullet mode only.

| Trimmed query                                      | Action                                        |
| -------------------------------------------------- | --------------------------------------------- |
| `=`                                                | Same-name copy (current `+` contract)         |
| `=NAME` (any extra text after `=`)                 | Named split/peel of `NAME`                    |
| anything else, including `+`, `++`, `C++`, `FOCUS` | Existing ordinary name/filter/create behavior |

Surrounding whitespace on the whole query is trimmed first, matching today's `+` / `++`
handling. After a leading `=`, the remainder is the requested name and is passed through
`normalizePomodoroName`, so `= review`, `=REVIEW`, and `=review` all request `REVIEW`.
Exact `=` with only surrounding whitespace is same-name copy, not an empty peel.

### 1. Exact `=` — same-name copy

This is a mechanical retarget of the current `+` branch in
`createPomodoroBulletMovePickerRows` and its tests/help text.

- Resolve the source by entry line from the complete input entries.
- A named, valid source yields one `kind: "new"` row displaying the resolved name, even
  when the source or another open entry already has that name. Confirmation goes through
  the existing creation transaction (`commitPomodoroBulletMoveSession` / ordinary
  `planPomodoroBulletMove`). Copy the name only: destination is a fresh open
  `- [ ] () — NAME` placeholder. Do not copy status or time range. Do not rename or peel
  the source.
- Partial moves leave the source; moving its last owned content deletes it. A new
  same-name destination must never merge into another same-name entry.
- Unnamed, missing, or invalid-name sources yield one non-actionable `kind: "invalid"`
  row. For an unnamed source, explain that `=` needs a named source and that the user
  can type a new name. Do not invent a name from a time range, ordinal, preview, or the
  `=` token.
- Named closed/cancelled sources that are already eligible for sub-bullet moves keep
  supporting the shortcut.

Example, cursor on `move`, query `=`:

```markdown
## Pomodoros

- [ ] () — FOCUS
  - keep
  - move
- [ ] () — FOCUS
  - existing
```

Result:

```markdown
## Pomodoros

- [ ] () — FOCUS
  - keep
- [ ] () — FOCUS
  - move
- [ ] () — FOCUS
  - existing
```

### 2. `=NAME` — peel NAME and move selected bullets into a new NAME

`NAME` is the normalized remainder after the leading `=`. Treat it as **one** Pomodoro
name (it may contain `C++`-style adjacent pluses). Do not parse a list of names.

Decompose the source name **after** `normalizePomodoroName`, then split on the canonical
`" + "` delimiter (the same delimiter Ctrl+X writes). Each part is one merge component.

Eligibility:

- The source must be a parsable named Pomodoro in `## Pomodoros`, with a captured header
  and selected targets that still match. Closed/cancelled merged sources remain
  eligible, as with today's sub-bullet picker and `++` split.
- `NAME` must be present as at least one merge component **or** must equal the entire
  normalized source name. Otherwise return one invalid row explaining that `NAME` is not
  part of this Pomodoro's name, and do not fall through to creating a literal `=NAME`
  Pomodoro or an ordinary `NAME` destination.
- Unnamed, missing, or invalid source names, invalid/empty `NAME`, over-length names,
  and unsupported trailing header content are invalid rows / failed plans. Write
  nothing.

Peel rule ("remove any `FOOBAR` and the associated `+`"):

- Remove **every** component whose normalized form equals `NAME`, then rejoin the
  leftover components with `" + "` in original order.
- If `NAME` equals the entire source name (`FOOBAR`, or typing the full merged string
  such as `=BUILD + REVIEW` when the source is exactly `BUILD + REVIEW`), the remainder
  is empty: strip the ` — NAME` suffix and leave the source unnamed. Do not pass an
  empty string through `planPomodoroEntryRename` / `normalizePomodoroName` (those reject
  empty names). Rewrite the header to the bytes through the parenthetical
  (`rawLine.slice(0, parsed.rangeEnd)`), matching unnamed ledger entries.
- Removing a middle component must not leave a dangling or doubled delimiter:
  `BUILD + REVIEW + EMAIL` with `=review` becomes `BUILD + EMAIL`.
- `BUILD + REVIEW + BUILD` with `=build` becomes `REVIEW` (all matches removed).
- `C++ + REVIEW` with `=c++` becomes `REVIEW`; `C++` or `A+B` with `=review` is invalid
  because `REVIEW` is not a component.
- Repeated `=NAME` uses on a multi-component source peel one requested name per
  confirmation; they are user-directed, not automatic.

Content, placement, and atomicity follow the current split planner, not ordinary
new-name moves:

- Move exactly the selected top-level sibling bullets at the invocation depth, each with
  its descendant/continuation subtree, using existing count and end-of-Pomodoro
  clamping.
- Preserve duplicates; do not collapse identical selected bullets.
- The source **always survives**, even when the selection consumes every owned
  substantive bullet. Do not delete its header or transfer its time range. Keep its
  checkbox status, exact placeholder/time body, emphasis, and `[t:: ...]` metadata. Do
  not invent an empty child placeholder when none remains.
- The new entry is always an open placeholder `- [ ] () — NAME` inserted immediately
  after the surviving source block, before the next Pomodoro.
- Derive rename-or-unname, insertion, and movement in one pure plan and apply
  `before`/`after` once with `applyEditorContentTransaction` (one undo group).
- Revalidate active file path, editor identity, full captured content, raw source
  header, and target lines before writing. Close the picker before committing through
  the existing `opening` latch. Place the cursor on the first moved bullet in the new
  Pomodoro. Failed or stale commits write nothing and must not emit a success notice.

`commitPomodoroBulletSplitSession` must pass the requested `NAME` into the pure planner
and **recompute** the peel against the frozen source. Do not trust `remainingName` from
the picker row, and do not keep the current last-component-only call.

Example: cursor on `email one` with count 1 so both EMAIL bullets are selected, query
`=email`:

```markdown
- [ ] (**0920-0950** [t:: 30m]) — BUILD + REVIEW + EMAIL
  - build work
  - review work
  - email one
    - follow-up
  - email two
```

Result:

```markdown
- [ ] (**0920-0950** [t:: 30m]) — BUILD + REVIEW
  - build work
  - review work
- [ ] () — EMAIL
  - email one
    - follow-up
  - email two
```

Peeling a non-last or non-trailing name is required. Selecting the review bullet and
confirming `=review` on the same source must yield `BUILD + EMAIL` on the timed source
and a new `REVIEW` placeholder under the selected subtree, even though `REVIEW` is the
middle component and the selection is not a trailing range. The name carries component
identity, not bullet ownership.

Example: source is a single name, query `=focus`:

```markdown
- [ ] (**0920-0950** [t:: 30m]) — FOCUS
  - keep
  - move
```

Result after selecting `move`:

```markdown
- [ ] (**0920-0950** [t:: 30m])
  - keep
- [ ] () — FOCUS
  - move
```

That is intentionally different from exact `=`, which would keep the source named
`FOCUS` and create a second `FOCUS`.

### Picker presentation

- Exact `=` keeps a single `kind: "new"` row titled like `New Pomodoro FOCUS`.
- `=NAME` keeps a single `kind: "split"` row such as `Split off EMAIL`, with metadata
  that the selected bullets create a new Pomodoro below and the source becomes the
  remaining name, or `unnamed` when the remainder is empty.
- Invalid sources/names return one invalid row instead of destination matches or
  new-name rows.
- Update the bullet picker placeholder and subtitle: `=` means same-name copy and
  `=NAME` means split/peel that name. Enter remains the confirmation key. Typing by
  itself must not mutate the note.
- Success notices:
  - Same-name `=`: keep `Moved N bullet(s) to new Pomodoro NAME`.
  - `=NAME`: keep the split notice shape, e.g.
    `Split 2 bullets into new Pomodoro EMAIL; source is now BUILD + REVIEW`, using
    `unnamed Pomodoro` when the remainder is empty.
- Failure notices must say that nothing was moved or nothing was split, matching the
  path that ran. Other pickers must not acquire these aliases.

## Implementation

1. Replace last-only `decomposePomodoroMergedName` with a named peel helper (keep the
   old name only as a thin wrapper if that reduces churn; prefer one exported helper
   tests can call directly). It must normalize source and requested names, split on
   canonical `" + "`, remove every matching component, support whole-name
   remainder-empty, and return frozen
   `{ valid, error, sourceName, remainingName, splitName }` where `remainingName` may be
   `""`. Export it through `helpers`.
2. Teach `planPomodoroBulletSplit` to take the requested component (for example
   `options.splitName`). Compose (a) suffix strip when remaining is empty or the
   existing rename when remaining is nonempty, then (b) `planPomodoroBulletMove` with
   `preserveDuplicates: true` and `preserveSourceEntry: true`. Invalid results return
   the original text as `after`. Do not route through separate editor writes.
3. In `createPomodoroBulletMovePickerRows` bullet mode, handle exact trimmed `=` before
   ordinary query normalization (the old `+` branch, with copy updated to `=`). Handle
   trimmed queries that start with `=` and have a remainder as the split/invalid row
   using the new peel helper. Delete the exact `++` reservation. Leave entry mode
   unchanged.
4. Pass `row.name` into `commitPomodoroBulletSplitSession` / the split planner as the
   requested component and recompute. Update picker placeholder, subtitle, and notices.
   Keep close-before-commit and the shared reentrancy latch.
5. Rewrite the `+` / `++` regressions in `scripts/test-navigation-hotkeys.cjs` onto `=`
   / `=NAME`, add the new peel cases below, update the README navigation description and
   coverage paragraph, and bump the navigation plugin feature version in the manifest
   and README table from the checkout's actual version.

## Verification and acceptance

Use explicit before/after Markdown fixtures, not expectations generated from the planner
under test.

Same-name `=`:

- `=` and whitespace-padded `=` select a fresh source-name destination, including when
  another open same-name entry exists. A preview containing `=` or `+` must not compete
  with the creation row.
- Named closed sources work; unnamed, missing, and invalid-name sources are
  non-actionable and do not write.
- Partial and counted moves preserve order and nested descendants, leave other same-name
  entries untouched, delete a fully emptied source, and preserve CRLF in a
  representative fixture.
- A modal/session test obtains the `=` row from the actual picker, confirms it, and
  asserts the expected document, one transaction/undo group, first-moved-bullet cursor,
  and resolved-name notice. Cover cancellation and stale-content rejection.

Named `=NAME` split:

- Two-name and three-or-more-name peels, first/middle/last component, repeated
  confirmation, extra whitespace around the delimiter, repeated component names, and
  `C++ + REVIEW` (peels `REVIEW` or `C++`; refuses `=email`).
- Whole-name peel that unnames the source; `NAME + NAME` with `=name` leaves the source
  unnamed; missing-component, unnamed source, `C++`/`A+B` with a non-member name, empty
  remainder after `=`, invalid/over-length names, and unsupported tails do not mutate.
- Current timed, future placeholder, and closed/cancelled sources keep their exact
  status/time body; every new entry is an open placeholder.
- Bare and counted selections, end clamping, non-trailing ranges, nested
  descendants/continuation lines, mixed indent, identical selected bullets, all-bullets
  selection with a surviving renamed or unnamed header, LF/CRLF with and without a
  terminal newline, first/middle/last ledger position, stale header/targets, and one
  guarded transaction/undo group with cursor and split notice.
- Drive the real picker with `=email` and with `= review`. Typing alone and Escape must
  not write.

Compatibility:

- `+` and `++` are no longer special in bullet mode (they follow ordinary name rules; do
  not recreate a last-component split on `++`).
- `C++` remains an ordinary typed name.
- Ordinary typed/existing destinations and their duplicate policy are unchanged.
- Entry-mode `=`, `=NAME`, `+`, and `++` are not same-name or split rows.
- Task and entry pickers do not gain these aliases. Ctrl+X merge still writes `+`.

From the opened plugin repository:

```sh
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

After those pass, deploy only this plugin from the path returned by `sase repo open`:

```sh
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo <opened-plugin-repo> --format json
```

Inspect both sync outputs. Confirm `manifest.json` and `main.js` deployed, no dirty
vault files were skipped, and the list reports the new navigation version as synced. Do
not add `--force` if the dirty-file guard refuses; report that limitation instead.

If an interactive Obsidian session is available, reload the plugin and use a disposable
note to exercise `Ctrl+Shift+M`, `=`, Enter (same-name copy) and `=NAME`, Enter (middle
and last component peels, plus a whole-name unname), including one counted move and one
undo each. Confirm `+` no longer same-name-copies, Escape is inert, and time metadata
stays on the source for splits. If interactive access is unavailable, report the
automated coverage and leave the smoke test unverified.

## Out of scope

- No bob-cli grammar, `capture-pomodoro-name`, or selector-character changes.
- No Ctrl+X merge changes and no entry-picker `=` shortcut.
- No SASE memory edits: `glossary:pomodoro` describes `N<ctrl+shift+m>` generically and
  does not mention `+` / `++`.
- No automatic last-component alias (`++` or `==`).
- Do not treat `=NAME` as a force-create of an unrelated name when `NAME` is absent from
  the source; ordinary typed names remain the way to create or select an unrelated
  destination.
