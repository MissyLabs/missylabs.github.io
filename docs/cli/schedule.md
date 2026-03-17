---
tags:
  - cli
  - scheduling
---

# Schedule Commands

Manage recurring agent tasks powered by APScheduler.

## missy schedule add

```bash
missy schedule add \
  --name "Daily digest" \
  --schedule "daily at 09:00" \
  --task "Summarise the news" \
  --provider anthropic
```

| Option | Required | Description |
|--------|----------|-------------|
| `--name` | yes | Human-readable job name |
| `--schedule` | yes | Schedule expression (e.g. `"every 5 minutes"`, `"daily at 09:00"`) |
| `--task` | yes | Prompt text to run on each firing |
| `--provider` | no | Provider to use (default: `anthropic`) |

## missy schedule list

```bash
missy schedule list
```

Shows all jobs with ID, name, schedule, provider, enabled status, run count, and next run time.

## missy schedule pause / resume

```bash
missy schedule pause JOB_ID
missy schedule resume JOB_ID
```

## missy schedule remove

```bash
missy schedule remove JOB_ID
```

Prompts for confirmation before removal.
