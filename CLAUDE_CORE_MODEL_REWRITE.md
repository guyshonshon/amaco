# Core Model Rewrite Instructions

## Read This First

This project is still beta. There are no real users to preserve compatibility
for. Do not spend implementation effort on backward compatibility, migration
wizards, legacy config fallbacks, or preserving old YAML shapes.

Rewrite the model cleanly.

The target product model is:

```text
Task + Flow + Crew = Run
```

The target nouns are:

```text
Flow
Step
Seat
Crew
Role
Profile
Provider
```

Meaning:

```text
Flow     = the recipe
Step     = one phase in the recipe
Seat     = the kind of participant a Step needs
Crew     = the local roster
Role     = behavior/instructions inside a Crew
Profile  = runtime/model/budget setup used by a Role
Provider = the local CLI/tool behind a Profile
```

Do not use `job`. The product word is `Seat`.

Do not introduce `Assignment` as a product model. A Crew role row is enough.

## Product Explanation

Use this mental model everywhere:

```text
Flow says what should happen.
Step says one phase of that Flow.
Seat says what kind of participant the Step needs.
Crew contains local Roles.
Role says how the agent behaves and which Seats it can fill.
Profile says how that Role runs.
Provider is the installed tool/CLI.
```

Important Profile rule:

```text
Profile power/effort must be provider-specific.
Do not force every provider into one fake low/medium/high/deep enum.
Different providers expose different reasoning/effort controls, and some expose none.
```

Example:

```text
Flow: Test-first feature

Step          Seat          Crew Role              Profile
---------------------------------------------------------------
Plan          planner       Planner                codex-balanced
Implement     implementer   Backend Implementer    claude-sonnet-deep
Review        reviewer      Reviewer               opus-deep
Verify        verifier      Verifier               codex-balanced
```

## Breaking Change Policy

This implementation is allowed to break:

```text
.vibestrate/project.yml
.vibestrate/flows/**/flow.yml
.vibestrate/composer-presets.json
CLI flags that expose provider overrides
API payloads that expose slotProviders / stepProviders
tests, fixtures, generated docs, screenshots
```

Remove old concepts instead of supporting both names.

Delete or rename:

```text
Flow slot          -> Flow seat
step.slot          -> step.seat
step.roleId        -> remove from Flow steps
role.provider      -> role.profile
provider override  -> profile override
effortMap          -> profiles / profile choice
slotProviders      -> seatProfiles or profileOverrides
stepProviders      -> stepProfileOverrides
```

Keep:

```text
kind
stage
inputs
outputs
loop
approval-gate
providers
permissions
skills
```

`kind` stays internal. It tells the engine how to run the step.

`Seat` is product vocabulary. It tells the user what kind of participant the
step needs.

## Target Config Shape

Rewrite `.vibestrate/project.yml` toward this shape:

```yaml
project:
  name: example
  type: generic

providers:
  codex:
    type: cli
    command: codex
    args: ["exec"]
    input: stdin

  claude:
    type: cli
    command: claude
    args: []
    input: stdin

profiles:
  codex-balanced:
    provider: codex
    label: Codex balanced
    model: null
    power: balanced
    budget: medium
    maxTokens: null
    timeoutMs: null

  claude-sonnet-deep:
    provider: claude
    label: Claude Sonnet deep
    model: sonnet
    power: deep
    budget: high
    maxTokens: null
    timeoutMs: null

  opus-deep:
    provider: claude
    label: Claude Opus deep
    model: opus
    power: deep
    budget: high
    maxTokens: null
    timeoutMs: null

crews:
  default:
    label: Default
    roles:
      planner:
        label: Planner
        fills: [planner]
        profile: codex-balanced
        prompt: .vibestrate/roles/planner.md
        permissions: read_only
        skills: []

      backend-implementer:
        label: Backend Implementer
        fills: [implementer, executor, builder]
        profile: claude-sonnet-deep
        prompt: .vibestrate/roles/executor.md
        permissions: code_write
        skills: []

      reviewer:
        label: Reviewer
        fills: [reviewer, challenger]
        profile: opus-deep
        prompt: .vibestrate/roles/reviewer.md
        permissions: read_only
        skills: []

defaultCrew: default
```

Notes:

```text
providers = raw local tools
profiles = reusable runtime setup
crews = local role roster
roles = behavior + seat filling + profile
```

Remove the old top-level `roles` map. Roles now live under `crews.<crewId>.roles`.

Remove `effortMap`. Runtime power is handled by Profiles.

## Target Flow Shape

Rewrite Flow definitions from `slots` to `seats`.

Before:

```yaml
slots:
  executor:
    label: Executor
    defaultRole: executor

steps:
  - id: implement
    label: Implement
    kind: agent-turn
    slot: executor
    roleId: executor
```

After:

```yaml
seats:
  implementer:
    label: Implementer
    description: Makes code changes.

steps:
  - id: implement
    label: Implement
    kind: agent-turn
    seat: implementer
    stage: executing
    inputs: [task-brief, plan, architecture]
    outputs: [execution, diff]
```

Rules:

```text
Flow defines required Seats.
Step references one Seat.
Flow does not reference local Role ids.
Flow does not reference local Profile ids by default.
Flow stays shareable.
```

Validation steps and approval gates do not need a Seat.

```yaml
- id: validation
  label: Validate
  kind: validation
  stage: executing
  inputs: [diff]
  outputs: [validation]
```

## Resolution Rules

When a run starts:

```text
1. Load Flow.
2. Load selected Crew, defaulting to project.defaultCrew.
3. For each Step with a seat:
   a. Find Crew roles where fills includes step.seat.
   b. If none, fail clearly.
   c. If more than one, fail clearly unless the run has a role override.
   d. Resolve the Role profile.
   e. Resolve the Profile provider.
4. Persist the resolved run snapshot with:
   step.seat
   step.resolvedRoleId
   step.resolvedRoleLabel
   step.profileId
   step.providerId
```

No hidden fallback to old role names.

Error messages should be user-readable:

```text
This Flow needs the "implementer" seat, but the selected Crew has no role that fills it.
Open Crew and add "implementer" to a role's Seats.
```

## Same Role, Different Profile

Support step-level Profile overrides.

Use this for a step that needs the same Role behavior but a stronger runtime:

```text
Implement config parser -> Backend Implementer -> codex-balanced
Implement auth crypto   -> Backend Implementer -> opus-deep
```

Do not duplicate Roles to express runtime strength.

Bad:

```text
Backend Implementer Fast
Backend Implementer Deep
Backend Implementer Opus
```

Good:

```text
Role: Backend Implementer
Default profile: claude-sonnet-deep
Step override profile: opus-deep
```

Recommended run payload shape:

```json
{
  "flow": {
    "id": "default",
    "crewId": "default",
    "stepProfileOverrides": {
      "implement-auth-crypto": "opus-deep"
    }
  }
}
```

## Files To Change

### Config Schemas

Change:

```text
src/project/config-schema.ts
src/roles/role-schema.ts
src/project/init-template.ts
src/project/config-loader.ts
src/setup/config-update-service.ts
src/setup/provider-setup-service.ts
```

Implement:

```text
profileConfigSchema
crewConfigSchema
crewRoleConfigSchema
defaultCrew
```

Remove top-level `roles` from `projectConfigSchema`.

Add top-level:

```text
profiles
crews
defaultCrew
```

`crewRoleConfigSchema` should include:

```text
label?: string
fills: string[]
profile: string
prompt: string
permissions: string
skills: string[]
mcpServers?: Record<string, ...>
```

`profileConfigSchema` should include:

```text
provider: string
label?: string
model?: string | null
power?: string | null
budget?: "low" | "medium" | "high" | string
maxTokens?: number | null
timeoutMs?: number | null
providerOptions?: Record<string, unknown>
```

Do not hardcode one global power scale.

Examples:

```text
Claude-like provider may expose one set of effort/reasoning values.
Codex-like provider may expose another set, or no equivalent option.
Local providers may only expose model/command args.
```

The UI must show valid power/effort options for the selected Provider/Profile
based on provider metadata or preset definitions. If the provider has no native
effort control, hide or disable the field instead of faking one.

Validate:

```text
Every profile.provider exists in providers.
Every crew role.profile exists in profiles.
Every crew has at least one role.
defaultCrew exists in crews.
fills values are non-empty safe tokens.
```

### Flow Schemas

Change:

```text
src/flows/schemas/flow-schema.ts
src/flows/runtime/flow-patch.ts
src/flows/runtime/flow-resolver.ts
src/flows/catalog/builtin-flows.ts
src/flows/catalog/flow-discovery.ts
src/flows/runtime/flow-participant-ledger.ts
```

Replace:

```text
slots -> seats
FlowSlot -> FlowSeat
ResolvedFlowSlot -> ResolvedFlowSeat
step.slot -> step.seat
step.roleId -> remove from FlowStep
slotProviders -> seatProfileOverrides or seatRoleOverrides
stepProviders -> stepProfileOverrides
```

Resolved step should include:

```text
seat: string | null
resolvedRoleId: string | null
resolvedRoleLabel: string | null
profileId: string | null
providerId: string | null
```

Turn kinds requiring a seat:

```text
agent-turn
review-turn
response-turn
summary-turn
```

Kinds that do not require a seat:

```text
validation
approval-gate
```

### Orchestrator

Change:

```text
src/core/orchestrator.ts
src/core/run-launcher.ts
src/core/state-machine.ts
src/core/effort-resolver.ts
src/core/project-context-service.ts
src/core/runtime-metrics.ts
src/core/agent-work-attribution-service.ts
src/core/final-report.ts
```

Remove run-wide provider override as the primary model.

Replace with:

```text
crewId
profileOverride
stepProfileOverrides
resolved profile/provider per step
```

`runRole` should receive:

```text
resolvedRoleId
role config from selected Crew
profile config
providerId from profile.provider
```

Prompt behavior still comes from Role prompt/instructions.

Provider execution comes from Profile -> Provider.

Budget fallback behavior should move from `effortMap` to Profile selection.
If spend cap needs downgrade, switch the active Profile to a cheaper configured
Profile rather than switching raw Provider ids.

If that is too large for the first pass, keep budget downgrade disabled with a
clear TODO and update tests/docs accordingly.

### CLI

Change:

```text
src/cli/index.ts
src/cli/wizards/flow-run-wizard.ts
src/cli/commands/run.ts
src/cli/commands/tasks.ts
src/cli/commands/provider/*.ts
src/cli/commands/skills/*.ts
```

Replace user-facing flags:

```text
--provider       -> --profile
--flow-slot      -> --seat-profile or remove if UI/API owns this
--effort         -> --profile or --power only if backed by Profiles
```

Recommended minimal CLI:

```text
vibestrate run "task" --flow default --crew default
vibestrate run "task" --profile opus-deep
vibestrate run "task" --step-profile implement=opus-deep
```

Provider commands should manage raw providers only.

Profile commands should manage runtime profiles.

Add or update commands:

```text
vibestrate profile list
vibestrate profile set <profileId> ...
vibestrate crew list
vibestrate crew show <crewId>
```

### Server API

Change:

```text
src/server/routes/project.ts
src/server/routes/providers.ts
src/server/routes/flows.ts
src/server/routes/runs.ts
src/server/composer-presets.ts
src/ui/lib/api.ts
src/ui/lib/types.ts
```

Replace:

```text
GET /api/roles
PATCH /api/roles/:roleId { provider }
```

with Crew-oriented endpoints:

```text
GET /api/crews
GET /api/crews/:crewId
PATCH /api/crews/:crewId/roles/:roleId
GET /api/profiles
PATCH /api/profiles/:profileId
```

Flow resolve payload should accept:

```text
crewId
profileOverride
seatRoleOverrides
seatProfileOverrides
stepRoleOverrides
stepProfileOverrides
```

Only implement the override set that the UI actually uses in the first pass.
Do not keep `slotProviders`.

### UI

Change:

```text
src/ui/app/routes/CrewPage.tsx
src/ui/app/routes/FlowBuilderPage.tsx
src/ui/app/routes/FlowsPage.tsx
src/ui/app/routes/MissionControlPage.tsx
src/ui/components/mission/v3/Composer.tsx
src/ui/components/runs/v3/*
src/ui/components/runs/StepsInspector.tsx
src/ui/components/skills/SkillsPanel.tsx
src/ui/components/design/roleTone.ts
```

UI language:

```text
Provider page = raw tools
Profile page = model/runtime setups
Crew page = local roles and the Seats they fill
Flow page = recipe and Steps
Mission Control = choose Flow + Crew, then Run
```

Mission Control should show:

```text
Flow: Test-first feature
Crew: Default

Step          Seat          Role                  Profile
---------------------------------------------------------------
Plan          planner       Planner               codex-balanced
Implement     implementer   Backend Implementer   claude-sonnet-deep
Review        reviewer      Reviewer              opus-deep
Verify        verifier      Verifier              codex-balanced
```

Hide in normal UI:

```text
kind
stage
inputs
outputs
provider args
raw schema names
```

Show in advanced/details only:

```text
kind
stage
inputs
outputs
provider id
profile id
resolved role id
```

Crew Page should let users edit:

```text
Role label
Seats filled
Profile
Permissions
Skills
Prompt
```

Profile UI should let users edit:

```text
Provider
Model
Power
Budget
Max tokens
Timeout
```

Flow Builder should use `Seat`, not `slot`.

## Built-In Defaults

Update the default Crew:

```text
planner fills planner
architect fills architect
executor/backend-implementer fills implementer, executor, builder
fixer fills fixer
reviewer fills reviewer, challenger
verifier fills verifier, arbiter
```

Update the default Flow seats:

```text
planner
architect
implementer
reviewer
fixer
verifier
```

Update quality arbitration seats:

```text
builder
challenger
arbiter
```

The same Crew role can fill multiple seats.

## Documentation Rewrite

Docs must become easier for non-developer, low-patience users.

Every concept page should use this shape:

```text
# Concept Name

## Basically

One or two plain sentences.

## Example

Small concrete example.

## More Detail

The precise model and edge cases.

## Advanced

Schema fields, CLI/API names, and implementation notes.
```

Use plain language first.

Examples:

```text
Profile

Basically:
A Profile is how strong and expensive a role should run.

More detail:
A Profile chooses the provider, model, power, token budget, and timeout.
```

```text
Crew

Basically:
A Crew is your local team of AI roles.

More detail:
Each role has instructions, permissions, skills, a Profile, and a list of Seats
it can fill in a Flow.
```

```text
Seat

Basically:
A Seat is what a Flow step needs filled.

More detail:
The Flow may ask for an implementer seat. Your Crew may fill that seat with a
role named Backend Implementer, Executor, Coder, or anything else.
```

Docs to update:

```text
README.md
docs/content/index.md
docs/content/getting-started/first-run.md
docs/content/getting-started/providers.md
docs/content/cli/dashboard.md
docs/content/concepts/flow.md
docs/content/concepts/role.md
docs/content/concepts/provider.md
docs/content/glossary.md
docs/content/extending/add-flow.md
docs/content/architecture/overview.md
docs/content/architecture/directory-map.md
docs/design/vocabulary.md
docs/design/flows-unification.md
docs/design/crew-flow-authoring.md
```

Generated docs under `docs/generated/*` must be regenerated or updated if this
repo has a generator for them. If no generator exists, update them manually
only where tests or visible docs require it.

## Naming Rules

Use these exact words in user-facing UI/docs:

```text
Flow
Step
Seat
Crew
Role
Profile
Provider
Run
```

Avoid:

```text
agent
job
slot
roleId
provider override
effortMap
assignment
runtime profile
```

Allowed internally only:

```text
kind
stage
inputs
outputs
providerId
profileId
resolvedRoleId
```

## Tests

Add/update tests for:

```text
Project config parses profiles + crews.
Init template creates profiles + default crew.
Flow schema requires seats for agent/review/response/summary turns.
Flow schema does not accept slots/roleId.
Flow resolver maps step.seat -> crew role -> profile -> provider.
Resolver fails on missing seat fill.
Resolver fails on ambiguous seat fill.
Step profile override changes only runtime Profile, not Role behavior.
Orchestrator runs provider from resolved Profile.
Crew API updates role profile and fills.
Flow Builder saves seat, not slot.
Mission Control submits crew/profile data, not slotProviders.
Docs examples match the new YAML shape.
```

Run at minimum:

```bash
pnpm typecheck
pnpm test
```

If full tests are too slow, run targeted tests first, then full typecheck.

## Implementation Order

1. Update schemas and init template.
2. Update built-in flows.
3. Update resolver and resolved snapshots.
4. Update orchestrator/provider execution path.
5. Update CLI/API payloads.
6. Update UI types.
7. Update Mission Control.
8. Update Crew/Profile/Provider pages.
9. Update Flow Builder.
10. Update docs.
11. Update tests.
12. Run verification.

Do not start with UI polish. The data model must compile first.

## Acceptance Criteria

The rewrite is complete when:

```text
No user-facing docs say job.
No user-facing docs say slot.
No normal UI says kind, roleId, or provider override.
Flow YAML uses seats and step.seat.
Project YAML uses profiles and crews.
Roles live inside crews.
Roles point to profiles, not providers.
Profiles point to providers.
Mission Control can resolve Flow + Crew into a Run.
Runs record resolved role/profile/provider per step.
Tests and typecheck pass.
```

## Final Product Story

This is the story users should understand:

```text
Install providers.
Create profiles for how you want models to run.
Create a crew of roles.
Tell each role which seats it can fill.
Pick a flow.
Vibestrate matches the flow's seats to your crew.
Start the run.
```

The lazy-user version:

```text
Flow = the recipe.
Crew = your AI team.
Role = what each teammate does.
Profile = which model/power they use.
Seat = what the recipe needs filled.
```
