---
name: sidequest
version: 2.1.0
description: |
  Spawn new Claude sessions in separate worktrees and cmux workspaces.
  Single: `/sidequest <name> [prompt]`. Batch: `/sidequest all` to auto-detect
  all distinct solutions/approaches in the conversation and spawn each one.
allowed-tools:
  - Bash
---

# Sidequest

Spawn a new Claude Code session in its own git worktree and cmux workspace. Each sidequest gets a fresh session with a focused prompt containing the relevant context from the current conversation.

## Parent session context

The parent conversation is stored at:

- Parent session: !`ls -t ~/.claude/projects/$(echo "$(pwd)" | sed 's|/|-|g')/*.jsonl 2>/dev/null | head -1`

When composing the sidequest prompt, append this line so the spawned session can look up the parent conversation if it needs more context:

> If you need more context about why this work was requested, the parent conversation (JSONL) is at `<path from above>`.

## Modes

### Single: `/sidequest <name> [prompt]`

- `name` — branch suffix and workspace label (e.g. `redis-approach`, `minimal-fix`)
- `prompt` — what the sidequest should work on. If not provided, you (Claude) should compose a rich prompt from the current conversation context.

Examples:
- `/sidequest solution-1`
- `/sidequest redis-approach "implement solution 3 — the redis caching approach"`

### Batch: `/sidequest all`

Review the current conversation and identify all distinct solutions, approaches, or directions discussed. For each one, compose:
- A short descriptive kebab-case name (e.g. `redis-cache`, `lazy-load`, `denormalize`)
- A detailed self-contained prompt with enough context that a fresh Claude session can work independently

Before launching, show the user the list of sidequests (name + one-line description) and ask for confirmation.

## Implementation

For each sidequest, run these commands:

```bash
# 1. Create cmux workspace in the current directory
WORKSPACE=$(cmux new-workspace --cwd "$(pwd)" | awk '{print $2}')

# 2. Get the terminal surface
SURFACE=$(cmux list-pane-surfaces --workspace $WORKSPACE | awk '{print $2}' | head -1)

# 3. Send the claude command (MUST pass both --workspace and --surface)
cmux send --workspace $WORKSPACE --surface $SURFACE "claude --permission-mode auto -w <name> -n '⚔️ <name>' '<prompt>'"
cmux send-key --workspace $WORKSPACE --surface $SURFACE enter
```

Key details:
- `--permission-mode auto` makes the sidequest autonomous — it won't block waiting for approval on every tool call. Safe because it's in an isolated worktree.
- `-w <name>` creates a git worktree via the WorktreeCreate hook (uses `~/code/worktrees/` convention, runs doppler + yarn install for escher)
- `-n '⚔️ <name>'` labels the session for `/resume`
- The prompt is the positional argument — shell-escape it properly
- Do NOT use `--tmux` (creates nested tmux inside cmux)
- Do NOT use `--fork-session` (can't fork a live session)
- Multiple sidequests can be launched in parallel

### Composing the prompt

Since sidequests are fresh sessions (no conversation history), the prompt must be self-contained. Include:
- What to do (the specific approach/solution to implement)
- Why (the problem being solved)
- Where (key files/paths to look at)
- Any constraints or preferences discussed in the conversation
- The parent session path (see "Parent session context" above) so it can look up more context if needed

### Fallback (no cmux)

If `CMUX_SOCKET_PATH` env var is not set, fall back to raw `claude`:
```bash
osascript -e 'tell app "Terminal" to do script "cd $(pwd) && claude --permission-mode auto -w <name> -n \"⚔️ <name>\" \"<prompt>\""'
```

After launching, confirm with a summary table of name + prompt.
