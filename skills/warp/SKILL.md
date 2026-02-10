---
name: oz
description: Use Warp's REST API and command line to run, configure, and inspect Oz cloud agents
---

# oz-platform

Use the Oz REST API and CLI to:
* Spawn cloud agents
* Get the status of a cloud agent
* Schedule cloud agents to run repeatedly
* Create and manage the environments in which cloud agents run
* Provide secrets for cloud agents to use

## Command Line

The Oz CLI is installed as `oz`. To get help output, use `oz help` or `oz help <subcommand>`. Prefer `--output-format text` to review the response, or `--output-format json` to parse fields with jq. You can find more information at https://docs.warp.dev/platform/cli.

The most important commands are:
* `oz agent run-cloud`: Spawn a new cloud agent. You can configure the prompt, model, environment, and other settings.
* `oz run list` and `oz run get <run-id>`: List all cloud agent runs, and get details about a particular run. This includes the session link to view that session in the cloud.
* `oz environment list` and `oz environment get`: List available environments, and get more information about a particular environment.
* `oz schedule list` and `oz schedule get`: List scheduled tasks with most recent runs, and get more information about a particular scheduled run.

Most subcommands support the `--output-format json` flag to produce JSON output, which you can pipe into `jq` or other commands.

### Examples

Start a cloud agent, and then monitor its status:

```sh
$ oz agent run-cloud --prompt "Update the login error to be more specific" --environment UA17BXYZ
# ...
Spawned agent with run ID: 5972cca4-a410-42af-930a-e56bc23e07ac
```

```sh
$ oz run get 5972cca4-a410-42af-930a-e56bc23e07ac
# ...
```

Schedule an agent to summarize feedback every day at 8am UTC:

```sh
$ oz schedule create --name "daily-feedback-summary" --cron "0 8 * * *" \
    --prompt "Collect all feedback from new GitHub issues and provide a summary report" \
    --environment UA17BXYZ
```

Create a secret

```sh
$ oz secret create JIRA_API_KEY --team --value-file jira_key.txt --description "API key to access Jira"
```

## REST API
Oz has a REST API for starting and inspecting cloud agents.

All API requests require authentication using an API key. The user can generate API keys in their Warp settings, on the `Platform` page (accessible via `{{warp_url_scheme}}://settings/platform`).

You can find the full OpenAPI specification here: https://docs.warp.dev/platform/agent-api-and-sdk/agent.md

In addition, there are SDKs for:
* TypeScript and JavaScript: https://www.npmjs.com/package/warp-agent-sdk
* Python: https://pypi.org/project/warp-agent-sdk/

All SDKs have sync and async support, and documentation at the links above.

### API Examples

```sh
curl -L -X POST {{warp_server_url}}/api/v1/agent/run \
    --header 'Authorization: Bearer YOUR_API_KEY' \
    --header 'Content-Type: application/json' \
    --data '{
        "prompt": "Update the login error to be more specific",
        "config": {
            "environment_id": "UA17BXYZ"
        }
    }'
```

```sh
curl -L -X GET {{warp_server_url}}/api/v1/agent/runs/5972cca4-a410-42af-930a-e56bc23e07ac \
    --header 'Authorization: Bearer YOUR_API_KEY' \
    --header 'Content-Type: application/json'
```

## GitHub Actions Integration

You can trigger Oz cloud agents from GitHub Actions workflows. This enables automation like:
* Triaging issues when they're created or labeled
* Running checks on pull requests
* Scheduling periodic tasks via workflow dispatch

### Action Setup

Use `warpdotdev/warp-agent-action@v1` in your workflow. Required inputs:
* `prompt`: The task description for the agent
* `warp_api_key`: API key (store in GitHub secrets, e.g., `${{ secrets.WARP_API_KEY }}`)
* `profile`: Optional agent profile identifier (can use repo variable, e.g., `${{ vars.WARP_AGENT_PROFILE || '' }}`)

The action outputs `agent_output` with the agent's response.

### Minimal Workflow Example

```yaml
name: Run Oz Agent
on:
  issues:
    types: [opened, labeled]

jobs:
  agent:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: warpdotdev/warp-agent-action@v1
        id: agent
        with:
          prompt: |
            Analyze the GitHub issue and provide a summary.
            Issue: ${{ github.event.issue.title }}
            ${{ github.event.issue.body }}
          warp_api_key: ${{ secrets.WARP_API_KEY }}
          profile: ${{ vars.WARP_AGENT_PROFILE || '' }}
      - name: Use Agent Output
        run: echo "${{ steps.agent.outputs.agent_output }}"
```

### Common Patterns

**Conditional steps**: Use `if: steps.agent.outputs.agent_output` to branch on agent results.

**Templating**: Use `actions/github-script@v7` to construct dynamic prompts from issue templates, repo context, or code.

**Error handling**: Check action success with `if: success()` or `if: failure()`.

**Git operations**: The action runs with checked-out code and Git credentials, so agents can commit and push changes.

## Environments

All cloud agents run in an environment. The environment defines:
* Which programs are preinstalled for the agent (based on a Docker image)
* The Git repositories to check out before the agent starts
* Setup commands to run, such as `npm install` or `cargo fetch`

You should almost always run cloud agents in an environment. Otherwise, they may not have the necessary code or tools available.

Cloud agents run in a sandbox, so they _can_ install additional programs into their environment. They also have Git credentials to create PRs and push branches.

Cloud environments DO NOT store secret values, like API keys. Use the `oz secret` commands instead.

### Listing and Finding Environments

List all available environments with metadata:

```sh
$ oz environment list --output-format json
```

Get details about a specific environment by ID:

```sh
$ oz environment get <ENVIRONMENT_ID> --output-format json
```

Use `jq` to filter and extract information, e.g., find environment by name:

```sh
$ oz environment list --output-format json | jq '.[] | select(.name == "Full stack") | .id'
```

### Updating Environments

Modify an environment using `oz environment update <ID>`. Common options:
* `-r, --repo owner/repo`: Add a Git repository (can be specified multiple times)
* `--remove-repo owner/repo`: Remove a Git repository
* `-c, --setup-command "command"`: Add a setup command
* `--remove-setup-command "command"`: Remove a setup command
* `-d, --docker-image IMAGE`: Change the Docker image
* `-n, --name NAME`: Rename the environment
* `--description TEXT`: Set a description

Example: Add a repository to an environment:

```sh
$ oz environment update SGY5YoiC4lWZLA0N0lu6OT -r warpdotdev/gitbook
```

### Creating Environments

For detailed guidance on creating environments with `oz environment create`, see [create-environment.md](./create-environment.md). This includes:
* Repository detection and analysis
* Docker image selection (Warp pre-built images or custom)
* Setup command determination
* Full workflow with mandatory confirmation points

## Schedules

Schedules allow cloud agents to run automatically on a recurring basis using cron expressions.

### Creating Schedules

Use `oz schedule create` with these **required** parameters:
* `--name <NAME>`: Unique identifier for the schedule
* `--cron <CRON>`: Cron expression (e.g., `"0 9 * * 1"` for Mondays at 9am UTC)
* `--environment <ID>`: Environment ID to run in
* `--prompt <PROMPT>` or `--skill <SPEC>`: Task for the agent

```sh
$ oz schedule create --name "weekly-cleanup" --cron "0 9 * * 1" \
    --environment iS2uiErC9g4GoyUq4z6kQA \
    --prompt "Clean up stale branches older than 30 days"
```

Common cron patterns:
* `"0 9 * * *"` - Daily at 9am UTC
* `"0 9 * * 1"` - Weekly on Monday at 9am UTC
* `"0 0 1 * *"` - Monthly on the 1st at midnight UTC

### Listing and Getting Schedules

```sh
$ oz schedule list --output-format text
$ oz schedule get <SCHEDULE_ID> --output-format text
```

### Updating Schedules

Modify an existing schedule with `oz schedule update <SCHEDULE_ID>`:

```sh
# Change the cron schedule
$ oz schedule update <SCHEDULE_ID> --cron "0 10 * * *"

# Update the prompt
$ oz schedule update <SCHEDULE_ID> --prompt "New task description"

# Pause a schedule
$ oz schedule update <SCHEDULE_ID> --paused

# Resume a paused schedule
$ oz schedule update <SCHEDULE_ID> --no-paused
```

### Deleting Schedules

```sh
$ oz schedule delete <SCHEDULE_ID>
```
