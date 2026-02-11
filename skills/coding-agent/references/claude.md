# Claude Code - Complete Guide

Best practices for using Claude Code CLI via OpenClaw.

## Quick Reference

```bash
# Interactive session
bash pty:true workdir:~/project command:"claude"

# One-shot task
bash pty:true workdir:~/project command:"claude 'Refactor auth module'"

# Background mode
bash pty:true workdir:~/project background:true command:"claude 'Build feature'"
```

---

## Requirements

Claude Code requires a git repository:

```bash
# ✅ Works
bash pty:true workdir:~/my-git-project command:"claude 'Your task'"

# ❌ Fails outside git repo
```

---

## Recommended Patterns

### Pattern 1: Interactive Session ✅

```bash
bash pty:true workdir:~/project command:"claude"
```

Claude Code is primarily designed for interactive use. It will:
- Start an interactive TUI
- Ask clarifying questions
- Show progress in real-time
- Request approval for changes

---

### Pattern 2: One-Shot with Prompt ✅

```bash
bash pty:true workdir:~/project command:"claude 'Add error handling to API routes'"
```

Claude will:
- Analyze the codebase
- Make the changes
- Exit when done

---

### Pattern 3: Background Mode ✅

```bash
# Start
bash pty:true workdir:~/project background:true command:"claude 'Implement feature'"

# Monitor
process action:log sessionId:XXX
```

---

## PTY Mode

**Always use PTY** - Claude Code is an interactive TUI application:

```bash
# ✅ Required
bash pty:true workdir:~/project command:"claude"

# ❌ Broken without PTY
bash workdir:~/project command:"claude"
```

---

## Timeouts

| Task | Timeout |
|------|---------|
| Quick tasks | 120s |
| Feature work | 300s |
| Complex refactoring | 600s |

---

## Comparison with Other Agents

| Feature | Claude Code | Codex | Kimi |
|---------|-------------|-------|------|
| Interactive TUI | ✅ Primary | ⚠️ Secondary | ❌ No |
| One-shot mode | ✅ Yes | ✅ Yes | ✅ Yes |
| Git required | ✅ Yes | ✅ Yes | ❌ No |
| PTY required | ✅ Yes | ✅ Yes | ❌ No |
| Background mode | ✅ Yes | ✅ Yes | ✅ Yes |

---

## Best Use Cases

1. **Complex refactoring** - Claude asks clarifying questions
2. **Exploration** - Interactive code understanding
3. **Architecture decisions** - Discussion before implementation
4. **Learning codebases** - "Explain this project to me"

---

## Tips

- Claude Code excels at interactive workflows
- Expect questions for ambiguous tasks
- Good for tasks requiring context understanding
- More conversational than other agents
