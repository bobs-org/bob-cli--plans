---
tier: tale
title: Give URL percent listen intent precedence over clipboard capture
goal:
  Percent-suffixed reference URLs queue companion audio and preview correctly in Bob Mac
  Capture without reading the clipboard.
size: medium
proposed_by: bbugyi200.apollo.6n
create_time: 2026-10-10 17:30:42
status: wip
---

# Give URL `%` listen intent precedence over clipboard capture

## Outcome and scope

For an eligible new reference, submitting `https://arxiv.org/pdf/2609.12039 %` through
Bob Mac Capture must queue that URL with companion audio requested, using the existing
ingest behavior behind `bob ref create <URL> -P <parent> -L`. It must not read the
clipboard or immediately write a URL task plus clipboard child into `mac_inbox.md`. Live
preview, explicit preview, and submission must agree on that interpretation.

This is a medium tale: one implementation agent can make the bounded grammar,
regression-test, and Mac presentation changes using the existing job/listen pipeline. No
new pipeline, CLI options, protocol version, or memory changes are needed. Work covers
bob-cli and bob-mac-capture; the shared recognizer's existing Google Keep consumer gets
matching tests and documentation, not a new import workflow. Do not alter the user's
existing inbox task or launch narration against the real vault as part of automated
verification.

## Diagnosis and evidence

- bob-cli HEAD inspected: `d1d50a6`
  (`feat(capture): add URL listen marker @ for companion audio`). Its implementation and
  `docs/capture.md` explicitly choose standalone `@` for listening and preserve `URL %`
  as clipboard syntax. `src/native/url_routing/intent.rs::is_standalone_listen_marker`
  currently returns only `token == "@"`.
- Reference claiming already precedes generic terminal-marker extraction in
  `src/native/capture_language/item.rs::claim_reference_item`. The editor has the
  corresponding early claim in `editor_parse.rs::parse_editor_ref_item`. `%` is not
  recognized by either reference claim; it falls through to
  `markers.rs::parse_clip_token`, becoming `ClipRequest::Current { header: None }`.
  `src/native/capture/plan.rs` then reads the clipboard and renders its children. The
  primary defect is the listen marker contract, not a Mac-side URL parser or a failure
  to forward a recognized listen request.
- A read-only reproduction with the host's installed `bob`, an isolated missing
  `BOB_CONFIG_FILE`, a fixture vault, fixed `BOB_NOW`, and
  `BOB_CLIPBOARD_CMD='/usr/bin/printf https://arxiv.org/pdf/2609.12039'` confirmed:
  `capture-parse -f json -- 'https://arxiv.org/pdf/2609.12039 %'` returns `mode: task`
  and a `clipboard` span at bytes 33..34; `capture -d -f json` returns the reported task
  dated 2026-10-10 and the duplicate URL as an inline clipboard child. The user's
  historical clipboard contents are inferred, not inspected.
- The same host binary treats a bare URL as `kind: ref`, and `%` under `-d -n` as
  literal task text. It does not yet recognize `URL @`, although HEAD implements it.
  Treat this as a separate installation-version observation on the agent host, not
  evidence of which binary the user's Mac runs. Use the newly built checkout binary in
  implementation tests; `bob --version` alone (0.1.0) cannot establish that a feature is
  installed.
- Bob Mac Capture HEAD inspected: `daf70c2`. `CaptureCore/BobProcessClient.swift` passes
  the aggregate draft unchanged to bob. Live preview uses
  `capture --dry-run --no-clip --format json -- <draft>`; explicit preview and
  submission allow clipboard expansion. Do not remove `--no-clip`: listening is
  reference intent and must be independent of that clipboard flag.
- Mac `CaptureModels.swift` currently lacks `ref_listen`, `ref.listen`, and
  `ref.listen_hint`; unknown JSON is ignored. `CompletionRowContent.swift` lacks a
  `ref_listen` style, and `CapturePanelModel.livePreviewUsesLiteralClipboard` uses
  `plainDraft.contains("%")`, incorrectly warning even for URL escapes or the proposed
  listen marker. These are presentation gaps, not the cause of the task write.
- Existing propagation is present: `RefIntent.listen` -> reference output and
  `ref_jobs::NewJob.listen` -> `JobFile.listen` -> worker `IngestRequest.listen` ->
  configured listen runner in `highlights_ref/ingest.rs`. Reuse it.
- Follow the accepted `decisions:mac-capture-is-a-thin-client` record: bob owns grammar
  and vault writes; Swift only decodes, orchestrates, and presents.

Before reading or editing the other repository, use `/sase_repo` and the path it prints.
During diagnosis, `sase repo open bob-mac-capture` failed because the configured primary
clone was absent; `sase repo open gh:bobs-org/bob-mac-capture` succeeded. Use those
audited mechanisms again in the implementation session; never rely on a previous agent's
checkout path. Read any applicable instructions in the opened repository.

## Required behavior

1. Make standalone `%` the documented URL listen spelling; keep the shipped `@` spelling
   as a compatible alias. Both use the same eligibility and downstream `listen` flag.
   This task does not retire the alias.
2. Claim only the existing complete reference shapes, substituting either listen token:
   `URL %`, `URL % @route`, and `@route URL %`. Support the existing optional `<URL>`
   wrapper, spaces/tabs, plain `@@route` inheritance, bare `-r route`, route aliases,
   and mixed marked/unmarked URL lists. Strip only the separate directive token from
   reference identity, never bytes within the URL.
3. The reference claim wins over generic `%` clipboard handling for these admitted
   shapes, including with `--no-clip`. It produces `clip: None`, `mode/kind: ref`,
   `ref_listen: true` and a `ref_listen` span in parse output, and `ref.listen: true` in
   capture output. Keep schema version 1 and the existing omission/default behavior for
   unmarked items.
4. Preserve explicit opt-outs and current non-reference behavior: `-R`, routing disabled
   or invalid, excluded hosts, forced `--clip`, forced section/task destinations, extra
   prose, authored children, schedules/priorities, dependencies, and operators do not
   become references just because `%` occurs. There is no new bypass of URL policy. In
   those cases normal clipboard rules still apply, and `--no-clip` keeps clipboard
   tokens literal.
5. Only exact standalone `%` and the existing exact `@` alias request listening. Non-URL
   `text %`, `%N`, and `%header` retain clipboard semantics. Document `URL %1` or
   `--clip URL` as ways to explicitly request clipboard capture for a URL; `-R 'URL %'`
   also keeps the clipboard path. Attached percent characters, `%25`/`%40`, queries,
   fragments, and URL userinfo keep their existing URL validation/cleaning semantics.
   Duplicate listen tokens, leading `%`, or misordered `URL @route %` are outside the
   admitted grammar and retain generic task/diagnostic handling. No broad string
   replacement or URL suffix trimming.
6. Keep existing deduplication and asynchronous behavior. New captures queue narration
   rather than waiting for it in the app. Already-known captures do not silently attach
   audio or upgrade old jobs; expose the existing `listen_hint` with the explicit
   `bob ref create ... -P ... -L` retry/attach command. Listen failures retain the
   established fallback/error behavior, including `-L` in retry guidance. An old job
   lacking `listen` remains false.

## Implementation

### 1. Fix the shared marker contract in bob-cli

Extend the exact-token recognizer in `src/native/url_routing/intent.rs` to accept `%`
plus the compatible `@` alias. Audit its callers in `capture_language/item.rs`,
`editor_parse.rs`, `completion.rs`, URL-list splitting, and `gkeep/plan.rs`. The
existing early reference claims should provide the precedence; do not disable or
globally rewrite `parse_clip_token`.

Ensure parse, execution, and completion agree on the admitted shapes. A claimed listen
marker has no route-completion replacement; a separate ` @sa` after `URL %` must still
complete the parent and preserve `%`. Cover the existing `@` completion rules too.
Preserve normal route and non-reference clipboard behavior.

Update comments/help and `docs/capture.md` to specify the context-dependent `%`
precedence and the live-preview `--no-clip` exception for references. Correct the
current statement that `URL %` always means clipboard capture and qualify the claim that
`%1` is equivalent to `%`. Check `capture/cli.rs` help and relevant help goldens. Update
the shared URL examples in `docs/gkeep.md`, `docs/highlights-create.md`, and
`docs/ref-jobs.md` where they describe listen markers, retaining the alias and the
asynchronous/dedupe qualifications.

### 2. Cover the actual queued execution path

Add focused regressions to `capture_language/tests/ref_grammar.rs` and
`tests/cli/capture/ref.rs` (or a narrowly scoped sibling registered in its module). Use
the user's exact arXiv URL plus representative article/PDF URLs. Extend the existing URL
and clipboard tables rather than inventing a second test harness.

Use isolated vault/config/state roots and the existing fake clip/listen tools. Disable
automatic job kickoff with `BOB_REF_JOBS_KICK=off` where inspecting jobs, then drain
deliberately with `bob ref jobs run`. A clipboard command that fails and records
invocation must remain uncalled for admitted listen references in parse, both dry-run
forms, and real capture. A URL-valued clipboard must not produce a duplicate child or an
immediate inbox task.

Verify the persisted job has `listen: true`, the correct parent, and unchanged
URL/dedupe identity. Add an offline worker test that takes a job originating from
`URL %` through a fake fetch/render/listen success, proving narration is invoked and its
audio result is installed by the existing pipeline. Include the exact arXiv route in the
fake coverage where feasible; at minimum assert its classifier and queued job plus one
complete fake worker execution. Verify a listen failure retains listen intent and `-L`
retry guidance. Reuse old-job/default-false and already-known/no-new-job coverage,
adding assertions where currently missing. Do not redesign ingest or change
deduplication behavior to make a test pass.

The shared Google Keep recognizer should accept `%` in its already-supported URL
positions as well. Add focused plan/list/pull regressions using existing fake Keep
adapters, including `listen: true`, preserved parent and URL identity, the `@` alias,
and existing non-reference shapes.

### 3. Present listen intent correctly in Bob Mac Capture

In the opened bob-mac-capture repository:

- Extend `Sources/CaptureCore/CaptureModels.swift` with additive, tolerant
  decoding/defaulted initializers for top-level and item `ref_listen`, plus
  `CaptureRef.listen` and optional `listenHint`. Older bob fixtures still decode with
  false/nil defaults; keep schema version 1.
- Handle the `ref_listen` semantic span in
  `Sources/CaptureCore/CompletionRowContent.swift` using an appropriate existing
  non-clipboard accent. Do not parse `%` or URL grammar in Swift.
- Extend `CaptureRefPresentation.swift` and the existing reference card as needed to
  show a concise companion-audio-requested indication. Queued wording must not promise
  audio is already available. Surface `listen_hint` for unchanged references. Include
  the request/hint in accessibility text and keep existing parent/fallback information
  legible. Match bob's shared detail wording where the presentation currently promises
  parity.
- Change `CapturePanelModel.livePreviewUsesLiteralClipboard` to derive the notice from
  bob's clipboard spans for the current draft, rather than searching raw text for `%`.
  Listen spans and URL-encoded `%25` must not show the clipboard notice; actual
  clipboard markers in a mixed draft must still show it. Avoid displaying
  classifications from stale parse responses.
- Preserve `BobProcessClient`'s aggregate-draft argv and live `--no-clip` guard. The
  File-under picker appends ` @route`; test that accepting it for `URL %` preserves the
  marker and produces the supported `URL % @route` shape.
- Add real bob JSON fixtures and focused decode, presentation, semantic-span,
  process-argv, and panel/model regressions using the existing reference tests. Cover
  `%`, the alias, mixed items, old payloads without new fields, and already-known
  `listen_hint`. Update README listen examples and its blanket claim that all `%`
  markers stay literal in live preview.

## Acceptance matrix

| Input/context                                                                 | Required outcome                                                        |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Exact reported arXiv URL plus `%`                                             | Ref with listen true; clipboard untouched; no immediate inbox task      |
| Same input with live `-d -n`                                                  | Same reference intent, no enqueue/fetch/narration/vault write           |
| Explicit preview `-d`, then real submission                                   | Preview agrees; submission queues one listen job                        |
| Wrapped URL, tab before `%`, explicit/global/forced parent                    | Clean URL identity, correct parent and span ranges                      |
| `%`-marked and unmarked URLs in one URL list or mixed draft                   | Per-item intent; no listen leakage; correct UTF-8 byte ranges           |
| URL with compatible `@` marker                                                | Existing listen behavior preserved                                      |
| Non-URL `text %`, URL `%1`, `%header`                                         | Existing clipboard capture, including `--no-clip` behavior              |
| `-R`, disabled/excluded routing, forced clip/destination, task-only shapes    | Existing task/diagnostic semantics; no unexpected listen job            |
| Attached/encoded `%`, queries/fragments, leading/duplicate/misordered markers | No false listen intent or mutation of URL bytes                         |
| Already-known reference or legacy job without listen                          | Existing dedupe/defaults; truthful no-new-listen hint                   |
| Mac File-under accept and later capture                                       | `%` survives route insertion; one unchanged aggregate draft reaches bob |
| Mac listen-only versus mixed clipboard draft                                  | Audio request visible; clipboard notice only for actual clipboard spans |

## Verification and delivery

Run focused Rust tests for the grammar, capture/reference/clipboard CLI surfaces,
job/ingest listen propagation, and affected Keep suites. Then run the repository's
canonical `just check` (format, clippy, all test binaries) once; use `/sase_monitor` for
long commands in SASE and follow its handoff instructions. Use the checkout build or
test harness, not whichever installed `bob` happens to be on PATH.

For Mac changes, run the repository's `just all`/equivalent documented CI sequence
(format lint, Apple-toolchain build, tests, bundle) on macOS 26. Linux source review
cannot replace Swift/AppKit validation; use the available macOS CI workflow and record
actual results or an explicit platform limitation, never a fabricated pass. Include the
relevant fixture/panel tests in that run.

Before declaring the issue resolved on the user's device, verify that the app's
configured executable override or `BobExecutableResolver` candidate is the updated bob
build. With an isolated vault, the exact input must preview as a listen reference and
submit a listen job using that executable. Updating only the app cannot change CLI
grammar. Record the tested CLI revision, Mac revision, test results, and any remaining
device-installation step. Do not run real-vault narration or repair the historical inbox
entry without a separate user request.
