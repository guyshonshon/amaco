# Pause / Resume — Smoke Tests

Tick-able checklist for the pause/resume phase landed on
`feature/replay-filter-search` (`pauseRequested` + `pausedAtStatus`,
orchestrator pause gates, `amaco pause` / `amaco resume`,
`POST /api/runs/:id/{pause,resume}`, RunHeader buttons).

> Setup: an `amaco init`-ed project, a real long-running provider
> configured (so you can pause between stages and see the round-trip),
> and the dashboard running via `amaco ui`.

---

## 0. Automated baseline

- [ ] `pnpm typecheck` exits 0.
- [ ] `pnpm test` reports **561/561** across **63** files (last green on `feature/replay-filter-search`).
- [ ] `pnpm build` succeeds; eager UI bundle still < 500 kB.
- [ ] `node dist/index.js --help` lists `pause` and `resume`.

---

## 1. State machine + pure helpers

These are the 11 unit tests in `tests/pause-service.test.ts`. Trust the
green test suite, or re-run just this file:

- [ ] `pnpm test tests/pause-service.test.ts` — 11/11 green.

Spot-check what the assertions actually cover:

- [ ] requestPause sets `pauseRequested=true`, doesn't change status, emits `run.pause_requested`.
- [ ] requestPause is idempotent (a second call doesn't double-emit).
- [ ] requestPause refuses a terminal run (`merge_ready` / `blocked` / `failed` / `aborted`) with PauseError.
- [ ] requestPause refuses an already-paused run.
- [ ] requestResume clears `pauseRequested` and emits `run.resume_requested`.
- [ ] requestResume on a still-running run with `pauseRequested=true` cancels the pending pause cleanly.
- [ ] requestResume refuses on a non-paused, non-pending-pause run.
- [ ] applyPauseIfRequested is a no-op when `pauseRequested=false`.
- [ ] applyPauseIfRequested transitions to `paused` (records `pausedAtStatus`), polls until the flag clears, then rounds back to `pausedAtStatus`. Both `run.paused` and `run.resumed` events appear in the log.
- [ ] applyPauseIfRequested observes an external abort during pause and returns the terminal state (orchestrator exits cleanly).
- [ ] applyPauseIfRequested clears an orphaned `pauseRequested` if the run reached terminal between the request and the polling tick.

---

## 2. CLI

- [ ] `amaco pause <runId>` on an in-flight run prints "Pause requested for …" and exits 0. Re-running it is idempotent — same exit, no error.
- [ ] `amaco resume <runId>` on a paused run prints "Resume requested for …" and exits 0.
- [ ] `amaco resume <runId>` on a still-running run with a pending pause request also exits 0 (cancels the pending pause).
- [ ] `amaco pause` on a terminal run prints "Run is in terminal state … pause has no effect." and exits 2.
- [ ] `amaco pause` on a non-existent run prints "Run not found: …" and exits 1.
- [ ] `amaco resume` on a non-paused, non-pending run prints "Run is not paused and has no pending pause request …" and exits 2.
- [ ] After `amaco pause <runId>`, `events.ndjson` for that run grows by exactly one `run.pause_requested` row (verify with `tail -n 1`).
- [ ] No git operation, no provider call, no worktree write happens as a side effect of pause/resume — `git status` / file mtimes in the worktree are unchanged.

---

## 3. HTTP routes

In the dashboard or via `curl`:

- [ ] `POST /api/runs/<runId>/pause` on an in-flight run returns `{ run: <state with pauseRequested=true> }`.
- [ ] `POST /api/runs/<runId>/pause` on a terminal run returns **409** with the message "Run is in terminal state …; pause has no effect."
- [ ] `POST /api/runs/<runId>/pause` on an already-paused run returns **409**.
- [ ] `POST /api/runs/<runId>/resume` on a paused run returns `{ run: <state with pauseRequested=false> }`. The orchestrator's pause gate will round-trip the status back to `pausedAtStatus` on its next poll tick.
- [ ] Path-traversal attempt `POST /api/runs/../etc/passwd/pause` returns **400** (the `assertSafeRunId` guard fires).
- [ ] Unknown run id returns **404**.
- [ ] No new GET endpoints were added — pause/resume is purely write-side.

---

## 4. Orchestrator round-trip (the real test)

Start an actual run with a non-trivial provider (so stages take long enough to observe pause):

```bash
amaco run "make a tiny script that prints hello"
```

In a second terminal:

- [ ] `amaco pause <runId>` while the run is in `planning`. The orchestrator's progress prints in terminal #1 continue *until the current stage finishes*, then stop. The run's status transitions to `paused` and `events.ndjson` records `run.paused`.
- [ ] `state.json` shows `pausedAtStatus: "planned"` (or whichever stage was about to start).
- [ ] `amaco resume <runId>`. The orchestrator picks up within ~1.5s (the default poll interval). Status transitions back to `pausedAtStatus`, `events.ndjson` records `run.resumed`, and the next stage runs.
- [ ] Pause then abort (`amaco abort <runId>` while paused). The orchestrator observes the terminal state and exits cleanly with the right final report (no `run.resumed` row written after the abort).
- [ ] Pause before the run starts: `amaco run …` in one terminal, immediately `amaco pause <runId>` (or use `--task` and pre-stage the pause). The very first pause gate fires and the run pauses at `created` without running any agent.

---

## 5. Dashboard

- [ ] On a run that's actively in `planning` / `executing` / etc., the run header shows a **Pause** button. Click it. The status badge briefly remains the same (the pause is queued), the page renders a small `pause queued` chip, and the button flips to **Cancel pause**.
- [ ] When the orchestrator picks up the pause (next stage boundary), the status badge flips to `paused` (warn tint) and the button reads **Resume**.
- [ ] **Resume** click → status badge returns to the pre-pause stage. Buttons revert to **Pause**.
- [ ] **Cancel pause** while the pause was still queued → `pause queued` chip disappears, the badge stays on the current stage, buttons revert to **Pause**.
- [ ] On a terminal run (`merge_ready` / `blocked` / `failed` / `aborted`) neither button is shown.
- [ ] Network tab confirms exactly one POST per click (`/api/runs/<runId>/pause` or `…/resume`) — no GETs polling.
- [ ] Error in flight (server down, route 409): the header surfaces the error message inline below the badges; the button stops spinning.

---

## 6. Replay sees the events

After a pause/resume round-trip, open the Replay tab for the run:

- [ ] `run.pause_requested` row appears in the timeline (synthetic / "other" phase since it doesn't match an existing prefix).
- [ ] `run.paused` and `run.resumed` rows appear in chronological order.
- [ ] The state-snapshot card on the right reflects the right status at each row's timestamp (paused row → `status: paused`, post-resume → `status: planned` / whatever).

---

## 7. Posture / safety

- [ ] Pause/resume never call any provider, never spawn a shell, never write to the run worktree. Add a fake provider and confirm zero `provider.started` events are emitted by a pause/resume cycle.
- [ ] No new file paths read or written by these surfaces beyond `state.json` and `events.ndjson` for the targeted run.
- [ ] The pause/resume routes refuse path traversal (covered in §3).
- [ ] Pause/resume have no effect on `waiting_for_approval` — a pause request while waiting on approval is *queued* (the flag is set) and only fires after the approval resolves and the orchestrator hits the next non-approval stage boundary.

---

## 8. After all boxes are ticked

- [ ] `docs/TODO.md` already flipped both the Near-term polish entry and the Larger-scope deliberately-deferred entry to `[x]`. A Shipped Phases line gets added at merge time.
- [ ] Decide: fast-forward merge `feature/replay-filter-search` into `main` (it now bundles replay-finish + pause/resume), push, and close out.
