---
tier: tale
title: Finish image-zoom target handling and land bob-cli-a1
goal: Preserve image target identity, complete missing resolver regressions, deploy
  the repair, and close bob-cli-a1.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a1
status: done
---

- **PARENT:**
  [202610/z_image_zoom.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/z_image_zoom.md)
- **BEAD:**
  [bob-cli-a1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a1/README.md)

# Finish image-zoom target handling and land bob-cli-a1

## Objective and authority

Finish the remaining resolver defects from the landing audit of epic `bob-cli-a1`, then
complete that epic's landing in this same coding turn. This is one bounded
implementation; it needs no child epic or separate land agent. The coder's commit
happens after its turn ends, so closeout must not wait for this repair's commit SHA,
push, or CI.

Read `sase bead read bob-cli-a1 -r "Need current landing notes and parent link"` and the
accepted plan through
`sase artifact read plan:202610/z_image_zoom.md "Need image zoom contract"`. The landing
audit found no `parent_bead` on bob-cli-a1.

The accepted epic decision is `image_zoom_record=no`, automatically selected without
human review. Its skipped memory proposal has already been triaged as `bob-cli-a3`; this
repair implements the existing behavior contract.

## Audit already completed

The lander read every child and every note: `bob-cli-a1.1`, `.2`, and `.3` are closed,
with three notes each. It reviewed the actual fragments, tests, CSS, registration, docs,
generated integration, and these bob-plugins commits:

- `7c6d037`: image-target resolver and fixture tests.
- `5e21b03`: owned modal, geometry, keys, cleanup, CSS and lifecycle tests.
- `049f466`: command, editor restoration, documentation and manifest 2.47.0.

All 67 existing image-zoom tests pass, and plugin validation passes.
`npm run docs:check` also passes, including 431 documentation tests with four
optional-dependency skips. The existing image tests miss the resolver cases below. Full
`npm test` produced 3278 passes and seven known failures: four unblocked-glyph cases
(`bob-cli-6v`), the Keep notice (`bob-cli-6w`), the reverse-conversion BLOCKS link
(`bob-cli-7i`), and the 16ms ranker flake (`bob-cli-3w`). Evidence is
`file:explicit:c8d89bc75d79562000f004b3`. The epic does not change the affected
implementations or tests. The ranker file passed 23/23 when rerun alone on the unchanged
tree.

The lander finished follow-up triage before proposing this tale:

- Each phase's note #1 is the same skipped memory proposal; one small ready memory task,
  `bob-cli-a3`, now owns it.
- Each phase's note #2 reports the three existing CI issues above; independent evidence
  was added to `bob-cli-6v`, `bob-cli-6w`, and `bob-cli-7i`.
- `.1` and `.2` additionally report the existing ranker flake; evidence was added to
  `bob-cli-3w`.
- No proposal was declined, and no duplicate task was created. The unrelated optional
  research ideas remain deferred as the accepted plan specifies.

These outcomes are also in the epic's `LAND FOLLOW-UP TRIAGE` note; carry them into the
final close note instead of filing them again.

Integration review: no unrelated bob-plugins commits landed after the first epic commit.
Since epic creation, bob-cli `0e9f915a` adds triple-at capture-template rewriting and
`ae9629ec` adds task-relation documentation; neither uses or conflicts with image zoom.
The vault's `10e8c4f4` changes only `obsidian_vimrc.md`, adding exactly one
`exmap bob_zoom_image` and one `nmap Z :bob_zoom_image<CR>`. The inspected vault
checkout is clean and matches origin/master. The scoped plugin sync dry-run reports all
three deployed files unchanged. The Mac UI has not been tested.

`sase bead epic-symbols bob-cli-a1` returned no entries. Neither inspected repo
currently has a `symvision` recipe. Recheck both facts during closeout.

## Remaining defects and exact expected results

The following were reproduced on clean bob-plugins `049f466` through the existing Node
harness. Evidence: `file:explicit:494ab4dc0d1aa3c9db0ea86f`.

1. **Remote URLs are decoded before loading.**
   `![](https://e.com/a%252Fb.png?sig=%252B%20x)` currently returns
   `src: "https://e.com/a%2Fb.png?sig=%2B x"`. T5 requires the original URL bytes,
   including its query and fragment, so this changes the resource and can break signed
   URLs.
2. **Escaped Markdown parentheses are not unescaped.** `![](img/a\(1\).png)` sends
   `img/a\(1\).png` to metadata resolution rather than `img/a(1).png`, failing even when
   the latter exists. T1 explicitly supports escaped parentheses.
3. **Vault percent escapes are decoded twice.** `![](img/a%2520b.png)` first becomes
   `img/a%20b.png`, then `resolveLinkTargetFile` normalizes it again and looks up
   `img/a b.png`. It must resolve the filename `img/a%20b.png`; retain normal `%20` →
   space and UTF-8 decoding for ordinary vault image paths.
4. **The extensionless-URL exception is too broad.** `![[https://e.com/page]]` is
   currently accepted because it has `embed=true`. T2 allows extensionless HTTP(S) only
   for Markdown image syntax `![...](url)`. Wiki references and plain Markdown links
   need an allowed image extension.
5. **URL display titles keep reserved escapes.** `![](https://e.com/a%23b.png)` produces
   title `a%23b.png`. T5 requires the decoded final pathname segment, `a#b.png`, while
   `src` must remain unchanged. Use a safe component decode for the title, with a
   malformed-escape fallback and hostname fallback when there is no segment.

The image-zoom docs also say, without qualification, that a non-image link under the
cursor shows a notice. T6 applies that notice only when there are no image candidates on
the line; otherwise the image pick rule still applies.

## Implementation

1. Open bob-plugins with `/sase_repo` and use only the printed path. Read its
   `AGENTS.md`. Work in `plugins/bob-navigation-hotkeys/src/679-plugin-image-zoom.js`
   and the existing `scripts/test-navigation-image-zoom-target.cjs`. Read the shared
   helpers in `680-plugin-link-parse.js` before choosing the smallest change.
2. Separate extracting Markdown syntax from URI decoding. Preserve an HTTP(S)
   destination exactly after removing its Markdown delimiters/title. Normalize vault
   targets once, including Markdown punctuation escapes, and retain the source note path
   for resolution. Keep raw and normalized targets distinct where needed so calling
   `resolveLinkTargetFile` does not double-decode. Prefer a zoom-local adaptation; if a
   shared helper needs a backward-compatible option, preserve its existing behavior for
   every other caller and test that seam.
3. Apply the exact T2 extensionless rule using both `form` and `embed`, and fix URL
   title decoding separately from the load URL. Preserve the existing supported schemes,
   image extensions, notices, inline/fenced-code exclusion, cursor pick rule, and
   no-read/no-write behavior. Do not broaden targeting beyond the line.
4. Add regression fixtures for all five cases. Include remote URLs with percent escapes
   in pathname/query/fragment, angle destinations and titles, malformed escapes,
   extensionless Markdown images versus wiki/plain links, escaped and balanced
   parentheses, ordinary encoded spaces/Unicode, and a vault fixture containing both
   `a%20b.png` and `a b.png` so incorrect double decoding cannot accidentally pass.
   Assert exact metadata lookups/source paths and exact `src` values; keep throwing
   vault/editor write stubs. Use the existing test file, already registered in
   `package.json`.
5. Use `/sase_memory_read` to read `docs.md` before editing `docs/image-zoom.md`.
   Clarify that the non-image notice applies when no image candidates exist on the line.
   This is the only necessary prose correction. Regenerate `main.js` with
   `npm run build`; never edit it by hand.

## Verification, integration and deployment

1. Recheck commits on the current branches and their remote base branches since the
   audit SHAs above. Integrate any newly landed code that actually conflicts with image
   zoom, and preserve unrelated work.
2. Run the three existing image-zoom test files together, then `npm test`,
   `npm run validate`, `npm run docs:check`, and `git diff --check` in bob-plugins.
   Every new regression and existing image-zoom test must pass. Distinguish the
   already-triaged clean-base failures from new failures; do not mark a red full suite
   green or make unrelated CI repairs part of this tale. Run `just check` in the primary
   bob-cli checkout as the file-change verification gate. Do not run `just check-full`.
3. Deploy the changed plugin with
   `bob plugins sync --repo <opened-bob-plugins-path> --no-pull --plugin bob-navigation-hotkeys --dry-run`,
   then the same command without `--dry-run`, and verify it reports up to date. Do not
   use `--force` to override a dirty destination.
4. Verify the existing vimrc mapping remains unique. For repository inspection, open
   `gh:bbugyi200/bob` with `/sase_repo` and use that printed checkout; the known mapping
   commit is `10e8c4f4`. A second mapping or fresh vault edit is unnecessary.
5. Provide Bryan the Mac smoke checklist in the final report, explicitly saying the
   agent host cannot claim a UI pass: reload Bob Navigation Hotkeys and vimrc; open both
   an embed and plain image link with `Z`; try `1`, `+/-`, `0`, `hjkl`, drag,
   wheel/pinch and double-click; verify `Esc`, `q`, `Ctrl+[`, `Z` and `ZZ` return to the
   same cursor/scroll/normal mode; verify `x`/`dd` write nothing, no-image lines only
   notify, Insert/Visual behavior stays intact, pop-out/Source mode and the palette with
   Vim off work, and the vault diff remains unchanged.

## Final step: close bob-cli-a1 in this turn

After the repair and verification are complete:

1. Read the epic and every current descendant again, including their notes and any
   linked child plan. Verify all required work is complete and carry the audit and
   follow-up outcomes above into the close note. If this tale has its own assigned bead
   beneath bob-cli-a1, finish and close that bead normally once its work is verified so
   it does not remain an unfinished descendant.
2. Run `sase bead epic-symbols bob-cli-a1`. Resolve each entry by wiring, privatizing,
   applying a justified non-test pragma, or deleting it under the Symvision policy.
   Re-key an exemption only when a specifically identified still-open later bead needs
   it. Do not leave any entry keyed to an epic/phase that is closing.
3. Run
   `sase bead close bob-cli-a1 --note "<actual verification and integration results, repaired resolver cases, test results including known failures, deployment/mapping evidence, and all follow-up outcomes>"`.
   If it rejects leftover symbols, clean them up and retry. If it names unfinished
   phases, finish or reopen them; never use `--force` to make successful nested landing
   advance. Cancellation/supersession is allowed only for a deliberate documented
   outcome with the required resolution/reason, not to bypass work.
4. After closing, run `just symvision` wherever that recipe is available; otherwise
   explicitly record its absence. Open the plans repository with `/sase_repo`, resolve
   `plan:202610/z_image_zoom.md`, and set `status: done` in that plan's YAML frontmatter
   at the `202610/z_image_zoom.md` path inside the opened plans repository. Preserve its
   other metadata and body.
5. Read `sase bead read bob-cli-a1 -r "Need the parent link"` after closing. This epic
   has no parent as of the audit, so normal completion is expected. If that link has
   changed, follow the lander's parent rules: close only a fully satisfied phase parent
   normally; for a plan parent recheck its previous landing note, all descendants/notes,
   linked plans, readiness and post-child drift before retiring symbols, closing,
   running available Symvision and marking its plan done. Stop and note a blocker at the
   first incomplete or ambiguous parent.
6. Include every edited repository in `/sase_final`, including bob-plugins and the plans
   sidecar. Do not wait for this tale's own commit, push or CI before closing.
