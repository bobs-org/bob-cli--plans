---
tier: epic
title: Capture lists under a lone @file header
goal: 'A capture item whose first line is only `@file` (or `@file#section`) files
  each bullet below it as its own new task (or section note) in that file. A trailing
  ` :` or ` ^` on a task bullet, or on one of its nested bullets, opens a block-ID
  prompt for that task with a default ID Bryan is likely to accept, and ` :<id>` also
  links the task into today''s daily note. Execution, `capture-parse`, `capture-complete`,
  the dry-run preview, and Bob Mac Capture always agree.

  '
phases:
- id: list-capture
  title: Lists in bob capture and capture-parse
  depends_on: []
  size: medium
  description: 'list-capture: split list headers into one capture item per bullet
    (zero-copy slices plus a list context), give list bullets the task-bullet grammar
    (` :id`, ` ^id`, per-task markers, refusals) in both the execution and editor
    parsers, wire planning, add the `list` JSON object and human summary, bump grammar
    to 6, and document the form.

    '
- id: id-suggester
  title: Default block-ID suggestions tuned on the vault
  depends_on: []
  size: medium
  description: 'id-suggester: add an ignored calibration harness over a real vault,
    then retune the shared block-ID suggester (filler lead-ins, content words, length)
    so its first suggestion is closer to the IDs Bryan writes himself, with measured
    before/after numbers.

    '
- id: editor-assists
  title: Block-ID prompt, toggle, and similar tasks for lists
  depends_on:
  - list-capture
  - id-suggester
  size: medium
  description: 'editor-assists: open the `project_task_block_id` prompt on list bullets
    and their nested bullets, return the owner-line `splice`, add the `:`/`^` toggle
    to capture-rewrite, name Bob''s suggested ID in unfinished-ID refusals, make capture-related
    anchors list-aware, and write the authorized memory changes.

    '
- id: mac-lists
  title: Bob Mac Capture presentation
  depends_on:
  - editor-assists
  size: medium
  description: 'mac-lists: in bob-mac-capture, apply Bob''s block-ID splice on accept,
    caption the owner task, group list results in the preview, open the first bullet
    after a lone header, and regenerate fixtures from a real bob.

    '
decisions:
  header_starts_item:
    ask: Should a lone `@file` line start a new list even without a blank line before
      it?
    default: true
    why: Several files in one draft without blank lines; that line is a usage error
      today
    answer: true
  project_note_parity:
    ask: Should project-note task bullets get the same ID grammar and prompt as list
      bullets?
    default: true
    why: One task-bullet grammar, and it fixes the silent `s:2 :id` trap in project
      notes
    answer: true
  memory_bulk_capture_glossary:
    ask: Update the Bulk Capture glossary entry to describe lists?
    memory:
    - glossary:bulk-capture
    corrects: Authored child bullets never start an item.
    default: true
    answer: true
  memory_list_decision_record:
    ask: Add a decisions record for capture lists under a lone @file header?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.c1
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-01 23:41:31
status: done
bead_id: bob-cli-fn
---

- **PROMPT:** [prompts/202611/capture_route_lists.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_route_lists.md)
- **BEAD:** [bob-cli-fn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fn/README.md)

# Plan: Capture lists under a lone `@file` header

## Why

Bryan wants to add several tasks (or several notes) to one area or project note in one
capture, without retyping `@file` on every line. Today a lone `@dev` line followed by
bullets is captured as **one inbox task whose text is the literal `@dev`**, and
`@dev#Ideas` + bullets is refused with "task text is required". Both shapes are unused
in practice, so they are free to mean "a list".

The fc epic's `@file+id` + bullets form is the precedent: a note-less marker line
followed by bullets files each bullet separately. Project notes (`@route^id@` + bullets)
already give task bullets a trailing ` :<id>` (Next + Pomodoro Task Link) or ` ^<id>`
(name only), and `bob capture-complete` already has a `project_task_block_id` prompt
with body-derived suggestions. This epic reuses both.

## Design principles

1. **One rule explains the whole form.** Each bullet under `@dev` captures exactly as
   `<bullet text> @dev` would on its own line, with its nested bullets as children. A
   ` :<id>` on the bullet is `@dev:<id>` with the route filled in. A ` ^<id>` is
   `@dev^<id>`. A bullet under `@dev#ideas` is `<bullet text> @dev#ideas`. Every
   per-task behavior that already exists (insertion point, duplicate-ID check, typed
   Pomodoro placement, future-schedule and blocked-project guards, `&`/`*`/`~`, `%c`)
   applies unchanged, because it is the same code.
2. **The draft reads like the note it writes.** A task list reads like the note's own
   task list, and a task's ID sits on that task's line. Typing ` :` on a nested bullet
   opens the prompt for the owner task, and accepting it puts the ID on the owner line.
3. **One grammar, every surface.** Execution, `capture-parse`, `capture-complete`,
   `capture-rewrite`, `capture-related`, and the Mac app read the same split and the
   same task-bullet lexer, so they cannot disagree. Bob Mac Capture stays a thin client
   (`decisions:mac-capture-is-a-thin-client`).

### Architecture: one capture item per bullet (zero-copy)

Split a list into one `CaptureItem` per nonempty first-level bullet inside
`push_capture_item` (`src/native/capture_language/draft.rs`). This is the same place
URL-list blocks and session-operator chains already split one physical item into
several. Each per-bullet item:

- is made of `RawLine` slices of the original draft (no rewritten text), so every span,
  range, and diagnostic offset stays exact;
- carries a new `list` context (suggested
  `ListContext { header: ItemLine, kind: Tasks | Notes, route token, position, count }`)
  that the execution parser (`item.rs`, `line.rs`) and the editor parser
  (`editor_parse.rs`, `editor_classify.rs`) both read;
- gets a unique draft-wide `index`, so the existing planner loop, ref verdicts, roll
  seeds, and `1/N` numbering keep working.

The planner (`capture/plan.rs` `plan_capture_item`) needs almost nothing new: a list
bullet arrives as an ordinary `Task`, Pomodoro-linked new task, or section `Bullet` item
with its route set from the header. **Do not** build a bespoke per-bullet loop like
`plan_multi_note_sub_bullet_item`; the closeout of fc found that such a loop drifts from
the ordinary item path.

Rejected alternatives:

- A textual `%t`-style expansion. It needs an offset map back to the draft, it gives
  parent-line meanings to bullet text (`- x` would close a Pomodoro), and it turns a
  nested `#` into a first-level drawer shortcut.
- One item with a per-bullet loop, the fc multi-note shape. It duplicates the ordinary
  item post-processing.

## Shared contract

### Vocabulary

- **List header**: the first physical line of an item when that line, ignoring
  surrounding whitespace, is exactly one token that lexes as a route marker of kind
  `@route` (task) or `@route#section` / `@route#"Literal"` / `@route#` (bullet). These
  are the same token kinds `parse_terminal_route_token` returns.
- **Task list**: header `@route`. **Section list**: header `@route#…`.
- **List bullet**: a nonempty first-level authored line under a header. A list bullet's
  **block** is that bullet plus the nested lines that follow it.
- **Owner**: for a nested line, the list bullet (or project-note task bullet) it sits
  under.

### Header outcomes

| Item's first line                                                         | Lines below                    | Outcome                                                                                                                                            |
| ------------------------------------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@dev`                                                                    | bullets                        | **Task list**: each list bullet becomes a new task in `dev.md`                                                                                     |
| `@dev#ideas`, `@dev#"Ideas"`, `@dev#`                                     | bullets                        | **Section list**: each list bullet becomes a section bullet, exactly as `<text> @dev#ideas` (prefix match, zeroth-section fallback, `[created::]`) |
| `@dev`, `@dev#ideas`                                                      | none, or only placeholder `- ` | Incomplete. `capture-parse`: `needs: ["list_item"]`. `bob capture` refuses `empty_list`. This replaces today's literal `@dev` inbox task           |
| `@`, `@de` (route not finished)                                           | bullets                        | Incomplete `needs: ["route"]` (route picker on the header); `bob capture` refuses with the existing incomplete-route copy                          |
| `@dev s:3`, `@dev #fix`, `@dev $1`                                        | bullets                        | Refused `list_header_marker`: a header stands alone; move the marker to the bullet it configures                                                   |
| `Plan trip @dev`, `@dev Plan trip`                                        | bullets                        | Unchanged: one task with child bullets                                                                                                             |
| `@dev+id`, `@dev+id#drawer`, `@dev^id`, `@dev:id`, `@dev^id@`, `@dev:id@` | bullets                        | Unchanged                                                                                                                                          |
| `@@dev`                                                                   | anything                       | Unchanged: a draft-wide declaration, never a header                                                                                                |

Forced `--route/--section/--task/--task-ref/--drawer` modes keep `@tokens` literal, so
no header is recognized there. In a `%t` template draft, a header in the template body
is refused (`template_list`), and template rows are never split by headers.

> [!decision] header_starts_item A column-zero line that is a list header also starts a
> new item when no blank line precedes it. Today that line is a prose-continuation usage
> error, so nothing that works today changes. For example, `@dev` / `- a` / `@cash` /
> `- b` is two lists. The rule is lexical and applies only outside template rows. Every
> draft consumer (`split_capture_draft`, `editor_item_at`, `capture-rewrite`,
> `capture-related`) sees the split.

> [!decision] header_starts_item = no Only blank lines separate items; a header line
> inside an item keeps today's usage error, with copy suggesting a blank line before it.

### Task-list bullets

Each list bullet captures as `<text> @dev`, `<text> @dev^<id>`, or `<text> @dev:<id>`:

| Draft                                 | Writes (and why)                                                                                                                                                                                                                                                   |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `- Call the bank`                     | `- [ ] #task Call the bank [created::D]` at the ordinary `@dev` insertion point (`insert_task_line`). Bullets land in draft order                                                                                                                                  |
| `- Call the bank ^call-bank`          | Ordinary task with that block ID (`@dev^call-bank`); `[?]` when scheduled                                                                                                                                                                                          |
| `- Call the bank :call-bank`          | Next `[*]` task plus `[[dev#^call-bank]]` under the Pomodoro the task's type places in today's daily note (`@dev:call-bank`). A future `s:<N>` gives `[?]`, unlinked; a blocked destination project gives `[?]` + `#prj_is_blocked`, unlinked, with the link guard |
| `- Call the bank s:2 #fix :call-bank` | Trailing markers configure **this** task. The ID token may sit anywhere in the line's trailing marker run (`:call-bank s:2` ≡ `s:2 :call-bank`)                                                                                                                    |
| `- Deploy &dev:build`                 | `&`, `*`, `~`, `~~`, and `%c` work per task. A later bullet may name a task an earlier bullet creates, because items plan in order against staged content                                                                                                          |
| `  - detail` (nested)                 | A child of that task, one indentation unit beneath it                                                                                                                                                                                                              |
| `  - detail s:2` (nested)             | Trailing markers on nested lines configure the owner task (today's item-wide rule). A nested-start `#fix` still types the owner, as it does everywhere                                                                                                             |
| `- [ ] Call the bank`                 | A leading open checkbox is dropped; the bullet becomes an open task                                                                                                                                                                                                |

Refusals and non-meanings on task-list bullets:

- **Task IDs**: a ` :id` / ` ^id` on a nested line is `misplaced_project_task_id`. A
  lone ` :` or ` ^` that ends a list bullet or one of its nested lines is an unfinished
  ID: `capture-parse` reports `needs: ["block_id"]` with an `interactive_placeholder`
  span and no diagnostic; `bob capture` refuses it. The existing codes apply: an invalid
  or reserved ID or an empty body is `invalid_project_task_id`, and two equal IDs in one
  list are `duplicate_project_task_id`, naming both lines. An ID already in `dev.md`
  fails at plan time with today's duplicate-block-ID copy. Sealed or suffixed IDs
  (`:id@`, `^id@`, `:id#name`, `:id=`) are `invalid_project_task_id` (see Non-goals).
- **Route-changing markers** are refused `list_bullet_marker`: any `@…` marker, a
  trailing `+`/`+query` parent selector, `%n`, `%t`, `%l`, and `!note:id`. These would
  change where or what the bullet captures.
- **Other checkbox states** (`- [x] …`, `- [?] …`) are refused `list_bullet_checkbox`.
- **A bullet-start `#`** (`- #fix …`, `- #`) is `drawer_bullet_position` with list copy.
  This keeps fc's lexical rule that a first-level bullet-start `#` never types.
- **Whole-item forms** do not apply. A list bullet is always task text with trailing
  markers, exactly like any later line of an item. `- x`, `- =`, `- +2`, `- https://…`,
  `- ^dev:foo`, and `- :dee` are never a close, start, adjust, ref job, caret link, or
  picker. Bob reads them as task text or as an ID token, and the ID-token rules above
  apply.
- **Drawers** are not offered under list tasks. A nested ALL-CAPS line is a plain note,
  because a drawer needs a third authored level (bob-cli-fi tracks that).

### Section-list bullets

Each list bullet captures as `<text> @dev#ideas`, with its nested lines as children. A
` :id` / ` ^id` stays literal text, as it does outside task bullets, and opens no
prompt. Route-changing markers are `list_bullet_marker`. A bullet-start `#` is
`drawer_bullet_position` with section-list copy. Every other marker behaves exactly as
on `<text> @dev#ideas`.

### Worked example

`dev.md` before:

```markdown
## Tasks

- [ ] #task Existing thing [created::2026-10-01] ^existing
```

Bryan types this. On the nested line he types ` :`, the prompt suggests
`fix-flaky-gkeep` for the owner, and he presses Return:

```text
@dev
- Fix flaky gkeep test
  - repro on CI first :
- Write release notes ^release-notes s:2
- Ping Alice about the cutover
```

The prompt moves the ID onto the owner line, and the caret returns to the end of the
nested line:

```text
@dev
- Fix flaky gkeep test :fix-flaky-gkeep
  - repro on CI first
- Write release notes ^release-notes s:2
- Ping Alice about the cutover
```

Capturing writes `dev.md`:

```markdown
## Tasks

- [ ] #task Existing thing [created::2026-10-01] ^existing
- [*] #task Fix flaky gkeep test [created::2026-11-02] ^fix-flaky-gkeep
  - repro on CI first
- [?] #task Write release notes [created::2026-11-02] [scheduled::2026-11-04]
  ^release-notes
- [ ] #task Ping Alice about the cutover [created::2026-11-02]
```

In the same atomic batch, it writes `[[dev#^fix-flaky-gkeep]]` under the Pomodoro the
new task's type places in today's daily note. If none fits, it creates the
`- [ ] () — TYPE` placeholder first, exactly as
`Fix flaky gkeep test @dev:fix-flaky-gkeep` does.

**Equivalence invariant (tested)**: when no nested line starts with `#`, a list writes
byte-for-byte what the equivalent blank-line batch writes. That batch is
`Fix flaky gkeep test @dev:fix-flaky-gkeep` / `- repro on CI first`, blank,
`Write release notes @dev^release-notes s:2`, blank,
`Ping Alice about the cutover @dev`. The same holds for section lists and
`<text> @dev#ideas`.

### Task-bullet grammar shared with project notes

The ID lexer (`capture_language/project_tasks.rs`, `ProjectTaskPass`) serves both list
bullets and project-note task bullets.

> [!decision] project_note_parity Project-note task bullets get the list rules:
>
> - The ID token is recognized anywhere in the bullet line's trailing marker run. Today
>   `- Draft memo s:2 :draft-memo` silently leaves `s:2` as literal body text; verified
>   with `capture-parse` at HEAD.
> - A lone ` :`/` ^` ending a nested line is an unfinished ID for the owner. It used to
>   be literal text.
> - The nested-line prompt, the owner-line `splice`, and the `:`/`^` toggle work there
>   too.
>
> What item-wide markers mean in a project note is unchanged: they configure the
> project.

> [!decision] project_note_parity = no Project-note bullets keep today's rules (the ID
> counts only as the last word, nested lone sigils are literal, there is no nested
> prompt), and the lexer branches by form.

### Refusal copy (Bob-authored; wording may be polished, codes may not)

- `empty_list`: "`@dev` heads a list but has no bullets: add `- <task>` lines below it,
  or put the task on the same line (`Call the bank @dev`)". In a section list, say
  `- <note>`.
- `list_header_marker`: "`@dev` heads a list, so it stands alone on its line: move `s:3`
  to the bullet it schedules (`- Call the bank s:3`)".
- `list_bullet_marker`: "every bullet under `@dev` files into dev.md; `@cash` would
  route this one elsewhere, so give it its own item (a blank line, then
  `Call the bank @cash`)". Add variants for `+` (its own `@dev+id` item), `%n`, and
  `!note:id`.
- `list_bullet_checkbox`: "list bullets become new open tasks: drop `[x]` (complete an
  existing task with `!note:id`)".
- `drawer_bullet_position` in a task list: "list bullets are tasks, so `#` cannot start
  one: type the task at the end of its line (`- Call the bank #fix`), or write `\#` to
  keep a literal `#`".
- `misplaced_project_task_id` on a nested line: "a task ID goes at the end of its task's
  own bullet: move `:call-bank` to `- Call the bank`".
- Unfinished ID: "` :` on `Call the bank` needs a task ID: end the bullet with
  ` :<block-id>`". Phase `editor-assists` replaces `<block-id>` with Bob's suggested ID
  (` :call-bank`), and nested copy names the owner.

### Output contract (additive)

- **`bob capture` JSON**: each list bullet is one result in `captures[]`; the legacy
  top-level fields are the first result, as for any batch. Each list result adds
  `list: {kind: "tasks"|"notes", route, section, header_line, position, count}`.
  `section` is the typed selector or null, `header_line` is 1-based, and `position` is
  1-based within the list. Non-list results omit `list`.
- **Human output**: per-item rendering and global `i/N` numbering are unchanged. One
  summary line precedes each list's first item: `list  dev.md · 3 tasks` or
  `list  dev.md#ideas · 2 notes`.
- **`capture-parse`**: one `items[]` entry per list bullet (`range` = the bullet's
  block). Each entry has its normal mode (`task`, `pomodoro_task`, `bullet`,
  `incomplete`), the same `list` object, and `header_range`. The header token gets
  exactly the spans a leading `@route`/`@route#section` marker gets. List-bullet IDs get
  `project_task_link_marker` / `project_task_block_id` spans. A header with no bullets
  is one `incomplete` item with `needs: ["list_item"]` and `list.count: 0`. A cursor or
  diagnostic on the header line belongs to the list's first item.
- `GRAMMAR_REVISION` goes from 5 to 6 (`src/native/capture_language/mod.rs`). The Mac
  gate stays at 3.

### Block-ID prompt contract (phase `editor-assists`)

The `project_task_block_id` completion context opens on a cursor inside a trailing ID
token (including a lone ` :`/` ^`) that is in the trailing marker run of a task-list
bullet or of a nested line under one. It opens the same way in project-note items under
`project_note_parity`. Section lists never open it.

- `replacement` keeps its meaning: the ID part of the token under the cursor.
- The `block_id` object for a list bullet uses:
  - `route` = the header route and `relative_target` = `dev.md`;
  - `note_exists`, `marker`, `marker_range` (the token under the cursor), and
    `intent: "new"`;
  - `body` = the owner's clean text (no ID, checkbox, or markers);
  - `suggestions` = at most 3, each free in the note and in the list. When the owner
    already has a valid ID, that ID comes first;
  - `used` = every ID in the destination note, with task status, exactly as
    `task_block_id` builds it (`collect_used`). Then the other list bullets' IDs follow
    (`task: true`, null status, draft line), deduplicated by first occurrence. The
    owner's own current ID is excluded.
- New additive fields:
  - `owner_line`: the 1-based draft line of the owner bullet.
  - `splice: {range: {start, end}, before, after}`, always present for this context. A
    client writes `before + ID + after` over `range` and puts the caret at the end of
    what it wrote. Cases:
    - **Same line, no other ID on the owner**: `range` = `replacement`, and `before` and
      `after` are empty. Today's insertion is unchanged.
    - **Owner already has a different ID token** (a second sigil typed on the owner
      line, or a prompt from a nested line): `range` starts at that token's sigil, so
      the old token is replaced.
    - **Nested line**: `range` runs from the owner's insertion point (the end of the
      owner line's text, or its existing token's sigil) to the end of the typed token.
      `before` is ` :` or ` ^` (just the sigil when replacing). `after` is the original
      bytes in between, minus the typed token and the one space before it. The caret
      therefore returns to where Bryan was typing.
- Older clients that ignore `splice` insert at `replacement`. On a nested line that
  gives a `misplaced_project_task_id` refusal teaching the move; it never writes the
  wrong file.
- **Sigil toggle** (`capture-rewrite`): typing `^` right after a complete `:id` that
  ends a task bullet gives `^id`, and `:` after `^id` gives `:id`. The key is consumed
  and the cursor lands after the ID, mirroring the block-ID separator toggle. A list
  bullet and, under `project_note_parity`, a project-note task bullet are both eligible.

### Default ID contract (phase `id-suggester`)

The prompt's default is its first suggestion. Phase `id-suggester` retunes the one
shared suggester (`capture_block_ids.rs` `suggest_ids_with_used`, `STOPWORDS`,
`LEADING_VERBS`, and `mint_block_id`). Every ID prompt and every machine-minted ID
improves together. Existing vault IDs are never rewritten.

**Baseline, measured for this plan.** The sample was 400 random task lines with a
trailing `^id` from Bryan's vault root notes, excluding `^prj` and `#ref` rows. Each was
run through `bob capture-complete` on `<text> @dev^`.

| Metric                                                                       | Current suggester | Candidate rules below |
| ---------------------------------------------------------------------------- | ----------------- | --------------------- |
| Top-1 exact match                                                            | 14.3%             | 8.8%                  |
| Top-3 exact match                                                            | 15.8%             | 16.8%                 |
| Mean word-set Jaccard, first suggestion                                      | 0.328             | 0.361                 |
| Mean Jaccard, best of 3                                                      | 0.412             | 0.437                 |
| First-suggestion Jaccard on the 337 rows not matching any current suggestion | 0.218             | 0.301                 |
| Mean first-suggestion length (authored mean 13.8 bytes)                      | 14.1              | 17.2                  |

Top-1 exact is biased toward the current suggester, because some vault IDs are its
accepted outputs. Do not gate on it.

**What the data says.** These authored IDs are 2–3 words (2: 35%, 3: 37%, 1: 15%, 4+:
13%) and drop low-content lead-ins. The share of tasks keeping a word in their authored
ID, when the word appears in the first six words of the text, is:

- almost never kept: `add` 1%, `support` 2%, `for` 3%, `start` 0%, `should` 0%,
  `ability` 0%, `make` 7%, `get` 11%, `use` 14%, `new` 17%, `all` 19%;
- usually kept: `fix` 100%, `apply` 88%, `research` 82%, `read` 80%.

For example, `Add support for hermes LLM provider!` is authored `hermes-llm`, but the
current suggester's first suggestion is `add-support-hermes`.

**Candidate rules** (the starting point; tune with the harness):

- Drop those filler words and lead-ins (`add support for`, `add ability to`,
  `make sure (that)`, `make it possible to`, `figure out (how to)`) along with the
  stopwords. Keep content verbs that authored IDs keep.
- The first suggestion is the first 3 content words when they fit in 24 bytes, else the
  first 2. Then offer the 2-word form, then the first phrase (code span, quoted phrase,
  or link label) when it is distinct.
- Keep the 32-byte cap, validation, dedupe, and `-2…-9` suffixing against `used`.

## Phase `list-capture`

1. **Split**:
   - In `push_capture_item`, detect a header (the first line, exactly one route token of
     the two kinds) and emit one `CaptureItem` per nonempty list bullet block, each with
     a `ListContext`. Skip placeholder rows, and keep the current owner across them.
   - A header with no list bullet is one item with `count: 0`.
   - Keep `orphaned_nested_bullet` for a nested line before any bullet.
   - Implement `header_starts_item` per the accepted decision.
   - Make template drafts refuse a header (`template_list`) and never split rows.
   - Give `@@` declaration lines no header meaning.
2. **Execution parse** (`item.rs`, `line.rs`, `project_tasks.rs`):
   - Parse a list bullet's line as a later line: task text plus trailing markers, with
     no leading-route, operator, URL-ref, caret-link, or picker forms.
   - Map its ` :id` / ` ^id` (lexed by `lex_project_task_id`, accepted anywhere in the
     trailing run) to the header route as `@route:id` / `@route^id`. A bullet with no ID
     inherits `@route` (or `@route#section`), like `inherit_global_destination` but
     item-local and exempt from `@@`.
   - Raise every refusal in the contract (`empty_list`, `list_header_marker`,
     `list_bullet_marker`, `list_bullet_checkbox`, `drawer_bullet_position`, ID codes,
     unfinished ID), all before planning.
   - Detect `list_header_marker` when line 1 is a route marker plus only item-wide
     markers and bullets follow; today that hits `missing_text_error`.
   - Apply `project_note_parity` per the accepted decision.
3. **Editor parse** (`editor_parse.rs`, `editor_classify.rs`, `capture_parse.rs`):
   - Report the same items, modes, spans, needs (`list_item`, `route`, `block_id`), and
     diagnostics with identical codes and copy.
   - Resolve cursor and diagnostic lookups on a header line to the list's first item.
   - Update `editor_item_at` and every `split_capture_draft` consumer that assumes one
     item per physical block (the explore pass found `editor_item_at`, `rewrite.rs`, and
     `capture_related/draft.rs`).
4. **Planning and output** (`capture/plan.rs`, `capture/output.rs`, `capture/batch.rs`):
   - Thread `ListContext` to results.
   - Add the `list` JSON object and the human `list  dev.md · N tasks` line.
   - No per-bullet planning loop.
5. **Grammar 6**:
   - Bump `GRAMMAR_REVISION`.
   - Update every grammar assertion.
   - Fix every stale grammar number in the docs. The explore pass found 4, 3, and 2 in
     `docs/capture.md`, `docs/reference/json-outputs.md`, the compatibility docs, and
     the `capture-parse --help` text.
   - Commit with `feat(capture)!:` and a `BREAKING CHANGE:` footer naming each change: a
     lone header plus bullets is a list; a lone header alone is `empty_list`; and, under
     parity, the project-note ID run and nested lone sigil. Per the release-train
     decision, never touch the `Cargo.toml` version or `CHANGELOG.md`.
6. **Docs**:
   - `docs/capture.md`:
     - Add a new `### Lists: several tasks or notes at once` section after "Authored
       sub-bullets", with the worked example, both tables, the equivalence rule, and the
       refusals. Add it to Contents.
     - Add grammar-at-a-glance rows for the header forms and list-bullet IDs.
     - Fix the "Authored sub-bullets" and "Multi-item capture" wording ("authored child
       bullets never start an item" no longer holds).
     - Update the project-notes ID paragraph under parity.
     - Document the `list`/`header_range` fields, the `list_item` need, and the new
       codes in the `capture-parse` reference.
   - Update `docs/reference/json-outputs.md`.
   - Update the `capture` and `capture-parse` `--help` text
     (`src/native/capture/cli.rs`, `src/native/capture_parse.rs`).
   - Run `just docs-gen` and `just docs-check`.
7. **Tests**: add a new `tests/cli/capture/lists.rs`, registered in
   `tests/cli/capture/mod.rs`, plus `capture_language/tests` unit tests. Cover:
   - **Worked example**: byte for byte (both files), JSON `list` objects, and human
     output.
   - **Equivalence**: list ≡ batch, as a dry-run comparison of written bytes, for a task
     list (plain, `^`, `:`, `s:`, `#type`, `&` naming an earlier bullet, nested
     children) and for a section list.
   - **`:` link guards**: a future `s:<N>` and a blocked destination project.
   - **Duplicate IDs**: within a list and against the note.
   - **Refusal table**: every row in parse and execution with the same code. These are
     conformance vectors: `capture-parse` diagnostics/needs ≡ `bob capture` refusals.
   - **Header variants**: lone header; `@` header; header plus markers;
     `header_starts_item`; a template body header; `@@` lines.
   - **Whole-item forms stay text** (`- x`, `- =`, `- https://…`).
   - **Atomicity**: a later item's failure rolls back the list.
   - **Parity**: the project-note vectors under `project_note_parity`.
   - **Unchanged forms**: existing multi-note `@dev+id` + bullets and `Plan trip @dev` +
     bullets.
8. **Gate**: `just check` and `just docs-check` pass. If `just check` fails on the clean
   base for pre-existing reasons, record the exact failures on the phase bead rather
   than claiming green.

## Phase `id-suggester`

1. **Harness first**:
   - Add an ignored test (suggested `block_id_calibration`) that reads a vault path from
     `BOB_ID_CALIBRATION_VAULT`. Collect `(clean task text, authored ID)` pairs from
     root `*.md` task lines ending in `^id`, skipping `prj`, `#ref`, and `[[ref/…` rows.
     Strip `#task`, inline `[field:: …]` fields, and tags from the text.
   - Call the suggester directly.
   - Print top-1 exact, top-3 exact, mean word-set Jaccard of the first and best-of-3
     suggestion, and mean lengths.
   - It never writes and never asserts on private data. No vault text enters the repo, a
     commit, or a bead note; only the aggregate numbers do.
2. **Baseline**: run it on `~/bob` before changing code, and record the numbers on the
   phase bead (expect values near the table above).
3. **Retune** `suggest_ids_with_used` from the candidate rules. Keep `mint_block_id`
   using the first suggestion, and keep 32-byte, validation, dedupe, and suffix
   behavior. Update the docs paragraph that specifies suggestions (`docs/capture.md`,
   the `block_id` object under `bob capture-complete`).
4. **Acceptance** (recorded before and after on the bead):
   - full-corpus first-suggestion Jaccard ≥ baseline + 0.03;
   - best-of-3 Jaccard and top-3 exact not lower than baseline;
   - mean first-suggestion length ≤ 17 bytes.
5. **Tests**:
   - Synthetic unit examples. `Add support for hermes LLM provider` starts with
     `hermes-llm`; `Fix flaky gkeep test` keeps `fix`. Also cover filler lead-ins,
     code-span phrases, the 24-byte rule, and suffixing.
   - Update every expectation that changes in minted-ID callers (`project_note.rs`
     picks, `blocks_mirror.rs`, `task_status_hooks`, `task_complete/successors.rs`,
     `relation_labels/reverse.rs`, `gcal/plan.rs`) and in picker suggestions. A change
     there must be a suggestion change, not a behavior change.
6. **Gate**: `just check` and `just docs-check`.

## Phase `editor-assists`

1. **Prompt** (`capture_language/completion.rs` `project_task_token_at` /
   `project_task_block_id_detail`, and `capture_block_ids.rs`
   `build_project_task_block_id_field`):
   - Detect the token on list bullets, and on nested lines through their owner.
   - Build the list `block_id` object (`collect_used` on the destination note plus
     sibling IDs, owner-first suggestions, `owner_line`, `splice`) per the contract.
   - Apply parity to project notes.
   - Keep `replacement` and every other context byte-identical.
2. **Toggle**: add the task-bullet `:`↔`^` rule to `capture_language/rewrite.rs` and
   its docs paragraph, which currently lists project-task IDs as never rewritten.
3. **Refusal copy**: in both the execution and editor paths, the unfinished-ID copy
   names Bob's body-derived first suggestion (computed lexically; the note is checked at
   plan time). Nested copy names the owner.
4. **Similar tasks** (`capture_related/draft.rs`):
   - List bullets are task-creating items with `marker_at` on the bullet line and
     `route_range` on the header token.
   - `discard_range` covers the bullet block, and also the header when it is the list's
     only bullet, so a discard never leaves an `empty_list`.
   - Actions keep today's verify-by-reparse rule, so a `note` action that would re-route
     a bullet drops out.
5. **Docs**:
   - `docs/capture.md`: in the capture-complete section, cover the list prompt,
     `owner_line`, `splice`, and nested prompts. Also update the Interactive editor
     markers section and the capture-rewrite toggle.
   - `docs/completion.md`.
   - `docs/reference/json-outputs.md`.
   - `capture-complete --help` (`capture_complete/cli.rs`) needs one sentence.
   - Run `just docs-gen` and `just docs-check`.
6. **Tests**:
   - `tests/cli/capture/complete_block_id.rs` and `rewrite.rs`, plus completion unit
     tests:
     - same-line and nested prompts in lists and project notes;
     - `used` merging note and sibling IDs with the owner's own ID excluded;
     - owner-first suggestions;
     - each `splice` case, applied to the draft and then reparsed to show the ID landed
       on the owner and the caret offset;
     - section lists opening nothing;
     - the toggle both ways.
   - A commit-agreement test: applying `splice` with the first suggestion and capturing
     yields the `body`'s task with that ID.
   - Related-tasks anchors for list bullets.
7. **Memory**: use `/sase_memory_write` and act only within the accepted memory
   decisions. Run `sase memory init` after editing.

   > [!decision] memory_bulk_capture_glossary In `glossary/bulk-capture.md`, replace
   > "Authored child bullets never start an item." Say that a lone `@file` or
   > `@file#section` header line followed by bullets is a list, where each first-level
   > bullet (with its nested bullets) is its own item routed by the header. Mention the
   > `header_starts_item` behavior if that decision was accepted. Leave the rest of the
   > entry as is.

   > [!decision] memory_list_decision_record Add the decisions strand
   > `capture-lists-file-each-bullet`, titled "Capture Lists File Each Bullet As Its Own
   > Capture", with the summary "A lone @file or @file#section header files each bullet
   > below it exactly as `<bullet> @file` would; a task bullet's ID lives on its own
   > line."
   >
   > - **Applies to**: bob-cli and Bob Mac Capture.
   > - **Claim**: the Shared contract above.
   > - **Why**: one sentence explains the form; it reuses the ordinary item path; and it
   >   fills the literal-`@dev` and refused `@dev#Ideas` gaps.
   > - **Rejected**: textual expansion; a bespoke per-bullet loop; header-wide default
   >   markers; IDs allowed on nested lines.
   > - **Cost**: grammar 6, and a lone header is refused.
   > - **Reopens when**: authored bullets gain a third level, or Bryan wants header-wide
   >   markers.
   > - **Evidence**: this epic's archived plan and its phase commits.
   >
   > Link `[[decisions:capture-bullets-mirror-the-task-block]]` and
   > `[[glossary:bulk-capture]]`.

8. **Gate**: `just check` and `just docs-check`.

## Phase `mac-lists`

Work in the linked `bob-mac-capture` repository, opened with `/sase_repo` (read its
`AGENTS.md` first). Bob stays the grammar authority; Swift decodes, applies Bob's edits,
and paints.

1. **Accept with splice**: in the New ID composer (`CapturePanelModel.swift` block-ID
   accept path, documented in `docs/task-pickers.md` "Block ID Picker"), when Bob sends
   `block_id.splice`, write `before + ID + after` over its range under the existing
   stale-draft guard and put the caret at the end of the written text. Fall back to
   `replacement` when it is absent. Undo restores the pre-accept draft. Announce
   "Inserted :fix-flaky-gkeep on ‘Fix flaky gkeep test’".
2. **Owner caption**: when `owner_line` is not the caret's line, the composer header and
   detail strip name the owner task ("ID for ‘Fix flaky gkeep test’ · New Next task
   linked into today's Pomodoro").
3. **Preview grouping**: decode the additive `list` object with `decodeIfPresent`, and
   stack a list's results under one capsule ("3 tasks → dev.md", or "2 notes → dev.md ·
   ideas"), with a VoiceOver phrase. Non-list output is unchanged.
4. **First bullet**: when `capture-parse` reports `list_item` for the item at the caret
   and the caret ends the header line, Return inserts `\n- ` instead of a bare newline.
5. Keep `CaptureGrammarGate.minimumRevision` unchanged.
6. **Fixtures**: regenerate the affected fixtures from a real bob built at this epic's
   head: a list dry run, a `capture-parse` of the worked example, a same-line prompt,
   and a nested-line prompt with `splice`. Add CaptureCore and hosted tests for splice
   accept, caret placement, caption, grouping, and Return-after-header.
7. **Gate**: agent hosts have no Swift toolchain, so macOS CI is the gate. Report the
   run's result on the phase bead. If CI fails for a pre-existing unrelated reason (for
   example the bob-cli-fj build error, if still open), record it instead of claiming
   green.

## Non-goals and follow-ups

- **Header-wide markers** (`@dev #fix` typing every bullet) are refused for now;
  `list_header_marker` teaches the per-bullet form.
- **Suffixes on list IDs** (`:id#pomodoro`, `:id=`, `:id=x`, `^id@`) are refused. A
  bullet's `#type` decides Pomodoro placement, and a separate `=` item starts a session.
- **Third-level authored bullets** and drawers under list tasks are tracked by
  bob-cli-fi.
- **Creating missing sections**: a section list keeps `@route#section`'s zeroth-section
  fallback.
- **Forced modes** (`--route/--section/--task/--task-ref/--drawer`), Work Log close
  bullets, and `%t` rows are unchanged. There are no new CLI commands or options.

## Verification

Each phase passes `just check` and `just docs-check` in bob-cli, or records exact
pre-existing failures. The final state is also verified by hand in a scratch vault
(`BOB_DIR` pointing at a temp dir, with `BOB_NOW` pinned and a daily note with
`## Pomodoros`). That pass runs `bob capture --dry-run` (human and `-f json`),
`bob capture-parse -f json`, `bob capture-complete -f json`, and
`bob capture-rewrite -f json` over:

- the worked example (before and after accepting the nested prompt);
- a section list;
- every refusal row;
- a project-note item under parity.

The `mac-lists` phase is verified by macOS CI.
