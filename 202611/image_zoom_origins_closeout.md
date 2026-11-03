---
tier: tale
size: medium
title: Finish image origin round trips and land bob-cli-gn
goal:
  Repair the four remaining image-origin defects, verify and deploy the integrated
  feature, and close bob-cli-gn.
proposed_by: bbugyi200.athena.bob-cli-gn.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gn
create_time: 2026-11-03 18:20:00
status: wip
---

- **PARENT:**
  [202611/image_zoom_origins.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/image_zoom_origins.md)
- **BEAD:**
  [bob-cli-gn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gn/README.md)

# Finish image origin round trips and land bob-cli-gn

This is the remaining work from the landing audit of epic `bob-cli-gn`. One coding agent
can implement it directly. This tale has no separate land agent: its final step is to
close the original epic and mark its original plan done in the same turn as the repairs,
before the host commits. Do not wait for this work's commit SHA, push, or CI result to
do that closeout.

## Fixed decisions and scope

- Original approved plan: `plan:202611/image_zoom_origins.md`.
- `copy_form = tilde`: `y` copies `origin.display`; restore also uses that display.
- `decision_record = no`: edit no SASE memory files.
- Bob owns path classification, file copying, and the origin journal. Plugins own the
  undoable editor replacement. Keep capture JSON byte-compatible; no version bump, new
  CLI option, deletion of retained vault images, or Mac Capture work.
- Open `bob-plugins` through `/sase_repo`, read its `AGENTS.md`, edit source fragments,
  and rebuild generated entrypoints. Use `/sase_memory_read` for docs guidance before
  changing docs. Use `/sase_repo` to open any other repository.

## Audit already performed

The lander read the original epic, all three closed children and every note, the
approved plan, CLI commit `0b4cebdf`, and plugin commits `1a2b572` and `7ad20f5`. It
reviewed journal serialization/resolution, import/save/rollback, capture append callers,
the additive attachments API, viewer chrome/actions, and path imports. The decisions are
honored. Existing-task-link captures carry no clip plan; the other inbox-route commit
caller is test-only. History attachment dedupe and the non-dry-run origin append remain
wired through the common capture entry point.

Audit baselines are bob-cli `a7c4b07f` and bob-plugins `7ad20f5`. Non-epic changes since
`0b4cebdf` include capture note-pull conformance (`093dcead`), project moves
(`b8688d9e`, `6650faa3`), target runs (`d1f34842`, `a7c4b07f`), task transplant
(`8d2eb09f`, `aac3947c`), and periods (`56d8dac3`). They keep the shared clip
commit/recording path; no additional attachment writer was introduced. The plugin
periods landing `945d9fc` adds `api.periodNotes` beside the new attachments API and
fixes the unregistered midnight timer. Preserve those additions when changing lifecycle
code. Review any newer commits before the final close.

Focused evidence: `file:explicit:2a36a78e87df1326774cd894`. The six image zoom and
attachments API files pass 109 tests. This does not prove the missing backend wrapper
case or cancellation case below. The navigation suite now exits normally, 23/23 passing.
The epic-symbol audit currently reports no entries, and `bob-cli-gn` has no parent bead.
Recheck both at closeout.

The authorized `gh:bobs-org/bob` checkout also confirms the tracked
`!/.bob/attachments/*.jsonl` allowlist, `merge=union` attribute, and the updated
image-path import comment above `nnoremap Z :bob_zoom_image<CR>`. No vault edit was
needed during the audit. Dry-run sync of each relevant plugin is allowed without force,
but would update its `main.js`; the final repaired build still needs deployment.

## 1. Make angle-wrapped image paths work through Bob

`findImagePathBullet('- </tmp/x.png>')` returns the wrapper-inclusive raw path and
passes it unchanged to `attachments.import`. But
`src/native/capture_clip/files.rs::strip_matching_quotes` recognizes only single and
double quotes. For a real file, `bob attachments import -f json -- '<PATH>'` returns
`not_absolute`, while the unwrapped path succeeds. Existing plugin tests mock a
successful import and miss this contract mismatch.

Extend the shared Bob classifier to strip one matching `<...>` wrapper as well as the
existing quotes, before the existing file-URI decode/tilde expansion. Keep the plugin
sending the raw wrapper-inclusive content and replacing that entire span; do not
introduce a second filesystem parser in JavaScript. Preserve interior path bytes,
spaces, case, and the existing URI decoding rules.

Add meaningful regression coverage in the existing attachment classifier/CLI tests:
angle-wrapped absolute paths with spaces, angle-wrapped file URIs, successful origin
source/display, and reuse. Keep the quoted and unwrapped cases passing. Check `%c`
through the same shared classifier and preserve its output shape. The backend test must
fail against the current implementation, not just use a successful mock. Keep the plugin
path detection/import/restore tests passing.

## 2. Cancel source actions when their viewer closes

`207a-image-zoom-origin.js::restoreImageZoomSource` awaits the origin and then edits
without checking `imageZoomClosed`. Reproduction using the existing navigation harness:
open a viewer with a deferred origin Promise, start `r`, close the viewer, resolve
`{ok:true, origin:{source:'/tmp/x.png', display:'~/x.png', state:'present'}}`. On the
unchanged current tree, the already-closed viewer performs one editor transaction and
replaces the embed. View/file/line guards all pass because the same note remains active.
`copyImageZoomSource` has the same deferred-action gap.

Guard action entry and completion after awaits against the closed viewer. A pending `r`
must not edit or raise a restoration notice after closing; a `y` still waiting for its
origin must not issue a clipboard write after closing. If an already-started clipboard
write finishes after close, do not recreate toast timers or UI. Preserve normal
pre-image-load actions, tilde copy, state refusals, the view/file/line guards, one
editor transaction, cursor restoration, and retained vault files.

Extend `scripts/test-navigation-image-zoom-origin.cjs` with deferred-Promise regressions
for `r` then close, `y` then close, and clipboard completion after close. Assert zero
late editor writes/notices/clipboard calls as appropriate and no rearmed toast timers.
Retain a successful deferred-origin control while open.

## 3. Repair the epic's two verification regressions

- In `src/native/catalog/tests.rs::document_shape` and
  `tests/cli/catalog.rs::protocol_1_parses_with_schema_version_one`, update stale
  command counts from 67 to 69 at the audit baseline (account for any subsequently added
  commands). Also verify the import and origin entries are present with the expected
  command paths. Retain existing period-command coverage. These failures are caused by
  `0b4cebdf`, as the epic's original note reports.
- Slim `plugins/bob-ledger-tools/src/170-plugin-lifecycle.js` enough to satisfy the
  existing fragment-length check without weakening it or removing either API. Before
  `1a2b572`, `split('\n').length` was 1000; adding the attachments namespace made
  it 1001. The intervening periods change preserves that count. The failure proposed as
  unrelated by `bob-cli-gn.3` note #5 is actually this epic's work. Run the existing
  `test-ledger-tools-task-link-marks-live.cjs` regression (currently 36 pass/1 fail) and
  plugin build checks.

## 4. Verify the integrated feature and deploy

Run `just check` for bob-cli. Do not run `just check-full`. Read any verification
failure rather than accepting old phase claims; repair epic-caused failures here. Run
the focused Rust attachments/catalog tests as useful during development, then the
canonical gate once the changes are final. Use `/sase_monitor` if a long gate needs a
continuation; that continuation still owns this tale's final closeout.

In bob-plugins run `npm run build`, the six
`test-navigation-image-zoom-{target,command,viewer,origin,path}.cjs` and
`test-ledger-tools-attachments-api.cjs` suites, the lifecycle regression, `npm test`,
and `npm run docs:check`. Known unrelated failures below remain separate; compare exact
failing nodes and record any new epic-caused issue. Keep generated entrypoints current.
If wrapper/cancellation prose needs updating, change only the relevant contract/docs and
run the corresponding docs generation/check commands.

Deploy with `bob plugins sync --no-pull --repo <opened-bob-plugins-path>` and verify
both relevant plugins. Inspect any protected-deploy refusal rather than forcing over
unknown work; the earlier epic note reports a dirty deployment conflict. Verify the
original phase's tracked vault journal allowlist/union attribute and the `Z` vimrc
comment through the authorized vault checkout/workflow, using `/sase_repo` and the
Obsidian memory. Keep vault commits owned by `bob vault-sync`.

## 5. Finish the landing of bob-cli-gn

The lander has already triaged every proposal; use its `FOLLOW-UP TRIAGE` epic note as
the durable record. Include these outcomes in the final close note:

- `gn.1 #1`: duplicate `note_pull: None` is already removed by `e74f4f08`; declined a
  new task because current source and the epic's own parent commit already have only one
  initializer per literal.
- `gn.2 #1`: routed phase CPU-budget evidence to existing `bob-cli-f8`; landing's
  isolated current run passes 5/5, which is recorded without claiming a fresh miss.
- `gn.3 #1`: independently reproduced and corroborated `bob-cli-f7` (Keep timezone).
- `gn.3 #2`: independently reproduced and corroborated `bob-cli-et` (freshness date).
- `gn.3 #3`: independently reproduced and corroborated `bob-cli-ff` (scheduling date).
- `gn.3 #4` and `#7`: independently reproduced and consolidated on `bob-cli-eh`
  (single-slot layout-ready fakes), one root cause, not two new tasks.
- `gn.3 #5`: retained as epic work, fixed in step 3; not the unrelated `bob-cli-ej`
  performance flake.
- `gn.3 #6`: declined a new task: `945d9fc` removes the unregistered interval and the
  unchanged navigation file now passes 23/23 in about 3.2 seconds.

Re-read `bob-cli-gn` and all three children with audited `sase bead read` calls, confirm
all descendants are closed and every note has an outcome, and review post-audit drift in
both repositories. Read the original plan through
`sase artifact read plan:202611/image_zoom_origins.md "Recheck original scope before closing gn"`.

Run `sase bead epic-symbols bob-cli-gn`. Resolve every entry by wiring, privatizing,
adding a legitimate non-test pragma, or deleting per Symvision policy; re-key only when
a still-open later bead truly needs the exemption. Close normally with
`sase bead close bob-cli-gn --note "<verified scope, decisions, integration, fixes, test results, deployment, and every triage outcome above>"`.
If entries remain, clean them and retry. If a named phase is incomplete, finish/reopen
it; never use `--force` merely to advance a successful landing. A genuinely
canceled/superseded outcome must be deliberate and carry its resolution and reason.

After closing, run `just symvision` if the repository provides that recipe and record
the result (the audited current bob-cli justfile has no such recipe). Open the plans
sidecar via `/sase_repo` and set `status: done` in the frontmatter of its
`202611/image_zoom_origins.md`, the PLAN path reported by the epic. Then run
`sase bead read bob-cli-gn -r "Need the parent link"`; this epic has no parent at the
audit baseline, so no ancestor is expected to need closing. If that link has changed,
follow the root landing prompt's parent-phase/parent-plan checks rather than forcing any
ancestor closed. Finish through `/sase_final`, committing every changed repository
including plans. Do not stop after coding and wait for another lander: this step is this
tale's responsibility.
