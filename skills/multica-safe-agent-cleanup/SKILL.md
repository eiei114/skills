---
name: multica-safe-agent-cleanup
description: Safely clean up unnecessary Multica agents after a roster change. Use when the user wants to archive or remove old agents, clean up stale squads, or rewrite live references from soon-to-be-removed agents. This is a mutating safety-first cleanup skill for Multica agent lifecycle management.
disable-model-invocation: true
---

# Multica Safe Agent Cleanup

Archive or remove unnecessary Multica agents only after checking live references.

## Use when

- The user wants to remove or archive old Multica agents.
- A roster rebuild left obsolete agents behind.
- A squad or issue still references agents that are about to be removed.

## Goal

- Remove unnecessary agents without breaking live Multica state.
- Preserve history while preventing accidental task starts.

## Do

1. Inspect references before mutating anything:
   - `multica agent list --output json`
   - `multica issue list --output json --limit 100`
   - `multica squad list --output json`
   - `multica autopilot list --output json`
2. Identify whether a candidate agent is referenced by:
   - non-done issues
   - active squads
   - active/paused autopilots
3. If a live issue still references the agent, inspect:
   - `multica issue get <id> --output json`
   - `multica issue runs <id> --output json`
4. Prefer safe outcomes in this order:
   - keep as-is if the reference is historical and harmless
   - reassign safely if the user asked to rewrite the reference
   - remove the parent squad first when the only remaining reference is the squad
   - archive the agent only after references are resolved or explicitly accepted
5. After any reassignment, inspect issue runs immediately to detect accidental starts.
6. Verify cleanup with fresh agent/squad/issue inventory.

## Do not

- Do not archive an agent before checking live references.
- Do not assume `issue assign` is passive.
- Do not call `rerun` unless the user explicitly asked to start work.
- Do not trigger autopilots as part of cleanup.

## Mutation boundary

Allowed mutations when explicitly requested by the user:

- `multica issue assign`
- `multica squad delete`
- `multica agent archive`

Use extra care around:

- any issue reassignment on non-done issues
- deleting a squad with active references

## Inputs

- Candidate agents to remove.
- Optional replacement agents.
- User preference on whether to preserve assignee history vs rewrite live references.

## Outputs

- Archived agents list.
- Deleted squads list.
- Reassigned live references list.
- Remaining live references, if any.

## Verification checklist

- No active squad points to a removed agent.
- No autopilot points to a removed agent.
- Any live issue reassignment was followed by a run check.
- Final agent list contains only intended survivors.

## Done when

- All intended unnecessary agents are archived.
- The remaining live references are either rewritten safely or explicitly preserved.
- Verification output is captured.

## Escalate when

- Reassigning a live issue would likely trigger unwanted execution and the safe replacement path is unclear.
- A removed agent is still needed by an active autopilot or squad with no approved replacement.
- The user has not clarified whether historical assignee preservation matters.

## Tool policy

- Use only the `multica` CLI for Multica reads/writes.
- Prefer `--output json`.
- Never print secrets or tokens.

## Reporting format

Always report:

1. Cleanup candidates.
2. Live references found.
3. Mutations applied.
4. Verification summary and any preserved references.
