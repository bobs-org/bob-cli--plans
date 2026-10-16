---
tier: epic
title: Saved capture templates (the Control-T template shelf)
goal: 'Every successful `@@template:` capture saves its template block to a bob-owned
  shelf on disk, and Bob Mac Capture''s Control-T opens a stash-style shelf of those
  templates: picking one checks it out into the draft, capturing puts it back on top
  (with your edits), abandoning leaves it untouched, and nothing is ever silently
  lost.

  '
decisions:
  pop_visible_in_count:
    ask: Should the Templates count leave out a checked-out template, so a pick visibly
      pops it until it settles?
    default: true
    why: it makes the requested pop feel literal while the copy on disk stays safe
    answer: true
  memory_saved_templates_decision:
    ask: Should the last phase add a decision record that saved capture templates
      are bob-owned and checked out, not popped?
    memory:
    - decisions
    default: false
    answer: false
  memory_saved_template_glossary:
    ask: Should the last phase add a glossary strand for Saved Template (alias template
      shelf)?
    memory:
    - glossary
    default: false
    answer: false
phases:
- id: store
  title: Template shelf store and auto-save in bob capture
  depends_on: []
  size: medium
  description: 'store: add the bob-owned template shelf (canonical block text, content-hash
    ids, locked atomic JSON at $XDG_STATE_HOME/bob-cli/capture/templates.json), save
    after every successful real template capture, the hidden --template-origin replace,
    the capture.max_saved_templates config key, additive JSON and human output, the
    A10 missing-rows wording, and docs.'
- id: protocol
  title: bob capture-templates protocol command
  depends_on:
  - store
  size: medium
  description: 'protocol: add the `bob capture-templates` group (list default, show,
    save, delete, clear) with ready-to-insert drafts, cursors, and capture-parse spans
    in JSON, colored human output, definition-only validation for save, help, completion,
    and docs.'
- id: mac-core
  title: Bob Mac Capture saved-template models, client, and fixtures
  depends_on:
  - protocol
  size: medium
  description: 'mac-core: decode the shelf protocol in CaptureCore, add BobProcessClient
    list/save/delete/clear plus an identity-gated --template-origin, shared picker
    helpers, the row presentation builder, the environment-key fix, recorded fixtures,
    and fake-bob routes.'
- id: mac-shelf
  title: Bob Mac Capture Control-T template shelf
  depends_on:
  - mac-core
  size: medium
  description: 'mac-shelf: the Control-T shelf itself: cache, opening rules, keys,
    syntax-colored rows with slot dots, one-undo-step insert (empty and rows-first
    drafts), two-press delete, Control-T/Control-S switching, and the Templates footer
    button.'
- id: mac-checkout
  title: Template checkout lifecycle, Command-S save, Settings, and docs
  depends_on:
  - mac-shelf
  size: medium
  description: 'mac-checkout: track the checked-out origin, release it on every abandon
    path, pass --template-origin on submit, announce save outcomes, add Command-S
    Save template, the Settings section, README docs, and the memory decisions.'
proposed_by: bbugyi200.apollo.research.1n.linker.w0
decided_by: auto
create_time: 2026-10-16 08:18:35
status: wip
bead_id: bob-cli-8b
---

- **PROMPT:** [prompts/202610/saved_capture_templates.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/saved_capture_templates.md)
- **BEAD:** [bob-cli-8b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8b/README.md)

# Plan: Saved capture templates (the Control-T template shelf)

## Why and what

Epic `bob-cli-7w` shipped `@@template:` drafts. Bryan's job-application waves reuse the
same template every time:

```text
@@template:
Apply to {0}! @job^apply-{1}
- **URL**: {2}

Foobar foo https://foobar.com/careers/1
"Two Sigma" two-sigma https://www.twosigma.com/careers/2
```

A successful capture clears the draft, so that template is thrown away every time it
works. This epic keeps it. The **template shelf** is a small, bob-owned library of
recently used template blocks. Bob Mac Capture opens it with **Control-T**, a sibling of
the Control-S stash.

This plan turns the merged research report
`research:202610/capture_template_shelf/capture_template_shelf.md` into a contract and
phases. Bryan accepted every requirement in it (adjustments A1–A10 and the recommended
answers to its open questions). Where the report left something open, this plan decides
it; those choices are marked **(decided here)**.

### The rules everything follows

1. **bob owns the shelf.** A saved template is a capture-language object. Deciding which
   lines form it, how it is normalized, and whether it is valid are grammar answers
   (decision `decisions:mac-capture-is-a-thin-client`). The Mac app renders what bob
   returns and never parses `@@template:` itself.
2. **Save the template block, never the rows.** Rows carry one-off private values
   (companies, URLs) and would replay old captures.
3. **A use is a successful, real `bob capture` of a template draft.** It is never a
   parse, a preview, `--dry-run`, `-E`, or a failed capture. One save per invocation,
   not one per row.
4. **Pop is presentation; checkout is storage.** Picking a template takes it off the
   shelf _in the app_, but the copy on disk is never deleted by a pick. A successful
   capture puts the used version back on top. If you edited it, the edit **replaces**
   the original in the same locked write. Abandoning the draft leaves the original
   untouched. So the useful part of pop (edits replace instead of piling up
   near-duplicates) stays, and its harmful part (the only copy vanishing on Discard,
   Control-C, quit, or a crash) is gone.
5. **Saving is best-effort and never fails a capture.** The vault write is the primary
   effect. A shelf failure is a warning, the capture still exits 0, and under checkout
   the original still exists, so failures end in "nothing lost".
6. **Additive protocol only.** `schema_version` stays `1` everywhere. Drafts without a
   header produce byte-identical output to today's.

### Lifecycle

```text
                Control-T, pick A
   shelf ──────────────────────────▶ checked out: A's block in the draft,
     ▲                               origin = A (app memory only), A still on disk
     │                                   │
     │  abandon: Discard · Control-C ·   │  capture succeeds: bob saves the used block;
     │  editor emptied · Command-Z of    │  if it changed, bob removes A in the same
     │  the insert · header deleted ·    │  locked write
     │  quit · crash                     ▼
     └────────── A unchanged ◀──── A (or edited A′) on top of the shelf
```

What each event does:

| Event                                                                    | Draft                                                                     | Shelf on disk                                   |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------- | ----------------------------------------------- |
| Pick A                                                                   | block inserted (one undo step), caret on the first row line, `origin = A` | untouched                                       |
| Escape or hide                                                           | retained                                                                  | untouched; checkout kept in memory              |
| Capture fails                                                            | kept, with bob's error                                                    | untouched                                       |
| Capture succeeds, A unchanged                                            | cleared                                                                   | A moved to the top, `use_count + 1` (`updated`) |
| Capture succeeds, edited to A′                                           | cleared                                                                   | A′ saved on top, A removed (`replaced`)         |
| Command-S in the template draft                                          | kept                                                                      | A′ saved, A removed; `origin` becomes A′        |
| Discard, Control-C, editor emptied, Command-Z of the insert              | —                                                                         | checkout released; A untouched                  |
| Fresh parse reports no template (header deleted, a future Expand action) | —                                                                         | checkout released immediately                   |
| Quit or crash                                                            | retained draft lost (existing behavior)                                   | A untouched                                     |

**Control-C on a checked-out draft** stashes the full draft without its origin. A later
restore-and-capture saves by content, so an edited version lands _beside_ A. That is
accepted; the stash schema does not change.

> [!decision] pop_visible_in_count
>
> When on, the footer's **Templates N** count and the shelf list leave out the
> checked-out entry while the checkout is active. The count drops by one on pick and
> comes back when the capture or the abandon settles it. When off, the count always
> equals the number of entries bob lists. Either way, disk behavior is identical.

## Contract: bob-cli

Every phase implements this section. Where a later phase needs a detail, this section is
the source of truth.

### The store

- **Path.** `bob_cli_state_dir()/capture/templates.json`, which is
  `${XDG_STATE_HOME:-~/.local/state}/bob-cli/capture/templates.json`. The environment
  variable `BOB_CAPTURE_TEMPLATES_FILE` overrides the exact file path, following the
  `BOB_VAULT_SYNC_STATE_FILE` precedent. The store is per machine and not per vault,
  like the ref-job spool.
- **Sibling files** (decided here; they work with the override):
  - lock: `<file>.lock` (for example `templates.json.lock`);
  - quarantine: `<stem>.corrupt.json` (for example `templates.corrupt.json`).
- **Permissions.** Directory 0700, files 0600. Reuse `create_state_lock_file` for the
  lock.
- **Format** (pretty JSON, trailing newline):

  ```json
  {
    "schema_version": 1,
    "templates": [
      {
        "id": "9c1f0e2a7b3d",
        "text": "Apply to {0}! @job^apply-{1}\n- **URL**: {2}",
        "first_saved_at": "2026-10-16T09:12:44-04:00",
        "last_used_at": "2026-10-16T09:40:02-04:00",
        "use_count": 3
      }
    ]
  }
  ```

- **Canonical `text`.** The lines of the analyzed template block (`template::analyze`),
  header excluded. Strip each line of trailing ASCII spaces and tabs and join the lines
  with `\n`. There is no trailing newline. Indentation, braces, escapes, quotes,
  Unicode, and literal text are preserved byte for byte. `TemplateBlock.lines` is
  private today; add a `pub(crate)` accessor rather than re-splitting the draft.
- **Identity.** `id` is the first 12 lowercase hex digits of SHA-256 (`sha2` is already
  a dependency) over the UTF-8 canonical text. Identical templates are the same entry,
  so deduplication is free and `delete` is idempotent. An edit yields a new id.
- **Order.** Most recently touched first. Every save moves its entry to index 0.
- **Times.** RFC 3339 with the local offset, at seconds precision, from bob's existing
  clock (it honors `BOB_NOW`, so tests are deterministic).
- **`use_count`** counts successful captures only. A Command-S save of a new template
  starts at 0, and a save never increments it.
- **Capacity.** The config key `capture.max_saved_templates` (decided here; matches
  `plan.max_*`) is an integer from 0 to 36 and defaults to 36. 36 fills the stash's
  one-key accelerator alphabet. Every write trims the list to the cap by dropping the
  least recently used entries. `0` turns saving off: bob never writes the shelf except
  for an explicit `delete` or `clear`, and existing entries stay listed and usable. An
  invalid value or a config read error prints one stderr warning and uses 36; it never
  fails a capture. Put the loader in `src/native/config/capture.rs` next to the other
  config modules.
- **Writes.** Take an exclusive `fs2` lock on the lock file, polling
  `try_lock_exclusive` for up to 2 seconds before giving up (`busy`). Then read, modify,
  write a 0600 temp file in the same directory, fsync it, rename it over the file, and
  fsync the directory. When the list becomes empty, remove the file instead.
- **Reads** (`list`, `show`) take no lock. The atomic rename guarantees a consistent
  snapshot. A missing file is an empty shelf.
- **Forward compatibility.**
  - Unknown top-level and per-entry fields are preserved on rewrite (a flattened
    `serde_json::Map`), so later additive fields such as `name` or `pinned` survive an
    older bob.
  - A **newer `schema_version`** is read-only for this bob. Reads decode what they can
    (`id` and `text` are required) and add a warning. Writes other than `clear` refuse
    and leave the file untouched.
  - An **unreadable file** (bad JSON or a missing required field) is reported as a
    warning plus an empty shelf on reads. The next write renames it to the quarantine
    path (replacing an older quarantine), warns once, and starts empty. The bytes are
    kept, matching the stash.
- **Replace (`origin`).** A save may carry an origin id `O` for the new id `N`:
  - `O == N`, or `O` is not on the shelf: a plain save (`new` or `updated`).
  - `O != N` and `O` is present: remove `O`. If `N` already exists, bump it and keep its
    own fields. Otherwise insert `N` inheriting `O`'s `first_saved_at`, `use_count`, and
    unknown fields, so a template keeps its history and any future name or pin through
    edits. Status `replaced`.
  - The replace happens inside the same locked write, so nothing is ever half done. A
    concurrent delete of `O` just makes the replace a no-op.

### `bob capture` additions (all additive)

- **When it saves.** After `commit_capture_batch` succeeds (and after the existing
  best-effort `ref_jobs::kick()`), on a run without `--dry-run`, when
  `batch.template.is_some()` and the cap is above 0. bob re-runs `template::analyze` on
  the draft text, canonicalizes, and saves. Verify that `CaptureRequest.raw_text` is the
  typed draft at that point; if planning rewrites it, keep the original before planning.
- **Never touches the store:** `-E`, `--dry-run`, failed or refused captures,
  `pull_in_choice_required`, drafts without a header, `capture-parse`,
  `capture-complete`, and `capture-rewrite`. None of them may create the state
  directory.
- **Hidden `--template-origin <ID>`** is an internal subprocess argument: hidden from
  help, no short alias (the CLI rules exempt internal arguments), and kept in
  alphabetical order in the builder. Any non-empty ASCII alphanumeric value of at most
  64 characters is accepted; anything else is a usage error (exit 2). It is ignored on
  non-template drafts, on `--dry-run`, and when saving is off.
- **JSON.** The top-level `template` object gains:
  - `saved: {"id": "9c1f0e2a7b3d", "status": "new" | "updated" | "replaced"}` when the
    save succeeded;
  - `not_saved: "<reason>"` when a save was attempted and failed.

  Both are absent on `--dry-run` and when saving is off. The batch `warnings` array is
  not touched.

- **Failure.** Print `bob capture: warning: template not saved: <reason>` on stderr in
  both output formats, the same way the `ref_jobs::kick` warning is printed. The exit
  code stays 0.
- **Human output.** The template summary line gains one segment:
  `template  3 rows · lines 5–7 · saved`. It is ` · saved` for `new` and `updated`,
  ` · saved, replacing 9c1f0e2a7b3d` for `replaced` (dim id), and ` · not saved`
  (yellow) on failure. There is no segment when saving is off or on dry runs.

### A10: the missing-rows wording

Today `template_missing_rows` at the end of a template always says "add a blank line,
then one row per capture". That is wrong in exactly the state every shelf pick produces,
where the blank separator already exists. When the separator line exists, the message
becomes (decided here):

- `add one row per capture: 3 values separated by spaces` when `field_count > 1`;
- `add one row per capture: one value per row` when `field_count == 1`;
- `add one row per capture` when there are no placeholders yet.

Without a separator, the old text stays. The header-only message ("write the template on
the next line, then a blank line and one row per capture") is unchanged. Update the
diagnostics table in `docs/capture.md`.

### `bob capture-templates` (Capture protocol)

A new `capture-*` sibling, allowed by the CLI rules ("the `capture-*` protocol names are
frozen siblings that may only grow additively"; never a subcommand under `bob capture`).
It is a `Target::Group` in `src/runner.rs`, following the `bob stats` precedent, with
members in alphabetical order. Bare `bob capture-templates` runs the flag-only,
read-only `list`, which the CLI rules allow.

```text
bob capture-templates [list] [-f, --format human|json]
bob capture-templates show <TEMPLATE> [-f, --format human|json]
bob capture-templates save [-f, --format human|json] [TEXT]...   # TEXT or stdin
bob capture-templates delete <TEMPLATE>... [-f, --format human|json]
bob capture-templates clear [-f, --format human|json]
```

- **`<TEMPLATE>`** is a 1-based index from `list` (all digits, at most two characters)
  or an id (decided here).
- **Error JSON** uses the capture shape: `{"ok": false, "error": "…", "code": "…"}`. The
  codes are:
  - `saved_template_not_found` (exit 1);
  - `saved_template_not_template` (exit 2): "start the draft with `@@template:` to save
    it as a template";
  - `saved_templates_disabled` (exit 2): "saving templates is off
    (capture.max_saved_templates: 0)";
  - `saved_templates_newer_schema` (exit 1);
  - `saved_templates_busy` and `saved_templates_io` (exit 1);
  - the existing `template_*` codes for invalid definitions (exit 2).

**`list -f json`** gives the app everything it needs to render and insert _without
parsing_:

```json
{
  "ok": true,
  "schema_version": 1,
  "capacity": 36,
  "saving": true,
  "file": "/Users/bryan/.local/state/bob-cli/capture/templates.json",
  "templates": [
    {
      "index": 1,
      "id": "9c1f0e2a7b3d",
      "text": "Apply to {0}! @job^apply-{1}\n- **URL**: {2}",
      "draft": "@@template:\nApply to {0}! @job^apply-{1}\n- **URL**: {2}\n\n",
      "cursor": 57,
      "field_count": 3,
      "line_count": 2,
      "route": "job",
      "spans": [{ "start": 9, "end": 12, "kind": "template_placeholder", "field": 0 }],
      "first_saved_at": "2026-10-16T09:12:44-04:00",
      "last_used_at": "2026-10-16T09:40:02-04:00",
      "use_count": 3
    }
  ],
  "warnings": []
}
```

- `draft` is the text to insert: `@@template:\n` + `text` + `\n\n`. `cursor` is the
  UTF-8 byte offset of the caret after an insert into an empty draft (today always
  `draft.len()`: the empty row line below the blank separator). Both are decided by bob,
  so a later layout change needs no app change.
- `spans` are exactly what `capture-parse` reports for `draft`, restricted to the
  template text and shifted to index into `text`. They use the same wire shape (`start`,
  `end`, `kind`, optional `field`). bob computes them in process with the capture-parse
  builder (pure, no vault), without URL routing. At 36 entries this costs microseconds.
  `route` is item 1's route from that same parse, or `null`.
- `capacity` is the configured cap and `saving` is `capacity > 0`. `file` makes any
  environment divergence visible. `warnings` covers unreadable files, newer schemas, and
  config warnings.
- New entry fields are additive; the app decodes them with `decodeIfPresent`.

**Human `list`** is designed to be read (Styler colors, `NO_COLOR` respected):

```text
Saved templates · 3 of 36 · ~/.local/state/bob-cli/capture/templates.json

 1  Apply to {0}! @job^apply-{1}
    - **URL**: {2}
    3 fields · job · used 14× · 2h ago · 9c1f0e2a7b3d

 2  Read {0} @reading^{1} p:2
    2 fields · reading · used 5× · 3w ago · 4be17c0d9a21
```

- Each `{k}` is painted in slot hue `k % 6`, in the same order as the Mac palette:
  orange, teal, pink, green, purple, brown (256-color codes through `Styler::paint`).
- The route token uses the existing route color. The metadata line and id are dim.
- A metadata line for `use_count == 0` reads `saved 2m ago`.
- The empty shelf prints: "No saved templates yet. Capture a `@@template:` draft and it
  lands here."
- With saving off, the header adds `saving is off (capture.max_saved_templates: 0)`.

**`show`** prints the raw insertable `draft` to stdout, unstyled, so it pipes cleanly.
With `-f json` it prints `{"ok": true, "template": {…list entry…}}`.

**`save`** reads one draft (TEXT arguments joined like `bob capture`, else stdin) and
saves its block with the same origin semantics, through its own hidden
`--template-origin`.

- It validates the **definition only**. Add a helper to `template.rs` that returns the
  first definition-scoped issue in draft order:
  - any issue inside the header or block (misplaced header, `template_needs_separator`,
    `template_invalid_placeholder`, `template_placeholder_gap`, a block-level
    `template_global_declaration`);
  - `template_missing_placeholder`, checked even when there are no rows;
  - a header with no block, which reports the existing header-range
    `template_missing_rows`.

  Row issues (`template_row_*`, quotes, arity, `template_too_large`, end-of-block
  `template_missing_rows`) are ignored, so the calm no-rows state saves fine.

- Instance-level refusals (`%`, session operators) are not checked; such a template
  fails at capture time with its existing message. This is accepted.
- `save` never bumps `use_count`.
- JSON: `{"ok": true, "saved": {"id", "status"}, "template": {…list entry…}}`.
- Human: `✓ saved template 9c1f0e2a7b3d · 3 fields · job`.

**`delete`** is idempotent. Indexes resolve under the lock. JSON:
`{"ok": true, "deleted": ["…"], "missing": ["…"]}`. Human: `✓ deleted 1 template`.

**`clear`** removes the file and its quarantine copy (the lock file stays), even under a
newer schema; it is the explicit escape hatch. JSON: `{"ok": true, "cleared": 3}`.

## Contract: Bob Mac Capture

### Keys

| Key                              | In the editor                          | Template shelf open                                                                                                             |
| -------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Control-T                        | Open the shelf (see the opening rules) | Close the shelf                                                                                                                 |
| `1–9 0 A–C - E–Z`                | —                                      | Use that template (same alphabet as the stash, so row _n_ has the same key in both)                                             |
| Return                           | —                                      | Use the selected template                                                                                                       |
| Down / Control-N, Up / Control-P | —                                      | Move the selection (wraps)                                                                                                      |
| Backspace                        | —                                      | Arm delete (red "Delete?" capsule); Backspace again deletes                                                                     |
| Control-S                        | Stash picker (unchanged)               | Close the shelf and open the stash (stash rules apply; if the stash refuses, the shelf stays open and shows the stash's status) |
| Escape / Control-[               | —                                      | Disarm if armed; otherwise close the shelf                                                                                      |
| Command-S                        | Save template (template drafts only)   | —                                                                                                                               |

- While delete is armed, Backspace deletes. Every other key disarms and then does its
  usual job, except Escape and Control-[, which only disarm. Moving the selection or a
  refresh that removes the armed row also disarms.
- In the stash picker, Control-T closes the stash and opens the shelf. A test pins that
  a modified `T` never reaches the stash's `T` accelerator.
- Inside capture pickers, inline prompts, and the successor-offer choice, Control-T is
  consumed, never AppKit's transpose. Control-T replaces AppKit's transpose-characters
  in the capture editor; that deliberate trade is documented in the README, in the same
  family as the existing Control-A/E/U/J/K/S/C overrides.
- Everything else in the shelf behaves exactly as in the stash picker. Printables that
  are not accelerators are consumed, and Shift-D does nothing (clearing lives in
  Settings).

### Opening rules

- **A prompt or picker is open:** the key is consumed with the stash's style of status
  ("Finish or cancel the block ID prompt before opening templates").
- **Empty draft:** opens.
- **Draft that the latest parse for the current text says has no template:** opens;
  using a template then _inserts it at the top_.
- **Draft with a template:** stays closed with the status "This draft already has a
  template".
- **Empty shelf:** "No saved templates yet — capture an @@template: draft and it lands
  here".
- **Old bob** (`capture-templates` is an unrecognized subcommand): "Update bob to use
  saved templates", and the footer button is hidden.
- **Shelf not loaded yet** (first launch race): "Loading saved templates…" and a
  refresh; no auto-open.

### Using a template

- **Empty draft:** insert `draft` and put the caret at `cursor`.
- **Rows present:** insert `draft` at offset 0. The caret keeps its place, shifted by
  the insertion.
- **Both:** the change is one native, undoable edit through the editor's `NSTextView`
  (`breakUndoCoalescing()` plus `insertText(_:replacementRange:)`, the way Control-J and
  Control-Shift-O edit), so one Command-Z removes the whole insert. Never install it
  with `setPlainDraft`, which is not undoable.
- **Immediately after:**
  - bob's next parse paints the slot hues on `{0} {1} {2}`;
  - the calm `template_row` state shows the A10 wording;
  - Control-J already inserts a plain newline in template drafts, so the first row never
    grows a stray `- `.

### Never send a flag an older bob would capture as text

Verified while planning, against a throwaway vault: an older
`bob capture --format json --template-origin abc -- hi` does **not** fail. `TEXT`
swallows the unknown flag, the `--`, and the draft, and bob captures
`- [ ] #task --template-origin abc -- hi` with exit 0. An "unexpected argument" fallback
can never fire for `bob capture`.

So `BobProcessClient` adds `--template-origin` **only** when a `capture-templates list`
from the same executable `identityToken` has succeeded. When the identity changed or the
list has never succeeded, it silently drops the origin; the worst case is an edited copy
saved beside the original. Both flags land in bob before any Mac phase starts (phases
`store` and `protocol`), so a successful list proves `--template-origin` support.

### Presentation

```text
╭───────────────────────────────────────────────────────────────────╮
│ ┃1┃  Apply to {0}! @job^apply-{1}                         ● ● ●   │ ← selected
│      - **URL**: {2}                                               │
│      3 fields · job · used 14× · 2h ago                           │
│ ┃2┃  Read {0} @reading^{1} p:2                            ● ●     │
│      2 fields · reading · used 5× · 3w ago                        │
│ ┃3┃  {0} @groceries                                       ●       │
│      1 field · groceries · used 1× · 6d ago                       │
├───────────────────────────────────────────────────────────────────┤
│  ↩ use   ⌫ delete   ⌃S stash   esc close                3 of 36   │
╰───────────────────────────────────────────────────────────────────╯
```

- **One system with the stash.** The shelf uses the same auxiliary region,
  `.regularMaterial`, corner radius 8, shadow, 24×22 keycaps, five visible rows then
  scroll, and the same selected-row accent fill (0.18, or 0.32 under Increase Contrast).
  Moving between Control-S and Control-T should feel like one system.
- **Faithful previews.** Each row's `spans` become an `AttributedString` through the
  existing `captureSemanticCategory(forSpanKind:)` and `CaptureEditorPalette`. `{k}`
  uses slot hue `k % 6`, `@job^` uses the route color, and links are underlined, so a
  row looks exactly like what lands in the editor.
  - The parent line is monospaced callout at full strength.
  - The first child line follows in the secondary color, with placeholders keeping their
    hue.
  - Then a tertiary `+N more lines` when `line_count > 2`.
  - A row never shows `@@template:` (every row would read the same).
- **Slot dots.** `field_count` small circles in slot hues, trailing, capped at six and
  then `+N`. The count is also in the metadata and the VoiceOver label, so color is
  never the only cue.
- **Metadata.** `3 fields · job · used 14× · 2h ago`, or `2 fields · saved just now`
  when `use_count == 0`. The route segment appears only when bob returns a route. Ages
  reuse the stash's relative-age wording.
- **Armed delete.** The slot dots give way to a red "Delete?" capsule (styled like the
  stash's Delete All keycap) and the row fill turns red (0.12, or 0.24 under Increase
  Contrast). The hint row reads: "⌫ again deletes · any other key keeps it".
- **A quiet hint row** in the slot (and height) of the stash's red Delete All bar shows
  keycap-styled `↩ use`, `⌫ delete`, `⌃S stash`, `esc close`, and a right-aligned
  tertiary `N of CAP`, or `saving off` when `saving` is false.
- **Footer button.** **Templates N** with the `square.stack.3d.down.right` symbol, the
  glyph the `@@template:` completion row already uses, so the two read as one feature.
  It sits left of **Stash N**, is disabled under the same conditions, and is hidden for
  an old bob. Tooltip: "Reuse a saved @@template: draft (Control-T)".
- **Accessibility.**
  - List label: "Saved templates, 3 entries".
  - Row label: "Template 1, Apply to {0}! @job^apply-{1}, 3 fields, job, used 14 times,
    2 hours ago". Row hint: "Press 1 or Return to use this template. Press Delete twice
    to delete it."
  - Arming announces "Press Delete again to delete template 1".
  - Reduce Motion is respected and there are no celebration animations.
- **Status copy after a capture**, appended to whatever status the existing chain picks:
  - `new`: " · Template saved — ⌃T to reuse". This teaching moment is how the feature
    becomes discoverable.
  - `replaced`: " · Template updated".
  - `updated`: nothing. The footer count is the tell.
  - `not_saved`: " · Template not saved: <reason>".

## Phase `store`: template shelf store and auto-save in `bob capture`

**Goal:** every successful real template capture lands on the shelf, reliably and
safely, before any reader exists.

1. **Store module** `src/native/capture_templates/` (`mod.rs` plus `store.rs`; the agent
   may reshape it). Implement the whole [store contract](#the-store): path resolution
   with `BOB_CAPTURE_TEMPLATES_FILE`, canonical text, ids, the locked atomic write,
   quarantine, the newer-schema guard, unknown-field preservation, cap trimming, and the
   origin replace. The lint gate is `-D unused`, so add only what this phase's consumers
   use (the save path and the read it needs). `list`, `delete`, and `clear` arrive in
   phase `protocol`.
2. **Config.** `capture.max_saved_templates` in `src/native/config/capture.rs`, with
   tolerant loading as specified, and documented wherever the other config keys are.
3. **`bob capture`.** Wire the post-commit save, the hidden `--template-origin`, the
   additive `template.saved` / `template.not_saved` JSON, the stderr warning, and the
   human ` · saved` segment, all as specified in
   [`bob capture` additions](#bob-capture-additions-all-additive). Add
   `BOB_CAPTURE_TEMPLATES_FILE` and `XDG_STATE_HOME` (base state directory) to the
   alphabetized Environment list in `bob capture --help`.
4. **A10.** Change the end-of-block `template_missing_rows` wording as specified, and
   update the affected unit tests, CLI goldens, and the `docs/capture.md` diagnostics
   table.
5. **Docs.**
   - Add `docs/capture.md` § Saved templates, after § Bulk templates. Cover what is
     saved (the block, never rows), when (successful real captures only), where (the
     path, the override, 0600), the cap and the off switch, replace-by-origin (in prose:
     "the app's checkout"), and that forced `-r/-s/-t` destinations are not saved.
   - Document the JSON additions in § Input, stdin, and JSON output.
   - Add `BOB_CAPTURE_TEMPLATES_FILE` to the README Environment section.
6. **Tests.**
   - Store unit tests:
     - normalization (CRLF, trailing whitespace, Unicode, escaped braces, indentation,
       blank lines between the header and the block);
     - id stability; dedup and bump; eviction at the cap; a lowered cap trimming on the
       next write; cap 0;
     - every origin case in [Replace](#the-store);
     - a corrupt file (quarantined on write, reported on read), a newer schema (writes
       refused, bytes unchanged), and an unknown-field round trip;
     - 0600/0700 permissions; the lock timeout (hold the lock in the test; use a short
       injected timeout).
   - CLI tests (they already isolate `XDG_STATE_HOME`):
     - a real template capture saves; `--dry-run`, `-E`, a failed capture, a headerless
       draft, `capture-parse`, and `capture-complete` leave the store path absent;
     - one-row and many-row captures save the same entry;
     - `template.saved` statuses across repeated captures and origin replaces, plus the
       missing-origin no-op;
     - the human segment;
     - an unwritable store (for example the override pointing under a regular file)
       keeps exit 0 with `not_saved` and the stderr warning;
     - a bad `--template-origin` is a usage error;
     - byte-identical output for drafts without a header.
7. **Done when** `just check` passes.

## Phase `protocol`: `bob capture-templates`

**Goal:** a terminal user and the Mac app can list, show, save, delete, and clear saved
templates through one additive, well-documented endpoint.

1. **Command group** `capture-templates` in `src/runner.rs`:
   - Section: Capture protocol, sorted after `capture-tasks`.
   - Members: `clear`, `delete`, `list` (default), `save`, `show`.
   - `extra_about`: "Bare `bob capture-templates` runs the read-only list."
   - `after_help` with examples, the store path, and "See docs/capture.md § Saved
     templates".
   - Every public option has a short alias, and options are alphabetized. Excellent `-h`
     output, per the CLI rules (`sase memory read cli_rules.md` first).
2. **Store API.** Add read-only `list`, `delete`, and `clear` to the store from phase
   `store`, plus the definition-only validation helper in `template.rs`.
3. **Spans and route.** Expose a small `pub(crate)` helper from `capture_parse.rs` that
   parses a draft with the existing builder and returns wire spans and item 1's route.
   `list` restricts the spans to the template text and shifts them by the header length.
   Assert in tests that a three-field example yields `template_placeholder` spans with
   `field` 0–2 and the route span.
4. **Output.** The JSON and colored human output exactly as specified. Share the
   slot-hue painter between `list` and any later human output.
5. **Docs.**
   - Add `docs/capture.md` § `bob capture-templates` (after § Discovery commands), with
     the full JSON contract, error codes, index/id rule, and compatibility (an older bob
     reports an unrecognized subcommand, exit 2).
   - Add the README Capture protocol table row and update its count ("The ten" → "The
     eleven").
   - Add `bob capture-templates` to the README Commands section.
6. **Tests.**
   - CLI: list (empty, populated, saving off, unreadable file, newer schema), the JSON
     shape (`draft`, `cursor`, `spans`, `route`, `index`), show by index and by id, and
     not found.
   - save: no rows, invalid definitions with their codes, not a template, disabled, and
     origin replace.
   - delete: idempotent, mixed found/missing, by index.
   - clear: removes the quarantine too.
   - The help goldens and completion (`bob <TAB>` capture-protocol group and
     `bob capture-templates <TAB>`).
7. **Done when** `just check` passes.

## Phase `mac-core`: models, client, and fixtures

Open the repo with `sase repo open bob-mac-capture` and read its `AGENTS.md` first. Its
compile rules are binding: never land Swift you have not compiled.

**Goal:** everything the shelf needs from bob is decoded, gated, and presentable in the
portable `CaptureCore` target, with recorded fixtures, before any UI exists.

1. **Models** (`Sources/CaptureCore/`). Decode tolerantly (`(try? …) ?? default`, with
   defaulted init parameters):
   - `CaptureSavedTemplatesResponse` (`schemaVersion` must be 1, `capacity`, `saving`,
     `file`, `templates`, `warnings`);
   - `CaptureSavedTemplate`, `Identifiable` by `id` (`index`, `text`, `draft`, `cursor`,
     `fieldCount`, `lineCount`, `route`, `spans: [CaptureSpan]`, dates, `useCount`);
   - the save, delete, and clear responses;
   - `CaptureTemplateSummary.saved` (`id` plus a status enum with an `unknown` fallback)
     and `notSaved`.
2. **`BobProcessClient`.**
   - `savedTemplates()`, `saveTemplate(_:origin:)`, `deleteSavedTemplates(ids:)`, and
     `clearSavedTemplates()`, all with `--format json` and their own lanes. They must
     not cancel the submit or preview lanes.
   - An unrecognized `capture-templates` subcommand maps to a distinct "unsupported"
     error, so the UI can hide the feature.
   - `capture(…, templateOrigin: String? = nil)` follows
     [the identity gate](#never-send-a-flag-an-older-bob-would-capture-as-text): the
     client records the `identityToken` of its last successful list and includes
     `--template-origin` only when it still matches.
3. **Shared picker helpers** in `CaptureCore`: move the stash's accelerator alphabet,
   accelerator lookup, wrap-around selection, and relative-age wording into one shared
   helper. `CanceledDraftStash` delegates to it with no behavior change (its existing
   tests stay green).
4. **Row presentation builder** (pure and portable): from a `CaptureSavedTemplate`,
   build the parent line and the first child line as span-categorized segments, the
   `+N more lines` count, slot-dot hues (cap 6 plus overflow), the metadata string, and
   the VoiceOver label and hint, exactly as in [Presentation](#presentation). Colors
   stay in `BobMacCapture` (`CaptureEditorPalette`); `CaptureCore` emits categories.
5. **Environment** (`BobEnvironment.preservedOverrideKeys`): replace the dead
   `BOB_CONFIG` with `BOB_CONFIG_FILE` (the name bob reads, `src/native/config/mod.rs`),
   and add `BOB_CAPTURE_TEMPLATES_FILE` and `XDG_STATE_HOME`, so the app and the
   terminal resolve the same shelf and config. Update the tests that pin the set.
6. **Fixtures.** Record real JSON with a bob built from bob-cli master (`cargo build` in
   the bob-cli workspace), with `XDG_STATE_HOME` and `BOB_DIR` pointing at temp
   directories, following the recipe in the header comment of
   `Tests/CaptureCoreTests/CaptureRefPresentationTests.swift`:
   - list: populated (a three-field two-line `@job^` template, a two-field template, and
     a one-field template), empty, and saving off;
   - save: new, replaced, and an invalid definition;
   - delete and clear;
   - capture successes carrying `template.saved` `new`, `replaced`, and `not_saved`;
   - a re-recorded `capture-parse-template-no-rows.json` for the A10 wording (update the
     assertions that quote the old text).

   Wire them into `Tests/Fixtures/fake-bob`: `capture-templates <member>` routes, an
   `FAKE_BOB_TEMPLATES_UNSUPPORTED=1` mode that prints clap's unrecognized-subcommand
   error with exit 2, and `capture` argv parsing that skips `--template-origin <ID>`
   before `--` (argv is already recorded for assertions).

7. **Tests** (`Tests/CaptureCoreTests`): decoding, including older payloads without the
   new fields; client argv and lanes; the identity gate (origin dropped before a list,
   sent after, and dropped again after the identity token changes); the unsupported
   mapping; the presentation builder (segments, dots, metadata singular/plural and
   unused, labels); and the shared helpers.
8. **Verification** per `AGENTS.md`:
   - run `just portable-check` (on athena if `swift` is missing; read
     `sase memory read tailnet.md` first);
   - make the best-effort scratch-copy compile on the mac (`bundle.sh` and
     `just typecheck-tests`), since the stash delegation touches `BobMacCapture`;
   - make sure CI is green.

   If the mac is offline, say in the final report that the macOS targets are unverified.

## Phase `mac-shelf`: the Control-T shelf

Same repo rules as `mac-core`. **Goal:** Control-T opens a shelf that looks and feels
like the stash's sibling, and picking a template drops it into the draft as one undoable
edit.

1. **Shelf store** (`Sources/BobMacCapture/SavedTemplateShelf.swift`, an `@MainActor`
   `ObservableObject`):
   - It holds the cached entries, capacity, saving flag, file, and availability
     (unknown, available, unsupported, or failed with a message).
   - `refresh()` coalesces concurrent calls and keeps the last good list on failure.
   - It refreshes on launch, on every panel show (stale-while-revalidate, like the idle
     agenda), after every capture success, after every shelf mutation, and after Recheck
     Bob or a bob identity change. The store is not in the vault, so vault events are
     not a trigger.
   - A refresh that lands while the shelf is open keeps the selected id when present and
     clamps otherwise.
2. **Router** (`CaptureKeyCommandRouter`):
   - Add `KeyCode.t` (17), a `templateShelfVisible` branch ahead of the stash branch,
     and the commands for the [Keys](#keys) table.
   - Add Control-T in the stash picker, Control-T consumed in pickers, prompts, and the
     successor-offer choice, and Command-S in the editor. Command-S is routed in this
     phase and implemented in `mac-checkout`; until then it consumes the key with a
     status, so AppKit never sees it.
3. **Model** (`CapturePanelModel`): the opening rules, selection, use by index or
   selection, the two-press delete (optimistic removal, then `deleteSavedTemplates`; on
   failure a refresh restores the row and the status shows bob's message; deleting the
   last entry closes the shelf with "Template deleted · shelf is empty"),
   Control-T/Control-S switching, and closing on hide and dismissal exactly where the
   stash closes.
4. **Insert** as one native undoable edit in both modes, through a
   `CapturePanelController` helper next to the Control-J and Control-Shift-O helpers.
   The model decides; the controller edits the `NSTextView`. Convert `cursor` from UTF-8
   with the existing `CaptureTextRanges` helpers. After the insert, focus returns to the
   editor and analysis runs as for any edit.
5. **View** (`CapturePanelView`):
   - the shelf picker in the auxiliary region (a new `AuxiliarySection` case), with rows
     built from the `mac-core` presentation builder and painted by
     `CaptureEditorPalette`;
   - the slot dots, the armed capsule, and the hint row;
   - a height policy parameterized from the stash's (template rows are taller; five
     visible rows, then scroll);
   - the **Templates N** footer button.
6. **Tests:**
   - router mapping, including "a modified T never reaches the stash's T accelerator"
     and "digits in the shelf never hit the close-task comma assist";
   - every opening rule;
   - pick in both modes, with one Command-Z restoring the exact prior text and caret;
   - arm, disarm, and delete, including failure restore;
   - the switches;
   - the stale refresh while open;
   - light and dark render fixtures of a three-row shelf with one row selected and one
     armed, through `RenderFixtureWriter`.
7. **Verification:** the same as `mac-core` (`just portable-check`, the scratch Mac
   compile with `bundle.sh` and `just typecheck-tests`, CI green; report the macOS
   targets as unverified if the mac is offline).

## Phase `mac-checkout`: checkout lifecycle, Command-S, Settings, and docs

Same repo rules as `mac-core`. **Goal:** the shelf's promise holds on every path: edits
replace, abandoning never loses anything, and the app says what happened.

1. **Checkout state** on the panel model: `origin: String?`, set on every pick.
   - Release it on Discard, `stashDraftAndClose`, any edit that empties the editor, and
     a fresh parse _for the current text_ that reports no `template`. A pending or stale
     parse never releases it.
   - Escape, hide, and a failed capture keep it. Restoring a stash entry never sets it.
   - > [!decision] pop_visible_in_count
     >
     > When on, the shelf's visible entries and the footer count exclude the checked-out
     > id while `origin` is set. The tooltip then says "1 template is in this draft; it
     > returns when you capture or discard". When off, skip this item.
2. **Submit:** pass `templateOrigin: origin` only while the latest parse still has a
   template (the client's identity gate does the rest). On success:
   - refresh the shelf and clear `origin`;
   - append the [status copy](#presentation) from `template.saved` or
     `template.not_saved` to whatever status the existing chain chose, without
     disturbing the toggle, close, start, ref, or bulk presentations.

   Control-C's stash status becomes "Canceled draft stashed · template kept" when a
   checkout was active.

3. **Command-S "Save template":** for a template draft with a supported bob, call
   `saveTemplate(draft, origin:)`.
   - Keep the draft, set `origin` to the returned id, and refresh.
   - The status is "Template saved — ⌃T to reuse", "Template already saved", or
     "Template updated".
   - Guard against double presses while a save is in flight.
   - A non-template draft gets "Only @@template: drafts can be saved as templates".
   - bob's refusal message (an invalid definition, or saving off) shows as status text,
     not error red.
4. **Settings** — a "Saved Templates" section after "Canceled Draft Stash":
   - the count and "up to CAP", or "Saving is off (capture.max_saved_templates: 0)";
   - bob's reported file path (monospaced, selectable, middle-truncated) with Reveal in
     Finder;
   - one line: "bob saves the template block after each successful @@template: capture.
     Rows are never saved.";
   - a confirmed **Clear Templates…** that calls `clearSavedTemplates()`, disabled when
     empty or unsupported;
   - "Update bob to use saved templates" for an old bob.
5. **README:**
   - the keyboard table (Control-T, Command-S, the shelf keys, and the transpose trade);
   - a Saved templates section with the [lifecycle table](#lifecycle) in prose form;
   - privacy (the file path, 0600, rows never stored, `clear` removes the quarantine
     copy);
   - persistence (the shelf survives quit; a checkout does not need to);
   - compatibility (an old bob hides the feature; the identity gate and why);
   - the required-bob-features list.
6. **Tests:** every row of the [lifecycle table](#lifecycle) as a model test with
   fake-bob (assert argv: `--template-origin` present only when expected); the status
   copy for each outcome; Command-S in each state; Settings clear; and the visible-count
   rule when `pop_visible_in_count` is on.
7. **Follow-up bead note.** Append a note to `bob-cli-83` (the Expand-template action):
   Control-T is taken by the template shelf, and Expand needs no checkout handling,
   because the parse then reports no template and the checkout releases itself.
8. **Memory.**

   > [!decision] memory_saved_templates_decision
   >
   > Add `decisions:saved-templates-are-checked-out` with your `/sase_memory_write`
   > skill. Its claim: saved templates are a bob-owned shelf of template blocks, saved
   > after successful real captures; a pick is a checkout that a successful capture
   > settles by replace-by-origin. Its alternatives are a Mac-owned store, a literal
   > pop, and saving as a staged participant in the vault commit (see the research
   > report's "Resolving the disagreements"). It reopens when cross-machine reuse,
   > names, or pinning are needed. Cite this epic and the research report.

   > [!decision] memory_saved_template_glossary
   >
   > Add `glossary:saved-template` (alias template shelf) with your `/sase_memory_write`
   > skill: a template block bob saved after a successful real `@@template:` capture,
   > reused through Bob Mac Capture's Control-T shelf or `bob capture-templates`. Link
   > `[[bulk-capture]]`.

   For each decision that is off, record a `PROPOSED FOLLOW-UP:` note on this phase's
   bead instead.

9. **Verification:** the same as `mac-core`, plus one end-to-end manual pass on the mac
   if it is online:
   - capture a template;
   - Control-T it back;
   - edit it and capture;
   - check `bob capture-templates` on the Mac shows one edited entry on top.

## Out of scope and follow-ups

Phase workers record these as `PROPOSED FOLLOW-UP:` notes on their own beads only when
they run into a concrete need:

1. Pinning, so a regular is never evicted (the schema already round-trips unknown
   fields).
2. Template names.
3. Option-Return "use as copy" (pick with no origin).
4. Saved templates as `@@` completion candidates.
5. A ghost `{0} {1} {2}` hint on the empty row line after a pick.
6. Applying a template to rows in a draft that already has one (needs a bob edit
   endpoint).
7. A search field, if the shelf ever regularly holds more than about 20 entries.
8. Cross-machine sync or "promote to vault", if Bryan starts templating from apollo.
