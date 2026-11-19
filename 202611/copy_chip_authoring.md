---
tier: tale
title: Copy-chip authoring card and Alt+Click editing
goal:
  Complete bob-cli-ni.5 with exact previewed copy-chip creation, conversion, and editing
  in one undo step.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ni.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-ni.5
create_time: 2026-11-19 12:50:37
status: wip
---

- **PARENT:**
  [202611/copy_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/copy_chips.md)
- **BEAD:**
  [bob-cli-ni.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ni/bob-cli-ni.5.md)

# Copy-chip authoring card and Alt+Click editing

## Scope and starting evidence

Complete only phase `author`, bead `bob-cli-ni.5`, of `plan:202611/copy_chips.md`. Add
the **Make or edit copy chip** editor command, default **Mod+Shift+Y**, and
**Alt+Click** editing in Live Preview. The card must create, convert, or edit literal
copy payloads with an exact live preview and one undoable editor write.

This is one bounded implementation in `bob-plugins`; use a tale, size medium. The
assigned phase has no previous implementation or remainder notes. Its prerequisites
already landed in the linked repo:

- `0b38f84`: grammar, serializer, vectors, and plugin registration.
- `76fa3f9`: inline renderers, Live Preview gestures, cursor API, feedback.
- `0a4a91f`: the navigation Copy card's C row.
- `febfc6b`: fence renderer and fence-aware cursor resolution, plugin v0.3.0.

At planning time the linked checkout is clean at `febfc6b`; no author command, modal, or
Alt+Click route exists. Forty-one targeted tests pass, covering grammar, round trips,
cursor resolution, fences, Live Preview, and the offline e2e wrapper. The fence phase
notes confirm deployment of 0.3.0. Neither standard Obsidian executable path is present
on this host; runtime Obsidian verification is still an explicit acceptance item, not
something the offline self-test proves. An SSH preflight confirmed that `athena` is
reachable, `/snap/bin/obsidian` is executable, Xvfb is installed, and Node 20.19.2 has a
CDP WebSocket implementation through the harness's supported fallback. Use that host for
the runtime cases.

Before implementation, re-read the assigned bead and open `bob-plugins` using
`/sase_repo`; use the path it returns and read its `AGENTS.md`. Recheck its current
history and dirty state so intervening phase work is preserved. Read the epic and
`research:202611/obsidian_copy_chips/obsidian_copy_chips.md` through
`sase artifact read`. The research checkout needed refreshing during planning; use
`/sase_repo` if its ref initially resolves missing.

The epic's decisions are final: no decision record may be written; only the
`glossary:copy-chip` memory addition is authorized, and that belongs to the adopt phase.
This tale edits no memory. Do not change bob-cli code, capture grammar, Bob Mac Capture,
plugin allowlists, the vault vimrc, or GTD tasks. Do not redo the Copy card C row. Honor
`decisions:q-and-ctrl-bracket-mirror-esc`; payloads always come from source, never
rendered text.

## 1. Resolve an immutable editing target

Add focused source fragments, for example `081-author-model.js`, `082-author-modal.js`,
and `083-author-command.js`, in `src/fragments.json` after the fence renderer and before
the plugin class. Keep each fragment under 1000 lines. Import Modal and any
editor/history APIs defensively in `010-load.js`; the existing pure helpers must remain
loadable without Obsidian.

The pure author model should return either a refusal or a snapshot containing the
original source, ordered selection/range, source bytes at that range, label/text, form
context, indentation, and initial focus. Bind it to the originating editor and note
identity at the runtime boundary. Resolve in this order:

1. More than one selection: refuse with **One selection at a time** and no card.
2. A nonempty selection: use its literal text, focus Label, and replace exactly that
   range. Support reversed and Vim visual selections.
3. With an empty selection, use the existing `copyChipAt` resolver and its shared cursor
   rule. A ready result opens **Edit copy chip** with its label/text. An ambiguous
   result refuses with **N chips on this line · move onto one**. Do not silently insert
   or fall back to the clipboard in that case.
4. Otherwise, a plain inline code span containing the cursor preloads its
   CommonMark-normalized text and focuses Label. Reuse `copyChipFindCodeSpans`,
   `copyChipNormalizeSpan`, and existing exclusions, rather than inventing a backtick
   regex that disagrees with the grammar. Do not convert apparent spans inside fenced
   code, links, frontmatter, math, or comments.
5. Otherwise insert at the captured cursor. Read the clipboard once, best effort, into
   Text and select it; unavailable clipboard reads leave an empty editable Text field. A
   late read must never overwrite typing or reopen a closed card.

For inline chips, `chipAt` offsets include delimiters. For block chips, its current
`line/from/to` cover the cursor line for flashing, **not the fence replacement range**.
Use `copyChipFindFenceBlock` to capture the opening through closing line, or through EOF
for an unclosed fence. Preserve source indentation characters and payload whitespace. Do
not change the public version-1 cursor API just to expose the author model.

## 2. Plan and apply one exact replacement

Keep replacement planning pure. Call the existing `copyChipSerialize` for the actual
payload and show the exact final replacement text in the Markdown preview. Use
table-aware serialization for a table-cell inline chip. Preserve labels, backticks,
pipes, Unicode, tabs, trailing spaces, and trailing LF according to the existing
grammar; the existing CRLF-to-LF normalization remains authoritative.

For multiline payloads, generate a `bob-copy` fence. If the target starts or ends
mid-line, include the required separating line breaks in the same replacement, leaving
the unselected prefix and suffix intact. Under list items, indent the opening, body, and
closing fence to the item's content column using the line's existing indentation
characters. Account for unordered, ordered, task, nested, and continuation lines without
stripping literal payload whitespace or duplicating indentation. Pin examples with exact
before/after text. Switching a fence to an inline payload replaces the entire old fence
and keeps surrounding structure. Never emit a multiline fence into a table cell in a way
that silently corrupts the row; an unsupported placement must produce a visible refusal
and zero writes.

At save, verify the originating note/editor identity and re-read the captured
source/range. Reject changed source with **The note changed — reopen the card**. A full
original-buffer comparison is an acceptable conservative guard and also protects empty
insertion ranges and shifted targets. A closed/detached editor or note switch must not
redirect the write to another document.

Apply the change and collapsed cursor immediately after it in one editor transaction.
Ensure the undo history separates this operation from preceding typing, using the
available CM6 history boundary where required. Never use `vault.modify`, whole-note
`setValue`, or multiple replacement writes. Guard duplicate Enter/hotkey events so one
save cannot run twice. After a successful visual-mode write, exit Vim visual mode
through the existing adapter pattern and restore the intended cursor; do not alter Vim
state while canceling the card.

## 3. Build the modal and command lifecycle

Register `make-or-edit`, **Make or edit copy chip**, as an `editorCallback` with
`{ modifiers: ["Mod", "Shift"], key: "y" }`. Keep at most one author card per plugin
instance; invoking the gesture again saves the current card. Clean up the modal, event
listeners, and outstanding clipboard work on close/unload.

Use theme-variable styling consistent with `.bob-copy-card*` in
`bob-navigation-hotkeys/styles.css`, without introducing a runtime dependency on that
plugin. Render:

- A one-line Label input, placeholder **optional — shown instead of the text**.
- An auto-growing monospace Text textarea that preserves whitespace.
- A live noninteractive preview using `copyChipBuildInline` or `copyChipBuildFence`,
  plus **inline chip** or **block card · N lines**.
- A scrollable, unelided Markdown row containing the exact replacement bytes.
- Inline serializer validation, including **nothing to copy**, **labels are one line**,
  and clear whitespace-only feedback. Invalid drafts cannot save.
- Footer: **⏎ save · ⇧⏎ newline · Tab next field · Esc cancel**.

Assign label, payload, and Markdown as text/value properties only; do not put payloads
in HTML strings or DOM attributes.

The inline builder already accepts `{interactive:false}`. The fence builder currently
creates focusable Copy/fold buttons; extend it with an inert preview option whose
default preserves existing rendered cards, and test that preview controls cannot copy,
steal focus, or mutate the note.

Use the modal's key scope with one shared dismissal route. Enter in either field saves;
Shift+Enter in Text inserts a newline and updates the form; Tab/Shift+Tab cycle the two
fields; Mod+Shift+Y saves. Esc, Ctrl+[ (Ctrl alone), and q outside text controls dismiss
through the same route; q inside the fields types normally. Do not inherit the existing
Copy card's blanket key consumption, which would swallow text input. Respect composition
events. Use the originating window's document for DOM/focus and release scope handlers
on close.

## 4. Route Alt+Click to that exact chip

Extend the existing Live Preview widget gesture in `050-live-preview.js`. Capture the
modifier with the gesture and, on an eligible Alt+primary release, open the same author
card for the clicked widget's source range. Resolve the editor via the widget's owning
CM6 view/editor info, not whichever editor happens to be active. Verify the current
range still represents that chip before opening it. Repeated identical chips must edit
the clicked occurrence.

An Alt gesture must not also execute the existing mouseup copy path. Preserve ordinary
click/tap and Enter/Space copying, movement thresholds, and drag/cancel behavior. Append
**· Alt+Click to edit** to the Live Preview tooltip only. Reading view and preview DOM
remain on their existing behavior. No hover pencil or new gesture is added to fence body
selection; fence editing is available via the editor command.

## 5. Verification and documentation

Add meaningful unit coverage in registered `test:unit` scripts, e.g.
`scripts/test-copy-chips-author-model.cjs` and
`scripts/test-copy-chips-author-modal.cjs`, extending the existing fake editor, Modal,
clipboard, and DOM harnesses as needed. Cover:

- Selection/context precedence, multiple selections, ambiguous lines, ordinary span
  exclusions, fence range versus flash range, and UTF-16 offsets.
- Exact replacement snapshots for list indentation, mid-line boundaries, table pipes,
  backticks, one-line/multiline transitions, and trailing whitespace.
- Zero-write invalid/stale cases, including insertion points, edits before the target,
  note/editor switches, double saves, and late clipboard completion.
- Initial focus and selection, live previews, validation, inert fence preview, dismissal
  aliases, q typing, composition, field traversal, and repeat hotkey.
- Alt+Click opens the exact occurrence and never copies, while ordinary gesture behavior
  remains unchanged.

Extend `scripts/obsidian-copy-chips-e2e.cjs` and its fixture/self-test support,
splitting author scenarios into a helper if needed. Use real CDP keyboard/mouse events
in the harness's isolated scratch vault. Include one-line selection, multiline list
selection, editing inline and block chips, plain-span conversion, clipboard prefill,
backticks, table cells, cancel keys, q typing, one Mod+Z restoring original bytes
without undoing earlier typing, Vim visual-mode save, Alt+Click, and second Mod+Shift+Y
save. Use a separate authoring note or explicit write assertions: the existing
render/copy scenarios assert that their seeded note stays byte-identical and must retain
that check. Read actual clipboard bytes for authored results. A unit counter for one
transaction does not prove actual editor undo behavior.

Add an Authoring section to `docs/bob-copy-chips.md`, covering typing, the card,
conversion, context precedence, editing and keys, preview/validation, exact save, and
stale refusal. Remove the stale future-phase wording for features already shipped
(including the Copy card C row), leaving adoption work accurately scoped. Bump only
`bob-copy-chips` to the next feature minor, ordinarily `0.4.0` from the current `0.3.0`;
preserve newer intervening changes. Regenerate `main.js` and docs inventory from source,
never hand-edit generated outputs.

Run `npm run build`, `npm run docs:gen`, the focused new/existing chip tests,
`npm test`, `npm run validate`, and `npm run docs:check`. Run the real harness under
`xvfb-run -a node scripts/obsidian-copy-chips-e2e.cjs` with report/screenshots outside
the repo. This host lacks Obsidian; read the `tailnet.md` reference before using SSH
alias `athena`, the Linux verification host named in the deployment docs. Transfer the
exact built plugin and test-harness snapshot into a unique scratch directory there and
run under Xvfb with an isolated vault/profile. Record hashes of the tested plugin files
with the report so a remote run proves this build. Do not edit or deploy from an
unrelated remote checkout. Use `/sase_monitor` for long verification commands and wait
for the staging command itself to exit. If that host is unavailable, report the specific
prerequisite failure and use an explicit SASE continuation to finish runtime
verification before claiming the phase complete; offline results are not an Obsidian
pass. For failures outside this change, reproduce on the clean base before treating them
as existing issues, and record the evidence as the user's requested
`PROPOSED FOLLOW-UP:` rather than leaving the phase open for a base failure.

## 6. Deployment and phase completion

Run `bob plugins sync` against the returned linked checkout with `--no-pull` (and
`-p bob-copy-chips` to scope deployment). Follow `AGENTS.md` and
`docs/develop/deployment.md` if another checkout's deployment is protected: fetch and
reconcile landed work, never use `-F` without authorization. Verify the plugin's
version/enabled/synced row, and distinguish deployed bytes from an Obsidian runtime
smoke pass. Use `/sase_final` for the host commit declaration covering every edited
repository; do not manually commit merely because the epic mentions committing. If
commit/deploy ordering needs a host handoff, use the appropriate SASE monitor/finalizer
workflow instead of leaving it as an untracked promise. Tell Bryan to reload Obsidian
after the verified sync.

Before closing, run `sase bead epic-symbols bob-cli-ni.5` again; there were no entries
during planning. Resolve any newly introduced entries, or re-key only legitimate
remaining work to a still-open parent/later phase. Close **only** `bob-cli-ni.5` with
`sase bead close bob-cli-ni.5 --note "<implementation and verification evidence, including any runtime limitation>"`.
Never change its status by hand or close an ancestor. Discovered unrelated work is
recorded only as `PROPOSED FOLLOW-UP:` notes on this phase, not new beads.
