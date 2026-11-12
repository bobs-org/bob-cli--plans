---
tier: tale
size: medium
title: Prove the bank UI in real Obsidian, then land bob-cli-kd.6.1 and bob-cli-kd.6
goal:
  A real-Obsidian synthetic-vault harness proves every bank UI contract (including no
  math rendering), its evidence is an audited artifact, and bob-cli-kd.6.1 and its
  parent phase bob-cli-kd.6 are closed.
proposed_by: bbugyi200.athena.bob-cli-kd.6.1.land
proposed_with: claude/opus
bead: bob-cli-kd.6.1
status: done
---

- **PARENT:**
  [202611/bank_obsidian.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_obsidian.md)
- **BEAD:**
  [bob-cli-kd.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kd/bob-cli-kd.6.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-kd.6.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-kd.6.1.land.md)
- **COMMITS:**
  - [2f02824](https://github.com/bobs-org/bob-plugins/commit/2f02824e2858c01109ddb50501a66a9eda860ada)
    — fix(ledger-tools): wire bank cards and chip through bankApi()

# Prove the bank UI in real Obsidian, then land bob-cli-kd.6.1 and bob-cli-kd.6

## Why this tale exists

Epic **bob-cli-kd.6.1** ("Bank views and safe editing in Obsidian", plan
`plan:202611/bank_obsidian.md`) has all five phases closed, and its land agent verified
the source in bob-plugins: commits `8a1daca`, `c35662b`, `564f3c1`, `4e4a580`, and
`d2e4137`. One acceptance requirement was never met:

- **bank-ui-marks** required "a real Obsidian fixture case demonstrating no math
  rendering" and said to "verify the actual rendering pipeline, not just a tokenizer".
- **bank-ui-proof** required verifying every contract "using a disposable synthetic
  vault and the repository's existing Obsidian smoke/E2E harness". If native UI was
  unavailable, it had to record the unverified cases and "resolve that limitation before
  claiming complete UI proof". It also required publishing screenshot or text captures
  for every view as an **audited artifact**.

The proof phase (bob-cli-kd.6.1.5) ran only 34 _model-layer_ checks plus the generic
startup smoke. Its own transcript says "see unit gates for DOM/render coverage". No
`scripts/obsidian-bank-*-e2e.cjs` exists. Native UI does work on this host: `xvfb-run`
and `/snap/bin/obsidian` exist, and the repo has many real-Obsidian E2E harnesses such
as `scripts/obsidian-keep-moving-marks-e2e.cjs`, `obsidian-notebook-ui-e2e.cjs`, and
`obsidian-startup-smoke-lib.cjs`. The real-Obsidian proof is therefore remaining epic
work. This tale also performs the epic closeout.

### Already done by the land agent (do not redo)

The land agent's commit is on bob-plugins master. Pull before starting.

- **Epic-caused test hang, fixed.** `241p-plugin-bank-marks.js` created its 60 s
  config-poll interval through a `globalThis` fallback without the timer-host guard that
  the other bank timers use. That kept
  `scripts/test-ledger-tools-day-strip-plan-host.cjs` alive forever. It is now guarded,
  and the test passes 19/19 and exits in about 3 s.
- **Epic-caused docs overage, fixed.** `d2e4137` put `docs/bob-ledger-tools.md` at
  6012/6000 words. The bank paragraph was trimmed and `docs:check` lint passes again.
  **The page sits at the cap.** Do not add words to it. Put any new prose in a
  develop/validation doc instead.
- **Version and deployment.** bob-ledger-tools is at 1.112.1, and its manifest
  description now mentions bank row and day-heading decoration. It was deployed with
  `bob plugins sync` (no `-F`). At that point the vault showed 10/10 synced.
- **Follow-up triage is complete and recorded on the epic bead.** Do not file or
  re-triage the child proposals again:
  - New task **bob-cli-ks**: the picker-runtime test never exits, so full `npm test`
    hangs.
  - Corroborated **bob-cli-fz**, **bob-cli-ki**, **bob-cli-kb**, and **bob-cli-il**.
- `sase bead epic-symbols bob-cli-kd.6.1` and `... bob-cli-kd.6` both report no entries.

## Context to read first

1. `sase bead read bob-cli-kd.6.1 -r "<why>"`, every child phase bead, and the epic plan
   it names (`plan:202611/bank_obsidian.md`). Read the bank-ui-cards, bank-ui-gestures,
   bank-ui-marks, and bank-ui-proof contracts. They are the acceptance list.
2. Open bob-plugins with `/sase_repo` and read its `AGENTS.md` and
   `docs/develop/deployment.md`.
3. Use `/sase_memory_read` for `docs.md` before touching docs, and for
   `sase_artifacts.md` before publishing the artifact.
4. Pattern sources:
   - `scripts/obsidian-startup-smoke-lib.cjs` for the isolated scratch vault and
     profile, the owned Obsidian process, and CDP.
   - `scripts/obsidian-keep-moving-marks-e2e.cjs` with its `-lib` and `test-` siblings
     for the CLI shape (`-e/-o/-r/-s/-h`), record and report structure, screenshots, and
     the offline self-test.
   - The bank fixtures in `plugins/bob-ledger-tools/scripts/fixtures/bank/`, which
     include `math-rows.md`.

## Work

### 1. Real-Obsidian bank harness

Add `scripts/obsidian-bank-ui-e2e.cjs` and `scripts/obsidian-bank-ui-e2e-lib.cjs` (pure
assertion and fixture helpers). Add `scripts/test-obsidian-bank-ui-e2e.cjs`, an offline
`--self-test` unit, and wire it into the `test:unit` list exactly as the other
`test-obsidian-*-e2e.cjs` files are. Keep each hand-written file at or below about 1000
lines; split helpers into the lib as needed.

- **Vault and process.** Use an isolated scratch vault and profile, and never open
  `~/bob`. Run under `xvfb-run -a`. Install the built plugins from this checkout:
  bob-ledger-tools and task-status-cycler. A second pass runs task-status-cycler
  **without** ledger-tools.
- **Synthetic vault only.** It contains:
  - a `bank.md` hub with a `bob-bank` `view: queue` fence, plus a `bank:` frontmatter
    config with a checking account and declared categories;
  - two canonical months, `bank/2026-10.md` and `bank/2026-11.md`, with `bob-bank`
    `view: month` fences;
  - rows covering transfers, cash-out, inflows, bold and comma and Unicode-minus
    amounts, `⚑`/`!!` and `—`/`--` tails, links, imported `^bank-…` IDs, custom block
    IDs, ID-less rows, an empty real day heading, and two dollar amounts on nearby rows;
  - a canonical daily note for "today", one historical daily note (each with a
    `bob-plan` block), and an ordinary non-bank note with `#task` rows and `$x^2+y^2$`
    math.
- **bob.** Point the plugin at a small **recording shim** script. Discover how the bank
  service resolves `bob` (`apiStatsBobPath` / `statsVaultBasePath` in
  `241j-plugin-bank-data.js`) and use PATH or the plugin's configured bob path,
  whichever the existing resolver honors. The shim logs every argv to a file the harness
  reads. By default it execs the real `bob` (the installed bob-cli, or one built from
  bob-cli master) against the scratch vault, so list, report, ensure, and ack are a real
  round trip on synthetic data. A control file switches it into failure modes: missing
  binary, a sleep past the timeout, `schema_version: 2`, malformed JSON, and an error
  envelope. If the real `bob bank` cannot run against a scratch vault on this host,
  serve the existing synthetic fixture JSON from the shim instead. State that choice in
  the report.

### 2. Walkthrough assertions (plan bank-ui-proof items 1–7)

Each case below is recorded as a pass/fail entry, and both Live Preview and Reading view
are covered where the contract names both views.

1. **Cards.**
   - The month card shows the title plus `so far`; account status and caveat;
     Out/In/Net; waiting and flagged counts; category bars in report order with
     unclassified amounts shown explicitly; and `destination unknown` for cash.
   - The heat strip runs Monday-first, outlines today, and marks waiting days. Clicking
     a day lands on that real heading. Clicking a day with no heading never lands on
     line 1 or frontmatter.
   - Footer links go to the existing previous and next months and the hub, with no
     dangling link.
   - The queue card lists days oldest first, has line links and per-row checkboxes, and
     each day button carries exactly its drawn IDs. It shows `nothing to acknowledge`
     with the caveats kept, plus the import command and at most six recent months.
   - Repeat at a narrow pane width with a screenshot. Show an invalid fence error and
     the empty-month teaching state.
2. **Daily chip.**
   - Today's chip shows `out · in` plus the all-month waiting count. The historical chip
     has no waiting count.
   - Clicking the chip runs `bob bank ensure -D <date> -f json` (shim argv) and opens
     the returned path at the returned row, converting from 1-based to 0-based. It must
     not open a second leaf.
   - The chip is absent once `bank.md` is removed, appears exactly once across
     rerenders, still works with the Day strip disabled, and the
     `stats.bank_chip: false` setting hides it.
3. **Marks.**
   - Rows and day headings are decorated with amount column, green `+` inflows,
     category, account, and flag chips, `bank ✓` only on imported IDs, a waiting bar,
     and dimmed acknowledged rows. Unknown labels stay literal.
   - Real headings show report Out/In and the waiting badge. An empty real heading shows
     `nothing posted`. A missing day gets no heading decoration.
   - The cursor line reveals raw syntax. Source mode stays raw.
   - `toggle-bank-marks` restores raw content in the main window and in a popout window.
   - Links still work. The note bytes are unchanged on disk.
   - The `bank` property row is hidden only on `bank.md`.
4. **Typing and math.**
   - Enter at a complete row end inserts `- [x] $`. Enter on an empty stub clears it.
     Ordinary Enter is unchanged in code, nested children, non-row checklists, and
     mid-line.
   - `[cat:: ` suggestions follow the JSON order and insert only the key.
   - **No `.math`/MathJax element** appears for bank-row dollars in either view, even
     with marks toggled off. The ordinary note's `$x^2+y^2$` still renders as math.
5. **Acknowledgment.**
   - Single-row and bulk ack submit exactly the drawn IDs: shim argv is
     `bank ack <ids…> -f json`, never `-D`, never empty, and deduplicated. A double
     click sends one call.
   - A row appended to the note after the card drew stays waiting.
   - An ID-less checkbox flips only its own original row's `[ ]` → `[x]` byte. A changed
     or moved snapshot is refused.
   - A failed ack (shim error) stays visibly failed.
6. **Guards.**
   - On a bank row, the cycler cycle/complete commands, counted and Vim paths, and the
     ledger-tools task gestures attach no `✅`/`#task`/freshness/status metadata. Only
     `[ ]` and `[x]` toggle.
   - Ordinary `#task` rows in the non-bank note still cycle normally.
   - Repeat the bank-row guard with ledger-tools disabled.
7. **Data honesty.**
   - A metadata change to the month note refreshes the cards without a reload.
   - A simulated local date rollover and a window focus revalidate.
   - Timeout, missing bob, schema mismatch, and malformed JSON each show an error or
     stale state, never `$0.00` success.
   - Closing the view and unloading the plugin repaint nothing and leave no duplicate
     chip.

Where a case cannot be driven through CDP (for example a native popout window under
xvfb), record it as **unverified**, with the reason, in the report. Never record it as
passed. Then try the nearest faithful alternative, such as the `window-open` mirror path
the other mark-gate E2Es use.

### 3. Fix what the walkthrough finds

Any bank-UI defect the harness exposes is epic work. Fix it in the bank source fragments
(`241h`–`241q`, `192-bank-row-guard.js`) with a focused unit test. Run `npm run build`,
bump the bob-ledger-tools (and, if touched, task-status-cycler) manifest patch or minor
version, and run `npm run docs:gen`. Never hand-edit generated `main.js`. Keep every
fragment at or below 1000 lines. Do not add words to `docs/bob-ledger-tools.md`, because
it is at its 6000-word cap. If the harness deserves a docs mention, add one line where
the other E2E harnesses are documented (search `docs/develop/`).

### 4. Gates, deployment, and evidence

- Run these in bob-plugins: `npm run build`, `npm run build:check`, `npm run docs:gen`,
  `npm run validate`, the focused bank units
  (`node --test scripts/test-ledger-tools-bank-*.cjs scripts/test-task-status-cycler-bank-guard.cjs scripts/test-ledger-tools-day-strip-plan-host.cjs scripts/test-obsidian-bank-ui-e2e.cjs`),
  the new harness under `xvfb-run -a` with `-o <dir> -r <report.json>`, and the isolated
  startup smoke.
- Run `npm run docs:check` with `BOB_CLI_DIR` set to the bob-cli checkout. The only
  acceptable failure is the known strict-Zensical missing page
  `develop/plugins/validation.md:55 → ../bob-vim-yank-flash.md#verification`, which
  bob-cli-kb owns.
- **Do not** use the full `npm test` as a gate. It never exits because of bob-cli-ks.
- Use `/sase_monitor` for any long command.
- Deploy with `bob plugins sync`, following AGENTS.md. Never pass `-F`. On a refusal,
  follow `docs/develop/deployment.md` "When sync refuses an overwrite". Confirm with
  `bob plugins list` that the result is 10 synced, 0 drift.
- Publish the harness JSON report, the screenshots (every view, including narrow,
  popout, and Reading/Live Preview pairs), the shim argv log, and a short
  known-limitations text. Publish them as an **audited artifact** with
  `sase artifact create ... --bead bob-cli-kd.6.1`, following `sase_artifacts.md`. Only
  synthetic data may appear. Record the artifact ref in a
  `sase bead note bob-cli-kd.6.1`.

### 5. Closeout (final step, same turn as the final code)

1. Run `sase bead epic-symbols bob-cli-kd.6.1`. Resolve every listed `--epic-symbol`
   entry by wiring it up, privatizing it, adding a non-test pragma, or deleting it. Only
   when a still-open later bead needs an exemption, re-key that Justfile line to the
   open bead. (None existed at planning time.)
2. Close the epic with `sase bead close bob-cli-kd.6.1 --note "<verification>"`. The
   note should cover:
   - the phase commits;
   - the land agent's fixes;
   - the real-Obsidian harness result, with pass, fail, and unverified counts;
   - the artifact ref;
   - the gates run and the deployment state;
   - follow-up outcomes, by pointing to the epic's LANDING TRIAGE note.

   Never use `--force` merely to make the close succeed. If it is rejected for
   unfinished phases or leftover symbols, fix the cause and close again.

3. Run `just symvision` in the bob-cli checkout and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the EPIC PLAN
   path printed by `sase bead read bob-cli-kd.6.1`, `plan:202611/bank_obsidian.md` in
   the plans repo.
5. Handle the parent. bob-cli-kd.6.1's `parent_bead` is the **phase bead bob-cli-kd.6**
   ("Obsidian bank cards, styled rows, and the daily chip").
   - Confirm that its description is fully delivered by this child epic: the `bob-bank`
     month and queue cards, styled rows and day headings, the daily chip, the open-today
     and acknowledge commands, Enter continuation with `- [x] $`, category suggestions,
     bank-row guards, and `bob plugins sync`.
   - Run `sase bead epic-symbols bob-cli-kd.6`, then
     `sase bead close bob-cli-kd.6 --note "<what you verified>"`.
   - **Do not** close the containing epic **bob-cli-kd** or the phase **bob-cli-kd.7**
     (bank-wrapup). bob-cli-kd's own land agent owns those.
6. Finish through `/sase_final`, which commits the bob-plugins changes and the plan file
   edit.
