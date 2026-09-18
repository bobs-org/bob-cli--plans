---
tier: tale
title: Keep route completion active for a leading single-at capture marker
goal:
  In Bob Mac Capture, typing `@` into an empty capture editor immediately offers
  project/area destination notes, typing the route fragment keeps that completion
  active, and typing `+` after the chosen route continues into the existing task-ID
  chooser so a marker-only `@<file>+<id>` task toggle can be authored entirely through
  completion without changing bob-cli's capture grammar.
size: small
proposed_by: bbugyi200.athena.0mz
create_time: 2026-09-18 11:21:39
status: wip
---

# Plan: Keep route completion active for a leading single-at capture marker

## Why this exists

Bob already has the required completion contract. `bob capture-complete` recognizes a
bare leading `@` and a leading `@fragment` as route-completion contexts, returns a
replacement range that excludes the sigil, and continues from `@route+` into task
completion. The parse endpoint intentionally has a different job: a leading `@fragment`
with no task body and no separator remains literal task text if submitted, so
`bob capture-parse` reports no completion need or route span for that intermediate
draft. It only reports the leading bare `@` as an incomplete route and recognizes the
marker again after `+` is present.

`CapturePanelModel` currently decides whether to ask for completion entirely from the
parse response's `needs` and spans. Consequently, an initial leading `@` can open route
completion, but the next route character changes the draft to the parse-valid literal
form `@fragment`; the gate then dismisses completion even though `capture-complete`
still identifies the route field. This blocks the natural marker-only task-toggle flow:

```text
@  ->  @fi  ->  @file+  ->  @file+task-id
```

The fix belongs in the Mac app's completion-trigger orchestration, not in bob-cli's
capture parser. Changing parse semantics would make a marker-only `@file` look routed
even though Bob deliberately treats it as literal text until a body or marker separator
establishes capture meaning.

## Work

### 1. Recognize the leading route fragment as an editor completion context

In `Sources/BobMacCapture/CapturePanelModel.swift`, introduce one UTF-8-safe helper for
the narrow source form that parse cannot describe: the draft starts at byte zero with a
single `@`, the caret is in that leading route component, and the component has not yet
reached a marker separator or body-text boundary. Have the helper return the route
replacement range excluding `@`, including the zero-width range immediately after a bare
`@`.

Use that result in both places that need the same decision:

- Allow `scheduleAnalysis`'s completion gate to proceed for this source form even when
  `capture-parse` has no completion need/span for `@fragment`.
- Let cached route completion use the returned range, preserving the existing
  authoritative target-cache ranking for project, area, and inbox candidates. If the
  cache is unavailable or has no match, preserve the current fallback to
  `bob capture-complete`, which remains the source of truth for the returned context and
  replacement.

Keep the exception deliberately narrow. Do not reinterpret `@@` global declarations,
route components after `#`, `^`, `:`, or `+`, arbitrary body text, wikilinks, another
batch item, or a non-collapsed selection; all of those continue through the existing
parse-span/completion paths. Continue validating all cursor and replacement offsets as
UTF-8 byte boundaries.

Do not special-case the later `+` behavior. A leading route response must populate the
same `completionResponse` and `completionDraftSnapshot` state as any other route
completion so the existing `commitRouteCompletionOnPlus` path handles both exact typed
routes and partial/navigated candidates. Once the draft is `@route+`, Bob's existing
task completion context should take over unchanged.

### 2. Cover the actual parse/completion disagreement and the full toggle handoff

Extend `Tests/Fixtures/fake-bob` with explicit leading-local-marker responses rather
than relying on its permissive catch-all:

- `capture-parse` for a leading `@fragment` must reproduce the real contract: literal
  task mode with no route need or span.
- `capture-complete` for that same draft must return route context, a replacement range
  after the sigil, and a project/area candidate.
- The resolved leading `@route+` draft must return the existing task context and a
  selectable block-ID candidate, so the transition can be tested rather than assumed.

Add focused `CapturePanelModelTests` that prove:

1. A bare `@` at byte zero produces route candidates with a zero-width replacement after
   the sigil.
2. Typing a leading fragment such as `@fi` keeps route completion visible and filters to
   the expected cached target despite a parse response with no completion metadata.
3. A cache miss still calls `bob capture-complete` with the complete draft and correct
   UTF-8 cursor, then applies Bob's replacement range.
4. Typing `+` with the leading route completion visible commits an exact or selected
   route to `@file+`, restores the caret after `+`, and opens task completion; selecting
   the returned task produces the expected `@file+<id>` marker without regressing the
   existing Add block ID path for tasks that lack IDs.
5. Nearby syntax remains isolated: `@@` uses the global-declaration path, separators and
   ordinary in-body `@route` markers retain their current contexts, and unrelated plain
   text does not start a completion subprocess.

Prefer assertions on the recorded fake-Bob argv as well as UI model state. They should
make it explicit when the target cache is authoritative and when the Bob completion
fallback is expected, preventing a passing test that only happens because the fake's
catch-all returned a route row.

### 3. Document and verify the editor contract

Update the README's inline-completion runtime contract to state that a single leading
`@` and its still-typing route fragment remain completion-active even before the draft
has enough syntax to be a routed capture. Connect that behavior explicitly to the
marker-only `@route+block-id` toggle flow and retain the documented distinction between
route completion and the task chooser opened by `+`.

Run the repository's Apple-toolchain checks on macOS 26:

```sh
just format-lint
just build
just test
just bundle
```

Then exercise the installed/development app from an empty editor: type `@`, continue
through a real project/area route fragment, choose or finish the route, type `+`, choose
an existing task ID, and confirm the resulting marker and toggle preview/action are
correct. Also spot-check an ordinary body-first `text @route` capture and an `@@route`
declaration to ensure their completion behavior is unchanged.

## Done when

- From an empty Bob Mac Capture editor, the first `@` immediately shows destination note
  candidates and the list remains active/filterable while the leading route is typed.
- Both accepting a route explicitly and typing `+` against the visible route completion
  lead into the existing task picker, yielding a valid `@<file>+<id>` marker and the
  expected task-toggle preview/action.
- Cached project/area/inbox targets remain authoritative when they match, while cache
  misses still fall back to Bob's completion response.
- Regression tests encode the real difference between `capture-parse` and
  `capture-complete` for a leading `@fragment`, cover the `+` handoff, and guard
  adjacent local/global completion contexts.
- Formatting, build, tests, and bundle verification pass with the selected macOS 26
  Apple toolchain.

## Out of scope

- Changing bob-cli's capture grammar or making a submitted marker-only `@file` route a
  capture by itself.
- Redesigning the existing task, Add block ID, Pomodoro, section, wikilink, or global
  declaration completion flows.
- Broadly invoking `capture-complete` on every editor change regardless of source or
  parse context.
