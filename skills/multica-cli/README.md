# Multica CLI Skill

Pi Agent Skill for operating the `multica` CLI.

Use it for:

- Multica setup/auth/daemon checks
- workspace/runtime/agent/project/issue inventory
- issue creation and updates
- assigning or rerunning agent work with confirmation
- Multica skill/autopilot inspection and management
- handing off focused bulk operations to related Multica setup skills

## Installation

Install from this repo:

```bash
gh skill install eiei114/skills multica-cli
```

Skill path after install:

```text
.pi/skills/multica-cli/SKILL.md
```

Invoke explicitly:

```text
/skill:multica-cli
```

## Related skills

- [`multica-agent-roster-apply`](../multica-agent-roster-apply/) — create/update a full agent roster from a strategy doc
- [`multica-autopilot-strategy-apply`](../multica-autopilot-strategy-apply/) — create/update/pause autopilots from an operating model
- [`multica-safe-agent-cleanup`](../multica-safe-agent-cleanup/) — archive or rewrite obsolete agent references safely

## Example prompts

- `Show me the current Multica agent roster.`
- `Inspect the runs for DOT-33.`
- `List the PR monitor autopilots.`

