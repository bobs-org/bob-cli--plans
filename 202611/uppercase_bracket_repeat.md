---
tier: tale
title: Add ]} / [{ uppercase bracket repeat to Bob Navigation Hotkeys
goal:
  After any bracket walk, ]} / [{ repeat that family's uppercase jump (]S / [S review
  groups, ]W / [W lanes) the way ]] / [[ repeat its lowercase step.
size: small
decisions:
  link_family:
    ask:
      What should `]}` / `[{` do when the active bracket family is links (`n`), which
      has no `]N` / `[N`?
    choices:
      notice:
        Show a short notice and flash the footer; nothing moves, which matches the
        missing `]N` / `[N`
      lowercase:
        Open the next / previous link exactly like `]]` / `[[`, so the keys always do
        something
    default: notice
    why:
      There is no uppercase link walk to repeat, and a visible no-op never moves the
      cursor by surprise.
    answer: notice
proposed_by: bbugyi200.apollo.dm
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 10:36:55
status: wip
---

# Add `]}` / `[{` uppercase bracket repeat

## Goal

`]]` / `[[` repeat the last bracket-walk family (`s` review, `w` lanes, `n` links)
forward / backward using the lowercase key. Add `]}` / `[{`, which repeat the same
remembered family with its **uppercase** key instead:

| Active family | `]}` runs                                                                       | `[{` runs                                              |
| ------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `s`           | `]S`: first task of the next occupied review group                              | `[S`: first task of the previous occupied review group |
| `w`           | `]W`: first task of the next lane                                               | `[W`: first task of the previous lane                  |
| `n`           | see the `link_family` decision                                                  | see the `link_family` decision                         |
| none yet      | the existing empty notice (`Nothing to repeat yet — press ]s, ]w, or ]n first`) | same                                                   |

Everything else about the repeat stays as it is for `]]` / `[[`:

- The key sets the direction (`]}` forward, `[{` backward), whichever way the last walk
  moved.
- `]}` / `[{` never change the remembered family. They never call
  `rememberBracketRepeatType`, and they never write notes, stats-journal events,
  freshness stamps, or local storage.
- Counts carry over: `N]}` behaves like `N]S` / `N]W` (N occupied groups or N lanes).
  That already works if the delegated call happens **synchronously**, because
  `jumpToDueTask(…, { groupJump: true })` and `jumpToLaneTask(…, { laneJump: true })`
  consume the pending Vim count before their first await.
- The busy-walk swallow, wrap notices, and landing toasts come from the delegated
  methods unchanged. Like `]]`, a repeat shows no toast of its own.

## Where the code lives

Everything is in the **bob-plugins** linked repo, plugin
`plugins/bob-navigation-hotkeys`, plus one vault file. Open bob-plugins with
`/sase_repo` (`sase repo open bob-plugins -r "…"`), work only in the path it prints, and
read its `AGENTS.md` first. In short: edit `src/` fragments, run `npm run build`, never
hand-edit `main.js`, keep each hand-edited fragment at or under 1000 lines, and run
`bob plugins sync` at the end. No bob-cli source changes are needed. bob-cli's
`docs/reference/plugins/` is a gitignored mount of bob-plugins docs, so do not edit it.

Prior art to follow: commit `a2fd9f4` ("feat(navigation): add repeatable bracket walks")
and `4b62bc4` (footer indicator).

## Implementation

### 1. Repeat planner and runtime — `src/546-plugin-bracket-repeat.js`

- Add a frozen `BRACKET_REPEAT_UPPERCASE_PLANS` table next to `BRACKET_REPEAT_PLANS`:
  - `s`: forward `{ method: "jumpToDueTask", args: [1, { groupJump: true }] }`, backward
    `[-1, { groupJump: true }]`. These are the exact arguments of the
    `jump-to-last-due-task` / `jump-to-first-due-task` callbacks, which are `]S` / `[S`
    in the vimrc.
  - `w`: forward `{ method: "jumpToLaneTask", args: [1, { laneJump: true }] }`, backward
    `[-1, { laneJump: true }]`. These match `jump-to-next-lane` / `jump-to-prev-lane`,
    which are `]W` / `[W`.
  - `n`: `null` (no uppercase plan).
    > [!decision] link_family = lowercase Make `n` reuse the lowercase plans instead:
    > `openLabeledBodyLink("next")` / `openLabeledBodyLink("prev")`.
- Extend `planBracketRepeat(type, direction, options = {})`. With
  `options.uppercase === true` it reads the uppercase table and returns `null` when the
  family has no uppercase plan. Without the option, its behavior and return values must
  not change; existing tests pin them.
- Extend `repeatBracketWalk(direction, options = {})`:
  1. Read the type exactly as today. If no type is set, show
     `BRACKET_REPEAT_EMPTY_NOTICE` and flash `"empty"`, as today, for both key pairs.
  2. Ask the planner for a plan, passing `uppercase`. If the uppercase plan is `null`
     (only `n` under the default decision), show a new exported constant
     `BRACKET_REPEAT_NO_UPPERCASE_NOTICE`, flash `"empty"`, and return `false`.
     Suggested text: `No uppercase link walk — ]} / [{ repeat ]S / [S or ]W / [W`.
  3. Otherwise delegate **synchronously**: `this[plan.method](...args)`. Give each
     object argument a fresh, unfrozen shallow copy (`{ ...arg }`) so a callee can never
     hit a frozen shared options object. Flash `"forward"` / `"backward"` as today and
     return the delegate's result.
- Update the file's header comment to cover `]}` / `[{`.

### 2. Commands — `src/490-plugin-lifecycle.js`

Register two commands directly after `repeat-next-bracket-walk` /
`repeat-prev-bracket-walk`. They get no default `hotkeys`; the vimrc binds them.

- `repeat-next-uppercase-bracket-walk`, named
  `Repeat last bracket walk forward, uppercase (]S or ]W)`, with callback
  `() => this.repeatBracketWalk(1, { uppercase: true })`.
- `repeat-prev-uppercase-bracket-walk`, named
  `Repeat last bracket walk backward, uppercase ([S or [W)`, with callback
  `() => this.repeatBracketWalk(-1, { uppercase: true })`.

> [!decision] link_family = lowercase Name them `… uppercase (]S, ]W, or ]n)` /
> `… uppercase ([S, [W, or [n)`.

### 3. Footer indicator — `src/547-bracket-mode-indicator.js`

The `[[ s w n ]]` status-bar item is how Bryan sees what a repeat will do, so it should
teach the new keys without a layout change:

- Add an `upperTooltip` to each `BRACKET_MODE_FAMILIES` entry:
  - `s`: `]} / [{ jump to the next / previous review group (]S / [S)`
  - `w`: `]} / [{ jump to the next / previous lane (]W / [W)`
  - `n`: `]} / [{ have no link variant (no ]N / [N)` (`link_family = lowercase`:
    `]} / [{ also open the next / previous link (]n / [n)`)
- Build the active tooltip as
  `${family.tooltip}\n${family.upperTooltip}\n\n${BRACKET_MODE_ACTIVE_HINT}`. The first
  line stays the existing `]] / [[` line. The "none" tooltip does not change.
- Keep the glyphs, thumb, click behavior, and flash kinds as they are. `]}` / `[{` reuse
  the `"forward"` / `"backward"` flash, and the no-uppercase notice reuses `"empty"`.

### 4. Exports — `src/700-exports.js`

Export `BRACKET_REPEAT_NO_UPPERCASE_NOTICE` (and anything else new the tests need)
through `helpers`. **This fragment is already exactly 1000 lines.** Add the names to
existing lines, for example the
`BRACKET_REPEAT_STORAGE_KEY, BRACKET_REPEAT_EMPTY_NOTICE,` line, so the file does not
grow past the limit.

### 5. Manifest and generated docs

- `plugins/bob-navigation-hotkeys/manifest.json`: bump the minor version to the next
  minor above whatever is current when you start (it is `2.80.0` while this plan is
  written, so `2.81.0`). In the description, extend the existing `]]` / `[[` sentence
  ("… forward / backward, remembered per device.") with:
  `` `]}` / `[{` repeat that family's uppercase jump — `]S` / `[S` review groups or `]W` / `[W` lanes. ``
- Run `npm run build`, then `npm run docs:gen`. This regenerates the facts row, the
  commands table in `docs/bob-navigation-hotkeys.md`, and the version in
  `docs/index.md`.

### 6. Hand-written docs — `docs/bob-navigation-hotkeys.md`

- In **Bracket repeat**, add a short "Uppercase repeat" paragraph and the family → `]}`
  / `[{` table from the Goal section, with the `n` row matching the chosen decision.
  Cover these points: the key owns direction; the keys never change the selected family;
  `N]}` / `N[{` keep count parity with `N]S` / `N[S` and `N]W` / `N[W`; and the empty
  and no-uppercase notices.
- Fix the sentence "Uppercase `[S` / `]S` cycle occupied review groups but still select
  family `s`, so `]]` / `[[` perform the ordinary lowercase review walk". It should now
  also say that `]}` / `[{` perform the group jump.
- In "Only the family command callbacks … record a type", add `]}` / `[{` next to `]]` /
  `[[` as keys that do not record.
- **Footer indicator**: mention the second tooltip line.
- **Notes, tabs, and keyboard layers**: the vimrc bullet should say it maps `[{` / `]}`
  to the two uppercase repeat commands, alongside `[[` / `]]`.

### 7. Tests

`scripts/test-navigation-bracket-repeat.cjs` (already in `npm run test:unit`):

- Pure planner: uppercase plans for `s` and `w` in both directions have the exact
  methods and args above. `n` uppercase returns `null` (`link_family = lowercase`:
  returns the lowercase link plans). Lowercase plans are unchanged.
- Registration: both new commands exist with the exact names and no `hotkeys`. The
  existing repeat commands are untouched.
- Delegation: for each family and direction,
  `repeatBracketWalk(dir, { uppercase: true })` calls the right method with the right
  args and returns its result. The options object it receives is not frozen, and each
  call gets a fresh one.
- Sequence: `jump-to-prev-due-task` then `repeat-next-uppercase-bracket-walk` calls
  `jumpToDueTask(1, { groupJump: true })`. The type stays `s`, and local storage sees no
  new write. Do the same for `w` with `jumpToLaneTask(-1, { laneJump: true })` via `[{`.
- Cold start and invalid stored values: both uppercase commands show
  `BRACKET_REPEAT_EMPTY_NOTICE` and move nothing. With type `n` (default decision), both
  show `BRACKET_REPEAT_NO_UPPERCASE_NOTICE`, return `false`, and call no walk method.
- Count parity: mirror the existing "repeat delegates synchronously so the real lane
  walk consumes Vim count before landing" test with `{ uppercase: true }` and a two-lane
  queue. Assert that `consumePendingLaneWalkRepeat` runs before the first await and that
  the landing is the lane jump's target.

`scripts/test-navigation-bracket-mode-indicator.cjs`:

- Update the family tooltip expectations (around the `]] / [[ repeat the review walk`
  rows) so each active tooltip's second line is that family's `upperTooltip`.
- Add a case where `repeatBracketWalk(±1, { uppercase: true })` flashes `forward` /
  `backward`, and the no-uppercase path flashes `empty`.

### 8. Vault vimrc — `~/bob/obsidian_vimrc.md`

This is vault configuration, not plugin code. `a2fd9f4` was paired with the same kind of
direct vault edit. Directly after the `]]` / `[[` block, add:

```vim
" ]} / [{ repeat the last bracket walk's uppercase jump (]S / [S review groups, ]W / [W lanes) forward / backward.
exmap bob_repeat_next_bracket_upper obcommand bob-navigation-hotkeys:repeat-next-uppercase-bracket-walk
exmap bob_repeat_prev_bracket_upper obcommand bob-navigation-hotkeys:repeat-prev-uppercase-bracket-walk
nnoremap ]} :bob_repeat_next_bracket_upper<CR>
nnoremap [{ :bob_repeat_prev_bracket_upper<CR>
```

Use `nnoremap`, as the file header requires. User mappings take precedence over
CodeMirror Vim's built-in `]<character>` / `[<character>` symbol motions, which is the
same reason `]]` / `[[` already work. Do not commit the vault by hand: the
`bob-vault-sync` service commits and pushes it. Make this edit only after the plugin
build passes, so the mapping never points at a missing command.

## Validation

In the bob-plugins checkout:

1. `node --test scripts/test-navigation-bracket-repeat.cjs scripts/test-navigation-bracket-mode-indicator.cjs`
2. `npm run build:check`, `npm run validate`, and `npm test`. `npm test` is long, so
   hand it to `/sase_monitor` if it might outlive a foreground timeout.
3. `npm run docs:check`
4. `bob plugins sync`. Follow the AGENTS.md refusal procedure if it refuses, and never
   pass `-F` on your own.
5. Optional visual check: `node scripts/preview-bracket-mode.cjs` renders the indicator
   headlessly. Confirm the tooltip text if the preview shows it.

Then commit the bob-plugins changes through the turn's final declaration. In the final
report, tell Bryan to reload Obsidian (or the vimrc plugin) so obsidian-vimrc-support
picks up the new `]}` / `[{` mappings. Give him a quick manual check: `]s`, then `]}`
lands on the next review group's first task; `]w`, then `[{` lands on the previous
lane's first task.

## Out of scope

- No new footer glyphs. The status bar keeps `[[ s w n ]]`.
- No `]N` / `[N` link walk.
- No change to `]]` / `[[`, `]S` / `[S`, `]W` / `[W`, or how families are recorded.
- No bob-cli code, docs, or memory changes.
