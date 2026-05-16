# Per-task Effort + Read-only — Smoke Tests

Tick-able checklist for the Phase A + Phase B work landed on
`feature/task-effort-and-readonly` (task effort + provider override +
read-only investigation-only runs).

> Setup: `amaco init` in a project, two providers configured (e.g.
> `claude` + `codex`), and the dashboard running via `amaco ui`. For
> `effortMap` checks add an `effortMap:` block under your top-level
> config in `project.yml`.

---

## 0. Automated baseline

- [ ] `pnpm typecheck` exits 0.
- [ ] `pnpm test` reports **576/576** across **65** files (current green on this branch).
- [ ] `pnpm build` succeeds; eager UI bundle still **< 500 kB**.
- [ ] `node dist/index.js run --help` lists `--effort`, `--provider`, `--read-only`.
- [ ] `node dist/index.js tasks add --help` lists the same three flags.

---

## 1. Pure resolver (effort + provider override)

Trust the 7 unit tests in `tests/effort-resolver.test.ts`, or spot-check:

- [ ] `pnpm test tests/effort-resolver.test.ts` — 7/7 green.
- [ ] No-override → `agentDefault` (each agent uses its configured provider).
- [ ] Explicit `providerOverride` beats any `effort` setting.
- [ ] `providerOverride` to an unknown provider id → falls back to `agentDefault`, note names the bad id.
- [ ] Valid `effort: low` + `effortMap.low = haiku` → resolves to `haiku`.
- [ ] `effort` set but `effortMap` empty → fallback + honest note.
- [ ] `effort` set but `effortMap` points at an unknown id → fallback + honest note.

---

## 2. CLI for effort + provider

- [ ] `amaco run "tiny task" --effort low` starts a run; `state.json` carries `effort: "low"`. If `effortMap.low` is set, `resolvedProviderId` matches; otherwise `null` and a `policy.warning` row in `events.ndjson` explains the fallback.
- [ ] `amaco run "tiny task" --provider codex` starts a run; `state.providerOverride === "codex"`, `state.resolvedProviderId === "codex"`. Agent invocations use `codex` regardless of `agents.<id>.provider`.
- [ ] `amaco run "tiny task" --effort low --provider codex` → `providerOverride` wins, `resolvedProviderId === "codex"`.
- [ ] `amaco run "x" --effort huge` rejects with exit 2 and a clear message ("must be one of low|medium|high").
- [ ] `amaco tasks add "..." --effort low --provider codex --read-only` creates a task carrying all three fields. Verify with `amaco tasks show <id> --json`.
- [ ] `amaco run "..." --task <id>` inherits effort/provider/read-only from the task when the CLI doesn't pass its own flags.
- [ ] When the CLI passes `--effort high`, it overrides what the task carries.

---

## 3. Dashboard — effort + provider

- [ ] Open the task detail page. The new "effort / provider override / read-only" panel renders below the priority/risk pills.
- [ ] Change effort from "— none —" to "low" → task updates immediately; reload shows the new value persisted.
- [ ] Type a provider id, blur the input → PATCH fires and the chip appears.
- [ ] Clear the provider input, blur → PATCH sets it to null.
- [ ] An unknown provider id is accepted by the PATCH (validation happens at run start, not at PATCH time). When you later `amaco run --task <id>`, the run logs the honest fallback warning.
- [ ] Run a task that has effort/provider set. The run header shows the effort chip (`Zap` icon) and the resolved-provider chip (`Cpu` icon, accent tint).
- [ ] A run with `state.providerOverride` set but no resolution (e.g. typo) shows the effort chip but no resolved-provider chip. The replay tab carries the `policy.warning` row with the explanation.

---

## 4. CLI for read-only

- [ ] `amaco run "audit the login flow" --read-only` starts a run. `state.readOnly === true`. `events.ndjson` carries the "Read-only run: …" `policy.warning` line at startup.
- [ ] The run stages stay within: created → planning → planned → architecting → architected → reviewing → merge_ready / blocked. No `executing`, no `validating`, no `fixing`, no `verifying`. Confirm with `amaco replay <runId>`.
- [ ] No files in the worktree are modified during the run. `git status` in the run worktree before and after is unchanged (empty).
- [ ] Reviewer reads `plan + architecture` priors only (no execution artifact).
- [ ] A `CHANGES_REQUESTED` review on a read-only run flips to `BLOCKED` (no fix loop possible).
- [ ] An `APPROVED` read-only run lands at `merge_ready` with the run-completion event carrying `readOnly: true`.

---

## 5. HTTP routes — read-only refusal

In the dashboard or via curl against a known read-only run:

- [ ] `POST /api/runs/<id>/suggestions/<sid>/apply` returns **409** with the message ending in "Apply / validate / revert / approve are disabled. Start a non-read-only run …"
- [ ] `POST /api/runs/<id>/suggestions/<sid>/validate` returns **409**.
- [ ] `POST /api/runs/<id>/suggestions/<sid>/revert` returns **409**.
- [ ] `POST /api/runs/<id>/suggestion-bundles/<bid>/apply` returns **409**.
- [ ] `POST /api/runs/<id>/suggestion-bundles/<bid>/smart-apply` returns **409**.
- [ ] `POST /api/runs/<id>/suggestion-bundles/<bid>/validate` returns **409**.
- [ ] `POST /api/runs/<id>/suggestion-bundles/<bid>/revert` returns **409**.
- [ ] `POST /api/runs/<id>/suggestions/<sid>/approve` and `…/reject` still work (metadata only, no worktree mutation).
- [ ] `PATCH /api/runs/<id>/suggestions/<sid>/profile` still works (metadata only).
- [ ] `GET /api/runs/<id>/suggestions` still works (read-only inspection allowed).

---

## 6. Dashboard — read-only

- [ ] On a read-only run, RunHeader shows a bright **READ-ONLY** badge (`Eye` icon, warn tint).
- [ ] On the same run, the Suggestions tab shows the suggestion rows BUT the per-row action buttons (Approve / Reject / Apply / Validate / Revert) are replaced by a single chip "read-only run — actions disabled".
- [ ] Cross-link "Replay" buttons (replay-finish phase) still work on read-only runs.
- [ ] On a non-read-only run, the badge is absent and the actions render normally (regression guard).
- [ ] Replay tab on a read-only run shows the abbreviated phase list (planning / architecting / reviewing only; no executing / validating / verifying rows).

---

## 7. Permission profile override

- [ ] Open `.amaco/project.yml` and confirm `permissions.profiles.readOnly` exists (Amaco's default templates ship it). Read-only runs depend on this profile.
- [ ] Start a read-only run that asks the planner to do something that would normally write (e.g. "create a new file …"). The planner's prompt is built under `readOnly` permissions; the planner's output should be a recommendation only.
- [ ] If `permissions.profiles.readOnly` is absent from the project, the run fails fast at the planner stage with a clear "no profile named 'readOnly'" error — not a silent permission downgrade.

---

## 7b. Heuristic effort auto-detect (Phase C)

The classifier is pure and free — same input always returns the same output.
Trust the 11 unit tests in `tests/effort-heuristic.test.ts`, or spot-check:

- [ ] `pnpm test tests/effort-heuristic.test.ts` — 11/11 green.

CLI surface:

- [ ] `amaco tasks add "fix a typo in the README" --files README.md` prints, after the "✓ Task added" header, a line: `effort: (none) — suggested low @ 1; pass --auto-effort or --effort low to apply` plus up to three reason bullets ("Short task …", "Low-effort keyword: typo.", "All targeted files are docs …").
- [ ] `amaco tasks add "..." --effort low` shows `(matches suggestion @ N)` when the heuristic agrees.
- [ ] `amaco tasks add "refactor scheduler architecture" --files src/scheduler/scheduler.ts --effort low` shows `(suggested high @ N)` — the user's explicit choice still wins; the line is honest about the disagreement.
- [ ] `amaco tasks add "..." --auto-effort` (no `--effort`) applies the heuristic verdict; the saved task carries it.
- [ ] `amaco run "fix typo"` (no `--task`) prints the same verdict line before kickoff.
- [ ] `amaco run "fix typo" --auto-effort` runs with the heuristic-picked effort; `state.json` shows it.
- [ ] An LLM is **never** called. Verify with `grep provider.started .amaco/runs/<runId>/events.ndjson` — heuristic output happens before any provider invocation.

Server route:

- [ ] `curl -X POST http://localhost:4317/api/effort/classify -H 'Content-Type: application/json' -d '{"text":"refactor","files":[]}'` returns `{ effort: "high", confidence: <n>, reasons: […] }`.
- [ ] Same input → same output across multiple POSTs (determinism).
- [ ] An empty body returns `{ effort: "medium", confidence: 0.1, reasons: ["Empty task — defaulting to medium."] }` — never throws.

Dashboard:

- [ ] Open a task whose effort is unset (or set to a value the heuristic disagrees with). The effort/provider/read-only panel shows a soft-accent banner: "heuristic suggests X @ N" with an **apply** button and an expandable "why?" listing the reasons verbatim.
- [ ] Click **apply** — the dropdown immediately reflects the new value; banner disappears; a quieter "✓ effort matches the heuristic suggestion @ N" hint replaces it.
- [ ] Change the task's title or touchedFiles (via CLI in a side terminal) — re-poll picks up the new heuristic verdict on the next interval; banner re-renders if the verdict now disagrees with the saved effort.
- [ ] When effort matches the heuristic, the green ✓ hint is present (regression guard so the suggestion-source-of-truth is visible).

Posture:

- [ ] Classifier never calls a provider. Never reads `.env` or any file outside the task text + the provided file paths (it only looks at the paths' extensions, not their contents).
- [ ] CLI verdict line still prints when running offline / with no providers configured (heuristic is local).

---

## 8. Effort × read-only interactions

- [ ] A task with both `effort: low` and `readOnly: true` runs as read-only AND with the resolved low-effort provider on every agent.
- [ ] `amaco run "..." --effort low --read-only` similarly carries both.
- [ ] The README's "Pause and resume" still works on a read-only run — it pauses between stages and resumes as before.

---

## 9. Posture / safety

- [ ] Pause/resume + effort + read-only never call any provider on metadata writes. Spot-check `events.ndjson` for the targeted run: only the resolver's `policy.warning`, no `provider.started`.
- [ ] Cross-tab navigation from the task detail page back to a run preserves effort/read-only state in the URL chips.
- [ ] No new file paths read or written beyond `state.json` + `events.ndjson` + `.amaco/roadmap/tasks/<id>.json`.
- [ ] `assertSafeRunId` still fires on path traversal in the new routes (covered by the existing route-security tests).

---

## 10. After all boxes are ticked

- [ ] `docs/TODO.md` Near-term polish entries for Phase A and Phase B flip from `[~]` to `[x]` once merged; a Shipped Phases entry lands at merge time.
- [ ] Fast-forward merge `feature/task-effort-and-readonly` into `main`, push, close out.
