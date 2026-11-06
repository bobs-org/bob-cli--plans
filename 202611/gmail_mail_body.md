---
tier: epic
title: Gmail message model and inert body renderer
goal: Complete bob-cli-hg.3 with a pure, tested renderer that preserves readable email
  content while preventing sender text from becoming active vault syntax.
phases:
- id: mime-model
  title: Model gog thread payloads and decode MIME body candidates
  size: medium
  depends_on: []
  description: 'mime-model: add the verified gog thread envelope, typed messages and
    MIME parts, private body decoding, attachment discovery, and fetched-body inputs
    described in section 3. Include synthetic wire fixtures and decoding tests.'
- id: inert-output
  title: Make sender text inert in every Markdown and YAML context
  size: medium
  depends_on:
  - mime-model
  description: 'inert-output: implement the shared final serialization boundary, safe
    text helpers, strict link policy, and adversarial property tests in section 4.
    Preserve only renderer-owned Markdown structure and keep operational identifiers
    opaque.'
- id: html-body
  title: Render HTML bodies with quote folding and safe images
  size: medium
  depends_on:
  - inert-output
  description: 'html-body: add the HTML5 DOM walker, quote and signature extraction,
    data-table versus layout-table handling, bulletin mode, and image decisions in
    section 5, using the established inert serializer and synthetic golden tests.'
- id: body-integration
  title: Complete the pure message renderer and its acceptance corpus
  size: medium
  depends_on:
  - html-body
  description: 'body-integration: assemble the public RenderedBody interface, plain-text
    reply splitting, HTML/plain/snippet selection, and fetched-body retry behavior.
    Complete all parent acceptance fixtures, entry-point property tests, documentation,
    and downstream consumer examples described in section 6.'
proposed_by: bbugyi200.apollo.bob-cli-hg.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hg.3
create_time: 2026-11-05 13:16:19
status: done
bead_id: bob-cli-hg.3.1
---

- **PROMPT:** [prompts/202611/gmail_mail_body.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_mail_body.md)
- **PARENT:** [202611/bob_gmail.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_gmail.md)
- **BEAD:** [bob-cli-hg.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hg/bob-cli-hg.3.1.md)

# Gmail message model and inert body renderer

## 1. Scope and completion ownership

This is a child epic for **bob-cli-hg.3**, implementing `mail-body` in
`plan:202611/bob_gmail.md` sections 4.4 and 5.3. Attach the approved epic beneath that
phase. The result is a pure Rust library boundary that the outer epic's mail-note,
gmail-save, and search/peek phases can consume. It does not implement commands,
authentication, vault writes, card layout, or sending.

The original phase is in progress, has no prior implementation notes, and currently has
no `--epic-symbol` entries. The inspected bob-cli tree at `f36cc039` has no
`src/native/gmail/` module and no matching implementation in available commit history.
Re-check current code and bead notes before each phase: outer epic workers may have
added the Gmail module or transport by then. Extend their declarations without replacing
their work.

Four sequential medium phases are appropriate because the wire model, context-aware
inert serializer, HTML interpreter, and complete MIME selection pipeline each have
distinct failure modes and acceptance evidence. Sequencing also avoids concurrent edits
to Cargo dependencies, Gmail module declarations, and the shared output types. Each
phase implements directly; none is a placeholder for another planning turn.

The outer epic's decisions are final: full bodies by default; links labelled with
subject and correspondent; no new Gmail decision records or Mail Note glossary entry.
Its Inbox File memory authorization belongs to gmail-pull-attach. **This child epic
edits no memory.** Never change the package version or CHANGELOG.md.

Child phase workers close only their own assigned bead. The child epic's land agent
verifies the combined work and supplies completion for bob-cli-hg.3; only after all
child work is closed and the entire scope is verified may that land agent close
bob-cli-hg.3. No worker here closes **bob-cli-hg** or another ancestor plan.

## 2. Evidence and shared architecture

The gog source was opened with `/sase_repo` as `gh:openclaw/gogcli`, at
`330b98709934abcf158f75be829eb4c03e5bdbbe`. Reopen it with that skill if further
inspection is necessary. Relevant files within that repository:

- `internal/cmd/gmail_thread.go`: JSON output is
  `{thread: <Gmail Thread>, downloaded: ...}`. It requests `format=full`; `--full`
  controls human display truncation, not a different JSON body representation. Normal
  JSON preserves the original recursive payload. Sanitized output is a different
  contract.
- `internal/cmd/gmail_thread_run_test.go`: synthetic envelope, nested parts, and
  attachment examples. Indexed attachment mode removes opaque attachment IDs, so Bob's
  model must consume ordinary output, without that mode.
- `internal/cmd/gmail_payload_representation_test.go`: `body.data` and `body.size` are
  preserved even when their lengths disagree. Size is metadata, not a body integrity
  check or a reliable empty-body test.
- `internal/gmailcontent/content.go`: display decoding is separate from raw JSON. Do not
  assume gog has decoded payload bytes for Bob.

The official
[message resource](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages)
documents recursive payloads and string-valued `internalDate` and `historyId`. The
[thread resource](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.threads)
documents its message list. The
[part-body resource](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages.attachments)
documents base64url data and the separate-fetch case. Use synthetic fixtures based on
these shapes; never fetch real mail or access the live Bob vault for testing.

Place code under `src/native/gmail/`, with `model.rs` and a `body/` module containing
`mod.rs`, `mime.rs`, `inert.rs`, `html.rs`, `plain.rs`, and tests as needed. Exact
private file organization may change. Keep the consumer-facing model and renderer names
stable once published. Add `base64`, `encoding_rs`, and an HTML5 DOM parser such as
`scraper`; add a property-testing dev dependency if needed. Update Cargo.lock normally
and preserve the package version.

Useful local precedents are `task_text_escape.rs`,
`gcal/description/{html,plain,bullets}.rs`, and `gcal/render.rs`. Do not reuse the
Calendar pipeline wholesale: it extracts directives, caps descriptions, permits `tel:`
links, and has different image semantics. Keep Calendar output unchanged. Its escaping
is also insufficient as the complete Gmail boundary: for example,
`task_dependencies::raw_wikilink_spans` searches literal delimiters, so visual Markdown
escaping alone does not prove that Bob will ignore a sender's wikilink.

### Public boundary

Use crate-visible APIs, not a new externally supported JSON API:

- `GmailThread`, `GmailMessage`, `MimePart`, `MimeBody`, and `Header` model the wire
  data. An envelope parser gives transport callers a typed thread.
- `render_message(&GmailMessage, &FetchedBodies) -> Result<RenderedBody, BodyError>` is
  pure. `FetchedBodies` is keyed by message ID and attachment ID and contains
  already-fetched bytes. No network, filesystem, capture parser, subprocess, clock, or
  global configuration access occurs in rendering.
- `RenderedBody` has `markdown`, `quoted: Option<String>`, `signature: Option<String>`,
  `source: Html|Plain|Snippet`, `bulletin`, `inline_images`, `attachments`, and
  `needs_fetch` as required by the parent. All three prose fields are already inert.
  They contain body content, not the outer mail card or the nested quote/signature
  callout wrappers; mail-note owns those wrappers.
- Inline image descriptors preserve CID, MIME, size, filename, and an explicit
  bytes-or-attachment-ID source. Attachment descriptors preserve stable document order,
  filename, MIME, size, and provider attachment ID. Filename display text uses inert
  helpers; raw names remain untrusted path inputs for the save phase.
- Operational IDs, CIDs, MIME labels, and bytes remain typed data. Never alter an
  attachment ID with Markdown escaping. Never interpolate raw metadata into output.
  Avoid body-revealing `Debug`, `Display`, serde error text, or logging; diagnostics
  contain typed reasons and structural locations, not input values or snippets.

`needs_fetch` identifies unresolved body parts, not ordinary user attachments. The
caller already knows the message ID. Re-rendering the same message with fetched bytes
resolves those entries without mutating the original model. A missing part must never
masquerade as a successfully saved empty body.

## 3. Phase mime-model

1. Add/extend Gmail module declarations in `src/native.rs` and
   `src/native/gmail/mod.rs`. Scope any temporary unused-code allowance to the
   unconsumed internal boundary, with an explicit consumer reference. Do not weaken
   repository-wide lint settings.
2. Deserialize the verified `thread` envelope; tolerate unrelated envelope fields.
   Preserve thread ID/history ID and each message's ID/thread ID, labels, internal date,
   snippet, headers, and recursive parts. Keep headers ordered, with case-insensitive
   lookup and the parent-required From/To/Cc/Date/Subject/
   Message-ID/List-Id/List-Unsubscribe/Precedence accessors. Retain Content-Type,
   Content-Disposition, and Content-ID for MIME interpretation.
3. Distinguish omitted optional values, truly empty leaves, container nodes, and
   attachment-backed leaves. Missing IDs or invalid structural values produce body-free
   typed errors. Support the documented decimal-string timestamp and history
   representations without floating-point conversion. Unknown fields must not break
   parsing.
4. Walk multipart/alternative, mixed, and related trees deterministically. Collect
   HTML/plain candidates, inline resources, and ordinary attachments separately. A
   named/disposition attachment such as `notes.txt` is not the message's main plain
   body. Attached `message/rfc822` content is not promoted to main text. Respect
   `multipart/related` root selection when specified.
5. Decode padded and unpadded base64url data, then the declared Content-Type charset
   with `encoding_rs` (UTF-8, Windows-1252, and representative ISO-8859 variants). Do
   not run an unconditional second MIME transfer decode on Gmail's part data merely
   because a historical Content-Transfer-Encoding header remains; preserve literal `=XX`
   text and prove this with a fixture. Handle malformed base64, unknown charsets, and
   invalid byte sequences deterministically with documented fallback/error behavior. A
   size discrepancy alone is not an error.
6. Establish the `FetchedBodies`, source enums, descriptor types, and `RenderedBody`
   definitions above. Decode fetched bytes through the same charset path, without
   base64-decoding bytes a second time. Keep limits on traversal depth/node count
   explicit and test them; exceedance returns a typed failure, never silent truncation
   or a panic.

Acceptance: synthetic JSON fixtures cover full envelope, empty thread, missing optional
fields, repeated/case-varied headers, nested alternative/related/mixed, large decimal
IDs, padded/unpadded data, an attachment-backed body before/after hydration, a real
empty body, charset cases, malformed encoding, and size mismatch. Tests demonstrate that
no body or encoded data leaks through errors or debug output. Run focused model/MIME
tests and `just check`.

## 4. Phase inert-output

Implement a small structured block/inline representation and its final serializer, so
renderer-owned paragraphs, lists, emphasis, tables, and links can survive while sender
content cannot introduce structure. It may remain private to `body/`. The last step for
main, quote, and signature content is this inert serializer; never sanitize a string and
then append unchecked sender bytes.

Expose documented helpers for single-line headers/snippets, multiline prose, link
labels/aliases, and values destined for YAML. YAML serialization is still performed by
serde_yaml in the note layer: the helper neutralizes sender syntax, not YAML quoting.
Test the actual serialized document and decoded value. Filename display uses the
single-line helper; filesystem slug/path policy remains with mail-note/save. Helpers
must state their input context and single-application contract, avoiding accidental
double escaping.

Implement every parent section 4.4 outcome:

- Neutralize `[[`, `]]`, embeds, trailing block IDs, both bracketed and bare inline
  fields, tags, checkboxes in any list/callout depth, `%%`, and math delimiters. Encode
  or break delimiter pairs where raw Bob scanners otherwise still detect them. Sender
  backslashes, entities, control characters, and Unicode spacing must not undo the
  protection. Preserve readable visible text.
- Raw HTML/comment and Templater openers cannot survive in any output context, including
  serialized YAML. Sender text cannot forge Gmail managed markers.
- Fenced code is emitted only with `text` info, or rendered as literal text. If fences
  are emitted, delimiter length accounts for runs inside the content. Inline queries
  beginning with trimmed `=` or `$=` are broken; sender code cannot become Dataview,
  Tasks, Bases, Mermaid, or Templater execution.
- Allow only parsed absolute http/https URLs and mailto links. Reject all other schemes,
  protocol-relative/relative targets, and control-obfuscated schemes. Encode destination
  delimiters and neutralize label text separately. Invalid links retain readable text.
  Account for dangerous syntax inside an otherwise allowed URL without breaking ordinary
  fragments and query parameters.
- Only the writer may introduce a verified local image embed. Sender-provided Markdown
  images, raw HTML images, and fake local embeds cannot bypass this rule.

Tests must use independent observations, not merely compare a function against itself.
For body output and each header/YAML/alias helper, combine random Unicode with hostile
grammar fragments. Use Bob's actual scanners where relevant:
`task_status_hooks::parse_tasks`, `task_fields::all_inline_fields`,
`task_dependencies::raw_wikilink_spans`, and `collect_done::block_ids_in_markdown`. Also
assert no executable fences, raw Templater/comment opener, unsafe Markdown link
destination, or remote image embed. Test within a simulated card, YAML document,
heading, table cell, and alias. Trusted wrapper syntax is excluded from sender-content
assertions explicitly. Run focused inert/property tests and `just check`.

## 5. Phase html-body

Use a real HTML5 parser and an owned walker; do not build a sanitizer with regexes or
hand HTML tokenization. Convert a decoded HTML candidate into the structured
representation, then serialize only through section 4.

1. Drop head/script/style/forms/frames and their executable or hidden contents,
   comments, event attributes, and hidden preheaders. Handle case variation, `hidden`,
   inline display/visibility rules, zero dimensions, and malformed HTML. Treat CSS as
   input to a conservative visibility check, never a resource to load.
2. Preserve readable paragraph/line boundaries, headings, lists, emphasis, and code.
   Unwrap presentation/layout tables in reading order. Keep semantic data tables (for
   example th/thead/caption evidence) as Markdown tables; use a readable linear fallback
   for unsupported spans or nesting without dropping cell text. Escape pipes/newlines
   within cells. Record deterministic classification fixtures.
3. Extract `.gmail_quote`, `blockquote[type=cite]`, Outlook `divRplyFwdMsg`, and strong
   From/Sent/To/Subject reply dividers into a quote segment. Recognize `On ... wrote:`
   boundaries without treating an ordinary sentence mentioning `From:` as history.
   Extract `.gmail_signature` independently. Preserve any new reply text after a quoted
   block. Quote and signature content receives the same sanitizer and inert
   serialization as main text.
4. A remote image yields an ordinary safe link labelled `🖼 <alt or host>`, never image
   syntax. Drop tracking images at most 2 by 2 pixels, zero-size images, and hidden
   images. Unsupported image targets become safe text. Handle `picture`, `srcset`,
   SVG/object, and CSS URLs conservatively without loading content.
5. Resolve CID references only against this message's typed MIME resources. Missing or
   ambiguous CIDs remain readable placeholders. Return only referenced, non-tracking
   inline resource descriptors. Their filenames are not trusted paths. Produce no embed
   until gmail-save has copied a validated local file. Document the handoff: descriptors
   can be displayed/imported in stable occurrence order; do not use a sender-forgeable
   string replacement marker. Save owns the 2 MiB cap, hashes, repeated-logo
   suppression, local paths, and origin journal writes.
6. In bulletin mode emit only text and safe links: flatten tables and suppress all
   images and inline import candidates. Keep meaningful article text and links.

Acceptance goldens: HTML-only message, real receipt table, layout-table message,
newsletter, ordinary/tracking/hidden images, CID image in data and attachment forms,
Gmail quote/signature, Outlook reply, malformed HTML, and hostile links/attributes.
Render twice to verify deterministic results. Run focused HTML tests plus the
inert/property suite and `just check`.

## 6. Phase body-integration

1. Implement plain-text segmentation with quoted `>` runs, `On ... wrote:` blocks,
   strong Outlook dividers, and the exact `-- ` signature delimiter. Plain email is
   untrusted text, not raw Markdown passed through. Preserve useful paragraph and list
   shape through the structured serializer. Keep inline replies outside quoted runs in
   main text; do not delete ambiguous content.
2. Assemble `render_message`. Prefer usable HTML, falling back to plain when HTML is
   absent or merely wraps the same text. Compare normalized visible text and meaningful
   structure, not raw tag stripping: HTML with meaningful links, tables, or images is
   not a trivial wrapper. Do not concatenate both alternatives. If sanitization leaves
   no useful HTML, use plain, then the inert snippet. Missing fetchable body content
   retains `needs_fetch`, even when a provisional snippet/plain fallback is shown.
   Resolve it on a second pure call with bytes.
3. Determine bulletin mode by header presence (List-Unsubscribe or List-Id),
   `Precedence: bulk`, or exact Gmail category labels `CATEGORY_PROMOTIONS`,
   `CATEGORY_UPDATES`, `CATEGORY_FORUMS`. Do not lowercase opaque label IDs.
4. Exclude `DRAFT` messages from the importable message iterator. A direct attempt to
   render a draft returns a typed refusal so callers cannot accidentally import it. Keep
   original IDs and stable MIME-order attachment descriptors for note layout. Define
   descriptor indices as 1-based within a message; the note layer assigns thread-wide
   display numbers in document order.
5. Complete fixtures for every parent acceptance case: plain reply chain, HTML-only,
   receipt, newsletter, CID images, attachment-backed long body, Windows-1252, Outlook
   reply, empty body, and the complete hostile input. Include a six-message re-quoting
   chain: each message's new prose appears only in its main output, with history in
   `quoted`, ready for one folded callout. Keep golden outputs reviewable and entirely
   synthetic.
6. Exercise every public renderer entry with generated hostile strings, including
   headers, attachment display names, snippets, body data, quote/signature nodes, link
   labels/URLs, and HTML entity decoding. Cover repeated backslashes, Unicode, CRLF,
   multi-backtick code, mixed nested callouts, and hostile text at segment boundaries.
   Test a consumer-shaped note fragment so concatenation cannot reconstruct escaped
   hazards. This is in addition to the helper tests.
7. Add consumer examples as unit/integration tests showing a thread parsed from gog,
   missing body hydration and re-render, safe metadata formatting, and
   main/quoted/signature placement inside mail-note's owned callout wrappers. Document
   the API and trust boundary in module comments for the outer phases.
8. Read `docs.md` via `/sase_memory_read`; add the body-rendering behavior section to
   `docs/gmail.md` (extend gmail-shell's page if present; otherwise create only this
   scoped section for later merge). Explain HTML/plain fallback, folded
   history/signatures, bulletin rendering, image behavior, and inert sender text. Do not
   advertise unfinished commands as implemented or create new memory.

Run all focused Gmail body/model tests, `just check`, `just docs-gen`, and
`just docs-check`. Use `/sase_monitor` for commands requiring a handoff, and wait for
the start command itself to finish. No live Gmail, real messages, real vault, or browser
automation is needed.

## 7. Landing and evidence

Each phase records its tests and the APIs it makes available. If it temporarily
introduces `--epic-symbol` entries, key them to an open consuming phase or the child
epic; resolve or re-key them before closing that phase. Do not leave entries keyed to a
closed bead.

The child epic land agent audits all parent section 5.3 acceptance items, reruns the
combined checks when integration changes justify it, and verifies that mail-note and
save can consume the types without an alternative parsing/sanitization path. If those
outer modules have landed, adapt the shared boundary and verify their integration tests;
do not take over their remaining features.

Before completion of bob-cli-hg.3, run `sase bead epic-symbols bob-cli-hg.3` and resolve
every result or re-key it to a still-open consumer/outer epic. Run the same check for
the child epic and its closing phases. Record explicit evidence, then close only the
beads this child epic's landing role is authorized to close, including bob-cli-hg.3 once
its children are finished. Never close bob-cli-hg.

A check failure reproduced identically on the clean base is recorded as
`PROPOSED FOLLOW-UP:` on the responsible assigned phase with reproduction evidence and
any existing tracking ID; it does not leave completed phase work open. Child workers
create no task beads. Use `feat(gmail): ...` implementation commit messages through the
SASE finalizer workflow.
