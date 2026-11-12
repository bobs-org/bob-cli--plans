---
tier: epic
title: "Album record schema: render in the header, edit through bob"
goal: "The record frontmatter mapping on album notes renders as beautiful, honest header
  chrome instead of a truncated JSON row, and every common schema edit becomes one safe,
  previewed, revertable gesture through bob, with YAML editing kept first-class.

  "
phases:
  - id: report-additions
    title: Complete album schema JSON and the Obsidian-style emitter
    depends_on: []
    size: medium
    description:
      "report-additions: add suggest, default, bounds, line, filled, per-choice counts,
      dated_notes, legacy, and a semantic schema fingerprint to album list JSON, and
      emit the record block in Obsidian's YAML sequence style with round-trip fixtures."
  - id: schema-writer
    title: The bob albums update writer
    depends_on:
      - report-additions
    size: large
    description:
      "schema-writer: add bob albums update with a JSON ops envelope, dry-run diffs and
      impact counts, a fingerprint guard, Rust-side validation, span-preserving emit,
      one scoped transactional commit with index rebuild, typed refusals, and inverse
      ops."
  - id: header-reshape
    title: Collapsed schema summary, chip view, and read-only Schema Card
    depends_on:
      - report-additions
    size: medium
    description:
      "header-reshape: reshape the album header's schema surface into a
      collapsed-by-default summary line, an expandable chip view with usage counts, a
      read-only Schema Card modal, and a YAML jump with lint marks from bob's issues."
  - id: card-editor
    title: Editable Schema Card gestures with consequences and Undo
    depends_on:
      - header-reshape
      - schema-writer
    size: large
    description:
      "card-editor: make the Schema Card keyboard-first editable through bob albums
      update, with per-gesture dry-run consequence lines, scoped commits, an inverse-ops
      Undo, taught refusals, and palette, Command Mode, and record-note entry points."
decisions:
  commit_granularity:
    ask:
      Should each safe Schema Card gesture commit immediately as its own scoped commit?
    choices:
      gesture:
        One scoped commit per gesture; Undo replays inverse ops; history stays honest
        and revertable
      session:
        The card batches edits into a draft; one commit per explicit save; Undo exists
        only pre-save
    default: gesture
    why:
      Matches bob's one-writer transaction model and keeps every schema change
      independently revertable
    answer: gesture
  mobile:
    ask: Should album schema surfaces degrade gracefully on Obsidian mobile?
    choices:
      degrade:
        Header and chips render read-only on mobile with an edit-in-YAML hint; no
        subprocess writes
      skip: Desktop-only; on mobile the native record row stays visible and untouched
    default: degrade
    why:
      Follows the album service's existing desktop-gated subprocess pattern at little
      extra cost
    answer: degrade
  chord:
    ask:
      Should Ctrl+Shift+P open the Schema Card on album notes, where the Task Card chord
      is idle?
    choices:
      overload:
        The Task Card chord opens the Schema Card on album notes, editing the properties
        of what you are on
      separate:
        The Schema Card opens only from the header, the command palette, and Command
        Mode
    default: overload
    why: The chord is otherwise idle on album notes and the mnemonic stays coherent
    answer: overload
  memory_decision:
    ask:
      Should the land agent record the album-schemas-are-edited-through-bob decision
      strand?
    memory:
      - decisions:album-schemas-edited-through-bob
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.4b.linker.w0
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 07:00:40
status: wip
---

- **PROMPT:**
  [prompts/202611/album_schema_render_and_edit.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_schema_render_and_edit.md)

# Album record schema: render in the header, edit through bob

## Scope and completion owner

The `record` frontmatter mapping on album notes (`eat.md`, `podcasts.md`, `tv.md`,
`movies.md`) renders in Obsidian's Properties panel as a truncated JSON string behind an
unknown-type `?` icon, and editing it means hand-editing nested YAML. Obsidian's
Properties engine supports only flat scalars and string lists, exposes no public
property-widget API, and its unknown-type widget has corrupted nested mappings before
(bob ships `recover_album_noun` in `src/native/records/schema.rs` precisely because a
stringified record has leaked). The fix is therefore not a prettier Properties row: the
schema renders as bob-owned album-header chrome, and every common edit goes through a
new guarded bob writer. YAML editing stays first-class throughout.

This epic implements the user-endorsed consolidated research
`research:202611/album_record_schema_ux/album_record_schema_ux.md`. Read it with
`sase artifact read` before starting any phase; it is the design authority for this
plan, and its "Recommended solution" section is normative where this plan does not
override it. Also read `plan:202611/album_and_record_notes.md` (the album/record
contract) and `plan:202611/album_plugin_ui.md` (the in-flight plugin UI child epic this
work builds on).

Each phase below is bounded to one coding agent. This epic's land agent owns the
integrated proof and closes only this epic's own beads. Nobody working this epic closes,
reopens, or edits `bob-cli-k3` beads or their descendants. Phase workers never create
task beads: append discoveries as `PROPOSED FOLLOW-UP:` notes on the assigned phase
bead.

Out of scope, by design: flattening the schema into top-level properties, a body code
block or any second schema home, third-party nested-properties plugins, a plugin-side
YAML writer, Bases-as-schema-editor (`eat.base` remains the records table), capture
grammar for schema, a generic nested-properties framework, per-record value editing,
field/choice key renames and merges (v2 migration flow), kind changes, and noun changes.
The refused edits are refused with taught alternatives, never half-implemented.

## Coordination with the in-flight album UI child epic

`bob-cli-k3.7.1` (album cards, headers, navigation) was in flight when this plan was
authored: `bob-cli-k3.7.1.1` (album-data) had landed as bob-plugins `9c79123`
(`241l-plugin-album-data.js`, `241m-plugin-album-data-service.js` in
`plugins/bob-ledger-tools`), while `.2`–`.5` — including `.4`, the album header with a
scoped CSS hide of the `record` Properties row and an always-open read-only schema table
— were in progress. A coordination note pointing at the research was left on
`bob-cli-k3.7.1.4`.

Rules for this epic:

- `report-additions` and `schema-writer` touch only bob-cli and may start immediately.
- `header-reshape` and `card-editor` touch bob-plugins. Before editing any fragment, the
  worker MUST read `bob-cli-k3.7.1` with `sase bead read` and confirm it is closed. If
  it is still open, monitor for its completion with `/sase_monitor` rather than
  competing over the same fragments; if it cannot complete, record the blockage on the
  assigned phase bead and stop rather than duplicating the header.
- `header-reshape` starts by auditing what the child epic actually shipped (fragments,
  tests, docs) and reshapes it; it does not assume the authored spec matches the landed
  code. Whatever landed, the end state defined below is the deliverable, and existing
  landed fixtures (Properties isolation, unrelated `record` key on a non-album note,
  type change, empty schema, ordered choices, unload restoration) must keep passing.

## Starting evidence and repository discipline

Inspected bases: bob-cli `4ffa85d0`, bob-plugins `564f3c1`. Verified in bob-cli at that
base: `AlbumField::to_json` in `src/native/albums/report.rs` emits only
`key/label/kind/required/list/choices`, and the album entry omits `dated_notes` and
`legacy`; `RecordField` in `src/native/records/schema.rs` carries `default`, `suggest`,
`min`, `max`; `bob albums` has `list`, `check`, `sync`, `create` and no `update`;
`bob records update` (`src/native/records/cli.rs`) is the span-preserving dry-runnable
writer precedent; `src/native/albums/create.rs` and `write.rs` emit the `record:` block
with serde-style unindented sequences. Recheck current history and assigned-bead notes
when each worker starts; other work continues to land.

For bob-plugins work: open the repo with `/sase_repo` and read its `AGENTS.md`. Edit
`src/` fragments and `src/fragments.json` order; rebuild `main.js`, never hand-edit it;
every edited fragment stays at or below 1000 lines; prefer new cohesive fragments. Run
`npm run build`, `npm test`, `npm run validate`, and
`BOB_CLI_DIR=<primary bob-cli checkout> npm run docs:check`; register new tests
explicitly in `package.json`. Run `bob plugins sync` after changes and follow the repo's
deployment-conflict instructions; never force an overwrite on your own.

For bob-cli work: read `cli_rules.md` and `docs.md` through `/sase_memory_read` before
CLI or docs changes. bob-cli releases ride the daily train: never edit the `Cargo.toml`
version or `CHANGELOG.md`. Everything in this epic is additive — list JSON stays schema
version 1 and no commit needs a `!` contract-break marker. Use `/sase_monitor` for long
verification commands, and end every turn through `/sase_final`.

## Shared design contract

All four phases implement one design. Phase sections refer back to this contract instead
of restating it.

**Principles.** bob writes, the plugin presents: no UI code parses schema YAML or
rebuilds schema Markdown, matching the `plugins-albums` shared contract and
`decisions:mac-capture-is-a-thin-client`. The edits that actually happen (add a choice,
relabel, add a field, toggle required/suggest) are one-gesture operations; the edits
that strand data are refused with the impact count and a taught alternative. No gesture
writes without a dry run. Broken state renders visibly broken — an invalid schema never
impersonates a healthy empty one. YAML stays first-class: `y` jumps to the exact
Source-mode line, bob's diagnostics paint as lint marks, and kind/noun/key changes
happen there.

**Visual vocabulary.** The schema surface reuses the `[!records]` index's tokens: the
`🔖 To try` chip on the card is the same token as `🔖 11 to try` in the index. Owner-hue
accent and notebook-header typography (8px radius, 2px hue start border), light/dark
through Obsidian theme variables, clean wrap at ~375px, tabular numerals for counts,
labels lead with stored keys shown dim in monospace only where they matter (add and
rename stages), quiet `*` for required with a screen-reader "required" label. All
report-sourced strings are inserted as text, never HTML.

**The collapsed summary line** (the only schema chrome visible by default, under the
breadcrumb/title header):

```text
▸ Restaurant · 5 fields · dated notes · grouped by Status, sub-grouped by Location
```

Grammar: schema label (fall back to humanized noun, then stem) · field count with
correct singular/plural · `dated notes` only when true · grouping clause from `index_by`
(omitted when null) with `sub-grouped by` from `index_facets`. Expanding the disclosure
shows the chip view; `⊞ Schema` in the header opens the Schema Card. Collapse state
lives in view state keyed by album, never in frontmatter. An invalid schema renders the
line in the error state with the first issue message and expands to the lint list; the
`＋ <noun>` copy affordance disables with the same reason. An empty schema teaches: "No
fields yet — add one so `Name @eat/` can collect facts."

**The chip view** (expanded disclosure): one row per field in declaration order; human
label leading; kind as muted suffix text (`date`, `url`, `text · suggest`,
`number 1888–2100`, `list of text`); required as the quiet `*`; choice chips in stored
order with their emojis; `legacy` patterns as muted trailing chips; the field named by
`index_by` annotated "groups the index" and `index_facets` fields "sub-groups the
index", because that explains the generated index below.

**The Schema Card**, a keyboard-first modal in the Task Card idiom:

```text
╭─ 🍽️ Restaurant · Body › Restaurants ───────────────────────────────╮
│  104 restaurants · dated notes · grouped by Status                  │
│  ▸ Status *     🔖 To try 11   💚 Liked 86   🚫 Not liked 7   ＋    │
│    Category *   text · suggest                                      │
│    Location *   text · suggest · sub-groups the index               │
│    Visited      date · set on 2 of 104                              │
│    Url          url · set on 31 of 104                              │
│    ＋ Add field                                                     │
│  adds 🔁 Regular after 💚 Liked · no record changes                 │
│  ⏎ edit  a add field  c add choice  ! required  s suggest           │
│  ⌥J/⌥K move  ⌃D remove  y YAML  u undo  esc close                   │
╰─────────────────────────────────────────────────────────────────────╯
```

Title row: album emoji, schema label, owner breadcrumb. Stats line: record count, dated
notes, grouping. Field rows: live usage counts on choice chips and `set on N of M` from
the report. Consequence line: the dry-run result for the staged gesture, always shown
before Enter. Keymap footer: the read-only card (`header-reshape`) shows only
navigation, `y`, and `esc`; the editable card (`card-editor`) shows the full map.

**Report contract.** The plugin reads everything from the shared album report service
(bob-plugins `241l`/`241m`, backed by `bob albums list -f json`); `issues[]` already
carries `line`, `key_path`, `message`, and `suggestions` for lint marks. The additions
in `report-additions` complete that model; decoders treat them additively.

> [!decision] mobile = degrade On mobile (no subprocess), the summary line and chip view
> render read-only from the service's last-good data with an edit-in-YAML hint;
> `⊞ Schema` and all write gestures are absent. Follow the service's existing
> desktop-gating pattern.

> [!decision] mobile = skip Album schema chrome mounts only on desktop. On mobile the
> native `record` Properties row stays visible and untouched; add no mobile-specific
> code or tests.

## report-additions

Complete the album schema JSON model and fix the schema emitter's YAML style, in
bob-cli, additively under list JSON schema version 1.

1. Extend `AlbumField::to_json` in `src/native/albums/report.rs` with `suggest` (bool),
   `default` (typed as the field's kind — string, number, or bool; emitted only when a
   default is set, so absent is distinguishable from empty, false, and zero), `min`,
   `max` (numbers, emitted only when set), `line` (1-indexed line of the field's key in
   the album note, for YAML jumps), and `filled` (count of records with a nonempty value
   for the field). Extend each choice with `count` (records using that choice). Extend
   the album entry with `dated_notes` (bool), `legacy` (ordered string array), and
   `schema_fingerprint`.
2. `schema_fingerprint` is a stable hash of the parsed `record` mapping's semantic
   content — noun, label, plural, dated_notes, legacy, and every field and choice with
   their declaration order and all attributes — independent of byte formatting, so a
   cosmetic rewrite (such as Obsidian re-serializing `summary`) does not change it but
   any semantic edit does. Define it in one place that `schema-writer` can reuse for its
   `expect` guard, and document the definition in the JSON contract docs.
3. Mirror the same field model anywhere record/album schema fields are emitted as JSON
   (`bob records show -f json` and friends), so the two surfaces stay one model. Audit
   rather than assume which commands emit field JSON.
4. Fix the schema emitter: `src/native/albums/create.rs` and `write.rs` emit the
   `record:` block (notably `legacy:`) with serde-style unindented sequence items, while
   Obsidian writes indented ones, so an unrelated Properties edit rewrites the block's
   bytes. Emit Obsidian's style (indented sequence items) everywhere bob writes the
   `record:` block, and add a byte-level fixture asserting the emitted block matches the
   form Obsidian's own serializer preserves. The epic plan's "re-serialize is a no-op"
   risk claim becomes true; the real-Obsidian round-trip proof lands with the land
   agent's e2e.
5. Tests: JSON shape fixtures covering every new field including null/absent handling;
   fingerprint stability across byte-only changes and instability across each semantic
   change class (order, label, flag, choice set); `filled`/`count` accuracy against a
   fixture vault including invalid records; emitter byte fixtures; existing
   list/check/sync/create fixtures updated. Update `docs/records.md`'s JSON contract
   (read `docs.md` memory first). The bob-plugins album service needs no change in this
   phase; decoders are additive-tolerant — verify, don't assume.

## schema-writer

Add `bob albums update`, the only writer of album schema YAML, in bob-cli. Read
`cli_rules.md` through `/sase_memory_read` before shaping the CLI.

**CLI.**
`bob albums update STEM [-d|--dry-run] [-f|--format human|json] [-i|--input FILE]`,
nested under the existing `albums` noun, options alphabetical, every public long option
with a short alias, excellent `-h` text, shell completion updated. `-i` takes the ops
envelope (`-` for stdin); ergonomic flags for the most common gestures (such as adding a
choice) may mirror envelope ops, but the envelope is the canonical interface and argv is
never built from labels or YAML. Commit subject:
`albums: update <stem>.md (<short op summary>)`, e.g.
`albums: update eat.md (status +regular)`.

**Envelope.** `{"schema_version": 1, "expect": "<schema_fingerprint>", "ops": [...]}`,
ordered ops applied atomically. v1 ops: `album.set` (label, plural, dated_notes);
`field.add` (label, kind from the existing kind set defaulting to text, list flag,
suggest — always optional at birth; the derived key is returned by dry run before
creation); `field.set` (label, required, suggest, default, min, max — typed values,
never stringified, with explicit unset distinct from empty/false/zero); `field.move`;
`field.remove` (only when `filled` is 0); `choice.add` (label, emoji, position; value
key derived from label); `choice.set` (label, emoji — the stored value key never
changes); `choice.move`; `choice.remove` (only when `count` is 0).

**v1 refusals, teaching the path.** Renaming a field or choice key, merging choices,
removing a used field or choice, changing a field's kind, and changing the noun are
refused with the impact count ("refused: 86 restaurants are 💚 Liked") and the taught
alternative ("edit in YAML (`y`) or wait for the migration flow"). Cross-album
storage-shape clashes (a key stored as a different type in another album, like `year`)
are refused by name. Refusals are typed in JSON output so the card can paint them.

**Writer guarantees** (the correctness contract; each is a test):

1. Open/view/dry-run/cancel perform zero vault writes. A no-op apply is byte-identical,
   makes no commit, and triggers no sync.
2. The post-image is validated by the same Rust validator capture uses
   (`src/native/records/schema.rs`); refusal, never an invalid write. Dependent
   references (`index_by`, `index_facets`, group field) are updated or refused, never
   left dangling.
3. The emit rewrites only the `record:` block's value spans, in Obsidian's YAML style
   (per `report-additions`); every other byte of the note is preserved, following the
   `bob records update` span-editing precedent. Constructs the span editor cannot
   preserve (anchors, aliases, duplicate keys) produce a typed refusal pointing at
   Source mode, never a regex guess.
4. The `expect` fingerprint guards against concurrent edits: a schema changed between
   report and apply (another host, Source mode) refuses with a typed stale-note error.
5. One transaction on the same guarded write path `bob albums create` and
   `bob albums sync` use (shared vault lock, guards, one scoped commit): album
   frontmatter plus the rebuilt `[!records]` index when grouping, choices, or labels
   change. Owner `## Albums` catalog bytes stay untouched (counts are drawn live). If
   the index cannot be safely rebuilt, the apply refuses with the reason and writes
   nothing.
6. Dry run returns the diff, impact counts from a complete record scan (never a
   UI-visible subset), derived keys, and a one-line human consequence ("adds 🔁 Regular
   after 💚 Liked · no record changes"); it takes no locks.
7. The apply result includes `inverse_ops` sufficient to revert the change as a normal
   edit, keeping git history linear. A sync failure after a successful local commit
   reports "saved locally; sync failed" and retries only the sync, never the mutation.

> [!decision] commit_granularity = session The envelope and guarantees are unchanged;
> the envelope simply carries the batched ops of one card session in order, and the
> commit subject summarizes them (`albums: update eat.md (3 schema edits)`). Per-op dry
> runs still happen in the card.

Tests: a refusal-matrix fixture covering every refused op with counts; stale
fingerprint; no-op byte identity; span preservation against a hostile-but-legal album
note; post-image validation failures; index-rebuild transaction and its refusal path;
inverse-ops round-trip (apply, then apply inverse, byte-identical); dry-run
lock-freedom; human and JSON output. Document the command and envelope in
`docs/records.md`.

## header-reshape

Reshape the album header's schema surface in bob-plugins (`plugins/bob-ledger-tools`)
into the contract's collapsed-by-default design. First audit what `bob-cli-k3.7.1`
landed (see Coordination above) and reuse it; whatever landed, the end state is:

1. The scoped CSS hide of `.metadata-property[data-property-key="record"]` on album
   surfaces only, exactly as the child epic specified: restored on type change, file
   switch, unload, and non-album notes; the property is never deleted or rewritten;
   Source mode stays untouched; a missed selector means a visible raw row, never
   corruption. Keep (or land, if missing) all its fixtures.
2. The collapsed summary line, the expandable chip view, and the error/empty states,
   exactly per the shared design contract, replacing any always-open schema table the
   child epic shipped. Reading view and Live Preview, one header per source surface,
   using the notebook header (`241e`–`241g`) lifecycle patterns.
3. `⊞ Schema` opens the Schema Card read-only: full layout, counts from the report
   (`filled`, per-choice `count` from `report-additions`), lint banner for issues, and a
   footer of navigation keys, `y`, and `esc` only. No write gestures exist yet.
4. `y` — from the card or the chip view — jumps to the exact Source-mode line of the
   schema (per-field `line` when invoked on a field row, the first issue's `line` when
   invoked on a lint mark). bob's `issues[]` paint as lint marks with `message` and
   `suggestions` rendered as text.
5. The `＋ <noun>` copy affordance and breadcrumb behavior the child epic shipped are
   preserved; invalid schema disables copy with its reason, per the contract.

Follow the mobile decision callout in the shared contract. Byte-identity: no interaction
in this phase writes the vault. Tests extend the landed header tests:
collapsed-by-default, disclosure and view-state persistence per album, chip rows for
every kind and flag combination, grouping annotations, legacy chips, error and empty
states, read-only card contents and counts, `y` jump targets, lint marks, narrow pane,
theme variables, keyboard and screen-reader access, repaint idempotence, cleanup on
switch/unload. Update the album reference page in the bob-plugins docs tree (read
`docs.md` memory first) and run the full build/test/docs/sync gauntlet from Starting
evidence.

## card-editor

Make the Schema Card editable, in bob-plugins, with every write going through
`bob albums update` via the shared service's spawn conventions (one in-flight apply,
timeouts, safe argv; extend `241l`/`241m` rather than spawning ad hoc).

1. **Gestures**, per the contract keymap: `⏎` edit the selected row's label (or album
   label/plural/dated_notes on the stats line), `a` add field (label typed, derived key
   previewed before creation, kind picked from the existing set defaulting to text), `c`
   add choice (label and emoji, derived value previewed, positioned after the selected
   choice), `!` toggle required, `s` toggle suggest, `⌥J`/`⌥K` move field or choice,
   `⌃D` remove (unused only), `y` YAML jump, `u` undo, `esc` close. Defaults and numeric
   bounds edit through typed inputs that distinguish unset from empty/false/zero.
2. **Every gesture dry-runs first**: after a typing pause (debounced, never
   per-keystroke), the staged op runs `bob albums update -d` and the consequence line
   paints the result — impact counts for allowed ops ("12 restaurants have no category ·
   they'll show in bob records check"), typed refusals for refused ones ("refused: 7
   restaurants are 🚫 Not liked · edit in YAML (y)"). Enter is inert until a dry run for
   the staged op has succeeded.
3. **Enter applies** with the `expect` fingerprint from the card's report generation.
   Before apply, flush the open editor buffer through a supported save path. On success
   the card stays open and repaints from the refreshed report (schema edits come in
   bursts — a deliberate break from the Task Card's write-closes-card rule); an Undo
   toast offers `u`, which replays the returned `inverse_ops` as a normal edit. A
   stale-fingerprint refusal repaints the card from fresh data and says what changed.
   "Saved locally; sync failed" surfaces bob's message and offers retry of the sync
   only.
4. **Entry points**: the header's `⊞ Schema`; a command palette entry **Edit album
   schema** that resolves to the owning album even when invoked from a record note
   (where a new choice is most often wanted); and Command Mode, following its existing
   registration conventions.
5. **YAML round trip**: `y` from the card closes to the exact line; the card's "Done"
   affordance after a YAML excursion refuses to leave Source mode while bob reports
   schema errors, painting the lint line instead.
6. The plugin never computes diffs, validates schemas, or writes YAML; it stages ops,
   renders bob's answers, and treats all returned text as text.

> [!decision] chord = overload Register `Ctrl+Shift+P` on album notes to open the Schema
> Card, scoped so the Task Card chord's behavior everywhere else is untouched; add a
> fixture proving a task row inside an album note still gets the Task Card.

> [!decision] chord = separate Add no chord binding; the three entry points above are
> the only openers.

> [!decision] commit_granularity = session The card stages ops into a visible draft
> list; an explicit save gesture submits one envelope. `u` undoes staged ops pre-save;
> the Undo toast reverts the whole session.

Tests, using the existing DOM/editor/harness patterns: gesture-to-envelope mapping for
every op; debounce behavior; consequence and refusal painting from fixture JSON (no live
bob in unit tests); Enter inert without a dry run; stale repaint; undo replay;
entry-point resolution from album and record notes; byte-identical vault after cancelled
and refused flows (the plugin itself never writes); keyboard-only operation;
screen-reader labels; narrow pane; light/dark. Extend the album reference page with the
editing behavior and keymap, and run the full build/test/docs/sync gauntlet.

## Integration and land-agent acceptance

The land agent audits all four deliveries against this plan, fixes integration defects,
and owns the bounded final proof:

1. Extend the album e2e coverage (the scratch-vault/CDP pattern, plus offline fixture
   tests registered in `npm test`) with schema scenarios: healthy grouped album, empty
   schema, invalid schema, an unrelated `record` property on a non-album note, and the
   editable-card flow against actual bob JSON from a temporary vault.
2. Real-Obsidian acceptance, screenshots inspected and defects fixed:
   - The Properties round-trip fixture: edit `summary` through the Properties panel in
     real Obsidian and diff the `record:` block's bytes — unchanged after
     `report-additions`' emitter fix.
   - The two-pane race: a Source-mode schema edit with a delayed save between card open
     and Enter refuses with the stale-note surface, never a corrupted write.
   - No gesture writes without a dry run; cancelled and no-op flows byte-identical.
   - Label edits leave stored tokens and record files untouched.
   - A broken schema shows the error state and lint line; "Done" refuses to leave Source
     mode while errors stand; the empty schema teaches; neither impersonates the other.
   - Collapsed by default; disclosure, card, chips, counts, and consequence lines in
     light/dark at ~375px; keyboard-only operation; screen-reader labels; focus
     retention through repaints.
   - Disabling the plugin restores the native `record` row.
3. In bob-plugins run `npm run build`, `npm run docs:gen`, `npm test`,
   `npm run validate`, and `BOB_CLI_DIR=<primary bob-cli checkout> npm run docs:check`;
   retain only intended generated changes. In bob-cli run the required `just check` gate
   through `/sase_monitor`. Run `bob plugins sync` and verify deployment per
   `docs/develop/deployment.md`. A failure reproduced identically on clean base is
   recorded with evidence as a proposed follow-up.
4. Record implemented features, exact checks, screenshot/artifact references, and
   deployment outcome on this epic's bead through the SASE artifact workflow. Triage
   phase workers' `PROPOSED FOLLOW-UP:` notes into task beads via `/sase_new_task`,
   including the deferred v2 migration flow (`field.rename`, `choice.merge` with
   span-preserving record rewrites in one transaction) if no bead exists yet.
5. Run `sase bead epic-symbols` for this epic and resolve or re-key remaining entries.
   Close only this epic's own phase beads and epic bead per the prescribed completion
   workflow; never close `bob-cli-k3` or its descendants. Every turn ends through
   `/sase_final`.

> [!decision] memory_decision Only if answered yes: the land agent authors the
> `decisions:` web strand `album-schemas-edited-through-bob` through the
> `/sase_memory_write` flow — claim: album record schemas are rendered by the plugin
> from bob's report and edited only through `bob albums update`, with YAML first-class
> and Properties never a schema surface; reopens if Obsidian ships a public
> property-widget API or native nested-property editing — then runs `sase memory init`.
> If answered no, no memory file changes in this epic.

The completion criterion is the whole behavior this plan promises, deployed and
verified: the `record` row hidden on albums and replaced by the collapsed header chrome,
the chip view and Schema Card rendering every schema in the vault, and every v1 edit
flowing through `bob albums update` with previews, refusals, and Undo behaving as
specified.
