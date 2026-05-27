---
name: multica-agent-roster-apply
description: Apply a Multica agent roster from a strategy note or explicit spec. Use when the user wants to create, rename, retarget runtime/model, or re-skill Multica agents in bulk from a defined roster. This is a mutating setup skill for Multica agent configuration.
disable-model-invocation: true
---

# Multica Agent Roster Apply

Apply a desired Multica agent roster safely and deterministically.

## Use when

- The user wants to rebuild Multica agents from a strategy document.
- The user wants to standardize runtime/model choices across agents.
- The user wants to create missing agents, rename existing ones, or update descriptions/instructions/skill assignments in bulk.

## Goal

- Bring the live Multica agent roster into the desired state.
- Minimize accidental churn.
- Leave a clear created/updated/unchanged summary.

## Do

1. Read the desired roster spec first.
2. Probe only what is needed:
   - `multica runtime list --output json`
   - `multica agent list --output json`
   - `multica skill list --output json`
3. Resolve the target runtime for each agent before mutating anything.
4. Diff desired agents against existing agents by name first, then by obvious continuity clues.
5. Create missing agents.
6. Update existing agents when any of these drift:
   - name
   - description
   - instructions
   - runtime
   - model
   - visibility
7. Set skill assignments explicitly with `multica agent skills set` when the desired roster includes skills.
8. Verify the final roster with `multica agent list --output json` and `multica agent skills list <agent-id> --output json`.

## Do not

- Do not archive or delete agents unless the user explicitly asked for cleanup too.
- Do not trigger issue execution.
- Do not assign issues or rerun tasks as part of roster application.
- Do not assume archived or legacy agents are safe to remove without reference checks.

## Mutation boundary

Allowed mutations:

- `multica agent create`
- `multica agent update`
- `multica agent skills set`

Not part of this skill by default:

- `multica agent archive`
- issue assignment
- issue rerun
- autopilot trigger

## Inputs

- A strategy summary, roster note, or explicit agent table.
- Desired runtime policy.
- Desired model policy.
- Optional desired skill mapping.

## Outputs

- Created agents list.
- Updated agents list.
- Unchanged agents list when relevant.
- Final roster snapshot: name, id, runtime, model, status.

## Verification checklist

- Every intended agent exists.
- Every intended agent points to the intended runtime.
- Every intended agent has the intended model string.
- Every intended agent has the intended skill set.
- No unexpected execution-triggering commands were run.

## Done when

- The live Multica roster matches the desired roster spec.
- Verification output is captured.

## Escalate when

- The required runtime is missing or offline.
- A rename cannot be matched safely to an existing agent.
- Live issues depend on an agent that appears to need replacement rather than update.
- The user also wants archival cleanup of old agents. In that case, hand off to `multica-safe-agent-cleanup` after roster application.

## Tool policy

- Use only the `multica` CLI for Multica reads/writes.
- Prefer `--output json`.
- Never print secrets or tokens.

## Reporting format

Always report:

1. Inputs used for the roster decision.
2. Commands category: read-only vs mutating.
3. Created / updated / unchanged agents.
4. Final verification summary.
