---
tier: epic
title: Slash Go To — open any open task in Obsidian from Bob Mac Capture
goal: 'Typing `/` as the whole capture draft opens a Go To picker over exactly the
  `:` task catalog; picking a row hides the panel, clears the draft, and lands Obsidian''s
  caret on that task in its home area/project/inbox note (centred, with a `<C-o>`
  jump) without writing anything to the vault.

  '
decisions:
  memory_goto_record:
    ask: Should the last phase add a `decisions:task-goto-never-writes` record for
      this feature?
    memory:
    - decisions:task-goto-never-writes
    default: false
    answer: false
phases:
- id: cli-goto
  title: bob-cli task_goto grammar, candidates, docs, and digest vectors
  depends_on: []
  size: medium
  description: 'cli-goto: add the whole-draft `/query` claim (parse need `task_goto`,
    capture refusal, `task_goto` completion with `note_path`/`target` candidates over
    the `:` catalog), help, docs/capture.md, CLI tests, and the shared line-digest
    vector fixture.'
- id: plugin-goto-uri
  title: bob-navigation-hotkeys obsidian://bob-goto-task handler
  depends_on: []
  size: medium
  description: 'plugin-goto-uri: register the read-only `bob-goto-task` protocol handler
    that maps the absolute path into the vault, re-finds the task by ref digest /
    block ID, and lands through a neutral helper factored out of the task-move landing;
    tests, README, version, build, sync.'
- id: mac-goto-core
  title: CaptureCore Go To source, goto-mode index, URL builder
  depends_on:
  - cli-goto
  size: medium
  description: 'mac-goto-core: add `.taskGoto` source/need strings, decode `target`,
    give `TaskLinkPickerIndex` a goto mode (ID-less rows selectable, destinations,
    action lines), add `ObsidianOpenURL.taskGotoURL`, picker-view strings, compile-only
    switch arms, fixtures, and CaptureCore tests; green macOS CI.'
- id: mac-goto-panel
  title: Panel wiring, dispatch, keys, README, and closeout
  depends_on:
  - mac-goto-core
  - plugin-goto-uri
  size: medium
  description: 'mac-goto-panel: route `task_goto` needs and completions into the picker,
    accept through a separate dispatch branch (injectable opener, in-flight lock,
    hide and clear on success, picker stays on failure), key mapping, Ctrl-C discard,
    README, panel tests, green macOS CI, and the optional decision record.'
proposed_by: bbugyi200.athena.research.4b.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 09:49:11
status: wip
bead_id: bob-cli-7c
---

- **PROMPT:** [prompts/202610/slash_task_goto.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/slash_task_goto.md)
- **BEAD:** [bob-cli-7c](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/README.md)

# Slash Go To in Bob Mac Capture

## Why

Bryan wants `/` typed as the first key in the capture panel to start a task search that
works exactly like `:`. Picking a task should open its area or project note in Obsidian
with that task focused, instead of inserting a link.

This epic implements the consolidated research report
`research:202610/slash_task_goto_in_obsidian/slash_task_goto_in_obsidian.md`
(2026-10-13). Bryan accepted every requirement adjustment it recommends (R1–R8). Its
open questions are settled:

- **⌘↩** opens the task, the same as Return (R3).
- **R7** stands: a `/` token that contains a second `/` stays prose.
- **Focus** means caret, centring, and a `<C-o>` jump, with no extra line highlight
  (R4).

The facts that shape the design, all verified by the report against the live vault on
athena:

- The `:` catalog lists **464 tasks**, and **307 of them (66%) have no block ID**. If
  `/` copied `:`'s accept, most picks would go through Add block ID, which writes the
  vault just to _look_ at a task. So `/` never writes, and ID-less rows open directly.
- `obsidian://open` has no line parameter, and its `#^block` form only reaches the 34%
  of tasks that have IDs. Only a plugin can place the caret. `bob-navigation-hotkeys`
  already has the "open this note and put the caret on this task" code that `]s` and
  Ctrl+Shift+M use. No bob plugin registers a protocol handler yet.
- `/` is free in the grammar. None of the 3,574 `#task` lines in the vault starts with
  `/`.

Following `decisions:mac-capture-is-a-thin-client`, the work spans three repositories:

- **bob-cli** owns the grammar and the candidates.
- **bob-plugins** owns landing inside Obsidian.
- **bob-mac-capture** decodes the candidates, presents the picker, and dispatches the
  URI.

## The experience (north star)

1. Bryan summons the panel from any app. The empty draft shows the idle agenda.
2. He types `/`. The agenda gives way, just as it does for `:`, and a picker card opens
   with the caption **Go To**. It has the same rows, groups, and ranking as `:`'s Open
   Tasks: queued Pomodoro sections, then In Progress, then Next, then one section per
   note. The status line reads "Pick a task to open — press Tab to browse".
3. He types `sync dev`. AND-term filtering narrows the list, exactly as with `:`.
   - Every row is selectable, including ID-less ones. They show `dev.md:42` with no
     `plus.circle` suggestion.
   - The detail strip reads "↩ Opens dev.md at ^fix-sync" or "↩ Opens dev.md at line
     42".
   - A scheduled date still shows as context. There is no "pulls it forward" warning.
4. He presses Return (or Tab, ⌘↩, or clicks the row).
   - The app dispatches `obsidian://bob-goto-task?…` and Obsidian comes to the front.
   - Once dispatch succeeds, the panel hides and the draft clears. It is not stashed.
   - If the note is already showing in a leaf, that leaf is reused. Otherwise the note
     opens in the active leaf.
   - The caret lands at column 0 of the task line, the line is centred, and the editor
     has focus. `<C-o>` jumps back to where he was.
   - Nothing in the vault changes.
5. If the line moved since the search, the handler finds it again by its `ref` digest or
   block ID. If the task itself changed, the handler lands nearby and shows the Notice
   "Task changed since search — landed near line N". It never silently lands on a
   different task.
6. Escape clears the filter first, then cancels to the chip. Backspace on an empty
   filter deletes the `/`, and the agenda comes back. Shift-↩ does nothing.

## Departures from and refinements of the research

| Topic                     | Research                                                       | This plan                                                                                          | Why                                                                                                                                                                                                                 |
| ------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Accept order              | Hide the panel, clear the draft, then open                     | Lock, **dispatch, then on success** clear and hide. On failure the picker stays open with an error | R3 also says a failed dispatch keeps the picker open, which is impossible once the panel is hidden. The existing ⌘↩-after-capture flow also opens first and hides second (`completeSubmit`)                         |
| Phase order               | Build the plugin first, then bob-cli, then the Mac app         | The plugin and bob-cli phases run **in parallel**. Both Mac phases come after                      | The digest vectors are fixed in this plan ([§5](#5-line-digest-vectors-shared-fixed-here)), so neither repo waits on the other. Bryan's Terminal check of the handler on the Mac still happens before the app ships |
| Digest trimming in JS     | Node `crypto` or `crypto.subtle`                               | Also an explicit **Rust-compatible trailing-whitespace strip**, never `String.prototype.trimEnd`   | JS `trimEnd` strips U+FEFF and keeps U+0085, while Rust `trim_end` does the opposite. The vectors pin both cases                                                                                                    |
| URL encoding              | `URLComponents`                                                | Strict percent-encoding, with `+` sent as `%2B`                                                    | `URLComponents` leaves `+` literal, and query parsers may decode it as a space                                                                                                                                      |
| Ctrl-C on a `/` draft     | "A retained canceled draft must never come back as a `/query`" | Ctrl-C on a Go To draft **discards** it and never pushes it to `CanceledDraftStash`                | This makes the lifecycle check true by construction. A `/query` is trivial to retype                                                                                                                                |
| Mac phases                | One source                                                     | Two phases: `mac-goto-core` (CaptureCore) and `mac-goto-panel` (model)                             | This keeps each worker's slice of the 6.5k-line panel model small, as epic `bob-cli-66` did                                                                                                                         |
| `capture-complete` extras | Not stated                                                     | `task_goto` returns **no `query` field and no `picker` descriptor**, exactly like `task_link`      | The app reuses the `:` lifecycle (it reads the query from bytes and refetches at `replacement.start`)                                                                                                               |
| Re-find source            | The editor buffer if the note is open, else `cachedRead`       | The same, plus the landing re-checks the resolved anchor against the live buffer after opening     | Edits between the read and the caret placement cannot misplace the caret                                                                                                                                            |

## Ground rules (every phase)

- **Repositories.**
  - `cli-goto` works in this project's own bob-cli workspace.
  - `plugin-goto-uri` works in the linked repo **bob-plugins**. Open it with
    `sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`.
  - `mac-goto-core` and `mac-goto-panel` work in the linked repo **bob-mac-capture**.
    Open it with `sase repo open bob-mac-capture -r "<reason>"`.
  - If a linked checkout is missing on the host, use
    `sase repo open gh:bobs-org/<repo> -r "<reason>"`. Work only in the path the command
    prints.
- **Thin client.** bob decides what a `/` draft means and which tasks are listed. The
  app never parses `/` itself. It reacts to `capture-parse` `needs` and to the
  `capture-complete` `context`, as it already does for `:`.
- **Navigation never writes.** No phase may add a vault write on the Go To path. That
  rules out:
  - block-ID minting, `capture-task-id` calls, and `capture` calls;
  - freshness stamps, lane changes, and ledger edits;
  - walk park or advance;
  - Task Card opens and task completion.

  Tests in every repo assert this.

- **The contract below is fixed.** The names to keep are `task_goto`, `TaskGoto`,
  `note_path`, `target`, `bob-goto-task`, the URI parameters, `.taskGoto`, and the
  strings. If a compile constraint forces a rename, add an `INTERFACE CHANGE:` note on
  the phase bead.
- **Additive JSON only.** `capture-parse` and `capture-complete` stay at
  `schema_version` 1. `:`, `+`, `!`, and `&` output stays byte-identical. The app
  decodes every new field with `decodeIfPresent`.
- **Fixtures are synthetic.** Never commit real vault titles or paths.
- **Tests match their neighbours.**
  - bob-cli: in-module `TempDir` tests and the `tests/cli/capture/` harness.
  - bob-plugins: `node:test` scripts in `scripts/` that use
    `navigation-hotkeys-harness.cjs`.
  - bob-mac-capture: the `fake-bob` script, the `#filePath` fixtures, `waitUntil`, and
    closure injection for `targetOpener` and `panelDismisser`.
- **No Swift toolchain for the AppKit targets on agent hosts.**
  - GitHub Actions (`.github/workflows/ci.yml`, `macos-26`) is the only compiler for
    `BobMacCapture`, so write conservatively:
    - Swift 5 language mode;
    - `@MainActor` on AppKit-touching code;
    - long-stable APIs only;
    - swift-format default style (4-space indent, lines ≤ 100 columns);
    - no long `+` or `??` chains, which time out the type checker.
  - athena has Linux Swift (`/usr/bin/swift`). It builds only `CaptureCore`, `RefsCore`,
    and `refs-rank`. Run `swift test --filter CaptureCoreTests` on a scratch copy of the
    tree for fast feedback.
- **Commit and CI loop for bob-mac-capture phases.** This plan explicitly authorizes
  each bob-mac-capture phase to do the following:
  1. Run `git pull --rebase` on `master`.
  2. Commit to `master` with `/sase_git_commit` from the bob-mac-capture checkout. Use
     conventional headers such as `feat(capture): …` and `test(capture): …`.
  3. Find the CI run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
  4. Watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`).
  5. Fix forward until the whole job is green. On a failure, read
     `gh run view <id> --log-failed` and grep for ` error:`.

  A phase is not done until its CI run is green. The bead note records the run URL and
  the SHA.

- **Commit for bob-plugins.** This plan explicitly authorizes `plugin-goto-uri` to
  commit to bob-plugins `master` with `/sase_git_commit`, after `npm run build`,
  `npm test`, and `npm run validate` pass.

## Shared contract (decided; implement exactly)

### 1. The `/query` claim (bob-cli grammar)

Add `task_goto_query_token(item: &CaptureItem<'_>) -> Option<Token<'_>>` in
`src/native/capture_language/tokens.rs`, next to `task_link_query_token`. Its doc
comment states that execution, the editor parse, and completion all call this one
predicate. It matches only when all three conditions hold:

1. The item has exactly one physical line, and that line is exactly one
   whitespace-delimited token.
2. The token starts with `/` and contains **no other `/`** (R7).
3. The token is the **whole draft**: `item.source.trim() == token.text`. That means no
   other item, no `@@` declaration line, and no child line (R1). `CaptureItem.source` is
   the complete raw draft.

| Draft                               | Reading                                                |
| ----------------------------------- | ------------------------------------------------------ |
| `/`                                 | Query `""`: the picker opens on the full list          |
| `/fix`                              | Query `fix`                                            |
| `/sase:deep-fix`                    | Query `sase:deep-fix` (a `:` inside the query is fine) |
| `  /fix\n`                          | Query `fix` (surrounding whitespace is ignored)        |
| `//`, `/Users/x`, `/r/rust`         | Prose: a second `/`                                    |
| `/a b`, `Buy /x`                    | Prose: several tokens                                  |
| `/x` with a child line              | Prose: several lines                                   |
| `/x`, a blank line, then `Buy milk` | Prose: two items, both captured as usual               |
| `@@work` on one line, then `/x`     | Prose: `/x` routes to `work` as usual                  |

The `:` claim is per item; the `/` claim is per draft. Their first bytes differ, so the
two claims never overlap.

### 2. `bob capture-parse`

- A claimed draft reports mode `incomplete` and `needs: ["task_goto"]` (top-level and on
  the item). It has one `interactive_placeholder` span over the whole token, sigil
  included, and no diagnostics. This mirrors `parse_editor_task_link_item`. Add
  `parse_editor_task_goto_item` and dispatch it before the `:` item at the existing
  dispatch site (`editor_parse.rs`, near the `parse_editor_task_link_item` call).
- Add `Need::TaskGoto` with label `task_goto` directly after `TaskLink`. The documented
  `needs` order becomes `…, active_task, task_link, task_goto, task_parent, …`.

### 3. `bob capture` refusal

A claimed draft is never executable. In `src/native/capture_language/item.rs`, refuse it
**before any other item parser**: before the `&` dependency scan, and so before the `:`
refusal. Forced flags such as `-r` and `-s` change nothing. Use the usual
`capture item 1 starting on line L: ` prefix. The teaching error (T3) reads:

`` `/dee` opens the Go To picker; pick a task to open it in Obsidian — it never captures ``

The quoted token is the exact typed token, so a bare `/` reads `` `/` opens … ``.
`--dry-run` gives the same error. No vault file changes.

### 4. `bob capture-complete`: context `task_goto`

- **Field.** `task_goto_completion_field_at(item, cursor)` mirrors
  `task_link_completion_field_at`:
  - The cursor may sit anywhere in `[token.start, token.end]`.
  - The query is the token text between the sigil and the cursor. It is empty at
    `token.start` or just after the sigil.
  - The replacement covers the whole token, sigil included. Refetching at
    `replacement.start` therefore returns the unfiltered list, and Backspace on an empty
    filter can remove the `/`.
  - Dispatch it in `completion_field_at` on the leading line, before the `!` and `:`
    checks.
- **Context.** Add `CompletionContext::TaskGoto`, serialized as `task_goto`. Add it to
  every exhaustive match: the engine, `render.rs`, `shell.rs`, and the `unreachable!`
  arms.
- **Candidates.** `task_goto_candidates(bob_dir, query)` uses
  `capture_link_tasks::discover` plus `rank`, exactly like `task_link_candidates`. Rows,
  order, ranking, and warnings are therefore identical to `:` for the same query. Like
  `task_link`, the response has no `query` field and no `picker` descriptor.
- **Candidate shape (`TaskGotoCandidate`).** Keys appear in this order:

  | Key                                                                                        | Value                                                                                                              |
  | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
  | `replacement`                                                                              | Always `""`. Clients never insert it; it is kept so the shared decoder still works                                 |
  | `ref`                                                                                      | `LINE:digest`, as for `:`                                                                                          |
  | `route`                                                                                    | Display route, as for `:`                                                                                          |
  | `note_kind`                                                                                | As for `:`                                                                                                         |
  | `note_path`                                                                                | **New.** The exact vault-relative path with extension, as `!` and `&` already send                                 |
  | `target`                                                                                   | **New.** The absolute path, `bob_dir.join(note_path)` displayed, built the same way capture results build `target` |
  | `block_id`                                                                                 | `null` when the task has no ID                                                                                     |
  | `status_symbol`, `status_name`, `status_type`, `text`, `section`, `depth`, `line`, `group` | As for `:`                                                                                                         |
  | `scheduled`                                                                                | Shown as context only                                                                                              |
  | `pomodoro`                                                                                 | As for `:`                                                                                                         |
  | `task_kind`                                                                                | Omitted when absent                                                                                                |

  There is **no** `requires_block_id`, `block_id_suggestions`, or `pulls_forward`.

  `note_path` comes from `target.relative_path`, which `discover_at` already holds.
  Carry it on `LinkTask` without changing any existing JSON. If `LinkTask` is ever
  serialized, use `#[serde(skip)]`.

- **Text format.** One line per row: `[status] text  · note_path:line`. Mirror
  `task_link_line`'s style.
- **Shell format.** No rows, because nothing is insertable.

### 5. Line digest vectors (shared, fixed here)

bob's `ref` digest is `line_digest` in `src/native/note_tasks.rs`: the first 8 lowercase
hex characters of SHA-256 over the UTF-8 line after Rust `str::trim_end()`. That call
strips trailing Unicode `White_Space`:

- U+0009–U+000D
- U+0020
- U+0085
- U+00A0
- U+1680
- U+2000–U+200A
- U+2028, U+2029
- U+202F
- U+205F
- U+3000

It does **not** strip U+FEFF. JavaScript must strip exactly this set, with
`/[\t\n\v\f\r \u0085   -     　]+$/u`. It must never call `trimEnd()`.

`cli-goto` commits these vectors as `tests/fixtures/line_digest/vectors.json`, in the
`freshness_keeps` style:
`{"schema": 1, "doc": "…", "vectors": [{"id", "line", "digest", "runners": ["rust", "js"]}]}`.
A Rust test reads the file through `CARGO_MANIFEST_DIR` and asserts
`line_digest(line) == digest` for every vector. `plugin-goto-uri` mirrors the same
vectors inline, with a comment that cites the bob-cli fixture path. The `line` values
below are JSON string literals, so write them byte for byte:

| id                    | line (JSON)                                                                      | digest     |
| --------------------- | -------------------------------------------------------------------------------- | ---------- |
| `plain`               | `"- [ ] #task Same"`                                                             | `35a733bd` |
| `trailing-spaces`     | `"- [ ] #task Same   "`                                                          | `35a733bd` |
| `trailing-tab-cr`     | `"- [ ] #task Same\t\r"`                                                         | `35a733bd` |
| `trailing-nbsp`       | `"- [ ] #task Same "`                                                            | `35a733bd` |
| `trailing-nel`        | `"- [ ] #task Same\u0085"`                                                       | `35a733bd` |
| `trailing-bom-kept`   | `"- [ ] #task Same﻿"`                                                            | `080a2f04` |
| `leading-indent-kept` | `"\t- [ ] #task Indented ^blk"`                                                  | `95e4c084` |
| `leading-spaces-kept` | `"  - [ ] #task Same"`                                                           | `7c6b26a7` |
| `unicode`             | `"- [/] #task Café ☕ — résumé 日本語 ^cafe-1"`                                  | `c4c8a067` |
| `inline-fields`       | `"- [*] #task Ship it [created:: 2026-10-01] [scheduled:: 2026-10-20] ^ship-it"` | `c2201ec4` |

### 6. URI v1: `obsidian://bob-goto-task`

`obsidian://bob-goto-task?path=<absolute>&line=<1-based>&ref=<LINE:digest>[&block=<id>]`

| Param        | Rule                                                                                   |
| ------------ | -------------------------------------------------------------------------------------- |
| `path`       | Required. The absolute filesystem path of the note: bob's `target`                     |
| `line`       | Required. A positive integer, 1-based: bob's `line`                                    |
| `ref`        | Sent by the app. Must match `^[1-9][0-9]*:[0-9a-f]{8}$`. Optional for hand-typed links |
| `block`      | Optional. Must match `^[A-Za-z0-9-]+$`                                                 |
| other params | Ignored. This reserves room for a later `pane=tab`                                     |

- **Encoding.** Every value is percent-encoded. Only RFC 3986 unreserved characters and
  `/` stay literal, so a space becomes `%20`, `+` becomes `%2B`, and `&`, `=`, `#`, and
  `%` are encoded too.
- **Handler behaviour** (bob-navigation-hotkeys, read-only):
  1. **Registration.** `registerObsidianProtocolHandler("bob-goto-task", …)` runs
     synchronously in `onload`. Guard it with
     `typeof this.registerObsidianProtocolHandler === "function"`, because test stubs
     lack the method.
  2. **Wait.** Await `workspace.onLayoutReady`.
  3. **Validate.** A missing or malformed `path` or `line`, or a malformed `ref` or
     `block`, gets the Notice "Bob Go To: invalid task link" and does nothing else.
  4. **Map the path.** Strip the normalized `getVaultBasePath()` plus `/`. If the prefix
     does not match, retry once with `realpath` of both sides when Node `fs` is
     available.
     - Still outside the vault: Notice "Bob Go To: <path> is outside this vault".
     - Missing or non-markdown file: Notice "Bob Go To: <vault path> not found".
  5. **Read.** Use the open editor buffer for that path (`getOpenMarkdownEditorForPath`)
     if there is one, else `vault.cachedRead`.
  6. **Re-find** the 0-based line, in this order:
     1. the planned line (`line − 1`), when it is in range and its digest equals the
        `ref` digest;
     2. the task line carrying `^block`, via `findTaskLineByBlockId`, when `block` is
        given;
     3. the **only** task line whose digest equals the `ref` digest;
     4. otherwise the clamped planned line, plus the Notice "Task changed since search —
        landed near line N" (N is 1-based).

     Without a `ref`, steps 1 and 3 are skipped. A clamped landing without a `ref` shows
     no Notice. When the digest matches two or more lines, step 3 does not apply.

  7. **Land** through the neutral landing helper (see `plugin-goto-uri`):
     - reuse the leaf that already shows the note, else use the active leaf;
     - switch a Reading-view leaf to editing first;
     - put the caret at column 0, retrying frames until the editor renders;
     - centre the line and focus the editor;
     - record a Vim jump from the cursor that was active before the jump (no jump when
       no markdown editor was active).

     If the note cannot be opened: Notice "Bob Go To: could not open <name>".

  8. **Never** write the vault or any plugin state that has meaning, such as freshness,
     lanes, ledger, walk park or advance, Task Card, or completion. Opening a different
     note still runs `trackOpenedFile`, which clears a `]s` landing exactly as manual
     navigation does today.

### 7. App behaviour (bob-mac-capture)

| Gesture while the Go To picker is open       | Effect                                                                          |
| -------------------------------------------- | ------------------------------------------------------------------------------- |
| Return, Tab, ⌘↩, click                       | Go to the selected row                                                          |
| Shift-↩, Option-↩                            | Consumed; nothing happens                                                       |
| Escape                                       | Clears a non-empty filter; otherwise cancels to the chip (the draft keeps `/…`) |
| Backspace on an empty filter                 | Removes the whole `/…` token; the idle agenda returns                           |
| Any picker key while a dispatch is in flight | Swallowed                                                                       |

**Go to** works as follows:

1. Run the stale-draft guard, as `:` does. On failure, close the picker after the stale
   draft and announce it.
2. Build the URL from the row's destination. If there is no destination or no URL, set
   the error status "bob sent no location for this task — update bob" and keep the
   picker open.
3. Set the in-flight lock and the status "Opening in Obsidian…".
4. Call the injectable opener. In production it is
   `NSWorkspace.shared.open(url, configuration:completionHandler:)` with
   `activates = true`.
5. **On success:** discard the draft (no stash; reuse `discardDraft()`), clear the
   picker, and call `panelDismisser()`. **On failure:** release the lock, keep the
   picker open, and set the error status "Couldn't open Obsidian".
6. Never spawn `capture`, `capture-task-id`, or any other writing `bob` command.

## Phase `cli-goto`: bob-cli grammar, candidates, docs, digest vectors

Implement contract §§1–5 in this repository.

- **Grammar.** Add `task_goto_query_token` (§1), `parse_editor_task_goto_item` plus
  `Need::TaskGoto` (§2), the T3 refusal and its error helper beside the existing
  `task_link_picker_error` (§3), and `task_goto_completion_field_at` plus
  `CompletionContext::TaskGoto` (§4).
  - Search for every `TaskLink` / `task_link` match arm and list in `capture_language/`,
    `capture_complete/`, and `capture_parse.rs`, and decide for each whether `TaskGoto`
    belongs there.
  - Arms that mean "picker query, not a capture" need it.
  - Arms specific to linking, such as `@@` inheritance (irrelevant under the whole-draft
    rule), pull-forward, and `capture-task-id`, do not.
- **Candidates.** Add `note_path` to `LinkTask`, `TaskGotoCandidate`,
  `Candidates::TaskGoto`, `task_goto_candidates`, the text renderer line, and the empty
  shell arm (§4).
- **Help** (read `sase memory read cli_rules.md` first; no new options or subcommands).
  - `capture-parse`: add a sentence after the `:` sentence: "A whole draft that is one
    `/` token (with no second `/`) reports 'incomplete' needing `task_goto` with one
    `interactive_placeholder` span; `bob capture` refuses it." Add the example
    `bob capture-parse -f json -- '/fix'` and add `task_goto` to the **Needs** list.
  - `capture-complete`: add a sentence after the `:` sentence: "A whole-draft `/` token
    completes `task_goto`: the same rows and order as `task_link`, with `note_path` and
    absolute `target` for opening the task; the replacement covers the whole token and
    is never inserted." Add the example `bob capture-complete -c 1 -f json -- '/'` and
    add `task_goto` to the **Contexts** list.
  - Update any help golden this changes.
- **Docs (`docs/capture.md`).**
  - Add a `/<query>` row to "Grammar at a glance" and a Contents entry.
  - Add a new subsection, `### Going to a task with '/'`, directly after "Picking any
    open task with ':'". It covers:
    - the claim and the §1 table;
    - the T3 refusal text;
    - the catalog, which is the `:` catalog including the inbox, with a pointer to that
      section;
    - the candidate fields from §4 and which fields are absent;
    - "Go To never writes";
    - a pointer to bob-navigation-hotkeys' `obsidian://bob-goto-task` URI, whose
      contract lives in the bob-plugins README.
  - Insert `task_goto` into the `capture-parse` needs order and the `capture-complete`
    contexts list.
- **Digest vectors.** Commit `tests/fixtures/line_digest/vectors.json` and its Rust test
  (§5). If any vector fails, stop and record an `INTERFACE CHANGE:` note. Never edit a
  digest to make a test pass.
- **Tests.**
  - **Unit** (`capture_language/tests/`): predicate and completion-field cases for every
    row of the §1 table, plus a cursor at `token.start`, just after the sigil, and
    mid-token.
  - **CLI** (a new `tests/cli/capture/task_goto.rs`, registered in `mod.rs`):
    - `capture-parse` JSON for `/` and `/fix`: mode, needs, span, no diagnostics;
    - every prose row of §1 keeps its current parse;
    - `capture` and `capture --dry-run` refuse `/`, `/fix`, `/a&b`, and `/fix` with
      `-r inbox` using T3;
    - vault bytes are identical after every refusal;
    - `capture-complete -f json` for `/` and `/sas` returns context `task_goto` and the
      replacement range;
    - the candidates carry the exact §4 key set, with `note_path`/`target` for an area
      note, a project note, and the inbox, and without the three link-only keys;
    - the ID-less rows are listed;
    - the `(route, ref)` sequence equals `:`'s for the same query on the same fixture
      vault;
    - the text format line shape;
    - the shell format yields no rows.
  - Existing `:`, `+`, `!`, and `&` tests stay green unchanged.
- **Gate.** Run `just all` (or the `fmt`, `lint`, and `test` recipes). Then build with
  `cargo build --release` and probe the live vault read-only with `target/release/bob`.
  Do not install it.
  - The `capture-complete -c 1 -f json -- '/'` row count equals that of `-- ':'`.
  - `capture --dry-run -- '/x'` exits with T3.

  Record both results on the bead.

## Phase `plugin-goto-uri`: obsidian://bob-goto-task handler

Implement contract §§5–6 in bob-plugins `plugins/bob-navigation-hotkeys`. Edit only
`src/` fragments, run `npm run build`, and never hand-edit `main.js`. Every hand-edited
fragment must stay at or below 1,000 lines.

- **New fragment.** Use the next free number between 670 and 690 (for example
  `675-plugin-goto-task-uri.js`). Add it to `src/fragments.json` and add its mixin class
  to the installer list in `690-install-plugin-mixins.js`. The installer throws on
  duplicate method names, and fragments share one scope, so give top-level names a
  `goto`/`GoTo` prefix.
  - Put the pure helpers here as top-level functions:
    - `parseGoToTaskParams`;
    - `vaultPathFromAbsolute`;
    - `bobLineDigest` (Node `crypto` through `requireOptionalNodeModule("crypto")`,
      falling back to `globalThis.crypto.subtle`, so it is async);
    - `refindGoToTaskLine(lines, {line, ref, block})`, which returns
      `{line, source: "planned" | "block-id" | "digest" | "clamped"}`.
  - Put the handler method here too.
  - Export the pure helpers for tests the way `700-exports.js` exports existing ones.
- **Registration.** In `490-plugin-lifecycle.js` `onload`, call the guarded registration
  synchronously (§6.1). The handler awaits layout readiness inside itself.
- **Neutral landing helper.** Factor the body of `focusTaskMoveDestination`
  (`650-plugin-move-commit.js`) into a neutral method, for example
  `focusTaskAnchor(file, anchor, {vimJumpContext, failureNotice, ensureEditing})`.
  - It handles the open with leaf reuse, the frame-retried anchor landing, centring, and
    jump recording.
  - `focusTaskMoveDestination(file, anchor, vimJumpContext = null)` keeps its name and
    signature, because tests stub it. It delegates with the move's existing "Moved
    tasks, but could not open …" notice and `ensureEditing: false`, so move behaviour
    and every move test stay byte-identical.
  - The landing state (`pendingTaskMoveJump*`, `taskMoveLandingSeq`, and the retry
    count) keeps its current names, because the move, move-history, and review-advance
    tests reset those fields directly. Go To shares that single pending-jump slot, so a
    newer landing supersedes an older one. Move tests must not change.
  - `ensureEditing: true`, which only Go To passes, switches a Reading-view leaf to
    `mode: "source"` with `setViewState`. That keeps the leaf's live-preview or source
    preference. The only existing reference is `getLeafViewState`; nothing does this
    today.
  - Go To passes the anchor
    `{line: resolvedLine, text: liveLineText, blockId: block ?? null}`, so the move
    resolver confirms the line as `planned` at landing time. It passes a jump context
    created with `createVimJumpContextWithOrigin(currentCursor)`, or null when no
    markdown editor is active.
- **Tests.** Add `scripts/test-navigation-goto-task-uri.cjs` to the root `package.json`
  `test` list. It covers:
  - the §5 vectors mirrored inline (cite
    `bob-cli tests/fixtures/line_digest/vectors.json`) against `bobLineDigest`;
  - parameter validation (each malformed case gives the invalid-link Notice);
  - path mapping: inside the vault, outside it, a trailing slash on the base, and a
    missing file;
  - each re-find path: planned; block-id after an edit above the task; unique digest
    after a shift with no block; a duplicate digest falling to clamped with the Notice;
    out of range clamped; no `ref`;
  - the open buffer is preferred over `cachedRead`;
  - the leaf is reused when the note is already open;
  - Reading view is switched to editing;
  - a jump is recorded from the prior cursor, and none without one;
  - **no writes**: `vault.modify`, `vault.process`, `editor.replaceRange`, and
    `editor.setLine` are never called, and walk park and advance are never called;
  - `onload` with and without a `registerObsidianProtocolHandler` stub, registering
    `"bob-goto-task"`.

  All existing tests stay green, including the move, move-history, jump-history, and
  review-advance suites.

- **Docs and version.**
  - Bump the bob-navigation-hotkeys `manifest.json` minor version (2.31.0 → 2.32.0, or
    the next minor if HEAD moved) and its README table cell.
  - Add a root README section, `## Go to task URI (obsidian://bob-goto-task)`, before
    `## Layout`. It documents the URI v1 table, encoding, re-find order, Notices, and
    the no-writes rule, plus a Terminal example:
    `open 'obsidian://bob-goto-task?path=%2FUsers%2Fme%2Fbob%2Fdev.md&line=42&ref=42:35a733bd'`.
  - Mention the URI in the plugin's Plugins-table row.
- **Gate and deploy.**
  1. Run `npm run build`, `npm run build:check`, `npm test`, and `npm run validate`.
  2. Commit.
  3. Run `bob plugins sync --repo <bob-plugins checkout> -p bob-navigation-hotkeys` on
     this host. Never pass `-F`. Record its output on the bead.

  Deploying to the Mac vault is Bryan's step; see [Rollout](#rollout-and-compatibility).

## Phase `mac-goto-core`: CaptureCore source, goto-mode index, URL builder

Work in bob-mac-capture. This phase makes the Go To source fully presentable but leaves
it **inert**: `task_goto` is not yet in `completionNeeds`, so no picker can open.

- **`CapturePickerSource.taskGoto`** (`CapturePickerPresentation.swift`). Fill every
  switch with these values:

  | Property                     | Value                                            |
  | ---------------------------- | ------------------------------------------------ |
  | Trigger byte                 | `47`                                             |
  | Placeholder                  | "Search tasks to open in Obsidian"               |
  | Filter accessibility label   | "Search tasks to open"                           |
  | Scope symbol                 | `"/"`                                            |
  | Caption                      | "Go To"                                          |
  | Chip label                   | "Go to a task"                                   |
  | Chip icon                    | `arrow.up.forward.square`                        |
  | Chip help                    | "Reopen the Go To picker for the / draft (Tab)." |
  | Appeared-announcement prefix | "Go To"                                          |
  | Key hints                    | ↑↓ Move · ↩ Go To · esc Clear / Cancel           |
  | Used-defaults key            | `org.bobs.bob-mac-capture.task-goto-picker-used` |
  | `locatorMarker`              | `":"`, so rows read like `:` rows                |

  Write card accessibility label and hint wording parallel to `.taskLink`'s.

- **`CapturePickerNeed.taskGoto`.** The status reads "Pick a task to open — press Tab to
  browse". Update the precedence doc comment so `taskGoto` comes right after `taskLink`.
- **Decode.** Add `target: String?` to `CaptureCompletionCandidate`: a property, an init
  parameter, `decodeIfPresent`, and the coding key `"target"`. `note_path` already
  decodes.
- **Goto mode for `TaskLinkPickerIndex`.** Parameterise the index; do not copy it.
  - Add `public enum TaskLinkPickerMode { case link, goto }` and
    `init(candidates:mode: = .link)`. The sigil follows the mode (`":"` or `"/"`), both
    in `FuzzyQuery(leadingSigil:)` and in the `route:id` search field.
  - In `.goto` mode:
    - Every row is selectable. `pendingBlockID` and `insertion` are nil, and there is no
      suggestion.
    - An ID-less row's locator shows `<note name>:<line>` (for example `dev.md:42`).
    - `pullsForward` is false. `scheduledText` is kept.
    - Each row carries a new `public struct TaskGotoDestination`
      `{ target, line, taskRef, blockID, noteName }`. `noteName` is the last path
      component of `note_path`, falling back to `"<route>.md"`. A row whose candidate
      lacks `target`, `line`, or `ref` has a nil destination.
    - `actionLine` reads "↩ Opens dev.md at ^fix-sync" or "↩ Opens dev.md at line 42".
  - `.link` output must not change; every existing test stays green unchanged.
  - Pick whichever is cleaner for `CapturePickerIndex` (`CapturePickerState.swift`):
    reuse `.taskLink(TaskLinkPickerIndex)`, since the index carries its mode, or add
    `.taskGoto`.
- **URL builder.** Add
  `ObsidianOpenURL.taskGotoURL(target:line:taskRef:blockID:) -> URL?` (§6). It returns
  nil for an empty target or `line < 1`, omits `block` when nil, and encodes strictly
  through `percentEncodedQueryItems`.
- **Picker view strings** (`CapturePickerView.swift`):
  - `teachingLine(for: .taskGoto)`: "Pick a task to open it in Obsidian — nothing is
    written".
  - The detail strip (`insertionText`) uses the goto action line for `.taskGoto` rows.
  - ID-less goto rows draw no `plus.circle` or suggestion.
  - The pull-forward line stays `.taskLink`-only.
- **Compile-only arms.** Give `.taskGoto` the `.taskLink` behaviour in the panel model's
  exhaustive switches: `pickerMarkerHighlightRange`, the two `openPickerFromChip`
  switches, and the `cancelTaskIDPrompt` rebuild. `mac-goto-panel` finishes the
  behaviour. Leave a `// mac-goto-panel:` comment at each arm.
- **Fixtures** (`Tests/Fixtures/`).
  - `task-goto-parse-slash.json`: the `capture-parse` output for `/`.
  - `task-goto-complete-full.json`: derived from `task-link-complete-full.json` with the
    same rows in the same order. Change `context` to `task_goto`, set every
    `replacement` to `""`, drop the three link-only keys, and add `note_path` (for
    example `sase.md`) and a synthetic `target` (`/tmp/bob-fixture/<note_path>`).
  - Check the key set and order against a real run of the `cli-goto` build of
    `bob capture-complete -c 1 -f json -- '/'` on a synthetic temp vault.
- **Tests** (`Tests/CaptureCoreTests/`):
  - `.taskGoto` source strings, need status, and key hints;
  - `FuzzyQuery` with the `/` sigil;
  - a goto-mode presentation from the fixture: grouping and order equal to link mode for
    the same rows; ID-less rows selectable with a destination and no suggestion; both
    action-line spellings; scheduled text kept; pull-forward absent;
  - a candidate without `target` gets a nil destination;
  - `taskGotoURL` encoding: a space, `+`, `&`, `#`, `%`, a Unicode path, and `block`
    present and absent, with exact expected strings;
  - decoding with and without `target`.

  Run `swift test --filter CaptureCoreTests` on a scratch copy on athena, then follow
  the CI loop to green.

## Phase `mac-goto-panel`: panel wiring, dispatch, keys, README, closeout

Work in bob-mac-capture and implement contract §7.

- **Needs and routing** (`CapturePanelModel.swift`).
  - Add `"task_goto"` to `completionNeeds` (in `shouldRequestCompletion`) and to
    `pickerNeed(in:)`, right after `task_link`.
  - Route a `task_goto` context in `handleCompletionResponse` to a Go To handler with
    the same lifecycle as `handleTaskLinkCompletion`:
    - byte-range validation;
    - the `.selection` chip and auto-open suppression;
    - the query seeded from the bytes `r.start+1..<caret`;
    - the refetch at `r.start` when `cursor > r.start + 1`, which requires
      `context == "task_goto"`.

    Prefer parameterising the existing handler by source over copying it. Build the
    index in `.goto` mode.

- **Replace the compile-only arms** from `mac-goto-core` with real behaviour. Audit
  every `== .taskLink` comparison the earlier exploration found:
  - the ID-less row's Add block ID prompt: must **not** fire for goto;
  - the "Inserted …" announcement: none for goto;
  - Shift-↩: consumed;
  - Backspace sigil removal: `/` sits at `r.start`, like `:`;
  - the chip query and the refetch rule: mirror `:`;
  - `pickerSourceIsTaskLink`: stays false for goto.
- **Accept.** At the top of `acceptPickerRow(id:submitAfterInsert:)`, branch for
  `.taskGoto` into `acceptTaskGotoRow(id:)`, which implements §7 steps 1–6. It ignores
  `submitAfterInsert`, so ⌘↩ equals Return.
  - Add an injectable
    `var taskGotoOpener: (URL, @escaping @MainActor (Bool) -> Void) -> Void`, with the
    production `NSWorkspace` implementation from §7.
  - Add a `taskGotoDispatchInFlight` flag. While it is set, the router or model swallows
    picker commands.
  - Leave the existing `targetOpener` unchanged.
- **Ctrl-C.** In `stashDraftAndClose()`, when the open picker or chip source is
  `.taskGoto`, or the latest analysis reported the `task_goto` need, discard instead of
  pushing to `CanceledDraftStash`. The rest of the flow is unchanged.
- **Agenda.** After a successful Go To, the next show starts from a blank draft, so the
  agenda shows. After Backspace removes `/`, the existing blank-draft path restores the
  agenda. Confirm both with tests, since no new agenda code should be needed.
- **Tests** (`Tests/BobMacCaptureTests/`).
  - Add `fake-bob` branches: `capture-parse` for `/` and `/dee` (needs `task_goto`), and
    `capture-complete` for `/` and `/dee` (the `task-goto-*` fixtures).
  - Mirror the `:` tests:
    - typing `/` opens the Go To picker with one `capture-complete`;
    - the chip reopens without a refetch;
    - `/dee` presents a partial snapshot after a refetch at zero.
  - Accept by Return, Tab, click, and ⌘↩:
    - the opener receives the exact expected URL;
    - the panel is dismissed once;
    - the draft is empty and nothing was stashed;
    - the recorded argv contains no `capture` or `capture-task-id`.
  - An ID-less row opens directly, with no ID prompt.
  - Shift-↩ does nothing.
  - A dispatch failure keeps the picker open with the error, does not dismiss, and keeps
    the draft.
  - A double Return during an in-flight dispatch opens once.
  - A stale-draft accept closes without opening.
  - Escape clears the filter, then chips. Backspace on an empty filter removes `/` and
    the agenda is visible again.
  - Ctrl-C on a `/dee` draft discards it.
  - Router mappings for `.taskGoto`.
- **README.**
  - Add a `### Go To Picker` section after "Task Link Picker". It covers the experience,
    the keys, never writing, and the dependency on the plugin handler.
  - Add rows to the keyboard table.
  - Add requirement bullets:
    - a `bob` build with the `task_goto` completion context (`note_path` and `target`
      candidate fields);
    - bob-plugins **bob-navigation-hotkeys ≥ 2.32.0**, or the version `plugin-goto-uri`
      shipped, deployed to the vault with `bob plugins sync`, which provides
      `obsidian://bob-goto-task`.
- **Gate.** Follow the CI loop to green. Record the run URL and SHA on the bead.
- **Closeout** (memory decision below). Before finishing, add a bead note that copies
  the [Mac acceptance pass](#mac-acceptance-pass-bryan-after-the-epic) checklist for
  Bryan.

> [!decision] memory_goto_record Use `/sase_memory_write` to add the strand
> `sase/memory/decisions/task-goto-never-writes.md`, modelled on
> `task-move-never-advances-the-walk.md`, then run `sase memory init`.
>
> - **Title:** "Task Go To Never Writes".
> - **Applies to:** bob-cli, bob-plugins (bob-navigation-hotkeys), and Bob Mac Capture.
> - **Claim:**
>   - A whole-draft `/query` opens bob's `:` catalog.
>   - Accepting opens the task's home note through `obsidian://bob-goto-task` and lands
>     the caret, centred, with a `<C-o>` jump.
>   - ID-less rows open directly.
>   - Nothing is minted, stamped, linked, walked, or captured.
> - **Why**, with the rejected alternatives from the research's disagreement table:
>   - an app-side keyboard mode;
>   - keeping the Add block ID prompt;
>   - the built-in `#^block` URI;
>   - Advanced URI;
>   - per-item `/`.
> - **Cost.**
> - **Reopens when.**
> - **Evidence:** the research ref, this plan, and the phase commits.
>
> If this decision is off, record only a `PROPOSED FOLLOW-UP:` note when the decisions
> block says no human reviewed the plan.

## Rollout and compatibility

Every step is additive:

| Combination          | Behaviour                                                                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Old bob + new app    | `/fix` stays an ordinary task (today's behaviour). No picker opens                                                                       |
| New bob + old app    | `task_goto` is not in the app's `completionNeeds`, so no picker opens. The preview shows T3 and submit refuses, so no junk task is filed |
| New app + old plugin | The URI does nothing. The README requirement and the order below cover this                                                              |

Bryan's steps on the Mac, after the phases land:

1. From bob-cli, run `just install-all`. It pulls and installs bob-cli, bob-plugins (via
   `bob plugins sync`), and bob-mac-capture, and restarts Obsidian if plugins changed.
   Alternatively, run `bob plugins sync -p bob-navigation-hotkeys`, then install `bob`
   and the app separately.
2. Before using the panel, test the handler from Terminal with
   `open 'obsidian://bob-goto-task?…'`. Take the `target`, `line`, and `ref` from
   `bob capture-complete -c 1 -f json -- '/'`.
3. Run the acceptance pass below.

## Mac acceptance pass (Bryan, after the epic)

- **Cold start.** Check Obsidian both quit and running.
  - If the URI is dropped when the URI itself launches Obsidian, file a follow-up for
    the app: launch `md.obsidian`, wait for it to finish launching plus a short settle,
    then dispatch.
  - Never add a plain `open?path=` "insurance" URI. It replaces the active tab and
    defeats leaf reuse.
- **Landing.**
  - the note already open in a split;
  - the task under a folded heading or list;
  - a Reading-view leaf;
  - the caret on the row in Vim normal mode, and `<C-o>` returning;
  - Unicode and spaces in paths.
- **Moved lines.**
  - Edit above the task between the search and the accept: it lands correctly with no
    Notice.
  - Edit the task line itself: the Notice appears.
- **Lifecycle.**
  - Escape during the fetch, a double Return, a dispatch failure, and a reopened panel
    never launch Obsidian late or capture anything.
  - A canceled `/` draft never comes back from the stash.
- **No writes.** Accept one identified row and one ID-less row. The note and day files
  stay byte-identical (`git -C ~/bob status` is clean).

## Out of scope (follow-ups, not v1)

- An ⌥↩ "Go To" action on `:`, `!`, `&`, and `+` rows that reuses the same URI. Each of
  those catalogs would add `target`.
- ⌘↩ after a capture landing on the created line once bob reports that line.
- An Obsidian-side "Bob: Go to task" command that shares the re-find logic.
- `pane=tab` (open in a new tab) for the URI.
- A plugin-version check in the app, if an old plugin ever bites in practice.
