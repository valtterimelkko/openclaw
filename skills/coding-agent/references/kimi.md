# Kimi Code CLI - Complete Guide

Best practices for using Kimi CLI via OpenClaw, based on extensive testing.

## The Golden Pattern

```
Phase 1: Creation → Direct file writes (not "return content")
Phase 2: Editing  → StrReplaceFile for targeted changes
Phase 3: Fallback → StrReplaceFile (experimental) or return content
```

---

## Wording Matters

**The phrasing of your prompt significantly affects reliability.**

### What Worked in Testing

| Phrasing | Result |
|----------|--------|
| "Write all files directly to disk" | ✅ 17 files created in ~10 min |
| "Create PRD.md" (specific file) | ❌ Hung indefinitely |
| "Build a portfolio with index.html, css/, js/..." | ✅ Works reliably |

**Theory:** Explicit "write to disk" language activates Kimi's file operation mode directly, while ambiguous phrasing may trigger internal deliberation loops.

### Recommended Wording

**✅ Good - Explicit "write" language:**
```bash
bash workdir:~/project command:"kimi --yolo -p 'Write all files directly to disk:
- index.html
- css/styles.css
- js/app.js'"
```

**✅ Good - Action-oriented:**
```bash
bash workdir:~/project command:"kimi --yolo -p 'Build a React app. Create the following files...'"
```

**⚠️ Avoid - Passive language:**
```bash
# Less reliable - may hang
bash workdir:~/project command:"kimi --yolo -p 'PRD.md should contain...'"
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

**After creation, always verify:**
```bash
bash workdir:~/project command:"find . -type f | wc -l && tree -L 2"
```

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

### "Composing..." for Too Long

**If "Composing..." shows for >2 minutes without file writes, take action.**

| Duration | Action |
|----------|--------|
| 0-2 min | Normal for task startup |
| 2-5 min | Wait - may be planning complex changes |
| 5-10 min | Check SetTodoList progress; if no files created, consider killing |
| 10+ min | **Kill and retry** - likely hung |

**Kill stuck session:**
```bash
process action:kill sessionId:XXX
```

**Retry strategies:**
1. **Retry with different wording** (see "Wording Matters" above)
2. **Use StrReplaceFile** for the specific file
3. **Use return-content fallback** (last resort)

---

### Experimental: StrReplaceFile Fallback

> 🧪 **EXPERIMENTAL** - Try this before return-content fallback

If direct file writes hang, try creating empty files first, then using StrReplaceFile:

```bash
# Step 1: Create empty file (may work when "create with content" hangs)
bash workdir:~/project command:"kimi --yolo -p 'Create empty file PRD.md'"

# Step 2: Populate with StrReplaceFile
bash workdir:~/project timeout:120 command:"kimi --yolo -p 'Write the PRD content to PRD.md. Use StrReplaceFile.'"
```

**Why this might work:** Keeps Kimi in "file operation mode" instead of switching to "content generation mode."

**Test results:** Inconclusive - needs more validation. Documented as experimental.

---

### Progress Visibility

For long-running tasks, you won't see streaming progress. The output appears all at once when complete.

**What to expect:**
```
# You see this immediately:
Process still running...

# Then after 5-10 minutes, all output appears at once:
✓ Created css/variables.css
✓ Created css/components.css
...
```

**Monitoring options:**
1. **SetTodoList tracking** - Check logs for progress markers
2. **Poll periodically** - `process action:poll sessionId:XXX`
3. **Background + notify** - Add wake trigger to prompt (see main SKILL.md)

---

### File Count Expectations

Set realistic expectations for file creation timing:

| File Count | Expected Duration | Timeout Setting |
|------------|-------------------|-----------------|
| 1-3 files | 2-4 minutes | 120s |
| 5-10 files | 5-8 minutes | 300s |
| 10-20 files | 8-12 minutes | 600s |
| 20+ files | 12-15+ minutes | 900s |

**Example:** 17 files typically takes ~10 minutes.

---

### Verification Step

**Always verify file creation completed successfully:**

```bash
# After Kimi finishes, check files exist
bash workdir:~/project command:"find . -type f | head -20"

# Or get file tree
bash workdir:~/project command:"tree -L 2"

# Verify specific file
bash workdir:~/project command:"ls -la index.html"
```

**What to check:**
- All expected files exist
- Files have content (non-zero size)
- Directory structure matches request

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
2. **Wording matters:** Use "write to disk" language for reliability
3. **Experimental fallback:** StrReplaceFile on empty files (before return-content)
4. **Last-resort fallback:** Return content (only when all else fails)
5. **Always non-PTY** for file operations
6. **Always explicit timeouts** (300s+ for creation, based on file count)
7. **Chunk large tasks** - don't ask for multi-file returns
8. **Verify after creation** - Use `find` or `tree` to confirm files
9. **No HTTP servers** in sandboxed environments
