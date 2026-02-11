# Codex CLI - Complete Guide

Best practices for using OpenAI Codex CLI via OpenClaw.

## Quick Reference

```bash
# One-shot with auto-approve
bash pty:true workdir:~/project command:"codex exec --full-auto 'Add error handling'"

# Background mode
bash pty:true workdir:~/project background:true command:"codex exec --full-auto 'Refactor auth'"

# Review PR
bash pty:true workdir:~/review command:"codex review --base main"
```

---

## Requirements

### Git Repository Required

Codex **will not run** outside a trusted git directory:

```bash
# ✅ Works - inside git repo
bash pty:true workdir:~/my-project command:"codex exec 'Your task'"

# ❌ Fails - not a git repo
bash pty:true workdir:/tmp/no-git command:"codex exec 'Your task'"
# Error: Not a git repository
```

**For scratch work:**
```bash
SCRATCH=$(mktemp -d) && cd $SCRATCH && git init
bash pty:true workdir:$SCRATCH command:"codex exec 'Build a landing page'"
```

---

## Recommended Patterns

### Pattern 1: One-Shot Execution ✅

```bash
bash pty:true workdir:~/project command:"codex exec --full-auto 'Add form validation to Contact component'"
```

**Flags:**
- `exec` - One-shot mode, exits when done
- `--full-auto` - Auto-approves changes in workspace
- `--yolo` - No sandbox, no approvals (fastest, use with caution)

---

### Pattern 2: Background for Long Tasks ✅

```bash
# Start
bash pty:true workdir:~/project background:true command:"codex exec --full-auto 'Build full dashboard'"
# Returns: sessionId

# Monitor
process action:log sessionId:XXX
process action:poll sessionId:XXX

# Kill if needed
process action:kill sessionId:XXX
```

---

### Pattern 3: PR Reviews ✅

**Critical:** Never review in OpenClaw's own folder!

```bash
# Clone to temp
REVIEW_DIR=$(mktemp -d)
git clone https://github.com/user/repo.git $REVIEW_DIR
cd $REVIEW_DIR && gh pr checkout 130

# Review
bash pty:true workdir:$REVIEW_DIR command:"codex review --base origin/main"

# Cleanup
trash $REVIEW_DIR
```

**Or use git worktree:**
```bash
git worktree add /tmp/pr-review pr-branch
bash pty:true workdir:/tmp/pr-review command:"codex review --base main"
git worktree remove /tmp/pr-review
```

---

## Parallel PR Reviews (The Army)

```bash
# Fetch all PR refs
git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'

# Launch reviews in parallel
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #86. Diff: origin/main...origin/pr/86'"
bash pty:true workdir:~/project background:true command:"codex exec 'Review PR #87. Diff: origin/main...origin/pr/87'"

# Post results
gh pr comment 86 --body "$(cat review-86.md)"
```

---

## Flags Reference

| Flag | Effect |
|------|--------|
| `exec "prompt"` | One-shot execution, exits when done |
| `--full-auto` | Sandboxed + auto-approves in workspace |
| `--yolo` | No sandbox, no approvals (fast, dangerous) |
| `review` | Review mode for PRs |
| `--base` | Base branch for reviews |

---

## PTY Mode

**Always use PTY with Codex** - it works reliably:

```bash
# ✅ Correct
bash pty:true workdir:~/project command:"codex exec --full-auto 'Your task'"

# ❌ Missing PTY - may have broken output
bash workdir:~/project command:"codex exec --full-auto 'Your task'"
```

---

## Timeouts

| Task | Timeout |
|------|---------|
| Quick edits | 120s |
| Feature implementation | 300s |
| Full builds | 600s |
| PR reviews | 180s |

---

## Troubleshooting

### "Not a git repository"

**Solution:** Initialize git or navigate to a git repo:
```bash
cd ~/project && git status  # Verify
# or
git init
```

### Broken output

**Solution:** Add `pty:true` - Codex needs PTY for proper rendering.

---

## Key Differences from Kimi

| Aspect | Codex | Kimi |
|--------|-------|------|
| Git required | ✅ Yes | ❌ No |
| PTY mode | ✅ Required | ❌ Avoid for file ops |
| Sandbox | ✅ Default | ❌ No sandbox |
| Auto-approve | `--full-auto` | `--yolo` |
| One-shot flag | `exec` | `-p` |

---

## Summary

- **Always use PTY** - Codex requires it
- **Always in git repo** - Otherwise it won't start
- **`exec` for one-shots** - Clean exit when done
- **`--full-auto` for building** - Auto-approves changes
- **Temp folders for reviews** - Never review in live project
