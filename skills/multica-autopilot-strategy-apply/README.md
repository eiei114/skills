# multica-autopilot-strategy-apply

Apply a desired Multica autopilot strategy from a design note or explicit operating model.

Use this skill when you need to:

- create or update autopilots in bulk
- separate runners, sentinels, seeders, and monitors
- pause overlapping legacy autopilots safely
- rewrite schedules to match a new operating model

## Installation

Install from this repo:

```bash
gh skill install eiei114/skills multica-autopilot-strategy-apply
```

Skill path after install:

```text
.pi/skills/multica-autopilot-strategy-apply/SKILL.md
```

Invoke explicitly:

```text
/skill:multica-autopilot-strategy-apply
```

## Example prompts

- `Apply this operating model to the live Multica autopilots.`
- `Pause the legacy seeders and replace them with the new review sentinel.`
- `Align the schedules for the todo runner and weekly seed planner.`

## Notes

- Designed for strategy-driven autopilot updates.
- Does not manually trigger autopilots unless the user asks for a dry run.
- Pair with [`multica-cli`](../multica-cli/) for inspection before or after rollout.
