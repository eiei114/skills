# multica-agent-roster-apply

Apply a desired Multica agent roster from a strategy note or explicit roster spec.

Use this skill when you need to:

- create missing Multica agents
- update runtime/model/instructions in bulk
- normalize a workspace onto Pi-family runtimes
- apply skill assignments consistently across agents

## Installation

Install from this repo:

```bash
gh skill install eiei114/skills multica-agent-roster-apply
```

Skill path after install:

```text
.pi/skills/multica-agent-roster-apply/SKILL.md
```

Invoke explicitly:

```text
/skill:multica-agent-roster-apply
```

## Example prompts

- `Apply this strategy note to the live Multica agent roster.`
- `Migrate the Claude-oriented agents onto Pi runtimes.`
- `Create any missing agents and align their skill assignments.`

## Notes

- Focuses on roster application only.
- Does not archive/delete old agents by default.
- Pair with [`multica-safe-agent-cleanup`](../multica-safe-agent-cleanup/) for post-migration cleanup.
