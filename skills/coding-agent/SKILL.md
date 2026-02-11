---
name: coding-agent
description: Run Codex CLI, Claude Code, Kimi Code CLI, OpenCode, or Pi Coding Agent via background process for programmatic control.
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "kimi", "opencode", "pi"] } },
  }
---

# Coding Agent

Use bash to run coding agents (Codex, Claude Code, Kimi, Pi, OpenCode) for automated coding tasks.

## ⚠️ CRITICAL: PTY Mode by Agent

> 🚨 **Kimi: NEVER use PTY for file operations** — causes indefinite hangs on writes
> 
> 🟢 **Kimi: Always use NON-PTY** — `bash workdir:~/project command:"kimi ..."`
>
> 🟢 **Codex/Claude: Always use PTY** — `bash pty:true workdir:~/project command:"codex ..."`

---

## Quick Start

```bash
# Kimi - create files (NON-PTY mode)
bash workdir:~/project timeout:300 command:"kimi --yolo -p 'Create React app with src/, package.json'"

# Kimi - edit files (StrReplaceFile)
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Add dark mode to App.css. Use StrReplaceFile.'"

# Codex - requires PTY and git repo
bash pty:true workdir:~/project command:"codex exec --full-auto 'Add error handling'"

# Claude Code - interactive
bash pty:true workdir:~/project command:"claude 'Refactor auth module'"
```

---

## Agent Selection Guide

| Agent | Best For | Git Required | PTY Mode | See Details |
|-------|----------|--------------|----------|-------------|
| **Kimi** | File creation, editing, fast iteration | ❌ No | ❌ Non-PTY | [references/kimi.md](references/kimi.md) |
| **Codex** | Sandboxed execution, PR reviews | ✅ Yes | ✅ Required | [references/codex.md](references/codex.md) |
| **Claude** | Interactive refactoring, exploration | ✅ Yes | ✅ Required | [references/claude.md](references/claude.md) |
| **Pi** | Quick tasks, alternative provider | ❌ No | ✅ Recommended | Below |
| **OpenCode** | Open-source alternative | ❌ No | ⚠️ Varies | Below |

---

## The Golden Patterns

### Pattern 1: File Creation with Kimi

```bash
# Create files
bash workdir:~/project timeout:300 command:"kimi --yolo -p 'Write all files directly to disk:
- index.html
- css/variables.css, css/components.css
- js/main.js'"

# Verify creation
bash workdir:~/project command:"find . -type f | wc -l && tree -L 2"
```

**Why:** Kimi writes files directly and reliably in parallel. Explicit "write to disk" wording works best. Always verify after creation.

---

### Pattern 2: File Editing with StrReplaceFile

```bash
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update css/variables.css:
- Change primary color to blue
- Add border-radius variable
Use StrReplaceFile.'"
```

**Why:** StrReplaceFile is the most reliable editing method (~3 min vs 10+ min).

---

### Pattern 3: Chunk Large Changes

**Don't:** Ask for multiple large file returns (hangs)
```bash
# AVOID - hangs indefinitely
bash workdir:~/project command:"kimi -p 'Return all 8 redesigned CSS files...'"
```

**Do:** Break into sequential edits
```bash
# Step 1
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update variables.css. Use StrReplaceFile.'"
# Step 2
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update components.css. Use StrReplaceFile.'"
```

---

## Critical Rules

1. **Kimi: Non-PTY for file operations** - PTY causes indefinite hangs on writes
2. **Codex/Claude: PTY required** - They need PTY for proper operation
3. **Use explicit timeouts** - 300s+ for file creation, 120s+ for edits
4. **StrReplaceFile for edits** - More reliable than returning content
5. **Wording matters for Kimi** - Use "write to disk" language for reliability
6. **Kill if hung** - If "Composing..." >5 min with no files, kill and retry
7. **No HTTP servers in sandbox** - Testing phases with `http-server` get SIGKILL
8. **Codex needs git repo** - Won't run outside trusted git directory
9. **Kimi doesn't need git** - Works in any directory

---

## Fallback Pattern (When Direct Writes Fail)

### Experimental Fallback First 🧪

Try creating an empty file, then using StrReplaceFile:

```bash
# Step 1: Create empty file
bash workdir:~/project command:"kimi --yolo -p 'Create empty file PRD.md'"

# Step 2: Populate with StrReplaceFile
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Write PRD content to PRD.md. Use StrReplaceFile.'"
```

**Status:** Experimental - keeps Kimi in "file operation mode." Test if direct writes hang.

### Return-Content Fallback (Last Resort)

If StrReplaceFile fallback also fails:

```bash
# 1. Get content instead of file write
bash workdir:~/project command:"kimi --yolo -p 'Create the file content.
Return in a markdown code block. Do not write files.'"

# 2. Use OpenClaw WriteFile tool with the returned content
```

**Note:** This is a last-resort fallback. Direct writes usually work.

---

## PTY Mode Reference

### Non-PTY (Kimi Default)

```bash
bash workdir:~/project command:"kimi --yolo -p 'Task'"
```

- ✅ File writes work
- ⚠️ No streaming ("Composing..." until done)
- ✅ Can background

### PTY Mode (Codex/Claude Required)

```bash
bash pty:true workdir:~/project command:"codex exec --full-auto 'Task'"
```

- ✅ Streaming output
- ❌ Kimi hangs on file writes
- ⚠️ Blocks until done

---

## Timeouts

| Task Type | Timeout | Example | Expected Duration |
|-----------|---------|---------|-------------------|
| Quick read | 60s | "What does this function do?" | 10-30s |
| Single edit | 120s | StrReplaceFile one file | 2-3 min |
| Multi-file creation | 300s | Create 5-10 files | 5-8 min |
| Large file creation | 600s | Create 15-20 files | 10-15 min |
| Complex refactor | 300-600s | Redesign with multiple changes | 8-15 min |
| Full build | 600s+ | Complete application | 15+ min |

**Rule of thumb:** ~30-45 seconds per file for multi-file creation.

---

## Background Mode

For long tasks, use background mode:

```bash
# Start
bash workdir:~/project background:true timeout:300 command:"kimi --yolo -p 'Build app'"
# Returns: sessionId

# Check status
process action:poll sessionId:XXX

# View output
process action:log sessionId:XXX

# Kill if stuck
process action:kill sessionId:XXX
```

---

## Tool-Specific Details

### Kimi Code CLI

**Install:** `curl -LsSf https://code.kimi.com/install.sh | bash`

**Key flags:**
- `--yolo` / `-y` - Auto-approve
- `-p 'prompt'` - One-shot mode
- `--thinking` - Deep reasoning

**See full guide:** [references/kimi.md](references/kimi.md)

---

### Codex CLI

**Install:** `npm install -g @openai/codex`

**Requirements:** Git repository, PTY mode

**Key flags:**
- `exec 'prompt'` - One-shot
- `--full-auto` - Auto-approve in workspace
- `--yolo` - No sandbox (dangerous)

**See full guide:** [references/codex.md](references/codex.md)

---

### Claude Code

**Install:** `npm install -g @anthropic-ai/claude-code`

**Requirements:** Git repository, PTY mode

**Best for:** Interactive refactoring, exploration

**See full guide:** [references/claude.md](references/claude.md)

---

### Pi Coding Agent

**Install:** `npm install -g @mariozechner/pi-coding-agent`

```bash
bash pty:true workdir:~/project command:"pi 'Your task'"
bash pty:true command:"pi -p 'Quick summary'"
```

---

### OpenCode

```bash
bash pty:true workdir:~/project command:"opencode run 'Your task'"
```

**Note:** Test both PTY and non-PTY modes.

---

## Parallel Execution

Run multiple agents simultaneously:

```bash
# Kimi - multiple features in parallel
bash workdir:/tmp/feature-a background:true timeout:300 command:"kimi --yolo -p 'Build auth'"
bash workdir:/tmp/feature-b background:true timeout:300 command:"kimi --yolo -p 'Build dashboard'"

# Codex - PR review army
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #86'"
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #87'"

# Monitor all
process action:list
```

---

## Progress Updates

When spawning agents in background:

1. **Start:** Send 1 short message (what's running + where)
2. **Changes:** Update when:
   - Milestone completes
   - Agent needs input
   - Error occurs
   - Agent finishes
3. **Kill:** Immediately say you killed it and why

---

## Auto-Notify on Completion

Add wake trigger to long tasks:

```bash
bash workdir:~/project background:true timeout:300 command:"kimi --yolo -p 'Build API.
When done run: openclaw gateway wake --text \"API ready\" --mode now'"
```

---

## Troubleshooting

### File writes hang

**Kimi:** Ensure non-PTY mode. If "Composing..." >5 min with no file output:

1. **Kill the session:** `process action:kill sessionId:XXX`
2. **Retry with different wording** - Use "write to disk" phrasing
3. **Try experimental fallback** - Create empty file + StrReplaceFile
4. **Use return-content fallback** - Last resort (see Fallback Pattern section)

### "Not a git repository"

**Codex/Claude:** Initialize git or cd to a git repo.

### Session killed (SIGKILL)

Don't start HTTP servers in prompts. Resource limits kill them.

### "Composing..." forever (Kimi)

- Under 15 min: Normal for complex tasks
- Over 15 min: Kill and retry with chunked approach

---

## See Also

| File | Contents |
|------|----------|
| [references/kimi.md](references/kimi.md) | Complete Kimi CLI guide, patterns, troubleshooting |
| [references/codex.md](references/codex.md) | Codex CLI guide, PR reviews, flags |
| [references/claude.md](references/claude.md) | Claude Code guide, interactive workflows |
