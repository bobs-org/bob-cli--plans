---
tier: tale
title: URL listen marker for Google Keep and Bob Mac Capture
goal:
  Create new captured references with companion audio when a URL carries the chosen
  listen marker.
size: medium
decisions:
  listen_marker:
    ask: Which standalone marker should request listening immediately after a URL?
    choices:
      at:
        Use @ as requested in the prose; URL @ means listen, while @route still selects
        a parent.
      percent:
        Use % from the example; URL % means listen, overriding clipboard syntax for
        eligible refs.
    default: at
    why:
      The request explicitly names @; the % example conflicts with the existing
      clipboard shortcut.
    answer: at
proposed_by: bbugyi200.apollo.6l
decided_by: auto
create_time: 2026-10-10 16:07:00
status: wip
---

# Request companion audio when capturing a reference URL

## Outcome and scope

An eligible Google Keep note or Bob Mac Capture draft containing
`https://example.blog.com/somepost @` requests the same narration and companion-MP3
creation as `bob ref create <URL> -P <resolved-parent> -L`. Mac capture still queues a
durable background reference job and returns promptly; Keep still imports inline during
`bob gkeep pull`. The existing `bob ref scan` step creates the reference note and pairs
its PDF/audio. A bare URL without the marker keeps its current behavior.

Use a **tale, medium**: this is substantial but bounded work for one coding agent, with
one shared Rust import path and a small Swift presentation change. Separate
implementation agents or an epic phase graph are unnecessary. The steps below are
ordered parts of that single implementation, not separately launched phases.

This shorthand applies when creating a new capture (including replacement of a legacy
reference without a captured PDF). Preserve automatic URL routing's existing
deduplication: an existing intake/library reference, pending job, or duplicate draft
item does not start another operation. Unlike an explicit `ref create -L` invocation,
the shorthand does not add audio to existing captures. For marked items that are
unchanged, state clearly that no new listen job was queued and show the existing
explicit `bob ref create <URL> -P <parent> -L` route to attaching audio. This boundary
matches the request to select listen when creating a new reference and avoids turning an
ordinary duplicate capture into a potentially paid narration operation.

No new CLI flags, config keys, Keep adapter protocol, Swift grammar, memory edits,
live-account pulls, installation, or deployment are required by this plan.

## Evidence and architecture

The following source and documentation were inspected:

- `src/native/capture_language/item.rs::claim_reference_item` and
  `editor_parse.rs::parse_editor_ref_item` separately claim one URL with an optional
  plain route. A bare trailing `@` currently falls into incomplete route syntax.
- `src/native/url_routing/intent.rs` owns strict URL classification and lexical URL-list
  detection; `capture_language/draft.rs::flush_capture_item` uses the latter before
  reference routing. Do not make `classify_token` accept capture directives.
- `src/native/gkeep/plan.rs::url_only_intent` independently recognizes strict Keep notes
  and optional trailing routes. Its WebLink-title matching must compare the URL alone.
  Keep content fingerprints and guarded archives use the original note.
- `src/native/capture/plan.rs` stages `ref_jobs::NewJob`; `ref_jobs/spool.rs` persists
  it. `ref_jobs/worker.rs` and `gkeep/pull.rs::run_clip_pre_pass` both invoke
  `highlights_ref/ingest.rs::ingest_url`, whose defaults currently specify no audio.
  Ingest does not invoke the printing CLI create handler.
- `highlights_ref/create.rs::stamp_listen_and_install_pdf` and
  `highlights_ref/listen.rs` already implement command validation, placeholder quoting,
  MP3 validation, installation, collision checks, and recovery. However, `run_listen`
  prints to stdout and inherits child stdout, and `post_listen_error` prints a retained
  scratch path. Reuse requires an explicit non-stdout output mode.
- `docs/capture.md`, `docs/gkeep.md`, `docs/ref-jobs.md`, and
  `docs/highlights-create.md` specify offline capture previews, asynchronous Mac
  capture, guarded Keep archiving, and all-or-nothing listen installation.
- In **bob-mac-capture**: `Sources/CaptureCore/CaptureModels.swift`,
  `CaptureRefPresentation.swift`, `CaptureRefFileUnder.swift`,
  `CompletionRowContent.swift`, and the macOS workflow define tolerant additive JSON
  decoding, reference cards, and a File-under picker which appends ` @route` to the
  existing draft. Preserve that marker while choosing the parent.

The audited `decisions:mac-capture-is-a-thin-client` memory requires all parsing,
completion, previews, and vault writes to remain in bob-cli. Swift only decodes and
presents Bob's answers. Before modifying the Mac repo, use
`sase repo open bob-mac-capture -r <reason>` and read any applicable instructions.
During planning that configured checkout was unavailable;
`sase repo open gh:bobs-org/bob-mac-capture -r <reason>` succeeded. Reopen through that
mechanism if needed and use only its returned path; do not assume the planning checkout
persists.

## Exact syntax and compatibility

> [!decision] listen_marker = at The examples below use `@`. If review selects
> `percent`, replace only the standalone listen token with `%`; route syntax and all
> downstream fields stay identical. Under that choice, recognize the complete eligible
> reference shape before clipboard extraction. `URL %` gains listen semantics, while
> `%N`, `%header`, ordinary task clipboard capture, and ineligible URL/task shapes
> retain their current behavior. `--no-clip` still controls clipboard syntax only;
> `--no-ref` disables this shortcut.

“Immediately after” means the next **whitespace-delimited token on the same line**, as
in the supplied example. One or more spaces/tabs are allowed. Do not strip an attached
character from `https://example.com/path@`, URL queries/fragments, encoded `%40`/`%25`,
or userinfo. Such bytes remain part of the URL and retain existing URL validation.
Accept one optional `<...>` wrapper around the URL as today.

| Input                                                                    | Capture                                                              | Keep                                                         |
| ------------------------------------------------------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------ |
| `URL @`                                                                  | Reference, listen requested, default/global/forced parent            | Reference, listen requested, existing parent selection       |
| `URL @ @sase`                                                            | Reference with explicit parent                                       | Reference with trailing explicit parent                      |
| `@sase URL @`                                                            | Reference with leading explicit parent                               | Preserve existing behavior; do not add leading-route grammar |
| `URL @sase`                                                              | Ordinary reference, no listen                                        | Ordinary reference, no listen                                |
| `URL`                                                                    | Existing reference behavior, no listen                               | Existing reference behavior, no listen                       |
| `URL @sase @` or `URL @ @`                                               | Not a new supported shape; preserve generic task/diagnostic behavior | Preserve ordinary-note behavior                              |
| `Read URL @`, child lines, tags, scheduling, priority, task destinations | Preserve existing task/diagnostic behavior                           | Preserve existing strict reference eligibility               |

Under the default choice, `URL %` continues to capture clipboard content. Under either
choice, a non-URL bare `@` and prefixes such as `@sa` remain route completion.
Completing a claimed listen token itself must not replace it with a route. Typing
letters onto it deliberately changes its meaning back to `@route` under the default
choice; typing a separate ` @sa` after `URL @` completes the parent normally.

Keep supports the marked expression either in its body with an empty/matching
URL/WebLink title, or in its title with an empty body. Retain existing exclusions for
checklists, attachments, shared notes, unrelated titles, and extra prose. Do not
interpret capture globals, arbitrary task syntax, or multiple URLs inside one Keep note.
Preserve all existing unmarked note shapes.

The marker is item-local and opt-in. Apply normal capture/Keep URL-routing policy, host
exclusions, `-R`, and destination restrictions before claiming it. When not claimed,
preserve the existing task, clipboard, or incomplete-route behavior; do not silently
strip the token. Extend lexical URL-list splitting only to lines consisting of a URL
plus the selected standalone marker, allowing mixed marked/unmarked URL lists. Keep this
splitting independent of routing policy like current URL lists; each resulting item then
applies its own policy. Routes in such lists continue to follow existing item-boundary
rules.

## Implementation

### 1. Share the suffix recognizer and carry structured intent

Add a small pure recognizer in `url_routing` for the URL token plus optional adjacent
listen token, returning the URL and marker ranges separately. Reuse it from strict
capture parsing, editor parsing, URL-list detection, and Keep classification. Keep
entry-point-specific route and note-eligibility checks in their existing layers. Avoid
independent string-replacement heuristics in those callers.

Carry `listen: bool` as reference-request metadata (a dedicated reference intent
wrapping `UrlIntent`, rather than modifying the URL's identity). Defaults are false.
Update `CaptureKind::Ref`, parser/editor models, and Keep's planned-note/reference
metadata accordingly. Normalized reference bodies, cleaned URLs, display URLs, dedupe
keys, and network targets exclude the directive; original input and Keep content remain
available for provenance and archive guards.

Add `ref_listen` to reference items in `capture-parse` (and the top-level single-item
view, following `ref_parent` conventions), a `ref_listen` semantic span over the token,
and `ref.listen` to dry-run/submit results. Keep schema version 1 and make additions
optional/default-false for readers. Preserve exact UTF-8 offsets and route spans. For
unchanged marked results, include an optional `ref.listen_hint` with Bob's no-new-listen
explanation and properly quoted explicit command, so Swift presents it without inventing
operation semantics. Use the equivalent detail in Keep reports. Adapt the completion
path so only a recognized listen span suppresses route completion; verify that its
policy agrees with the parser rather than suppressing every trailing `@`. Preserve
`capture-rewrite` input bytes and route/global inheritance.

### 2. Persist listen intent and make the reference importer audio-capable

Thread the boolean through `StagedRefJob`, `NewJob`, serialized `JobFile`, the worker's
`IngestRequest`, and relevant job reports/history. Old job records without the field
deserialize as false; write/read/recovery/stuck-job paths retain true. Do not include
the marker or boolean in URL dedupe keys, or mutate already-running jobs to upgrade
them. Same-URL draft duplicates retain first-item-wins behavior, including mixed
marked/unmarked duplicates; the duplicate row explicitly reports that it queues no new
listen operation.

Extend `IngestRequest` with the boolean. Preserve the fast unchanged outcomes for
existing references. For a new marked import, resolve/validate the configured listen
command before fetch/render, while retaining the ingest lock and its under-lock dedupe
recheck. Support article, PDF URL, and arXiv paths using the same cleaned URL, resolved
title, unstamped PDF, and scratch audio placeholder meanings as CLI create. Keep
`ready`, route-default ref type, canonical parent, and no-force defaults.

Extract/reuse the existing stamp/listen/install operation and listen runner with an
explicit output policy; do not spawn `bob ref create`, parse its stdout, or implement a
second narration command contract. CLI `ref create -L` retains inherited terminal output
and its existing behavior, including attach mode. Ingest sends progress and child output
to stderr/log handling and never contaminates JSON stdout. Quiet mode must remain quiet;
avoid buffering unlimited child output and do not introduce a second worker process or
new short timeout for narration.

Keep PDF and audio in scratch until narration succeeds and produces a valid MP3. Recheck
target/audio collisions after narration, install audio then PDF, clean up only audio
created by this attempt if PDF installation fails, and retain paid audio with the
existing recovery hint on post-listen errors. Share these guarantees with create; do not
install a PDF first and then run narration on it. Preserve the original `ref create -L`
tests and behavior while extracting helpers.

Carry listen intent into all failure/retry paths, including reconstructed stuck-job
fallback messages: `bob ref create <properly-quoted-clean-URL> -P <parent> -L`.
Capture's fallback URL task remains the exact previewed line; the warning preserves the
requested operation and any retained-audio recovery information. Add deliberate typed
mappings for listen configuration, execution, missing-MP3, and interruption failures
instead of relying on unrelated network-message substring matching.

### 3. Integrate Keep without weakening archive guarantees

Pass each planned note's listen bit to the shared importer; expose it in Keep list
classification, pull dry-run/human output, and JSON reference reports. Both preview and
execution must classify the same marked note. Existing parent alias resolution and
interactive parent selection stay unchanged. Keep dry runs offline with respect to
clipping/narration and free of vault/spool changes, apart from the existing Keep
snapshot operation; capture-parse remains lexical and does not validate listen config.

For a new marked note, record `ref_created` and permit guarded archive only after both
PDF and companion audio succeeded. A missing/invalid command, failed narration,
interruption, or missing MP3 leaves the note unarchived and reports failure with the
listen-preserving retry/recovery guidance. Treat these narration failures as retryable
in Keep's existing failure flow; do not convert them into a plain successful clip.
Existing permanent non-listen clipping failures may still use the verified fallback task
flow, with `-L` retained when requested. Preserve original content fingerprints,
edit-during-pull guards, no-archive mode, and ref-created recovery semantics. A crash
after successful audio/PDF installation but before journal write must recover through
dedupe without narrating again. Existing known references remain unchanged outcomes,
with an explicit no-new-listen explanation.

### 4. Present the operation in the Mac client and CLI

In bob-mac-capture, add tolerant `decodeIfPresent` fields/defaults in
`CaptureModels.swift`; map the new semantic span in `CompletionRowContent.swift` and the
existing palette as needed. Render a small “Listen” cue and truthful
queue/preview/accessibility text in `CaptureRefPresentation.swift` and its card. Queued
means the request was saved, never that narration finished. On an unchanged marked item,
show that no new audio operation will run and the explicit retry/attach hint supplied by
Bob; do not imply a queued operation merely from `ref.listen`.

Keep File-under selection working on `URL @`: appending the existing ` @route` must
retain the listen token and reanalyze through Bob. Preserve single aggregate submission,
editor text on failure, and the existing subprocess timeout because capture still only
queues. No Swift URL or marker parser and no new direct invocation of `ref create` are
permitted. Older Bob payloads without listen fields keep their current presentation.
Ship the updated Bob worker before relying on marked queued jobs; old binaries that
ignore additive job fields must not be left consuming them.

Match CLI human wording to the card. Update `docs/capture.md`, `docs/gkeep.md`,
`docs/ref-jobs.md`, `docs/highlights-create.md`, and the Mac README with the chosen
token, whitespace rule, route composition, listen-command prerequisite, offline preview
behavior, scan requirement, failures, and existing-reference boundary.

## Verification and acceptance

Use isolated temporary vaults/state/config and fake Keep, curl/web-clip, and listen
commands. Reuse `tests/cli/highlights/listen.rs`'s MP3-writing/failure fixtures and the
existing capture, jobs, and Keep integration-test harnesses. No real narration, paid
API, feed publishing, browser login, or personal Keep/vault mutation is needed.

1. **Grammar and contract parity:** unit and CLI cases for the chosen token, brackets,
   spaces/tabs, leading/trailing routes, aliases, global/forced parent, mixed URL lists,
   Unicode byte ranges, excluded hosts, disabled routing, `-R`, extra words, child
   lines, literal/encoded URL punctuation, duplicate tokens, and the unchanged clipboard
   and route-completion behavior. Test parse, completion, rewrite, dry-run, and
   submitted job metadata together; routing-off behavior must not accidentally become a
   reference or consume the new directive.
2. **End to end through both entry points:** a new marked article produces PDF plus MP3
   with the expected parent; an unmarked article produces PDF without invoking listen.
   Cover PDF URLs and arXiv through the shared importer. Verify exact fake command
   arguments exclude marker/route and preserve URL/title quoting. A scan of at least one
   created pair yields a reference note with bound companion audio.
3. **Offline and asynchronous behavior:** capture parse/dry-run never invoke fetch,
   listen, clipboard access for the claimed shape, or spool writes. Keep dry-run
   snapshots through the fake adapter but never clips/narrates. Capture submit saves the
   bit and returns before a deliberately delayed worker listen finishes. A noisy fake
   listener cannot corrupt JSON stdout on either success or failure.
4. **Durability, failures, and dedupe:** old job JSON defaults false; marked jobs
   round-trip/recover true. Failed/missing-MP3 narration installs neither final file;
   post-listen collisions preserve recovery audio and no existing file is overwritten.
   Capture fallback and stuck retry contain `-L`. Keep narration failure never archives
   or writes a false success receipt; re-pull succeeds once repaired and edit guards
   still work. Confirm known references, pending jobs, and both orders of mixed same-URL
   duplicates do not trigger extra narration or claim audio was queued.
5. **Mac compatibility:** add decode fixtures with and without the new fields, span
   styling and reference presentation/accessibility tests, plus a File-under test
   proving `URL @` becomes `URL @ @route` and remains a listening reference. Use the
   existing fake Bob process contract in panel tests. No frontend syntax inference.

Target suites: `capture_language::tests::ref_grammar`,
`tests/cli/capture/{ref,ref_grammar,complete_ref_kind}.rs`,
`tests/cli/highlights/{jobs,listen,url_routing}.rs`, the `ref_jobs`/`ingest` unit tests,
and `tests/gkeep_{list,pull,pull_recovery}.rs`. Extend existing fixtures rather than
duplicating harnesses. Then run the repository gate `just check` (format, clippy, all
test binaries). For bob-mac-capture run `just format-lint`, `just build`, and
`just test` on its supported macOS toolchain/CI; Linux-only source review does not count
as passing Swift verification. Report any unavailable platform verification explicitly.
Use the SASE monitor workflow if a check needs a long-running handoff.

Completion requires the accepted syntax working in both entry points, additive Mac
presentation, reliable audio intent through jobs/recovery, honest unchanged outcomes,
updated documentation, and the relevant Rust and macOS checks passing. Implement no
product changes until this plan is approved.
