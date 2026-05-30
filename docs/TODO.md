# Vibestrate — TODO (single source of truth)

The one place for pending work. Sequenced. Shipped work lives in
[`CHANGELOG.md`](../CHANGELOG.md) + git log; older/superseded planning docs are
archived under `docs/archive/` (gitignored).

**Why-behind-the-what (design deep-dives, linked per item):**
- New-feature decisions: [`design/roadmap-and-sequencing.md`](./design/roadmap-and-sequencing.md)
- Phase 0 spec: [`CLAUDE_CORE_MODEL_REWRITE.md`](../CLAUDE_CORE_MODEL_REWRITE.md)
  (supersedes the non-breaking draft now in `docs/archive/CODEX_PLAN.md`);
  [`design/flows-unification.md`](./design/flows-unification.md),
  [`design/vocabulary.md`](./design/vocabulary.md)
- Safety pillar: [`design/policy-enforcement-assurance.md`](./design/policy-enforcement-assurance.md) (issue #7)
- Providers/metrics: [`design/provider-structured-output.md`](./design/provider-structured-output.md)
- Hub: [`design/flows-hub.md`](./design/flows-hub.md)
- SEO/GEO ops: [`SEO_GEO_INDEXING_PLAN.md`](../SEO_GEO_INDEXING_PLAN.md)

**Legend** — every phase heading and every step carries a status icon:
✅ done · 🟡 in progress · ⬜ not started · 🚧 blocked (waiting on a dependency) ·
🧭 needs design/scoping first. The `[x]`/`[~]`/`[ ]` checkbox mirrors the icon
(done / partial / open); `🔬` flags an item that needs a design spike before code.

---

## ✅ Phase 0 — Core model rewrite (do this first)

`Task + Flow + Crew = Run`; nouns `Flow / Step / Seat / Crew / Role / Profile /
Provider`. Breaking change, no users to preserve. **This completes Epic D** (it
*is* D2 Phase B "unify the two runners" + the D1 TUI rename). Own branch, all
internal steps end-to-end without stopping, full final report. Spec:
`CLAUDE_CORE_MODEL_REWRITE.md`.

- [x] ✅ Config schemas: `profiles`, `crews`, `defaultCrew`; removed top-level
  `roles` + `effortMap`; **provider-specific `power`** (free string, not a forced
  global enum)
- [x] ✅ Flow schema: `slots`→`seats`, `step.slot`→`step.seat`, dropped `step.roleId`;
  resolve payload → `crewId`/`profileOverride`/`stepProfileOverrides`
- [x] ✅ Resolver: `step.seat` → crew role (via `fills`) → profile → provider;
  clear failures on missing/ambiguous seat fill; persists resolved snapshot
  (incl. `crewId` + per-step `seat`/`resolvedRoleId`/`profileId`/`providerId`)
- [x] ✅ **Unified runner** — every run resolves a Flow and executes through the one
  `runFlowSequence` runner (landed pre-Phase-0; preserved through the rewrite)
- [x] ✅ Orchestrator/provider path runs from resolved Profile→Provider; budget
  downgrade is stop-only for now (TODO: `budget.fallbackProfile`)
- [x] ✅ CLI: `--profile` / `--step-profile` / `--crew`; provider commands manage raw
  providers only; flow-run wizard picks per-step Profiles
- [x] ✅ Server: `/api/crews`, `/api/crews/:id`, crew-oriented PATCH, `/api/profiles`,
  `PATCH /api/profiles/:id`; new resolve payload; dropped `/api/roles`/`slotProviders`
- [x] ✅ UI: web dashboard rewired — Crew page (roster + seat coverage), new
  Profiles page, Mission Control Seat→Role→Profile allocation table, Flow
  Builder (Seat not slot). (Branch `feature/ui-crew-model`.)
- [x] ✅ Run-time seat disambiguation: `seatRoleOverrides` threaded end to end
  (CLI `--seat-role`, `/api/runs`, orchestrator, run state); the allocation
  table picks a role inline for ambiguous seats.
- [x] ✅ TUI shell: `agents` page id → Crew (lists the default crew's roles)
- [x] ✅ Docs rewrite (plain-language concept pages) + regenerated `docs/generated/*`
- [x] ✅ Tests migrated to the new model; `pnpm typecheck && pnpm build` green
- [x] ✅ **Vocab freeze:** `Step` = a Flow phase only (kept free for the Phase-3 card
  Checklist / items)

## 🟡 Phase 1 — Safety pillar (Epic S)

Hard, code-enforced gates — Vibestrate's durable value over "prompt automation
with nice UI." Lay the core path down right after the rewrite, before features
pile on. Design: `design/policy-enforcement-assurance.md` (issue #7).

- [x] ✅ **S0 — Action Broker** — one Vibestrate-owned boundary every real effect
  crosses. `src/safety/action-broker.ts` (`decide`/`record`/`readActionLog`,
  evaluator chain, `runs/<id>/actions.ndjson` evidence log) + `createActionBroker`
  factory + `gateAction` helper. All effect kinds now routed, fail-closed with
  allow/deny + ok/fail evidence: **`provider.spawn`** & **`run.complete`**
  (orchestrator; a non-allow run.complete downgrades merge_ready→blocked),
  **`file.patch`** (suggestion + bundle apply/smartApply/revert),
  **`command.run`** (validation runner), **`file.write`** (MCP config),
  **`terminal.create`** (terminal service). Evaluators are wired in S2.
- [x] ✅ **S1 — Language cleanup** — reserve "policy enforcement" for code-enforced
  gates; call prompt-injected rules "instructions" in docs/UI. Glossary defines
  Instructions (`rules.md`, advisory) vs Policy (code-enforced) vs Action Broker;
  `vibe init` template + default-rules fallback reworded to "Project Instructions"
  with an up-front advisory note. Audit found the rest of docs/UI already correct.
- [x] ✅ **S2 — Policy engine V2** — `actions:` policies in
  `.vibestrate/policies/*.yml` gate broker effect kinds (`provider.spawn`,
  `command.run`, `file.patch`, `file.write`, `terminal.create`, `run.complete`)
  with `deny` / `require_approval`, matched by provider id / command regex /
  path glob / run status. Compiled to evaluators, lazy-loaded into every broker
  via `createActionBroker`; surfaced in CLI + API + dashboard. (`run.preflight` /
  `agent.turn.diff` surfaces land with S3.)
- [x] ✅ **S3 — Post-turn diff gate** — snapshot each write-capable turn
  (`git write-tree`), diff after, evaluate built-in secret/path safety +
  `file.patch` policies → accept / require_approval (block fail-closed) /
  deny→rollback (`read-tree`+`checkout-index`+`clean`)+block. `require_approval`
  **pauses for a human** via the standard approval flow (approve → keep changes;
  reject → rollback + block). In the orchestrator role path; `src/safety/diff-gate.ts`.
- [x] ✅ **S4 — Strict apply-only mode** — `policies.strictApplyOnly`: write roles
  run read-only and propose a unified diff; Vibestrate applies it through the
  broker gateway (`src/safety/apply-gateway.ts`: secret/path safety → file.patch
  policy → audited `git apply`), refusal blocks the run. Prompt instructs the
  agent to emit a ```diff block. (End-to-end efficacy is prompt-guided; the
  gateway + refusal paths are fully tested.)
- [x] ✅ **S5 — Run Assurance artifact** — `runs/<id>/assurance.json` with discrete
  verdicts (`blocked`/`unsafe`/`unverified`/`partially_verified`/`verified`), no
  fake confidence %. Derived at every terminal state from the broker log +
  review/verification (`src/safety/run-assurance.ts`); surfaced via
  `vibe assurance`, `GET /api/runs/:id/assurance`, and a run-detail badge.
- [ ] 🚧 **S6 — OS sandbox path** — tie into the Docker/sandbox backend so
  forbidden-path guarantees become process-level (ties to the deferred Docker
  execution backend). *Blocked: needs the Docker execution backend first.*

## ✅ Phase 2 — API contract + flow portability

Design deep-dive: [`design/api-contract.md`](./design/api-contract.md); user
docs: [`content/architecture/http-api.md`](./content/architecture/http-api.md).

- [x] ✅ **API hardening** — versioned `/api/v1` (aliased to `/api` via Fastify
  `rewriteUrl` — one seam, no route refactor); optional bearer-token auth
  (`VIBESTRATE_API_TOKEN`, constant-time) gating every `/api/*`; `vibe ui --host`
  for non-loopback binds, which **refuse to start without a token** (fail-closed).
  Endpoints documented in the new HTTP API page. (design §4)
- [x] ✅ **Single-flow import/export** — `vibe flows export/import` (file or URL),
  `GET /api/v1/flows/:id/export`, `POST /api/v1/flows/import`. Schema-validate +
  secret refusal + control-char/size guard + SSRF guard on URLs + overwrite
  policy + atomic write, all through one guarded writer. (design §5)
- [x] ✅ **Flow creator API** — `POST /api/v1/flows` writes a new project flow from
  a full definition (same guarded writer); dashboard Flows page gains Export /
  Import / New-flow controls (UI⇄CLI parity).

## ✅ Phase 3 — Planning board

Board = planning only; execution lifecycle + concurrency stay in Mission/Runs
(separate nav tabs — keep apart). Three altitudes: macro (proposal → cards) ·
meso (enhance → in-card checklist) · micro (Planner Role → impl plan). Full
design: `design/roadmap-and-sequencing.md` §1.

- [x] ✅ **Board as a Trello of cards** — coarse columns `Planned · In-progress ·
  Needs testing · Completed · Archived`, derived from status + the needs-testing
  / archived overlays (`coarseColumn` in roadmap-types). Auto-nudged as status
  changes. New `archived` flag + `setArchived` (refuses while a run is active),
  `vibe tasks archive|unarchive`, `POST /api/tasks/:id/archive`, archive button
  on the task. No `parentTaskId`. (Mission Control keeps the fine stages.)
- [x] ✅ **Card Checklist** — a card holds an ordered **Checklist** of **items**,
  kept inside the card. New Task-model field (`checklist`) + service CRUD/reorder
  + `vibe tasks checklist …` + `/api/tasks/:id/checklist*` + task-detail panel.
- [x] ✅ **Assist primitive** — one internal one-shot, read-only, structured-output
  run; reused by enhance / overview / suggest. `src/assist/assist-runner.ts`:
  resolve profile (crew planner) → broker-gated `provider.spawn` → parse +
  Zod-validate JSON (1 reprompt on failure). Audit bucket `runs/assist/`.
- [x] ✅ **"Enhance"** — assist run that *decomposes* a card into its Checklist
  (not rewording). `src/assist/enhance.ts` (propose = dry-run, apply = append) +
  `vibe tasks enhance [--apply]` + `POST /api/tasks/:id/enhance` + checklist-panel
  "Enhance" button (preview → Add all). Macro proposals still create separate cards.
- [x] ✅ **Pick-up execution (continuous-mode, locked)** — flow `checklistSegment`
  repeats once per item in one worktree; holistic-plan → per-item band
  (micro-plan · implement · commit · compact-summary forward-carry · between-item
  gate) → holistic review. **Continuous** or **Step-by-step** (gate reuses
  pause/resume). Linear + stop-on-failure. Per-item commits (id trailer) + status
  write-back. Built-in `pickup` flow; `vibe tasks pickup`, `--checklist`,
  `POST /api/runs {checklistMode}`, "Run checklist" button. The forward-carry was
  built + tested first (`src/pickup/item-summary.ts`), then the loop. (design §1)
  *Deferred: per-item bounded retries; resume-from-item; mid-loop profile downgrade.*
- [x] ✅ **"Needs testing"** — non-blocking advisory: reviewer/verifier emit
  `HUMAN_REVIEW: ADVISORY` (parsed like decision markers but non-blocking) →
  card flagged `needsTesting` + reason; human verdict (`pass`→done / `fail`→
  reopen) via API + task banner + board badge. Run keeps its real verdict.
- [x] ✅ **Promote item → card** — `promoteChecklistItem` creates a new card with a
  `derivedFrom` back-pointer + stamps the item's `promotedTaskId` (relation, not
  reparent — item stays, inherits the parent's roadmap item). Guards double-
  promote, allows re-promote after the card is deleted, and `deleteTask` clears
  the dangling pointer. `vibe tasks checklist promote`, `POST …/checklist/:id/
  promote`, ↗ button + "derived from" link in the task panel.
- [x] ✅ **Suggest-next** — pure ranker (`src/roadmap/suggest-next.ts`) over the
  backlog (status backlog/ready): ready-first → priority → fewer open blockers →
  oldest. `vibe tasks suggest [--all]`, `GET /api/tasks/suggest`, a "next:" pill
  on the board. Unknown deps count as open; done deps don't block.
- [x] ✅ **C1 — Flow-complexity warning** — flows declare a `complexity` (or it's
  inferred from agent-turn count); task effort comes from the effort heuristic;
  `flowComplexityAdvice` warns when the flow is heavier than the task (gap 1 =
  gentle "consider", gap 2 = "might be too much"). Non-blocking: printed by
  `vibe run`, returned as `flowAdvice` from `POST /api/runs`. Built-ins: default
  + quality-arbitration = high, pickup = medium.

## 🟡 Phase 4 — Context & providers

Context sources + non-CLI providers shipped (the latter also gives real metrics
for cloud/localhost). The two remaining items are blocked on real CLIs we can't
fake per the test policy: **A7 CLI metrics** (Codex/Gemini structured output —
needs each CLI's real format) and **OpenCode/Aider presets** (the repo refuses
to ship *unverified* flag presets).

- [x] ✅ **Context sources** — per-run/task `ContextSource[]` (`file`/`url`)
  materialized once at run start into `runs/<id>/artifacts/context/` and injected
  into **every** agent prompt via `PriorArtifact`. Files path-guarded (secret
  paths refused) + secret-content redacted; URLs SSRF-guarded + bounded +
  redacted (`src/core/guarded-fetch.ts`). `vibe run --context-file/--context-url`,
  task-level sources (`POST /api/tasks/:id/context` + panel) inherited by runs.
  Web search stays a per-Profile capability (not faked). *(pdf deferred — needs a
  local parser.)* (design §2)
- [x] ✅ **Non-CLI providers** — `http-api` (cloud: Anthropic/OpenAI, key via env
  ref ONLY, https + non-loopback) and `localhost-proxy` (Ollama/LM Studio/vLLM,
  loopback only ⇒ no egress). `src/providers/http-api-provider.ts` (one request
  per turn, key never logged/redacted, abort+timeout). Capabilities report real
  token usage; dashboard lists them + marks cloud **external**; doctor + safe-test
  handle them (cloud test = key-set check, no surprise spend). (design §6)
- [~] 🟡 **A7 — Real metrics** — cloud/localhost http-api providers now report
  **real** token usage from the response (above). CLI providers (Codex/Gemini)
  still need their own structured-output adapters → deferred.
- [~] 🟡 **Provider presets** — Codex + Ollama shipped; OpenCode + Aider deferred.

## ✅ Phase 5 — Parallel integration + hub

- [x] ✅ **Integration / merge-preview** — `src/integration/`: `mergePreview`
  cumulative `git merge --no-ff --no-commit` dry-runs in a throwaway scratch
  worktree (surfaces branch-vs-branch conflicts + files); `integrate` merges
  selected merge-ready branches into a **fresh, dedicated** branch (refuses
  main/existing, stops on first conflict, never pushes). `vibe integrate
  list|preview|apply --into`, `GET/POST /api/integration*`, an Integration panel
  on the Runs page. Parallel execution already worked via the scheduler. (design §3)
- [x] ✅ **Guides hub** — `src/flows/hub/flow-hub.ts`: fetch + validate a curated
  `index.json` (guarded/bounded), search it, and install a flow by downloading
  its raw `flow.yml` through the existing validated + secret/shell-guarded import
  writer. `vibe flows hub list|install`, `GET /api/flows/hub` +
  `POST /api/flows/hub/install`. No backend — static raw URLs. (design §5)
  *(A dashboard hub-browser UI is a follow-up; api client is wired.)*
- [x] ✅ **Skill fetching + AI-overview** — `src/skills/skill-fetch.ts`:
  `installSkillFromUrl` (guarded/bounded fetch → secret-redact → write
  `.vibestrate/skills/<name>.md`, path-guarded); `assessSkill` is a read-only
  assist run judging **helpful / already_present / conflicting** vs the local
  skills. `vibe skills fetch <url> [--assess]`, `POST /api/skills/fetch`.

## ✅ Phase 6 — Observability (opt-in)

- [x] ✅ **A6 — Webhooks** — already delivered for **approve** (`draftApproval*`)
  and **merge** (`draftRunCompleted`) via the existing `webhook` gateway; added
  the missing **cap-hit** (`draftSpendCapHit`, fired from the spend-cap enforcer).
  Any notification routes to a configured webhook URL (POST JSON).
- [x] ✅ **OTel/Langfuse exporter** — `src/telemetry/otel-exporter.ts`:
  dependency-free OTLP/HTTP builder mapping a run's persisted metrics → a root
  run span + per-role-turn child spans (provider/model/seat + `gen_ai.usage.*`
  tokens/cost), and `exportRunToOtlp` POSTs to a user-set collector (`/v1/traces`,
  bearer via env-ref, redacted). Off by default, explicit endpoint only —
  exporter over data we already have. `vibe telemetry trace|export`. (design §7)

## Backlog (larger / deferred)

- [ ] 🧭 **Multi-project / workspace — switch + run many projects at once** (own
  phase; scope first). Today the dashboard is single-project: `vibe ui` pins one
  `projectRoot` at start and every route/store reads it; the TopBar only *shows*
  the current project label + a read-only Project page (no real switcher). Want:
  pick from recent projects and switch instantly, and have runs across several
  projects executing + visible at once. Open questions to expand later —
  **(a)** a persisted project registry / "recent projects" (where: a user-level
  `~/.vibestrate/workspace.json`?); **(b)** server shape: one server serving N
  roots (route/store keyed by project) vs. a thin workspace server that
  proxies/launches a per-project server; **(c)** run isolation + a cross-project
  aggregated Mission Control / board (each run already carries its `projectRoot`);
  **(d)** the scheduler — per-project today; a workspace-level queue or N
  schedulers; **(e)** UI: project switcher in the TopBar, an "all projects"
  overview, per-project nav scoping; **(f)** safety: path guards + the Action
  Broker stay bounded **per project root** (no cross-project reads/writes). Keep
  local-first — no hosted backend.
- [ ] ⬜ **Rewind phase 2** — resume at review/verify/fix (needs per-phase worktree
  snapshots — commit/tag each phase; current capture only keeps the final tree).
- [ ] ⬜ **Custom workflow DAGs + parallel agents within a single task** (also the
  home for checklist-DAG + continue-past-failure + parallel item execution).
- [ ] ⬜ **Docker / cloud execution backends** (interface exists; ties S6).
- [ ] ⬜ **GitHub / GitLab PR creation** behind explicit authorization (never auto).
- [ ] ⬜ **Real WhatsApp adapter** (Twilio / WhatsApp Cloud API).
- [ ] ⬜ **E1 — Windows support** — audit path handling, detached spawns, signals,
  worktrees; decide supported scope.
- [ ] ⬜ **E2 — Homebrew tap** — `guyshonshon/homebrew-vibestrate` (deferred; npm +
  `curl | sh` cover macOS/Linux).
- [ ] 🧭 **Graphy (AI context-graph) integration?** — evaluate linking an external
  context-graph into the system. 🔬 scope first.

## UI/UX polish

Much is absorbed by the Phase-0 UI work; these are residuals from the scratch notes.

- [ ] ⬜ **UI⇄CLI parity for provider setup** — fixing/setting up a provider (e.g.
  codex) must be fully doable in the UI, not only `vibe provider setup`. A
  complete advanced provider UI. (recurring principle — never tell the user to
  drop to CLI as the in-UI fix.)
- [ ] ⬜ Remove/redesign subtitles & labels that read as "made by AI."
- [ ] ⬜ Crew page: confusing label/sub-label pairs (e.g. "Fix"/"fixer").
- [ ] ⬜ Provider page: the `ON / RUNS / OK` stat-bars are unexplained — label them
  or remove.
- [ ] ⬜ Provider list (Crew): dragging should drag the ghost of the draggable.
- [ ] ⬜ Provider list: animated lock/unlock SVG for locked state.

## Ops — SEO / GEO (final)

Site-deployment hygiene for vibestrate.shonshon.com. Detail + JSON-LD snippets:
[`SEO_GEO_INDEXING_PLAN.md`](../SEO_GEO_INDEXING_PLAN.md).

- [ ] ⬜ Real favicon set (ico/svg/png/apple) linked from every page (`/favicon.ico` is 404).
- [ ] ⬜ `/sitemap.xml` alias/redirect (only `/sitemap-index.xml` works today).
- [ ] ⬜ 404 pages → `noindex, follow`; redirects for stale docs URLs (e.g. `/docs/overview`).
- [ ] ⬜ Update `llms.txt` / `llms-full.txt` after the Flow/Crew/Seat/Profile rewrite.
- [ ] ⬜ Fix live-site metadata mismatch (repo Apache-2.0 / `0.1.1` vs site MIT / `0.9.2`);
  pick one license string, use it everywhere.
- [ ] ⬜ JSON-LD: `WebSite` (name "Vibestrate"), fix `SoftwareApplication`
  license/version, add `SoftwareSourceCode` → GitHub, `sameAs` for GitHub/npm/docs.

---

_Tick a line and add a `CHANGELOG.md` entry when it merges. Keep this lean — the
"why/how" detail belongs in the linked design docs, not here._
