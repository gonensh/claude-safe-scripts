# safe-scripts

A Claude Code plugin that builds a personal library of pre-approved safe scripts to eliminate repetitive permission dialogs.

## How it works

When Claude runs a Bash command that needs approval, the plugin intercepts and offers to save a generalized, parameterized version as a "safe script." Once saved, that script is permanently pre-approved — no more dialogs for that class of command.

## Usage

Once installed, the plugin works automatically. Here's the typical flow:

1. Claude considers running a command like `git log --oneline -10 -- src/App.tsx`
2. The plugin checks your saved scripts — if a match exists, Claude uses it directly (no dialog)
3. If no match, Claude asks: *"I can save this as a safe script `git-file-log` so future runs are auto-approved. Save it, or run once?"*
4. If you choose to save, Claude shows you the generalized script for confirmation, then saves it and runs it — no dialog, now or in future sessions

Saved scripts live in `~/.claude/safe-scripts/` and are automatically added to Claude's allow-list.

## Requirements

- Claude Code with plugin support
- `jq` ≥ 1.5 (`brew install jq` / `apt install jq`)
- bash ≥ 3.2

## Installation

```bash
claude plugin install <repo-url>
```

## Configuration

By default, scripts are saved to `~/.claude/safe-scripts/`.

To override per-project, create `.claude/safe-scripts-config.json`:

```json
{
  "safe_scripts_dir": "./.claude/safe-scripts"
}
```

A project-local directory can be committed to git for team-shared script libraries.

## Supported interpreters

bash, python3, node, perl, go (via bash wrapper), npx/ts-node (via bash wrapper), and inline heredoc scripts for any of the above.

## License

MIT
