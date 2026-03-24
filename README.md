# sidequest

A Claude Code and Codex skill that spawns autonomous side sessions in separate git worktrees and [cmux](https://github.com/anthropics/claude-code) workspaces. Each sidequest gets its own isolated branch, workspace, and focused prompt so you can investigate a few strong approaches in parallel without blocking your main session.

## Usage

### Single sidequest

```
/sidequest <name> [prompt]
```

Spawns one autonomous Claude or Codex session:

```
/sidequest redis-approach
/sidequest redis-approach "implement the redis caching layer from solution 3"
```

- `name` — becomes the branch suffix and workspace label
- `prompt` — what the sidequest should work on. If omitted, the skill composes a prompt from the current conversation context.

### Selective batch mode

```
/sidequest pick [n]
```

Reviews the current conversation, identifies the best distinct sidequests to run in parallel, and shows a shortlist first. Launch only the subset you want. If `n` is provided, the shortlist is capped at that number.

Example output before launch:

| Name | Description |
|------|-------------|
| `redis-cache` | Implement caching layer with Redis TTL invalidation |
| `denormalize` | Flatten the join into a materialized view |
| `lazy-load` | Defer hydration with intersection observer |

If you already know the names you want, the skill should skip the shortlist and launch only those named sidequests.

## Key features

- **Worktree isolation** — each sidequest runs in its own git worktree created or reused via `wtf` / `wtcd`, so parallel sessions never conflict
- **cmux integration** — spawns in a new cmux workspace with its own terminal surface. Falls back to a new Terminal.app window if cmux isn't available
- **Claude and Codex support** — Claude runs with `--permission-mode auto`; Codex runs with `--dangerously-bypass-approvals-and-sandbox` for yolo mode
- **Parent context injection for Claude** — includes a path to the parent session's JSONL so the spawned Claude session can look up additional context if needed
- **Self-contained prompts** — each sidequest gets a rich, standalone prompt with the what, why, where, and constraints from the conversation
- **Parallel execution** — launch multiple sidequests simultaneously; each gets its own branch, workspace, and port assignments

## How it works

1. Creates or reuses a dedicated worktree with your existing shell helpers
2. Creates a cmux workspace in that worktree
3. Gets the terminal surface for that workspace
4. Sends either a `claude` or `codex` command with the composed prompt

The worktree helper handles branch creation, dependency installation, and environment setup.

## Installation

Clone this repo into your Claude skills directory:

```bash
git clone https://github.com/dhruv92/sidequest.git ~/.claude/skills/sidequest
```

Then mirror it into the shared agent skill directories so Codex and other agents can see it too:

```bash
ln -s ../../.claude/skills/sidequest ~/.agents/skills/sidequest
ln -s ../../.claude/skills/sidequest ~/.codex/skills/sidequest
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or Codex with skills support
- [cmux](https://github.com/anthropics/claude-code) (optional — falls back to Terminal.app)
- Git worktree support plus the `wtf` / `wtcd` helpers
