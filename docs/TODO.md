# Amaco Implementation TODO

A running, spec-aligned checklist. Top-level sections mirror
[`amaco-claude-code-implementation-spec.md`](./amaco-claude-code-implementation-spec.md)
and [`amaco-ui-supervisor-addendum.md`](./amaco-ui-supervisor-addendum.md);
the shipped-phases section at the bottom tracks the post-V0 work we've
landed against `main`.

Legend
- `[x]` shipped on `main`
- `[ ]` not yet built
- `[~]` partial / scoped down — see note
- _italics_ = explicitly out-of-scope by the spec ("Must Not Implement Now")

---

## 1. What We Are Building

- [x] V0 of Amaco — local-first multi-agent orchestrator for software tasks

## 2. Core Product Philosophy

- [x] Local-first; no model API
- [x] CLI-first; UI is a supervisor over the same artifacts
- [x] Run-as-data: every run is an inspectable, replayable folder
- [x] Explicit-only write actions; gated apply / validate / revert

## 3. Scope for This Implementation

### Must Implement Now

- [x] `amaco init`, `run`, `status`, `abort`, `doctor`
- [x] Provider abstraction (Claude Code + generic CLI)
- [x] Plan → Architect → Execute → Validate → Review → Fix → Verify workflow
- [x] Per-stage prompts and per-agent permission profiles
- [x] Skills system (project + Claude Code skills, attach by id)
- [x] Per-run isolated git worktree + branch
- [x] Final report + run artifacts
- [x] Permission profiles enforced before invocation
- [x] Doctor checks
- [x] Tests (vitest)

### Must Design For, But Not Fully Implement Now

- [x] Supervisor dashboard (delivered as a sequence of UI phases — see below)
- [x] Multi-task scheduler beyond per-task `amaco queue` — `queuePolicy: "fair"`, per-source `sourceQuotas`, `defaultSourceConcurrency`, pure `pickNextEntry`. See Shipped Phases.
- [ ] Cloud / Docker execution backends (interface in place; only local-worktree implemented)
- [ ] Pluggable workflow DAGs (single linear workflow ships)

### Must Not Implement Now

- _Anthropic API client_ — never.
- _OpenAI / model SDK / hosted dashboard / GitHub or GitLab API_ — never (in V0).
- _Auto-push / auto-merge / arbitrary shell from HTTP_ — never.
- _In-browser editor / generic UI file write_ — never (write actions are narrow + gated).

## 4. Technical Stack

- [x] TypeScript, ESM, Node 18.17+
- [x] tsup for CLI bundle, Vite + React 19 for UI
- [x] Vitest, Fastify, zod, YAML, execa, commander, lucide-react, Tailwind 4
- [x] highlight.js for read-mode syntax highlighting

## 5. Architecture Overview

- [x] Core services: orchestrator, artifact store, event log, state machine, run-context
- [x] Provider runner abstraction
- [x] Execution backends (local-worktree)
- [x] Permissions / policy engine / validation runner / review + verification parsers
- [x] Server (Fastify, 127.0.0.1) + UI (Vite, hash router)

## 6. Repository Structure

- [x] `src/{agents,cli,core,execution,git,notes,notifications,permissions,policies,project,providers,reviews,roadmap,scheduler,server,setup,skills,terminal,ui,utils,workflow}/`
- [x] `tests/` mirrors source layout; `tests/integration/` for full-chain smokes

## 7. Core Concepts

### 7.1 Orchestrator

- [x] Linear state machine: created → planning → planned → architecting → architected → executing → validating → reviewing → fixing → verifying → merge_ready / blocked / failed / aborted / waiting_for_approval
- [x] Run isolation via git worktree
- [x] Review loop with bounded `maxReviewLoops`

### 7.2 Agents

- [x] planner / architect / executor / fixer / reviewer / verifier
- [x] Per-agent provider + prompt + permissions + skills
- [x] User-extendable via `.amaco/project.yml`

### 7.3 Providers

- [x] `cli` (generic) + `claude-code` discriminated union
- [x] Provider detection wizard
- [x] `amaco provider set/list/detect/setup/test`

### 7.4 Workflows

- [x] Built-in `default-plan-build-review` workflow
- [x] `workflow.maxReviewLoops` + `requireHumanMerge`

### 7.5 Permissions

- [x] Permission profiles config (read / write / execute scopes)
- [x] Enforced before provider invocation

### 7.6 Skills

- [x] Discover `.amaco/skills/` + Claude Code skills (`~/.claude/skills/`)
- [x] Assign skills to agents (CLI + dashboard)
- [x] `amaco skills list/show/assign/unassign`

## 8. Project Installation Model

- [x] `pnpm dlx amaco init` scaffolds `.amaco/`
- [x] Setup wizard for vibe-coders (`amaco setup`)

## 9. Default `.amaco/project.yml`

- [x] Generated template covers project meta, git, workflow, execution, providers, agents, commands, permissions, policies, scheduler, editor, validationProfiles

## 10. Default `.amaco/rules.md`

- [x] Generated rules template with project overview / architecture / code-style / testing / security / product / agent-behavior sections

## 11. Default Agent Prompt Files

- [x] Generated under `src/agents/default-prompts/` (planner, architect, executor, fixer, reviewer, verifier, roadmap-planner)

## 12. Agent Prompt Requirements

- [x] Planner — Markdown plan with sections (Goal / Scope / Non-Goals / Affected Areas / Implementation Steps / Validation Strategy / Risks / Human Approval Needed? / Reviewer Checklist)
- [x] Architect — Architecture / Risk Decision sections
- [x] Executor — Implementation Summary sections
- [x] Fixer — Fix Summary sections
- [x] Reviewer — Review sections + AMACO_FINAL_DECISION
- [x] Verifier — Final Verification sections + AMACO_FINAL_DECISION

## 13. Skills System

- [x] `# Attached Skills` block appended to prompts at runtime

## 14. Permissions System

- [x] `permissions.profiles.{readOnly, readWrite, readWriteExecute}` configurable via project.yml
- [x] Enforced via `assertExecutableContext`

## 15. Provider System

- [x] `cli` provider with arg/stdin input modes
- [x] `claude-code` provider with parsed output + cost/token telemetry

## 16. Execution Backend System

- [x] `local-worktree` backend ships
- [ ] _Cloud / Docker backends documented but not implemented_

## 17. Workflow System

- [x] Linear default workflow
- [ ] _Pluggable DAGs documented but not implemented_

## 18. Run Folder Structure

- [x] `.amaco/runs/<runId>/{state.json, events.ndjson, artifacts/, metrics/, approvals.json, ...}`
- [x] Suggestion / bundle artifacts (`suggestions.json`, `suggestion-bundles.json`, `suggestion-patches/`, `suggestion-validations/`, `suggestion-bundle-validations/`, `suggestion-bundles/`)

## 19. Run ID

- [x] `${slug(task)}-${timestamp}` with safe-id regex

## 20. State File

- [x] zod-validated `state.json` with status / branch / worktree / approvals fields

## 21. Event Log

- [x] `events.ndjson` append-only, typed `AmacoEventType` union (covers run / agent / provider / approval / skill / notification / editor / suggestion / bundle / validation_profile_updated events)

## 22. Git Behavior

- [x] Branch + worktree per run
- [x] Never pushes / merges automatically
- [x] Diff redaction for secret-like files

## 23. Policy Engine

- [x] `forbidMainBranchWrites`, `forbidSecretsAccess`, `forbidAutoPush`, `forbidAutoMerge`, `preserveArtifacts`
- [x] `requireApprovalAtStages` for forced approval gates

## 24. Validation Runner

- [x] `commands.validate` executed in the worktree
- [x] Per-command result file
- [x] Validation profiles (named subsets, `commands.validationProfiles`)
- [x] Suggestion + bundle scoped validation runners

## 25. Review Parser

- [x] Parses Reviewer artifact for decision + AMACO_FINAL_DECISION
- [x] Parses `AMACO_SUGGESTION` marker blocks (title / file / lines / body / proposed patch / validation profile)

## 26. Verification Parser

- [x] Parses Verifier artifact for decision

## 27. Prompt Builder

- [x] Composes plan + prior artifacts + rules + skills + validation context into per-stage prompts

## 28. CLI Commands

### 28.1 `amaco init`

- [x] Scaffold `.amaco/`, optional setup wizard

### 28.2 `amaco run`

- [x] Run a task end-to-end
- [x] `--task <taskId>` to link to a roadmap task
- [x] `--ui` / `--ui-port` shortcuts to start dashboard alongside

### 28.3 `amaco status`

- [x] List runs (JSON or table)

### 28.4 `amaco abort <run-id>`

- [x] Mark run aborted (preserves worktree)

### 28.5 `amaco doctor`

- [x] git / project / providers / agents / validation checks
- [x] Notification gateway / env-var checks
- [x] Validation profiles section (default + named + stale references + malformed files)
- [x] `--fix` adds dirs/templates only; never invents profiles or rewrites references

### Additional command groups (post-V0)

- [x] `amaco config get/set/show/validate`
- [x] `amaco provider detect/list/set/setup/test`
- [x] `amaco skills list/show/assign/unassign`
- [x] `amaco approvals list/show/decide`
- [x] `amaco roadmap items/tasks/plan/proposals/accept`
- [x] `amaco tasks list/show/queue/cancel/report/comments`
- [x] `amaco queue list/add/run/status/conflicts`
- [x] `amaco notifications list/read/resolve/settings/test`
- [x] `amaco gateways list/test/enable/disable`
- [x] `amaco editor detect/set/test`
- [x] `amaco suggestions list/show/approve/reject/apply/validate/revert/profile`
- [x] `amaco bundles list/create/add/remove/preflight/approve/reject/apply/smart-apply/validate/revert/profile`
- [x] `amaco validation profiles/profile show/usage/migrate/rename/clear-references/profile doctor/migrations`
- [x] `amaco policies list/check <patchFile>/doctor`
- [x] `amaco terminal list/close <sessionId>`
- [x] `amaco replay <runId> [--json]` — read-only projection mirroring the dashboard's Replay tab

## 29. Final Report

- [x] `12-final-report.md` with Run / Final Decision / Verification / Summary / Planner / Architecture / Execution / Validation / Runtime Metrics / Approval Decisions / Review / Review Loops / Policy Warnings / Review Suggestions / Review Passes / Next Steps

## 30. Open-Source README

- [x] Quickstart for vibe-coders
- [x] Provider detection / guided setup
- [x] Config without editing YAML
- [x] Doctor and recovery / validation commands
- [x] Run artifacts / safety model
- [x] Local Supervisor Dashboard sections (skills, approvals, board, runs, queue, proposals)
- [x] Notification center / browser notifications / communication gateways / secrets via env vars
- [x] Project-aware dashboard / Codebase explorer / Git status & history / Agent work attribution
- [x] Live codebase freshness / open in editor / review suggestions / applying suggestions safely / validation profiles
- [x] Apply & validate / auto-revert / smart apply / partial states / doctor profile support / editable profiles
- [x] Limitations of V0 / Roadmap / Contributing / License

## 31. GUI Future-Proofing

- [x] All run state lives in `.amaco/runs/`; UI reads the same files the CLI writes

## 32. Cloud Future-Proofing

- [x] Execution backend interface documented (only local-worktree ships)

## 33. Human Approval Gates

- [x] Agent-emitted `HUMAN_APPROVAL: REQUIRED`
- [x] Forced approvals via `policies.requireApprovalAtStages`
- [x] Structured approval objects (risk / source / requestedAction)
- [x] Bundle / suggestion approval reuses `ApprovalService`

## 34. Tests

### Config Loader / Slug / Artifact Store / State Machine / Prompt Builder / Review Parser / Verification Parser / Validation Runner / Permission Profiles / Workflow Schema

- [x] All originally-required test suites green
- [x] Plus: notifications, codebase-aware, live-suggestions, validation profiles, smart apply, profile audit, server-route integration, syntax highlighting, suggestion workflow integration smoke

## 35. Package Scripts

- [x] `pnpm dev`, `dev:ui`, `clean`, `build`, `build:cli`, `build:ui`, `typecheck`, `test`, `test:watch`, `lint`

## 36. Security / Safety Model

- [x] 127.0.0.1 bind only, origin whitelist
- [x] Central path guard (project root + run worktree only)
- [x] Secret-file redaction across diff / file tree / file viewer / patch safety check
- [x] Apply / revert / validate never touch project root or push / merge
- [x] Auto-revert is opt-in per action
- [x] Profiles are user-owned — doctor never invents them

## 37. Manual Smoke Test

- [x] V0 fake-provider smoke (per spec)
- [x] Per-phase smokes (notifications, codebase, suggestions ingest → apply → validate → revert, smart apply, profile editing) covered by integration tests
- [x] Run Replay tick-able checklist — [`docs/smoke-tests-replay.md`](./smoke-tests-replay.md) (CLI, dashboard, deep-links, cross-links, filter, permalink, keyboard, posture)
- [x] Pause / resume tick-able checklist — [`docs/smoke-tests-pause-resume.md`](./smoke-tests-pause-resume.md) (state primitives, CLI, server routes, RunHeader buttons, orchestrator round-trip, posture)
- [x] Effort + read-only tick-able checklist — [`docs/smoke-tests-task-effort-readonly.md`](./smoke-tests-task-effort-readonly.md) (resolver, CLI, dashboard task panel, route 409 refusals on read-only, abbreviated workflow, permission override)

## 38. Acceptance Criteria

- [x] V0 acceptance (init → run → merge_ready / blocked, artifacts, isolation, no auto-push)
- [x] UI acceptance (runs list, run detail, event stream, diff, artifacts, validation, notes, skills, approvals, board, queue, proposals, notifications, codebase, git, agent work, suggestions, review passes, validation profiles)

## 39. Roadmap to Document, Not Implement

- [x] Pause / resume runs — `state.pauseRequested` + `pausedAtStatus`, orchestrator pause gates at the major stage boundaries, `amaco pause` / `amaco resume` CLI, `POST /api/runs/:id/{pause,resume}` routes, RunHeader Pause/Resume buttons (`paused` status badge, `pause queued` chip when buffered while non-paused), 11 new unit tests for the state primitives
- [x] Interactive approval gates (richer transition surfaces) — structured approvals + per-stage policy approvals shipped (see Shipped Phases `9347194`)
- [ ] Pluggable workflow DAGs / parallel agents (V1+)
- [ ] Docker / cloud execution backends (V1+)
- [ ] GitHub PR creation / GitLab support / auto-merge under strict gates (V1+)
- [~] Provider presets for Codex / OpenCode / Aider (V1+) — Codex starter preset shipped (`src/providers/presets/codex.ts`, `buildCodexPresetConfig`, `amaco provider setup` wizard offers it as a choice, `KNOWN_PROVIDERS` notes updated, 8 new tests). OpenCode + Aider deferred to V2 because their flag matrices are less stable; honest "detected, needs setup" path remains.
- [x] Per-task effort + provider override — `Task.effort` / `Task.providerOverride`, project-shared `providers.effortMap`, pure resolver in `src/core/effort-resolver.ts` (7 new tests), orchestrator picks the effective provider, CLI `--effort` / `--provider` on `amaco run` + `amaco tasks add`, PATCH `/api/tasks/:id` accepts the new fields, dashboard task-detail dropdowns, run-header chips for effort + resolved provider.
- [x] Read-only tasks — `Task.readOnly`, `RunState.readOnly`, orchestrator skips executor + fix loop + verifying, forces `readOnly` permission profile on every agent, server refuses apply/validate/revert (409) on read-only runs, dashboard READ-ONLY badge in `RunHeader`, suggestion-row actions hidden, CLI `--read-only` on `amaco run` + `amaco tasks add`.
- [x] Heuristic effort auto-detect — pure `classifyEffort({ text, files })` in `src/core/effort-heuristic.ts` (11 new tests). CLI: `amaco run` + `amaco tasks add` always print a one-line "(suggested effort: X — reasons)" verdict; `--auto-effort` applies it when `--effort` isn't passed. Server: `POST /api/effort/classify`. Dashboard: task-detail panel surfaces the verdict + one-click "apply" when the heuristic disagrees, ✓-match hint when they agree.
- [x] Secret scanning / policy plugins / run replay UI — all shipped (see Open Backlog entries below and Shipped Phases `abf5304`, `d39bd21`, `3bb9a00`)
- [ ] Replace WhatsApp placeholder with real Twilio / Cloud-API adapter (V1+)

## 40. Development Order

- [x] Followed the published phase order (see Shipped Phases section at the bottom)

## 41. Final Response Required From Claude Code

- [x] V0 implementation report delivered (against spec §41)
- [x] Subsequent phase reports follow the same structure

## 42. Final Reminder

- [x] No model API. No cloud. CLI-first.

---

# Supervisor UI Addendum (`amaco-ui-supervisor-addendum.md`)

## 1. UI Product Goal

- [x] Local-first dashboard that supervises Amaco runs; same data the CLI uses

## 2. CLI + UI Relationship

- [x] Both speak to `.amaco/` directly; UI never duplicates orchestration

## 3. Recommended Architecture

- [x] Fastify @ 127.0.0.1, JSON API, SSE for live state
- [x] React 19 + Vite, hash router, Tailwind, lucide icons

## 4. UI Tech Stack

- [x] No SSR, no auth, no remote calls

## 5. Local Server

- [x] `amaco ui` boots Fastify + serves the bundle
- [x] Origin allow-list (`localhost` / `127.0.0.1`)

## 6. API Shape

- [x] Runs / events / artifacts / diff / metrics / notes / approvals / skills / setup
- [x] Roadmap / tasks / queue / proposals / scheduler
- [x] Notifications / gateways / project / tree / file / git / agent-work / code-references / codebase-events
- [x] Editor / suggestions / suggestion-bundles / validation/profiles
- [x] Policies / terminal / runs/:id/replay

## 7. Real-Time Visibility

- [x] SSE event stream per run
- [x] SSE codebase events for project + run worktrees (project.git.changed / run.git.changed / filetree.changed / codebase.snapshot.updated)

## 8. Diff Viewer

- [x] File list with status pills + insertions / deletions
- [x] Per-file diff with hunk colour, secret redaction, copy / open-in-project / open-in-worktree / open-in-editor
- [x] File viewer with line numbers + syntax highlighting

## 9. Notes and Annotations

- [x] Per-run notes panel + per-task comments

## 10. Supervisor UX

### 10.1 Runs List

- [x] Sidebar runs list + dedicated "All runs" page

### 10.2 Run Detail

- [x] Worktree block (branch / dirty / ahead-behind / latest commit) + freshness indicator
- [x] Inspector tabs: Diff / Artifact / Suggestions / Agent work / Git / Validation / Terminal / Replay / Logs / Notes / Skills / Approvals / Metrics

### 10.3 Workflow Timeline

- [x] Stage timeline + active-agent card

### 10.4 Active Agent Card

- [x] Shows provider + stage + skills + duration

### 10.5 Artifacts Panel

- [x] Tree view + viewer; code references clickable

### 10.6 Validation Panel

- [x] Per-command status + retries summary
- [x] Suggestion / bundle validation result blocks with profile name + source

### 10.7 Notes Panel

- [x] Inline note composer + thread

## 11. CLI Should Remain First-Class

- [x] Every UI action has a CLI equivalent

## 12. Package Scripts

- [x] `pnpm dev:ui`, `pnpm build:ui`

## 13. Important UI Scope Discipline

- [x] No editing, no auth, no model APIs
- [x] Interactive terminal is **opt-in** (`policies.allowInteractiveTerminal`, default false). PTY I/O over WebSocket only; CWD restricted to the run worktree; project root refused. `node-pty` is optional — missing → honest disabled state.
- [x] One narrow write surface: gated suggestion / bundle apply / validate / revert + editor handoff

## 14. Security for UI

- [x] Path guard everywhere, secret-file refusal, no shell from HTTP

## 15. Final Product Definition Update

- [x] "Local-first supervisor for autonomous local CLIs"

## 16. Acceptance Criteria Additions

- [x] All addendum acceptance criteria green (see § 38)

## 17. Recommended Implementation Strategy

- [x] Phase-by-phase delivery — see Shipped Phases below

---

# Shipped Phases (chronological, all merged to `main`)

- [x] **V0 (`72c1eda`)** — local-first orchestrator, plan→architect→exec→validate→review→fix→verify
- [x] **Vibe-coder onboarding (`f9e26f0`)** — provider detection + guided setup
- [x] **Supervisor UI + telemetry (`fb5cedf`)** — runtime metrics, Claude provider parser, skills discovery
- [x] **Skills + approvals (`c07714c`)** — dashboard skill assignment + human approval gates
- [x] **Structured approvals + policies (`9347194`)** — typed approval requests + per-stage approval policies
- [x] **Roadmap board + scheduler (`72c2d06`)** — board, micro-tasks, run-to-task linking, concurrent runs
- [x] **Proposals + dependency graph (`2f595f8`)** — proposal accept parser + dependency UX
- [x] **Notifications + gateways (`5ef425d`)** — notification center + CLI/in-app/webhook/Discord/Slack/Telegram (WhatsApp placeholder) + attention routing
- [x] **Codebase-aware UI (`2baf5dc`)** — project metadata, file viewer, git status / history, agent-work attribution
- [x] **Live freshness + gated actions (`653f4ff`)** — SSE freshness, open-in-editor, suggestion capture + gated apply
- [x] **Post-apply validation + bundles + revert (`1e7fe79`, `81f39d4` parser fix)** — explicit validation, review-pass bundles, safe revert
- [x] **Read-mode syntax highlighting (`75efe94`)** — highlight.js wrapper + amaco-themed CSS, file viewer integration
- [x] **Integrated QA + smart apply + auto-revert (`4612493`)** — opt-in validate-then-revert + smart-apply-step-by-step
- [x] **Validation profiles (`37025d4`)** — named profiles, marker support, smart-apply per-step profiles
- [x] **Profile-aware doctor + editable profiles (`9111ee3`)** — audit service, doctor section, PATCH endpoints, dashboard dropdown
- [x] **Run Replay UI + deep-link integration (`abf5304`, `d39bd21`, `3bb9a00`)** — read-only `GET /api/runs/:id/replay` projection + `LazyReplayPanel` (phase + event scrubber, run-wide summary cards, synthetic notification / terminal rows, missing-or-malformed honesty), then deep-link plumbing (`?tab=…&replayEvent=… | replayPhase=… | replayMatch=…`), focus resolver + scroll-into-view, and "Replay" cross-links on Suggestions / Approvals / Notifications rows + per-row link in `RunList`
- [x] **Run Replay concept finish (`cc2dec0`, `8b19f8e`, `11415c7`)** — client-side filter bar over `ev.type + ev.message` + multi-select phase chips; permalink button on the selected-event card; keyboard scrubber (`↑`/`k`, `↓`/`j`, `Home`, `End`); CLI `amaco replay <runId>` (text + `--json`); per-row "Replay" link in `RunList`; `docs/smoke-tests-replay.md` checklist. 21 new tests.
- [x] **Pause / resume (`9842b22`)** — `paused` status + `pauseRequested` / `pausedAtStatus` on `RunState`; orchestrator pause gates at five major stage boundaries with safe external-abort handling; `amaco pause` / `amaco resume` CLI; `POST /api/runs/:id/{pause,resume}` routes; RunHeader Pause/Resume buttons + `pause queued` chip + `paused` status badge; 11 new tests; `docs/smoke-tests-pause-resume.md` checklist.
- [x] **Codex starter preset (`12b2fa1`)** — opt-in starter preset for OpenAI Codex CLI (`src/providers/presets/codex.ts`, `buildCodexPresetConfig`); the setup wizard offers it when codex is on PATH; doctor --fix never auto-configures it (presetReady stays false in detection). 8 new tests.
- [x] **Per-task effort + read-only runs (`0446373`)** — `effort` (low/medium/high) + `providerOverride` on `Task` + `RunState` routed via project-shared `providers.effortMap`; pure resolver in `src/core/effort-resolver.ts` (7 new tests); `readOnly` task flag drives an investigation-only run (orchestrator skips executor/fix/verify, forces readOnly permission profile, server refuses apply/validate/revert with 409); CLI flags on `amaco run` + `amaco tasks add`; task-detail panel + run-header chips + suggestion-row gating.
- [x] **Panel Phase 3 — Roadmap + task CRUD (`2ba893c`)** — full kanban Roadmap page with 9 columns (Backlog, Ready, Queued, Running, Approval, Review, Blocked, Done, Closed=failed+cancelled), live cursor (`←/→/h/l` skips empty columns; `↑/↓/k/j` walks within), task detail pane below the board. Create / edit / delete / queue all happen inside the panel: `n` opens the New Task form, `e` opens the same form pre-seeded with the selected task's values, `d` asks for `y/N` confirmation then deletes (refused by `RoadmapService.deleteTask` when the task is linked to an active run), `Q` queues with `source=user`, `c` cycles a backlog task to ready. Form has tab-cycled focus across title/description/priority/effort/providerOverride/readOnly, enum pickers with `←/→`, space toggles readOnly, `D` suspends the panel into `$EDITOR` (argv-only spawn, no shell, temp file 0o600) to edit descriptions, Enter saves with pure `validateTaskForm` checking title presence and normalizing empty effort/providerOverride to null. Pure modules + tests: `board.ts` (column grouping + empty-skipping cursor), `form.ts` (reducer + validator). App-level keymap reshaped so lowercase `q` stays quit and uppercase `Q` is freed for page-specific use; App's useInput bails when the roadmap form / delete-confirm owns input so 'q' in a title field doesn't quit. 15 new tests across board, form, and the new `RoadmapService.deleteTask`.
- [x] **Panel Phase 2 — Dashboard + Runs detail (`962d853`)** — new Dashboard home (default landing tab) with overview stat cards (active runs, queue depth, pending approvals/suggestions, scheduler state), Active Runs short list, Recent Activity feed merged across all runs newest-first, Recently Finished block. Runs page upgraded to a two-column layout: narrow left list (with `a<n>` / `s<n>` chips for pending approvals/suggestions per run) and a rich right-side inspector with three sub-tabs — Overview (full fact sheet incl. effort, override, mode, paused-at, live skills + MCP), Events (event tail with type-colored rows and a `/` filter via ink-text-input that scopes by case-insensitive substring across `type + message`, with X/Y match counter), and Validation (last-N validation events with a "no validation yet" honest empty state). Keys: `tab` cycles sub-tabs, `o/e/v` jumps directly, `/` opens the filter (auto-jumps to Events), Esc closes it; `page.set` clears modal layers so cycling tabs never strands a half-open filter. Snapshot extended with `pendingApprovals` + `pendingSuggestions` per run, an `aggregates` block (also exposed via `amaco shell --once`), and a sorted `recentActivity` feed across all runs. 13 new tests across snapshot aggregates, reducer (inspector cycle + filter open + filter-text persistence across tab switches), and the pure `filterEvents` helper.
- [x] **Panel Phase 1 — ink foundation (`df90373`)** — rewrote `amaco shell` on top of `ink` (React for the terminal) so we get focus, flex layout, text-input, and modal layering for free. New surface: top tab bar with `1-0` hotkeys for every future page (Dashboard, Runs, Roadmap, Queue, Agents, Skills, Approvals, Suggestions, Notifs, Doctor), context-sensitive footer keymap, toast layer with auto-dismiss, vim-style `:` command palette with fuzzy filter + `↵`/Esc handling, `?` help overlay, `q`/Ctrl+C quit. Runs page is fully ported (no regression — same list + inspector + queue snippet + pause/resume/abort/y-N confirm). Other tabs render an honest "Phase N" placeholder so the structure is visible from day one. Pure modules: `ui-state.ts` (reducer + per-page selection cursors + toast cap), `palette.ts` (catalog + subsequence scorer + cap). 17 new tests. Old hand-rolled `shell-render.ts` + `shell-runtime.ts` removed; `shell-snapshot.ts` + `shell-actions.ts` reused as-is.
- [x] **Interactive shell / TUI panel (`3e80f3a`)** — new `amaco shell` command opens a full-screen alt-screen TUI that reflects live orchestrator state: active runs (id, status, current agent, provider, effort, read-only, pause flags), per-run inspector pane (effort/override, current skills + MCP servers, last N events colored by type), queue snapshot (source + priority + counts), and scheduler header (policy, max concurrent, quotas, paused indicator). Keyboard controls: ↑/↓/k/j navigate, p pause selected, r resume selected, a abort (with y/N confirmation), i toggle inspector, ? help overlay, q/Ctrl+C quit. Pure `buildShellSnapshot` reads state.json + events.ndjson + scheduler files (no in-process subscription needed — works against any project root, including watching another process's run). Pure `renderShell` returns a paint-once frame string so the loop only writes when the frame changed. Actions reuse `pause-service` + the same abort transition the existing `amaco abort` uses, so the orchestrator picks them up via its normal polling — no new IPC surface. 13 new tests covering snapshot derivation, action writes, frame rendering, confirm dialog, help overlay. Also exposes `--once` flag that emits the snapshot as JSON for scripting / smoke tests.
- [x] **Queue fairness + per-source quotas (`b878e6c`)** — every `QueueEntry` now carries a `source` (default `"user"`); scheduler config adds `queuePolicy: "fair"` (round-robin by source load, FIFO within a source), `sourceQuotas: { <source>: <max-in-flight> }`, and `defaultSourceConcurrency`. Pure `pickNextEntry` in `src/scheduler/picker.ts` (11 new tests) — verdicts are `pick | at-capacity | empty | all-blocked(reasons[])`. Scheduler service drives the picker, tracks in-flight source counts, and logs `source quota exhausted` when an entry is held. CLI `amaco queue add --source <name>` and the QueuePage surfaces the source chip + active quotas. Old queues without `source` parse back as `"user"`, preserving FIFO/priority behaviour.
- [x] **Agent + skill MCP servers (`7679b75`)** — agents declare `mcpServers` in `.amaco/project.yml`; skills declare them in a sibling `.mcp.json` next to `SKILL.md`. `src/mcp/{mcp-schema, mcp-resolve, mcp-config-writer}` (zod schema rejecting shell metachars, pure agent/skill merge with agent>earlier-skill precedence and collision report, 0o600 `mcp.json` writer). Orchestrator materializes one config per agent invocation under `mcp/<stage>-<agent>.json`, emits a new `mcp.attached` event with server names + sources, and threads `mcpConfigPath` through `runProvider`. `claude-code` provider auto-injects `--mcp-config <path>`; every provider also exports `AMACO_MCP_CONFIG=<path>` so any CLI provider can opt in. Skills API + SkillsPanel surface an "N MCP" chip + an inline `.mcp.json error` chip when validation fails. 18 new tests (mcp-resolve, mcp-config-writer, skill-discovery .mcp.json + error path).
- [x] **CLI hint overlay (`99eeea5`)** — bottom-right floating `Terminal` button on every dashboard view expands into a card with the equivalent `amaco …` commands for the current Route + run/task ID interpolation + flag tips (`--effort`, `--read-only`, `--provider`); pure `hintForRoute` in `src/ui/lib/cli-hints.ts` (5 new tests), `CliHintOverlay` component, wired in `App.tsx`.
- [x] **Heuristic effort auto-detect (`2bfe34f`)** — pure `classifyEffort({ text, files })` in `src/core/effort-heuristic.ts` (11 new tests); CLI prints a per-task verdict + reasons on every `amaco run` / `amaco tasks add`; `--auto-effort` applies; `POST /api/effort/classify` exposes the same logic; dashboard task-detail panel surfaces the verdict with a one-click apply when it disagrees.

---

# Open Backlog (post-V1)

## Near-term polish

- [x] Profile usage counters (telemetry-only) — `validation-profile-usage-service.ts`, CLI `amaco validation usage`, `GET /api/validation/profile-usage`, surfaced per-row in `ProfileMaintenancePanel`
- [x] Did-you-mean for stale profile references in doctor — `suggestProfileName` (edit-distance ≤ 2) wired into `doctor-service.ts` and the new `validation profile doctor` CLI
- [x] Live SSE for suggestion / bundle lists — `SuggestionsPanel` and `ReviewPassPanel` both subscribe via `streamRunEvents`; the 5 s `setInterval` is a fallback only
- [x] `amaco validation profile doctor --all` to lift the 50-run audit cap — new subcommand with `--all` / `--run <id>` / `--json`; audit service accepts a scope
- [x] Run Replay integration polish — deep-link route fields (`?tab=replay&replayEvent=<n> | replayPhase=<key> | replayMatch=<suggestion|approval|notification>:<id>`), pure `parseHashRoute` / `serializeRoute` in `src/ui/app/route.ts`, focus resolver + `scrollIntoView` + unresolved-focus banner in `ReplayPanel`, "Replay" cross-links on Suggestions / Approvals / Notifications rows, per-row "Replay" button in `RunList`, synthetic `notification.created` events now carry `data.id` for cross-link matching. 12 new tests (`tests/ui-route.test.ts` + extension to `tests/run-replay.test.ts`).
- [x] Run Replay concept finish — client-side filter bar (text search over `ev.type + ev.message`, multi-select phase chips, clear, visible/total counts, "no matches" empty state) backed by pure `filterReplayEvents` in `src/ui/components/replay/replay-filter.ts` (9 new tests); permalink button on the selected-event card that copies `?replayEvent=<n>` to clipboard; keyboard scrubbing (`↑`/`k`, `↓`/`j`, `Home`, `End`, disabled while the search input is focused); CLI `amaco replay <runId>` with text summary + `--json` (`src/cli/commands/replay.ts`) sharing the same `buildRunReplay` projection as the UI; README updated with the deep-link + CLI surface area.

## Larger scope (deliberately deferred)

- [x] Pause / resume active runs from CLI + UI — state-machine extension (`paused` status + `pauseRequested` + `pausedAtStatus`), orchestrator pause gates at the major stage boundaries, `amaco pause` / `amaco resume` CLI, `POST /api/runs/:id/{pause,resume}` routes, dashboard Pause/Resume buttons; see Shipped Phases below
- [ ] Custom workflow DAGs and parallel agents within a single task
- [ ] Docker / cloud execution backends
- [ ] GitHub / GitLab PR creation behind explicit user authorization
- [ ] Real WhatsApp adapter (Twilio / WhatsApp Cloud API)
- [x] Secret scanning + policy plugins + run replay UI — patch-content secret scan (`scanPatchContentForSecrets`), YAML-based user policy plugins (`.amaco/policies/*.yml`, `suggestion-apply` + `bundle-apply` surfaces, block-only, regex + glob matchers, CLI + dashboard, no JS plugins), and the read-only Run Replay tab (`GET /api/runs/:id/replay` + `LazyReplayPanel`) all shipped
- [x] Interactive terminal in the dashboard — opt-in per-run shell behind `policies.allowInteractiveTerminal` (default false). PTY I/O over WebSocket only; no command string crosses HTTP. CWD restricted to known run worktrees, project root refused. node-pty is optional; missing native module → honest disabled state in the panel.

---

_When you ship the next phase, tick the corresponding line above and add a new entry under **Shipped Phases**. New backlog items belong under **Open Backlog**._
