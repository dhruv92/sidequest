---
name: sidequest
version: 3.0.0
description: |
  Spawn new Claude or Codex sessions in separate worktrees and cmux workspaces.
  Single: `/sidequest <name> [prompt]`. Selective batch: `/sidequest pick [n]`
  to shortlist a few good sidequests and launch only the chosen subset.
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

### Selective Batch: `/sidequest pick [n]`

Review the current conversation and identify the best distinct solutions, approaches, or investigative threads to run in parallel. For each candidate, compose:
- A short descriptive kebab-case name (e.g. `redis-cache`, `lazy-load`, `denormalize`)
- A detailed self-contained prompt with enough context that a fresh Claude session can work independently

Behavior:
- Show a shortlist first. Do not auto-launch everything.
- Ask the user which subset to launch by name.
- If `n` is provided, shortlist at most that many candidates.
- If the user already named the sidequests they want, launch only those and skip the shortlist step.

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

# 3. Get the terminal surface
SURFACE=$(cmux list-pane-surfaces --workspace $WORKSPACE | awk '{print $2}' | head -1)

# 4a. Claude Code sidequest
cmux send --workspace $WORKSPACE --surface $SURFACE "claude --permission-mode auto -n '⚔️ <name>' '<prompt>'"

# 4b. Codex sidequest
cmux send --workspace $WORKSPACE --surface $SURFACE "codex --dangerously-bypass-approvals-and-sandbox '<prompt>'"

# 5. Start it
cmux send-key --workspace $WORKSPACE --surface $SURFACE enter
```

Key details:
- Use the Claude command when the current session is Claude Code or the user explicitly wants Claude sidequests.
- Use the Codex command when the current session is Codex or the user explicitly wants Codex sidequests.
- `wtf` creates the worktree from the latest `main` and applies your normal repo setup.
- `--permission-mode auto` keeps Claude sidequests autonomous inside an isolated worktree.
- `--dangerously-bypass-approvals-and-sandbox` is the Codex yolo mode.
- `-n '⚔️ <name>'` labels Claude sessions for `/resume`.
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
osascript -e 'tell app "Terminal" to do script "cd \"$WT_PATH\" && codex --dangerously-bypass-approvals-and-sandbox \"<prompt>\""'
```

After launching, confirm with a summary table of name + prompt.
