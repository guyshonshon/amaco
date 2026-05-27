
  Make the cost & token "ledger" metrics real and fully functional in amaco's
  metrics system. No mock or placeholder data — everything computed from actual
  run events and token counts.

  START BY READING (extend these, don't duplicate):
  - src/core/runtime-metrics.ts — per-run RuntimeMetrics, TokenUsage,
    PerModelCost (has costUsd), recomputeRunTotals().
  - src/core/overview-aggregator.ts — builds the /api/metrics/overview payload
    (daily, spendByAgent, phaseLatency, leaderboard, totals, spendCapDailyUsd).
  - src/server/routes/metrics.ts — the HTTP route.
  - src/providers/*output-parser.ts — where tokens/cost are read from each CLI's
    stdout. claude-code-output-parser.ts already captures input/output/cache
    tokens and cost_usd; check the other providers (codex, gemini, opencode,
    aider, ollama, etc.).
  - src/core/orchestrator.ts — run lifecycle (where a spend cap must gate runs).
  - src/ui/app/routes/MetricsPage.tsx + the overview types in src/ui/lib — the
    dashboard that renders the payload.

  WHAT ALREADY WORKS: token capture per run/model, and costUsd that today comes
  straight from CLIs that self-report it (Claude Code). spendCapDailyUsd is
  surfaced in the overview but NOT enforced.

  BUILD / CLOSE THESE GAPS:

  1. Pricing table → cost for ALL providers. Add a local static table of public
     list prices (USD per 1M input / output / cached tokens) keyed by model id,
     easy to update. Cost precedence: if the CLI reports a real cost, use it;
     otherwise compute tokens × price. Clearly label computed numbers as local
     ESTIMATES. No network calls.

  2. Make sure EVERY provider's output parser captures input + output (+ cache)
     tokens. Where a CLI doesn't report cost, the pricing table fills it in.

  3. Add to the metrics overview + dashboard:
     - Total tokens KPI for the window (with delta vs the previous window).
     - Tokens-by-role over time (stacked: planner / arbiter / executor /
       reviewer / verifier) for the selected range.
     - Per-model breakdown: model, calls, tokens, cost.
     - Median run duration alongside the existing average.

  4. Enforce the daily spend cap as a real hard stop: when today's estimated
     next phase) with a clear message; emit a warning notification at a
     configurable threshold (e.g. 80%). Make the cap configurable. 

  5. (Optional, if cheap) Fire a webhook (POST to a configured URL) on approve /
     merge / cap-hit, reusing the existing notifications system in
     src/notifications/.
     
  PRINCIPLES:
  - Local-first: estimates computed on-device, no telemetry, no proxying.
  - Persist per-run so 24h / 7d / 30d / 90d windows aggregate correctly.
  - Keep the /api/metrics/overview payload backward-compatible — add fields,
    don't break existing ones — and wire the new fields into MetricsPage.
    
  ACCEPTANCE:
  - A real run shows non-zero tokens AND cost for the model used, in the
    dashboard, for every provider.
  - Tokens-by-role chart and per-model table populate from real data.
  - Setting a low daily cap and exceeding it blocks the next run with a clear
    message; the warn threshold notifies.
  - pnpm typecheck and pnpm test pass; add unit tests for the pricing/cost
    computation and the cap enforcement.