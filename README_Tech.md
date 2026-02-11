# OpenClaw Server - Technical Documentation

## System Architecture

This OpenClaw installation uses a **secure, vault-backed architecture** on Ubuntu 24.04 with the following components:

```
┌─────────────────────────────────────────────────────────┐
│                    User: openclaw                        │
│  ┌──────────────┐         ┌─────────────────────┐      │
│  │              │         │                     │      │
│  │  HashiCorp   │────────▶│   OpenClaw Gateway  │      │
│  │    Vault     │ Secrets │   (Port 8081)       │      │
│  │  (Port 8200) │         │                     │      │
│  └──────────────┘         └─────────────────────┘      │
│         │                           │                    │
│         │ API Keys                  │                    │
│         ▼                           ▼                    │
│  ┌──────────────┐         ┌─────────────────────┐      │
│  │  Encrypted   │         │   Telegram Bot      │      │
│  │   Storage    │         │   + AI Models       │      │
│  │  (~/.vault)  │         │   (Multi-provider)  │      │
│  └──────────────┘         └─────────────────────┘      │
└─────────────────────────────────────────────────────────┘
         │                           │
         │ Localhost only           │ Internet
         ▼                           ▼
   127.0.0.1                  Telegram API
                              GLM API
                              OpenRouter API
```

## Vault Secrets Inventory

Vault secrets are organized by purpose in separate paths:

### OpenClaw Core Secrets (`secret/data/openclaw`)

The following secret keys are stored in HashiCorp Vault for OpenClaw Gateway operations:

1. **GLM_API_KEY** - API key for GLM-4.7 model (via api.z.ai)
2. **OPENROUTER_API_KEY** - API key for OpenRouter models
3. **TELEGRAM_BOT_TOKEN** - Telegram bot authentication token
4. **TELEGRAM_USER_ID** - Telegram user ID for allowlist (your user ID)
5. **GITHUB_PAT** - GitHub Personal Access Token for repository operations
6. **KIMI_API_KEY** - API key for Kimi Code CLI (Moonshot AI)

### Skills APIs Credentials (`secret/data/skills-apis`)

Credentials for external APIs used by skills are stored in isolated paths:

- **`secret/skills-apis/amadeus/`**
  - `test_api_key` - Amadeus flight search API key
  - `test_api_secret` - Amadeus flight search API secret

- **`secret/skills-apis/github/`**
  - `token` - GitHub Personal Access Token for skills operations

- **`secret/skills-apis/context7/`**
  - `api_key` - Context7 documentation API key

- **`secret/skills-apis/gmail/`**
  - `email` - Gmail account email address (valtteri.melkko@gmail.com)
  - `app_password` - Gmail App Password (16-character IMAP authentication token)

- **`secret/skills-apis/stackexchange/`**
  - `value` - Stack Exchange API key for accessing Stack Overflow and other SE sites

- **`secret/skills-apis/serpapi/`**
  - `value` - SerpAPI key for Google search result scraping

- **`secret/skills-apis/youtube/`**
  - `value` - YouTube Data API v3 key for video search and analytics

**Note**: Skills APIs are stored separately from OpenClaw core secrets to prevent credential exposure in OpenClaw's environment while still being accessible to refactored skill scripts running as the `openclaw` user.

### Accessing Vault Secrets

```bash
# Load Vault environment
source ~/.vault/vault-env.sh

# Retrieve OpenClaw core secret
vault kv get -field=GLM_API_KEY secret/openclaw

# Retrieve skills API secret
vault kv get -field=test_api_key secret/skills-apis/amadeus

# View all keys in a path
vault kv get secret/skills-apis/github
```

## Installation Summary

### What Was Accomplished

1. **User & Environment Setup**
   - Created dedicated `openclaw` user for service isolation
   - Installed Node.js v24.13.0, npm, pnpm (user-local)
   - Installed HashiCorp Vault v1.18.3

2. **OpenClaw Installation**
   - Cloned OpenClaw repository from GitHub
   - Built from source using pnpm
   - Installed to `/home/openclaw/openclaw-source/`
   - Created symlink at `~/.npm-global/bin/openclaw`

3. **HashiCorp Vault Configuration**
   - Configured Vault server with file storage backend
   - Set up systemd service with security hardening
   - Initialized Vault with single unseal key
   - Enabled KV v2 secrets engine
   - Stored all API credentials securely

4. **OpenClaw Configuration**
   - Multi-model setup with fallback chain:
     - **Primary**: GLM-4.7 (GLM API)
     - **Fallback 1**: Mistral Devstral 2512 (OpenRouter)
     - **Fallback 2**: Google Gemini 2.5 Flash (OpenRouter)
     - **Fallback 3**: Meta Llama 3.3 70B Instruct (OpenRouter)
   - Telegram integration with user allowlist
   - Gateway bound to localhost (127.0.0.1:8081)
   - Workspace: `~/.openclaw/workspace/`

5. **Security Hardening**
   - **Secrets Management**: All API keys stored in Vault (no plaintext in config files)
   - **File Permissions**:
     - Config files: 600 (rw-------)
     - Directories: 700 (rwx------)
     - Service files: 600
   - **Network Isolation**: Gateway listens only on loopback (127.0.0.1)
   - **Systemd Hardening**:
     - `ProtectSystem=strict`
     - `PrivateTmp=yes`
     - `NoNewPrivileges=yes`
     - Services run as non-root user
   - **Environment Variables**: Secrets loaded from Vault at runtime via wrapper script
   - **Git Security**: Comprehensive `.gitignore` excludes all sensitive files

6. **Systemd Services**
   - **vault.service**: HashiCorp Vault server
     - Auto-start: ✅ Enabled
     - Restart policy: on-failure
   - **openclaw.service**: OpenClaw Gateway
     - Auto-start: ✅ Enabled
     - Restart policy: always
     - Depends on: vault.service
     - **Wrapper Script**: `~/.openclaw/start-gateway.sh` loads secrets from Vault before starting gateway

7. **Version Control**
   - Initialized git repository
   - Created security-focused `.gitignore`
   - Made initial commits with documentation
   - Pushed to GitHub fork: `valtterimelkko/openclaw`
   - Branch: `main` (merged from installation-docs)

8. **Skills APIs & Credential Management**
   - Created `secret/data/skills-apis/` vault path for external API credentials
   - Implemented credential fallback chain: Environment Variables → Vault → .bashrc
   - Populated vault with initial credentials:
     - `secret/skills-apis/amadeus/` - test_api_key, test_api_secret
     - `secret/skills-apis/github/` - token
     - `secret/skills-apis/context7/` - api_key
     - `secret/skills-apis/gmail/` - email, app_password
   - Created `CREDENTIAL_LOADING_GUIDE.md` in `.skills-global` for refactoring reference
   - Allows skills to load credentials securely from vault when running as openclaw user

9. **Bot Workspace Access via Symlinks & ACL**
   - Bot has secure read/write access to five directories via symlinks in `~/.openclaw/workspace/`:
     - `skills-global` → `/root/.skills-global`
     - `ai_product_visualizer` → `/root/ai_product_visualizer`
     - `buildersuite` → `/root/buildersuite`
     - `meta-project-for-mvps` → `/root/meta-project-for-mvps`
     - `opari` → `/root/opari`
   - **Access Control Implementation**:
     - ACL on `/root`: `user:openclaw:--x` (execute-only, allows traversal)
     - Target directories: Group-owned by `openclaw` with `rwx` permissions
     - Git safe.directory configured for all repositories
   - **Security Model**:
     - All folders are synced to GitHub, providing automatic backup and data loss protection
     - `/root/.bashrc` protected with `600` permissions (root-only), preventing credential exposure
   - **Risk Mitigation**: Even if bot is prompt-injected to run destructive commands, data can be recovered from GitHub
   - **Access Level**: Read/write to workspace directories, no root/system access, no credentials exposure

## Service Management

### Check Service Status

```bash
# Check both services
sudo systemctl status vault openclaw

# View real-time logs
sudo journalctl -u vault -f
sudo journalctl -u openclaw -f
```

### Restart Services

```bash
# Restart Vault only
sudo systemctl restart vault

# Restart OpenClaw only
sudo systemctl restart openclaw

# Restart both (Vault first, then OpenClaw)
sudo systemctl restart vault && sleep 2 && sudo systemctl restart openclaw
```

### Stop Services

```bash
# Stop OpenClaw gateway
sudo systemctl stop openclaw

# Stop Vault (this will also stop OpenClaw since it depends on Vault)
sudo systemctl stop vault

# Stop both
sudo systemctl stop openclaw vault
```

### Start Services

```bash
# Start Vault first
sudo systemctl start vault

# Wait for Vault to initialize, then start OpenClaw
sleep 3 && sudo systemctl start openclaw

# Or start both (Vault will start first due to dependency)
sudo systemctl start vault openclaw
```

### Disable Auto-Start (if needed)

```bash
# Disable services from starting on boot
sudo systemctl disable openclaw
sudo systemctl disable vault

# Re-enable auto-start
sudo systemctl enable vault openclaw
```

## Gateway Access

- **URL**: `http://127.0.0.1:8081`
- **Protocol**: WebSocket (ws://127.0.0.1:8081)
- **Auth Mode**: Token-based
- **Auth Token**: Stored in `~/.openclaw/openclaw.json`
- **Access**: Localhost only (not exposed to internet)

## Bot Workspace Access

The OpenClaw bot has read/write access to five directories via symlinks in the workspace. This allows the bot to help with development tasks while maintaining data safety through GitHub backups (where applicable).

### Accessible Repositories

| Symlink                 | Target Directory              | Purpose                     | GitHub Sync   |
| ----------------------- | ----------------------------- | --------------------------- | ------------- |
| `skills-global`         | `/root/.skills-global`        | Global skills library       | ✅ Synced     |
| `ai_product_visualizer` | `/root/ai_product_visualizer` | Product visualization tools | ✅ Synced     |
| `buildersuite`          | `/root/buildersuite`          | Builder suite projects      | ✅ Synced     |
| `meta-project-for-mvps` | `/root/meta-project-for-mvps` | Meta project for MVPs       | ✅ Synced     |
| `opari`                 | `/root/opari`                 | Opari project files         | ❌ Not synced |

### Access Implementation Details

#### Symlinks

Symlinks are created in `~/.openclaw/workspace/` pointing to `/root/` directories:

```bash
# Symlinks located at:
ls -la ~/.openclaw/workspace/ | grep -E "skills-global|ai_product|buildersuite|meta-project|opari"

# Each symlink points to a root-owned directory:
skills-global -> /root/.skills-global
ai_product_visualizer -> /root/ai_product_visualizer
buildersuite -> /root/buildersuite
meta-project-for-mvps -> /root/meta-project-for-mvps
opari -> /root/opari
```

#### ACL (Access Control List) Security

Access to `/root/` for the `openclaw` user is controlled via ACL, not standard Unix permissions:

```bash
# Check ACL on /root:
getfacl /root | grep openclaw
# Result: user:openclaw:--x (execute-only, no read)
```

This allows openclaw to:

- ✅ Traverse into `/root/` to reach symlinks
- ✅ Access symlinked directories and their contents
- ❌ List `/root/` directory contents
- ❌ Read `/root/.bashrc` or other root files

#### Protected Files

- `/root/.bashrc` has permissions `600` (root-only)
- Openclaw cannot read it even with ACL traversal
- This prevents credential exposure via prompt injection

#### Repository Permissions

All target directories are:

- Group-owned by `openclaw` group
- Have group read/write/execute permissions (`rwx`)
- Allow openclaw to read, write, and execute files

### Security Model

- **Access Scope**: Bot can read/write files within workspace symlinks only
- **No System Access**: Bot cannot modify system files, config, or root directories
- **No .bashrc Access**: Bot cannot read root's .bashrc (credentials protected)
- **Data Protection**: All accessible folders are synced to GitHub
- **Recovery**: Any destructive action can be recovered from GitHub history
- **Isolation**: Bot runs as non-root `openclaw` user with limited privileges
- **Risk Mitigation**: Even if bot is prompt-injected, GitHub provides automatic backup and recovery

### Git Operations

Git is configured to trust root-owned repositories from the openclaw user:

```bash
# Already configured in openclaw's git config:
git config --global --list | grep safe.directory
# Shows:
# safe.directory=/root/.skills-global
# safe.directory=/root/ai_product_visualizer
# safe.directory=/root/buildersuite
# safe.directory=/root/meta-project-for-mvps
# safe.directory=/root/opari
```

This allows openclaw to:

- Clone/pull from repositories
- Commit changes
- Push to remote

### Accessing Repositories from Bot

Ask the bot to work on files within these directories:

```
"Check /workspace/skills-global/config.json"
"Create a new file in /workspace/buildersuite/..."
"List files in /workspace/meta-project-for-mvps/"
"Update /workspace/ai_product_visualizer/..."
"Work on files in /workspace/opari/..."
```

The bot will work within the workspace and changes are preserved in the target directories (which sync to GitHub).

### Managing Workspace Access

#### Adding New Directories

To grant openclaw read/write access to a new directory in `/root/`:

**As root user:**

```bash
# 1. Set group ownership to openclaw
sudo chgrp -R openclaw /root/new-directory

# 2. Grant group read/write/execute permissions
sudo chmod -R g+rwx /root/new-directory

# 3. Create symlink in openclaw's workspace
sudo ln -s /root/new-directory /home/openclaw/.openclaw/workspace/new-directory

# 4. Add to git safe.directory list (as openclaw user)
sudo su - openclaw -c "git config --global --add safe.directory /root/new-directory"
```

#### Removing Directory Access

To revoke openclaw's access to a directory:

**As root user:**

```bash
# 1. Remove symlink from workspace
sudo rm /home/openclaw/.openclaw/workspace/directory-name

# 2. Remove from git safe.directory (as openclaw user)
sudo su - openclaw -c "git config --global --unset safe.directory /root/directory-name"

# 3. Revoke group permissions (optional)
sudo chmod -R g-rwx /root/directory-name
sudo chgrp -R root /root/directory-name
```

#### Verifying Access

To verify openclaw can access a directory:

**As openclaw user:**

```bash
# 1. Check symlink exists
ls -la ~/.openclaw/workspace/directory-name

# 2. List directory contents
ls ~/.openclaw/workspace/directory-name

# 3. Test git operations
cd ~/.openclaw/workspace/directory-name && git status
```

## Troubleshooting Guide

### Overview

Most OpenClaw issues stem from:

1. Secrets not loading from Vault
2. Gateway not starting properly
3. Configuration errors
4. Model/API connectivity issues

### Diagnosis Steps

#### 1. **Check Service Status**

```bash
sudo systemctl status vault openclaw

# Should show:
# vault.service - active (running)
# openclaw.service - active (running)
```

**Issue**: Service shows `inactive (dead)` or restart counter is high (>10)

- **Cause**: Gateway failed to start
- **Solution**: Check gateway logs (see step 2)

#### 2. **Check Service Logs**

```bash
# Real-time OpenClaw logs
sudo journalctl -u openclaw -f

# Last 50 lines of OpenClaw logs
sudo journalctl -u openclaw -n 50 --no-pager

# Vault logs
sudo journalctl -u vault -n 50 --no-pager
```

**Common Errors and Solutions**:

| Error                                        | Cause                              | Solution                                                         |
| -------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------- |
| `Missing env var "GLM_API_KEY"`              | Secrets not loaded from Vault      | Verify Vault is running, check secrets in Vault, restart service |
| `EADDRINUSE: address already in use :::8081` | Another process using port 8081    | Kill the process: `lsof -i :8081` or change port in config       |
| `Connection refused` (Vault)                 | Vault not running or not listening | Start Vault: `sudo systemctl start vault`                        |
| `Gateway not responding`                     | Gateway crashed or not listening   | Check listening ports: `ss -tlnp \| grep 8081`                   |

#### 3. **Test Vault Connection**

```bash
# Load Vault environment
source ~/.vault/vault-env.sh

# Check Vault is running
curl http://127.0.0.1:8200/v1/sys/seal-status

# Test secret retrieval
curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://127.0.0.1:8200/v1/secret/data/openclaw | grep -o '"[A-Z_]*":'
```

**Expected Output**:

- Seal status: `"sealed":false`
- Secrets: Keys like `"GLM_API_KEY"`, `"OPENROUTER_API_KEY"`, etc.

**If Sealed**: Unseal Vault manually:

```bash
UNSEAL_KEY=$(cat ~/.vault/init-keys.json | grep -o '"keys":\["[^"]*"' | cut -d'"' -f4)
curl -X PUT http://127.0.0.1:8200/v1/sys/unseal -d "{\"key\": \"$UNSEAL_KEY\"}"
```

#### 4. **Test Gateway Startup**

```bash
# Load secrets and test gateway
export PATH=~/.npm-global/bin:/usr/bin:/bin:$PATH
source ~/.openclaw/load-vault-secrets.sh

# Verify secrets loaded
echo "GLM_API_KEY is: ${GLM_API_KEY:0:10}..."

# Test gateway startup (will timeout after 5 seconds)
timeout 5 /home/openclaw/.npm-global/bin/openclaw gateway run || true
```

**Expected**: Gateway starts without "Missing env var" errors

**If Missing Environment Variables**:

1. Check Vault is running: `sudo systemctl status vault`
2. Test secret retrieval: See step 3
3. Regenerate environment file:
   ```bash
   ~/.openclaw/export-vault-secrets.sh
   cat ~/.vault/vault-secrets.env
   ```

#### 5. **Check Gateway Listening**

```bash
# Is gateway listening on port 8081?
ss -tlnp | grep 8081

# Should show: tcp  LISTEN  127.0.0.1:8081
```

**If Not Listening**:

- Gateway likely crashed or hung during startup
- Check logs (step 2)
- Try restarting: `sudo systemctl restart openclaw`

#### 6. **Test Gateway Connectivity**

```bash
# Once gateway is listening, test connection
timeout 3 curl http://127.0.0.1:8081/ || true

# Should get a response (might be connection refused if using WebSocket)
```

#### 7. **Run OpenClaw Doctor**

```bash
export PATH=~/.npm-global/bin:/usr/bin:/bin:$PATH
source ~/.openclaw/load-vault-secrets.sh

# Health check
openclaw doctor

# Get detailed diagnostics
openclaw doctor --verbose
```

**Look for**:

- ✅ "Config" section: Should show no errors
- ✅ "Gateway" section: Should show "Gateway running"
- ✅ "Telegram" section: Should show bot connected

### Common Issues and Solutions

#### **Issue: Bot doesn't respond to messages**

**Diagnosis**:

```bash
# 1. Check gateway is running
sudo systemctl status openclaw | grep Active

# 2. Check gateway is listening
ss -tlnp | grep 8081

# 3. Check Telegram configuration
grep -A 10 '"telegram"' ~/.openclaw/openclaw.json

# 4. Check Telegram bot token is valid
source ~/.vault/vault-env.sh
curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://127.0.0.1:8200/v1/secret/data/openclaw | \
  grep TELEGRAM_BOT_TOKEN
```

**Solutions**:

1. **Gateway Not Running**:
   - Restart: `sudo systemctl restart openclaw`
   - Check logs: `sudo journalctl -u openclaw -n 50`

2. **Telegram Token Invalid**:
   - Verify token in Vault is correct
   - Test with Telegram API: `curl https://api.telegram.org/bot<TOKEN>/getMe`
   - Update token in Vault if needed

3. **User Not Whitelisted**:
   - Check `TELEGRAM_USER_ID` in Vault matches your ID
   - Find your ID: Send `/start` to bot, check logs for "Pairing code"

4. **Model Connection Issues**:
   - Check GLM API key is valid
   - Check OpenRouter API key has repository access
   - Test model connectivity: `openclaw doctor`

#### **Issue: Gateway crashes on startup**

**Diagnosis**:

```bash
# Check if restarting frequently
systemctl status openclaw | grep "restart counter"

# View crash logs
sudo journalctl -u openclaw -n 100 | grep -E "Error|Failed|Exception"
```

**Solutions**:

1. **Missing Secrets**:
   - Verify Vault is running and unsealed
   - Test secret loading: `source ~/.openclaw/load-vault-secrets.sh`
   - Check for "Missing env var" errors

2. **Port Already in Use**:
   - Kill process: `lsof -i :8081 | grep -v COMMAND | awk '{print $2}' | xargs kill -9`
   - Change port in config and restart

3. **Configuration Error**:
   - Test config: `openclaw doctor`
   - Check JSON syntax: `cat ~/.openclaw/openclaw.json | jq .`

4. **Disk/Memory Issues**:
   - Check disk space: `df -h`
   - Check memory: `free -h`

#### **Issue: High CPU or Memory Usage**

```bash
# Monitor resource usage
watch -n 1 'ps aux | grep openclaw | grep -v grep'

# Check for memory leaks
ps -o pid,vsz,rss -p $(pgrep -f "openclaw gateway")"
```

**Solutions**:

1. Restart service: `sudo systemctl restart openclaw`
2. Check for hung processes: `sudo systemctl stop openclaw && sleep 2 && sudo systemctl start openclaw`
3. Review logs for error loops

### Advanced Debugging

#### **Enable Verbose Logging**

```bash
# Modify systemd service to add verbose flag
# Edit start-gateway.sh and add --verbose flag to openclaw command
nano ~/.openclaw/start-gateway.sh

# Change line:
# exec /home/openclaw/.npm-global/bin/openclaw gateway run
# To:
# exec /home/openclaw/.npm-global/bin/openclaw gateway run --verbose

sudo systemctl restart openclaw
sudo journalctl -u openclaw -f
```

#### **Manual Gateway Start (Foreground)**

```bash
export PATH=~/.npm-global/bin:/usr/bin:/bin:$PATH
source ~/.openclaw/load-vault-secrets.sh

# Stop systemd service first
sudo systemctl stop openclaw

# Run gateway manually to see all output
/home/openclaw/.npm-global/bin/openclaw gateway run

# Press Ctrl+C to stop
```

#### **Debug Wrapper Script**

```bash
# Run wrapper script with debug mode
bash -x ~/.openclaw/start-gateway.sh 2>&1 | head -100
```

### Performance Testing

#### **Test Model Response Time**

```bash
# Install jq for JSON parsing
sudo apt install jq

# Test GLM model response
time timeout 30 /home/openclaw/.npm-global/bin/openclaw agent "What is 2+2?"
```

#### **Monitor Gateway Health**

```bash
# Continuous health check
while true; do
  curl -s http://127.0.0.1:8081/health && echo " [OK]" || echo " [FAILED]"
  sleep 10
done
```

## File Locations

### Configuration

- **OpenClaw Config**: `~/.openclaw/openclaw.json` (600)
- **Vault Config**: `~/.vault/config.hcl` (600)
- **Vault Init Keys**: `~/.vault/init-keys.json` (600) - **CRITICAL: Backup this file!**
- **Vault Environment**: `~/.vault/vault-env.sh` (600)
- **OpenClaw Secrets Loader**: `~/.openclaw/load-vault-secrets.sh` (700)
- **OpenClaw Secrets Exporter**: `~/.openclaw/export-vault-secrets.sh` (700)
- **OpenClaw Gateway Wrapper**: `~/.openclaw/start-gateway.sh` (700)

### Services

- **Vault Service**: `/etc/systemd/system/vault.service`
- **OpenClaw Service**: `/etc/systemd/system/openclaw.service`

### Data

- **Vault Data**: `~/.vault/data/` (700)
- **OpenClaw Workspace**: `~/.openclaw/workspace/` (700)
  - **Symlinks to Bot-Accessible Directories**:
    - `skills-global` → `/root/.skills-global` (GitHub-synced)
    - `ai_product_visualizer` → `/root/ai_product_visualizer` (GitHub-synced)
    - `buildersuite` → `/root/buildersuite` (GitHub-synced)
    - `meta-project-for-mvps` → `/root/meta-project-for-mvps` (GitHub-synced)
    - `opari` → `/root/opari` (not synced)
- **OpenClaw Agents**: `~/.openclaw/agents/` (700)
- **OpenClaw Credentials**: `~/.openclaw/credentials/` (700)

### Source Code

- **OpenClaw Source**: `/home/openclaw/openclaw-source/`
- **OpenClaw Binary**: `~/.npm-global/bin/openclaw`

## Backup Recommendations

### Critical Files to Backup

1. **Vault Unseal Keys**: `~/.vault/init-keys.json`
   - **WITHOUT THIS, YOU CANNOT ACCESS YOUR SECRETS AFTER VAULT RESTART!**

2. **Vault Data**: `~/.vault/data/`
   - Contains all encrypted secrets

3. **OpenClaw Config**: `~/.openclaw/openclaw.json`
   - Gateway configuration and model setup

### Backup Command

```bash
# Create backup
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz \
  ~/.vault/init-keys.json \
  ~/.vault/config.hcl \
  ~/.vault/data/ \
  ~/.openclaw/openclaw.json \
  ~/.openclaw/*.sh

# Verify backup
tar -tzf ~/openclaw-backup-$(date +%Y%m%d).tar.gz

# Store backup securely (off-server)
```

## Testing Telegram Bot

### Prerequisites

- OpenClaw service must be running: `sudo systemctl status openclaw`
- Gateway must be listening: `ss -tlnp | grep 8081`
- Telegram bot token must be valid
- Your Telegram user ID must be in the allowlist

### Test Steps

1. **Find your Telegram bot**:
   - Open Telegram
   - Search for your bot by its username (from @BotFather)

2. **Start conversation**:
   - Send `/start` to the bot
   - If your user ID is whitelisted, no pairing code needed
   - Bot should acknowledge the start command

3. **Send a test message**:
   - Send any message like "Hello" or "What's the weather?"
   - Bot should respond using GLM-4.7 within 10-30 seconds
   - If GLM is unavailable, models fallback automatically

4. **Check logs if no response**:
   ```bash
   sudo journalctl -u openclaw -f --since "5 minutes ago"
   ```

### Expected Behavior

- Messages are received by the bot
- Bot responds with AI-generated text
- Response time: 10-30 seconds depending on model
- Models fallback automatically if primary fails

### Telegram Bot Troubleshooting

| Issue                           | Cause                         | Solution                                     |
| ------------------------------- | ----------------------------- | -------------------------------------------- |
| No `/start` response            | Bot not connected to Telegram | Check `TELEGRAM_BOT_TOKEN` in Vault          |
| Messages ignored after `/start` | Gateway not running           | Restart: `sudo systemctl restart openclaw`   |
| Wrong user receives messages    | User ID mismatch              | Check `TELEGRAM_USER_ID` in Vault and config |
| Slow responses                  | Model is slow/unavailable     | Check `openclaw doctor`, fallback models     |
| Connection timeout              | API unreachable               | Check internet, firewall, API status         |

## Security Notes

1. **Vault Root Token**: The root token in `~/.vault/init-keys.json` has unlimited access. Store securely.
2. **Unseal Keys**: Required to unseal Vault after restart. Backup in secure location.
3. **File Permissions**: Never change permissions on config files to more permissive values.
4. **Network Access**: Gateway is localhost-only. Access remotely via SSH tunnel if needed.
5. **Firewall**: No firewall rules needed for OpenClaw (not exposed to internet).
6. **Git**: Never commit `.vault/`, `.openclaw/`, or any files with credentials.
7. **Wrapper Script**: The `start-gateway.sh` script handles secret injection securely at runtime.

## Maintenance

### Update OpenClaw

```bash
# Stop service
sudo systemctl stop openclaw

# Update source
cd /home/openclaw/openclaw-source
git pull
pnpm install
pnpm build

# Restart service
sudo systemctl start openclaw
```

### Rotate API Keys

```bash
# Load Vault environment
source ~/.vault/vault-env.sh

# Update secret (example for GLM key)
curl -X POST http://127.0.0.1:8200/v1/secret/data/openclaw \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{
    "data": {
      "GLM_API_KEY": "new-key-here",
      "OPENROUTER_API_KEY": "existing-key",
      "TELEGRAM_BOT_TOKEN": "existing-token",
      "TELEGRAM_USER_ID": "existing-id",
      "GITHUB_PAT": "existing-pat"
    }
  }'

# Restart OpenClaw to load new keys
sudo systemctl restart openclaw
```

### Add New Keys to Vault

To add new API keys or configuration values to Vault (e.g., a new model provider or service token):

```bash
# Load Vault environment
source ~/.vault/vault-env.sh

# Retrieve current secrets
CURRENT=$(curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://127.0.0.1:8200/v1/secret/data/openclaw | jq '.data.data')

# Add new key while preserving existing ones
curl -X POST http://127.0.0.1:8200/v1/secret/data/openclaw \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{
    "data": {
      "GLM_API_KEY": "existing-value",
      "OPENROUTER_API_KEY": "existing-value",
      "TELEGRAM_BOT_TOKEN": "existing-value",
      "TELEGRAM_USER_ID": "existing-value",
      "GITHUB_PAT": "existing-value",
      "NEW_KEY_NAME": "new-value-here"
    }
  }'

# Verify the new key was added
curl -s -H "X-Vault-Token: $VAULT_TOKEN" \
  http://127.0.0.1:8200/v1/secret/data/openclaw | jq '.data.data | keys'

# Update OpenClaw config to reference the new key
# Edit ~/.openclaw/openclaw.json and add reference to the new key using ${NEW_KEY_NAME}
nano ~/.openclaw/openclaw.json

# Restart OpenClaw to load new configuration
sudo systemctl restart openclaw
```

**Important Notes**:

1. **Preserve Existing Keys**: Always include all existing keys in the update payload or they will be lost
2. **Environment Variable Format**: Use `${KEY_NAME}` format in config files to reference Vault secrets
3. **Restart Required**: OpenClaw must be restarted to load new environment variables
4. **Key Naming**: Use UPPERCASE_WITH_UNDERSCORES for consistency
5. **No Special Characters**: Keep key names alphanumeric and underscores only
6. **Vault Token**: The token from `~/.vault/vault-env.sh` must still be valid (re-login if expired)

**Example Use Cases**:

- Adding a new model provider API key (e.g., `CLAUDE_API_KEY`)
- Adding service credentials (e.g., `DATABASE_URL`, `REDIS_PASSWORD`)
- Adding notification service tokens (e.g., `SLACK_WEBHOOK_URL`)
- Adding rate limit or feature configuration keys

---

**Installation Date**: 2026-02-02
**Last Updated**: 2026-02-08 (Added Gmail credentials to vault)
**OpenClaw Version**: 2026.2.1
**Vault Version**: 1.18.3
**Node Version**: v24.13.0
**Server User**: openclaw
**Operating System**: Ubuntu 24.04 LTS
**Gateway Command**: `openclaw gateway run` (started via wrapper script)

### Bot Workspace Access

- **5 directories** accessible via workspace symlinks (4 GitHub-synced, 1 not synced)
- **Data protection**: All changes backed up to GitHub
- **Use case**: Development assistance with code files and projects
