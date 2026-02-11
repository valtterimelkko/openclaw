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

## ⚠️ PTY Mode

Coding agents are **interactive terminal applications** that typically need a pseudo-terminal (PTY) to work correctly. However, **PTY behavior varies significantly by agent**:

### PTY Recommendations by Agent

| Agent | Recommended Mode | Notes |
|-------|------------------|-------|
| **Kimi** | **Non-PTY (default)** | PTY causes indefinite hangs on file writes. Use PTY only for interactive chat sessions. |
| **Codex** | **PTY recommended** | Requires git repo. PTY works reliably. |
| **Claude** | **PTY recommended** | PTY works well for interactive mode. |
| **OpenCode** | Test both | Behavior varies by platform. |
| **Pi** | **PTY recommended** | PTY works well. |

```bash
# ✅ Kimi - Non-PTY for file operations (default recommendation)
bash workdir:~/project command:"kimi --yolo -p 'Build a todo app'"

# ✅ Codex/Claude - PTY recommended
bash pty:true workdir:~/project command:"codex exec --full-auto 'Build a todo app'"

# ⚠️ Kimi with PTY - only for interactive chat (no file writes)
bash pty:true workdir:~/project command:"kimi"
```

### Non-PTY Mode: What to Expect

When using non-PTY mode (recommended for Kimi):
- **No streaming output** — You'll see "Composing..." until completion
- **Use longer timeouts** — 180-300s for file generation tasks
- **Output appears all at once** — When the agent finishes
- **More reliable file writes** — The tradeoff for no streaming

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

## Agent-Specific Quick Reference

### Recommended Flags by Task Type

| Task | Kimi | Codex | Claude | Pi |
|------|------|-------|--------|-----|
| **Create/Build** | `--yolo -p` | `exec --full-auto` | (interactive) | `-p` |
| **Edit files** | `--yolo -p` | `exec --full-auto` | (interactive) | `-p` |
| **Review code** | `-p` | `review` | (interactive) | `-p` |
| **Quick question** | `-p` | `exec` | (interactive) | `-p` |

### Key Flags Explained

| Agent | Flag | What It Does |
|-------|------|--------------|
| **Kimi** | `--yolo` / `-y` | Auto-approve all operations |
| **Kimi** | `-p 'prompt'` | One-shot mode (exits when done) |
| **Codex** | `exec 'prompt'` | One-shot execution, exits when done |
| **Codex** | `--full-auto` | Auto-approves changes in workspace |
| **Codex** | `--yolo` | No sandbox, no approvals (dangerous) |
| **All** | `--thinking` | Enable deep reasoning mode |

---

## Quick Start: One-Shot Tasks

For quick prompts/chats:

```bash
# Kimi (no git required!) - Non-PTY recommended
SCRATCH=$(mktemp -d)
bash workdir:$SCRATCH command:"kimi --yolo -p 'Your prompt here'"

# Codex (needs a git repo!) - PTY recommended
SCRATCH=$(mktemp -d) && cd $SCRATCH && git init && codex exec "Your prompt here"

# Or in a real project
bash workdir:~/Projects/myproject command:"kimi --yolo -p 'Add error handling to the API calls'"
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

## Foreground vs Background: When to Use Each

| Use Case | Mode | Timeout | Example |
|----------|------|---------|---------|
| Quick questions (<30s) | Foreground | 60s | "What does this function do?" |
| Single file edits | Foreground | 120s | "Add error handling to auth.js" |
| Multi-file changes | Background | 300s+ | "Refactor the auth module" |
| Long builds/tests | Background | 600s+ | "Build and test the full app" |
| Parallel batch work | Background | 300s+ | "Review 5 PRs simultaneously" |

**Guidelines:**
- **Foreground**: You wait for the result; good for quick tasks where you need the output immediately
- **Background**: Returns a sessionId; use for long tasks or when you want to run multiple agents in parallel
- **Always use explicit timeouts** — Don't rely on defaults for long tasks

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
# Interactive mode (PTY needed for TUI)
bash pty:true workdir:~/project command:"kimi"

# One-shot prompt - Non-PTY recommended for file operations
bash workdir:~/project command:"kimi --yolo -p 'Your task'"

# With explicit timeout for generation tasks
bash timeout:180 workdir:~/project command:"kimi --yolo -p 'Build a dashboard'"

# Background session
bash workdir:~/project background:true timeout:300 command:"kimi --yolo -p 'Your task'"
```

### Building/Creating

```bash
# Quick one-shot with auto-approve - Non-PTY
bash workdir:~/project command:"kimi --yolo -p 'Build a dark mode toggle'"

# Background for longer work with timeout
bash workdir:~/project background:true timeout:300 command:"kimi --yolo -p 'Refactor the auth module'"

# With thinking mode enabled (deep reasoning)
bash workdir:~/project command:"kimi --thinking --yolo -p 'Design a complex system'"
```

### Session Management

```bash
# Resume a previous session (PTY for interactive TUI)
bash pty:true workdir:~/project command:"kimi --continue"

# Resume specific session by ID
bash workdir:~/project command:"kimi --session abc123 -p 'Continue working on this'"

# Note: --continue and --session are mutually exclusive
```

### Loop Control (Ralph Loop)

Ralph Loop feeds the same prompt repeatedly until the agent outputs `<choice>STOP</choice>` or the iteration limit is reached. Great for iterative tasks!

```bash
# Loop up to 5 times
bash workdir:~/project command:"kimi --max-ralph-iterations 5 -p 'Refactor this code until it passes all tests'"

# Unlimited iterations (until STOP)
bash workdir:~/project command:"kimi --max-ralph-iterations -1 -p 'Keep improving this implementation'"

# Disabled (default)
bash workdir:~/project command:"kimi --max-ralph-iterations 0 -p 'One-shot task'"
```

### Thinking Mode

Thinking mode enables deep reasoning before the model responds. Requires a model that supports the `thinking` capability.

```bash
# Enable thinking mode
bash workdir:~/project command:"kimi --thinking --yolo -p 'Design a complex system architecture'"

# Explicitly disable thinking mode
bash workdir:~/project command:"kimi --no-thinking --yolo -p 'Quick question about this function'"

# Uses last session's setting if not specified
bash workdir:~/project command:"kimi --yolo -p 'Your task'"
```

### Scratch Work (No Git Required!)

Unlike Codex, Kimi doesn't require a git repository. Just use any directory:

```bash
# Create temp directory - no git needed!
SCRATCH=$(mktemp -d)
bash workdir:$SCRATCH command:"kimi --yolo -p 'Your scratch task'"
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

# 2. Launch agents in each (background mode)
# Kimi (no git repo needed in worktree, non-PTY recommended)
bash workdir:/tmp/issue-78 background:true timeout:300 command:"pnpm install && kimi --yolo -p 'Fix issue #78: <description>. Commit and push.'"
bash workdir:/tmp/issue-99 background:true timeout:300 command:"pnpm install && kimi --yolo -p 'Fix issue #99: <description>. Commit and push.'"

# Or use Codex (requires git repo, PTY recommended)
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

1. **PTY depends on the agent** - Use non-PTY for Kimi (file writes hang with PTY). Use PTY for Codex and Claude.
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
# Kimi (non-PTY recommended)
bash workdir:~/project background:true timeout:300 command:"kimi --yolo -p 'Build a REST API for todos.

When completely finished, run: openclaw gateway wake --text \"Done: Built todos REST API with CRUD endpoints\" --mode now'"

# Codex (PTY recommended)
bash pty:true workdir:~/project background:true command:"codex --yolo exec 'Build a REST API for todos.

When completely finished, run: openclaw system event --text \"Done: Built todos REST API with CRUD endpoints\" --mode now'"
```

This triggers an immediate wake event — Skippy gets pinged in seconds, not 10 minutes.

---

## Troubleshooting

### File Write Operations Hang

**Symptoms:** Agent starts, reads files fine, but gets stuck at "Now I'll create..." and never completes file writes.

**Solutions:**

1. **Use non-PTY mode for Kimi** - PTY causes indefinite hangs on file writes:
   ```bash
   # Non-PTY for Kimi (default recommendation)
   bash workdir:~/project command:"kimi --yolo -p 'Your task'"
   ```

2. **Use longer timeouts** - Default 60s is too short for generation tasks:
   ```bash
   # Longer timeout for file generation
   bash timeout:180 workdir:~/project command:"kimi --yolo -p 'Your task'"
   ```

3. **Fallback to content return** - If agent can't write files, have it return the content so OpenClaw can write directly:
   ```
   "If file write operations don't work, output the complete file content 
   in a code block and I'll handle writing it."
   ```

### Session Stalls Without Output

**Symptoms:** Background session starts but produces no output.

**Check:**
- Is the working directory valid?
- Does Codex have a git repo? (Kimi doesn't need one)
- Try polling the session: `process action:poll sessionId:XXX`
- Check logs: `process action:log sessionId:XXX`

### Platform-Specific Notes

| Platform | Notes |
|----------|-------|
| Linux/Docker | Kimi PTY mode may hang on file writes — use non-PTY |
| macOS | PTY mode usually works for all agents |
| WSL | Test both modes; may vary by WSL version |

---

## Timeout Recommendations

| Task Type | Timeout | Notes |
|-----------|---------|-------|
| Quick read/analysis | 60s | File reading, simple questions |
| Small file edits | 120s | Single file changes |
| Feature implementation | 300s+ | Multi-file changes, builds |
| Large refactoring | 600s+ | Complex rewrites, testing |

**Always use explicit timeouts** - Don't rely on defaults for long tasks:
```bash
# Good - explicit timeout for Kimi (non-PTY)
bash timeout:300 workdir:~/project command:"kimi --yolo -p 'Build a dashboard'"

# Good - explicit timeout for Codex (PTY)
bash pty:true timeout:300 workdir:~/project command:"codex exec --full-auto 'Build a dashboard'"
```

---

## Fallback Pattern: When Agents Can't Write Files

**Use this pattern when:**
- Agent hangs during file write operations (common with Kimi + PTY)
- You want more control over file creation
- Agent file operations fail silently

### Step-by-Step Fallback

**Step 1: Ask agent to return content instead of writing**

```bash
bash workdir:~/project timeout:180 command:"kimi --yolo -p 'Create a todo app with HTML/CSS/JS.
IMPORTANT: Do NOT write files directly. 
Instead, return the complete code in markdown code blocks.
Label each block with the filename like: ```html filename=index.html```
'"
```

**Step 2: Parse the agent output and write files with OpenClaw tools**

The agent output will look like:
```
Here's the todo app:

```html filename=index.html
<!DOCTYPE html>
<html>...</html>
```

```css filename=styles.css
body { ... }
```
```

Extract each code block and use `WriteFile` to create the files.

### Why This Works

- Bypasses agent file operation issues entirely
- You have full control over file naming and location
- No hanging on WriteFile operations
- Works reliably with Kimi in non-PTY mode

---

## Learnings (Jan 2026)

- **PTY varies by agent:** Codex and Claude usually need PTY. Kimi requires non-PTY for file operations — PTY causes indefinite hangs.
- **Git repo required:** Codex won't run outside a git directory. Use `mktemp -d && git init` for scratch work. **Kimi doesn't need git** - works in any directory.
- **exec is your friend:** `codex exec "prompt"` runs and exits cleanly - perfect for one-shots. Kimi uses `-p` or `--prompt` instead.
- **submit vs write:** Use `submit` to send input + Enter, `write` for raw data without newline.
- **Sass works:** Codex responds well to playful prompts. Asked it to write a haiku about being second fiddle to a space lobster, got: _"Second chair, I code / Space lobster sets the tempo / Keys glow, I follow"_ 🦞
- **Kimi's Ralph Loop:** `--max-ralph-iterations N` is great for iterative tasks - feeds the same prompt repeatedly until `<choice>STOP</choice>`.
- **Kimi's Thinking Mode:** `--thinking` enables deep reasoning (like Claude's thinking models). Requires a model that supports the `thinking` capability.
- **File write hangs:** Kimi in PTY mode may hang indefinitely during WriteFile operations. Use non-PTY mode or the fallback pattern (return content instead of writing).
