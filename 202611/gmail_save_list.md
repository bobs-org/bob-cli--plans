---
tier: epic
title: Complete Gmail save and offline list
goal: Finish bob-cli-hg.6 with confirmed thread saving, inert local images, append-only
  refresh, exact-path vault commits, and offline saved-mail listing.
phases:
- id: resolve-policy
  title: Confirm Gmail identities and apply save refusal rules
  size: medium
  depends_on: []
  description: 'resolve-policy: implement the shared read-only URL/ID/note resolver
    and never_save policy described in sections 3 and 4.1. Confirm exact account/thread
    identity, preserve typed failures, add read-only label-name lookup, and extend
    the fake gog with faithful synthetic scenarios. Do not write the vault.'
- id: prepare-content
  title: Prepare inert messages and local inline image imports
  size: medium
  depends_on:
  - resolve-policy
  description: 'prepare-content: implement in-memory attachment-body hydration through
    gog''s narrowly allowed Discovery read, headers mode, and CID image planning with
    size caps, hash deduplication, logo suppression, safe paths, and trusted resized
    embeds. Return prepared content and binary imports without writing.'
- id: save-transaction
  title: Save and refresh mail notes under the vault lock
  size: medium
  depends_on:
  - prepare-content
  description: 'save-transaction: implement the save CLI and reusable staged save
    API, writer gate, parent-at-birth, hubs, agent warning, lock/re-read/retry, atomic
    text writes and safe image promotion, exact-path commit, and post-commit journals.
    Add save catalog coverage, docs, and adversarial transaction tests.'
- id: list-diagnostics
  title: List saved mail offline and diagnose identity health
  size: medium
  depends_on:
  - save-transaction
  description: 'list-diagnostics: implement real bare/explicit list with pagination
    and backlinks, body-free JSON, decoder and duplicate diagnostics, catalog/help/docs
    updates, and the remaining cross-layer acceptance checks. Leave remote refresh
    and link-failure footer population to the existing outer epic phases.'
proposed_by: bbugyi200.apollo.bob-cli-hg.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hg.6
create_time: 2026-11-05 23:47:33
status: done
bead_id: bob-cli-hg.6.1
---

- **PROMPT:** [prompts/202611/gmail_save_list.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_save_list.md)
- **PARENT:** [202611/bob_gmail.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_gmail.md)
- **BEAD:** [bob-cli-hg.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hg/bob-cli-hg.6.1.md)

# Complete bob-cli-hg.6: Gmail save and offline list

## 1. Objective and scope

Complete assigned phase **bob-cli-hg.6**, `gmail-save`, from `plan:202611/bob_gmail.md`
section 5.6. A user can save or refresh a chosen Gmail thread as an inert mail note,
receive its stable labelled wikilink, and browse saved notes offline. The original phase
remains open until this entire child epic is verified; its closure belongs to this child
epic's land agent, not to any child phase worker. Never close the outer epic
**bob-cli-hg**.

The required planning handoff is warranted by four separate boundaries: identity and
policy; untrusted MIME and binary data; durable vault mutation; and offline reporting.
Each phase is direct implementation work sized for one coding agent. Dependencies are
deliberately serial because each consumes the preceding API and they share Gmail
integration files. This is not a restart: the prerequisite parser, renderer, note
engine, and shell have already landed.

Final reviewer choices inherited from the outer epic:

- `default_body = full`: full bodies by default, with explicit headers mode.
- `link_label = subject_who`: default `✉️ {subject} · {who}`.
- `memory_gmail_decisions = no` and `memory_mail_note_term = no`.
- `memory_inbox_file = yes` applies exclusively to outer phase `gmail-pull-attach`; this
  child epic edits **no SASE memory**.

No additional decision or approval choice is needed. Do not add search, peek, remote
listing, link replacement, capture workers, send, pull, on-demand ordinary attachments,
plugins, or Mac changes. Their existing outer phases depend on this phase and consume
its interfaces.

## 2. Evidence and code to reuse

At planning time `sase bead read bob-cli-hg.6` reports `in_progress`, with no prior-work
notes and no children. Its three recorded dependencies are closed.
`sase bead epic-symbols bob-cli-hg.6` reports no entries. The primary checkout was
clean. Recheck these facts and newly landed work before implementation.

Landed implementation includes:

- `src/native/gmail/ids.rs`: offline `GmailRef` / `GmailVerdict`, URL scanner, token
  decoder, canonical account-qualified URLs. Reuse its parsing rather than recency or
  subject heuristics.
- `gmail/model.rs`, `gmail/body/`: typed gog thread envelope, draft filtering, inert
  strings, MIME fetch needs, `FetchedBodies`, and `render_message`. Renderer integration
  landed in `45118e1b` and `232e94bb`.
- `gmail/note/{layout,merge,parse,path,label,index,hub}.rs`, landed in `cfdb68c1`: note
  construction and metadata, ownership-preserving append-only merge, fixed birth paths,
  labels, duplicate detection, and hub templates. Preserve these existing tests and
  linter-compatible output.
- `gmail/{cli,config,gog,doctor,journal,ui}.rs` and `native/google/`: configuration,
  account selection, isolated `bob-gmail` transport, typed errors, diagnostics.
  `gmail/mod.rs::list_saved` currently returns an empty placeholder. There is no real
  `save` command or explicit `list` command yet.
- `capture::{StagedTextFile,write_staged_files,validate_target_parent}`,
  `ob::{acquire_lock_waiting,detect_git_worktree,commit_paths}`, and `gcal/sync.rs` for
  lock/replan/preimage retry and scoped commits.
- `attachments::journal::{append_origin_events,serialize_event}` and `pref_journal` for
  origin events, host labels, timestamps, and Gmail events.
- `tests/gmail_support/`: isolated HOME/config/vault/time/host and fake gog. Its current
  thread fixture is empty and its attachment response hides real gog's disk side effect;
  extend the fake rather than trusting those defaults.
- `src/native/catalog/table.rs`, `docs/gmail.md`, `docs/reference/cli/`, and
  `tests/cli/docs_reference.rs` for command exposure.

Audited upstream inspection through `/sase_repo` opened `gh:openclaw/gogcli` at
`e8428c36dd31db13eecba55fcef6f4ee75951188`; the relevant behavior also exists at tag
`v0.44.0`:

- `internal/cmd/gmail_attachment.go`: `--inline` **still calls
  downloadAttachmentFreshToPath and writes a cache file**, then returns `contentBase64`.
  The current Bob `Gog::attachment` wrapper therefore cannot satisfy the no-body-cache
  or dry-run contract when activated as written.
- `internal/cmd/api.go`:
  `api call gmail v1 gmail.users.messages.attachments.get --params JSON --scope https://www.googleapis.com/auth/gmail.readonly --no-cache`
  returns the API object on stdout. Allow exactly `api.call` and
  `api.gmail.users.messages.attachments.get` for this invocation in addition to the
  common read-only and no-send guards. Do not permit generic API writes.
- `internal/cmd/gmail_labels.go`: `gmail labels list` returns `{labels:[...]}` with
  opaque case-sensitive IDs and display names. This is necessary because
  `never_save.labels` contains names, while message payloads contain IDs.
- `internal/cmd/gmail_get.go`: message JSON is under `message`; metadata mode is
  supported. `gmail_thread.go` confirms the typed `thread` envelope and that thread
  reads do not download attachments without `--download`.

Future workers must reopen upstream through `/sase_repo` before reading it; do not copy
an absolute checkout path into implementation or fixtures. No upstream edits are
planned. Tests use only synthetic responses and fake gog.

## 3. Contracts shared by all phases

### 3.1 Identity and input scope

Accept the outer plan's CLI:

```text
bob gmail save <URL|ID|NOTE>... [-b DIR] [-B full|headers] [-d]
    [-f human|json] [-P ROUTE] [-q] [-R]
bob gmail [list] [-a] [-b DIR] [-f table|json]
```

Long names are `--bob-dir`, `--body`, `--dry-run`, `--format`, `--parent`, `--quiet`,
`--rerender`, and `--all`. Keep options and subcommands alphabetic, with useful examples
and environment help. Both bare and explicit list are the same offline operation. Do not
add `--remote`; outer hg.9 owns it.

Identity is `(account, thread_id)`. URL account slots and `authuser` are not account
selectors. Resolve account using the existing noninteractive Google account resolver for
network operations. Offline list never needs gog, OAuth, or account discovery; display
the locally indexed accounts, optionally scoped by an explicitly configured account, and
include account in JSON identities.

Input resolution supports a vault mail-note path (with optional `.md`, using existing
note resolution conventions), parsed Gmail URLs, a bare 16-hex thread ID, decoded
message IDs, and `rfc822msgid:` references from the parser. Use only in-vault,
non-symlink paths; a note must have valid mail identity. Index matches reuse the
existing path, but explicit save still fetches the confirmed current thread before
refresh. Duplicate identities are ambiguous, never silently pick one. Reject a note
belonging to another selected account.

A decoded thread ID must match the fetched thread. A message reference must match the
fetched message ID, then resolve that message's thread. RFC search must establish
exactly one thread; zero matches is unresolvable, multiple or unexhausted results are
ambiguous. No recency, subject, or date fallback. Return `gone` for 404, `other_account`
only when supported by account evidence (never infer it from a 404), `ambiguous`,
`unresolvable`, `refused`, `never_save`, and `awaiting_sign_in` as body-free typed
errors. Resolution failures exit 1 and write nothing. CLI/config/writer setup failures
exit 2; successful commands exit 0. Transient transport failures remain distinguishable
from permanent resolution failures for the later link worker.

### 3.2 Reusable pipeline and reporting

Keep resolution, preparation, locked planning, and persistence separate. Names may
follow local conventions, but expose crate-visible value types and functions for these
boundaries:

1. Resolve identity and fetch the confirmed `GmailThread` without writes.
2. Prepare rendered messages and proposed binary imports entirely in memory.
3. Given current vault contents, build text preimages/changes, chosen paths, imports,
   post-commit events, and a body-free result record.
4. A standalone save orchestrator applies and commits those changes. Later link/pull
   callers can instead combine the prepared text edits with their own edits and commit
   once; they must not need to shell out to `bob gmail`.

Avoid a separate generic framework or a new persistent index. Keep prepared body values
private to the write path: no Debug dumps, serde exposure, telemetry, or persisted body
caches. The pure note index stays frontmatter-only.

For multiple inputs, resolve and prepare the whole batch before mutation; coalesce
identical identities while preserving input-to-result association. A failing input
prevents any batch write. Distinct threads can share a single lock and scoped commit.
`save -f json` has `schema_version: 1`, `ok`, and `results`, with one record per input
including `thread_id`, `account`, `note`, `link`, `status` (`created`, `refreshed`,
`unchanged`, or `would_*`), message count, and added count. For one input also expose
top-level `note` and `link` for the planned plugin caller. Errors use
`{kind,message,hint?}`. No body, raw MIME, byte payload, or body diff appears in JSON or
error output.

Human success output follows the outer plan's short styled Saved / refreshed / unchanged
lines. Quiet suppresses success chatter while retaining failures; it must not suppress a
requested JSON result.

Dry-run may perform read-only Gmail calls but writes no notes, images, hubs, journals,
AGENTS text, persistent caches, or commits and creates no vault lock file. It reports
paths/counts and a structural diff with message prose and snippet contents redacted,
including for rerender. This preserves the outer plan's preview requirement and its
stronger invariant that bodies appear only in notes/imported files, not diagnostics.
Existing renderer output never passes through capture grammar.

### 3.3 Privacy, ownership, and boundaries

Gmail remains read-only. All requests carry the configured account and isolated
`--client bob-gmail`, `--readonly`, `--gmail-no-send`, and a narrow command allowlist.
No login prompts, token reuse from another bucket, label mutation, send, `--download`,
sanitize/wrap flags, or cache-file attachment fallback. Do not run against a real vault
or mailbox in an agent session.

Preserve the landed mail-note ownership: `parent` is user-owned after birth;
Notes/Reply/other authored text stays; old cards survive Gmail deletion; append-only
refresh and the note engine's safe fold update stay intact;
damaged/missing/duplicate/reordered markers refuse. Rerender is explicit and preserves
unavailable historical cards and text outside the managed block.

## 4. Implementation phases

### 4.1 resolve-policy

Add a resolver module and a focused policy module under `gmail/`. Reuse ids, the
frontmatter index, the account resolver, and the typed gog adapter. Separate identity
resolution from fetching/rendering full content so outer peek can reuse the identity
stage with a metadata fetch later. Verify every response's identity before exposing a
successful result. Normalize comparison of account emails consistently with account
selection; preserve Gmail IDs as opaque tokens apart from the parser's canonical hex
normalization.

Add `gmail labels list` to the read adapter and the specific allowed command only when
needed. Resolve configured label names case-insensitively to opaque IDs; match those IDs
exactly against all importable messages. Block a thread when any importable message
matches an address, an exact `@domain` suffix boundary (case-insensitive), or a blocked
label. Use a proper mailbox/header parser already used in the note layer; a display-name
substring is not a sender address. If label lookup fails, refuse rather than bypass
policy. An absent label name blocks no thread. Filter DRAFT messages consistently.

Extend fake-gog scenarios for nonempty thread envelopes, message metadata, RFC search
zero/one/many/pagination, labels, 404, auth expiry, malformed JSON, identity mismatch,
and poison sentinels in all output channels. Retain the shell tests and their existing
scenario defaults. Unit tests or a testable transport boundary exercise the resolver
without requiring an unfinished CLI.

Acceptance: every resolution branch and typed refusal; known-path reuse; duplicate and
cross-account refusal; no malformed hex success; no arbitrary filesystem read;
sender/domain/label policy; and no vault/cache/journal writes. Ensure no full body is
fetched merely to resolve a message to a thread when metadata suffices. Keep shared
Calendar outputs and tests unchanged.

### 4.2 prepare-content

Replace the dormant `Gog::attachment` implementation with the memory-only Discovery GET
described above. Build `--params` as a serialized argument, never shell code, containing
only userId `me`, messageId, and attachment ID. Parse raw `{data,size}` using strict
standard/base64url decoding already covered by the adapter tests; validate actual byte
length and return raw bytes once. Set `--no-cache` and explicit readonly scope. Keep
account/client flags. If a gog build refuses generic reads, surface a setup/capability
hint; never fall back to the attachment command that writes to disk. Update fake-gog
allowlist parsing and tests to model this exact argv and response.

For full mode, render each importable message, fetch each `needs_fetch` attachment body
at most once into `FetchedBodies`, then rerender. A remaining fetch need or transport
failure fails preparation, rather than silently saving a truncated fallback. All network
work precedes the vault lock. Headers mode uses inert snippets and the existing
not-saved marker, with attachment metadata only; it performs no body-part or image
downloads and exposes no body prose.

Plan only referenced, non-tracking CID images that the renderer supplies; ordinary
attachments remain listings. Enforce the 2 MiB cap using both metadata and received
length. Oversized/non-image candidates are skipped with a typed, body-free reason; no
remote URL is fetched. Hash bytes with sha256. Within a thread, identical bytes share a
destination; repeated hashes under 4 KiB seen in distinct messages are suppressed as
signature logos (not merely two uses in one message). Keep the 4 KiB/2 MiB boundary
behavior in tests.

Use sanitized names, safe extensions, and collision handling below
`img/mail/<thread hex>/`; deterministic reuse is by verified hash, never just a sender
filename. Carry proposed relative paths and bytes to the transaction phase, which
rechecks collisions and symlinks before installation. Reuse or extend the existing
trusted-image helper so final cards contain exactly `![[img/mail/...|400]]`. Only a
validated writer-owned imported path may mint that syntax. The current helper returns
Markdown image syntax, so do not assume it already satisfies this contract. Add embeds
in stable descriptor order inside their message cards, leaving the pure renderer inert
and keeping sender strings unable to forge placeholders or embeds.

Return body-free attachment origin descriptors suitable for
`append_origin_events(..., "gmail")`: use a deterministic Gmail identity locator
composed only of account/message/part IDs as source, plus saved path and sha256; no raw
URL token or bytes. Do not append events yet.

Acceptance: attachment-backed HTML/plain hydration, no double decoding, fetch failure,
headers zero-fetch, draft exclusion, bulletin no-images, duplicate CID names across
messages, binary-size boundaries, small repeated logo suppression, malicious names/path
traversal, duplicate hashes, and exact safe embeds. The fake gog must fail if
`gmail attachment` is invoked, and tests assert no persistent files anywhere in the
isolated HOME/state/cache.

### 4.3 save-transaction

Add save args/dispatch, the reusable save planner/orchestrator, and typed Gmail save
journal events. Check `gmail.writer` against `vault_sync::hostname_slug()` before a real
save; dry-run is permitted on any host. Resolve and validate `-P` with the existing
parent-note resolver and set its canonical link only for new notes. `-B` overrides the
config for this call. Preserve saved paths and authored parents on refresh, including
when the subject changes.

Build a byte-diff plan against the current index and parsed note, using the landed
`path::allocate`, `layout::render`, `merge::merge`, label helpers, and hub templates.
Check all occupied paths, including unrelated files, when allocating. Create `mail.md`
and `mail.base` only if absent. Add the outer plan's one-line untrusted-mail warning to
the target vault's AGENTS.md once, preserving every other byte and reflecting the
configured folder when it is not `mail`. Implement this as a runtime vault edit tested
in temporary vaults; do not edit the repository's generated AGENTS.md or a real user's
vault while developing. Let vault-sync commit this agent-warning edit, as the outer plan
specifies; it is not part of the save's mail-content git commit.

For a mutation, acquire `bob_sync.lock` with a 30-second bounded wait. Re-read the
index, target notes, hubs, and parent/path preconditions under the lock; replan from
that snapshot. Write text through `capture::write_staged_files`. On typed `NoteChanged`,
re-read/replan once and refuse a second race without overwriting edits. No network
request occurs under the lock.

Install binary imports with validated in-vault parent paths and atomic create/promote
semantics; never overwrite a different existing file. The whole plan must be validated
before mutation. Keep track of newly created files so a failed text apply can remove
only this attempt's unchanged new images; never remove preexisting images or someone's
concurrent edits. Ensure no successful note refers to a missing asset. Test this
behavior with injected write/race failures rather than timing-dependent sleeps.

Commit only changed mail notes, images, and newly created hubs with `ob::commit_paths`
when the vault is a git worktree. Use outer-plan messages `gmail: save <subject>` /
`gmail: refresh <subject>` (for a batch choose the first stable subject plus a count);
sanitize control characters in subjects. Preserve unrelated staged and unstaged work.
This is product code using the vault helper, not authorization for an agent to invoke
git commit manually.

After successful commit (or successful durable writes in a non-git vault), append
`gmail.save` in `.bob/gmail/<host>.jsonl` with the shared timestamp/host/ kind/value
envelope, account/thread ID, note path, created flag, and added count. Append origin
events with `via: gmail` for distinct newly referenced imports. No-op refresh emits
neither new journal events nor an empty commit. Follow existing origin failure handling
with explicit body-free warnings. A commit failure is reported as
files-written-but-uncommitted, with no success journal; a retry must reuse the
note/assets and finish the scoped commit without duplicated cards. Test this recovery
boundary. Leave post-commit journals for vault-sync, as established by the outer plan's
ordering.

Expose success results and safe structural previews from section 3.2. Add catalog
`gmail-save` as a preview-first write with `-d`, including input selection consistent
with similar noun commands. Update the save section of `docs/gmail.md`, generate help
references, and test catalog coverage now so this phase itself has a green gate.

Acceptance: create, refresh, repeated/aliased input deduplication, preserved paths/user
text/parent/old cards, malformed markers, rerender, parent at birth, headers override,
never_save and writer refusal, hub/warning idempotence, image persistence/origin,
byte-identical no-op, dry-run filesystem and git purity, mixed-batch refusal,
lock/race/write/commit failures, touched-path-only git commits, and redaction sentinels
absent from all output and journal bytes.

### 4.4 list-diagnostics

Replace the empty list stub with the real offline read and add an explicit `list`
command plus `-a` on both spellings. Extend frontmatter index metadata with
started/updated/message count as necessary without reading bodies into the index. Sort
by parsed updated instant descending with a stable path tie break; default to 20 rows,
`-a` shows all. Malformed metadata is reported or skipped deterministically rather than
crashing; duplicate identities stay visible as diagnostics, never selected as canonical.
Count backlinks using the repository's Markdown-aware link scanner/resolver where
available; count distinct source notes, including aliases and block/heading targets,
excluding self-links and fenced/inline code. Scan once per listing, not once per
returned row. Document this count's meaning.

Table columns: updated date, subject, who, messages, backlinks. JSON includes the same
fields, identity, path and label/link, schema_version and ok. List never invokes gog,
creates a directory, changes a file, or needs the writer host. Add a footer integration
seam; the outer `gmail-link` phase hg.7 owns waiting/gone counts and failure-journal
folding, so do not fabricate counts. Until populated, use an empty/absent footer rather
than claiming zero links. Remote message comparisons remain owned by hg.9.

Add doctor rows for decoder health (total Gmail URLs, resolvable count and share,
including a sensible zero-URL case) and duplicate `(account,thread)` identities/paths.
Reuse the existing URL scanner's code/frontmatter exclusions. Make these local checks
run even if gog or sign-in checks fail; they must not fetch threads or write repair
state. Return counts and body-free reasons, not raw messages or untrusted diagnostic
text. Keep existing doctor row and exit behavior compatible while adding these checks.

Add `gmail-list` as a catalog read, retain `gmail-doctor`, and make CLI coverage tests
pass. Update `docs/gmail.md` to remove placeholder claims only for the functionality now
present, document limits/JSON/flags/errors/image policy, and add a save/list Try it
checklist for Bryan. Regenerate reference pages with `just docs-gen` and run
`just docs-check`.

Acceptance: bare/explicit equivalence; 20/all/order behavior; custom folder;
multiple-account identities; empty/malformed/duplicate notes; backlinks; no configured
account or installed gog; no network or filesystem mutation; doctor decode
success/refusal/unknown-token counts and duplicates; body-free JSON, logs, errors and
journals; representative save-to-list end-to-end flow.

## 5. Verification and land-agent completion

Each phase runs meaningful focused unit/integration tests, `just fix` as appropriate,
then `just check`. A CLI/doc change also runs `just docs-gen` and `just docs-check`. Use
`/sase_monitor` for long gates and follow its staging/finalization rules. Keep all tests
synthetic and isolated. Assert sentinel absence not only in stdout/stderr/JSON but in
Gmail and attachment journals and isolated cache/state directories. Do not duplicate the
already landed pure renderer corpus; test the newly connected trust boundaries.

The land agent must review the combined diff against outer plan sections 3, 4.1-4.8, and
5.6, verify every acceptance item above, and confirm downstream callers have the
reusable resolver/save boundaries and single-input note/link fields. Remove only
allowances now made obsolete by actual consumers; do not delete APIs still owned by
later outer phases. Run final `just check` and `just docs-check` on the integrated tree,
regenerating docs if stale.

Before closing each assigned child phase, run `sase bead epic-symbols` for that phase
and resolve or re-key leftovers to a still-open owner. Workers close only their own
assigned phase. Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` on the
original phase `bob-cli-hg.6` (or their assigned phase with a pointer on hg.6); do not
create task beads. For an unrelated gate failure that reproduces identically on the
clean base, record the reproduction and any existing tracking bead, then complete the
phase as directed by the original task rather than leaving it stranded.

Once all child work and this child epic's normal landing are settled, the child epic's
**land agent** runs `sase bead epic-symbols bob-cli-hg.6`, clears any owned leftovers,
and closes **bob-cli-hg.6** with
`sase bead close bob-cli-hg.6 --note "<implemented contracts and verification>"`. The
user explicitly requested a child epic whose land agent completes this phase when work
is split. Do not close bob-cli-hg or any other ancestor. All turns finish through
`/sase_final`; keep hg.6 open during planning and intermediate work. Never hand-edit
bead status.
