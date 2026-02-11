# Kimi Code CLI - Complete Guide

Best practices for using Kimi CLI via OpenClaw, based on extensive testing.

## The Golden Pattern

```
Phase 1: Creation → Direct file writes (not "return content")
Phase 2: Editing  → StrReplaceFile for targeted changes
```

---

## Recommended Patterns (Do This)

### Pattern 1: Direct File Creation ✅

**Best for:** Initial scaffolding, creating multiple new files

```bash
bash workdir:~/project timeout:300 command:"kimi --yolo -p 'Build a portfolio with:
- index.html with semantic sections
- css/variables.css, css/components.css
- js/main.js for interactivity
- README.md with setup instructions'"
```

**Why this works:**
- Kimi handles 10+ file writes in parallel reliably
- Uses internal SetTodoList for progress tracking
- No streaming output issues
- Fastest approach for new projects

---

### Pattern 2: StrReplaceFile for Editing ✅

**Best for:** Modifying existing files, targeted changes

```bash
# Single file edit
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update css/variables.css:
- Change primary color to #3b82f6
- Add --border-radius-sm: 4px
Use StrReplaceFile.'"

# Multi-file coordinated changes
bash workdir:~/project timeout:180 command:"kimi --yolo -p 'Add dark mode:
1. Update css/variables.css with dark theme
2. Add toggle to index.html nav
3. Create js/theme.js for logic
Use StrReplaceFile for all changes.'"
```

**Why this works:**
- StrReplaceFile is Kimi's native editing tool
- Completes in ~3 minutes vs 10+ for alternatives
- Most reliable for modifications

---

### Pattern 3: Chunk Large Refactors ✅

**Best for:** Redesigns, large multi-file changes

**Don't do this:**
```bash
# AVOID - hangs on multi-file returns
bash workdir:~/project command:"kimi -p 'Redesign and return all 8 CSS files...'"
```

**Do this instead:**
```bash
# Step 1: Update colors (2 min)
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update variables.css color palette. Use StrReplaceFile.'"

# Step 2: Add effects (2 min)  
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Add glassmorphism to components.css. Use StrReplaceFile.'"

# Step 3: Redesign sections (2 min)
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Redesign hero in index.html. Use StrReplaceFile.'"
```

---

## Anti-Patterns (Don't Do This)

### ❌ Return Content for Multiple Files

```bash
# HANGS - Never ask for multiple large file returns
bash workdir:~/project command:"kimi --yolo -p 'Redesign and return complete 
content for 5 CSS files in markdown code blocks'"
```

**What happens:** Kimi plans successfully, then hangs outputting large code blocks.

**Use instead:** Pattern 2 (StrReplaceFile) or Pattern 3 (chunked edits)

---

### ❌ Complex Everything-at-Once Prompts

```bash
# Too large - may timeout or hang
bash workdir:~/project timeout:300 command:"kimi --yolo -p 'Create full-stack app 
with React, Node, MongoDB, auth, tests, CI/CD, Docker...'"
```

**Use instead:** Break into phases:
1. Frontend structure
2. Backend API
3. Database layer
4. Auth integration
5. Deployment config

---

### ❌ HTTP Servers in Sandbox

```bash
# Gets SIGKILL - resource limits
bash workdir:~/project command:"kimi --yolo -p '...then start http-server for testing'"
```

**Use instead:** Skip server startup. Files are correct without runtime testing.

---

## PTY vs Non-PTY

### Non-PTY (Default for File Operations)

```bash
bash workdir:~/project command:"kimi --yolo -p 'Create files...'"
```

| Pros | Cons |
|------|------|
| File writes work reliably | No streaming output |
| Can run in background | "Composing..." until done |
| No indefinite hangs (rare exceptions) | Less feedback during execution |

**Best for:** All file creation and editing tasks

---

### PTY Mode (Interactive Only)

```bash
bash pty:true workdir:~/project command:"kimi"
```

| Pros | Cons |
|------|------|
| Full TUI with streaming | **Hangs on file writes** (Linux/Docker) |
| Real-time progress | Cannot background effectively |
| Good for debugging | Blocks until completion |

**Best for:** Interactive chat sessions only. Never for file operations.

---

## Timeouts by Task Type

| Task | Timeout | Typical Duration |
|------|---------|------------------|
| Quick read/analysis | 60s | 10-30s |
| Single file edit | 120s | 2-3 min |
| Multi-file creation (5-10 files) | 300s | 5-8 min |
| Complex refactor | 300s | 8-10 min |
| Full project build | 600s | 10-15 min |

**Always use explicit timeouts.** Default 60s is too short for most tasks.

---

## Understanding Kimi's Internal Behavior

### SetTodoList Progress Tracking

Kimi tracks progress internally. You may see:

```
✓ Read existing files
⏳ Creating css/variables.css
○ Building index.html
```

This is normal and indicates healthy execution.

### Execution Flow

1. **Analyze** - Reads requirements/existing files
2. **Plan** - Creates internal todo list
3. **Scaffold** - Creates directory structure
4. **Create/Edit** - Writes files (parallel for multiple)
5. **Verify** - Confirms files written correctly

---

## Troubleshooting

### "Composing..." for 10+ Minutes

**Likely causes:**
- Genuinely complex task (wait longer)
- Attempting multi-file content return (kill and retry with StrReplaceFile)

**Solution:**
```bash
# Kill stuck session
process action:kill sessionId:XXX

# Retry with StrReplaceFile approach
bash workdir:~/project command:"kimi --yolo -p 'Make the change with StrReplaceFile'"
```

---

### File Write Failed

**First:** Check you're using non-PTY mode (PTY hangs on writes)

**Second:** Retry - transient issues happen

**Third:** Use fallback pattern:
```bash
bash workdir:~/project command:"kimi --yolo -p 'Create the content and return 
in a code block. Do not write files.'"
# Then use OpenClaw WriteFile tool
```

---

### Session Killed (SIGKILL)

**Likely causes:**
- Starting HTTP servers (resource limits)
- Memory/CPU limits exceeded
- Timeout exceeded

**Solution:** Remove server startup from prompts. Test files are valid without serving.

---

## Summary Table

| Approach | Use Case | Result | Time |
|----------|----------|--------|------|
| Direct file write | Creation | ✅ Reliable | 5-10 min |
| StrReplaceFile | Editing | ✅ Best | ~3 min |
| Return content (fallback) | When writes fail | ✅ Works | ~8 min |
| Return multi-file content | Complex edits | ❌ Hangs | — |
| Chunked StrReplaceFile | Large refactors | ✅ Best | 2-3 min each |

---

## Quick Reference

```bash
# Create project
bash workdir:~/project timeout:300 command:"kimi --yolo -p 'Create React app with src/, public/, package.json'"

# Edit file
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Update src/App.css with dark mode. Use StrReplaceFile.'"

# Interactive session (no file ops)
bash pty:true workdir:~/project command:"kimi"

# Continue session
bash workdir:~/project command:"kimi --continue"

# With thinking mode
bash workdir:~/project command:"kimi --thinking --yolo -p 'Design architecture'"
```

---

## Key Takeaways

1. **Primary pattern:** Direct file creation + StrReplaceFile for edits
2. **Fallback pattern:** Return content (only when direct writes fail)
3. **Always non-PTY** for file operations
4. **Always explicit timeouts** (300s+ for creation)
5. **Chunk large tasks** - don't ask for multi-file returns
6. **No HTTP servers** in sandboxed environments
