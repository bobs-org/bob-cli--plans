---
tier: tale
title: "Copy card: the C row reaches the cursor's task"
goal:
  Mod+Y then C copies the only copy chip on the cursor line or, failing that, the only
  chip in the cursor's task, and says which.
size: medium
decisions:
  gy_follows_task:
    ask:
      Should `gy` also fall back to the only chip in the cursor's task, like the Copy
      card's `C` row?
    default: true
    why: The copy-chips epic promised one cursor rule, so gy and Mod+Y C never disagree.
    answer: true
  memory_glossary_copy_chip:
    ask:
      Rewrite the glossary:copy-chip cursor-rule sentence to include the task fallback?
    memory:
      - glossary:copy-chip
    default: true
    corrects:
      "The cursor rule, shared by `gy` and the Copy card: a chip under the cursor wins;
      otherwise the only chip on the line; otherwise the `bob-copy` fence the cursor is
      inside; two or more chips on the line with the cursor on none is ambiguous and
      never guessed."
    answer: true
proposed_by: bbugyi200.apollo.f4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 16:26:55
status: wip
---

# Plan: Copy card `C` row reaches the cursor's task

## Context

Bryan wants the `Cmd+Y` Copy card to offer a copy chip whenever there is exactly one
chip on the cursor line **or in the cursor's task**.

Most of that already exists. Epic `bob-cli-ni` (`plan:202611/copy_chips.md`) shipped
copy chips. Its landing tale (`plan:202611/copy_card_chip_row_land.md`, bob-plugins
commit `d6d2c33`, `bob-navigation-hotkeys` 2.90.0, deployed) gave the card a `C` row
under a `CHIP` divider. But that row only follows the **cursor rule**:

1. A chip under the cursor wins.
2. Otherwise, the only chip on the cursor line.
3. Otherwise, the `bob-copy` fence the cursor is on or inside.
4. Two or more chips on the line with the cursor on none is ambiguous.

A chip one line away is invisible. Take this note:

```markdown
- [ ] #task **Deploy the docs site** ^deploy-docs
  - run `copy[deploy]: just docs-check && just docs-deploy` from the bob-cli root
  - Depends-On: [[bob-cli#^docs-check-green]]
```

With the cursor on the task line, which is where `]s` lands, `Mod+Y` shows no `C` row.

**The design decision this plan makes:** do not add a second chip row or a second key.
Keep the single `C` row; `c` is already the reserved key for chips. Teach it one more
step: when the cursor line has no chip, use the only chip in the cursor's task. Two rows
offering the same chip would be noise, and two keys for "copy the chip" would make it
unclear which key to press.

The one key, `C`, keeps the same meaning: copy the chip I mean. What changes is how far
the card looks to find it.

## The product

### What the `C` row shows

| Cursor                                                | The cursor's task has | `C` row                                                               |
| ----------------------------------------------------- | --------------------- | --------------------------------------------------------------------- |
| On a chip                                             | anything              | that chip                                                             |
| On a line with exactly one chip                       | anything              | that chip                                                             |
| On a line with 2+ chips, on none of them              | anything              | dimmed: "N chips on this line · move onto one"                        |
| On or inside a `bob-copy` fence                       | anything              | that fence                                                            |
| Inside any other fence                                | —                     | no row (code samples stay inert; the card already sees no task there) |
| On a chipless line of a task (task line or its notes) | exactly 1 chip        | **that chip**; the CHIP divider reads **CHIP in this task**           |
| On a chipless line of a task                          | 2+ chips              | **dimmed: "N chips in this task · move onto one"**                    |
| On a chipless line of a task                          | 0 chips               | no row                                                                |
| On a chipless line outside any task                   | —                     | no row                                                                |

Rows 6 and 7 are new; every other row is today's behavior, unchanged. The rule still
**never guesses**: with two chips, `C` nudges, and the dimmed row says how to fix it.

**"The cursor's task"** is the task that the card's TASK divider already names, the one
`resolveCopyCardTask` resolves:

- the task line, its child notes, and its Depends-On, BLOCKS, LINKS, and LINKED FROM
  lines;
- **not** a nested `#task` child. A subtask is its own task, with its own chips: it gets
  its own TASK divider when the cursor is inside it.

Fences inside the task follow the same rules as everywhere else:

- a `bob-copy` fence counts as one chip, its whole payload;
- any other fence contributes nothing, so code samples with `copy:` spans never count;
- frontmatter never counts.

A Task Link to a task in another note never lends that task's chip. This matches the
card's same-file invariant for Block IDs, and it keeps the card reading only the live
buffer.

### How it looks

The row keeps the shape of today's ready row: key, `Chip · <face>`, payload preview. The
only new element is a short muted title on the CHIP divider, and only when the chip came
from the task. It reuses the TASK divider's label-plus-title look, so the divider says
where the chip was found. Directly under the TASK divider, it reads as one phrase: "TASK
Deploy the docs site … CHIP in this task".

```text
╭────────────────────────────────────────────────────────────────╮
│ ⧉  Copy                                                        │
│    projects/docs.md                                            │
│ P  Path            ~/bob/projects/docs.md                   ↵  │
│ A  Absolute path   /home/bryan/bob/projects/docs.md            │
│ F  File name       docs.md                                     │
│ N  Note name       docs                                        │
│ ──── TASK  Deploy the docs site ────────────────────────────── │
│ I  Block ID        deploy-docs                                 │
│ T  Task ref        the ^deploy-docs task in the ~/bob/proj…    │
│ ──── CHIP  in this task ─────────────────────────────────────  │
│ C  Chip · deploy   just docs-check && just docs-deploy         │
│ ↵ copy path   ↑↓ select   esc q close                          │
╰────────────────────────────────────────────────────────────────╯
```

The ambiguous form has no title, because the reason text already names the scope:

```text
│ ──── CHIP ───────────────────────────────────────────────────  │
│ C  Chip            2 chips in this task · move onto one        │   (dimmed)
```

These details stay as they are today:

- `P` stays the `↵` default.
- The copy goes through the card's clipboard path, then shows the `Copied …` Notice.
- `api.copyChips.flash` pulses the chip where it lives in the task, or the revealed
  source range. A chip scrolled out of view simply does not flash: the Notice still
  confirms, and the flash never turns a successful copy into a failure.

For screen readers, the chip row's `aria-label` adds "in this task" when the title is
shown.

### `gy`

> [!decision] gy_follows_task `gy` and the palette command (id `copy-on-cursor-line`,
> unchanged, so the vimrc mapping keeps working) use the same task-aware rule as `C`.
>
> - **Palette name.** It becomes "Copy chip at cursor", because "on cursor line" would
>   now be false.
> - **Notices.**
>   - "No copy chip on this line" outside a task;
>   - "No copy chip in this task" inside one;
>   - "N chips on this line · move onto one" for an ambiguous line;
>   - "N chips in this task · move onto one" for an ambiguous task.
> - **Without bob-navigation-hotkeys**, `gy` stays line-only, exactly as today.

> [!decision] gy_follows_task = no `gy` is untouched. The task fallback is the Copy
> card's alone. Docs and the glossary say so: `gy` never copies a chip that `C` would
> not copy, but `C` can also reach the task.

`Mod+Shift+Y` (make or edit) **never** uses the task step, in either branch. The card
edits only the chip you are on or the only one on your line. Opening it on a chip three
lines away would surprise, and from a chipless line it should insert at the cursor as it
does today. Clicks are unaffected.

## Where the work lands

- **bob-plugins.** Open it with `sase repo open bob-plugins -r "<why>"` and work only in
  the printed path. Read its `AGENTS.md` first.
  - Edit `src/` fragments only, then run `npm run build`; never hand-edit a generated
    `main.js`.
  - Keep each hand-edited fragment at or under 1000 lines.
    `plugins/bob-navigation-hotkeys/src/101-copy-card-model.js` is at about 975 lines,
    so put new Copy card logic in `src/101b-copy-card-chips.js`, which is small, or in a
    new `101c-…` fragment registered in `src/fragments.json`.
- **bob-cli** (this workspace), memory decision only:
  `sase/memory/glossary/copy-chip.md` and the generated `docs/reference/glossary.md`.
- **The vault `~/bob`**, gy branch only: one comment line in `obsidian_vimrc.md`.

Out of scope:

- no bob-cli code change, so no Cargo version or CHANGELOG edits
  (`decisions:bob-cli-releases-ride-a-daily-train`);
- no Bob Mac Capture change;
- no grammar change;
- no new key.

## Steps

### 1. bob-copy-chips: an opt-in task scope for `chipAt` (API v2)

Work in `plugins/bob-copy-chips/src/065-cursor.js` (pure; about 340 lines) and
`090-plugin.js`.

- **Request.** `chipAt({ lines, line, ch, scope })` accepts an optional
  `scope: { kind: "task", lines: number[] }`, the zero-based line indices the caller's
  task owns.
  - Drop non-integer, negative, and out-of-range indices, and collapse duplicates. Scan
    in document order.
  - Treat the scope as absent unless `kind === "task"`, `lines` is an array, and the
    cleaned set contains the cursor line. A scope that does not hold the cursor is
    nonsense, so ignore it.
  - Without a scope, behavior is byte-for-byte today's. This keeps `081-author-model.js`
    (Mod+Shift+Y) and every existing caller unchanged.
- **Rule.** Run today's steps first, unchanged:
  - frontmatter → none;
  - fence block → the `bob-copy` block chip, or none;
  - a chip under the cursor;
  - the only chip on the line;
  - 2+ chips on the line → ambiguous.

  Only when the cursor line resolves to none **because it holds zero chips**, and is not
  fenced or frontmatter, scan the scope:
  - every inline chip from `copyChipScanLine` on each unfenced, non-frontmatter scope
    line, with the per-line table detection the cursor line uses;
  - each `bob-copy` fence touched by a scope line, counted **once** (keyed by its
    opening line), as a block chip built like `copyChipFenceBlockChip`, but with `line`
    set to the fence's **opening** line and `[from, to]` covering that line;
  - no other fenced line contributes anything.

  The result:
  - exactly one chip → `{ state: "ready", count: 1, scope: "task", chip }`, where the
    chip carries its own `line` / `from` / `to`, so `flash` finds it;
  - two or more → `{ state: "ambiguous", count: N, scope: "task" }`;
  - zero → `{ state: "none", count: 0 }`.

- **The `scope` field.** Ready and ambiguous results from the cursor steps gain
  `scope: "cursor"`. `none` keeps its exact shape. All results stay frozen.
- **Cost.** Work out fence membership for the scope with **one forward pass** up to the
  last scope line. Never call `copyChipFindFenceBlock` per scope line: it rescans from
  line 0 each time, which is quadratic on a big task.
- **Version.** Set `version: 2` in `copyChipsBuildApi` and in the fallback stub in
  `onload`. Everything else on the namespace is unchanged and still never throws.

> [!decision] gy_follows_task **The cursor command.** In `070-commands.js`,
> `copyChipRunCursorCommand` gets the plugin's `app`; pass it from
> `copyChipRegisterCursorCommand`.
>
> - **Getting the task scope.** Look up
>   `app.plugins.plugins["bob-navigation-hotkeys"]?.api?.taskScope`. Accept it when
>   `version >= 1` and `at` is a function. Call `at({ lines, line: cursor.line })`
>   inside `try/catch`. A missing plugin, a throw, or a `null` result means no scope,
>   which is today's behavior.
> - **Resolving.** Pass `{ kind: "task", lines: result.lines }` to `copyChipAt`.
> - **Notices.** Show the four notices listed under [`gy`](#gy), picking "on this line"
>   or "in this task" from the result's `scope`. For `none`, pick by whether a scope was
>   passed.
> - **Name.** Rename the command `name` to "Copy chip at cursor". Keep the id
>   `copy-on-cursor-line`.

**Manifest.** Bump `plugins/bob-copy-chips/manifest.json` one minor version from
whatever it holds when you start (now `0.4.0` → `0.5.0`).

### 2. bob-navigation-hotkeys: the card resolves the task's lines and labels the source

- **Scope helper** (in `101b-copy-card-chips.js` or a new `101c-…` fragment):
  `copyCardTaskScopeLines(lines, ownerLine)` → frozen ascending line indices.
  - It returns `ownerLine`, plus every line of
    `findCurrentBulletChildBlock(lines, ownerLine)` whose owning task is `ownerLine`.
    That excludes nested `#task` subtrees.
  - It must agree exactly with `findOwningTaskLine(lines, i) === ownerLine` for every
    line in the block.
  - Do not call `findOwningTaskLine` per line: it rebuilds
    `getMarkdownLineContextsForLines` over the whole note each time. Compute the
    contexts once and walk owners with a memo (`isObsidianTaskAtLine` with the shared
    contexts, `findNearestParentListItem`).
  - Fenced lines inside the block stay in the set; `chipAt` decides what they
    contribute.
- **Wiring** (`641-plugin-copy-card.js`).
  - When `resolveCopyCardTask` returns a task (`state` `ready` or `no-id`), build the
    scope from `task.line` and pass `scope: { kind: "task", lines }` through
    `resolveCopyCardChip`.
  - Pass `result.scope` through, normalized: anything but `"task"` is `"cursor"`, so a
    v1 bob-copy-chips that ignores the scope degrades to today's line-only row.
  - Ambiguous results keep `count` and gain `scope`.
- **Row** (`buildCopyCardChipRow`).
  - Carry `chipScope: "cursor" | "task"` on the chip row. Add the field to
    `buildCopyCardRow`, `null` for other rows.
  - The ambiguous reason is `N chips on this line · move onto one` or
    `N chips in this task · move onto one`.
  - The ready label, value, and preview are unchanged.
- **Modal** (`103-copy-card-modal.js`).
  - When the chip row is ready with `chipScope === "task"`, the `is-chip` divider gets a
    `bob-copy-card-divider-title` reading `in this task`, in the same markup order as
    the TASK divider. No CSS change should be needed; check it in the e2e screenshot.
  - `copyCardRowAriaLabel` reads `Chip · <face>, in this task: <payload>` for that row.
  - The cursor-scope CHIP divider is unchanged (no title).

> [!decision] gy_follows_task **Nav API.** Add a frozen `taskScope` namespace, version
> 1, to `createDependencyNavApi`. The root `version` stays 3; the namespace is additive.
>
> - **`at({ lines, line })`** runs `resolveCopyCardTask` on a collapsed selection at
>   `line`, then the scope helper.
> - **It returns** a frozen `{ ownerLine, lines }`, or `null` for no task, a fenced
>   line, or invalid input.
> - **It is pure,** synchronous, and never throws, and it works with a `null` plugin.
>
> The Copy card and `gy` therefore compute the task with one function.

**Manifest.** Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version from
whatever it holds (now `2.90.0` → `2.91.0`).

### 3. Tests

Add every new `node --test` script to `test:unit` or `test:perf` in `package.json`.

**`scripts/test-copy-chips-cursor-api.cjs`** gets these new cases:

- A chipless cursor line plus a scope holding one chip on another line → ready,
  `scope: "task"`, with that chip's own `line` / `from` / `to` and a byte-exact payload
  (backticks, trailing spaces, table `\|`).
- Two chips across the scope → ambiguous task, `count: 2`.
- Precedence:
  - a chip on the cursor line beats the scope (`scope: "cursor"`, line count);
  - a 2-chip cursor line stays a line-ambiguous result.
- A `bob-copy` fence in the scope counts once, as a block chip on its opening line, even
  when several of its lines are in the scope. A plain fence holding `copy:` spans
  contributes nothing.
- A cursor inside a plain fence with a scope → none. Frontmatter lines in the scope are
  skipped.
- Malformed scopes are ignored and never throw:
  - not an object;
  - the wrong `kind`;
  - `lines` not an array;
  - junk indices;
  - a set without the cursor line.
- The namespace reports `version` 2, results are frozen, and `scope` is present on ready
  and ambiguous.
- The author model (`test-copy-chips-author-model.cjs`) still resolves contexts with no
  task step. The existing tests must pass untouched.
- **gy branch:**
  - the cursor command with a fake nav `taskScope` copies the task chip;
  - it shows each of the four notices;
  - nav absent → line-only;
  - a throwing `at` → line-only, with no throw;
  - the command name is "Copy chip at cursor".

**`scripts/test-navigation-copy-card-model.cjs`** gets these new cases:

- The scope helper agrees with `findOwningTaskLine` for every line on fixture notes:
  - nested subtasks;
  - Depends-On, BLOCKS, and LINKS lines;
  - a list-nested fence and a four-space fence under a child bullet;
  - blank lines inside the block.
- The cursor on a task line with the chip in a child note → a ready row with
  `chipScope: "task"`.
- The cursor on a child note with the chip on the task line → ready.
- A chip only in a nested subtask → no row from the parent; a ready row from inside the
  subtask.
- Two chips in the task → a dimmed row, `2 chips in this task · move onto one`; `C`
  nudges.
- No task (a plain bullet), a cursor in a fence, or a selection whose ends have
  different owners → today's line-only behavior.
- A v1-style API stub that ignores `scope` → no row on a chipless line. This is graceful
  degradation.
- **gy branch:**
  - `taskScope.at` equals the card's scope;
  - it returns `null` outside tasks;
  - it is frozen and never throws on junk.

**`scripts/test-navigation-copy-card-view.cjs`:**

- A task-scope ready chip renders `CHIP` plus the title `in this task`.
- A cursor-scope chip renders no title.
- The ambiguous task row is dimmed, has no title, and nudges.
- The aria-label carries "in this task".

**Perf:** add `scripts/test-navigation-copy-card-task-scope-performance.cjs` to
`test:perf`. Model it on `scripts/test-copy-chips-live-preview-performance.cjs` and
`scripts/cpu-work-perf-support.cjs`.

- **Input:** a 2,000-line note whose task owns about 400 child lines, with 40 chips and
  several fences, some of them `bob-copy`.
- **Measure:** the task, the scope, and `chipAt` with the scope.
- **Pass:** the whole resolve stays within the harness budget.

**Real-Obsidian e2e.** Extend `scripts/obsidian-copy-card-e2e.cjs` and its `-lib.cjs`.

- **Fixture.** The fixture already loads both plugins. Add three tasks:
  - one with a single chip on a child note, whose payload has backticks and a trailing
    space;
  - one with two chips on different lines;
  - a parent whose only chip sits in a nested subtask.
- **Probe.** Teach `cardProbeExpression` to read the CHIP divider title.
- **Cases:**
  1. Cursor on the first task's line, `Mod+Y` then `C`:
     - the clipboard gets the payload byte for byte;
     - the divider title reads `in this task`;
     - a `Copied …` Notice shows.
  2. Cursor on a chipless child line of the two-chip task:
     - the dimmed row says `2 chips in this task · move onto one`;
     - `C` leaves the clipboard unchanged.
  3. The parent of the nested subtask shows no `C` row.
  4. All existing chip cases still pass.
  5. **gy branch.** Run `bob-copy-chips:copy-on-cursor-line` on the first task's line:
     the clipboard gets the child chip. With `bob-navigation-hotkeys` disabled over CDP,
     the same command shows `No copy chip on this line`. Re-enable the plugin before
     later cases.

Keep `--self-test` (`scripts/test-obsidian-copy-card-e2e.cjs`) passing offline.

Run the e2e under `xvfb-run -a` in an isolated scratch vault, never the live vault.
apollo lacks Obsidian and `xvfb-run`. If this host lacks them too:

- run the e2e on athena against a snapshot of the built checkout, as earlier copy-chip
  phases did;
- read the `tailnet.md` reference memory with `/sase_memory_read` before using Tailscale
  SSH;
- if neither is possible, say so plainly in your final report, and never claim the e2e
  passed.

Capture one e2e screenshot of the task-scope card and look at it: the divider title
should sit like the TASK title, muted and ellipsized, never wrapping.

### 4. Docs (bob-plugins)

**`docs/copy-card.md`:**

- **Intro.** "An optional `C` chip row copies the copy chip at the cursor, or the only
  chip in the cursor's task, when Bob Copy Chips is enabled."
- **Keys table, `C` row.** "…the copy chip at the cursor, or the only chip in the
  cursor's task".
- **Resolution paragraph.** Replace it with the
  [What the `C` row shows](#what-the-c-row-shows) rules in prose. Cover:
  - the task fallback;
  - subtasks excluded;
  - fences;
  - the same-file invariant;
  - the `in this task` divider title;
  - both ambiguous reasons.

**`docs/bob-copy-chips.md`:**

- **Cursor rule section.** Add the opt-in task step and the `scope` request and result
  fields. Say that `Mod+Shift+Y` never uses the task step, and update the Authoring
  context sentence that calls it "the same cursor rule as Copy on cursor line".
- **gy branch.** Update the commands table and the gestures table for the new palette
  name and the task fallback.

**`docs/develop/plugin-apis.md`:**

- **Bob Copy Chips.** The namespace is version 2:
  - `chipAt({ lines, line, ch, scope? })`;
  - the `scope` result field;
  - a v1 provider ignores `scope`;
  - update the example to feature-detect `version >= 2` before passing a scope.
- **gy branch.** Add a `taskScope` row (version 1) to the Bob Navigation Hotkeys table,
  with one sentence on `at`.

Then run `npm run docs:gen` and `npm run docs:check`.

### 5. Verify and deploy

In the bob-plugins checkout:

- Run `npm run build` and `npm run build:check`.
- Run `npm test`. The only acceptable failures are the known load-sensitive ledger-tools
  budgets tracked as `bob-cli-km` (event-link-times) and `bob-cli-io` (frees-marks), and
  each must pass when rerun in isolation.
- Run `npm run test:perf`, `npm run validate`, `npm run docs:gen`, `npm run docs:check`,
  and `git diff --check`.

Then deploy each plugin:

```bash
bob plugins sync --repo "<bob-plugins checkout>" --no-pull -p bob-copy-chips
bob plugins sync --repo "<bob-plugins checkout>" --no-pull -p bob-navigation-hotkeys
```

- On a refusal, follow `AGENTS.md` and `docs/develop/deployment.md`; never pass `-F` on
  your own.
- Confirm that `bob plugins list` shows both plugins at their new versions, synced and
  enabled.

### 6. Vault comment (`gy_follows_task` only)

> [!decision] gy_follows_task Before touching the vault, read the `obsidian.md`
> reference memory with `/sase_memory_read`, then run `git status` in `~/bob`.
>
> In `~/bob/obsidian_vimrc.md`, rewrite only the comment above the `gy` mapping (now
> line 87) to:
> `" gy yanks the copy chip at the cursor — the one under the cursor, the line's only chip, or the task's only chip — and flashes it like a yank.`
> Leave the `exmap` and `nnoremap` lines alone.
>
> Commit only that file with `git add obsidian_vimrc.md && git commit`. Never use
> `/sase_git_commit` in the vault.

### 7. Memory (memory_glossary_copy_chip)

> [!decision] memory_glossary_copy_chip Read `/sase_memory_write` first. In
> `sase/memory/glossary/copy-chip.md`, replace only the cursor-rule paragraph.
>
> **When `gy_follows_task` is on**, the paragraph becomes:
>
> > The cursor rule, shared by `gy` and the Copy card: a chip under the cursor wins;
> > otherwise the only chip on the line; otherwise the `bob-copy` fence the cursor is
> > inside; otherwise, on a line with no chip, the only chip in the cursor's task (its
> > own lines, never a nested subtask's). Two or more chips on the line, or in the task,
> > with the cursor on none is ambiguous and never guessed. `Mod+Shift+Y` uses the rule
> > without the task step.
>
> **When `gy_follows_task` is off**, the paragraph becomes:
>
> > The cursor rule, shared by `gy` and the Copy card: a chip under the cursor wins;
> > otherwise the only chip on the line; otherwise the `bob-copy` fence the cursor is
> > inside; two or more chips on the line with the cursor on none is ambiguous and never
> > guessed. The Copy card's `C` row adds one step: on a line with no chip, the only
> > chip in the cursor's task (its own lines, never a nested subtask's).
>
> Then, in this bob-cli workspace:
>
> 1. Run `sase memory init`.
> 2. Run `just docs-gen` to regenerate `docs/reference/glossary.md`.
> 3. Run `just docs-check`.

### 8. Report

Tell Bryan:

- to reload Obsidian on each host;
- to run `bob plugins sync` on the Mac and athena so both builds reach them.

Name the e2e status honestly: run on which host, or not run.
