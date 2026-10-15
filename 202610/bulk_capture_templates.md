---
tier: epic
title: Bulk capture templates (@@template:)
goal: "A draft that starts with `@@template:` captures one item per row from a
  `{0}`-style template, exactly as if the expanded items had been typed, all or nothing;
  `bob capture`, the `capture-*` editor protocol, and Bob Mac Capture make templates
  easy to write, easy to check, and pleasant to read.

  "
decisions:
  mac_expand_action:
    ask:
      Should the Mac phase also add an "Expand template" editor action backed by `bob
      capture -E`?
    default: false
    why: v1 stays small; -E ships now, so the action is a cheap later add-on
    answer: false
  memory_bulk_capture_glossary:
    ask:
      Should the last bob-cli phase update the glossary's Bulk Capture strand to cover
      template drafts?
    memory:
      - glossary:bulk-capture
    default: true
    corrects:
      "Only a `@@route` declaration spans items: it routes every item that lacks its own
      route or mode marker."
    answer: true
  memory_template_decision:
    ask:
      Should the last bob-cli phase add a decision record that bulk templates are a
      declared expansion before parsing?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: language
    title: Template language core and bob capture -E
    depends_on: []
    size: medium
    description:
      "language: add the pure template module (framing, placeholders, row lexer,
      expansion, bounds, issue vocabulary) and its first consumer, `bob capture -E,
      --expand`."
  - id: execution
    title: Template execution in bob capture
    depends_on:
      - language
    size: medium
    description:
      "execution: run template drafts through the unchanged planner as N instances, with
      row-scoped errors, refusals, additive JSON, human summary, help, docs, and the
      equivalence oracle."
  - id: parse
    title: Template-aware capture-parse
    depends_on:
      - execution
    size: medium
    description:
      "parse: teach `capture-parse` template drafts through the masked copy, with
      placeholder and field spans, row items, mapped diagnostics, needs rules, the
      template object, and the header-less hint."
  - id: complete
    title: Template-aware completion and rewrite
    depends_on:
      - parse
    size: small
    description:
      "complete: gate `capture-complete` and `capture-rewrite` for template drafts,
      offer the `template:` header candidate, finish the protocol docs, and apply the
      memory decisions."
  - id: mac
    title: Bob Mac Capture template presentation
    depends_on:
      - execution
      - parse
      - complete
    size: medium
    description:
      "mac: decode the additive template fields and present them in Bob Mac Capture:
      slot-colored placeholders and fields, a calm no-rows state, in-place diagnostic
      underlines, a template preview chip, and the header candidate row."
proposed_by: bbugyi200.apollo.research.1j.linker.w0
decided_by: auto
create_time: 2026-10-15 14:04:16
status: wip
---

- **PROMPT:**
  [prompts/202610/bulk_capture_templates.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bulk_capture_templates.md)

# Plan: Bulk capture templates (`@@template:`)

## Why and what

Bryan creates runs of near-identical captures, for example "Apply to 29 job openings" in
waves of 10–12 roles. Today each item is typed in full, so a typo in the route or ID
shape is repeated, or slips into one item only. A **bulk template** lets the first block
of a draft be a template, and every later line is a **row** of values that fills it:

```text
@@template:
Apply to {0}! @job_2026_apply_29:{1}
- **URL**: {2}

Foobar foo http://foobar.com/career/some_foo_job
Bazbuz baz http://bazbuz.com/career/some_baz_job
"Two Sigma" two-sigma https://www.twosigma.com/careers/some_role
```

This captures exactly what these three typed items would capture:

```text
Apply to Foobar! @job_2026_apply_29:foo
- **URL**: http://foobar.com/career/some_foo_job

Apply to Bazbuz! @job_2026_apply_29:baz
- **URL**: http://bazbuz.com/career/some_baz_job

Apply to Two Sigma! @job_2026_apply_29:two-sigma
- **URL**: https://www.twosigma.com/careers/some_role
```

Bryan's original layout, with blank lines between rows, also works once the header line
is added.

The design follows the merged research report
`research:202610/bulk_capture_template_drafts/bulk_capture_template_drafts.md`. Bryan
accepted all of its requirements. This plan turns them into a concrete contract and
splits the work into phases. Where the report left something open, this plan decides it;
the decisions are marked **(decided here)**.

### The one rule

> Capturing a template draft `D` is exactly capturing `expand(D)`.

The planner, the all-or-nothing commit, duplicate block-ID checks, reference jobs, the
plan budget, and the Mac preview cards do not change. Templates differ from typing the
expansion in only three ways: error locations point at the typed draft, a few constructs
are refused, and JSON gains additive metadata.

### Architecture

- **bob-cli owns everything.** A new pure module,
  `src/native/capture_language/template.rs`, frames the draft and expands it. It does no
  vault, clipboard, or network I/O.
- **No new execution machinery.** There is no new `CaptureKind`, no recursion, and no
  subprocess per row.
- **Bob Mac Capture stays a thin client** (decision
  `decisions:mac-capture-is-a-thin-client`). It never expands templates and never
  decides template syntax. It decodes the new additive fields and presents them.
- **Additive protocol only.** `schema_version` stays `1` for `capture`, `capture-parse`,
  `capture-complete`, and `capture-rewrite`. Drafts without a header produce output that
  is byte-identical to today's.

## Normative grammar

Every phase implements this section. Where a later phase needs a detail, this section is
the source of truth.

### Header

- A draft is a **template draft** exactly when its first nonblank physical line, trimmed
  of ASCII spaces and tabs, equals `@@template:`. The match is ASCII case-insensitive,
  like routes **(decided here)**.
- Any other whitespace-separated token whose ASCII-lowercase form starts with
  `@@template:` is a misplaced header (`template_header_misplaced`). This covers:
  - a header that is not on the first nonblank line;
  - a second header;
  - an inline header such as `@@template: Apply …`;
  - an option-like header such as `@@template:tsv`, which is reserved.

  The error replaces today's `invalid_global_destination` for that token. Its message:
  "write `@@template:` alone on the draft's first line".

- Today's bob already refuses `@@template:` with exit 2 (`invalid_global_destination`),
  verified with a probe. So an older bob never captures a template draft literally.

### Template block

- The template is the first block of nonblank lines after the header. Blank lines
  between the header and the template are allowed.
- The template is one capture item: a parent line plus optional authored child bullets,
  at both depths, exactly as in `docs/capture.md` § Authored sub-bullets.
- The template ends at the first blank line. Inside the block, a line that does not look
  like a bullet (optional spaces, then `-`, `*`, or `+`, then a space or tab) gives
  `template_needs_separator`: "line N directly under the template is not a child bullet;
  add a blank line between the template and its rows". A line that looks like a bullet
  but is malformed is left to the existing grammar's errors.
- If no template line follows the header, the draft is incomplete
  (`template_missing_rows`; see Diagnostics).

### Placeholders

- A placeholder is `{` + a canonical ASCII decimal + `}`: `{0}`, `{1}`, … `{999}`.
  "Canonical" means no leading zeros, so `{01}` is invalid. They are zero-based, like
  Python's `str.format`.
- `{{` is a literal `{` and `}}` is a literal `}`.
- Any other brace form is `template_invalid_placeholder`, with the hint "write `{{` for
  a literal brace". This covers:
  - `{}`, `{name}`, and `{0:x}`;
  - a lone `{` or `}`;
  - an unterminated `{` or `{12`.

  These forms stay reserved for named slots, defaults, and filters later.

- The indices used must be exactly `0..N-1`. Repeats and any order are fine. A gap is
  `template_placeholder_gap`:
  - When the indices are dense from 1, the message teaches the 1-based habit:
    "placeholders start at {0}; this template uses {1}–{3}".
  - Otherwise: "this template uses {0} and {2} but not {1}; number placeholders
    {0}–{N-1} without gaps".
- At least one placeholder is required (`template_missing_placeholder`). While the draft
  has no rows yet, this check is suppressed; `template_missing_rows` already says what
  to do next.
- Placeholders may appear anywhere in the parent line or children, including inside
  markers and links: `@job:{1}`, `@job^apply-{1}`, `[[{0}]]`, `` `{0}` ``.
- Expansion is single-pass: an inserted value is never scanned again.
- `N` is the template's **field count**.

### Rows and fields

- After the blank line that ends the template, every nonblank physical line is one
  **row**, in order. Blank lines between rows are ignored. LF, CRLF, and bare CR are
  handled as `split_physical_lines` already handles them.
- A row whose trimmed text starts with `- `, `* `, `+ ` (or the same marker plus a tab)
  is `template_row_bullet`. Per-row child bullets are reserved for later. The message:
  "rows can't have child bullets yet; put each capture's values on one row".
- A row is trimmed of leading and trailing ASCII spaces and tabs.
- **One-field templates** (`N == 1`): the whole trimmed row is the value. There is no
  splitting and no quote handling, so `{0} @groceries` with the rows `oat milk` and
  `green onions` just works.
- **Multi-field templates** (`N > 1`): fields are separated by runs of ASCII spaces or
  tabs.
  - A field that starts with `"` or `“` is quoted.
  - It closes at the first `"` or `”` that is followed by whitespace or the end of the
    row. Either closing quote ends either opening style **(decided here)**: pasted text
    mixes straight and smart quotes, and the Mac editor keeps smart quotes on.
  - Any other quote inside a quoted field is literal.
  - Inside quotes, only `\"` → `"` and `\\` → `\` are escapes. This is bob's existing
    quoted-note rule.
  - `""` and `“”` are empty fields.
  - A quote in the middle of an unquoted field, as in `27"`, is literal. Apostrophes, as
    in `O'Reilly`, are never quotes.
  - A quoted field with no valid closing quote is `template_unterminated_quote`.
- **Exact field count.** Every row must have exactly `N` fields (`template_row_arity`).
  Example message:
  `template row 3 (line 7) has 4 fields but the template uses {0}–{2}; quote a field that contains spaces: "Two Sigma"`.

### Draft-wide refusals and bounds

- **Other `@@` tokens.** Any whitespace token starting with `@@`, other than the header,
  in the template or in any row, is `template_global_declaration`: "a template draft
  can't declare `@@` destinations; put the destination in the template line". This is
  lexical and checked in the language phase. A value cannot smuggle in a `@@` either,
  because rows are tokenized too.
- **Clipboard markers.** If an instance carries a `%`, `%N`, or `%header` clipboard
  marker, or the run passes `-c, --clip`, the draft is refused with
  `template_clipboard`: "a template can't capture the clipboard: `%` would repeat once
  per row".
  - This is checked as if clipboard markers were being parsed, even under
    `-n, --no-clip` **(decided here)**. The Mac live preview runs `--no-clip`, so a
    refusal that depended on that flag would let the preview pass where submit fails.
  - The `URL %` listen marker on a reference is not a clipboard marker and stays
    allowed.
- **Session operators.** An instance that is a whole-item session form is refused with
  `template_session_operator`: "a template can't drive the Pomodoro session; capture
  `=`, `=x`, `+N`, and `++N` on their own". The forms are:
  - `+N`, `-N`, `++N`, `--N`;
  - `=` and `=<X>` starts, `==` overrides, `=x…` closes;
  - a same-line operator chain, or anything else that splits into more than one item;
  - a marker carrying a `=<X>` start or `=x` close suffix.
- **Incomplete instances.** An instance that is still an editing state, such as an empty
  field leaving `@job:` or a value of `:dee`, fails capture with that form's existing
  error. `capture-parse` reports it as `template_row_incomplete`.
- **Bounds.** More than 1,000 rows, or more than 1 MiB of expanded text, is
  `template_too_large`. The draft is refused, never truncated.

### What values can do

A value is inserted as capture source. A value that forms a marker (`p:1`, `@home`) acts
as that marker. This is ordinary macro behavior: it is documented, and the preview shows
it.

Every marker family in the template applies to each instance:

- `@route`, `@route^{1}`, `@route:{1}`, `@route+id`, `#section`, `s:N`, `p:N`,
  `&note:id`, and `*note:id` all work per instance. Each instance rolls its own `p:N`
  date.
- `!note:{0}` (bulk complete), `^route:{0}` and `@route:{0}` (bulk link existing tasks),
  and bare-URL instances (reference jobs) all work.
- Duplicate IDs across rows are refused by the existing planner.

The docs must explain the `:` trade-off. `@route:{1}` links every row under the one
selected Pomodoro, and warns once today passes `plan.max_links`. `@route^apply-{1}`
creates plain Ready tasks that Bryan links as he works them.

### Expansion

- Each valid row produces one **instance**: the template's lines, with every placeholder
  replaced by that row's field value, `{{` replaced by `{`, and `}}` replaced by `}`.
- Lines within an instance are joined with `\n`, and instances are joined with `\n\n`.
  The header is dropped, and the text ends with one `\n`.
- `expand(D)` is the identity for drafts without a header. This is the text
  `-E, --expand` prints.

### Diagnostics vocabulary

Each code is a stable snake_case identifier. Execution reports the first issue in draft
order as a usage error (exit 2, nothing written). `capture-parse` reports every issue as
a diagnostic, with ranges in the typed draft.

| Code                           | Severity | When                                                                                                                                                                                      |
| ------------------------------ | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `template_header_misplaced`    | error    | `@@template:` token anywhere but alone on the first nonblank line                                                                                                                         |
| `template_missing_rows`        | error    | header with no template yet ("write the template on the next line, then a blank line and one row per capture"), or a template with no rows ("add a blank line, then one row per capture") |
| `template_needs_separator`     | error    | non-bullet line inside the template block                                                                                                                                                 |
| `template_missing_placeholder` | error    | template has no placeholder (only once rows exist)                                                                                                                                        |
| `template_invalid_placeholder` | error    | brace form other than `{N}`, `{{`, `}}`                                                                                                                                                   |
| `template_placeholder_gap`     | error    | indices not exactly `0..N-1`                                                                                                                                                              |
| `template_row_bullet`          | error    | row starts with a bullet marker                                                                                                                                                           |
| `template_unterminated_quote`  | error    | quoted field never closes                                                                                                                                                                 |
| `template_row_arity`           | error    | row field count ≠ N                                                                                                                                                                       |
| `template_global_declaration`  | error    | another `@@` token                                                                                                                                                                        |
| `template_clipboard`           | error    | `%` clipboard marker, or `--clip`                                                                                                                                                         |
| `template_session_operator`    | error    | instance is a session form                                                                                                                                                                |
| `template_row_incomplete`      | error    | instance is an editing state (`capture-parse` only)                                                                                                                                       |
| `template_too_large`           | error    | over 1,000 rows or 1 MiB expanded                                                                                                                                                         |
| `template_header_missing`      | warning  | no header, the first item contains `{N}`, and later items exist (`capture-parse` only)                                                                                                    |

Error prefixes, **(decided here)**:

- **Row-scoped errors**: `template row K (line L): …`.
- **Template-scoped errors**: `template line L: …`.
- **Planner errors** on an instance use the row prefix instead of today's
  `capture item K starting on line L:`.
- **Line numbers inside messages.** Instances are parsed with physical line numbers
  mapped to the typed template's lines. So an existing message such as
  `capture line 3 must be a column-zero bullet…` names the right typed line.

## JSON contract (all additive)

### `bob capture` (real runs and `--dry-run`)

- **Top-level `template` object:** `{"line": 1, "field_count": 3, "row_count": 2}`.
  `line` is the header's line.
- **Per-capture `template_row` object:** `{"row": 1, "line": 5}` on each entry of
  `captures[]`.
  - A one-row template keeps the legacy single-result shape. Its flattened top-level
    item carries `template_row`.
- **Drafts without a header** produce byte-identical output to today's.

### `bob capture -E, --expand`

- `-f json` prints `{"ok": true, "expanded": "<text>", "template": {…summary…}}`.
  `template` is omitted for drafts without a header.
- Errors use the same `{"ok": false, "error": "…"}` shape as `bob capture`, with exit 2.

### `bob capture-parse`

- **Top-level `template` object:**

  ```json
  {
    "header": { "range": { "start": 0, "end": 11 }, "line": 1 },
    "range": { "start": 12, "end": 64 },
    "line_start": 2,
    "line_end": 3,
    "field_count": 3,
    "rows": [
      {
        "row": 1,
        "line": 5,
        "range": { "start": 66, "end": 112 },
        "mode": "pomodoro_task",
        "body": "Apply to Foobar!",
        "fields": [
          { "field": 0, "value": "Foobar", "range": { "start": 66, "end": 72 } }
        ]
      }
    ]
  }
  ```

  - `range`, `line_start`, and `line_end` cover the template block.
  - A row's `mode` and `body` are the instance's real mode and body. Both are omitted
    when the row did not expand, for example a row with an arity or quote error.

- **Spans.** New span kinds:
  - `template_header` over the header token;
  - `template_placeholder` over each `{N}`, carrying an additive integer `field` (= N);
  - `template_field` over each lexed row field, quotes included, carrying `field` (its
    0-based position).

  Mis-split rows still get field spans with their positions, so a client can show the
  mis-split. `field` is omitted on every other span, so the JSON of drafts without a
  header is unchanged. Spans stay sorted, non-overlapping, and on UTF-8 boundaries.
  Never reuse `interactive_placeholder`; the Mac close-pending logic depends on it.

- **`items[]`.**
  - Item 1 is the template, read through the masked copy, with its typed range.
  - Each row becomes one more item (`index` = row + 1), with these fields:
    - `mode: "template_row"`;
    - the row's typed range, and `line_start` = `line_end` = the row's line;
    - `body` = the instance body;
    - `route`, `section`, and `block_id` from the instance;
    - **`needs: []`**, because the Mac combines every item's needs into its picker
      decision;
    - an additive `template_row` object, the same shape as `template.rows[k]`.

    A row item never carries fields with instance-coordinate ranges (`dependencies`,
    `blocked_tasks`, `task_complete`, `sub_bullet*`).

- **Needs.**
  - New need `template_row`: the template draft has no rows yet (or no template yet).
  - Top-level needs are the template item's needs plus `template_row` when it applies.
- **`global_destination`** is absent for template drafts. The header never produces
  `invalid_global_destination`.

### `bob capture-complete`

- **The header candidate.** When the cursor is in a `@@fragment` route field that is the
  only token on the draft's first nonblank line, and `template:` starts with the
  fragment (case-insensitive, empty included), the `route` context adds this candidate:

  ```json
  {
    "replacement": "template:",
    "route": "template:",
    "label": "Bulk template: one capture per row below",
    "kind": "template",
    "status": null
  }
  ```

  It is ranked first when the fragment is non-empty and last when it is empty **(decided
  here)**. That way `@@`+Tab keeps accepting a route as it does today.

## Phase `language`: Template language core and `bob capture -E`

**Goal:** add the pure template module and its first consumer, `-E, --expand`. Every
rule in [Normative grammar](#normative-grammar) that needs no capture-item parse lands
here.

1. **New module `src/native/capture_language/template.rs`** (registered in
   `capture_language/mod.rs`). The API is a sketch; the agent may rename:
   - `analyze(raw: &str) -> TemplateAnalysis` returns `NotTemplate` or a framed
     analysis. The analysis holds:
     - the header (range, line);
     - the template block (range, lines, line span);
     - placeholders (index, typed range) and escapes;
     - `field_count`;
     - rows (1-based row number, line, typed range, lexed fields with value and typed
       range);
     - `issues: Vec<TemplateIssue>` in draft order.
   - It also reports a misplaced header in a draft that is otherwise not a template.
   - `TemplateIssue` carries `code() -> &'static str`, `message() -> String`, and a
     typed `range()`. It covers every language-level code in the table: header, missing
     rows, needs separator, missing placeholder, invalid placeholder, gap, row bullet,
     unterminated quote, arity, global declaration, and too large. The message text is
     the single source of truth for both callers, following the module's existing
     `Result<T, String>` convention.
   - `expand(raw) -> Result<Option<TemplateExpansion>, TemplateIssue>` is strict: it
     returns the first issue. `TemplateExpansion` holds the expanded text, one instance
     per row (row number, row line, typed row range, instance text range), the header
     line, `field_count`, and `row_count`.
   - Reuse `split_physical_lines` and the existing tokenizer for `@@` detection. Only
     add API that this phase's consumers use: the lint gate is `-D unused`. Later phases
     extend the module.
2. **`-E, --expand` on `bob capture`** (`src/native/capture/cli.rs`).
   - Help text: "Print the draft with its @@template: expanded, then exit without
     capturing". Add it in alphabetical option order, following the CLI rules.
   - It prints `expand(D)` raw to stdout, with no styling, so it pipes cleanly. For
     drafts without a header it prints the draft unchanged.
   - It never loads the vault or the routing policy, and never plans.
   - Template issues exit 2 with the issue message, in the human or JSON error shape.
   - JSON as in the contract above.
   - Other options are accepted and ignored. The help says it exits before planning.
3. **Tests.**
   - Unit tests in `src/native/capture_language/tests/template.rs` cover every rule in
     the grammar:
     - header variants (case, trailing whitespace, misplaced, repeated, inline,
       `@@template:tsv`);
     - blank lines between the header and the template;
     - template children at both depths, and the separator error;
     - placeholders: repeats, reordering, `{{ }}`, `{}`, `{name}`, `{01}`, `{0:x}`, lone
       braces, unterminated forms, gaps, the 1-based message, none at all, and a
       placeholder inside a block ID;
     - the row lexer: spaces and tabs, `"Two Sigma"`, `“Two Sigma”`, mixed quote styles,
       `\"`, `\\`, `""`, `27"`, `O'Reilly`, closing quotes not followed by space, and
       unterminated quotes;
     - CRLF and bare CR, emoji and accented values, and one-field whole-row values;
     - arity both ways, row bullets, `@@` in the template or a row, and both bounds;
     - Bryan's example expanding byte-for-byte to the expected text.
   - CLI tests in `tests/cli/capture/template_expand.rs` (registered in
     `tests/cli/capture/mod.rs`):
     - human and JSON `--expand`;
     - identity on a plain draft;
     - error exit 2;
     - `--help` lists `-E, --expand` alphabetically (extend
       `tests/cli/help_options.rs`).
4. **Done when** `just check` passes.

## Phase `execution`: Template execution in `bob capture`

**Goal:** `bob capture` and `bob capture --dry-run` capture template drafts per
[the one rule](#the-one-rule), with row-scoped errors and additive output.

1. **Draft parse** (`capture_language/draft.rs`,
   `parse_capture_draft_with_clip_control`).
   - When `template::expand` frames a template, parse each instance on its own through
     the existing split and item parsers.
   - Map the instance's physical line numbers onto the typed template lines, for example
     with a line-offset variant of `split_capture_draft`.
   - Require exactly one item per instance. More than one item means a session chain:
     `template_session_operator`.
   - Apply the refusals from
     [Draft-wide refusals and bounds](#draft-wide-refusals-and-bounds):
     - clipboard: parse instances with clipboard markers recognized whatever `--no-clip`
       says;
     - session kinds: `PomodoroAdjust`, `PomodoroShift`, `PomodoroStart`,
       `PomodoroClose`, and a `start` or `close` spec on `Pomodoro` or `PomodoroLink`.
   - Remap each `ParsedCaptureItem`'s `start`, `end`, `line_start`, and `line_end` to
     its typed row.
   - Add an additive `template_row: Option<…{row, line}>` to `ParsedCaptureItem`, and a
     `template` summary to `ParsedCaptureDraft`.
   - Template drafts have no `@@` global, so the global-destination code paths are
     untouched.
2. **Planner errors** (`capture/plan.rs`). Build the error prefix from `template_row`
   when present (`template row K (line L): …`). The two
   `capture item … starting on line …` sites share one helper.
3. **Request checks.** A template draft with `-c, --clip` fails with
   `template_clipboard`. Forced destination flags (`-r`, `-s`, `-t`, `-S`) apply to
   every instance, as they do for any batch. `-p, --pull-in` works as it does for any
   batch.
4. **Output** (`capture/output.rs`).
   - JSON adds the top-level `template` summary and the per-item `template_row`. Reuse
     the summary type that the language phase added for `--expand`.
   - Human output prints a dim summary line before the items, styled like the existing
     `global  foo.md` line: `template  3 rows · lines 5–7`. The usual `1/N` lines
     follow. `1/N` numbering equals row numbering.
5. **Help.** Add one paragraph to `bob capture --help`'s long help after the multi-item
   paragraph. It covers the header, `{N}` and `{{ }}`, the rows and quoting rules, exact
   arity, the one rule, `-E`, and the refusals. Add one example, for instance:
   `printf '@@template:\nApply to {0}! @job^apply-{1}\n\nFoobar foo\n"Two Sigma" two-sigma\n' | bob capture`.
6. **Docs.**
   - Add a new `docs/capture.md` § **Bulk templates**, right after § Multi-item capture,
     with a Contents entry. It covers:
     - the example and its expansion;
     - the header, template, placeholders, rows, and quoting, with a small table;
     - what values can do, and the `:` vs `^` trade-off;
     - the refusals;
     - the error-code table with example messages;
     - human and JSON output;
     - `-E, --expand`.
   - Add a `@@template:` row to the grammar-at-a-glance table.
   - Update § Command-line options and § Input, stdin, and JSON output for `-E` and the
     JSON fields.
   - In `README.md` § Capture, add a sentence to the multi-line paragraph and a
     `@@template:` row to its marker table.
7. **Tests.**
   - **Equivalence oracle.** Add `src/native/capture/tests/template.rs`, using the
     existing capture test vault helpers. For each golden template draft,
     `plan_capture_batch(D)` and `plan_capture_batch(expand(D))` must stage identical
     file contents and identical item results once `template_row` is stripped.
     - Run both with the same `BOB_NOW`, config, day file, and `BOB_PRIORITY_ROLL_SEED`.
     - Cover `@route`, `@route^{1}`, `@route:{1}`, `@route+id`, `#section`, `s:N`,
       `p:N`, `&note:id`, `*note:id`, `!note:{0}`, `^route:{0}`, children with
       placeholders, and bare-URL rows (reference jobs).
   - **CLI tests** (`tests/cli/capture/template.rs`):
     - Bryan's example, dry-run and real: exact titles, IDs, URL child bullets, order,
       and Task Links;
     - JSON `template` and `template_row`, including a one-row template;
     - human summary;
     - atomicity: a bad ID on row 5 writes nothing for rows 1–4;
     - duplicate IDs across rows refused with a row prefix;
     - `%` refused with and without `--no-clip`, and `--clip` refused;
     - `URL %` listen rows allowed;
     - session forms refused;
     - a `capture line N` message that names the typed template line;
     - drafts without a header are byte-identical: assert that one existing batch golden
       is unchanged.
8. **Done when** `just check` passes.

## Phase `parse`: Template-aware `capture-parse`

**Goal:** the editor protocol understands template drafts. Every range points into the
typed draft, rows are opaque data that never open pickers, and template problems arrive
as precise diagnostics.

1. **Masked copy.** In `template.rs`, add `masked(raw) -> String`, the same length as
   the typed draft:
   - header bytes and row bytes become ASCII spaces, keeping line terminators;
   - each `{N}` becomes the same number of `x` bytes, and `{{` / `}}` become `xx`;
   - an unterminated `{` or `{digits` running to whitespace or the end of the line is
     masked the same way, so it raises only `template_invalid_placeholder` and no
     `invalid_*_block_id`.

   `parse_for_editor_with(masked)` then yields exactly one item, the template. Its
   spans, needs, and diagnostics already index the typed draft. Wikilink spans are
   computed on the masked copy, never on rows.

2. **Template-draft branch** in `capture_parse.rs` (`CaptureParseResult::new`) or in
   `parse_for_editor_with`, whichever keeps one implementation.
   - **Spans.**
     - Emit `template_header`.
     - Take the masked template spans, merged with wikilink spans as today, and split
       every span that contains a placeholder around it. Emit `template_placeholder`
       spans carrying `field`. Drop zero-width pieces.
     - Drop any `interactive_placeholder` span that overlaps a placeholder.
     - Add `template_field` spans for every lexed field of every row.
     - Use a wire-level span struct in `capture_parse.rs` with an optional `field`. Do
       not add a field to the internal `Span`, which is built at about 145 sites.
   - **Template item.**
     - `mode`, `route`, `section`, and `block_id` come from the masked parse.
     - `body` shows placeholders as typed (`Apply to {0}!`), never mask bytes. Body
       normalization only drops marker tokens and collapses whitespace, so rebuild it
       from the same token ranges over the typed text.
     - **Needs:**
       - drop `pomodoro_id` and `block_id`: block-ID slots after `:` and `^` are per-row
         in a template, and the Block ID Picker would trap typing `{1}`;
       - clear all template needs when an `interactive_placeholder` span overlapped a
         placeholder (a query such as `!{0}`, `:{0}`, or `+{0}` is never a picker);
       - otherwise keep needs: `&`, `*`, `+`, route, section, and Pomodoro-name pickers
         still help write the template's constant parts.
   - **Rows.** For each row that expanded, parse the instance with
     `parse_for_editor_with` (same options) and build the row item and
     `template.rows[k]` as specified above. Rows that did not expand get an item with
     `body: ""` and no `mode` in their `template_row`.
   - **Diagnostics,** in this order:
     1. language issues from `analyze`, with typed ranges;
     2. masked-template diagnostics that do not touch a placeholder;
     3. instance diagnostics mapped through a per-instance source map, recorded during
        expansion as segments (literal bytes map to their typed template bytes, and
        inserted values map to their field). A range over literal bytes only maps to the
        typed template range. A range touching exactly one value maps to that field's
        range, and anything else maps to the row's range. Deduplicate on (code, range,
        message), so one template-literal problem is reported once, not once per row;
     4. refusal diagnostics (`template_clipboard`, `template_session_operator`), checked
        on the masked template and on each instance, mapped and deduplicated the same
        way;
     5. `template_row_incomplete` on the row whose instance has needs or mode
        `incomplete`;
     6. `template_missing_rows` last, so a real error shows first in the Mac status
        line, which shows only the first non-info diagnostic.
   - **Needs:** add `template_row` to the top level when there is no template or there
     are no rows. `global_destination` is `None`.
   - **Top-level `template` object** and the per-item `template_row`, as specified in
     the JSON contract.
3. **Header-less hint.** On drafts without a header: when the first item's text contains
   a `{digits}` placeholder and the draft has at least two items, emit a
   `template_header_missing` **warning** ahead of the other diagnostics, ranged over the
   first item. Message: "add `@@template:` as the first line to use this item as a
   template for the lines below". `bob capture` keeps today's behavior for these drafts.
4. **Human output** of `bob capture-parse`: add a `template` field, for example
   `3 fields · 2 rows`. Template spans and diagnostics print through the existing
   renderers.
5. **Docs.** In `docs/capture.md` § `bob capture-parse`, document:
   - the new span kinds and `field`;
   - the `template_row` mode and need;
   - the `template` object;
   - every diagnostic code, and the mapping and deduplication rules;
   - the masking caveat: a placeholder that changes a token's meaning per row, such as
     `p:{0}`, is highlighted as the mask reads it; the per-row parses still report real
     problems.

   Link § Bulk templates.

6. **Tests.** Extend `src/native/capture_language/tests/` and
   `tests/cli/capture/parse.rs` (or a new `parse_template.rs`):
   - golden JSON for Bryan's example: spans, items, `template` object, no
     `global_destination`, and no `invalid_global_destination`;
   - span invariants: sorted, non-overlapping, on char boundaries, multi-byte values,
     and a placeholder splitting a route or block-ID span;
   - rows never carry needs; a value such as `:dee` gives `template_row_incomplete`;
   - `!{0}` and `:{0}` templates give no needs;
   - `@job:` typed before `{1}` gives no `pomodoro_id`;
   - a constant `&` picker still gives `task_dependency`;
   - `template_row` need in both the header-only and the no-rows state;
   - an arity error ranged on the row;
   - a bad block-ID value ranged on its field;
   - a template-literal error reported once across three rows;
   - the header-less warning;
   - drafts without a header are byte-identical.
7. **Done when** `just check` passes.

## Phase `complete`: Template-aware completion and rewrite

**Goal:** completion helps write the template and never fires on rows. Rewrites never
touch template drafts. The new header is discoverable from `@@`.

1. **One gate, shared by `capture_complete/engine.rs` and `capture_complete/shell.rs`.**
   For a template draft:
   - The cursor in the header line, in the rows region, or inside or touching a
     placeholder → empty success (`context: null`).
   - Otherwise, run the normal completion on the masked copy; offsets are unchanged.
     Return empty when:
     - the replacement range overlaps a placeholder (for example a block-ID candidate at
       `:{2}` would overwrite `{2}`);
     - the context is `pomodoro_block_id`, `task_block_id`, or `project_task_block_id`
       (per-row slots, as in the parse phase).
   - Route, section, Pomodoro-name, `&`, `*`, `+`, and wikilink completion in the
     template keep working. `capture_block_ids.rs` must see the masked text too.
2. **`template:` candidate,** as specified in the JSON contract. `RouteCandidate.kind`
   is a `CaptureTargetKind` today. Serialize `"template"` additively, either with a
   small string-backed kind or a new variant if its exhaustive matches stay few. The
   JSON of every existing candidate must not change.
3. **`capture-rewrite`.** For a template draft, return no rewrite (`changed: false`, no
   rule, no edits): no `@@` absorption and no `:`/`^` toggle.
4. **Docs.** Update `docs/capture.md` § `bob capture-complete` (template gate, block-ID
   suppression, `template:` candidate) and § `bob capture-rewrite` (template drafts are
   never rewritten). Update `README.md`'s `capture-rewrite` row if its wording becomes
   incomplete.
5. **Memory,** following `/sase_memory_write`:

   > [!decision] memory_bulk_capture_glossary
   - Update `glossary:bulk-capture` so it names `@@template:` alongside `@@route` as the
     draft-wide forms.
   - Say that a template draft's later lines are rows, one capture per line, not
     blank-line items.
   - Say that capturing it is exactly capturing its expansion.
   - Keep the strand's length about the same.

   > [!decision] memory_template_decision
   - Add a `decisions` strand, "Bulk Templates Are A Declared Expansion Before Parsing".
   - Claim: `capture(D) ≡ capture(expand(D))`; an explicit `@@template:` header, never
     implicit `{N}` detection; expansion only in bob.
   - Evidence: the research report and `docs/capture.md` § Bulk templates.
   - Reopens when: the header proves too costly in daily use.

   Run `sase memory init` after either change. When a decision is off, record a
   `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

6. **Tests.** Extend `tests/cli/capture/complete_editor.rs` (or add
   `complete_template.rs`):
   - a cursor in a row or the header → null;
   - a block-ID context at `@job:` or at `:{1}` → null;
   - route completion after `@` in the template works;
   - a constant `&` completion works;
   - `@@te` on line 1 offers `template:` first;
   - `@@` offers it last;
   - `@@te` on line 3 does not offer it;
   - rewrite on a template draft is a no-op.
7. **Done when** `just check` passes.

## Phase `mac`: Bob Mac Capture template presentation

**Goal:** writing a template in the panel feels designed:

- the header reads as a directive;
- each placeholder and its row fields share a color, so a mis-split row is visible at a
  glance;
- an unfinished template is calm, not red;
- problems are underlined where they are;
- the preview says it came from a template.

Open the repo with `sase repo open bob-mac-capture` and read its `AGENTS.md` first. Its
compile rules are binding: never land Swift you have not compiled.

1. **Models** (`Sources/CaptureCore/CaptureModels.swift`). Decode everything tolerantly
   (`(try? …) ?? nil`), with defaulted init parameters, so older bob still works:
   - `CaptureSpan.field: Int?`;
   - `CaptureParseResponse.template` (header, range, lines, `field_count`, rows with
     fields);
   - `CaptureParseItem.templateRow`;
   - `CaptureCommandSuccess.template` (`line`, `field_count`, `row_count`);
   - `CaptureCommandSuccess.templateRow` (`row`, `line`).

   Store the parse-side `template` on the panel model in `applyParse`.

2. **Highlighting.**
   - `template_header` maps to the `.route` category, the same family as `global_route`,
     so all `@@` directives look alike.
   - `template_placeholder` and `template_field` map to a new slot category colored by
     `field % 6` from a six-hue slot palette in `CaptureEditorPalette`. `{k}` and every
     row's k-th field share one hue.
   - Start from orange, teal, pink, green, purple, and brown. Tune them in light and
     dark render fixtures for contrast; adaptive system colors only.
   - A template span without `field` stays neutral. Unknown kinds keep falling back to
     neutral.
   - Keep `validatedSpanRanges`' all-or-nothing guard. Bob guarantees valid spans.
3. **Calm unfinished state.** Treat the `template_row` need like the picker needs in
   `pickerNeed(in:)` / `applyQuietIncompletePicker`: clear the preview, skip the doomed
   dry run, and show bob's `template_missing_rows` diagnostic message as the status, in
   the calm incomplete style, never error red. Bob owns the wording.
4. **In-place diagnostics.** When the parse has a `template` object, underline every
   non-info diagnostic's range in the editor with a thin red underline. The app ignores
   diagnostic ranges today. Template ranges always point into the typed draft, so this
   is safe. Drafts without a header keep today's rendering exactly.
5. **Preview chip.** When the dry run has `template`, show a capsule chip
   `Template · N rows` (`N` = `row_count`, singular for 1) beside the existing
   `N captures` header in `PreviewPane.previewContent`. Style it like the plan-budget
   delta chip. Give it the accessibility label "N capture items from a template". Cards,
   the footer title (**Capture**), and notifications are unchanged.
6. **Header candidate row.** A `route`-context candidate with `kind == "template"`
   renders with:
   - its own SF Symbol, for example `square.stack.3d.down.right`;
   - the primary text `@@template:`;
   - bob's label as the secondary text;
   - no badges;
   - the accessibility hint "Starts a bulk template: one capture per row".

   Accept inserts `replacement` verbatim. Exclude these candidates from the `+`
   route-commit shortcut (`commitRouteCompletionOnPlus`).

7. **Expand action.**

   > [!decision] mac_expand_action

   Add an "Expand template" editor action that replaces the draft with the expansion, as
   one undoable edit, using `bob capture -E -f json`.
   - Bind it to a key that is unbound today, per the README keyboard table.
   - It is enabled only when the parse has a `template` object.
   - On error it shows bob's message and leaves the draft unchanged.
   - Document it in the README keyboard table.

   When this decision is off, skip the step and record a `PROPOSED FOLLOW-UP:` note on
   this phase's bead.

8. **Fixtures and tests.**
   - Record real fixtures with a bob built from this epic's bob-cli master
     (`cargo build` in the bob-cli workspace), following the recipe in the header
     comment of `Tests/CaptureCoreTests/CaptureRefPresentationTests.swift`:
     - `capture-parse` for Bryan's example, the header-only and no-rows states, and an
       arity error;
     - `capture --dry-run --no-clip -f json` for the example and for a one-row template;
     - `capture-complete` at `@@te`.
   - Wire them into `Tests/Fixtures/fake-bob`.
   - Add decoding tests (`CaptureModelTests`), span-category and slot-color tests
     (`CompletionRowContentTests`), and presentation tests for the chip, the calm
     `template_row` state, and the underline. Add light and dark render fixtures for a
     template draft.
9. **README.** Update:
   - the required-bob-features list (older bob decodes as absent);
   - the highlighting legend (slot hues);
   - the needs and quiet-state text;
   - the preview description (chip);
   - the completion text (header candidate);
   - the compatibility notes (old app + new bob shows neutral spans; any app + old bob
     refuses the header and writes nothing).
10. **Verification,** per `AGENTS.md`:
    - run `just portable-check` (on athena if `swift` is missing; read
      `sase memory read tailnet.md` first);
    - make the best-effort scratch-copy compile on the mac with `bundle.sh` and
      `just typecheck-tests`;
    - make sure CI is green.

    If the mac is offline, say in the final report that the macOS targets are
    unverified.

## Out of scope and follow-ups

These are not part of this epic. Phase workers record them as `PROPOSED FOLLOW-UP:`
notes on their own beads when they run into them.

1. **A `{N|slug}` filter.** It would derive `two-sigma` from `"Two Sigma"`, removing the
   slug column: `@job^apply-{0|slug}`. This is the most valuable follow-up. It should
   reuse the block-ID suggestion slugger.
2. **Per-row child bullets.** The syntax is already reserved by `template_row_bullet`.
3. **Markdown pipe-table rows,** if LLM tables become the source of rows.
4. **Named placeholders and defaults** (`{company}`, `{0:default}`), already reserved.
5. **Saved or reusable templates** (snippet or vault note).
6. **Implicit header-less templates.** Reopen only if the 12-character header chafes in
   real use; the `template_header_missing` hint already points the way.
