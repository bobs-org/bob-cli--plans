---
tier: epic
title: Keep a typed / literal after album routes and open the record list
goal: 'In Bob Mac Capture, typing `/` right after an album route such as `@eat` inserts
  a literal `/` (the `@eat/` record route) instead of the `/` → `#` capture alias,
  an explicit `#` still opens the section menu, and a draft that is only `@eat/` immediately
  lists that album''s existing record notes.

  '
phases:
- id: bob_rewrite
  title: bob capture-rewrite keeps / literal after album routes
  depends_on: []
  size: medium
  description: 'bob_rewrite: teach capture-rewrite''s / → # alias (and its typing-tip
    proof) to stay literal when the / is glued to a single-@ album route, via one
    more bounded album-head read; add Rust unit and CLI tests, update docs/capture.md,
    the CLI help, and the capture-alias glossary strand.'
- id: mac_record_list
  title: Mac Capture opens the record list on a bare @album/ draft
  depends_on:
  - bob_rewrite
  size: small
  description: 'mac_record_list: in bob-mac-capture, skip the cached route list when
    the caret''s record_route span is the whole draft so bob''s record_target rows
    show; add real-bob fixtures, fake-bob cases, hosted tests, and docs; verify on
    macOS CI.'
decisions:
  capture_alias_glossary:
    ask: Update the Capture Alias glossary strand to say / stays literal after an
      album route?
    memory:
    - glossary:capture-alias
    default: true
    corrects: '`/` → `#` wherever a `#` menu would open'
    answer: true
proposed_by: bbugyi200.athena.1d8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 06:44:43
status: wip
bead_id: bob-cli-mn
---

- **PROMPT:** [prompts/202611/album_slash_record_list.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_slash_record_list.md)
- **BEAD:** [bob-cli-mn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mn/README.md)

# Plan: Keep a typed `/` literal after album routes and open the record list

## Background

Two repositories are involved. `bob-cli` is this project. `bob-mac-capture` is a linked
repository: open it with `sase repo open bob-mac-capture -r "<reason>"` and read its
`AGENTS.md` before editing.

- The `/` → `#` capture alias belongs to bob, not the app
  ([[decisions:mac-capture-is-a-thin-client]]). After each keystroke the Mac app sends
  the post-key draft and caret to `bob capture-rewrite --cursor` and applies whatever
  edit bob returns. The alias lives in `src/native/capture_language/rewrite.rs`:
  `expand_sigil_alias` → `expand_hash_alias`. It turns `/` into `#` wherever the
  `#`-swapped candidate opens a `#` menu.
- `@eat#` opens the Section menu, so today `@eat/` becomes `@eat#` (verified:
  `bob capture-rewrite -c 5 -f json '@eat/'` returns `"text":"@eat#"` with rule
  `expand_sigil_alias`). However, `@eat/` is also the record-route grammar for an album
  (`docs/records.md` "Capture", `docs/capture.md` "Record notes"): `@eat/` creates a
  record and `@eat/<stem>` updates one.
- bob already serves the record list. `bob capture-complete -c 5 '@eat/'` returns
  context `record_target` with every existing record (replacement 0..5, empty query,
  rows `@eat/<stem>`, continuation `next_missing_field: true`).
  `bob capture-parse '@eat/'` reports a `record_route` span 0..5 and needs
  `["record_title","record_value"]`.
- In the Mac app, `CapturePanelModel.shouldRequestCompletion` already requests
  completion for a `record_route` span. But `cachedRouteCompletion` runs first. It
  treats `record_route` as a route span (`routeReplacementRange`), so the query becomes
  `eat/`. That query prefix-matches the cached `capture-targets` record row (route
  `eat/`, kind `record`), so the app shows the cached "🍽️ New restaurant · N" route row
  and never asks bob for the `record_target` list. A literal `@eat/` therefore does not
  list records today, even when the `/` arrives by another route.
- `capture-rewrite` already performs one bounded vault read: notebook route frontmatter.
  This happens only when `bullet_sigil_may_need_notebook_policy` matches (see
  `CaptureRewriteResult::new` in `src/native/capture_rewrite.rs`). The record-route
  policy is built by `capture_record::record_policy_for_draft(vault_root, raw_text)`. It
  resolves only the lexical `@dir/` candidates, one album head each, and
  `RecordRoutePolicy::route_decision` / `is_usable_record_route` expose the result.
- The typing tip (`sigil_alias_tip_with` in `alias_tip.rs`) proves a tip by swapping the
  candidate key in and rerunning the forward rewrite. It must see the same album policy.
  Otherwise an explicitly typed `@eat#` would teach "Next time type `/`", and that is
  now wrong.

## Phase `bob_rewrite`: bob capture-rewrite keeps `/` literal after album routes

### Behavior

- Typing `/` glued to a complete single-`@` route token (`@eat/`, `Buy milk @eat/`,
  `@eat/ Title` while typing, and any other position where the `#` menu would open)
  stays literal when that route resolves to an album. That covers both
  `PerRouteDecision::AlbumValid` and `AlbumInvalid`, because the record grammar claims
  both and a broken album still teaches through its `album_schema_invalid` diagnostic.
  `Unclaimed` and `DiscoveryIncomplete` routes keep today's `/` → `#` alias. So do a
  missing vault and non-album routes such as `@dev/` → `@dev#`.
- An explicitly typed `#` is unchanged: `@eat#` still opens the Section menu with its
  `record_create` row.
- The typing tip for an explicitly typed `@eat#` no longer offers `/`. For `@dev#` it
  still teaches `/`.
- Out of scope, and unchanged: the `@@route/` global-destination collapse
  (`collapse_global_destination`) and every other alias.

### Implementation

1. In `src/native/capture_language/rewrite.rs`, add a cheap lexical preflight next to
   `bullet_sigil_may_need_notebook_policy`. Name it
   `slash_route_may_need_album_policy(raw_text, cursor) -> Option<String>` or similar.
   It returns the route when the byte before `cursor` is `/` or `#`, and that byte is
   glued to `@<route>`, where `<route>` satisfies `is_route_token`. The `@` must be
   single (not `@@`) and sit at a token boundary, meaning start of text or preceded by
   whitespace. Re-export it from `capture_language/mod.rs` like the notebook preflight.
2. Thread an album policy through the rewrite pipeline. Replace the growing positional
   parameters of `rewrite_draft_with` and `sigil_alias_tip_with` with one small borrowed
   struct, e.g.
   `RewritePolicies<'a> { notebook: Option<&NotebookRoutePolicy>, record: Option<&RecordRoutePolicy> }`.
   Keep `rewrite_draft(raw, cursor)` as the policy-free wrapper. Update the doc
   comments: "the only policy-aware assist" is no longer true.
3. In `expand_hash_alias` (or at the `/` branch of `expand_sigil_alias`), return `None`
   before any candidate evaluation when the `/` ends `@<route>/` and the record policy
   decides that route is `AlbumValid` or `AlbumInvalid`. Add a
   `RecordRoutePolicy::is_album_route(route)` helper if that reads better than matching
   `route_decision` inline. With no record policy (`None`), behavior is exactly today's.
4. In `src/native/capture_rewrite.rs` `CaptureRewriteResult::new`, when a cursor is
   present and the new preflight returns a route, build the policy with
   `record_policy_for_draft(&bob_env::bob_dir(), &format!("@{route}/"))`. That is one
   album-head resolution for exactly that route, and it also serves the `#` tip proof.
   Pass it to both the rewrite and the tip proof. No other draft triggers this read.
   Preserve the notebook read exactly as it is.
5. Update the `capture-rewrite` long help in `src/native/capture_rewrite.rs`. The
   command now has two bounded vault reads (notebook frontmatter for a bullet-start
   `/`/`#`, and one album head for a `/`/`#` glued to `@route`). In the Sigil alias
   paragraph, state that `/` stays literal after an album route (`@eat/` keeps the
   record route) and that a typed `#` still opens the section menu. Adjust
   `capture_rewrite_help_describes_only_the_remaining_rules` if its assertions cover the
   changed sentences.

### Tests

- Unit tests in `src/native/capture_language/tests/rewrite.rs` (and the alias-tip
  tests). Build `RecordRoutePolicy::absent()` plus `insert_route` with an `AlbumValid`
  and an `AlbumInvalid` decision. Follow the existing record-policy test helpers, so
  search `capture_language/tests` for `insert_route`. Cover:
  - `@eat/` (cursor 5), `Buy milk @eat/`, and `@Eat/` stay unchanged.
  - The invalid album also stays literal.
  - `@dev/` with a policy that lacks `dev`, and `@eat/` with no policy, still become
    `#`.
  - `@@eat/` still collapses to `@eat#`.
  - A tip on `@eat#` with the album policy is `None`, while `@dev#` still proves the `/`
    tip.
- CLI tests in `tests/cli/capture/rewrite.rs`, modeled on `notebook_alias_vault` /
  `capture_rewrite_json_expands_slash_to_hash_in_a_notebook_list`. Write a temp vault
  with a root album note (`type: "[[album]]"` plus a minimal `record:` schema; copy the
  shape from `tests/cli/records.rs` or `tests/cli/albums.rs`) and set `BOB_DIR`. Assert:
  - `capture-rewrite -c 5 -f json '@eat/'` returns `changed: false`, text `@eat/`, and
    `sigil_alias_tip: null`.
  - `@notes/` for a non-album area still returns `@notes#`.
  - `-c 5 '@eat#'` reports `sigil_alias_tip: null`.
  - `capture_rewrite_never_touches_the_vault_or_clipboard` still passes, and the
    missing-vault case never creates the directory. Add an `@eat/` probe against the
    missing vault, which should give `@eat#`.

### Docs and memory

- `docs/capture.md`, in the `bob capture-rewrite` "Sigil alias" paragraph (the sentence
  beginning "Typing `/` becomes `#`"): add the album-route exception and the second
  bounded read, replacing "the command's one bounded vault read". In the "Record notes"
  section, add one sentence: in Bob Mac Capture, typing `/` after an album route keeps
  `@eat/`, and a draft that is only `@eat/` lists the album's records. Read the
  `docs.md` reference memory first. Run `just docs-check`.

> [!decision] capture_alias_glossary

- When `capture_alias_glossary` is accepted, edit the `glossary:capture-alias` strand
  with the `/sase_memory_write` skill. In the `/` → `#` bullet, add that `/` stays
  literal right after an album route (`@eat/` is the record route; type `#` for the
  section menu). Then run `sase memory init`, and `just docs-gen` to regenerate
  `docs/reference/glossary.md`, which is generated from the glossary web. If declined,
  follow the skill's declined-change rule.

### Verification and landing

`just fmt`, `just lint`, `just test` (or `just all`), and `just docs-check`. Then
manually run `bob capture-rewrite -c 5 -f json '@eat/'` against a vault with an album.
Use the conventional commit `feat(capture): keep / literal after album routes` (not
breaking). Never edit `Cargo.toml`'s version or `CHANGELOG.md`; the release train owns
them ([[decisions:bob-cli-releases-ride-a-daily-train]]).

## Phase `mac_record_list`: Mac Capture opens the record list on a bare `@album/` draft

Work in the `bob-mac-capture` linked repo, opened via `sase repo open`. Follow its
`AGENTS.md`: compile everything, format only the changed lines, run `just docs-check`
for docs, and watch macOS CI.

### Behavior

- When the whole draft is a single album record route with the caret at its end
  (`@eat/`), the app asks bob (`capture-complete`) instead of the cached route list. bob
  returns `record_target`, so the existing record notes appear right after the `/`
  keystroke, with no extra key. Accepting a row keeps today's record-target accept path.
- Every other draft is unchanged. A title-bearing `Ramen Nagi @eat/` still shows the
  cached "New restaurant · N" row, and typing on (`@eat/ra`) already falls through to
  bob's filtered `record_target` list because the cache has no match.

### Implementation

1. In `Sources/BobMacCapture/CapturePanelModel.swift` `cachedRouteCompletion`, return
   `nil` (defer to bob) when `parse.spans` holds a `record_route` span with
   `start == 0`, `end == draft.utf8.count`, and `cursor == span.end`. That means bob's
   own span covers the entire draft. No Swift-side grammar sniffing: the decision uses
   only bob's span kind and range, in keeping with the thin-client rule. Put a short
   comment on the guard explaining why.
2. Confirm that the deferred path reaches bob. `shouldRequestCompletion` already returns
   true through `leadingSingleAtRouteReplacementRange` and the `record_route` span kind.
   Also check that `recordCompletionDidArrive` and the record-target rendering
   (`CompletionRowContent` `.recordTarget`) accept a title-less `@eat/` response, and
   fix any gate that assumes a title.

### Fixtures and tests

- In `Scripts/generate-record-fixtures.sh`, add
  `run_parse "@eat/" record-parse-eat-bare.json` and
  `run_complete "@eat/" 5 record-complete-target-bare.json`. Also add a
  `capture-rewrite` fixture for `@eat/` showing `changed: false`, mirroring how other
  `capture-rewrite-*.json` fixtures are produced. Generate with a `bob` built from
  bob-cli master that contains phase `bob_rewrite`. The fixtures are generated from real
  bob; never hand-edit the JSON. Commit the new fixtures and the README update. Commit
  unrelated regenerated churn only when hosted tests stay green with it, and otherwise
  leave those files untouched.
- In `Tests/Fixtures/fake-bob`, under `FAKE_BOB_RECORD_ASSIST`, route the `@eat/` draft
  to the new parse fixture (near the existing `"Blaze @eat/"` parse case) and the
  completion fixture (near the `capture-complete` record cases around the
  `"Blaze @eat/"` / `"Ramen Nagi @eat/"` entries). Route the rewrite case to its new
  fixture.
- Hosted tests, in `CaptureRecordPanelTests` or `CapturePanelModelTests`, following the
  movies-target cached tests that inject a record row into the targets cache:
  1. With the cache holding the `eat/` record row, simulate typing `@eat` then `/`. Use
     the same intent and `editorTextDidChange` path the existing alias tests use. Assert
     that the draft stays `@eat/`, the completion context is `record_target`, and the
     rows come from the fixture.
  2. Regression: `Ramen Nagi @eat/` with the same cache still shows the cached route row
     (context `route`). Async tests must never await an untimed continuation.

### Docs

- `docs/keyboard.md`, the `\ or /` row: `/` stays literal right after an album route
  (`@eat/`); type `#` for the section menu.
- `docs/capture-panel.md`: update the `/` alias sentence near "Else `/` aliases `#`
  where Bob opens its menu". Also find the record-capture text and state that a draft of
  only `@eat/` lists the album's existing records.
- Check `docs/typing-tips.md` for a `/` tip example that names a route and would now be
  wrong. Run `just docs-check`.

### Verification and landing

`just portable-check` on athena. Compile a scratch copy on the Mac (rsync without
`.build`/`.git`, `bundle.sh`, then `just typecheck-tests`; delete the copy afterward;
never use Bryan's checkout or `just install`). If the Mac is offline, say in the final
report that the macOS targets are unverified. Run `Scripts/macos-ci-check` before
landing. After the push, watch the exact-SHA macOS run to green with `/sase_monitor`.
Use the commit `feat(capture): list album records after a typed @album/`.
