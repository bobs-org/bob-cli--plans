---
tier: epic
title: Capture type leads — `#f` ⇥ types `#fix Fix ▌`
goal: 'Accepting a configured task type from the Capture panel''s `#` menu at the
  start of an empty new-task item also types that type''s lead word (`#f` ⇥ → `#fix
  Fix ▌`). The menu previews the lead before you accept, one Backspace or ⌘Z removes
  it, and accept-and-capture never mints a one-word task. The five leads (BUILD, FIX,
  PLAN, READ, WRITE) come from `capture.type_leads` in the chezmoi-managed bob config.

  '
phases:
- id: lead_config
  title: bob-cli capture.type_leads loader, bob types surfacing, rename guard
  depends_on: []
  size: medium
  description: 'lead_config: add the deploy-safe `capture.type_leads` loader with
    parser-probe validation and per-entry degradation, show leads and their warnings
    in `bob types`, guard `bob types rename`, and document the config key.'
- id: lead_complete
  title: bob-cli task_type candidates carry an optional lead
  depends_on:
  - lead_config
  size: medium
  description: 'lead_complete: compute the optional `lead` object on editor `task_type`
    candidates behind Bob''s eligibility gates and one parser probe, keep shell completion
    and every write path lead-free, render it in human output, and document the contract.'
- id: mac_lead_preview
  title: Mac decodes the lead and previews it in the Task type menu
  depends_on:
  - lead_complete
  size: small
  description: 'mac_lead_preview: decode `lead` tolerantly and show it as a calm `Fix…`
    hint after the type pill, in the selected-row footer, and in VoiceOver, with protocol
    docs and CaptureCore tests.'
- id: mac_lead_accept
  title: Mac inserts the lead undoably, peels it with one Backspace, skips it on ⌘↩
  depends_on:
  - mac_lead_preview
  size: medium
  description: 'mac_lead_accept: apply a lead-bearing accept as two native undo groups
    bound to the response snapshot, arm a one-key Backspace peel, skip the lead on
    Control/Command- Return, update keyboard and panel docs, add hosted tests, and
    watch macOS CI green.'
- id: lead_rollout
  title: Enable the five leads in chezmoi and verify end to end
  depends_on:
  - lead_config
  - mac_lead_accept
  size: small
  description: 'lead_rollout: add the `capture.type_leads` block to the chezmoi bob
    config, apply it, verify with a master-built bob, and write the approved memory
    records.'
decisions:
  memory_decision_record:
    ask: Add a decisions record "Type Leads Are Editor Text" once the feature ships?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_term:
    ask: Add a "Type Lead" glossary term once the feature ships?
    memory:
    - glossary
    default: false
    answer: false
proposed_by: bbugyi200.athena.1hh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 15:03:17
status: done
bead_id: bob-cli-rb
---

- **PROMPT:** [prompts/202611/capture_type_leads.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_type_leads.md)
- **BEAD:** [bob-cli-rb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-rb/README.md)

# Plan: Capture type leads

Source research (adopted in full, with the refinements called out below):
`research:202611/capture_type_lead_words/capture_type_lead_words.md`. Bryan agreed with
all of its requirements; this plan owns the split, the exact contract, and the polish.

## The experience

**Type lead**: the text Bob offers to type right after a `#<type>` marker you accept at
the start of an empty new-task item. Once inserted it is ordinary, editable title text.
It has no lasting link to the type, no placeholder behavior, and no effect on parsing,
storage, or `[type::]`. It is called a "lead" rather than an "expansion" because `%`
already owns "how the draft expands" (`decisions:capture-sigils-say-where-what-how`).

```text
TASK TYPE                                        ● ◆ BUILD · dev suggests BUILD
  ◆ BUILD  Build…                                                    today
  ◆ CHORE                                                          Nov 26
  ◆ FIX    Fix…                                                      today   ← selected
  ◆ PLAN   Plan…                                                     today
types “Fix” · this task becomes FIX · dev suggests BUILD       ⏎ choose  ⌃D hide
```

- **Intuitive.** A row that will type a lead shows it as `Fix…` right after its tinted
  pill, so the row reads like the draft it produces (`#fix Fix▌`). A row without a lead
  shows nothing. Nothing is hidden and nothing animates.
- **Reliable.** Bob decides, from the draft and its own parser, whether a lead applies,
  what it says, and its exact bytes (`decisions:mac-capture-is-a-thin-client`). The Mac
  applies it only to the exact draft and caret the response was computed for. Old Macs
  ignore the field, and old bobs never send it. Every version-skew combination degrades
  to today's behavior, never to junk tasks.
- **Beautiful.** The lead lands as its own undo step. One Backspace right after an
  untouched lead removes the whole word (`#fix Fix ▌` → `#fix ▌`), joining the panel's
  atomic-Backspace family. ⌘Z removes it too. ⌃↩/⌘↩ skip it, so they keep today's "needs
  a task" refusal. Typing `#fix` and Space never accepts a row, which leaves a zero-cost
  "type only, no lead" path.

### Where a lead applies

A lead applies at the start of **any empty new-task item**: an empty panel, a
route-first item (`@cash #f`), any item of a bulk draft, and after an edge clock tag
(`#off #f`). The predicate is Bob's (phase `lead_complete`). Its outcomes:

| Draft at accept (caret at end)                                | Result                       | Why                                    |
| ------------------------------------------------------------- | ---------------------------- | -------------------------------------- |
| `#f` in an empty panel                                        | `#fix Fix ▌`                 | The motivating case                    |
| bare `#`, choose FIX                                          | `#fix Fix ▌`                 | Same gates                             |
| `/f` via the hash-menu alias                                  | `#fix Fix ▌`                 | The alias rewrote `/` → `#` first      |
| `@cash #f`                                                    | `@cash #fix Fix ▌`           | A leading route is transparent         |
| `Buy milk` ⏎⏎ `#w`                                            | second item `#write Write ▌` | Each item is its own start             |
| `#off #f`                                                     | `#off #fix Fix ▌`            | A clock tag is a task tag, not a title |
| `#f` above `  - child`                                        | `#fix Fix ▌` + child         | Children don't make a title            |
| `#f Fix it` (caret after `#f`)                                | `#fix Fix it`                | A title exists, so no lead             |
| `Fix it #f`                                                   | `Fix it #fix`                | Trailing marker                        |
| `#f @sase`                                                    | `#fix @sase`                 | Text follows the marker (v1 rule)      |
| `@sase:foo #f` / `@sase+foo #f`                               | retype only                  | `subject == "existing"`                |
| `@sase^foo@ #f` / `%t #f` / record or notebook route          | bare                         | Not a plain new task                   |
| `Parent` ⏎ `  - #f`                                           | bare                         | Child bullet line                      |
| `#c` → CHORE                                                  | `#chore ▌`                   | No lead configured                     |
| refused (`type_not_configured`, `type_marker_not_applicable`) | bare                         | Leads never rescue a refusal           |
| zsh `bob capture #f<TAB>`                                     | `#fix`                       | Shell completion never carries leads   |
| `bob capture`, `capture-parse`, paste, stdin, templates       | unchanged                    | Leads exist only at accept             |

### The wire contract (additive, `schema_version: 1`, no `grammar` bump)

`bob capture-complete` `task_type` candidates gain one optional object, omitted
(`skip_serializing_if`) whenever no lead applies. `replacement` and `cursor_after` keep
today's bytes in every case:

```json
{
  "name": "FIX",
  "key": "fix",
  "replacement": "#fix ",
  "cursor_after": 5,
  "lead": { "text": "Fix", "insert_at": 5, "insertion": "Fix ", "cursor_after": 9 }
}
```

- `text`: the configured lead, for display.
- `insert_at`: a UTF-8 byte offset in the draft after `replacement` is applied.
- `insertion`: the exact text to insert there. It always ends with one space. It starts
  with a space only when no space or tab separates the marker from the insert point. Bob
  owns all spacing, so double spaces cannot occur.
- `cursor_after`: a UTF-8 byte offset in the draft after `insertion` is inserted.

Spacing rule, with `e` the byte offset just past the accepted marker itself (the
`replacement` minus any trailing space, so a `#"DEEP WORK"` literal ends after its
closing quote) in the post-replacement draft:

- If the byte at `e` is a space or tab (Bob's own pushed space or an authored one), then
  `insert_at = e + 1` and `insertion = text + " "`.
- Otherwise (end of draft, `\n`, `\r`), `insert_at = e` and
  `insertion = " " + text + " "`.
- Either way, `cursor_after = insert_at + insertion.len()`.

So `#f` gives `{5, "Fix ", 9}`. `#f␠` gives a candidate `cursor_after: 4` and a lead of
`{5, "Fix ", 9}`. `#f` followed by `\n  - child` gives `{4, " Fix ", 9}`.

### Governing decisions

- `decisions:mac-capture-is-a-thin-client`: Bob computes eligibility, text, and offsets.
  Swift renders and applies them, and decodes with `decodeIfPresent`.
- `decisions:capture-sigils-say-where-what-how`: no new syntax. The expanded draft
  parses exactly like hand-typed text.
- `decisions:capture-reserves-clock-tags`: clock-tag spellings are rejected as lead
  keys, and a leading clock tag stays transparent.
- `decisions:capture-bullets-mirror-the-task-block`: no leads on child or drawer
  bullets.
- `decisions:theme-hide-is-recent-only`: a hidden type accepted by exact name keeps its
  lead.
- `decisions:similar-tasks-are-a-settled-heads-up`: a lead-only title ("Fix") must stay
  below the band's bars.
- `decisions:bob-cli-releases-ride-a-daily-train`: bob-cli commits are additive
  `feat`/`docs` with no `!`. Never edit the `Cargo.toml` version or `CHANGELOG.md`.

### Rollout safety

Nothing activates until the chezmoi map exists, so `lead_rollout` comes last. Before
that, every phase is inert in daily use. Unknown `capture:` keys are ignored by every
current bob loader, and a Mac that doesn't know `lead` inserts nothing. Never put leads
under `types:`: older bobs reject a mapping value there and drop the whole vocabulary.

---

## Phase `lead_config`

**Repo:** bob-cli.

### Loader (`src/native/config/capture.rs`)

Add `load_type_leads_config(path) -> TypeLeadsConfig` as a sibling of
`load_clipboard_history_config`. It parses `capture.type_leads` independently of every
other key and never returns an error. A problem degrades only the affected entry, or
only the map, and records a bounded warning naming `capture.type_leads.<KEY>` and the
reason. Errors never change how `types:`, `max_saved_templates`, or the clipboard keys
load, and vice versa.

`TypeLeadsConfig { leads: BTreeMap<String /* normalized key */, TypeLead { name, text } >, warnings: Vec<String> }`

Rules:

- A missing, `null`, or `{}` map turns the feature off silently. A non-mapping value
  warns once and turns the feature off. A whole-file YAML failure yields no leads and no
  extra warning, because the existing loaders already report the file.
- **Keys** must be strings. They normalize through `normalize_component` (the type
  identity) and must normalize to a non-empty key.
  - Reserved clock-tag names (`clock_tags::is_reserved_type_name`) are skipped with a
    warning.
  - Two spellings that normalize alike (`FIX` and `fix`) are both skipped, with one
    warning naming both.
- **Values:** `null` disables an entry silently. Any non-string value warns and skips.
  Otherwise the value is trimmed and must be non-empty, one line, at most 64 characters,
  and free of control characters.
- **Parser probe.** Validate with Bob's own parser, not a character blocklist. Run
  `capture_language::parse_for_editor` on `#bob-lead-probe <text>` and on
  `#bob-lead-probe <text> x`. The second probe catches text that only means something at
  the trailing edge. Each probe must yield exactly one item, in mode `Task`, with no
  diagnostics, exactly one `task_type` span (the probe marker), and a body equal to
  `<text>` (resp. `<text> x`). Otherwise skip the entry with a warning such as
  `lead must read as plain title text`. A value like `@home`, `%t`, `~x`, `x=`, or
  `#now` therefore never loads. Never interpolate or execute anything.

### `bob types` surfacing (`src/native/pomodoro_themes/cli.rs`)

Add `lead: Option<String>` to `ListTheme` (JSON `themes[]`, omitted when none). The
human row meta appends `lead "Fix"`, for example
`■ FIX  today  35 days  red · config · lead "Fix"`. This is the "is my config live?"
check. `warnings` gains the loader warnings, plus a non-fatal stale-key warning when a
`types:` vocabulary is configured and a lead key is not in it (the lead can never show).
Keep the existing warning rendering: stderr in human mode, `warnings` in JSON.

### Rename guard (`src/native/pomodoro_themes/rename.rs`, `config_reference`)

If OLD has a lead and NEW has none, refuse with `config_reference`, using the message
`<OLD> has a type lead; add capture.type_leads.<NEW> first` and the hint
`edit the chezmoi config to add NEW first`. This mirrors the existing `types:` check, so
a rename never silently orphans a lead.

### Docs (read `sase/memory/docs.md` via `/sase_memory_read` first)

- `docs/reference/configuration.md` § `capture`: a `capture.type_leads` row (mapping of
  type → one-line lead; validation; per-entry degradation; editor-only; link to the
  capture section `lead_complete` writes).
- `docs/task-types.md` § Rename coverage: one sentence on the new guard.
- The `bob types` JSON reference wherever `themes[]` fields are documented: the optional
  `lead` field.

### Tests

- Loader units in `capture.rs`:
  - missing, null, `{}`, and non-mapping maps;
  - a null entry;
  - non-string, empty, multi-line, over-64, and control-character values;
  - each probe rejection (`@home`, `~x`, `x=`, `%t`, `#now`);
  - reserved keys;
  - normalized duplicates (both dropped, one warning);
  - trimming;
  - a multibyte lead (`Réparer`);
  - independence in both directions from `max_saved_templates`, `clipboard_history`, and
    `types:`.
- CLI tests: `bob types` JSON and human output with leads, stale-key and invalid-entry
  warnings, and the rename refusal. Use `BOB_CONFIG_FILE` for fixture configs.
- Gate: `just check`, and `just docs-check` for the docs.

## Phase `lead_complete`

**Repo:** bob-cli.

### Model (`src/native/capture_complete/model.rs`)

Add a `TaskTypeLead { text, insert_at, insertion, cursor_after }` struct and
`lead: Option<TaskTypeLead>` on `TaskTypeCandidate`, with
`#[serde(skip_serializing_if = "Option::is_none")]`.

### Computation (`src/native/capture_complete/task_types.rs`)

Load `load_type_leads_config(&config::config_path())` only in the editor `build` path.
Pass an explicit flag so `build_for_shell` never loads or attaches leads. Attach leads
after `subject_refusal` has run. A candidate carries a lead iff **all** of these hold
(check cheap gates first):

1. The candidate is not `refused`, and the loaded leads contain `candidate.key`. This
   holds for hidden exact-match rows too. `effect` (`set`, `keep`, or `clear`) is
   irrelevant, so an inherited type still gets its lead.
2. `subject == "new"`.
3. **Marker geometry** on the pre-accept draft:
   - the completion marker is `leading` (`task_type_completion_at`);
   - it sits on its capture item's first physical line, never a child or continuation
     line, even though `type_marker.rs` marks those `leading` too;
   - every byte from `field.replacement.1` to the end of that physical line is
     whitespace.
4. **Parser probe.** Build D0 (the draft with `replacement` applied) and D1 (D0 with the
   computed `insertion` at `insert_at`). Run `parse_for_editor(D1)`. For the item with
   the same index, all of these must hold:
   - its mode is `EditorMode::Task`;
   - it has a `task_type` span covering the accepted marker;
   - it introduces no diagnostic code that the same item lacks in
     `parse_for_editor(D0)`, ignoring `type_marker_needs_task`, which D1 legitimately
     clears;
   - its trimmed body equals `text` or ends with `" " + text`.

   This one probe is the authority that rejects project notes, `%t` templates, record
   and notebook routes, and anything else that is not a plain new task. (Live probe:
   `@dev^foo@ #f` reports `refused: null` today even though capture refuses it, and gate
   4 is what keeps a lead off it.)

Offsets follow the spacing rule in
[the wire contract](#the-wire-contract-additive-schema_version-1-no-grammar-bump). Use
byte offsets throughout, so multibyte leads and `#"DEEP WORK"` literals work unchanged.

### Human output (`src/native/capture_complete/render.rs`)

`task_type_line` appends ` · types "Fix"` to the detail when a lead is present. The
shell description (`shell.rs::task_type_description`) stays unchanged.

### Docs

- `docs/capture.md` § "Typing tasks with `#<type>`": a short user-facing "Type leads"
  paragraph covering what, where, how to configure, and that Backspace/⌘Z remove a lead
  and ⌃↩ skips it (link the Mac docs). Lead with the `#f` ⇥ → `#fix Fix ▌` example.
- `docs/capture.md` `capture-complete` `task_type` contract (the paragraph that ends
  "`cursor_after` gives the resulting absolute caret offset"): the `lead` object, its
  four fields and coordinate spaces, the spacing rule, the eligibility rules in one
  paragraph, editor-only scope (not shell, not `bob capture`), and the JSON example with
  `lead`.
- `docs/reference/json-outputs.md` `bob capture-complete` row: mention type leads.

### Tests (`tests/cli/capture/complete_task_types.rs` and units)

- Every row of the [worked-case table](#where-a-lead-applies), with exact `insert_at`,
  `insertion`, and `cursor_after`. Include:
  - the authored-space case (`#f␠`);
  - the newline/child case (`" Fix "`);
  - offsets in a later bulk item;
  - a multibyte lead;
  - a `#"DEEP WORK"` literal with a configured lead.
- Hidden FIX accepted by exact query keeps its lead. A refused row never has one.
- Without `capture.type_leads`, existing `task_type` JSON stays byte-identical (no
  `lead` key anywhere).
- Shell completion of `#f` never carries leads and keeps its descriptions.
- Leads never appear at write time: with leads configured,
  `bob capture --dry-run -- '#fix '` still refuses with `type_marker_needs_task`, and
  `capture-parse` output is unchanged.
- `capture-related` on `#fix Fix ` in a fixture vault holding `Fix …` tasks returns no
  lexical band (short-query rule; `decisions:similar-tasks-are-a-settled-heads-up`).
- Gate: `just check`, `just docs-check`.

## Phase `mac_lead_preview`

**Repo:** bob-mac-capture (open with `/sase_repo`; follow its `AGENTS.md`). Capture one
real `bob capture-complete -f json -- '#f'` response from a bob built from bob-cli
master, with leads configured through `BOB_CONFIG_FILE`, and use it as the byte-exact
test fixture.

- **Decode** (`Sources/CaptureCore/CaptureModels.swift`, `CaptureCompletionCandidate`):
  add `lead: CaptureTaskTypeLead?` (`text`, `insertAt`, `insertion`, `cursorAfter`;
  snake_case keys). Decode it tolerantly with `try? container.decodeIfPresent(...)` so a
  malformed `lead` becomes nil instead of failing the whole response. Add an init
  parameter defaulting to nil.
- **Presentation** (`Sources/CaptureCore/CaptureTaskTypeMenuPresentation.swift`):
  - `Row.leadHint: String?` = `"\(text)…"` when `lead != nil` and the row is not
    refused.
  - The selected-row footer gains the prefix `types “Fix” · ` before Bob's existing
    effect copy (`types “Fix” · this task becomes FIX · dev suggests BUILD`).
  - Row accessibility appends ` Types Fix.`
  - Rows without a lead stay byte-identical to today.
- **View** (`Sources/BobMacCapture/CaptureTaskTypeMenuView.swift`):
  - Render `leadHint` immediately after the pill, before badges, so it reads as the
    draft `#fix Fix…`.
  - Use `.caption`, the regular weight, `.secondary` foreground, `lineLimit(1)`, and
    tail truncation. No second color, no italics, no animation, and never inside the
    pill: the pill is the type, the hint is the sentence.
  - Give it a higher `layoutPriority` than the usage caption, so at narrow widths the
    hint outranks `today`.
  - Keep the row height (`CapturePanelLayout.taskTypeRowHeight`) unchanged.
- `Sources/CaptureCore/CompletionRowContent.swift` `.taskType`: if this generic row path
  renders task-type rows anywhere, give it the same hint as `secondaryText` (only when
  there is no `similarTo` caption) and the same accessibility suffix.
- **Docs:**
  - `docs/protocol.md` § Optional fields: the `task_type` candidate `lead` field.
  - The capability table: one row for candidate `lead`, showing the menu preview and
    accept-time typing on a new bob, versus no hint and today's bare marker on an older
    bob, with the remedy "Upgrade `bob` and configure `capture.type_leads`".
  - `just docs-check`.
- **Tests** (`Tests/CaptureCoreTests/CaptureTaskTypeTests.swift`,
  `Tests/BobMacCaptureTests/CaptureTaskTypeDesignTests.swift`):
  - decoding with, without, and with a malformed lead;
  - hint, footer, and accessibility copy;
  - refused rows never hinting;
  - an older-bob response rendering exactly as before.
- **Gates:** `just portable-check`, plus a best-effort scratch compile on the Mac
  (`bundle.sh`, `just typecheck-tests`) per `AGENTS.md`. Then `Scripts/macos-ci-check`,
  and after the push watch the exact-SHA macOS run to green with `/sase_monitor`. If the
  Mac is offline, say the macOS targets are unverified.

## Phase `mac_lead_accept`

**Repo:** bob-mac-capture.

### Accept (Tab, Return, click)

In `CapturePanelModel.acceptSelectedCompletion` (via
`applySelectedCompletionReplacement`): when `context == "task_type"`, the selected
candidate has `lead`, and the gesture allows leads, apply the accept as **two native
`NSTextView` edits**. Use `CaptureNativeRewriteEdit` through `applyNativeRewriteEdit`,
following the pattern of `applyRecordDraftEdit`:

1. The marker edit: the response's `replacement` range in the pre-accept draft →
   `candidate.replacement`. The result is D0.
2. The lead edit: insert `lead.insertion` at `lead.insert_at` in D0, with the caret at
   `lead.cursor_after`. The result is D1.

`applyNativeRewriteToDraftEditor` calls `breakUndoCoalescing` and inserts synchronously.
That makes each edit its own undo group (⌘Z peels the lead; a second ⌘Z restores `#f`),
and step 2 sees D0 in the text view. The current `setPlainDraft` swap would leave the
text view stale for step 2, so do not chain the lead after it.

After the edits:

- Sync `attributedDraft` the way `applyRecordDraftEdit` does.
- Set `suppressedCompletionAcceptanceDraft = D1`.
- Schedule one analysis at the final caret with `requestCompletion: false`.
- Arm the peel.
- Request editor focus.

Nothing may reopen the type menu after the accept.

**Snapshot binding.** Apply the lead only when all of these hold:

- `completionDraftSnapshot == plainDraft`;
- the collapsed caret equals the response's `cursor`;
- `lead`'s offsets convert to UTF-16 in D0 and D1 (`utf16Range`).

**Fallbacks keep today's behavior.** If the snapshot is stale or step 1 fails, run the
existing marker-only path unchanged. If step 2 fails, keep D0 (marker accepted) and sync
the model. Never insert a lead the panel cannot peel. Candidates without `lead` keep the
existing `setPlainDraft` path byte-for-byte.

### Accept-and-capture skips the lead

`acceptSelectedCompletionThenSubmitAndOpen` (Control-Return, and Command-Return, which
shares it) accepts without the lead. Pass an explicit parameter; never use ambient
state. The submission is `#fix `, and bob's existing "needs a task" refusal keeps the
panel open, exactly as today.

### One-key peel (`CapturePanelController` `.deleteBackward` chain, `CapturePanelModel`)

Arm a `TypeLeadPeel { draft: D1, caretUTF16, range }` after a successful lead edit.
`range` covers the trailing `text + " "` of the insertion in D1 UTF-16, so any leading
separator Bob added stays and the caret lands right after `#fix `.

Add `deleteTypeLeadInEditableTextView` first in the `.deleteBackward` chain. It fires
only when all of these hold:

- the text view is editable and has no marked text;
- the selection is collapsed at the armed caret;
- `textView.string == armed draft`.

When it fires, it calls `model.dismissCompletion()`, deletes `range` with native
`insertText("", replacementRange:)` (so ⌘Z restores the lead), sets the caret, consumes
the arm, and returns true. Otherwise it returns false, and the existing requirement,
drawer, comma, and bullet handlers and then AppKit own the key. Modified Backspace stays
native.

**Disarm** on every observed draft or selection change that differs from the armed state
(so type-then-delete round trips stay disarmed), and on any completion accept, draft
replacement (template, history), submit, clear, or panel hide. Model it on the
`closeCommaProvenance` record/consume/reconcile seams. After a disarm, Backspace is 100%
ordinary.

### Docs

- `docs/keyboard.md` Backspace row: add "an untouched type lead".
- `docs/capture-panel.md` inline-completion section, by the Task type menu bullet:
  - what a lead is and the `Fix…` hint;
  - that Tab, Return, and click type it;
  - that Backspace or ⌘Z removes it;
  - that ⌃↩/⌘↩ skip it;
  - that Space after a typed `#fix` never adds one;
  - a link to bob-cli `docs/capture.md`.
- `just docs-check`.

### Hosted tests (`Tests/BobMacCaptureTests`)

- Tab, Return, and click on `#f` each give `#fix Fix ▌`.
- `@cash #f`, a second bulk item, the `#f` + `\n  - child` case (`" Fix "`), and a
  non-ASCII lead.
- ⌘Z gives `#fix ▌`; a second ⌘Z gives `#f`.
- Backspace peels to `#fix ▌`, and ⌘Z then restores the lead.
- Typing `x` then Backspace, then Backspace again, deletes one space natively, not the
  lead.
- A caret move away and back disarms the peel.
- ⌃↩ and ⌘↩ submit `#fix ` and surface the refusal with no lead.
- A stale response (the draft changed after the response) gives a marker-only accept.
- A response without `lead` behaves exactly as today.
- No menu reopens after the accept.

Async tests must never await an untimed continuation.

**Gates:** the same as `mac_lead_preview`: compile, `Scripts/macos-ci-check`, then watch
the exact-SHA macOS run to green with `/sase_monitor`.

## Phase `lead_rollout`

**Repos:** chezmoi (open with `/sase_repo`; follow its `AGENTS.md`), plus bob-cli for
memory when approved.

1. In chezmoi's `home/dot_config/bob/config.yml`, extend the existing `capture:` block.
   Keep `clipboard_history: alfred`, and match the `types:` key order:

   ```yaml
   capture:
     clipboard_history: alfred
     # Type leads: accepting one of these types from the Capture panel's # menu
     # at the start of an empty new task also types its lead, so `#f` ⇥ gives
     # `#fix Fix ▌`. Backspace or ⌘Z right away removes just the lead; ⌃↩
     # never adds it. Older bob builds ignore this key (bob-cli docs/capture.md).
     type_leads:
       BUILD: Build
       FIX: Fix
       PLAN: Plan
       READ: Read
       WRITE: Write
   ```

2. Commit, then run `chezmoi update -a --force` as that repo's `AGENTS.md` requires.
3. Verify with a bob built from bob-cli master. The released `bob` on PATH may lag the
   daily train.
   - `cargo run --quiet -- types` shows `lead "Build"` through `lead "Write"` on the
     five rows, and no type-lead warnings.
   - `cargo run --quiet -- capture-complete --cursor 2 -f json -- '#f'` returns FIX with
     `lead {"text":"Fix","insert_at":5,"insertion":"Fix ","cursor_after":9}`.
   - `cargo run --quiet -- capture-complete --cursor 2 -f json -- '#c'` returns CHORE
     with no `lead`.

   Report the results. Mention in the final report that the BUILD/READ/WRITE leads are
   the predicted low-hit-rate canaries (research vault census), and that a one-line
   config edit prunes or rewords them (for example `WRITE: Draft`).

> [!decision] memory_decision_record Write a decisions web strand **Type Leads Are
> Editor Text** via `/sase_memory_write`. Its claim: a lead is inserted only by an
> accepted `#` row at the start of an empty new-task item, is previewed before accept,
> is removable with one key, is skipped by accept-and-capture, is never added at write
> time (`bob capture`, `capture-parse`, paste, stdin, shell), lives in
> `capture.type_leads` (never `types:`), and has eligibility, text, and offsets decided
> by Bob. Use the record's usual sections (applies to, claim, why with rejected
> alternatives, cost, reopens when).
>
> - Rejected alternatives: baking the lead into `replacement`, keystroke rewrite,
>   inferring the type from the verb, ghost text, split accept keys, nesting under
>   `types:`, and hardcoded verbs.
> - Reopens when: a verb-based type suggestion, widening to `#f @route`, or a second
>   editor client.
> - Evidence: the research ref and this epic's commits.
>
> Republish with `sase memory init`.

> [!decision] memory_glossary_term Write a glossary strand **Type Lead** via
> `/sase_memory_write`. Define it as the text Bob offers to type after an accepted
> `#<type>` marker at the start of an empty new-task item, configured in
> `capture.type_leads` and inserted by Bob Mac Capture as ordinary editable title text.
> Link `[[task-type]]`, `[[capture-panel]]`, and the decision when it exists. Republish
> with `sase memory init`.

When a memory decision is off and no human reviewed the plan, record a
`PROPOSED FOLLOW-UP:` note on this phase bead instead of writing memory.

## Out of scope

- Per-type snippets, placeholders, caret templates, and route-specific leads.
  `CandidateContinuation` is the natural future home if ever needed.
- Leads for `#f @route` (text after the marker). Widen only on demonstrated need.
- Fixing `@route^id@ #f` reporting `refused: null` in completion while capture refuses
  it. This is tracked as its own task bead; gate 4 already keeps leads off it.
- Migrating every completion accept to the native edit lane. This epic only moves
  lead-bearing task-type accepts.
