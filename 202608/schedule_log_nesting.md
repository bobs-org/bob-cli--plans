---
tier: tale
title: Nest schedule-log entries under the marker and restyle the log
goal:
  Each logged reschedule reason is written as a sub-sub-bullet nested under a `🗓️ **SCHEDULE LOG**` child bullet, with
  italic `*<from> → <to>*` entry dates, matching the target screenshot; legacy on-disk logs are still recognized and the
  one existing vault log is migrated in place.
proposed_by: bbugyi200.athena.tu.f0
create_time: 2026-08-06 08:29:56
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.tu.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.tu.f0.md)
- **COMMITS:**
  - [e5cca2d](https://github.com/bobs-org/bob-plugins/commit/e5cca2d342aa6811b6a442dc3a0765c46cbde340) —
    fix(bob-navigation-hotkeys): nest schedule-log entries under the marker

# Plan: Nest schedule-log entries under the marker and restyle the log

## The bug

`planScheduleLogEntry` in `plugins/bob-navigation-hotkeys/main.js` (~line 1604) renders the marker bullet and the first
entry bullet at the **same** indentation when it creates a fresh log:

```js
const indent = getDependencyChildIndent(lines, taskIndex);
const lineTexts = Object.freeze([
  formatScheduleLogParentBullet(indent, "-"),
  formatScheduleLogEntryBullet(indent, "-", entryFields), // <-- sibling, not child
]);
```

So the first reason logged on a task lands as a **sibling** of the `🗓️` marker rather than a child of it. The
prepend-onto-an-existing-marker branch is already correct: it uses `getScheduleLogEntryIndent`, whose fallback is
`markerIndent + "\t"`. Only the create branch is wrong, which is why the bug shows up on the first entry and then
silently "heals" from the second one onward (the second entry indents relative to the marker, landing one level deeper
than the first).

This is visible on disk right now at `~/bob/sase.md:105-106`.

## Scope (confirmed with the user)

Three changes, all confirmed:

1. **Nesting** — the entry becomes a grandchild of the task / child of the marker. This is the reported bug.
2. **Marker label** — `🗓️ **Schedule log:**` → `🗓️ **SCHEDULE LOG**` (all caps, colon dropped). The all-caps form
   matches the plugin's `🔗 **DEPENDS ON:**` house style; the colon is dropped because, unlike `DEPENDS ON:`, nothing
   follows the label on the same line, so a trailing colon dangles.
3. **Entry emphasis** — `**<from> → <to>**` → `*<from> → <to>*`. Bold marker over italic entries gives the log a real
   visual hierarchy instead of two competing bold spans.

Because (2) and (3) change the written format, the parse regexes must accept **both** the new and the legacy forms, and
the single existing log in the vault gets migrated in place.

Target shape:

```markdown
- [?] #task Ship the thing [priority:: medium] [scheduled:: 2026-08-20] ^ship
  - ![[#^blocked-by-this]]
  - Some freeform note I wrote by hand
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-13 → 2026-08-20_ — waiting on the API review to land
    - _2026-08-06 → 2026-08-13_ — was out sick
```

## Repos touched

- **`bob-plugins`** (linked repo — open with `/sase_repo` first, and use the printed path as the only path): all plugin
  code, tests, README, and manifest changes. Everything under `plugins/bob-navigation-hotkeys/` and
  `scripts/test-navigation-hotkeys.cjs`.
- **`bob-cli`** (your own workspace checkout): `docs/projects.md` only.
- **`~/bob/`** (Bryan's Obsidian vault — not a repo, no `/sase_repo`, no commit): a one-time two-line migration of
  `sase.md`.

Do not edit plugin files under `~/bob/.obsidian/plugins/`; they are overwritten by `bob plugins sync`.

`styles.css` needs **no** change — the `.bob-cnp-schedule-reason-*` rules carry no label text.

## Implementation

### 1. Constants (`main.js`, ~lines 244-257)

Replace the existing block with:

```js
// Managed "schedule log" child bullet, e.g. `  - 🗓️ **SCHEDULE LOG**`, with a
// newest-first list of `*<from> → <to>* — <reason>` entries nested one level
// under it. The emoji is `U+1F5D3 U+FE0F` (keep the variation selector). A
// marker written by hand without the emoji, and the legacy `**Schedule log:**`
// spelling, are both still recognized so an existing log is never orphaned or
// silently rewritten.
const SCHEDULE_LOG_EMOJI = "🗓️";
const SCHEDULE_LOG_LABEL = "SCHEDULE LOG";
const LEGACY_SCHEDULE_LOG_LABELS = Object.freeze(new Set(["Schedule log"]));
const SCHEDULE_LOG_MARKER_TEXT = `${SCHEDULE_LOG_EMOJI} **${SCHEDULE_LOG_LABEL}**`;
const SCHEDULE_LOG_ENTRY_EMPHASIS = "*";
const SCHEDULE_LOG_INDENT_UNIT = "\t";
const SCHEDULE_LOG_SEPARATOR = " — ";
const SCHEDULE_LOG_TRANSITION = " → ";
const SCHEDULE_LOG_PARENT_RE = new RegExp(
  `^(?<indent>\\s*(?:>\\s*)*)(?<marker>(?:[-*+]|\\d+[.)]))[ \\t]+(?<emoji>${SCHEDULE_LOG_EMOJI}[ \\t]+)?\\*\\*(?<label>${[
    SCHEDULE_LOG_LABEL,
    ...LEGACY_SCHEDULE_LOG_LABELS,
  ]
    .map((label) => label.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"))
    .join("|")}):?\\*\\*[ \\t]*$`,
);
const SCHEDULE_LOG_ENTRY_RE = new RegExp(
  `^(?<indent>\\s*(?:>\\s*)*)(?<marker>(?:[-*+]|\\d+[.)]))[ \\t]+(?<emphasis>\\*\\*?)(?:(?<from>.+?)${SCHEDULE_LOG_TRANSITION})?(?<to>.+?)\\k<emphasis>${SCHEDULE_LOG_SEPARATOR}(?<reason>.+)$`,
);
```

Notes for the implementer:

- The label alternation is built exactly the way `DEPENDENCY_NAVIGATION_BULLET_RE` (line 233) builds its
  `LEGACY_DEPENDENCY_NAVIGATION_LABELS` alternation, including the metacharacter escape `.map`. Keep that shape.
- `:?` (rather than `:`) makes the trailing colon optional for **both** spellings. `**SCHEDULE LOG:**` and
  `**Schedule log**` therefore also match; that is deliberate forgiveness, not an oversight.
- `(?<emphasis>\*\*?)` + the `\k<emphasis>` backreference accepts a single-`*` (new) or double-`*` (legacy) span while
  still rejecting a mismatched `**x*`. In `new RegExp(...)` the backreference must be written `\\k<emphasis>`.
- `SCHEDULE_LOG_INDENT_UNIT` mirrors `task-status-cycler`'s `CHILD_BULLET_INDENT_UNIT` (Obsidian's default Tab indent)
  and exists so the create branch and `getScheduleLogEntryIndent`'s fallback can never drift apart again — which is the
  root cause of this bug.

### 2. Formatters (`main.js`, ~lines 1438-1449)

```js
// Render the managed schedule-log marker bullet, e.g. `  - 🗓️ **SCHEDULE LOG**`,
// reusing an existing marker's own indent/marker character.
function formatScheduleLogParentBullet(indent, marker) {
  return `${indent}${marker} ${SCHEDULE_LOG_MARKER_TEXT}`;
}

// The text of one schedule-log entry without its indentation or list marker:
// `*<from> → <to>* — <reason>`, or `*<to>* — <reason>` when there was no
// previous value. Split out from formatScheduleLogEntryBullet so the modal
// preview renders the exact text the writers insert instead of duplicating the
// format inline.
function formatScheduleLogEntryText({ from, to, reason }) {
  const emphasis = SCHEDULE_LOG_ENTRY_EMPHASIS;
  const fromText = from ? `${from}${SCHEDULE_LOG_TRANSITION}` : "";
  return `${emphasis}${fromText}${to}${emphasis}${SCHEDULE_LOG_SEPARATOR}${reason}`;
}

function formatScheduleLogEntryBullet(indent, marker, fields) {
  return `${indent}${marker} ${formatScheduleLogEntryText(fields)}`;
}
```

Add `formatScheduleLogEntryText` to `module.exports` alongside the existing schedule-log helpers (~line 20314).

Leave `parseScheduleLogParentBullet`'s return shape as `{ indent, marker, hasEmoji }`. Do **not** add `label`/`isLegacy`
fields: a legacy marker is reused in place exactly like a current one, so nothing would consume them.

### 3. The nesting fix (`main.js`, `planScheduleLogEntry` create branch, ~line 1604)

```js
const block = findCurrentBulletChildBlock(lines, taskIndex);
const markerIndent = getDependencyChildIndent(lines, taskIndex);
// The entry is a grandchild of the task: one Obsidian Tab level deeper than
// the marker it belongs to, matching getScheduleLogEntryIndent's fallback for
// a marker that exists but has no entries yet.
const entryIndent = `${markerIndent}${SCHEDULE_LOG_INDENT_UNIT}`;
const lineTexts = Object.freeze([
  formatScheduleLogParentBullet(markerIndent, "-"),
  formatScheduleLogEntryBullet(entryIndent, "-", entryFields),
]);
```

Also change `getScheduleLogEntryIndent`'s fallback (~line 1545) from `` `${markerIndent}\t` `` to
`` `${markerIndent}${SCHEDULE_LOG_INDENT_UNIT}` ``. Behaviour is identical; the point is that the two code paths now
share one source of truth.

Nothing else in `planScheduleLogEntry` changes: `insertLine` (`block.endLineExclusive`), `createdParent`, and the
`lineTexts.join("\n")` for `lineText` all stay as they are, and both content-level writers
(`applyScheduleLogEntryToLines`) and the editor-level writer (`insertEditorLine`) already handle a two-line insert.

Keeping `getDependencyChildIndent` for the **marker** indent is correct: the marker is a direct child of the task and
should adopt an existing sibling's indentation. Only the entry moves one unit deeper. When a task's children use
two-space indentation, this yields `"  "` for the marker and `"  \t"` for the entry — mixed, but exactly what
`getScheduleLogEntryIndent` already produces for a pre-existing marker, so the two branches stay consistent.

### 4. Modal preview (`main.js`, `renderScheduleReasonPreviewItem`, ~lines 12281-12302)

Replace the inline title construction (the local `fromText` variable and its bold template literal) with the shared
formatter, so the preview can never drift from what is written:

```js
appendHighlighted(
  titleEl,
  formatScheduleLogEntryText({
    from: pending ? pending.from : "",
    to: pending ? pending.to : "",
    reason: item.reason,
  }),
  query,
);
```

and replace the two hard-coded marker strings at the end of the same method with `SCHEDULE_LOG_MARKER_TEXT`:

```js
textEl.createDiv({
  cls: "bob-cnp-schedule-reason-preview",
  text: item.parentExists
    ? `Appends to the existing ${SCHEDULE_LOG_MARKER_TEXT} on this task`
    : `Adds a ${SCHEDULE_LOG_MARKER_TEXT} child bullet to this task`,
});
```

No other modal change. The stage routing, footer hints, subtitle, `confirmScheduleReason`, and the empty-reason branch
(which still reads `scheduled → <date> only; no schedule log entry`) are all unaffected.

## Tests (`scripts/test-navigation-hotkeys.cjs`)

### Fixtures to update

Every `**Schedule log:**` literal becomes `**SCHEDULE LOG**`, and every entry bullet's `**…**` becomes `*…*`. Affected
lines today: 6211, 6219, 6229, 6233, 6245, 6260, 6300-6301, 6313-6314, 6325-6326, 6330, 6348-6349, 6364-6365, 6372-6373,
6384, 6432-6433, 6440-6441, 6467-6469, 6472-6473, 6540-6541, 6582.

Three of those fixtures also change **shape**, because they exercise the create branch:

- `planScheduleLogEntry creates, prepends, guards, and preserves blockquote context` (line 6336):

  ```js
  assert.deepEqual(created.lineTexts, [
    "\t- 🗓️ **SCHEDULE LOG**",
    "\t\t- *2026-08-13 → 2026-08-20* — waiting on the API review to land",
  ]);
  // ...
  assert.deepEqual(createdAfterNote.lineTexts, ["  - 🗓️ **SCHEDULE LOG**", "  \t- *2026-08-20* — kickoff"]);
  // ...
  assert.deepEqual(inQuote.lineTexts, ["> \t- 🗓️ **SCHEDULE LOG**", "> \t\t- *2026-08-20* — quoted context"]);
  ```

  The `createdAfterNote` case is the one that documents the mixed `"  \t"` indent described in §3 — add a short comment
  saying the marker adopts the sibling note's indentation and the entry goes one Tab deeper.

- `counted scheduled reason logs one entry per changed task...` (line 6437): Gamma's created log becomes

  ```js
  "\t- 🗓️ **SCHEDULE LOG**",
  "\t\t- *2026-07-05 → 2026-07-10* — sprint replan",
  ```

  Alpha's prepend stays at `"    - "` (its marker already exists). `scheduleLoggedTaskCount`, `cursorLine`, and the
  changed/unchanged counts are unchanged.

- `setBulletPropertyValue writes an inline scheduled date plus a schedule log entry` (line 6507):

  ```js
  "  - 🗓️ **SCHEDULE LOG**",
  "  \t- *2026-08-13 → 2026-08-20* — waiting on the API review to land",
  ```

- `setProjectNoteScheduledValue writes a schedule log entry under the ^prj task` (line 6547): the regex becomes

  ```js
  /- \[ \] #task Ship #hide \^prj\n\t- 🗓️ \*\*SCHEDULE LOG\*\*\n\t\t- \*2026-08-13 → 2026-08-20\* — waiting on the API review to land/,
  ```

### Tests to add

1. **Regression guard for the reported bug** — in the `planScheduleLogEntry` test, assert the relationship rather than
   just the literals, so the two lines can never collapse to the same level again:

   ```js
   const [markerLine, entryLine] = created.lineTexts;
   assert.equal(helpers.getBulletIndent(entryLine), `${helpers.getBulletIndent(markerLine)}\t`);
   ```

   (Use whichever indent accessor is already exported; if `getBulletIndent` is not in `module.exports`, compare the
   leading-whitespace prefixes directly.)

2. **Legacy marker recognition** — new assertions in the formatting/parsing round-trip test:

   ```js
   assert.deepEqual(helpers.parseScheduleLogParentBullet("\t- 🗓️ **Schedule log:**"), {
     indent: "\t",
     marker: "-",
     hasEmoji: true,
   });
   assert.deepEqual(helpers.parseScheduleLogParentBullet("  + **Schedule log:**"), {
     indent: "  ",
     marker: "+",
     hasEmoji: false,
   });
   ```

   Keep the existing rejection of `"  - **SCHEDULE LOG** trailing"` and of `"plain text"`.

3. **Legacy entry emphasis** — `parseScheduleLogEntryBullet` accepts both spans:

   ```js
   assert.deepEqual(helpers.parseScheduleLogEntryBullet("\t\t- **2026-08-13 → 2026-08-20** — legacy bold"), {
     indent: "\t\t",
     marker: "-",
     from: "2026-08-13",
     to: "2026-08-20",
     reason: "legacy bold",
   });
   ```

   plus the new italic round-trip through `formatScheduleLogEntryBullet` (already covered by the updated fixtures).

4. **`formatScheduleLogEntryText`** — returns exactly the bullet text minus indent and marker, with and without `from`.

5. **A legacy on-disk log is extended correctly** — this is the shape sitting in the vault today, where the old entry is
   a _sibling_ of the marker:

   ```js
   const legacy = [
     "- [ ] #task Ship [scheduled:: 2026-09-06] ^ship",
     "\t- 🗓️ **Schedule log:**",
     "\t- **2026-08-09 → 2026-09-06** — Because I like it.",
   ].join("\n");
   const next = helpers.planScheduleLogEntry(legacy, 0, {
     from: "2026-09-06",
     to: "2026-09-20",
     reason: "slipped again",
   });
   assert.equal(next.createdParent, false);
   assert.equal(next.insertLine, 2);
   assert.deepEqual(next.lineTexts, ["\t\t- *2026-09-06 → 2026-09-20* — slipped again"]);
   ```

   The legacy marker is found and reused in place; because it has no _children_, `getScheduleLogEntryIndent` falls back
   to marker + one Tab, so the new entry is correctly nested even though the old sibling entry below it is not. Add a
   comment recording that this transitional mixed shape is expected and is why the vault note is migrated by hand
   (below).

### Running the suite

From the `bob-plugins` repo root:

```bash
npm test
node scripts/validate-manifests.mjs
```

The suite is 300 tests today and must stay green.

## Vault migration (`~/bob/sase.md`)

Exactly one log exists on disk. Lines 105-106 are currently (leading whitespace is a single Tab on both):

```markdown
    - 🗓️ **Schedule log:**
    - **2026-08-09 → 2026-09-06** — Because I like it.
```

Rewrite them to:

```markdown
    - 🗓️ **SCHEDULE LOG**
    	- *2026-08-09 → 2026-09-06* — Because I like it.
```

That is: marker keeps its single Tab, entry gets two Tabs. Before editing, re-read the file and confirm those two lines
still match byte-for-byte; if they do not (the note is live and Obsidian Sync may have moved things), **skip the
migration and say so in the final report** rather than guessing. `~/bob/` is the vault, not a repo — no `/sase_repo`, no
commit.

Verify afterwards with `grep -rn "Schedule log" ~/bob --include=*.md`, which must return nothing.

## Docs and release

1. **`plugins/bob-navigation-hotkeys/manifest.json`** — bump `version` from `1.18.0` to `1.18.1` (a fix to the feature
   shipped in 1.18.0).
2. **`bob-plugins/README.md`** line 16 — update the version column to `1.18.1` and change
   ``records it under a `🗓️ **Schedule log:**` child bullet`` to
   ``records it as a nested entry under a `🗓️ **SCHEDULE LOG**` child bullet``.
3. **`bob-cli/docs/projects.md`**, the _"Schedule-log reason prompt"_ section (lines 284-316):
   - line 289: `` `🗓️ **Schedule log:**` `` → `` `🗓️ **SCHEDULE LOG**` ``.
   - the fenced example (lines 294-301) → the target shape shown at the top of this plan.
   - line 304-306: "The bolded date span shows…" → "The italicized date span shows…", and `` `**<date>** — <reason>` ``
     → `` `*<date>* — <reason>` ``.
   - line 303: make the nesting explicit — entries are children of the marker bullet, read newest first.
4. **Deploy** — from the `bob-plugins` repo root, after committing:

   ```bash
   bob plugins sync -r "$PWD"
   ```

   The explicit `-r "$PWD"` is required; without it `bob plugins sync` resolves the default repo path instead of the
   linked-repo checkout.

Commit `bob-plugins` and `bob-cli` separately, each with the `/sase_git_commit` skill.

## Edge cases the implementation must handle

1. **Legacy marker on disk** — recognized by `SCHEDULE_LOG_PARENT_RE`, reused in place, and never rewritten to the new
   spelling. A user's hand-written line is never silently changed.
2. **Legacy sibling entries under a legacy marker** — the next entry nests correctly under the marker; the old siblings
   stay put. Transitional and expected; covered by test 5.
3. **Legacy bold entry emphasis** — still parses, so nothing that reads the log breaks mid-migration.
4. **Emoji-less hand-written marker** — still recognized, emoji still not back-filled.
5. **Marker exists with properly nested entries** — unchanged behaviour: newest entry prepended at `parent.line + 1`
   with the existing entries' indentation and the marker's own list-marker character.
6. **Two-space-indented children** — marker at `"  "`, entry at `"  \t"`; consistent with the existing
   `getScheduleLogEntryIndent` fallback.
7. **Blockquoted task** — `getBulletIndent` keeps the `>` prefix, so both new lines stay inside the quote and the entry
   is `"> \t\t- "`.
8. **Task is the last line of the file** — `insertEditorLine` still appends both lines past the end; `insertLine` stays
   `1`.
9. **Empty reason** — unchanged: nothing written, no marker created.
10. **`bob projects sync`** — still unaffected. `parse_prj_sub_block` collects every descendant of a `^prj` task but
    only marks `🧩 **Sub-projects:**` lines as `is_marker`, and `subprojects_need_normalization` only inspects marker
    lines, so a deeper-nested schedule-log entry is as invisible as a shallower one. The "no preceding blank line"
    constraint is untouched by this change.

## Out of scope

- Auto-migrating legacy markers/entries in place when the plugin next touches a log. The regexes read them, the one real
  instance is migrated by hand, and silently rewriting a user's line is against this feature's stated contract.
- A configurable label, emoji, emphasis style, or indent unit.
- Any change to when the reason prompt fires, to `Ctrl+D`, or to priority rolls.
- Reading or querying the log from `bob`.
