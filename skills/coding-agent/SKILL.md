---
name: coding-agent
description: Run Codex CLI, Claude Code, Kimi Code CLI, OpenCode, or Pi Coding Agent via background process for programmatic control.
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "kimi", "opencode", "pi"] } },
  }
---

# Coding Agent (bash-first)

Use **bash** (with optional background mode) for all coding agent work. Simple and effective.

## ⚠️ PTY Mode Required!

Coding agents (Codex, Claude Code, Kimi, Pi) are **interactive terminal applications** that need a pseudo-terminal (PTY) to work correctly. Without PTY, you'll get broken output, missing colors, or the agent may hang.

**Always use `pty:true`** when running coding agents:

```bash
# ✅ Correct - with PTY
bash pty:true command:"codex exec 'Your prompt'"

# ❌ Wrong - no PTY, agent may break
bash command:"codex exec 'Your prompt'"
```

### Bash Tool Parameters

| Parameter    | Type    | Description                                                                 |
| ------------ | ------- | --------------------------------------------------------------------------- |
| `command`    | string  | The shell command to run                                                    |
| `pty`        | boolean | **Use for coding agents!** Allocates a pseudo-terminal for interactive CLIs |
| `workdir`    | string  | Working directory (agent sees only this folder's context)                   |
| `background` | boolean | Run in background, returns sessionId for monitoring                         |
| `timeout`    | number  | Timeout in seconds (kills process on expiry)                                |
| `elevated`   | boolean | Run on host instead of sandbox (if allowed)                                 |

### Process Tool Actions (for background sessions)

| Action      | Description                                          |
| ----------- | ---------------------------------------------------- |
| `list`      | List all running/recent sessions                     |
| `poll`      | Check if session is still running                    |
| `log`       | Get session output (with optional offset/limit)      |
| `write`     | Send raw data to stdin                               |
| `submit`    | Send data + newline (like typing and pressing Enter) |
| `send-keys` | Send key tokens or hex bytes                         |
| `paste`     | Paste text (with optional bracketed mode)            |
| `kill`      | Terminate the session                                |

---

## Quick Start: One-Shot Tasks

For quick prompts/chats:

```bash
# Kimi (no git required!)
SCRATCH=$(mktemp -d)
bash pty:true workdir:$SCRATCH command:"kimi -p 'Your prompt here'"

# Codex (needs a git repo!)
SCRATCH=$(mktemp -d) && cd $SCRATCH && git init && codex exec "Your prompt here"

# Or in a real project - with PTY!
bash pty:true workdir:~/Projects/myproject command:"kimi -p 'Add error handling to the API calls'"
bash pty:true workdir:~/Projects/myproject command:"codex exec 'Add error handling to the API calls'"
```

**Why git init?** Codex refuses to run outside a trusted git directory. Kimi doesn't need git - it works in any directory.

---

## The Pattern: workdir + background + pty

For longer tasks, use background mode with PTY:

```bash
# Kimi (no git needed!)
bash pty:true workdir:~/project background:true command:"kimi --yolo -p 'Build a snake game'"

# Codex (needs git repo!)
bash pty:true workdir:~/project background:true command:"codex exec --full-auto 'Build a snake game'"
# Returns sessionId for tracking

# Monitor progress
process action:log sessionId:XXX

# Check if done
process action:poll sessionId:XXX

# Send input (if agent asks a question)
process action:write sessionId:XXX data:"y"

# Submit with Enter (like typing "yes" and pressing Enter)
process action:submit sessionId:XXX data:"yes"

# Kill if needed
process action:kill sessionId:XXX
```

**Why workdir matters:** Agent wakes up in a focused directory, doesn't wander off reading unrelated files (like your soul.md 😅).

---

## Codex CLI

**Model:** `gpt-5.2-codex` is the default (set in ~/.codex/config.toml)

### Flags

| Flag            | Effect                                             |
| --------------- | -------------------------------------------------- |
| `exec "prompt"` | One-shot execution, exits when done                |
| `--full-auto`   | Sandboxed but auto-approves in workspace           |
| `--yolo`        | NO sandbox, NO approvals (fastest, most dangerous) |

### Building/Creating

```bash
# Quick one-shot (auto-approves) - remember PTY!
bash pty:true workdir:~/project command:"codex exec --full-auto 'Build a dark mode toggle'"

# Background for longer work
bash pty:true workdir:~/project background:true command:"codex --yolo 'Refactor the auth module'"
```

### Reviewing PRs

**⚠️ CRITICAL: Never review PRs in OpenClaw's own project folder!**
Clone to temp folder or use git worktree.

```bash
# Clone to temp for safe review
REVIEW_DIR=$(mktemp -d)
git clone https://github.com/user/repo.git $REVIEW_DIR
cd $REVIEW_DIR && gh pr checkout 130
bash pty:true workdir:$REVIEW_DIR command:"codex review --base origin/main"
# Clean up after: trash $REVIEW_DIR

# Or use git worktree (keeps main intact)
git worktree add /tmp/pr-130-review pr-130-branch
bash pty:true workdir:/tmp/pr-130-review command:"codex review --base main"
```

### Batch PR Reviews (parallel army!)

```bash
# Fetch all PR refs first
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'

# Deploy the army - one Codex per PR (all with PTY!)
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #86. git diff origin/main...origin/pr/86'"
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #87. git diff origin/main...origin/pr/87'"

# Monitor all
process action:list

# Post results to GitHub
gh pr comment <PR#> --body "<review content>"
```

---

## Claude Code

```bash
# With PTY for proper terminal output
bash pty:true workdir:~/project command:"claude 'Your task'"

# Background
bash pty:true workdir:~/project background:true command:"claude 'Your task'"
```

---

## OpenCode

```bash
bash pty:true workdir:~/project command:"opencode run 'Your task'"
```

---

## Pi Coding Agent

```bash
# Install: npm install -g @mariozechner/pi-coding-agent
bash pty:true workdir:~/project command:"pi 'Your task'"

# Non-interactive mode (PTY still recommended)
bash pty:true command:"pi -p 'Summarize src/'"

# Different provider/model
bash pty:true command:"pi --provider openai --model gpt-4o-mini -p 'Your task'"
```

**Note:** Pi now has Anthropic prompt caching enabled (PR #584, merged Jan 2026)!

---

## Kimi Code CLI

**Install:** `curl -LsSf https://code.kimi.com/install.sh | bash` or `uv tool install --python 3.13 kimi-cli`

**Git requirement:** None! Kimi works in any directory (unlike Codex which requires git).

### Flags

| Flag | Short | Effect |
| ---- | ----- | ------ |
| `--prompt` | `-p` | One-shot prompt, exits when done |
| `--command` | `-c` | Alias for `--prompt` |
| `--yolo` | `-y` | Auto-approve all operations |
| `--yes` | | Alias for `--yolo` |
| `--auto-approve` | | Alias for `--yolo` |
| `--model` | `-m` | Specify LLM model |
| `--thinking` | | Enable thinking mode (deep reasoning) |
| `--no-thinking` | | Disable thinking mode |
| `--session` | `-S` | Resume session with specified ID |
| `--continue` | `-C` | Continue previous session in current directory |
| `--max-ralph-iterations` | | Loop prompt N times until `<choice>STOP</choice>` |
| `--print` | | Non-interactive print mode (implies `--yolo`) |
| `--quiet` | | `--print --output-format text --final-message-only` |
| `--work-dir` | `-w` | Working directory (default: current) |

### Basic Usage

```bash
# Interactive mode
bash pty:true workdir:~/project command:"kimi"

# One-shot prompt (exits when done)
bash pty:true workdir:~/project command:"kimi -p 'Your task'"

# With auto-approve
bash pty:true workdir:~/project command:"kimi --yolo -p 'Your task'"

# Background session
bash pty:true workdir:~/project background:true command:"kimi -p 'Your task'"
```

### Building/Creating

```bash
# Quick one-shot with auto-approve
bash pty:true workdir:~/project command:"kimi --yolo -p 'Build a dark mode toggle'"

# Background for longer work
bash pty:true workdir:~/project background:true command:"kimi --yolo -p 'Refactor the auth module'"

# With thinking mode enabled (deep reasoning)
bash pty:true workdir:~/project command:"kimi --thinking -p 'Implement a complex algorithm'"
```

### Session Management

```bash
# Resume a previous session
bash pty:true workdir:~/project command:"kimi --continue"

# Resume specific session by ID
bash pty:true workdir:~/project command:"kimi --session abc123 -p 'Continue working on this'"

# Note: --continue and --session are mutually exclusive
```

### Loop Control (Ralph Loop)

Ralph Loop feeds the same prompt repeatedly until the agent outputs `<choice>STOP</choice>` or the iteration limit is reached. Great for iterative tasks!

```bash
# Loop up to 5 times
bash pty:true workdir:~/project command:"kimi --max-ralph-iterations 5 -p 'Refactor this code until it passes all tests'"

# Unlimited iterations (until STOP)
bash pty:true workdir:~/project command:"kimi --max-ralph-iterations -1 -p 'Keep improving this implementation'"

# Disabled (default)
bash pty:true workdir:~/project command:"kimi --max-ralph-iterations 0 -p 'One-shot task'"
```

### Thinking Mode

Thinking mode enables deep reasoning before the model responds. Requires a model that supports the `thinking` capability.

```bash
# Enable thinking mode
bash pty:true workdir:~/project command:"kimi --thinking -p 'Design a complex system architecture'"

# Explicitly disable thinking mode
bash pty:true workdir:~/project command:"kimi --no-thinking -p 'Quick question about this function'"

# Uses last session's setting if not specified
bash pty:true workdir:~/project command:"kimi -p 'Your task'"
```

### Scratch Work (No Git Required!)

Unlike Codex, Kimi doesn't require a git repository. Just use any directory:

```bash
# Create temp directory - no git needed!
SCRATCH=$(mktemp -d)
bash pty:true workdir:$SCRATCH command:"kimi --yolo -p 'Your scratch task'"
```

### Models & Providers

Kimi supports multiple providers:
- **Kimi Code** (default) - Kimi's platform with search/fetch services
- **Moonshot AI Open Platform** (China/global)
- **OpenAI** (Chat Completions API)
- **Anthropic** (Claude API)
- **Google Gemini**
- **Google Vertex AI**

Models can have capabilities like `thinking`, `always_thinking`, `image_in`, `video_in`. Configure in `~/.kimi/config.toml` or use `/login` command.

---

## Parallel Issue Fixing with git worktrees

For fixing multiple issues in parallel, use git worktrees:

```bash
# 1. Create worktrees for each issue
git worktree add -b fix/issue-78 /tmp/issue-78 main
git worktree add -b fix/issue-99 /tmp/issue-99 main

# 2. Launch agents in each (background + PTY!)
# Kimi (no git repo needed in worktree)
bash pty:true workdir:/tmp/issue-78 background:true command:"pnpm install && kimi --yolo -p 'Fix issue #78: <description>. Commit and push.'"
bash pty:true workdir:/tmp/issue-99 background:true command:"pnpm install && kimi --yolo -p 'Fix issue #99: <description>. Commit and push.'"

# Or use Codex (requires git repo)
bash pty:true workdir:/tmp/issue-78 background:true command:"pnpm install && codex --yolo 'Fix issue #78: <description>. Commit and push.'"
bash pty:true workdir:/tmp/issue-99 background:true command:"pnpm install && codex --yolo 'Fix issue #99: <description>. Commit and push.'"

# 3. Monitor progress
process action:list
process action:log sessionId:XXX

# 4. Create PRs after fixes
cd /tmp/issue-78 && git push -u origin fix/issue-78
gh pr create --repo user/repo --head fix/issue-78 --title "fix: ..." --body "..."

# 5. Cleanup
git worktree remove /tmp/issue-78
git worktree remove /tmp/issue-99
```

---

## ⚠️ Rules

1. **Always use pty:true** - coding agents need a terminal!
2. **Respect tool choice** - if user asks for Codex, use Codex.
   - Orchestrator mode: do NOT hand-code patches yourself.
   - If an agent fails/hangs, respawn it or ask the user for direction, but don't silently take over.
3. **Be patient** - don't kill sessions because they're "slow"
4. **Monitor with process:log** - check progress without interfering
5. **--full-auto for building** - auto-approves changes
6. **vanilla for reviewing** - no special flags needed
7. **Parallel is OK** - run many Codex processes at once for batch work
8. **NEVER start Codex in ~/clawd/** - it'll read your soul docs and get weird ideas about the org chart!
9. **NEVER checkout branches in ~/Projects/openclaw/** - that's the LIVE OpenClaw instance!

---

## Progress Updates (Critical)

When you spawn coding agents in the background, keep the user in the loop.

- Send 1 short message when you start (what's running + where).
- Then only update again when something changes:
  - a milestone completes (build finished, tests passed)
  - the agent asks a question / needs input
  - you hit an error or need user action
  - the agent finishes (include what changed + where)
- If you kill a session, immediately say you killed it and why.

This prevents the user from seeing only "Agent failed before reply" and having no idea what happened.

---

## Auto-Notify on Completion

For long-running background tasks, append a wake trigger to your prompt so OpenClaw gets notified immediately when the agent finishes (instead of waiting for the next heartbeat):

```
... your task here.

When completely finished, run this command to notify me:
openclaw system event --text "Done: [brief summary of what was built]" --mode now
```

**Example:**

```bash
# Kimi
bash pty:true workdir:~/project background:true command:"kimi --yolo -p 'Build a REST API for todos.

When completely finished, run: openclaw gateway wake --text \"Done: Built todos REST API with CRUD endpoints\" --mode now'"

# Codex
bash pty:true workdir:~/project background:true command:"codex --yolo exec 'Build a REST API for todos.

When completely finished, run: openclaw system event --text \"Done: Built todos REST API with CRUD endpoints\" --mode now'"
```

This triggers an immediate wake event — Skippy gets pinged in seconds, not 10 minutes.

---

## Learnings (Jan 2026)

- **PTY is essential:** Coding agents are interactive terminal apps. Without `pty:true`, output breaks or agent hangs.
- **Git repo required:** Codex won't run outside a git directory. Use `mktemp -d && git init` for scratch work. **Kimi doesn't need git** - works in any directory.
- **exec is your friend:** `codex exec "prompt"` runs and exits cleanly - perfect for one-shots. Kimi uses `-p` or `--prompt` instead.
- **submit vs write:** Use `submit` to send input + Enter, `write` for raw data without newline.
- **Sass works:** Codex responds well to playful prompts. Asked it to write a haiku about being second fiddle to a space lobster, got: _"Second chair, I code / Space lobster sets the tempo / Keys glow, I follow"_ 🦞
- **Kimi's Ralph Loop:** `--max-ralph-iterations N` is great for iterative tasks - feeds the same prompt repeatedly until `<choice>STOP</choice>`.
- **Kimi's Thinking Mode:** `--thinking` enables deep reasoning (like Claude's thinking models). Requires a model that supports the `thinking` capability.
