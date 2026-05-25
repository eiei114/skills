---
name: multica-cli
description: Operate the Multica CLI for human + agent project management. Use when the user wants to set up Multica, inspect workspaces/runtimes/agents/issues/projects, create or update Multica issues, assign work to agents, rerun/cancel agent tasks, manage skills, or inspect/autopilot Multica workflows.
license: MIT
metadata:
  source-docs:
    - https://multica.ai/docs/cli
---

# Multica CLI

Use the `multica` CLI as the source of truth. Prefer CLI commands over direct HTTP calls to the Multica API.

## Safety model

Classify the requested operation before running commands.

### Read-only: run without asking again

- `multica version`, `multica config show`, `multica auth status`
- `multica daemon status`, `multica daemon logs`, `multica daemon disk-usage`
- `multica workspace list/get`, `multica runtime list/usage/activity`
- `multica issue list/get/search/runs/run-messages`, `multica issue comment list`
- `multica agent list/get/tasks`, `multica project list/get`, `multica skill list/get`, `multica autopilot list/get/runs`

Use `--output json` when available. For large JSON, summarize the relevant fields instead of pasting everything.

### Mutating: require explicit user intent

Proceed if the user clearly asked for the mutation. Otherwise ask a short confirmation.

- Setup/login/logout/config changes: `setup`, `login`, `auth logout`, `config set`
- Creating/updating/deleting/archive/restore of workspaces, agents, projects, labels, skills, squads, autopilots
- Issue changes: create/update/status/assign/comment/label/metadata/subscriber
- Daemon lifecycle: start/stop/restart/update

### Execution-triggering: require clear target + confirmation

These can start or interrupt agent work. Always inspect the target first, summarize, then confirm unless the user explicitly named the exact target and action.

- `multica issue assign <id> --agent ...`
- `multica issue rerun <id>`
- `multica issue cancel-task <id>`
- `multica autopilot trigger <id>`

Never print full PATs or secret environment values. If command output includes a token, mask it.

## Initial probe

When starting a Multica task, run only the probes needed for the request:

```powershell
multica version
multica config show
multica auth status
multica daemon status
```

If output says `No server configured`, use:

```powershell
multica setup cloud
```

For self-hosted setups, ask for `server-url` and `app-url`, then use:

```powershell
multica setup self-host --server-url <api-url> --app-url <app-url>
```

If setup opens a browser and waits, tell the user to complete authentication and then poll the running command.

## Common inventory commands

```powershell
multica workspace list --output json
multica runtime list --output json
multica agent list --output json
multica project list --output json
multica issue list --output json --limit 20
multica skill list --output json
multica autopilot list --output json
```

Summarize counts and names/statuses:

- workspace slug/name
- daemon running/stopped
- runtime provider/name/status
- agent name/status/runtime/provider
- project title/status/issue count
- issue identifier/title/status/assignee/priority
- autopilot title/status/last run

## Issue workflows

### List and inspect

```powershell
multica issue list --output json --limit 20
multica issue list --status todo --output json
multica issue search "README" --output json
multica issue get DOT-33 --output json
multica issue comment list DOT-33 --output json
multica issue runs DOT-33 --output json
```

Issue IDs may be issue keys like `DOT-33` or UUIDs.

### Create an issue

For short descriptions:

```powershell
multica issue create --title "Title" --description "Short description" --priority medium --status backlog --output json
```

For multi-line or Japanese descriptions on Windows, prefer a UTF-8 file:

```powershell
Set-Content -Path .pi/tmp/multica-issue.md -Value $Body -Encoding UTF8
multica issue create --title "Title" --description-file .pi/tmp/multica-issue.md --project <project-id> --status backlog --output json
```

After creating, report the issue identifier and title.

### Update status or assignment

Inspect first:

```powershell
multica issue get DOT-33 --output json
```

Then mutate:

```powershell
multica issue status DOT-33 --set todo --output json
multica issue assign DOT-33 --agent "Multica Helper" --output json
```

Assignment may trigger work depending on workspace rules. Treat it as execution-triggering.

### Run or monitor agent work

Before rerun, inspect current issue and current assignee:

```powershell
multica issue get DOT-33 --output json
multica issue runs DOT-33 --output json
```

Start a fresh task for the current agent assignment:

```powershell
multica issue rerun DOT-33 --output json
```

Monitor:

```powershell
multica issue runs DOT-33 --output json
multica issue run-messages <run-id> --output json
multica agent tasks "Multica Helper" --output json
```

Cancel only with explicit confirmation:

```powershell
multica issue cancel-task DOT-33 --output json
```

## Agent workflows

List runtimes before creating an agent:

```powershell
multica runtime list --output json
```

Create agent:

```powershell
multica agent create --name "Agent Name" --runtime-id <runtime-id> --description "..." --instructions "..." --model "..." --visibility workspace --output json
```

Prefer `--custom-env-file` or `--custom-env-stdin` for secrets; never put real secrets in shell history.

Useful commands:

```powershell
multica agent get <slug-or-name> --output json
multica agent update <slug-or-name> ... --output json
multica agent tasks <slug-or-name> --output json
multica agent skills <command> ...
multica agent archive <slug-or-name> --output json
```

## Project workflows

```powershell
multica project list --output json
multica project get <project-id> --output json
multica project create --title "Project title" --description "..." --output json
multica project status <project-id> --set planned --output json
multica project resource <command> ...
```

When creating issues for an existing project, pass `--project <project-id>`.

## Skill workflows

Multica skills are workspace resources, separate from local Pi Agent Skills. Avoid confusing them.

```powershell
multica skill list --output json
multica skill get <skill-id> --output json
multica skill create --name "skill-name" --description "..." --content "..." --output json
multica skill import <url> --output json
multica skill files <command> ...
```

Attach/detach skills through `multica agent skills ...`.

## Autopilot workflows

```powershell
multica autopilot list --output json
multica autopilot get <autopilot-id> --output json
multica autopilot runs <autopilot-id> --output json
multica autopilot trigger <autopilot-id> --output json
multica autopilot trigger-add <autopilot-id> ...
```

Manual trigger can create issues or run agents. Treat it as execution-triggering.

## Reporting format

For each operation, report:

1. command category: read-only / mutating / execution-triggering
2. command(s) run, redacting tokens/secrets
3. result summary: identifiers, statuses, counts, next action
4. any blocked step, especially browser auth or daemon stopped

Keep user-facing output concise. Include exact issue keys, agent names, runtime names, and project IDs when relevant.

