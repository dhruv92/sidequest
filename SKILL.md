---
name: sidequest
version: 3.1.0
description: |
  Spawn new Claude or Codex sessions in separate worktrees and cmux workspaces.
  Use `/sidequest <name> [prompt]` for one focused sidequest. If the user's
  request naturally splits into a few independent investigations, launch a few
  sidequests in parallel. Keep the number small and intentional.
allowed-tools:
  - Bash
---

# Sidequest

Spawn a new Claude Code or Codex session in its own git worktree and cmux workspace. Each sidequest gets a fresh session with a focused prompt containing the relevant context from the current conversation.

## Parent session context

If you are running this skill from Claude Code, the parent conversation is stored at:

- Parent session: !`ls -t ~/.claude/projects/$(echo "$(pwd)" | sed 's|/|-|g')/*.jsonl 2>/dev/null | head -1`

When composing the prompt for a Claude sidequest, append this line so the spawned session can look up the parent conversation if it needs more context:

> If you need more context about why this work was requested, the parent conversation (JSONL) is at `<path from above>`.

If you are running this skill from Codex, do not rely on Codex session storage paths. Make the sidequest prompt fully self-contained instead.

## Modes

### Single: `/sidequest <name> [prompt]`

- `name` — branch suffix and workspace label (e.g. `redis-approach`, `minimal-fix`)
- `prompt` — what the sidequest should work on. If not provided, you (Claude) should compose a rich prompt from the current conversation context.

Examples:
- `/sidequest solution-1`
- `/sidequest redis-approach "implement solution 3 — the redis caching approach"`

### Multiple sidequests

If the user's request naturally decomposes into a few independent solutions, approaches, or investigative threads, it is good to launch multiple sidequests in parallel. For each sidequest, compose:
- A short descriptive kebab-case name (e.g. `redis-cache`, `lazy-load`, `denormalize`)
- A detailed self-contained prompt with enough context that a fresh Claude session can work independently

Guidelines:
- Keep the number small. Usually `2-4` sidequests is the right range.
- If the user already named the investigations they want, launch those directly.
- If the split is ambiguous, show a shortlist and ask before launching.
- Do not invent a large batch just because parallelism is available.

## Implementation

For each sidequest, run these commands:

```bash
# 1. Create or reuse a dedicated worktree from your existing helpers
BRANCH="<name>"
[[ "$BRANCH" != */* ]] && BRANCH="feat/$BRANCH"
WT_PATH=$(zsh -ic '
branch="'"$BRANCH"'"
if wtcd "$branch" >/dev/null 2>&1; then
  pwd
else
  wtf "$branch" >/dev/null && pwd
fi
')

# 2. Create a cmux workspace in that worktree
WORKSPACE=$(cmux new-workspace --cwd "$WT_PATH" | awk '{print $2}')
cmux rename-workspace --workspace "$WORKSPACE" "<name>"

# 3. Get the terminal surface
SURFACE=$(cmux list-pane-surfaces --workspace "$WORKSPACE" | awk '{print $2}' | head -1)

# 4a. Claude Code sidequest
cmux send --workspace "$WORKSPACE" --surface "$SURFACE" "claude --permission-mode auto -n '⚔️ <name>' '<prompt>'"

# 4b. Codex sidequest
cmux send --workspace "$WORKSPACE" --surface "$SURFACE" "alias codex >/dev/null 2>&1 && codex '<prompt>' || command codex --dangerously-bypass-approvals-and-sandbox '<prompt>'"

# 5. Start it
cmux send-key --workspace "$WORKSPACE" --surface "$SURFACE" enter
```

Key details:
- Use the Claude command when the current session is Claude Code or the user explicitly wants Claude sidequests.
- Use the Codex command when the current session is Codex or the user explicitly wants Codex sidequests.
- `wtf` creates the worktree from the latest `main` and applies your normal repo setup.
- `--permission-mode auto` keeps Claude sidequests autonomous inside an isolated worktree.
- In this shell, `codex` is usually already aliased to yolo mode. Check for that alias in the target shell before appending the flag yourself, or you can accidentally pass the yolo flag twice.
- `-n '⚔️ <name>'` labels Claude sessions for `/resume`.
- Rename the cmux workspace to the sidequest name immediately so the main thread can find it later.
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
- For Claude sidequests, the parent session path (see "Parent session context" above) so it can look up more context if needed

### Fallback (no cmux)

If `CMUX_SOCKET_PATH` env var is not set, fall back to a new Terminal.app window:
```bash
osascript -e 'tell app "Terminal" to do script "cd \"$WT_PATH\" && claude --permission-mode auto -n \"⚔️ <name>\" \"<prompt>\""'
osascript -e 'tell app "Terminal" to do script "cd \"$WT_PATH\" && (alias codex >/dev/null 2>&1 && codex \"<prompt>\" || command codex --dangerously-bypass-approvals-and-sandbox \"<prompt>\")"'
```

## Main thread check-ins

After launching, confirm with a summary table of name + workspace + surface + worktree path.

Useful check-in commands:

```bash
# Read the recent terminal output without stealing focus
cmux read-screen --workspace "<name or workspace ref>" --scrollback --lines 80

# Jump into the workspace to watch it live
cmux select-workspace --workspace "<name or workspace ref>"

# Ask the sidequest for a concise checkpoint
cmux send --workspace "<name or workspace ref>" --surface "<surface ref>" "give me a 3-line status update: current finding, blocker, next step"
cmux send-key --workspace "<name or workspace ref>" --surface "<surface ref>" enter
```

Use short nudges. Do not dump a whole new task into the sidequest unless you intend to redirect it.
