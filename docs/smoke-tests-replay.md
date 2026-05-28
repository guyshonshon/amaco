# Run Replay — Smoke Tests

A single, tick-able checklist for everything the Run Replay concept ships
across both branches (`feature/run-replay-ui` → already on `main`, and
`feature/replay-filter-search` → currently 2 commits ahead).

Render this file in any markdown viewer that supports GFM task lists
(GitHub, VS Code preview, Obsidian) and tick boxes as you go.

> All steps assume:
>
> - a project initialized with `vibestrate init` (`.vibestrate/` present),
> - at least one finished run under `.vibestrate/runs/`,
> - the dashboard started with `vibestrate ui` (or `pnpm dev` in dev),
> - the latest commit of `feature/replay-filter-search` checked out.
>
> Whenever you see `<runId>`, use a real id from `.vibestrate/runs/`.

---

## 0. Automated baseline (no human action required)

These run from the repo root and must all be green before the manual
section starts. If you trust the last CI / local run, leave them ticked.

- [ ] `pnpm typecheck` exits 0 (no errors in either `tsconfig.json` or `tsconfig.ui.json`).
- [ ] `pnpm test` reports **550/550** across **62** files (last green count on `feature/replay-filter-search`).
- [ ] `pnpm build` succeeds. Eager UI bundle is **< 500 kB** (current: ~483 kB / gzip ~135 kB). ReplayPanel async chunk renders as its own file (`ReplayPanel-*.js`, ~17 kB).
- [ ] `node dist/index.js --help` lists `replay` among the commands.

---

## 1. CLI: `vibestrate replay <runId>`

- [ ] `vibestrate replay <runId>` prints a `Run …` header line, status, branch, worktree, event count.
- [ ] Phase section is present when the run has events; numbers per phase look plausible.
- [ ] Approvals / Suggestions / Bundles / Policy refusals / Notifications / Terminal sessions sections appear only when non-empty.
- [ ] Terminal sessions section ends with the "Metadata only — Vibestrate never persists terminal stdout/stderr." dim note.
- [ ] Runtime metrics section shows duration, provider calls, review loops; files-changed / +/- line if available.
- [ ] If the run has truncation, the count line shows the yellow `(truncated from N)` suffix.
- [ ] If the run is missing optional files, a yellow `Skipped N file(s)…` block lists them with reasons.
- [ ] Last line is the dim hint pointing at `#/runs/<runId>?tab=replay`.
- [ ] `vibestrate replay <runId> --json` emits parseable JSON; piping to `jq '.events | length'` returns a number.
- [ ] `vibestrate replay does-not-exist` prints `Run not found: …` in red and exits with code **1** (`echo $?`).
- [ ] `vibestrate replay <runId>` does **not** touch the worktree or run any provider (no new commits, no `.env` reads, no shell exec beyond the projection's reads). Spot-check with `git status` in the run worktree before / after.

---

## 2. Dashboard — Replay tab basics

- [ ] Open the dashboard, open a run, click the **Replay** tab. Left column shows phase groups; right column shows the latest event selected by default.
- [ ] Phases with zero events do not appear.
- [ ] Click a different event row. Right column updates with timestamp, source (`event` / `synthetic`), phase label, type, message, optional referenced artifacts, and an expandable `event data` JSON block.
- [ ] Snapshot line ("run state at this timestamp: …") appears for events that have a preceding `state.changed`.
- [ ] If `events.ndjson` was capped at 10 000, a yellow truncation banner appears above the timeline with the honest "Showing the most recent 10000 of N events" copy.
- [ ] If any optional file was missing/malformed, the collapsible `N file(s) skipped while building replay — click for details` appears and lists them.
- [ ] Lazy chunk: open DevTools → Network with cache disabled → switch to Replay tab. A single `ReplayPanel-*.js` request fires only the first time you open the tab.

---

## 3. Deep-link URL forms

Paste each URL into the address bar (replace `<runId>` first). The browser
must NOT reload the page if you were already on the dashboard.

- [ ] `#/runs/<runId>?tab=replay` — opens the run on the Replay tab.
- [ ] `#/runs/<runId>?tab=replay&replayEvent=0` — selects event 0; the matching row is highlighted; the left column scrolls so the row is in view.
- [ ] `#/runs/<runId>?tab=replay&replayEvent=999999` — shows the warn banner *"Couldn't locate event #999999 in this run's timeline. The row may have been truncated, or the link points at a different run."*
- [ ] `#/runs/<runId>?tab=replay&replayPhase=verifying` — expands the verifying phase (if collapsed) and selects its first event.
- [ ] `#/runs/<runId>?tab=replay&replayPhase=invented` — falls back to default selection; no warn banner (invalid phases are dropped silently at parse time — by design).
- [ ] `#/runs/<runId>?tab=replay&replayMatch=suggestion:<existing-suggestion-id>` — selects the originating `suggestion.*` row.
- [ ] `#/runs/<runId>?tab=replay&replayMatch=approval:<existing-approval-id>` — selects the originating `approval.*` row.
- [ ] `#/runs/<runId>?tab=replay&replayMatch=notification:<existing-notification-id>` — selects the synthetic `notification.created` row.
- [ ] `#/runs/<runId>?tab=replay&replayMatch=suggestion:does-not-exist` — warn banner *"Couldn't locate suggestion does-not-exist …"*.
- [ ] `#/runs/<runId>?tab=fictional` — the `tab` query is dropped (whitelist), Diff tab loads.
- [ ] Reload the page on any of the deep-link URLs. Same focus + same warn banner state.

---

## 4. Cross-links from other panels

- [ ] **Run list:** the All Runs table has a rightmost column with a small **Replay** button per row. Clicking it opens the run on the Replay tab without going through Diff first. Clicking the row itself still opens Diff (the button's `stopPropagation` works).
- [ ] **Suggestions tab:** each suggestion row has a **Replay** button on the right end of the action bar. Clicking it switches to Replay and highlights the row with the matching `data.id`. The active suggestion id appears in the address bar as `?replayMatch=suggestion:<id>`.
- [ ] **Approvals tab:** each approval row's `id: <approvalId>` footer has a right-aligned **Replay** button. Clicking it switches to Replay focused on the matching `approval.*` event. Address bar updates with `?replayMatch=approval:<id>`.
- [ ] **Notifications drawer:** for any notification whose `runId` matches the currently-open run, the row has a **Replay** button next to Resolve. Clicking it navigates to that run's Replay tab with `?replayMatch=notification:<id>`.
- [ ] Notifications without a `runId` (project-wide notifications) do **not** show a Replay button.
- [ ] Cross-link from inside the same run page: tab switches without a full reload, no spinner flash on the Replay panel re-render.

---

## 5. Filter + search

- [ ] Open Replay on a run with many events. Type a substring (e.g. `validation`) into the filter input. The visible/total count at the right shrinks; phases with zero matches disappear; the per-phase headers show `visibleN/totalN`.
- [ ] Clear with the **Clear** button — count returns to total, all phases reappear.
- [ ] Click a phase chip (e.g. `Approvals`). Only that phase shows. Click again — chip deselects, all phases reappear.
- [ ] Multi-select two phase chips. Both phases are visible; others are hidden.
- [ ] Search + chips compose. Search `approval` with the `Approvals` chip on returns only `approval.*` events; with `Notifications` on returns only the notification row whose message contains "Approval resolved" (the message-search regression test).
- [ ] Type a search that matches nothing → the left column shows "No events match the filter."; right column keeps the previously-selected event (so the deep-link target isn't lost).
- [ ] Whitespace-only search behaves like an empty filter (no rows hidden).
- [ ] Open a `?replayMatch=…` deep-link, then add a filter that excludes the matched row. The right pane keeps showing the selected event; the row is just not in the left column. Clear the filter — the row re-appears, still selected.

---

## 6. Permalink

- [ ] Select any event. Click **Permalink** on the right-side card. The button briefly flips to **Copied** (with the green check) for ~1.5s.
- [ ] Paste the URL into a new browser window. The same run, same tab, same event are restored.
- [ ] The URL contains `?tab=replay&replayEvent=<n>` and uses the same hash routing the rest of the dashboard uses.
- [ ] Permalink in a restricted context (e.g. Firefox without HTTPS, or DevTools "Clipboard write" denied): the fallback `window.prompt()` opens with the URL pre-filled so the user can copy manually. **No silent failure.**

---

## 7. Keyboard scrubber

Inside the Replay tab, with focus *outside* the search input:

- [ ] `↓` or `j` selects the next visible event; the row scrolls into view if needed.
- [ ] `↑` or `k` selects the previous visible event.
- [ ] `Home` jumps to the first visible event.
- [ ] `End` jumps to the last visible event.
- [ ] At the boundaries (first / last visible event), pressing the boundary key is a no-op (no wrap, no console error).
- [ ] With a filter active, keys only step through visible rows (not the hidden ones).
- [ ] Click into the filter search input. Type letters that include `j` / `k` — they appear in the input, the selected event does **not** change.
- [ ] Press `Ctrl+ArrowDown` (or `Cmd+ArrowDown`) — selection does not change (modifier keys bypass the handler so OS / browser shortcuts still work).
- [ ] Press `Tab` to focus the search input, then `Tab` away. `j` / `k` resume stepping. (Make sure no element keeps an invisible focus that swallows the keys.)

---

## 8. Edge cases / honesty

- [ ] Open Replay on a brand-new run that has only one or two events — the panel still loads, no errors, default selection lands on the last event.
- [ ] Open Replay on a `blocked` or `aborted` run — phase grouping still renders; the `Other` bucket may carry the lion's share, which is fine.
- [ ] Open Replay on a run whose `.vibestrate/runs/<id>/runtime-metrics.json` does **not** exist (an old run). The "Runtime metrics" summary card is empty / absent; the projection still renders; the file is listed under "Skipped N file(s)…" with `reason: not present`.
- [ ] If a run has a terminal session: the synthetic `terminal.session.opened` / `closed` rows appear in the timeline; the summary card lists the session id, cwd, exit code. **No** stdout/stderr field anywhere. Click `event data` → only metadata fields.
- [ ] If a run has notifications: synthetic `notification.created` rows appear with `id` in the `data` block (open the JSON expander to verify).
- [ ] Project-wide notifications (other run's `runId`) are filtered out — they do **not** appear in this run's Replay even if they sit in the same `.vibestrate/notifications/notifications.json`.

---

## 9. Posture / safety

These are sanity checks; none should ever pass against a malicious link.

- [ ] No HTTP `POST` / `PUT` / `DELETE` to any endpoint while you click around the Replay tab — verify in DevTools → Network. The only Replay request is `GET /api/runs/<runId>/replay`.
- [ ] No "Apply", "Approve", "Validate", "Revert", "Abort", or "Rerun" button appears anywhere inside the Replay panel.
- [ ] The Replay panel never reads `.env` content (open a run worktree that has a `.env` file with secrets; the value never appears in the timeline, in the right-side card, or in the JSON of any event).
- [ ] `GET /api/runs/../../etc/passwd/replay` returns 400 (path traversal refused at the route guard).
- [ ] `GET /api/runs/unknown-id/replay` returns 404 with the `RunReplayError` shape.
- [ ] `vibestrate replay <runId>` and `vibestrate replay <runId> --json` do not touch `git`, do not run validation, do not call providers (check `git reflog` / process tree if in doubt).
- [ ] Permalink URL contains only the runId and the event index — no event payload, no project path, no secret leaks. Inspect any generated permalink before sharing.

---

## 10. After all boxes are ticked

- [ ] Open `docs/TODO.md`; the Near-term polish entry for "Run Replay concept finish" can flip from `[~]` to `[x]` once the branch is merged. A Shipped Phases entry will be added at that point (commits `cc2dec0`, `8b19f8e`).
- [ ] Decide: fast-forward merge `feature/replay-filter-search` into `main`, push, and close out.
