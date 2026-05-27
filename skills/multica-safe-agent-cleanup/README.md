# multica-safe-agent-cleanup

Safely archive or remove unnecessary Multica agents after checking live references.

Use this skill when you need to:

- archive obsolete agents after a roster migration
- remove stale squads that still point at old agents
- rewrite live issue references to a replacement agent carefully
- inspect whether cleanup would accidentally trigger new work

## Installation

Install from this repo:

```bash
gh skill install eiei114/skills multica-safe-agent-cleanup
```

Skill path after install:

```text
.pi/skills/multica-safe-agent-cleanup/SKILL.md
```

Invoke explicitly:

```text
/skill:multica-safe-agent-cleanup
```

## Example prompts

- `Safely clean up the obsolete Multica agents.`
- `Check live issue, squad, and autopilot references before archiving anything.`
- `Reassign this archived assignee to a Pi runtime agent without starting unwanted work.`

## Notes

- Safety-first skill.
- Checks issues, squads, and autopilots before archival.
- Treats issue reassignment as execution-triggering and verifies runs immediately after mutation.
