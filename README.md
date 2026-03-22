# sidequest

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that spawns autonomous Claude sessions in separate git worktrees and [cmux](https://github.com/anthropics/claude-code) workspaces. Each sidequest gets its own isolated branch, workspace, and focused prompt — letting you explore multiple approaches in parallel without blocking your main session.

## Usage

### Single sidequest

```
/sidequest <name> [prompt]
```

Spawns one autonomous Claude session:

```
/sidequest redis-approach
/sidequest redis-approach "implement the redis caching layer from solution 3"
```

- `name` — becomes the branch suffix and workspace label
- `prompt` — what the sidequest should work on. If omitted, Claude composes a prompt from the current conversation context.

### Batch mode

```
/sidequest all
```

Reviews the current conversation, identifies all distinct solutions or approaches discussed, and spawns a sidequest for each one. Shows you the list for confirmation before launching.

Example output before launch:

| Name | Description |
|------|-------------|
| `redis-cache` | Implement caching layer with Redis TTL invalidation |
| `denormalize` | Flatten the join into a materialized view |
| `lazy-load` | Defer hydration with intersection observer |

## Key features

- **Worktree isolation** — each sidequest runs in its own git worktree (`-w <name>`), so parallel sessions never conflict
- **cmux integration** — spawns in a new cmux workspace with its own terminal surface. Falls back to a new Terminal.app window if cmux isn't available
- **Autonomous mode** — runs with `--permission-mode auto` so sidequests don't block on tool approvals. Safe because they're isolated in worktrees
- **Parent context injection** — includes a path to the parent session's JSONL so the spawned session can look up additional context with a `!command` if needed
- **Self-contained prompts** — each sidequest gets a rich, standalone prompt with the what, why, where, and constraints from the conversation
- **Parallel execution** — launch multiple sidequests simultaneously; each gets its own branch, workspace, and port assignments

## How it works

1. Creates a cmux workspace in the current directory
2. Gets the terminal surface for that workspace
3. Sends a `claude` command with `--permission-mode auto`, `-w <name>` (worktree), and the composed prompt
4. The worktree hook handles branch creation, dependency installation, and environment setup

## Installation

Clone this repo into your Claude Code skills directory:

```bash
git clone https://github.com/dhruv92/sidequest.git ~/.claude/skills/sidequest
```

Or if you manage skills manually, copy `SKILL.md` to `~/.claude/skills/sidequest/`.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with skills support
- [cmux](https://github.com/anthropics/claude-code) (optional — falls back to Terminal.app)
- Git worktree support (standard git)
