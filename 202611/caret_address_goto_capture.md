---
tier: epic
title: '@route^id Go To: open a task by its exact address from Bob Capture'
goal: 'A capture draft that is exactly one bare `@route^block-id` token resolves read-only
  to that block''s line and, on Return, opens it in Obsidian through the same `obsidian://bob-goto-task`
  landing the `/` picker uses. The panel shows the destination before Return, typing
  `@route^` browses that note''s IDs as Go rows, misses are calm did-you-mean states,
  nothing is ever written, and every other caret form keeps today''s meaning byte
  for byte.

  '
phases:
- id: cli-goto
  title: 'bob-cli: claim, resolve, and refuse the bare caret address'
  depends_on: []
  size: medium
  description: 'cli-goto: add the shared whole-draft caret-address claim, the read-only
    resolver (exact hit, duplicates, archive hop, did-you-mean), the capture-parse
    task_goto mode and goto payload, the bob capture refusal, grammar +1, docs, and
    tests.'
- id: cli-browse
  title: 'bob-cli: Go rows for @route^ completion, mint-collision copy, and /route^id
    search'
  depends_on:
  - cli-goto
  size: medium
  description: 'cli-browse: add the additive block_id.goto rows to task_block_id completion
    for bare @route^ and partial IDs, teach the new form in the mint-collision error,
    index route^id in the / search fields, and document all three.'
- id: plugin-refind
  title: 'bob-plugins: any-line block fallback in the Go To re-find ladder'
  depends_on: []
  size: small
  description: 'plugin-refind: let bob-goto-task re-find a unique non-task line carrying
    ^block when the task-only lookup misses, with tests, docs, build, and plugin sync.'
- id: mac-card
  title: 'Mac: Go To destination card and Return dispatch'
  depends_on:
  - cli-goto
  size: medium
  description: 'mac-card: decode the parse goto payload, render the found, miss, and
    duplicate cards in place of the write preview, relabel the primary button, and
    dispatch the Go To on Return through the / landing path with an accept-time fresh
    parse.'
- id: mac-browse
  title: 'Mac: Go rows in the block-ID card'
  depends_on:
  - cli-browse
  - mac-card
  size: medium
  description: 'mac-browse: turn the goto-eligible block-ID card into a scoped Go
    To browse (selectable Go rows, exact preselect, archived section, Return lands,
    Tab completes), update mint copy, and mirror the /route^id search key.'
- id: closeout
  title: End-to-end verification and docs closeout
  depends_on:
  - plugin-refind
  - mac-browse
  size: small
  description: 'closeout: verify every acceptance case against the real vault read-only,
    confirm plugin deploy and green macOS CI on the landed SHAs, reconcile cross-repo
    docs, and write the decision record only if approved.'
decisions:
  caret_goto_decision_record:
    ask: Add a decisions memory strand recording that a bare whole-draft @route^id
      is a read-only Go To?
    memory:
    - decisions:caret-address-goes-to
    default: false
    answer: false
proposed_by: bbugyi200.athena.1g6
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 15:25:22
status: wip
bead_id: bob-cli-q7
---

- **PROMPT:** [prompts/202611/caret_address_goto_capture.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/caret_address_goto_capture.md)
- **BEAD:** [bob-cli-q7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q7/README.md)

# Plan: `@route^id` Go To in Bob Capture

## Why

Today a draft that is exactly `@bob^intention-rollovers` is a dead end. `capture-parse`
calls it a complete `task` (`needs: []`, empty body), the panel enables Capture, and the
live dry run fails with `task text is required; pass TEXT or pipe it on stdin`. The
panel then paints that error twice (inline red text and the red preview box). Every
lexical neighbour already means something: `@foo` pulls, `@foo:bar` links, `@foo+bar`
ensures Next, `text @foo^bar` mints, and `@foo^bar@` seals a project note. The bare
caret form is the only unclaimed hole in that family, and a read-only Go To fits it
naturally.

`/` searches when you remember words. `@foo^bar` addresses a task when you know where it
is. The two complement each other, and the second reaches what `/` cannot: closed tasks,
archived tasks, non-task blocks, and notebook notes.

This plan implements the consolidated research
`research:202611/caret_address_goto_capture/caret_address_goto_capture.md`. Bryan
accepted all of its recommended requirements, which settles its four open questions:

- The archive fallback ships in v1.
- Any unique block is a valid target, badged by what it is.
- The primary button reads `Go To ↗`.
- Return in the scoped browse lands immediately.

The memory change is the only exception. It is gated by the decision below.

## Product contract (all phases)

### What counts as a caret Go To

A draft is a caret Go To exactly when all of these hold:

- After trimming surrounding whitespace, it is one item on one physical line.
- That line is one whitespace-delimited token `@<route>^<block-id>`.
- `<route>` uses today's route charset and is matched lower-cased, exactly as minting
  resolves it.
- `<block-id>` is non-empty and passes the Bob validator `[A-Za-z0-9-]+`.
- The token has no body text at either edge. It also has no `@` seal, `#section`,
  `#type`, `s:`, `$`, `%` directive, `=` suffix, `+`/`:` separator, children, second
  item, or `@@` declaration.

This is the same whole-draft rule `/` and `//` use. Bryan's examples:

| Draft                                                 | Behavior                                                                                                                               |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `@bob^intention-rollovers` (± surrounding whitespace) | Go To card; Return opens `bob.md` at that line                                                                                         |
| `@bob^` / `@bob^int`                                  | Scoped Go To browse (block-ID card with Go rows); a partial ID that matches nothing exactly shows the calm miss card behind the picker |
| `@bob^x Fix it` / `Fix it @bob^x`                     | Mint with the authored ID, unchanged, including the duplicate-ID refusal                                                               |
| `@bob^x@`, `@bob^x@ %n`, `@bob^x#section`             | Seal, notebook creation, or section bullet, unchanged                                                                                  |
| `@bob^x` + a child bullet, or + a second item         | Not claimed; today's behavior, byte for byte                                                                                           |
| `@bob:x`, `^bob:x`, `@bob+x`                          | Link and ensure-Next writes, unchanged                                                                                                 |
| `@taxes^z-250414-15` on a notebook route              | Go To the named note bullet, badged `Note` (today: `notebook_note_required`)                                                           |
| `Discuss @bob^x later`                                | Literal task text, unchanged                                                                                                           |

### What it lands on

The resolver lands on the unique line whose trailing block ID is exactly `^<block-id>`.
The match is case-sensitive, and lines inside frontmatter or fenced code don't count.
The line may hold:

- a task of any status, open or closed;
- a notebook note bullet;
- a plain bullet;
- a heading;
- a paragraph.

The card badges what it found. A block ID is a note-unique address, and Obsidian's own
`[[foo#^bar]]` link opens whatever sits there.

### What it never does

Go To has no side effects of any kind:

- no Task Link;
- no lane raise;
- no freshness stamp;
- no ledger write;
- no walk move;
- no block-ID minting;
- no stash entry;
- no capture-history entry;
- no "Captured" toast.

The vault's `git status` is identical before and after a resolution and after an
accepted navigation.

### Resolution ladder (bob-cli, read-only)

1. **Resolve the route** exactly as minting does: `bob_dir.join(route_label(route))`.
   - Missing note → `missing_note`.
2. **Scan the living note** for lines whose trailing block ID equals the requested ID.
   - One hit → `found`.
   - Two or more hits → `duplicate_block_id`, naming every hit. This fails closed and
     never guesses.
3. **Zero hits → follow the archive pair once.**
   - Read the note's `done_tasks:` frontmatter link (actual YAML frontmatter only, as
     `collect_done/pair.rs` reads it).
   - If that link is absent, use `collect_done::pair::canonical_archive_path` when that
     file exists.
   - Scan the archive the same way. A unique hit → `found` with `archived: true`, and
     `note_path` names the archive file.
4. **Still nothing → `missing_block_id`**, with up to three did-you-mean suggestions:
   - First, IDs in the same living note that differ only by case, extend the typed ID as
     a prefix, or are within edit distance 2.
   - Only when that pass finds nothing: the exact same ID in other routable notes from
     the catalog `/` already reads (`capture_link_tasks` / `capture-targets`).

   Each suggestion carries the exact replacement draft (`@other^id`). The panel applies
   it only as an explicit draft edit.

Prefixes never auto-resolve. `@bob^intention` does **not** land on
`intention-rollovers`. Prefixes belong to the browse.

### Wire contract (additive, `schema_version` stays 1)

**`bob capture-parse`.** For a claimed draft, the top-level object and the single item
report `mode: "task_goto"`, `needs: []`, and no diagnostics.

- **Spans:** the existing `task_block_id_route` and `task_block_id` spans stay, so the
  route stays blue and the ID indigo.
- **Grammar:** `grammar` is bumped by one from whatever master holds when `cli-goto`
  lands (31 today).
- **`goto` object:** the response gains a top-level `goto` object, omitted for every
  other draft:

```json
"goto": {
  "status": "found",
  "route": "bob",
  "block_id": "intention-rollovers",
  "note_path": "bob.md",
  "target": "/abs/vault/bob.md",
  "archived": false,
  "landing": {
    "block_id": "intention-rollovers",
    "note_path": "bob.md",
    "target": "/abs/vault/bob.md",
    "line": 82,
    "ref": "82:1a2b3c4d",
    "target_kind": "task",
    "text": "Allow rollovers of intentions",
    "status_symbol": "*",
    "status_name": "Next",
    "status_type": "TODO",
    "section": "Tasks",
    "closed": null
  }
}
```

**`goto` fields by status:**

- **`status`** is one of `found`, `missing_note`, `missing_block_id`, or
  `duplicate_block_id`.
- **`landing`** is present only for `found`.
- **`hits`** is present only for `duplicate_block_id`: a list of rows in document order.
- **`suggestions`** is present only for `missing_block_id`, possibly empty. Each entry
  is a row plus a `draft` field (the whole replacement draft) and a `reason` field
  (`case`, `prefix`, `near`, or `other_note`).
- **`target`** is omitted for `missing_note`.

**The shared GotoRow shape** is reused by `landing`, `hits`, `suggestions`, and the
completion rows in `cli-browse`.

- **Fields:**
  - `block_id`;
  - `note_path` (vault-relative with extension);
  - `target` (absolute);
  - 1-based `line`;
  - `ref` (`<line>:<digest>`, computed with exactly the digest the `/` rows use);
  - `target_kind`;
  - `text`;
  - optional `status_symbol` / `status_name` / `status_type` (tasks only);
  - nullable `section` (the nearest preceding heading);
  - optional `closed` date;
  - `archived` (omitted when false).
- **`target_kind`** is one of:
  - `task`;
  - `notebook_note` (a bullet on a notebook route);
  - `bullet`;
  - `heading`;
  - `paragraph`.

**`bob capture` and `bob capture --dry-run`.** Both refuse a claimed draft before any
other item parser, beside the T3/T4 `/` and `//` refusals.

- Forced flags change nothing, and the whole batch rolls back.
- The new T5 teaching error keeps the usual `capture item 1 starting on line L: …`
  prefix:
  `` `@bob^intention-rollovers` opens bob.md at ^intention-rollovers in Obsidian — it never captures; add task text after the ID to create a task with this ID ``
- On a notebook route the clause reads "add note text after the ID to create a note with
  this ID" when the notebook policy is available at that point. Otherwise it keeps the
  task wording.
- The refusal is purely lexical and never reads the vault.

**`bob capture-complete`.** On `@route^` / `@route^<partial>` that is the whole draft:

- The context stays `task_block_id`.
- `block_id.intent` stays `"new"`.
- The existing `used` / `suggestions` fields stay unchanged.
- `block_id` gains an additive `goto` object (shape below, built in `cli-browse`).

Old Mac apps therefore keep today's composer.

**Version skew:**

|         | Old Mac app                                                                                                  | New Mac app                                               |
| ------- | ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| Old bob | Today's error                                                                                                | No `goto` object, so the panel behaves as today; no crash |
| New bob | Unknown mode is ignored; the dry run hits the T5 refusal (an honest error, never a phantom "Captured" toast) | Full feature                                              |

Releases ride the daily train. Commit subjects use `feat(capture): …` **without `!`**.
No input that used to capture successfully changes meaning: the claimed form always
failed in execution, so this is an additive mode for previously dead drafts. Record that
reasoning in the commit body.

### Panel experience (the "beautiful" part)

The panel always shows the destination **before** Return. It never surfaces red for a
form the user typed correctly. The found card sits in the auxiliary region at the fixed
eye line, where the write preview normally renders, and reuses the `PreviewPane` chrome
(thin material, radius 8, padding 10):

```text
┌──────────────────────────────────────────────────────────────┐
│ ↗ GO TO                                   bob.md · line 82   │  quiet small caps; locator right-aligned, monospaced digits
│ [*]  Allow rollovers of intentions                           │  status glyph + text: the strongest content, 2-line clamp
│      Next · Tasks · ^intention-rollovers                     │  quiet meta; the ID in indigo
│ ──────────────────────────────────────────────────────────── │
│ Opens in Obsidian · nothing is written                       │  caption, tertiary
└──────────────────────────────────────────────────────────────┘
```

**Badges.** Small capsules after the header: `Archived · done/bob_done.md`, `Note`,
`Heading`, `Bullet`, `Paragraph`, `Done`, `Canceled`. Closed tasks use their status
glyph.

**Footer.**

- The primary button reads **`Go To ↗`**. It is enabled only for a fresh `found` parse.
- The Preview button and the `⌃↩ Open` chip are hidden.
- The footer status reads `Go To`.
- No live dry run is spawned.

**Miss card** (`missing_block_id`). Calm, never red:

- A `questionmark.circle` glyph in secondary color, then
  `No ^intention-rolovers in bob.md`.
- A `DID YOU MEAN` group of up to three rows in the picker row style, each with a status
  glyph, `^id` in indigo, text, and `note · line`. Clicking a row (or ⌘1–⌘3) replaces
  the draft with that row's `draft` value.
- A closing teaching line: "Add task text after the ID to create a task with this ID."
- The primary button stays visible but disabled.

**Missing note.** `No note named bob2.md` plus the same teaching line. Minting creates
the note, which was verified live.

**Duplicate.** `^x appears 2× in bob.md` with each hit as a row (line, glyph, text).
Clicking a hit opens exactly that line. This is an explicit pick, so it never lands
somewhere the card did not show. The copy below the rows reads: "Block IDs should be
unique — fix one in Obsidian." Return does nothing.

**Keys.**

- Return, ⌘↩, ⌃↩, and the primary button all dispatch the same Go To.
- Ctrl-C discards without stashing.
- Esc, q, and Ctrl+[ behave as `decisions:q-and-ctrl-bracket-mirror-esc` requires
  (unchanged).

**Accept.** Every accept follows the same sequence:

1. **Guard** behind the in-flight flag already used by `/`.
2. **Fresh parse.** Run one fresh `capture-parse` of the current draft. Dispatch only if
   it still reports `found`, and use the fresh `landing`.
3. **Dispatch** `obsidian://bob-goto-task` with `path`, `line`, `ref`, and `block`.
4. **Settle.**
   - Success → discard the draft (no stash) and hide the panel.
   - Dispatch failure → keep the draft and show "Couldn't open Obsidian".
   - The fresh parse no longer finds the target → apply that parse so the card updates,
     and keep the draft.

Return pressed before the first parse lands uses the same fresh-parse path. If the fresh
parse is not `task_goto`, the Return is dropped exactly as today.

**Scoped browse (`@bob^`, `@bob^int`, `@bob.` via the `.` → `^` alias).**

- **Rows.** The block-ID card becomes "Go To · bob.md" with selectable Go rows (status
  glyph, text, `^id` in indigo, `line N` quiet).
- **Archived section.** A trailing "Archived · done/bob_done.md" section appears only
  once the typed ID is non-empty.
- **Selection.** An exact match is preselected.
- **Keys.** Return / ⌃↩ / click land immediately. Tab completes the ID into the draft,
  which shows the found card.
- **Hidden rows.** "Next free" and the new-ID row are hidden.
- **Leaving browse.** Typing body text at either edge reverts live to today's mint
  composer, with no red flash in between. The same holds across the glued `.` toggle
  `@bob:id` ↔ `@bob^id`, which becomes link ↔ Go To.

**Naming.** Never call this feature "jump" in code or copy: Ctrl+J successor pins and
`CaptureJumpJson` own that word. Use `caret_goto` / `CaretGoto` in identifiers. This
keeps them distinct from the `/` picker's existing `task_goto` need and context.

---

## Phase `cli-goto` — bob-cli: claim, resolve, and refuse

The closest precedent is `fc3d6eb4` (`//` Go To Notes). Mirror its file footprint and
test style.

1. **Shared claim predicate.** In `src/native/capture_language/tokens.rs`, beside
   `task_goto_query_token` / `note_goto_query_token`, add `caret_address_token(item)`.
   - It returns `{route, block_id (may be empty), token, route_range, id_range}` for a
     whole-draft, single-line, single-token `@route^id-or-empty`, with
     `item.source.trim() == token.text`.
   - The `^` must be followed only by block-ID bytes
     (`capture_block_ids::is_block_id_byte`).
   - Route bytes must use the existing route validator. Reject `@@`.
   - Add `caret_goto_token(item)`, which returns `Some` only when `block_id` is
     non-empty. Parse and execution use it. Completion eligibility in `cli-browse` uses
     `caret_address_token`.
   - Nothing else may re-derive the rule. Doc-comment both functions like their
     siblings.
2. **Every claim site honors it.** Grep every caller of `task_goto_query_token` and
   `note_goto_query_token` and give the caret claim the same treatment:
   - `editor_parse.rs` `parse_editor_item_inner` (claim before the notebook editor view
     and before `task_link`);
   - `notebook.rs:1398` (the notebook classifier must step aside so `@taxes^z-…` stops
     reporting `notebook_note_required`);
   - `capture_task_wikilinks.rs:122`;
   - `editor_parse.rs:576`;
   - `completion.rs`.
3. **Editor parse.** Add `EditorMode::TaskGoto` (label `task_goto`) in `editor_model.rs`
   and `parse_editor_caret_goto_item`. It reports the item with route, block ID,
   `needs: []`, no diagnostics, and the same `task_block_id_route` / `task_block_id`
   spans today's parse emits for the token.
4. **Execution refusal.** In `item.rs`, beside the T3/T4 checks (before the `&`
   dependency scan), return the T5 error built by a new
   `caret_goto_error(token, route_label, block_id, notebook)` in `markers.rs`.
   - Update the pin in `src/native/capture/tests/grammar.rs:948`
     (`("@cash^id", "task text is required")`) to the T5 text.
   - Check the other `task text is required` tests in that file, which should stay
     unchanged.
5. **Resolver module.** Add a new `src/native/capture_caret_goto.rs`, registered in
   `src/native.rs`, that owns the ladder above and the `GotoRow` builder.
   - **Reuse, don't re-derive:**
     - `capture::route_label`;
     - `note_tasks::read_settings` + `note_tasks::scan` (task status, description, and
       fence awareness);
     - `collect_done::trailing_block_id_in_line`;
     - the non-task text cleanup in `capture_block_ids` (`nontask_text`; make it
       `pub(crate)`);
     - the same digest/`ref` helper the `/` rows use (`LinkTask::task_ref` /
       `note_tasks::line_digest`);
     - heading detection from `capture::sections`;
     - the closed-date extraction used by `closed_task_goto_candidates`;
     - the notebook route check `is_notebook_route`.
   - **Archive link:** expose a small `pub(crate)` helper in `collect_done/pair.rs` that
     reads the `done_tasks:` frontmatter value. It should wrap the existing
     `frontmatter_value`. Resolve `[[done/x]]` / `[[done/x|alias]]` to a vault-relative
     `.md` path under the vault. Never escape it, and never build a vault-wide
     `NoteIndex` per keystroke.
   - **Suggestions:** edit distance is a tiny local function (no new crate). The
     cross-note pass reads the routable catalog only when the in-note pass found
     nothing.
6. **Attach in `capture-parse`.** In `capture_parse.rs::run`, after
   `CaptureParseResult::new` (beside `attach_gmail_spans`), call
   `attach_caret_goto(&mut result, &bob_dir)`.
   - Add the `goto: Option<…>` field with `skip_serializing_if`. Keep the grammar layer
     pure, as Gmail and themes do.
   - The human (non-JSON) renderer prints one readable line, for example
     `go to  bob.md:82  [*] Allow rollovers of intentions` or
     `go to  no ^x in bob.md (did you mean ^y?)`.
7. **Grammar, help, and docs.**
   - Bump `GRAMMAR_REVISION` (+1) with a revision note in `capture_language/mod.rs`.
   - Update the `capture-parse` long help: `grammar: N`, the `task_goto` mode in the
     Modes list, a `goto` paragraph, and an alphabetized example
     `bob capture-parse -f json -- '@bob^intention-rollovers'`.
   - Regenerate the CLI reference with `just docs-gen`.
   - **`docs/capture.md`:** add a new section, "Going to a task with `@route^id`", right
     after "Going to a note with '//'".
     - Explain the claim using the table above.
     - Describe the ladder and the no-side-effects promise.
     - Include the T5 text and the parse contract.
     - Add a four-form teaching table: `@r:x` links, `^r:x` links, `@r^x` opens,
       `text @r^x` creates.
     - Update the Contents list.
   - **Reference pages:**
     - add a row and an anchor in `docs/reference/capture-syntax.md`, cross-linked from
       `#at-route-caret`;
     - update `docs/reference/json-outputs.md` and `docs/reference/compatibility.md`
       (new grammar row).
   - Run `just docs-check`.
8. **Tests.**
   - **Unit tests:** add `capture_language/tests/caret_goto.rs`, mirroring
     `note_goto.rs`, for every row of the acceptance table, including whitespace,
     notebook, and `@@`-in-another-item cases.
   - **CLI tests:** add `tests/cli/capture/caret_goto.rs` against a synthetic vault that
     covers:
     - an open task;
     - a closed task;
     - a heading block;
     - a bullet block;
     - a `^id` inside a fence (ignored);
     - a duplicate;
     - an archive pair through both `done_tasks` and the canonical path;
     - a notebook route;
     - a missing note;
     - case, prefix, near, and other-note suggestions.
   - **Assertions:** check exact JSON, and that dry-run and real capture refuse
     identically. Check that a vault tree hash is unchanged after parse and after both
     refusals.
   - Update existing CLI tests that pin `grammar` or the bare-token error.
   - **Timing:** on athena, time `capture-parse` for a found address and a worst-case
     miss on the real vault (read-only). Keep each under ~50 ms, and record the numbers
     in the bead notes.

## Phase `cli-browse` — bob-cli: completion Go rows, mint copy, `/` index

1. **Completion `goto` payload.** In
   `capture_block_ids.rs::build_block_id_field_with_notebooks`, when the marker is `^`,
   the intent is `New`, and `caret_address_token` claims the draft (empty or partial
   ID), attach an additive `goto` object to `BlockIdField` (`skip_serializing_if`):
   - `eligible: true`;
   - `note_path`;
   - `target`;
   - optional `archive_path`;
   - `rows: [GotoRow]`, built with the `cli-goto` resolver's row builder.

   **Rows:**
   - Every uniquely identified block in the living note comes first, in document order.
   - IDs that occur more than once are omitted; the parse card explains them.
   - Rows from the paired archive follow, flagged `archived: true`.
   - The query is **not** filtered server-side. The Mac filters, as it does `used`.

   **Unchanged:** `intent`, `used`, `suggestions`, and every non-eligible draft stay
   byte-identical.

   Update the `capture-complete` help paragraph for `task_block_id`, then run
   `just docs-gen`.

2. **Mint-collision copy.** The ordinary task mint refusal
   `block ID ^<id> already exists in <path>` gains the clause
   `; remove the task text to go to it with @<route>^<id>`, so the error teaches the new
   form. Find its single construction site and update pinned tests.
3. **`/` search index.** In `capture_link_tasks.rs::link_task_search_fields`, add
   `format!("{}^{block_id}", task.route)` beside the existing `route:block-id` field, so
   `/bob^intention-rollovers` finds the task. Add a ranking test, and update any
   `task_link` / `task_goto` ranking pins that legitimately shift.
4. **Docs.**
   - Extend the new `docs/capture.md` section with the browse contract (the
     `block_id.goto` shape, ordering, and archived rows) and the `/route^id` search key.
   - Note in `docs/reference/capture-syntax.md` that `@route.` → `@route^` opens the
     browse.
   - Run `just docs-check`.
5. **Tests.**
   - Completion unit and CLI tests cover: empty ID, partial ID, the notebook route, a
     note with an archive pair, a missing note (`note_exists: false`, no rows), and
     body-at-either-edge (no `goto`).
   - Add a `task_goto` completion test for `/bob^…`.

## Phase `plugin-refind` — bob-plugins: any-line block fallback

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. Edit `src/` fragments
only, never `main.js`.

1. In `plugins/bob-navigation-hotkeys/src/675-plugin-goto-task-uri.js`, add
   `findAnyLineByBlockId(lines, blockId)`.
   - It returns the index of the **unique** line whose trailing block ID
     (`getTrailingBlockId`) equals the ID, skipping fenced code with the existing fence
     helpers if any.
   - It returns `null` for zero or several hits.
   - Keep it in the 675 fragment, because `180-editor-transactions.js` is already at 946
     of its 1000 lines.
2. In `refindGoToTaskLine`, after the task-only `findTaskLineByBlockId` returns `null`,
   try `findAnyLineByBlockId` and return `{ source: "block-id-any" }`. Task behavior and
   URI v1 stay unchanged.
3. Extend `scripts/test-navigation-goto-task-uri.cjs` to cover:
   - a shifted heading block;
   - a shifted bullet block;
   - a fenced `^id` ignored;
   - duplicate non-task IDs falling through to digest/clamp;
   - task-first precedence unchanged.
4. Update `docs/goto-task-uri.md` step 2 of "Re-finding the line". This page is mounted
   into the Bob site as `reference/plugins/goto-task-uri.md`.
5. Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`,
   following the AGENTS.md refusal protocol.

## Phase `mac-card` — Mac: destination card and Return dispatch

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`. Never land uncompiled
Swift.

1. **Decode** (`Sources/CaptureCore/CaptureModels.swift`). Add the `CaptureCaretGoto`
   and `CaptureGotoRow` types.
   - `status` is an enum with an `.unknown` fallback.
   - Add `goto` to `CaptureParseResponse` with the tolerant
     `(try? decodeIfPresent(...)) ?? nil` pattern used by `record` / `mergeIn` (stored
     property, init default, CodingKey).
   - Feature detection is the presence of `goto` plus `mode == "task_goto"`. Do **not**
     raise `CaptureGrammarGate.minimumRevision`.
2. **Model** (`CapturePanelModel.swift`).
   - **State:** add `caretGoto`, derived only from an applied parse whose draft,
     generation, and client identity match the current draft (the
     `captureGrammarAllowsSubmit` / `mergeInNeedsSource` freshness check).
   - **No dry run:** in `scheduleAnalysis` (the `pickerNeeded == nil` gate near
     `:20220`) and `isCurrentLivePreviewRequest`, skip the live dry run for caret Go To
     drafts. Reset `previewState = .idle` and `errorMessage = nil` the way
     `applyQuietIncompletePicker` does, so the doubled red error can never appear.
   - **Footer:** put a `Go To ↗` branch first in `primaryActionTitle`.
     `primaryActionEnabled` requires `found` and no in-flight dispatch. Hide the Preview
     button and `openHintChipVisible` in this mode, and suppress the Similar-tasks band.
   - **Submit:** intercept at the top of `submit(openAfterCapture:)` so every key path
     funnels into `acceptCaretGoto()`. That method implements the accept steps from the
     contract (in-flight guard, fresh parse, dispatch, settle). Factor a shared
     `dispatchGoto(destination:)` from `acceptTaskGotoRow` / `openFocusedSimilarRow`:
     - build the URL with `ObsidianOpenURL.taskGotoURL`;
     - open it with `taskGotoOpener`;
     - on success, call `discardDraft()` and then `panelDismisser()`;
     - on failure, announce "Couldn't open Obsidian".

     Then use it from all three sites.

   - **Stale-parse Return:** await the in-flight analysis with a bounded wait, reusing
     the `submitAfterCurrentAnalysis` / `PendingExplicitSubmitIntent` machinery. Route
     to the Go To only for a `task_goto` result; otherwise drop the Return as today.
   - **Ctrl-C:** add the caret mode to `draftIsReadOnlyGoto` so Ctrl-C discards without
     stashing.
   - **Picker ownership:** while `caretGoto` is active, don't auto-open today's New-ID
     block-ID picker (`handleBlockIDCompletion`). Leave the chip only, so the card owns
     Return. Otherwise pasting a full address mounts the "already used" picker, which
     disables the button and captures Return. `mac-browse` brings the picker back in its
     Go-rows form.

3. **View** (`CapturePanelView.swift`). Add `CaptureGotoCard` as a new branch in
   `auxiliaryRegion` / `auxiliaryContent`, ranked just below the picker so an open
   browse picker still wins.
   - It renders the found, miss, missing-note, and duplicate layouts from the contract.
   - Reuse `CaptureEditorPalette.taskStatus`, `CapturePickerTaskStatus(symbol:name:)`,
     the `CapturePickerRowView` locator styling (route `.accentColor`, ID `.indigo`),
     and the `PreviewPane` chrome.
   - Suggestion rows apply their `draft` through the model as an explicit edit, via
     click and ⌘1–⌘3. Duplicate hits dispatch that exact row.
   - Respect the eye-line tests (`CapturePanelEyeLineTests`).
4. **Fixtures and fake bob.**
   - Build bob from your bob-cli checkout (which contains `cli-goto`) and point
     `BOB_BIN` at it.
   - Add `Scripts/generate-caret-goto-fixtures.sh`, modeled on
     `generate-note-goto-fixtures.sh` with a synthetic vault, producing
     `task-caret-goto-parse-{found,archived,notebook,missing-block,missing-note,duplicate}.json`.
   - Add the matching `capture-parse` cases to `Tests/Fixtures/fake-bob`.
5. **Tests.**
   - **`CaptureCoreTests` (portable):** decoding, including an old response without
     `goto`.
   - **`BobMacCaptureTests`**, modeled on `CaptureTaskGotoPanelTests` with an injected
     `taskGotoOpener` / `panelDismisser`:
     - Return, ⌘↩, ⌃↩, and the button dispatch exactly once;
     - a double Return dispatches once;
     - a dispatch failure keeps the draft;
     - a fresh parse turning into a miss keeps the draft and updates the card;
     - Return before the first parse lands still goes to the target;
     - miss and duplicate cards never dispatch on Return;
     - no `capture --dry-run` is spawned;
     - the button title and hidden chip/preview are correct;
     - Ctrl-C does not stash;
     - an old-bob response behaves as today.
   - Async tests must never await untimed continuations.
6. **Docs.**
   - `docs/capture-panel.md`: a new "Go To card" section, plus the Live Preview and
     Submit notes.
   - `docs/keyboard.md`: Return in Go To mode.
   - `docs/task-pickers.md`: one cross-link near "Go To Picker".
   - `docs/protocol.md`: a compatibility row for the `goto` object.
   - Run `just docs-check`.
7. **Gate.**
   - Run `just portable-check`.
   - Best effort: type-check on the Mac per AGENTS.md (scratch rsync copy, `bundle.sh`,
     `just typecheck-tests`).
   - Run `Scripts/macos-ci-check` before landing.
   - After the push, watch the exact-SHA macOS run to green with `/sase_monitor`.

## Phase `mac-browse` — Mac: Go rows in the block-ID card

1. **Decode** the additive `block_id.goto` on `CaptureBlockIDField`: `eligible`,
   `note_path`, `target`, optional `archive_path`, and `rows: [CaptureGotoRow]`. Reuse
   the `mac-card` row type.
2. **Index.** In `BlockIDPickerIndex.newIDPresentation`, when `goto.eligible` holds,
   replace the `new-id` / taken / "Next free" / `used-ids` sections with:
   - **"Go To · <note>":** selectable rows filtered by the typed ID with the existing
     matcher, ordered exact → prefix → the rest, keeping bob's order within each tier.
     An exact match is preselected.
   - **"Archived · <archive_path>":** only when the typed ID is non-empty.
   - **No match:** a single non-selectable info row reading "No ^xyz in bob.md — add
     text after the ID to create a task".

   Each row carries a `taskGotoDestination` built from the row (target, note path, line,
   ref, block).

3. **Accept.** In `acceptPickerRow`, add a branch for rows with a `taskGotoDestination`
   from the block-ID source that calls the shared `dispatchGoto(destination:)`, with the
   `/` stale-draft guard.
   - Map Return, ⌃↩, and click to land, checking `CapturePickerKeyCommandRouter`, where
     ⌃↩ means accept-and-submit-and-open for block-ID rows today.
   - Tab inserts the row's ID over the typed body, leaving the found card.
   - The auto-open on paste of a full address must preselect the exact row, so Return
     lands.
4. **Mint copy.** In mint mode (body present), the taken row's copy gains "— remove the
   text to go to it".
5. **`/` local matcher.** If `TaskLinkPickerIndex` filters `task_goto` / `task_link`
   rows locally with its own match keys, add the `route^id` key so it agrees with
   `cli-browse`.
6. **Fixtures, tests, docs, and gate.**
   - Generate `block-id-goto-{empty,partial,archived,notebook,missing-note}.json`.
   - Index tests cover filtering, ordering, the archived section, the empty state, and
     hidden "Next free".
   - Panel tests cover: Return lands from a browse row; Tab completes into the card;
     body text reverts to the mint composer with no red flash; the `.` toggle swaps link
     ↔ Go To; an old-bob completion keeps today's composer.
   - Update the "Block ID Picker" section of `docs/task-pickers.md`.
   - Apply the same gate as `mac-card`, including watching CI to green.

## Phase `closeout` — end-to-end verification

1. **Real vault, read-only, on athena.** Record `git -C <vault> status --porcelain`
   before and after. Then run `bob capture-parse -f json` for:
   - `@bob^intention-rollovers` (found, `bob.md:82` at research time; re-check the live
     line);
   - an archived ID such as `@bob^use-list-for-reflections`;
   - a typo;
   - a missing note;
   - a notebook ID.

   Also run `bob capture --dry-run` on the bare form (T5), `Fix it @bob^new-id` (mints
   exactly as before), and `bob capture-complete` on `@bob^int` and `/bob^intention`.
   Confirm the vault status is unchanged.

2. **Plugin.** Confirm `bob plugins sync` reports the new build deployed.
3. **macOS CI.** Confirm the CI runs for the landed `mac-card` and `mac-browse` SHAs are
   green.
4. **Manual GUI checklist for Bryan.** The final report lists, for the Mac and Obsidian:
   - paste-and-Return lands with a `<C-o>` jump;
   - `@bob.` browse;
   - miss → did-you-mean;
   - archived badge;
   - a heading landing after inserting lines above it (exercises `plugin-refind`).
5. **Docs.** Run `just docs-check` in bob-cli and in both sibling repos. Fix cross-links
   between bob-cli `docs/capture.md` and the Mac `docs/capture-panel.md`.
6. **Decision record.**

> [!decision] caret_goto_decision_record

Only when this decision is accepted: use `/sase_memory_write` to add the strand
`sase/memory/decisions/caret-address-goes-to.md`, then run `sase memory init`. The
strand records: "a whole-draft bare `@route^id` is a read-only Go To — exact
case-sensitive address, any unique block badged, one archive-pair hop, duplicates fail
closed, never writes; text at either edge still creates; `/` stays vault-wide search."
It cites this plan and the research report, follows the web's record format (Applies to
/ Claim / Why / Cost / Reopens when / Evidence), and lists the rejected alternatives
from the research (execute-in-`bob capture`, tasks-only targets, prefix auto-resolve,
land-first-on-duplicate). When the decision is declined, write nothing.

## Out of scope (follow-ups, not v1)

- A whole-draft `[[foo#^bar]]` paste as a Go To.
- A capability-gated strict plugin lookup with no clamped fallback.
- An Obsidian "Copy Bob address" command.
- Terminal navigation via a future `bob open`.
- Notebook Option-landing refinements.
