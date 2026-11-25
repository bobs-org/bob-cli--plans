---
tier: epic
title: Complete the meeting notes reference command
goal: bob ref meet lists, previews, files, and replaces event-linked meeting references
  through the existing intake pipeline with a stable JSON contract and complete docs,
  completing the assigned ref-meet-cli phase bob-cli-pw.5.
phases:
- id: preparation
  title: Prepare meeting filings and render structured reports
  size: medium
  depends_on: []
  description: 'preparation: build the internal read-only meeting request, preparation,
    identity, and report layer from the landed resolver, normalizer, and config. Implement
    list/preview data, match evidence, parent choices, dedupe/overwrite selection,
    exact input hash guards, and human/JSON output models. Test without exposing an
    unfinished public command. See the preparation section for the contract and acceptance.'
- id: first-filing
  title: Ingest a meeting dossier with its event re-root in one commit
  size: medium
  depends_on:
  - preparation
  description: 'first-filing: add create/meet_route.rs using the prepared request,
    meeting PDF profile, marker, source store, and shared intake. Extend the locked
    intake transaction for a checked in-place event-root edit and structured outcomes.
    Verify fresh filing, source preservation, queued drain, status, scoped commits,
    and failure paths.'
- id: replacement
  title: Replace meeting notes while preserving recap ownership and task identity
  size: medium
  depends_on:
  - first-filing
  description: 'replacement: wire meeting overwrite into the existing recapture/overwrite
    transaction. Preserve reference and reading-task identity, LINKS bytes, and user-edited
    recap; replace only owned recap inside the transaction, keep recap out of ref_history,
    and preserve existing coordination, rollback, and recovery.'
- id: command-and-docs
  title: Expose the complete ref meet CLI and document its contract
  size: medium
  depends_on:
  - replacement
  description: 'command-and-docs: add the clap leaf and dispatch over the completed
    services, clipboard/stdin, TTY choices and editor review, receipts and exit codes,
    catalog rows, help goldens, and all requested documentation. Verify the public
    CLI end to end. Prepare acceptance evidence for the child epic land agent; do
    not close any ancestor bead from this phase.'
proposed_by: bbugyi200.apollo.bob-cli-pw.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pw.5
create_time: 2026-11-24 15:14:19
status: done
bead_id: bob-cli-pw.5.1
---

- **PROMPT:** [prompts/202611/ref_meet_cli.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/ref_meet_cli.md)
- **PARENT:** [202611/meeting_notes_event_refs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/meeting_notes_event_refs.md)
- **BEAD:** [bob-cli-pw.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/bob-cli-pw.5.1.md)

# Complete `bob ref meet` for bob-cli-pw.5

## Objective and scope

Finish the `ref-meet-cli` phase of `plan:202611/meeting_notes_event_refs.md`: one
command lists meetings or files supplied notes as a `meetings` reference, with a review
task, an inert `notes from` relation to the event, a full owned recap, and a
meeting-profile PDF. This is a child epic under **bob-cli-pw.5**, not a replacement for
the outer epic **bob-cli-pw**.

The remaining work crosses read-only preparation, intake transaction extension,
overwrite lifecycle, and public CLI integration. Those are four bounded coding-agent
units. Dependencies are deliberately serial because the units share intake and create
types. Each phase has meaningful tests; the last phase wires already-tested services and
completes public contract coverage.

This planning turn found no prior work notes or partial implementation on bob-cli-pw.5.
Its prerequisites have landed:

- `b832e58d`: meeting resolver, matching, recent rows, parent choices, agenda notes.
- `923df1a6`: marker-carried meeting birth, full recap, LINKS, and `meetings:` config.
- `2b3817fe`: dossier print profile over the shared note-snapshot renderer.
- `src/native/meetings/` already has intake, normalization, dossier parsing/rendering,
  and recap ownership helpers. Reuse these; do not redo the earlier phases.

`bob ref meet`, its create route, and its catalog entries are absent. The current
`IntakeRequest` cannot carry a daily-note edit; `IntakeSession::ingest` computes and
commits its own path set. The overwrite executor reconstructs the live body and can
freeze authored prose into history, so meeting recap preservation requires explicit
integration there, not an after-the-fact write.

## Fixed decisions and boundaries

The outer epic's reviewed choices are final: `recap_scope = full`,
`scaffolding = rehang`, `memory_decision_record = no`, and `memory_glossary_term = no`.
Do not ask these again or add decisions frontmatter. **No memory files may be changed.**
First-use scaffolding belongs to bob-cli-pw.6. Plugin implementation and deployment
belong to bob-cli-pw.7, and Mac implementation belongs to bob-cli-pw.8. Do not close or
implement those beads here.

All code, fixtures, documentation, plans, artifacts, and commit messages are public. Use
synthetic meeting text, invented names, and temporary vaults. Never read or copy the
first real meeting's notes for this work. Do not change versions or CHANGELOG.md.

Keep the existing source-store and reference-mutation contracts: preserve the exact
Pandoc Markdown in the write-once store, acquire the existing maintenance/scan locks in
their existing order, retain coordinated PDF writes, and use scoped vault commits. No
AI, Google API write, Wispr adapter, new capture grammar, or separate note writer. Event
content is unchanged except an explicitly selected inbox/missing-root re-root.

## Final command and behavior contract

`bob ref meet [OPTIONS] [EVENT]...` belongs in the Highlights pipeline help group,
alphabetically. With `-c`, `-i`, or `-e` it files; with none it lists. `-l` explicitly
lists and conflicts with filing-only options. Clipboard and input conflict.

| Long option                       | Short | Meaning                                                                                   |
| --------------------------------- | ----- | ----------------------------------------------------------------------------------------- |
| `--bob-dir DIR`                   | `-b`  | Existing vault selection/defaults                                                         |
| `--clipboard`                     | `-c`  | Read notes through Bob's clipboard helper                                                 |
| `--day DAY`                       | `-D`  | Local day using the gcal note parser                                                      |
| `--dry-run`                       | `-d`  | Resolve and preview; no vault/store writes, locks, PDF rendering, network, or model calls |
| `--edit`                          | `-e`  | Review normalized notes in VISUAL, EDITOR, or vi; empty starting buffer if no source      |
| `--format human\|json`            | `-f`  | Human default; one JSON envelope on stdout                                                |
| `--input FILE`                    | `-i`  | UTF-8 text file; `-` reads stdin                                                          |
| `--list`                          | `-l`  | Recent meetings and filed state                                                           |
| `--overwrite`                     | `-O`  | Replace the existing occurrence's notes                                                   |
| `--parent NOTE`                   | `-P`  | Filing destination; may authorize inbox/missing-root re-root                              |
| `--producer NAME`                 | `-p`  | Override configured producer                                                              |
| `--status next\|ready\|wip\|read` | `-s`  | New task status; explicit override on overwrite                                           |
| `--title TITLE`                   | `-T`  | Override derived reference title                                                          |
| `--transcript`                    | `-t`  | PDF/source appendix only; never the recap                                                 |
| `--window DAYS`                   | `-w`  | Lookback, default configured 7 days                                                       |

Also support hidden `--expect-sha256 HEX`, a client guard without a required short
alias. Help includes sorted long options, inherited `-h`, examples, modes, conflicts,
and failure hints. Do not rename any existing command.

### Meeting and destination selection

- Accept bare/path `note#^id`, aliased wikilinks, `note:id`, leading `~`, a Google
  Calendar URL carrying `eid`, or title-query words. Join query arguments as the
  existing parser expects. Explicit identities win over query and automatic matching.
- Reuse `resolve_event`, `recent_meetings`, `auto_match`, `content_words`, and
  `meeting_parent`; expose their types with narrow crate-visible re-exports.
- Automatic candidates have started and are not cancelled. The landed 75%/two-word
  strong-match and runner-up-below-50% rules remain unchanged. Preserve evidence and
  recurring-title ambiguity. An explicit future/cancelled event is permitted with a
  warning. Missing block IDs need an actionable refusal.
- List newest first, with date, span, title, root, missing/filed state, and EVENT token;
  count meetings needing notes in the header. Respect day/window and explicit query
  filters without touching clipboard or stdin in list mode.
- Use `-P`, otherwise the event root through `parent_notes::resolve_parent`. An inbox,
  missing, unresolvable, or ineligible root needs a choice; return candidates off TTY.
- Re-root only a missing or inbox root when a valid explicit/prompted parent is chosen.
  An override of an ordinary root changes only the reference's destination. An
  unresolvable but non-inbox root is not automatic permission to rewrite the event.
- Derive stem as meeting `YYMMDD` plus at most six event-title content words. Reuse
  target validation and collision allocation (`_2`, etc.); use `meeting` as the suffix
  when the event title has no content words. Derived title is
  `Cedar Labs review — Meeting Notes (Tue 2026-11-24, 14:00–14:30)` in the synthetic
  example. Title override does not redefine meeting identity.
- One reference per occurrence: Google event ID when known, otherwise resolved event
  note/block identity. Use the filed index and existing recapture preflight; add a
  narrowly scoped meeting identity only where URL identity is insufficient. URLs with
  different sharing parameters must not create duplicates. Different recurring
  occurrences must remain distinct. Do not conflate a same-stem collision with dedupe.

### Notes, config, and editing

Read at most 1 MiB with existing `read_notes`; retain its exact-input-byte SHA-256
before BOM/newline/trailing-space normalization. Reject empty, binary, unreadable,
invalid UTF-8, and oversized notes with the specified errors. Compare the expected hash
before editor launch, rendering, or mutation. Read a chosen source once per invocation,
and file that snapshot. Retain the original source hash in the receipt when a user
subsequently edits the draft.

Reuse `MeetingsConfig` and `load_meetings_config` in `src/native/config/meetings.rs`:
producer unset, relation label `notes from`, status `next`, window 7 (1–365). The landed
config accepts next/ready/wip/read and normalizes labels; additionally reuse
`meeting_birth::meeting_relation_label` to refuse effect-bearing labels before writes or
rendering. Do not implement a parallel config parser.

Keep normalization structural and deterministic. The full recap excludes transcripts and
escapes notes-derived tokens. Without `-t`, omit transcript bytes from both the PDF
input and saved source; with `-t`, include the PDF appendix, never the note. Preserve
the landed word-order/round-trip properties.

Editor review uses a temporary, human-editable normalized draft and the gcal-note
scissors convention, with trusted identity/instructions outside the editable notes.
Parse the accepted body back into notes and set dossier `edited` for user edits, so the
PDF provenance does not incorrectly claim unchanged wording. Do not accept
editor-authored changes to trusted event/parent metadata. Empty output is `no_notes`;
editor failure/cancellation does not file. Match an editor-only draft after obtaining
its notes. For `-d`, do not launch an editor: preview supplied notes; an empty
editor-only dry run returns `no_notes` with a hint to supply notes or drop `-d`.
Document this combination so dry-run remains free of file writes.

### Output and failures

Use `REF_SCHEMA_VERSION` and `generated_at` from `ref_library/output.rs`, without a
version bump. Success has `ok`, `schema_version`, `command: "ref meet"`, and
`generated_at`. Error envelopes retain existing `error: {code,message,hint?}` and add
the relevant root/candidates/existing details. Put diagnostics on stderr in JSON mode;
never print create/intake human receipts into JSON stdout.

List payload: `meetings[]`, each containing `ref`, `path`, `block_id`, `date`, `start`,
`end`, `title`, `root`, `root_is_inbox`, `status`, optional `event_id`, and
`notes: null | {note,task,status}`. Keep vault-relative paths and the existing task
locator spellings consistent across list, preview, success, and errors.

Preview/file payload:

- `outcome: would_file | filed | queued`;
- `meeting`: the same row shape as list;
- `match: {method: explicit|query|auto|picked, evidence: [...]}`;
- `filing: {parent,parent_kind,reroot?:{from,to},title,stem,note,pdf, task:{destination,block_id,status},relation:{label,target}}`;
- `notes: {sha256,shape,producer,outline:{tldr,next_steps:[{owner,action}], sections:[{heading,items}]},transcript: kept|omitted|none}`;
  outline section `items` is the count, not a second copy of the whole section;
- `warnings[]`, optional `commit`, and optional `queued` detail. A queued result
  describes planned task/note paths without claiming they already exist.

Exit 0 for completed list/preview/file/queue. Exit 1 for operational/input failure. Exit
2 for `no_meeting`/`ambiguous_meeting` needing selection, `needs_parent`, or
`already_filed`. Required codes: `notes_unreadable`, `no_notes`, `no_meeting`,
`ambiguous_meeting` with evidenced candidates, `needs_parent` with root and parent
candidates, `already_filed` with `{note,task}`, and `notes_changed`. Map underlying
operational errors to useful typed codes/hints without converting incomplete writes or
refused overwrites into success.

Human preview shows meeting/evidence, destination/title, outline, and warnings. Success
shows note, reading task/status, relation, PDF/pages, commit, and the normal reconcile
timing for the event mirror. Queued output explicitly says queued, names the PDF and
`bob ref scan`, and never says filed. Report skipped re-root on queue, recurring-series
root learning, and `gcal sync -F` resetting that root where relevant.

## Phase preparation

Add a focused internal preparation module under `highlights_ref/` (for example
`meet/prepare.rs`, `meet/model.rs`, and `meet/output.rs`). Keep the public CLI
unregistered until command-and-docs. Avoid duplicating the domain model: presentation
DTOs wrap the existing `MeetingRow`, `MeetingNotes`, `ResolvedParent`, and recapture
types.

Define a typed prepared filing shared by first-filing/replacement/CLI: resolved event
identity, notes/hash, match evidence, resolved parent, optional guarded root edit,
effective config/status/title/stem/target plan, optional existing-reference selection,
and warnings. No borrowed scratch state or clap dependency belongs in this type.
Preserve whether status/title/parent were explicitly supplied so overwrite keeps the
existing task unless explicitly changed. Provide typed choice/refusal data instead of
prompting or printing inside preparation. Include serialization and human report
functions over these data, with consistent error rendering.

Preparation may read vault/config but must not create directories, take locks, sync,
persist state, render, or spawn an editor. Use the existing plan-target helpers only
where they are read-only. Separate selection from execution so CLI pickers can feed an
explicit chosen event/parent back through validation. A caller that picks must retain
`method: picked` and the chosen candidate's evidence.

Tests: synthetic explicit forms and URL identity, query and auto match, ambiguity and
missing-ID errors, day/window, config precedence and effect-label rejection, parent
candidates, status, naming collisions, filed identity and replacement selection, hash
mismatch, transcript outline, and JSON data. Assert unchanged vault tree and lock state
and that forbidden subprocess hooks are not called. Reuse existing resolver tests rather
than copying their entire matrix. Supply helpers usable by later phases.

Done when focused tests and the repository gate pass and the typed service is ready for
both write routes. Remove obsolete suppression attributes only for newly used symbols;
do not introduce blanket lint exemptions.

## Phase first-filing

Add `src/native/highlights_ref/create/meet_route.rs`, modeled on podcast_route's
stamping/store/intake integration. Construct `CreateOptions` directly; never run notes
through `prepare_target`, which treats in-vault Markdown as a snapshot.

Render the prepared dossier through `meetings::render_meeting_pdf` and shared render
filters/scratch. Stamp title, parent, status, ID, canonical event link, event_id when
known, meeting_start/end, notes_producer, and the row's Calendar source_url. Always use
type `meetings`. Preserve PDF Info and typed render warnings. Persist exact rendered
Markdown and its render-time catalog through existing source-store helpers, with the
installed PDF digest and expected pointer for birth.

Call shared intake with producer `bob ref meet`. Prefer a structured route result using
`IntakeOutcome` over `finish_create_intake`, which prints/discards facts needed by the
CLI. Keep existing producers' behavior unchanged. Fresh success must produce the
already-landed reading task, one LINKS child, managed embed, full recap, and Highlights
region; the route must not author a second copy of any of them.

Extend intake with an optional typed event-root mutation, not an arbitrary closure that
writes after commit. Under its existing locks and post-pre-sync reread, verify the exact
event row and allowed root transition, use/expose `gcal::render::set_row_metadata` for
the in-place edit, and add the daily note to dirty checks, preimages/rollback handling,
and the scoped path set. Never use `task_root::set_task_root`: that changes the
Google-hashed buffer/badge span. Recheck meeting identity/dedupe and occupied
destinations before mutation. A changed event must cause a typed refusal/queue without
applying a stale line edit.

Apply the re-root only with successful ingest. If intake queues (lock, sync, config, or
guard condition), keep the existing queued-PDF mechanism, leave the daily note
untouched, and report the skipped re-root. Later `ref scan -i` births the stored meeting
marker/source normally; it does not replay an abandoned re-root intent. Include source
files, PDF, note, task parent, and daily note when applicable in the single scoped
ingest commit, subject `bob ref meet: ingest meetings/<stem>`. No post-success cleanup
write may make that claim false. Keep honest incomplete, commit-warning, and
push-pending diagnostics from intake.

Tests use a temp vault-sync pair and existing PDF/Pandoc/XeLaTeX test helpers: ordinary
parent first filing, each status including read, PDF marker/source pointer, no snapshot
routing, exact input source preservation, transcript inclusion/exclusion, one
commit/path set and one-commit revert, inbox/missing-root re-root with unchanged
buffer/badge bytes, ordinary-root override without re-root, queued result and later
drain, stale root/identity, and failures at the new mutation boundary. Assert no stray
root edit on refused/queued ingest. Run existing ingest and meeting-birth tests.

## Phase replacement

Use preparation's existing-reference selection and the existing create recapture
preflight/overwrite executor. Retain note path, recorded ID, stem, reading-task
residence and block ID, LINKS bytes, and default task status. Only an explicitly
provided status updates status. Dedupe refusal without `-O` must precede rendering. With
no existing occurrence, `-O` follows the existing create force semantics rather than
allocating an unrelated overwrite target.

Add narrow meeting-aware request/staged data to the overwrite transaction. Render and
stage the replacement PDF/source privately, then use the executor's locked recheck,
source-pointer update, PDF coordination, verification, scoped commit, rollback, and
crash recovery. Parameterize its hard-coded create producer where necessary so meeting
receipts and commits name `bob ref meet`; preserve existing create subjects.

Compute recap ownership from the live note at the transaction's checked preimage.
Replace the region only when `recap_state`/`replace_recap` permit it. An edited recap
stays byte-for-byte with a warning. Keep the recap live when annotated overwrite
archives the old reading; it must not migrate into the frozen `ref_history/` note. Keep
ordinary annotation history and unrelated authored-note behavior unchanged.
Missing/malformed markers must not authorize deletion of user text. Put all recap
changes inside the transaction and its recovery state; never patch it after commit.
Preserve LINKS even when an orphan task was adopted earlier.

Tests cover owned recap reseed, edited recap preservation, absent/malformed region
handling, annotated-history separation, unchanged task/link identity, explicit versus
default status, source store history, renamed existing references, Google and manual
event dedupe, coordination hold, stale selection, and rollback/recovery after recap
changes. Confirm non-meeting overwrite regressions remain green. Reuse the executor's
test-only failpoints; do not add a public test/failure injection option.

## Phase command-and-docs

Add `highlights_ref/meet_cli.rs` (or a small directory), builder registration and
`HELP_GROUPS` in `highlights_ref/cli.rs`, module/dispatch in `highlights_ref/mod.rs`,
and the exact flag contract above. This layer coordinates the existing preparation,
first-file, and replacement services; it must not implement another vault writer.

Connect `capture_clip::clipboard::read_clipboard`, file/stdin sources, and editor
review. Avoid consuming notes stdin as picker input. TTY ambiguity uses gcal's numbered
`Which? [1] ›` convention, Enter selects 1, q cancels. Parent prompt is
`File under which note? ›`, with `resolve_parent` hints. Off-TTY and JSON clients
receive structured choice errors promptly. Editor subprocess output must not corrupt
JSON stdout. Preserve the draft across recoverable choices and mark picked matches.

Catalog rows in `catalog/table.rs`: `ref-meet` writes, preview-first, preview `-d`,
EVENT text slot, adapter `ref-meet`, invalidates references/tasks; `ref-meet-list`
reads, fixed `-l`, report presentation. Set preview/action arguments consistently so the
normal filing row actually supplies a notes source. Ensure leaf coverage,
slot/fixed-argument, command-mode, and catalog JSON invariants hold.

Documentation, with fictional examples only:

- `docs/meeting-notes.md`: observable contract, recap ownership, statuses, overwrite,
  dry-run/editor interaction, queue, re-root, undo, and JSON examples.
- `docs/guides/file-meeting-notes.md`: CLI flow and the outer epic's Mac/Obsidian
  gestures, explicitly conditioned on clients that expose them; do not invent a Mac
  shortcut that bob-cli-pw.8 has not selected.
- `docs/reference/configuration.md`: existing meetings keys, defaults, valid ranges,
  precedence, and effect-label prohibition.
- `docs/ref.md`, `docs/concepts/reference-library.md`, and
  `docs/reference/json-outputs.md`: reference type and contract links.
- Navigation through the established docs generator and `mkdocs.yml` conventions; run
  `just docs-gen` for generated CLI reference/help and contract indexes.

Public integration tests in `tests/cli/` cover every final outcome using synthetic
vault-sync pairs and renderer stubs: explicit/query/auto/picked matches, ambiguity exit
2 and evidence, parent choice then same-commit root edit, already_filed then
owned/edited overwrite, bare/explicit list, each input source, expected-hash refusal,
editor edits/cancel/failure/empty buffer, no transcript leakage, dry-run no writes or
locks/subprocess render, queued receipt/drain, JSON-only stdout, option conflicts, help,
and catalog. Use pseudo-TTY or injected I/O seams for prompts, never interactive test
hangs. Prefer checking actual commit membership and note/PDF identity over only matching
a success string. Reuse service tests rather than reimplementing them.

Run focused CLI/catalog/help tests, `just docs-gen`, `just docs-check`, and
`just check`. Measure dry-run on a realistic synthetic fixture; the target is under 300
ms on the vault, with no flaky timing assertion in the suite. Keep the existing real
renderer tests; no repetition of the earlier profile's visual redesign is required.

## Completion and bead ownership

Each child phase closes only its own assigned bead after verification and after
`sase bead epic-symbols <its-id>` has no leftovers (resolve or re-key to an open owner).
No child phase closes bob-cli-pw.5, this child epic, or bob-cli-pw. The command-and-docs
phase leaves complete evidence for the child epic land agent.

The **child epic land agent** checks all four phases, final public contract coverage,
and required gates, then resolves/re-keys remaining symbols reported by
`sase bead epic-symbols bob-cli-pw.5`. After completing and closing its own child epic
and all descendants, so the descendant-close guard is satisfied, it closes
**bob-cli-pw.5** with
`sase bead close bob-cli-pw.5 --note "<verified implementation and checks>"`. This is
the specifically authorized child land-agent closure of the original assigned phase,
only after its whole scope is complete. It must not close outer epic bob-cli-pw. The
parent's existing dependents then continue their own work.

Use `/sase_monitor` for long verification, as project instructions require. Record
unrelated discovered issues as `PROPOSED FOLLOW-UP:` notes on the assigned phase; never
create task beads from a phase. A check failure reproduced identically on a clean base
is documented with evidence (and an existing task ID when known), not grounds to leave a
completed phase open. No manual status changes or reopening of bob-cli-pw.5. Re-read the
assigned bead and current source before implementation so any newly landed work is
preserved.
