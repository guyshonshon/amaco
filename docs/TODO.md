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
- [ ] Multi-task scheduler beyond per-task `amaco queue` (we have FIFO + priority + dependency, no fairness/quotas)
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

- [x] `src/{cli,core,execution,git,notifications,project,providers,permissions,reviews,roadmap,scheduler,server,setup,skills,ui,utils}/`
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
- [x] `amaco validation profiles` + `amaco validation profile show <name>`

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

## 38. Acceptance Criteria

- [x] V0 acceptance (init → run → merge_ready / blocked, artifacts, isolation, no auto-push)
- [x] UI acceptance (runs list, run detail, event stream, diff, artifacts, validation, notes, skills, approvals, board, queue, proposals, notifications, codebase, git, agent work, suggestions, review passes, validation profiles)

## 39. Roadmap to Document, Not Implement

- [ ] Pause / resume runs (V1)
- [ ] Interactive approval gates (richer transition surfaces; V1)
- [ ] Pluggable workflow DAGs / parallel agents (V1+)
- [ ] Docker / cloud execution backends (V1+)
- [ ] GitHub PR creation / GitLab support / auto-merge under strict gates (V1+)
- [ ] Provider presets for Codex / OpenCode / Aider (V1+)
- [ ] Secret scanning / policy plugins / run replay UI (V1+)
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
- [x] Inspector tabs: Diff / Artifact / Suggestions / Agent work / Git / Validation / Logs / Notes / Skills / Approvals / Metrics

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

- [x] No editing, no terminal, no auth, no model APIs
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

---

# Open Backlog (post-V1)

## Near-term polish

- [x] Profile usage counters (telemetry-only) — `validation-profile-usage-service.ts`, CLI `amaco validation usage`, `GET /api/validation/profile-usage`, surfaced per-row in `ProfileMaintenancePanel`
- [x] Did-you-mean for stale profile references in doctor — `suggestProfileName` (edit-distance ≤ 2) wired into `doctor-service.ts` and the new `validation profile doctor` CLI
- [x] Live SSE for suggestion / bundle lists — `SuggestionsPanel` and `ReviewPassPanel` both subscribe via `streamRunEvents`; the 5 s `setInterval` is a fallback only
- [x] `amaco validation profile doctor --all` to lift the 50-run audit cap — new subcommand with `--all` / `--run <id>` / `--json`; audit service accepts a scope
- [~] Run Replay integration polish (unmerged, on `feature/run-replay-ui`) — deep-link route fields (`?tab=replay&replayEvent=<n> | replayPhase=<key> | replayMatch=<suggestion|approval|notification>:<id>`), pure `parseHashRoute` / `serializeRoute` in `src/ui/app/route.ts`, focus resolver + `scrollIntoView` + unresolved-focus banner in `ReplayPanel`, "Replay" cross-links on Suggestions / Approvals / Notifications rows, per-row "Replay" button in `RunList`, synthetic `notification.created` events now carry `data.id` for cross-link matching. 12 new tests (`tests/ui-route.test.ts` + extension to `tests/run-replay.test.ts`).

## Larger scope (deliberately deferred)

- [ ] Pause / resume active runs from CLI + UI
- [ ] Custom workflow DAGs and parallel agents within a single task
- [ ] Docker / cloud execution backends
- [ ] GitHub / GitLab PR creation behind explicit user authorization
- [ ] Real WhatsApp adapter (Twilio / WhatsApp Cloud API)
- [x] Secret scanning + policy plugins + run replay UI — patch-content secret scan (`scanPatchContentForSecrets`), YAML-based user policy plugins (`.amaco/policies/*.yml`, `suggestion-apply` + `bundle-apply` surfaces, block-only, regex + glob matchers, CLI + dashboard, no JS plugins), and the read-only Run Replay tab (`GET /api/runs/:id/replay` + `LazyReplayPanel`) all shipped
- [x] Interactive terminal in the dashboard — opt-in per-run shell behind `policies.allowInteractiveTerminal` (default false). PTY I/O over WebSocket only; no command string crosses HTTP. CWD restricted to known run worktrees, project root refused. node-pty is optional; missing native module → honest disabled state in the panel.

---

_When you ship the next phase, tick the corresponding line above and add a new entry under **Shipped Phases**. New backlog items belong under **Open Backlog**._
