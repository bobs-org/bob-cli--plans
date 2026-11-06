---
tier: tale
title: Repair the Gmail renderer and finish bob-cli-hg.3.1 landing
goal:
  Make sender output inert and complete, integrate typed transport, and close the
  renderer epic and parent phase normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-hg.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-hg.3.1
status: done
---

- **PARENT:**
  [202611/gmail_mail_body.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_mail_body.md)
- **BEAD:**
  [bob-cli-hg.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hg/bob-cli-hg.3.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-hg.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-hg.3.1.land.md)
- **COMMITS:**
  - [6e63268](https://github.com/bobs-org/bob-cli--plans/commit/6e63268021e9c4abaef71269304926ff313cd40e)
    — docs(plans): mark gmail_mail_body done

# Repair the Gmail renderer and finish bob-cli-hg.3.1 landing

Complete the remaining correctness and integration work found while landing
`bob-cli-hg.3.1`, then close that epic and its parent phase `bob-cli-hg.3` in this
implementation turn. This is one bounded Rust implementation task, not another
multi-phase epic. The original plan is `plan:202611/gmail_mail_body.md`; the outer
contract is `plan:202611/bob_gmail.md`, sections 4.4 and 5.3.

## Scope and evidence

The lander audited all four closed phases and every note, and reviewed their actual
source and commits: `df9178df`, `1cbe9703`, `540f3e6f`, `45118e1b`. The implementation
has the required typed model, pure renderer, charset/base64 support, draft exclusion,
bulletin detection, CID descriptors, and synthetic acceptance corpus. The review found
holes in those implementations, not missing new product features. Preserve the public
`render_message(&GmailMessage, &FetchedBodies)` and `RenderedBody` contract.

The reviewed base is `6ea08f74`; a fresh fetch found no additional origin/master
commits. Review intervening commits again before editing. Since the first child commit,
`fe120a1e` extracted shared Google plumbing and `88907074` added the Gmail shell and
transport. There are still no mail-note/save/search consumers in the reviewed checkout.
If consumers have landed by implementation time, adapt them to the existing shared
boundary and run their tests; do not create a parallel parser or sanitizer.

Audited reproduction source and results are preserved as
`file:explicit:6196bea7b81628e40c86c7a4`; read it with `sase artifact read` if useful.
Eight of nine added synthetic probes failed, while the existing 77 Gmail tests passed.
The probes were removed from production source after recording the evidence. The current
formatter check passes. No full workspace check was claimed for this planning turn; the
implementation must run `just check` after the repairs.

All accepted parent decisions remain binding: full bodies by default, subject plus
correspondent link labels, no Gmail decision records or Mail Note glossary entry. This
tale changes no memory, package version, or changelog. Use synthetic mail and fake gog;
no live Gmail, live vault, network access from rendering, or browser automation.

## 1. Make serialization safe across node boundaries and URLs

Fix `src/native/gmail/body/inert.rs` and its HTML integration. Per-string escaping
currently does not guarantee that the assembled document is inert:

- `<p>!<a href='https://example.test/tracker.png'>load</a></p>` serializes the sender's
  `!` immediately before a renderer-owned link, creating a remote Markdown image.
- Test `<p><span>key:</span><span>: value</span></p>` and similar split delimiters
  against a bare-Dataview-field observer as well as Bob's bracketed-field scanner; the
  latter alone passed the lander's probe and does not establish the whole inertness
  contract. Also cover split percent-comment delimiters, math delimiters, code/query
  boundaries, backslashes, and node boundaries introduced by dropped hidden elements.
  Coalesce adjacent literal text where useful, but enforce safety at every composition
  edge, including edges against trusted link/code markup.
- `SafeUrl::parse("https://example.test/[[forged]]")` permits raw wiki delimiters in the
  destination. A valid scheme is insufficient: Bob's raw scanner still sees them.
  Serialize or encode dangerous destination bytes contextually, or refuse a destination
  whose semantics cannot be preserved safely. Keep ordinary query parameters, fragments,
  mailto addresses, and valid IPv6 URLs usable. Never Markdown-escape opaque operational
  IDs or mutate transport inputs.

Use the existing structured Block/Inline boundary; do not sanitize a completed document
by stripping all renderer-owned structure. Preserve visible literal punctuation and
trusted links/emphasis/lists/tables. Add public-entry and helper regression tests using
Bob's actual task, inline-field, wikilink, and block-ID scanners. Exercise the resulting
main, quote, signature, YAML/header/alias, table, and callout contexts. Tests must
inspect the assembled result, including whether a Markdown image can form, rather than
just assert that each input fragment was escaped. Add generated tests that vary DOM
partitioning as well as the hostile text itself.

## 2. Keep attachment subtrees separate from message bodies

In `body/mime.rs`, `collect_candidates` descends into `parts` before testing
`is_attachment`. Consequently a `message/rfc822` or named/disposition multipart
attachment can donate its descendants as main-body candidates. A mixed message with
plain `Outer text`, followed by `forward.eml` whose child is plain `Attached history`,
currently selects the attachment text. The existing RFC822 test covers only a leaf.

Stop candidate traversal at every attachment boundary. Keep the attachment descriptor,
but prevent its descendants from polluting main-body candidates, body fetch needs, or
the containing message's CID resource namespace. Preserve ordinary alternative/mixed/
related traversal, related `start` selection, stable attachment order, and same-message
CID behavior. Add nested RFC822 and named/disposition multipart fixtures, including an
attachment with a CID matching an outer resource and an unresolved nested text body.

## 3. Preserve usable bodies and folded segments during selection

In `body/render.rs`, selecting only `.last()` loses a readable earlier candidate when
the last candidate is empty or sanitizes to nothing. Iterate in a documented stable
preference order until a usable candidate is found. Keep unresolved fetch requirements
even with provisional fallback; do not concatenate alternative bodies.

`html_is_useful` currently rejects all HTML whose main field is empty, dropping messages
containing only a quoted block or signature. Treat readable quote/signature content as
usable. Comparing only alphanumeric skeletons of the main output also treats distinct
punctuation, word boundaries, or different history as equivalent. Compare normalized
visible content across all segments and track meaningful HTML structure explicitly
enough to preserve links, tables, and referenced images. Do not infer semantic table
structure solely from a literal pipe character in serialized text.

Add cases for quote-only and signature-only HTML, a useful candidate followed by an
empty/hidden candidate, equal main text with different folded content, punctuation/
whitespace differences, genuinely equivalent wrappers, and HTML with meaningful links or
images. Preserve true empty-body behavior, typed errors, hydration, and draft refusal.

## 4. Preserve content and segmentation inside HTML containers

In `body/html.rs`, `scan_table_children` reduces cells through `inlines_to_text`, which
discards every link destination. This breaks layout-table email and bulletin articles:
`<table><tr><td><a href='https://example.test/article'>Read article</a></td></tr></table>`
loses its only article URL. Keep structured inline content through layout flattening,
semantic table rendering, and span/nested-table fallback, with final escaping owned by
the inert serializer. Preserve safe links, cell boundaries, and all readable cells;
avoid truncating extra cells to the first header row's width. Bulletin output remains
text and safe links, with no image embeds/import candidates.

Preserve quote/signature segmentation through layout tables and lists as well. Current
`append_nodes_as_blocks` flattens those segment nodes back into main text. Test a
table-wrapped Gmail quote/signature and new reply after it. Detect Outlook
`id='divRplyFwdMsg'` explicitly: current `is_quote_divider` checks a class instead, and
its existing test passes only because the element also has two bold header labels. Keep
strong From/Sent/To/Subject recognition and avoid folding ordinary prose that merely
mentions From. Add a divider-ID fixture without bold header labels, plus existing
Gmail/Outlook and hidden/tracking/CID regressions.

## 5. Keep inline plain-text replies in main content

In `body/plain.rs`, `On ... wrote:` currently switches permanently into a quote tail as
soon as any later `>` line exists. Thus
`On Tue, Pat wrote:\n> old question\nNew inline answer` puts the answer into folded
history. Separate quoted-run handling from unprefixed forwarded-message tails so new
unquoted replies remain main text. Preserve attribution/history and do not delete
ambiguous content. Implement the original plan's exact `-- ` signature delimiter; a bare
`--` or unrelated dash rule must not silently move ordinary prose into a signature.
Cover strong Outlook header blocks, mixed inline replies, signatures before history,
CRLF, and the six-message chain.

## 6. Integrate the transport that landed during this epic

`src/native/gmail/gog.rs::Gog::thread_get` currently parses into `serde_json::Value` and
strips the outer `thread` field, but `GmailThread::from_gog_json` requires that
envelope. Route the transport's original response bytes through the typed parser and
return a typed thread, or introduce one explicit typed adapter that every thread
consumer uses. Errors must remain body-free; do not include serde input values or raw
responses. Add a fake-gog test spanning transport, envelope parse, and rendering,
including missing-body hydration and malformed-response redaction. Preserve shared
Google authentication/client isolation and the read-only/no-send command flags.

The same transport introduced handwritten base64 decoding despite the dependency added
by the renderer. Replace that duplicate with strict decoding using the existing crate
while retaining accepted standard/base64url padded and unpadded attachment forms and
deterministic body-free failures. Fetched bytes must reach `FetchedBodies` without being
base64-decoded again. Test malformed padding/trailing data and the successful attachment
round trip. Refresh stale consumer comments/allowance references now that the four child
phases are closed; remaining unused boundary allowances must identify still-open outer
consumers rather than pretend a closed child is pending.

## 7. Verify and finish the landing in this same turn

Run focused model/body/property and fake-transport tests, followed by `just check`. Do
not run `just check-full`. Use `/sase_monitor` if a required command needs a handoff;
the continuation must complete this tale's landing. Read `docs.md` through
`/sase_memory_read` before touching docs. Update `docs/gmail.md` only if fixes require
clarifying its behavior, and run required generation/docs checks for such changes.
Record actual results; do not assert a full check passed when an unrelated failure
stopped it. Existing unrelated failures have already been triaged by the lander; verify
their current status without creating duplicate tasks or expanding renderer scope.

Re-read `bob-cli-hg.3.1` and its four children, every note, original plan, and commits
since `6ea08f74`. Confirm every repair above and the original acceptance items. Copy the
lander's follow-up disposition note into the final verification/close note; it records
all six child proposals and their owner/decline outcomes. New issues caused by this
renderer remain in scope. Use `/sase_new_task` for genuinely new unrelated work.

Finish with these concrete operations, without a dependency on this turn's future commit
SHA, push, or CI result:

1. Run `sase bead epic-symbols bob-cli-hg.3.1`. Resolve each listed symbol by wiring,
   privatizing, a justified non-test pragma, or deletion, or re-key its Justfile entry
   only to a still-open consumer bead. No stale exemption may remain. The lander's audit
   found none. If this tale has a materialized child bead that blocks the epic, complete
   that bead normally once its implementation and checks are finished before closing the
   epic; do not force through an open descendant.
2. Run
   `sase bead close bob-cli-hg.3.1 --note "<actual original-scope, repair, integration, verification, and follow-up outcomes>"`.
   If rejected, finish the named work or symbol cleanup and retry. Never use `--force`
   to make successful nested landing advance. Cancellation/supersession requires a
   deliberate supported outcome, explicit reason and matching resolution; it is not this
   tale's success path.
3. Run `just symvision` when available (the reviewed Justfile has no such recipe). Open
   the plans repository with `/sase_repo`, using only its returned path, and set
   `status: done` in the frontmatter of `202611/gmail_mail_body.md`, the epic PLAN file
   shown by `sase bead read`. Read it through `sase artifact read` for audited context.
4. Run `sase bead read bob-cli-hg.3.1 -r "Need the parent link"`. Its parent is phase
   `bob-cli-hg.3`. Re-read that phase, confirm all mail-body requirements and its child
   are now complete, and run `sase bead epic-symbols bob-cli-hg.3`; resolve or re-key
   every entry to an open outer consumer. Close only that phase normally with
   `sase bead close bob-cli-hg.3 --note "<verified typed model, inert renderer, transport integration, acceptance evidence, and child completion>"`.
   Leave outer epic `bob-cli-hg` and its plan status for its already-waiting land agent.
5. Finish through `/sase_final` with commit decisions for all changed repositories,
   including the plans sidecar, and report completed closes and any material check
   limitations. A code commit becoming visible is not a prerequisite for these closes.
