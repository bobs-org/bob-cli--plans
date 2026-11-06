---
tier: epic
title: Gmail compose, reply, draft, and confirmed send
goal: Complete bob-cli-hg.12 with editable compose/reply notes, exact draft previews,
  confirmed draft-id sends, crash recovery, and preserved sent notes.
phases:
- id: compose-notes
  title: Compose note model and new/reply commands
  size: medium
  depends_on: []
  description: 'compose-notes: implement the source-region model, safe note creation,
    reply/all insertion, CLI exclusions, and preservation tests in section 4.1.'
- id: compose-render
  title: Outgoing Markdown renderer and attachment snapshots
  size: medium
  depends_on:
  - compose-notes
  description: 'compose-render: implement conservative HTML/plain output, syntax refusals,
    flattened links, and validated attachment snapshots from section 4.2.'
- id: gmail-drafts
  title: Draft transport, persistence, and exact preview
  size: medium
  depends_on:
  - compose-render
  description: 'gmail-drafts: implement remote create/update/readback, compose scope
    checks, durable identity, and canonical preview/digest without exposing send;
    see section 4.3.'
- id: send-recovery
  title: Durable send state and recovery engine
  size: medium
  depends_on:
  - gmail-drafts
  description: 'send-recovery: implement durable state transitions, serialization,
    draft-id-only dispatch, and recovery classification with fault tests; keep the
    engine internal as in section 4.4.'
- id: send-finalize
  title: Confirmed send command and sent-note finalization
  size: medium
  depends_on:
  - send-recovery
  description: 'send-finalize: expose confirmed send, integrate recovery and sent-card
    refresh, promote compose notes in place, and preserve authored edits as in section
    4.5.'
- id: compose-docs
  title: Compose guide and consumer contract acceptance
  size: small
  depends_on:
  - send-finalize
  description: 'compose-docs: finish the guide and Bob Mail JSON fixtures, audit acceptance
    coverage, and supply land evidence from section 4.6; do not close ancestor beads.'
proposed_by: bbugyi200.apollo.bob-cli-hg.12
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hg.12
create_time: 2026-11-06 07:12:27
status: wip
bead_id: bob-cli-hg.12.1
---

- **PROMPT:** [prompts/202611/gmail_compose.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_compose.md)
- **PARENT:** [202611/bob_gmail.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_gmail.md)
- **BEAD:** [bob-cli-hg.12.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hg/bob-cli-hg.12.1.md)

# Gmail compose, reply, draft, and confirmed send

## 1. Scope and observed baseline

This is a child epic implementing **bob-cli-hg.12**, the `gmail-compose` phase of
`plan:202611/bob_gmail.md`, especially sections 3, 4.2, 4.5–4.8, and 5.12. The assigned
phase is already in progress. Its audited read showed no earlier work notes; do not
reopen or manually change its status. The planning checkout was clean at `fcc4efa5`.
Check current code and bead notes again before implementing, because other Gmail phases
may land meanwhile.

The existing implementation already supplies:

- `src/native/gmail/{cli,mod,config,login,gog,model,resolve,prepare,save,list}.rs`:
  login including `-s compose`, isolated `bob-gmail` authentication, typed thread reads,
  body preparation, transactional save, and offline list.
- `src/native/gmail/note/{layout,parse,merge,path,index,hub}.rs`: managed conversation
  regions, collision-safe paths, inert inbound cards, and preservation of authored
  material. `save::plan_prepared_saves` exposes a reusable staged boundary, but it
  currently requires an indexed thread note and therefore cannot directly promote a
  compose note.
- `tests/gmail_support/{mod.rs,fake_gog.py}`: isolated vaults, configuration, fake
  authentication and mail reads, invocation recording, and race-test helpers.
- `tempfile` and `sha2` dependencies. `pulldown-cmark` is not yet in Cargo.toml.

There are no `new`, `reply`, `draft`, or `send` commands, outgoing renderer, or draft
transport. The existing card renderer computes whether a message is the account's own,
but currently emits `[!mail]` for both directions; this epic owns the required
`[!mail-sent]` correction when it integrates sent cards.

The work needs separate bounded agents: authored Markdown and preservation, outgoing
rendering, remote draft synchronization, recovery from ambiguous send results, and
sent-note finalization each have different failure boundaries. The dependency chain is
intentional: it establishes tested interfaces before irreversible send becomes public.
No implementation is performed by the planning turn.

## 2. Decisions and limits

Inherit the parent epic's final reviewer choices without asking again:

- `default_body = full`, `link_label = subject_who`.
- No decision-record or mail-note glossary edits. The only authorized memory change in
  the parent epic is `glossary:inbox-file`, owned by a different phase. **This child
  epic edits no memory files.**
- Gmail remains the mailbox. Sending is explicit, through a Gmail draft, with the
  confirmation bound to the exact preview. No direct `gmail send`, `gmail reply`,
  `messages.send`, SMTP, mailbox mutations, or automatic retries with a new draft.
- Agents use synthetic fixtures and fake gog only. Never invoke real Gmail operations
  against the user's vault or `bob-gmail` grant. A guide gives Bryan the live checks.
- All vault mutations honor `gmail.writer`, use the existing maintenance lock and
  preimage-checked staged writer, and commit only their touched paths. Network and TTY
  waits occur outside the vault maintenance lock. Use a separate compose-operation lock
  to serialize Bob's draft/send work across processes.
- Follow the read versions of `cli_rules.md` and `docs.md`: alphabetic help, short
  aliases, catalog exclusions, and generated CLI docs. Do not edit the package version
  or CHANGELOG.md. Use `feat(gmail): ...` for implementation commit messages.
- Bob Mail's UI is owned by bob-cli-hg.13. This epic provides its CLI/JSON contract and
  synthetic fixtures; no plugin, capture, or macOS app changes belong here.

### Verified gog compatibility

The installed gog is v0.44.0 (`cf42bb7c`). Its `drafts create`, `update`, `get`, and
`send` help was inspected without authentication. The upstream repository was opened
through `/sase_repo`; source at tag **v0.44.0**, rather than its newer default branch,
was inspected in `internal/cmd/gmail_drafts.go`.

Important concrete constraints for implementation:

1. gog rejects passing `--thread-id` and `--reply-to-message-id` together. Use
   `--reply-to-message-id <confirmed Gmail message id>` for an exact reply target; it
   establishes the thread and RFC reply headers. Verify the returned `threadId` against
   the selected thread. Add `--reply-all` for that mode. This implements the parent's
   threading intent with a supported invocation, rather than reproducing its impossible
   two-flag example. Never substitute an RFC Message-ID for the Gmail message ID
   accepted by this flag.
2. `--body-file` and `--body-html-file` are supported. Use private temporary files for
   both; bodies never appear in argv. `--signature` is supported for the configured
   `gmail` signature. Read back the actual draft to include its resolved recipients,
   From, and signature in the preview.
3. Update preserves omitted recipients and attachments. Explicitly clear empty Cc/Bcc
   and use `--clear-attachments` when the declared list becomes empty; otherwise removed
   information could remain in the sent message. Set the complete intended recipient
   state, with reply-all resolution handled deliberately.
4. Create/update JSON has `draftId`, `message`, `threadId`, and reply-context fields;
   get JSON wraps the resource in `draft`. Get supports `--format full|raw`. Readback
   must never use `--download`, which writes a gog cache. Use full MIME data or raw
   readback entirely in memory to verify all alternatives and attachments.
5. `drafts send <id>` submits only the existing draft ID. Google deletes a sent draft
   and creates a different message with the SENT label; updating a draft can also change
   its underlying message ID. A missing draft alone is not proof of a send. See Google's
   [draft lifecycle](https://developers.google.com/workspace/gmail/api/guides/drafts)
   and
   [draft send API](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.drafts/send).

These findings require no gog source changes or extra Gmail scopes. Implementers who
need further upstream source must open it with `/sase_repo` themselves.

## 3. Shared implementation contracts

Suggested module boundary: `src/native/gmail/compose/` with `source`, `render`,
`attachments`, `draft`, `preview`, `state`, `send`, and `finalize` modules. Names can
change when existing code supplies a better fit; preserve the responsibilities below.
Keep sensitive body values free of automatic Debug/error serialization.

### Source and edit boundaries

A parsed `ComposeSource` represents either a standalone compose note or one reply in a
valid saved thread. It carries the selected region's byte ranges, source fingerprint,
authored sending text, recipients/subject/attachments as applicable, and exact reply
identity. Source fingerprints bind sending inputs, not unrelated edits in `## Notes`.
Every actual write still compares the complete note preimage and preserves unrelated
authored edits when retrying.

Compose frontmatter includes `parent: "[[mail]]"`, `type: "[[mail]]"`,
`cssclasses: [bob-mail]`, `status: draft`, `to`, `cc`, `bcc`, `subject`, and
`attachments` as vault-relative paths. The body under the H1 starts empty. Use the
existing subject slug, timezone, year directory, and collision allocator at birth; the
path never changes after send. Unknown authored frontmatter survives later edits.
Recipient fields accept a documented scalar list or YAML sequence; malformed headers,
control characters, and CR/LF injection fail before any network mutation.

Reply headings are exactly `## Reply` or `## Reply all`, after the managed block. Use
Markdown structure to distinguish headings from fenced code, callouts, or examples.
Multiple reply headings, malformed managed regions, or conflicting draft markers refuse
with typed errors. A reply is bounded by the next level-1/level-2 heading or EOF.
Compose bodies exclude an authored `## Notes` section and its descendants; they must
never absorb Notes text into an outgoing message. Preserve all unrelated sections.

### Draft identity, preview, and confirmation

Store `gmail_draft_id` in a compose note, or `<!-- bob:gmail:draft <opaque id> -->`
immediately under a reply heading. Validate and encode opaque IDs so they cannot corrupt
markers or command arguments. Pin the account with draft metadata; a config-account
change must refuse rather than updating another account's draft. Do not make compose
notes appear as indexed saved threads before send.

An internal `PreparedDraft` contains the durable draft ID, exact thread/message/RFC
identities, source fingerprint, and the canonical preview. The preview is derived from
the **stored** draft, with all required bodies hydrated in memory; it never silently
falls back to a snippet or a missing MIME part. Do not reuse the inbound renderer to
construct outgoing content: it neutralizes syntax, folds quotes, and omits drafts.

The public `send -d -f json` contract is a schema-1 success object with
`status: preview`, `note`, `draft_id`, `digest`, and `preview`. `preview` includes:

- `account`, actual `from`, ordered `to`, `cc`, and `bcc` lists;
- `subject`, `reply_target` (null or thread ID, Gmail message ID, RFC Message-ID, mode);
- exact `text_body`, an HTML-content SHA-256, and explicit signature mode;
- `attachments`, each with vault path, transmitted filename, byte length, and SHA-256;
- `flattened_links`, each with original target and resulting text.

Define canonical serialization and digest with a versioned typed struct, fixed field
order, UTF-8, and unambiguous list semantics. Include the note identity and every
effective content/envelope field in the canonical digest. Exclude volatile request
times, MIME boundaries, and newly generated message IDs so unchanged content can be
previewed and confirmed across invocations. HTML-only changes must invalidate the digest
even when plain text is unchanged. Never omit Bcc or signature changes.

`send -d` intentionally creates/updates and persists a draft before showing it; it means
**do not send**, as specified by the parent. Document this difference from `save -d`. It
still requires the writer host and compose grant. It never records a send intent or
calls the send adapter. A non-TTY send without `-c` likewise leaves the draft intact,
exits 2, and supplies a shell-safe exact confirmation command. Return preview bodies
only in the intentional preview success payload/TTY display; errors, logs, diagnostics,
journals, and durable recovery state remain body-free.

Immediately before a permitted send, recheck the source inputs, attachments, and stored
draft contents. A stale digest exits 2 with `confirmation_mismatch` and a re-preview
hint. A remote edit between preview and confirmation must be detected before overwriting
that edit; report a conflict/re-preview requirement rather than silently replacing it.
No generic force option or environment variable bypasses confirmation. Google offers no
atomic compare-and-send precondition here: document the remaining possibility of an
external Gmail edit after the final readback, and do not claim global exactly-once or
cross-host locking.

### Durable send state

Use one small reserved, versioned `gmail_send` frontmatter object for the active
compose/reply operation. It contains metadata only: operation ID, stage, account, draft
ID, source and preview digests, draft message/RFC identities, thread ID, subject hash,
attempt time, and acknowledged sent IDs when known. It can also store the last known
remote content fingerprint to detect edits. Avoid raw subject text, recipient addresses,
bodies, MIME, or attachment bytes in this state.

The state lives with the note so crash recovery is not lost with one host's local cache.
Host-local locks under the existing Gmail state directory serialize processes on the
same note/draft. Do not hold `bob_sync.lock` while waiting for Gmail or user
confirmation. Before network send, durably persist `sending` under the maintenance lock
and revalidate its preimage. If that persistence fails, do not send.

Recovery always precedes rendering, updating, or creating any draft when an operation is
`sending`, acknowledged, or otherwise unresolved. Recovery has three outcomes:

| Observation                                                     | Behavior                                                                                                                        |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Recorded draft still exists                                     | Return to draft state; require a fresh preview/confirmation before another attempt. A recovery invocation itself does not send. |
| Draft absent and exactly one matching SENT message proven       | Record acknowledged sent identity and run finalization only.                                                                    |
| Missing, ambiguous, failed lookup, or no matching SENT evidence | Report `outcome_unknown` or the typed lookup failure; retain state and never create/update/send a replacement.                  |

Prefer the recorded RFC Message-ID for sent lookup. If it is unavailable, use the
recorded thread and subject hash with SENT status and sufficient attempt evidence to
reject old, unrelated, or duplicate candidates. Subject similarity or recency alone
never proves identity. If the API cannot establish a unique sent result, stay unknown.
An acknowledged send whose local refresh fails remains acknowledged and is retried only
for local finalization. A completed operation is a no-op on repeated send; starting a
later explicit Reply starts a new operation.

## 4. Implementation phases

### 4.1 compose-notes — local source model and commands

Implement `new` and `reply` in `cli.rs`/`mod.rs`, with these public spellings:

```text
bob gmail new [-b DIR] [-B|--bcc LIST] [-c|--cc LIST] [-f human|json] [-s|--subject TEXT] [-t|--to LIST]
bob gmail reply NOTE [-a|--all] [-b DIR] [-f human|json]
```

Add long `--bob-dir`/`--format` options with the existing aliases, complete help,
examples, environment, and catalog exclusions. Do not expose draft/send stubs. Return
schema-1 JSON with `ok`, `status`, `note`, and a cursor location for Bob Mail; document
line/column indexing explicitly. New/reply are local vault operations and need no
network login. Empty To/Subject is valid while authoring; draft/send later require valid
sending inputs.

Build the source parser and targeted edit functions. Create hub files only when missing
by reusing the save/hub conventions. Validate in-vault regular paths and symlink
containment. Apply writer refusal before writes. Use staged writes and exact-path
commits, preserving unrelated dirty vault paths. A second reply refuses without
modifying the existing one.

Tests: new default and populated YAML, paths/collisions/timezone/custom folder, quoted
and Unicode subjects, reply/all headings, existing reply refusal, malformed
notes/markers, fake headings in code, writer mismatch, symlink/traversal refusal, source
extraction that excludes Notes, stale preimages, unrelated text and dirty-path
preservation, zero gog calls, JSON/help/catalog coverage. Extend `docs/gmail.md` for
these commands and regenerate CLI docs.

### 4.2 compose-render — outgoing Markdown and attachments

Add `pulldown-cmark` without changing package versions. Render a documented conservative
subset to inline-styled HTML and useful plain text: paragraphs, headings, emphasis,
lists, safe links, blockquotes, and ordinary code. Escape raw HTML rather than executing
it; reject unsafe link destinations and remote image embedding with clear diagnostics.
Never run Dataview, Tasks, or Templater, fetch URLs, or traverse wikilinks to gather
content.

Flatten `[[target|alias]]` to alias, or unaliased links to their visible target; collect
the flattened notices deterministically. Refuse actual task checkboxes, block IDs,
inline fields/inline queries, Dataview/Tasks fences, and Templater syntax with a kind
and location rather than body excerpts. Test literal examples in escaped or ordinary
code independently of active syntax. Reject `![[embeds]]` unless they refer to an
explicitly declared attachment; accepted attachment embeds become a visible filename
reference plus the attachment, not expanded note content.

Validate attachment paths against the vault, including symlinks, non-files, missing
files, and control characters. Snapshot validated bytes once into a private temporary
directory (0700, files 0600) and hash those exact bytes; gog must read the snapshots,
not race-prone originals. Retain original transmitted basenames. Private body and
attachment staging is the narrowly scoped temporary exception required for transport;
RAII cleanup removes it on success and error, and it is never a durable body cache.

Expose a pure render result and an attachment-preparation boundary consumed next. Tests
cover each refusal, no Notes leakage, empty/multiline/Unicode content, wikilink notices,
HTML escaping, safe URLs, multipart-intended plain/HTML equivalence, declared versus
undeclared embeds, path safety, permissions/cleanup, and attachment changes. Update the
contract's supported syntax section. If an intermediate lint allowance is needed, keep
it narrow and tied to the consuming open phase.

### 4.3 gmail-drafts — remote draft and preview boundary

Extend `gmail/gog.rs` with typed draft create/update/get adapters. Keep ordinary reads
unchanged. Read calls retain `--readonly --gmail-no-send`. Draft create/update omit
`--readonly`, retain `--gmail-no-send`, and allow only the particular `gmail.drafts.*`
operation needed. All use `--client bob-gmail`, an explicit account,
`--no-input --json --color never`, and configured timeouts. Redact raw gog errors before
reporting them. Validate the isolated grant supports compose; a read-only grant returns
the existing Mac `bob gmail login -s compose` hint, without reconsenting automatically
or widening scopes.

Expose `bob gmail draft NOTE [-b DIR] [-f human|json]`. For replies, confirm the note's
account/thread and select a non-DRAFT reply target; pass its Gmail message ID and
reply-all mode as described above. Verify returned threading. For standalone drafts, set
recipients explicitly and avoid any inherited reply context. Always rebuild both
plain/HTML alternatives from the authored region so updates do not duplicate the
signature. Honor `gmail.signature: gmail|none`.

Persist the returned draft ID before reporting success, then append a body-free
`gmail.draft` event with note and ID. Reuse the same ID for updates. Handle failure
between remote creation and note persistence by retaining enough body-free recovery
metadata to identify that known remote draft; never report successful persistence or
silently create another draft on retry. An unknown remote-create result is an explicit
conflict requiring review, not permission to loop creates. Hold a compose-operation lock
around this sequence and recheck source state after network I/O.

This phase introduces the minimal durable draft state and lock that send-recovery will
extend. Persist a `creating` operation marker before remote creation; if a crash loses
the returned ID, keep that marker and report the unresolved create instead of creating
again. When the ID is known but the authored region changed, patch only the ID/state
into a freshly re-read note, preserve the edit, and report that the draft needs
updating. Never make phase 3 depend on a state helper that only phase 4 adds.

Read the stored draft and implement `PreparedDraft`, the canonical preview, remote
fingerprint, and digest contracts in section 3. Obtain complete data for signature,
envelope, both MIME alternatives, and attachment verification without caching raw MIME
or accepting the inbound renderer's lossy output. Reuse safe MIME decoding primitives;
add a bounded parser dependency if raw readback requires it. Fail closed if exact
content cannot be verified. This phase does not expose `send`.

Extend fake gog with persistent create/update/get behavior matching v0.44.0 JSON,
signatures and reply-all resolution, attachment-clearing semantics, changing draft
message IDs, missing scope, and injected failure points. The fake should reject both
mutually exclusive threading flags together and all forbidden direct-send commands.
Check temp-file contents/permissions while the fake runs, with only synthetic assertions
retained.

Tests: initial and repeated draft, compose/reply identity persistence, threading and
reply-all recipients, empty recipient clearing, attachment replacement/removal,
signature once, draft readback mismatches, account switch, missing scope, network and
note-write failures, no source loss, no body in error/journal/argv, canonical digest
stability, and changes to every digest-bound field. Add draft docs and help.

### 4.4 send-recovery — internal delivery engine

Implement the durable state transition helpers, operation lock, and a single narrow
`drafts send <recorded id>` adapter. The send adapter enables only `gmail.drafts.send`;
it omits the no-send/readonly flags solely after the caller has provided a validated
confirmation result. Avoid exposing a general mutating gog runner or direct-message send
path. Verify the v0.44.0 send JSON shape from source before freezing its decoder.

The engine receives a prepared, approved snapshot, persists `sending`, rechecks the
source/attachments/remote draft immediately before dispatch, sends once, and persists
the returned sent message/thread identities before requesting finalization. Use an
internal outcome enum distinguishing draft-present, unknown, acknowledged,
finalization-pending, and complete. Recovery from section 3 runs before any
create/update/render when previous send state is unresolved. Do not let a failed refresh
erase an acknowledged send.

Keep the engine internal in this phase: a public send command only lands when the next
phase can finish notes correctly. Tests may inject a finalization callback to exercise
every state without a live Gmail account.

Fault tests: crash before dispatch, timeout before Gmail accepts, accepted send with
lost response, draft gone plus unique SENT evidence, unrelated old SENT message,
ambiguous matches, draft gone with no evidence, lookup failure, failure to persist
intent, malformed state, and failure to persist acknowledgement. Restart from disk for
the cases that claim crash recovery. Assert zero extra send/create/update calls on
recovery and unresolved outcomes. Race two processes against the same note and assert at
most one dispatch. Test scope/writer refusal and that the read transport continues to
enforce its restrictions. Journals/state must contain no body sentinel.

### 4.5 send-finalize — command, confirmation, and sent cards

Expose the complete command, not a partially functional placeholder:

```text
bob gmail send NOTE [-b DIR] [-c|--confirm DIGEST] [-d|--dry-run] [-f human|json]
```

Wire draft preparation, exact preview, TTY `[y/N]`, non-TTY digest requirement, stale
confirmation refusal, and recovery before mutation. `-d` always wins over confirmation
and never sends. Prompt EOF or anything except affirmative confirmation keeps the draft.
Keep JSON stdout clean; prompts/progress use stderr. Return `schema_version: 1`, `ok`,
typed status and identities on success. Errors retain the established
`{kind,message,hint?}` contract without body-bearing preview data.

Factor the existing save pipeline only as far as needed to share policy, preparation,
inert card rendering, staged writes, imported-image handling, exact-path commits, and
journaling. A sent thread must be refreshed through this shared path. Do not spawn
`bob gmail save` and then perform unrelated blind note edits.

For a reply, add the verified sent card and remove only the exact submitted Reply
section and its draft marker. For a standalone compose note, promote at the original
path: add thread identity/managed metadata, replace submitted content with the
conversation, remove compose-only status/recipient/draft fields, and preserve parent,
unknown frontmatter, Notes, and other authored sections. Do not allocate a second note
for the newly known thread. Detect existing conflicting thread identities and refuse the
local promotion while retaining the acknowledged send.

Fix the owned-message callout to `[!mail-sent]` with existing fold behavior; verify
inbound cards remain `[!mail]`. Do not rewrite historical cards unnecessarily. If source
text changed during send, never delete those edits: retain the acknowledgement and
report a clear `sent_needs_review`/finalization-pending result. A later invocation may
retry finalization but never transmit the same operation again. Similarly retain state
when a fetch, policy refusal, staged write, or commit blocks local finalization.

Append `gmail.sent` only with thread ID, sent message ID, recipient count, and subject
hash. Make its replay idempotent by the operation/message identity. Persist completion
so a crash between local writes, commit, journal append, and result output can be
reconciled without a duplicate send or deleting a newer Reply.

Tests exercise the actual CLI and fake process: new → draft → preview → confirm →
in-place promotion; reply/all → preview → confirm → reply removal; exact JSON; TTY
decline/accept; non-TTY no confirmation; mismatched digest; changed attachments,
HTML-only edits, Gmail signature and remote-draft edits; fresh replies on completed
threads; repeated send; source edits in flight; refresh/promotion/journal failures;
three recovery outcomes and no resends. Assert correct sent cards, unrelated text and
dirty vault paths preserved, no Notes sent, and no forbidden direct-send invocation.

### 4.6 compose-docs — guide and consumer acceptance

Complete `docs/gmail.md` Writing and sending, public JSON examples, syntax refusal
reference, signature/attachment behavior, dry-run semantics, scope hints, recovery
outcomes, and the acknowledged-send/local-finalization distinction. Remove obsolete
compose placeholders only; other unfinished Gmail phases retain their accurate status.

Add `docs/guides/send-email-from-obsidian.md`, with appropriate guide index and mkdocs
navigation. Explain the Bob Mail flow as the bob-cli-hg.13 consumer, using CLI steps
that are available now and identifying UI work accurately if not yet landed. Its Try it
section has Bryan choose a test address, grant compose on the Mac, preview, and
explicitly confirm. Agents do not execute these live steps. Keep the undo countdown in
the plugin, never a CLI delay setting.

Publish synthetic fixtures under an appropriate `tests/fixtures/` location for the
schema-1 preview and errors. Document the exact fields the plugin renders, especially
text body, To/Cc/Bcc, signature, attachments, flattened links, and digest mismatch.
Check fixtures against CLI-produced JSON in focused tests so they cannot drift from the
implementation. No plugin repository edits are required.

Audit each parent section 5.12 acceptance item against a named test and record the
evidence on this child phase. Run the integration gates below. Record out-of-scope
discoveries only as `PROPOSED FOLLOW-UP:` notes on the worker's assigned bead.

## 5. Verification and landing

Each phase runs focused tests for its changed behavior and the required `just check`.
CLI/docs phases also run `just docs-gen` and `just docs-check`; follow repository
instructions for long checks through `/sase_monitor`. Preserve gcal and existing Gmail
read/save/list behavior. Fix formatting before launching the full gate, and do not
repeat a passing unchanged gate unnecessarily.

The child epic's land agent verifies the complete synthetic acceptance matrix:

| Parent requirement                                                  | Primary owner                 |
| ------------------------------------------------------------------- | ----------------------------- |
| new/reply/reply-all and precise source selection                    | compose-notes                 |
| every outgoing syntax refusal and flattened links                   | compose-render                |
| draft identity, threading, signature, scope, private body transport | gmail-drafts                  |
| stable exact preview and changed-input digest invalidation          | gmail-drafts + send-finalize  |
| TTY and non-TTY confirmation, preview sends nothing                 | send-finalize                 |
| all three send recovery outcomes and no automatic resend            | send-recovery + send-finalize |
| original-path promotion, Reply removal, and sent callout            | send-finalize                 |
| no direct-send command and no body-bearing journal/error            | all remote-operation phases   |
| useful docs and Bob Mail JSON fixtures                              | compose-docs                  |

Before any worker closes its assigned phase, run `sase bead epic-symbols <that-id>` and
resolve each temporary symbol or re-key it to a still-open consumer. Narrow intermediate
allowances must be gone by final landing. The planning baseline has no `--epic-symbol`
entries for bob-cli-hg.12, but recheck before final closure.

After all child phase beads are closed, the child epic's land agent settles its own
assigned epic through the normal authorized land flow. It then has the user's specific
authorization to finish **the original assigned phase bob-cli-hg.12**: reread its
current status and notes, run `sase bead epic-symbols bob-cli-hg.12`, then
`sase bead close bob-cli-hg.12 --note "<implemented behavior and verification>"`. Child
phase workers do not close this ancestor. Do not close bob-cli-hg or any other ancestor
plan bead. Planning and proposal alone do not complete bob-cli-hg.12.

If a check reproduces identically on the clean base, record a `PROPOSED FOLLOW-UP:` note
with reproduction evidence and any existing tracking bead, then close completed phase
work as instructed by the user. Do not create task beads, silently omit gates, or leave
a completed phase open for an unrelated base failure.
