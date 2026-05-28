# Flow / Crew / Role Design Memo

Implementation note:

```text
For the actual breaking rewrite, use CLAUDE_CORE_MODEL_REWRITE.md.
This project is beta, so the implementation should replace old slot/roleId
schema with Seat rather than preserving compatibility forever.
```

## Short Decision

Keep the model small.

```text
Task + Flow + Crew = Run
```

Use five product models:

```text
Flow
Step
Crew
Role
Profile
```

Remove the proposed `job` model/field.

Do not add `Assignment` as a product model.

Use **Seat** as the product word for the thing a Flow step needs.

```text
Step requests a Seat.
Role fills one or more Seats.
```

Implementation detail:

```text
Seat is not a new product model.
It maps to today's Flow `slot` / `roleId`.
```

Use `fills` as the small matching field that connects Crew roles to those Seats.

## Why Seat / fills

`job` is a bad name because it sounds like a separate work object.

`Seat` is better because the Flow already has the idea of participant seats.

What we actually need is simpler:

```text
This Flow step needs a seat.
This Crew role can fill that seat.
```

Example:

```text
Backend Implementer fills: executor, implementer, builder
Reviewer fills: reviewer, challenger
Planner fills: planner
```

So when a shared Flow asks for:

```text
slot: executor
roleId: executor
```

the local Crew can answer:

```text
Use Backend Implementer with claude-sonnet-deep.
```

Someone else can answer:

```text
Use Executor with codex-deep.
```

Same Flow. Different local Crew. No forced role names.

## Current Code Model

| Current Thing | What It Means Today | Keep? |
|---|---|---|
| `Flow` | Saved recipe with slots, steps, and optional loop. | Yes |
| `Step` | One phase in the Flow. | Yes |
| `kind` | Engine mode: `agent-turn`, `review-turn`, `validation`, etc. | Yes, internal |
| `slot` | Participant seat in a Flow. | Yes; show as Seat in product UI |
| `roleId` | Role/prompt requested by the Step. | Yes; use as requested Seat when present |
| `stage` | Lifecycle bucket for status/resume. | Yes, internal |
| provider override | Runtime override for slot/step. | Replace with Profile override |

Current example:

```text
id: implement
label: Implement
kind: agent-turn
slot: executor
roleId: executor
stage: executing
```

Current problem:

```text
Flow asks for executor.
Local users may call that role Coder, Builder, Implementer, or Executor.
Today those names do not resolve cleanly.
```

New wording:

```text
Flow asks for Seat: executor.
Local Crew decides which Role fills that Seat.
```

## New Minimal Model

### Flow

Meaning:

```text
The recipe.
```

Owns:

```text
steps
slots
order
loops
human pauses
```

Does not own:

```text
local provider choice
local model choice
local budget
```

### Step

Meaning:

```text
One visible phase in the Flow.
```

Owns:

```text
label
kind
requested Seat
slot / roleId internally
stage
inputs
outputs
optional
approval
```

Example:

```text
label: Implement
kind: agent-turn
seat: executor

internal:
  slot: executor
  roleId: executor
```

No `job` name needed.

The Flow can stay close to the current schema.

### Crew

Meaning:

```text
The local roster.
```

Owns:

```text
roles
which Seats each role fills
which Profile each role uses
```

Example:

```text
Crew: Default

Role                  fills                         profile
----------------------------------------------------------------------
Planner               planner                       codex-balanced
Backend Implementer   executor, implementer, builder claude-sonnet-deep
Reviewer              reviewer, challenger           opus-deep
Verifier              verifier, arbiter              codex-balanced
```

This is the main simplification:

```text
Crew contains Roles.
Each Crew Role has a Profile.
Each Crew Role declares which Seats it fills.
```

### Role

Meaning:

```text
The behavior.
```

Owns:

```text
name
prompt/instructions
permissions
policies
skills
fills
profile
```

Example:

```text
Role: Backend Implementer
fills: executor, implementer, builder
profile: claude-sonnet-deep
permissions: code_write
prompt: .vibestrate/roles/backend-implementer.md
```

Note:

```text
`fills` belongs on the Crew Role, not inside the prompt text.
```

### Profile

Meaning:

```text
The runtime setup.
```

Owns:

```text
provider/tool
model
effort/power
budget
max tokens
timeout
```

Example:

```text
Profile: claude-sonnet-deep
provider: claude
model: sonnet
power: deep
budget: high
```

Profile should stay runtime-focused.

The "can this role fill the executor Seat?" question belongs to the Crew Role
via `fills`.

## Resolution

When a Run starts:

```text
Flow Step asks for a Seat.
Internally, that Seat is backed by slot/roleId.
Crew finds a Role that fills it.
Role points to a Profile.
Profile picks provider/model/budget.
```

Simple version:

```text
Step -> Role -> Profile
```

Non-breaking resolution order:

```text
1. If step.roleId exactly matches a configured role id, use it.
2. Else find a Crew Role where fills includes step.roleId.
3. Else use the step slot's defaultRole exactly.
4. Else find a Crew Role where fills includes the slot id or defaultRole.
5. Else fail with a clear "no role fills this Flow seat" message.
```

Why this is non-breaking:

```text
Existing Flow slot/roleId values stay valid.
Existing project roles keep working.
fills only adds alias support.
```

Default compatibility:

```text
role id: executor
fills: [executor]
```

So a current Flow that asks for `roleId: executor` still resolves to
`executor` exactly.

Example:

```text
Flow Step:
  label: Implement
  seat: executor

Internal:
  slot: executor
  roleId: executor

Crew resolves:
  Backend Implementer
  profile: claude-sonnet-deep

Run executes:
  Implement -> Backend Implementer -> Claude Sonnet / deep
```

## Same Role, Different Profile

Sometimes two steps should use the same behavior with different runtime power.

Do this with a step override:

```text
Step                         Role                  Profile
----------------------------------------------------------------
Implement config parser      Backend Implementer   codex-balanced
Implement auth crypto        Backend Implementer   opus-deep
```

Do not create separate behavior roles like:

```text
Backend Implementer Fast
Backend Implementer Deep
Backend Implementer Opus
```

That mixes behavior with runtime.

## Mission Control UI

Normal UI:

```text
Flow: Test-first feature

Plan          Planner              codex-balanced
Implement     Backend Implementer  claude-sonnet-deep
Review        Reviewer             opus-deep
Verify        Verifier             codex-balanced
```

Advanced details:

```text
requested seat: executor
step.kind: agent-turn
step.slot: executor
step.roleId: executor
resolved role: Backend Implementer
resolved profile: claude-sonnet-deep
```

## Migration

Keep:

```text
Flow
Step
kind
slot
roleId
stage
inputs / outputs
loop
approval-gate
```

Add:

```text
Profile
Crew roles
Crew role fills Seats
Crew role profile
```

Change:

```text
roles.<id>.provider -> roles.<id>.profile
provider override -> profile override
```

Compatibility:

```text
If role.profile is missing, use old role.provider.
If role.fills is missing, default it to [role id].
If UI says Seat, store it as current slot/roleId for now.
```

That means existing roles still work:

```text
executor:
  provider: claude
```

becomes:

```text
executor:
  fills: [executor]
  profile: claude-default
```

## Build Order

1. Add `Profile`.
2. Add optional `fills` to roles.
3. Add optional `profile` to roles while keeping old `provider`.
4. Resolve Step `slot` / `roleId` through Crew role `fills`.
5. Change Crew UI to show `Role -> Seats filled -> Profile`.
6. Show Seat in normal UI; hide raw `kind`, `slot`, and `roleId`.
7. Add step-level Profile override for special cases.

## Final Rule

```text
Flow says the recipe.
Step says the phase and requested seat.
Crew contains the local roles.
Role says behavior and what Seats it fills.
Profile says runtime.
kind stays internal.
```
