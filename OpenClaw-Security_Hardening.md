# OpenClaw - Security Hardening

Favorite: No
Archived: No
Created: 2 February 2026 21:10
Updated: 2 February 2026 22:22
Project: Serveri (https://www.notion.so/Serveri-2bf45010ad5d80068a3ecdbbad8ef1c9?pvs=21)

## A good security practice 1:

## Secure OpenClaw Installation for Wide Access on Hetzner VPS

Here's the **production-grade installation** balancing wide access (filesystem, shell, APIs) with hardened security. This assumes Ubuntu 24.04, your existing Caddy/code-server setup. Total time: ~45 minutes.[1][2][3][4]

### Prerequisites (5 min)

```bash
ssh root@your-hetzner-ip
apt update && apt upgrade -y
apt install -y curl git ufw fail2ban
ufw allow OpenSSH
ufw enable

```

### 1. Non-Root User & Isolation (5 min)

```bash
adduser --disabled-password --gecos "" openclaw
usermod -aG sudo openclaw
# Optional: Add to your project group for shared access
usermod -aG youruser openclaw  # Replace 'youruser' with your main user
su - openclaw

```

SSH as `openclaw` going forward: `ssh openclaw@your-hetzner-ip` (add SSH key to `~openclaw/.ssh/authorized_keys`).[1][2]

### 2. Install Node.js & Dependencies (5 min)

```bash
# As openclaw user
curl -fsSL <https://deb.nodesource.com/setup_lts.x> | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm
pnpm --version  # Verify

```

### 3. Clone & Install OpenClaw (10 min)

```bash
git clone <https://github.com/openclaw/openclaw.git>
cd openclaw
pnpm install

```

### 4. Secrets Manager: HashiCorp Vault (10 min)

```bash
# Install Vault (self-hosted, no cloud dependency)
wget <https://releases.hashicorp.com/vault/1.17.5/vault_1.17.5_linux_amd64.zip>
unzip vault_1.17.5_linux_amd64.zip
sudo install vault /usr/local/bin/
vault --version

# Quick setup (dev mode for single-node)
vault server -dev &
export VAULT_ADDR='<http://127.0.0.1:8200>'
vault kv put secret/openclaw ANTHROPIC_API_KEY=sk-ant-your-key

```

OpenClaw fetches at runtime:

```bash
# In openclaw config or .env
ANTHROPIC_API_KEY=$(vault kv get -field=ANTHROPIC_API_KEY secret/openclaw)

```

**No plaintext on disk**.[5][2]

### 5. Hardened Configuration (`~/.openclaw/config.json`)

```json
{
  "gateway": {
    "bind": "127.0.0.1:8080",  // Loopback only
    "auth": {
      "mode": "token",
      "token": "your-32-char-random-token-here"
    }
  },
  "agents": {
    "defaults": {
      "tools": ["filesystem", "shell_exec"],  // Wide access
      "scope": "workspace",  // Your project dirs
      "allowlist": ["your-phone-number", "your-telegram-id"]  // Only you
    }
  },
  "security": {
    "sandbox": false,  // Skipped per your preference
    "log_retention_days": 7
  }
}

```

chmod 600 `~/.openclaw/config.json`[3][1]

### 6. Firewall & Network (5 min)

```bash
# Gateway localhost only
sudo ufw allow from 127.0.0.1 to any port 8080

# Access via SSH tunnel (recommended):
ssh -L 8080:localhost:8080 openclaw@your-vps

# Or Caddy reverse proxy (if exposing):
# Caddyfile: openclaw.yourdomain.com { reverse_proxy localhost:8080 }
# Add basic auth or token validation

```

**No public exposure**.[3][4]

### 7. Wide Access Workspace Setup (5 min)

```bash
mkdir -p ~/workspace ~/workspace/projects
chmod 755 ~/workspace  # Agent can read/write
ln -s ~/workspace/projects /path/to/your/codeserver/projects  # Shared access

```

Agent can now:

- `fs_read/write` anywhere in `~/workspace`
- `shell_exec` commands (non-root)
- Manage your projects, Git repos, APIs[1]

### 8. Systemd Service (5 min)

```bash
# ~/openclaw.service
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
User=openclaw
Group=openclaw
WorkingDirectory=/home/openclaw/openclaw
ExecStart=/home/openclaw/openclaw/bin/openclaw gateway start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```

```bash
sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw
sudo systemctl status openclaw

```

Auto-restart, logs to journalctl.[1]

### 9. Messaging Integration (Telegram - Recommended)

```bash
# Create Telegram bot via @BotFather
# Add bot to private chat with yourself only
# config.json:
"messaging": {
  "telegram": {
    "bot_token": "your-bot-token-from-vault",
    "allowed_users": ["your-telegram-user-id"]
  }
}

```

**No WhatsApp ban risk**, official API.[1]

### 10. Monitoring & Maintenance (Ongoing)

```bash
# Daily cron: Log cleanup
0 2 * * * find ~/.openclaw/agents/*/sessions -mtime +7 -delete

# Weekly: Token rotation
# vault kv put secret/openclaw ANTHROPIC_API_KEY=new-key

```

`journalctl -u openclaw -f` for real-time monitoring.[3]

## Security Validation Checklist

| Check | Status | Command |
| --- | --- | --- |
| Non-root | ✅ | `ps aux |
| Loopback gateway | ✅ | `netstat -tlnp |
| No plaintext creds | ✅ | `grep -r ANTHROPIC ~/.openclaw` (not found) |
| Allowlist only you | ✅ | Check messaging config |
| Permissions tight | ✅ | `ls -la ~/.openclaw` (700/6Then prompt: |

## A good security practice 2:

## Best Practices to Reduce Plaintext Storage Risks

If using OpenClaw despite its risks, you can substantially reduce credential exposure through layered defenses. No single practice eliminates risk entirely, but the cumulative approach reduces it to ~90%.[1][2][3]

### Foundational: File Permissions

Lock down the directory containing all credentials:[4][5]

```bash
chmod 700 ~/.openclaw              # User only
chmod 600 ~/.openclaw/credentials/**/*.json
chmod 600 ~/.openclaw/agents/*/agent/auth-profiles.json

```

This blocks malware running as other users (not root). Combined with full-disk encryption on your Hetzner VPS, it prevents casual filesystem theft.[3][6]

### Primary: Secrets Manager (Replaces Plaintext Storage)

Stop storing API keys on disk entirely. Fetch credentials at runtime from a vault:[1][7]

```bash
# Instead of: export ANTHROPIC_API_KEY="sk-ant-..."
# Use: export ANTHROPIC_API_KEY=$(vault kv get -field=key secret/anthropic)

```

Options for your setup:

- **HashiCorp Vault** (self-hosted): Automatic rotation, audit logs, no disk storage[1][3]
- **Pulumi ESC**: Infrastructure-as-code secrets; good for Hetzner deployments[8]
- **AWS Secrets Manager**: Managed service with IAM integration[3]

This eliminates long-lived keys sitting in `~/.openclaw/`—the core vulnerability.[1][7]

### Execution Isolation: Docker Sandboxing

Run tool execution (file ops, shell commands) in isolated containers while the gateway stays on the host:[2][9]

```bash
# In openclaw config:
agents:
  defaults:
    sandbox:
      mode: "all"              # Sandbox every session
      scope: "session"         # New container per session, destroyed after
      workspaceAccess: "none"  # No filesystem access to host
      docker:
        network: "none"        # Blocks outbound exfiltration

```

Even if a prompt injection succeeds, the compromised tool runs in an isolated container that can't reach `~/.openclaw/` on the host.[2][3]

### Network Boundary: IP Allowlisting & Port Isolation

Gateway auth is mandatory, but add network controls:[5][6]

```bash
# Only accept gateway connections from localhost/VPS management IP
gateway.auth.token: $(openssl rand -hex 32)  # Strong random token
# Firewall: Allow only from 127.0.0.1 or your Hetzner VPC

```

Never expose port 8080 to `0.0.0.0`—this is how Shodan found the 954+ exposed instances. IP allowlisting alone prevented ~30-40% of public exposure in January 2026 research.[10][11][5]

### User Isolation

Run OpenClaw under a dedicated low-privilege OS user, not your own account:[6][12]

```bash
useradd -m -s /sbin/nologin openclaw
chown -R openclaw:openclaw ~/.openclaw

```

If your personal account is compromised elsewhere, attackers can't access the OpenClaw credentials (which are owned by the `openclaw` user).[3][12]

### Practical Minimal Stack for Your Setup

For a Hetzner self-hosted instance, this balances security with maintenance burden:[1][2][3][12]

1. **File permissions** (chmod 700/600)
2. **Secrets manager Vault**
3. **Dedicated OS user** (openclaw account)
4. **IP allowlisting** (gateway port only localhost) — 5 minutes, free