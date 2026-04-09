# Claude Commands

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash commands for everyday dev workflows.

## Install

```bash
claude install github:mManishTrivedi/claude-commands
```

## Commands

### `/git-merge-cleanup`

Delete merged git branches locally and optionally from remote. Protects `main`, `master`, and `release*` by default.

```
/git-merge-cleanup                       # delete all merged local branches
/git-merge-cleanup feature*              # delete merged branches matching a pattern
/git-merge-cleanup --remote              # also delete from remote (origin)
/git-merge-cleanup feature* --remote     # pattern + remote cleanup
```

### `/aws-logs`

Download AWS CloudWatch logs, extract errors, and generate a summary — all via bash (zero AI tokens spent on log parsing). Uses [`awslogs`](https://github.com/jorgebastida/awslogs) for fast, paginated log retrieval.

```
/aws-logs -g /ecs/my-api-prod                         # last 2h of logs + error summary
/aws-logs -g /ecs/my-api-prod -l 5d                   # last 5 days of logs
/aws-logs -g /ecs/my-api-prod -l 1h -p my-profile     # custom AWS profile
/aws-logs -g /aws/lambda/my-func -l 2d --errors-only  # lambda logs, errors only
/aws-logs -g /ecs/my-api-prod -f "OutOfMemory" -l 6h  # download then grep for pattern
/aws-logs -g /ecs/my-api-prod -t                      # live tail (real-time streaming)
/aws-logs -g /ecs/my-api-prod --from '2026-04-03 00:00:00' --to '2026-04-03 23:59:59'  # date range
```

**Options:**

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--log-group` | `-g` | auto-discover | Full CloudWatch log group path |
| `--prefix` | | `/ecs` | Log group prefix for auto-discovery |
| `--last` | `-l` | `2h` | Time window (`30m`, `2h`, `5d`) |
| `--from` / `--to` | | | Date range (`YYYY-MM-DD HH:MM:SS` or ISO 8601) |
| `--profile` | `-p` | `default` | AWS CLI named profile |
| `--region` | `-r` | `us-east-1` | AWS region |
| `--filter` | `-f` | | Grep pattern applied after download |
| `--errors-only` | | | Pipe through error grep (skip raw file — faster) |
| `--raw` | | | Skip error extraction, just download |
| `--tail` / `--watch` | `-t` / `-w` | | Live-stream logs in real time |
| `--streams` | | | Filter to specific log stream(s) |
| `--output` | `-o` | `./aws-logs/` | Output directory |

**Outputs** (saved to `./aws-logs/`):
- `*-raw.log` — full log dump
- `*-errors.log` — filtered error/exception lines
- `*-summary.txt` — error counts, top patterns by frequency, hourly timeline, HTTP 5xx breakdown

**Requires**: [`awslogs`](https://github.com/jorgebastida/awslogs) (`pip3 install awslogs`) + valid AWS credentials/profile

## Adding Your Own Commands

Drop any `.md` file into `commands/` following the [Claude Code custom slash command format](https://docs.anthropic.com/en/docs/claude-code/slash-commands):

```
commands/
├── git-merge-cleanup.md
├── aws-logs.md
└── your-command.md
```

## License

MIT
