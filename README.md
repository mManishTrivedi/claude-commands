# Claude Commands

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash commands for everyday dev workflows.

## Install

```bash
claude install github:<your-username>/claude-commands
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

Download AWS CloudWatch logs, extract errors, and generate a summary — all via bash (zero AI tokens spent on log parsing).

```
/aws-logs                                             # auto-discover log groups, last 2h
/aws-logs -g /ecs/my-api-prod -l 5d                   # specific log group, last 5 days
/aws-logs --prefix /ecs -e staging -l 1h              # discover staging groups, last hour
/aws-logs -g /aws/lambda/my-func -l 2d --errors-only  # lambda logs, errors only
/aws-logs -p my-profile -l 12h -f "OutOfMemory"       # custom profile + filter pattern
/aws-logs -g /ecs/my-api-prod -t                      # live tail (real-time streaming)
/aws-logs --from 2025-04-01T00:00:00Z --to 2025-04-02T00:00:00Z  # specific date range
```

**Options:**

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--log-group` | `-g` | auto-discover | Full CloudWatch log group path |
| `--prefix` | | `/ecs` | Log group prefix for auto-discovery |
| `--env` | `-e` | `production` | Environment filter |
| `--last` | `-l` | `2h` | Time window (`30m`, `2h`, `5d`) |
| `--from` / `--to` | | | ISO 8601 date range |
| `--profile` | `-p` | `default` | AWS CLI named profile |
| `--region` | `-r` | `us-east-1` | AWS region |
| `--filter` | `-f` | | CloudWatch filter pattern |
| `--errors-only` | | | Only download error lines (faster) |
| `--raw` | | | Skip error extraction |
| `--tail` | `-t` | | Live-stream logs in real time |
| `--output` | `-o` | `./aws-logs/` | Output directory |

**Outputs** (saved to `./aws-logs/`):
- `*-raw.log` — full log dump
- `*-errors.log` — filtered error/exception lines
- `*-summary.txt` — error counts, top patterns by frequency, hourly timeline, HTTP 5xx breakdown

**Requires**: [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) + valid credentials/profile

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
