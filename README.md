# Claude Commands

A collection of reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash commands.

## Commands

| Command | Description |
|---------|-------------|
| `/git-merge-cleanup` | Delete merged git branches locally & remotely. Protects `main`, `master`, and `release*`. |

## Install

```bash
claude install github:manishtrivedi/claude-commands
```

## Usage

Once installed, the commands are available in any Claude Code session:

```
/git-merge-cleanup                       # delete all merged local branches
/git-merge-cleanup manish*               # delete merged branches matching pattern
/git-merge-cleanup --remote              # also delete from remote
/git-merge-cleanup manish* --remote      # pattern + remote cleanup
```

## Adding More Commands

Drop any `.md` file into the `commands/` directory following the [Claude Code command format](https://docs.anthropic.com/en/docs/claude-code/slash-commands):

```
commands/
└── your-command.md
```

## License

MIT
