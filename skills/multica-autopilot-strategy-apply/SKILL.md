---
name: multica-autopilot-strategy-apply
description: Apply a Multica autopilot strategy from a design note or explicit operating model. Use when the user wants to create, update, pause, replace, or reschedule Multica autopilots such as runners, signal monitors, sentinels, or seed planners. This is a mutating setup skill for Multica autopilot configuration.
disable-model-invocation: true
---

# Multica Autopilot Strategy Apply

Apply a desired Multica autopilot operating model safely.

## Use when

- The user wants to rebuild Multica autopilots from a strategy doc.
- The user wants to separate seeding, monitoring, and execution responsibilities.
- The user wants to pause legacy autopilots and replace them with new ones.
- The user wants schedules or descriptions rewritten to match a new operating model.

## Goal

- Align live Multica autopilots with the desired strategy.
- Prevent overlapping or conflicting autopilots.
- Leave a verified title/id/status/assignee/schedule summary.

## Do

1. Read the target strategy before mutating anything.
2. Probe current state:
   - `multica autopilot list --output json`
   - `multica autopilot get <id> --output json` for targets and likely overlaps
   - `multica agent list --output json` when agent resolution is needed
3. Classify desired autopilots by role:
   - signal monitor
   - scheduled task seeder
   - todo runner
   - review sentinel
   - weekly seed planner
4. Update existing autopilots in place when the title continuity is clear.
5. Create missing autopilots.
6. Pause or retire overlapping legacy autopilots when the new strategy supersedes them.
7. Apply or update schedule triggers with explicit cron + timezone.
8. Verify with:
   - `multica autopilot list --output json`
   - `multica autopilot get <id> --output json`

## Do not

- Do not manually trigger autopilots as part of setup unless the user explicitly asks for a dry run.
- Do not assign issues or rerun issue tasks from this skill.
- Do not mix external signal monitoring with execution unless the strategy explicitly says so.
- Do not leave old overlapping autopilots active when they can create duplicate work.

## Mutation boundary

Allowed mutations:

- `multica autopilot create`
- `multica autopilot update`
- `multica autopilot trigger-add`
- `multica autopilot trigger-update`
- pausing superseded autopilots

Not part of this skill by default:

- `multica autopilot trigger`
- issue assignment
- issue rerun

## Inputs

- An autopilot strategy note or explicit desired autopilot table.
- Desired assignee agents.
- Desired schedules.
- Desired controller/metadata rules.

## Outputs

- Created autopilots list.
- Updated autopilots list.
- Paused superseded autopilots list.
- Final autopilot snapshot: title, id, status, assignee, schedule.

## Verification checklist

- Every intended autopilot exists.
- Superseded overlapping autopilots are paused.
- Schedules match the intended cron/timezone.
- Assignee agents are valid.
- No manual triggers were run unless explicitly requested.

## Done when

- The live autopilot list matches the target strategy.
- Verification output is captured.

## Escalate when

- Two live autopilots overlap but the strategy does not clearly supersede one.
- A trigger exists but changing it would disrupt an active experiment the user may care about.
- The desired controller issue or referenced assignee agent is missing.

## Tool policy

- Use only the `multica` CLI for Multica reads/writes.
- Prefer `--output json`.
- Never print secrets or tokens.

## Reporting format

Always report:

1. Strategy inputs used.
2. Autopilots created / updated / paused.
3. Final active autopilots and schedules.
4. Any unresolved overlap or ambiguity.
