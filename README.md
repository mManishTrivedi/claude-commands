# Claude Commands

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash commands.

## Commands

| Command | Description |
|---------|-------------|
| `/git-merge-cleanup` | Delete merged git branches locally & remotely. Protects `main`, `master`, and `release*`. |
| `/aws-logs` | Download AWS CloudWatch logs, extract errors, and generate a summary — all via bash (zero AI tokens on log parsing). |

## Install

```bash
claude install github:manishtrivedi/claude-commands
```

## Usage

Once installed, the commands are available in any Claude Code session:

### `/git-merge-cleanup`

```
/git-merge-cleanup                       # delete all merged local branches
/git-merge-cleanup manish*               # delete merged branches matching pattern
/git-merge-cleanup --remote              # also delete from remote
/git-merge-cleanup manish* --remote      # pattern + remote cleanup
```

### `/aws-logs`

```
/aws-logs                                        # last 2h of production API logs + error summary
/aws-logs -s api -l 5d                           # last 5 days of API logs
/aws-logs -s store -e staging -l 1h              # staging store logs, last hour
/aws-logs -s webapp -l 2d --errors-only          # only errors, last 2 days
/aws-logs -p my-profile -l 12h -f "OutOfMemory"  # filter for specific pattern
/aws-logs -s api -t                              # live tail (real-time streaming)
/aws-logs --from 2025-04-01T00:00:00Z --to 2025-04-02T00:00:00Z  # specific date range
```

**Outputs** (saved to `./aws-logs/`):
- `*-raw.log` — full log dump
- `*-errors.log` — filtered error lines
- `*-summary.txt` — error counts, top patterns, timeline, HTTP 5xx breakdown

**Requires**: AWS CLI installed + valid credentials/profile

## Adding More Commands

Drop any `.md` file into the `commands/` directory following the [Claude Code command format](https://docs.anthropic.com/en/docs/claude-code/slash-commands):

```
commands/
└── your-command.md
```

## License

MIT
