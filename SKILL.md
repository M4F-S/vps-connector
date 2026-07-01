---
name: mo-vps-connector
description: >
  Create and manage a stable SSH connection to a remote Linux VPS from any Kimi session.
  The SSH key and connection config are stored PERSISTENTLY in the workspace and survive
  across sessions. No need to regenerate keys for each new session. Any future Kimi session
  on this machine can connect automatically using the saved config. Triggered by:
  "connect to my VPS", "setup VPS connection", "stable SSH to server", "VPS access",
  "connect to remote server", "SSH to VPS", "run command on server".
---

# mo-VPS Connector

Create and manage a stable, persistent SSH connection to your remote Linux VPS that **survives across Kimi sessions**.

## How Sessions Work (Important!)

**You do NOT need to regenerate the SSH key for each new Kimi session.**

Here's how persistence works:

- The SSH private key (`vps_agent_key`) is stored in your workspace folder: `~/Documents/Kimi/Workspaces/Mnemosyne/`
- The connection config (`.vps_connection.json`) is also stored in the same workspace
- Both files **persist across sessions** — they are saved to disk, not memory
- Any new Kimi session on this machine can read these files and connect automatically
- The public key is already installed on your VPS in `/root/.ssh/authorized_keys`

### What This Means

| Scenario | What Happens |
|----------|-------------|
| New Kimi session starts | The agent reads `.vps_connection.json` and `vps_agent_key` from the workspace automatically |
| You say "connect to my VPS" | The agent uses the saved key and config — no password or key regeneration needed |
| You say "run uptime on my server" | The agent connects via SSH using the persistent key and runs the command |
| This machine restarts | Files are still there — connection works immediately |
| You want a new key | You can regenerate, but it's optional for security rotation only |

---

## Quick Start

### 1. Check if a connection already exists

```bash
cat /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json 2>/dev/null || echo "No connection configured"
```

### 2. Create a new connection (only needed once!)

```bash
# Generate SSH key pair (only do this once, ever)
ssh-keygen -t ed25519 -f /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key -N '' -C 'kimi-agent-access'

# Read the public key
cat /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key.pub
```

Then add the public key to the VPS (one-time setup):
```bash
# On the VPS:
echo '<PASTE_PUBLIC_KEY_HERE>' >> /root/.ssh/authorized_keys
sort -u /root/.ssh/authorized_keys -o /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 3. Save connection config (one-time setup)

```bash
cat > /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json << 'EOF'
{
  "host": "<VPS_IP_OR_HOSTNAME>",
  "user": "<VPS_USER>",
  "key_path": "/Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key",
  "ssh_command": "ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key -o StrictHostKeyChecking=no -o IdentitiesOnly=yes <VPS_USER>@<VPS_IP_OR_HOSTNAME>"
}
EOF
```

### 4. Test the connection

```bash
ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  <VPS_USER>@<VPS_IP_OR_HOSTNAME> "uptime"
```

---

## Using from a New Session

**No setup needed.** Just mention your VPS and the agent will auto-connect:

```
"Run apt update on my VPS"
"Check disk space on the server"
"What's the load on my VPS?"
"Connect to my VPS and show me running processes"
"Scan my server for malware"
```

The skill auto-triggers from the description. The agent will:
1. Read `.vps_connection.json` from the workspace
2. Use the saved `vps_agent_key` private key
3. Connect to your VPS automatically
4. Execute the requested commands

---

## Python Connection Helper

```python
import paramiko
import json
import os

CONFIG_PATH = "/Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json"

def load_vps_config():
    with open(CONFIG_PATH) as f:
        return json.load(f)

def connect_to_vps():
    config = load_vps_config()
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(
        config["host"],
        username=config["user"],
        key_filename=config["key_path"],
        timeout=30
    )
    return client

def run_on_vps(cmd, timeout=60):
    client = connect_to_vps()
    try:
        stdin, stdout, stderr = client.exec_command(cmd, timeout=timeout)
        out = stdout.read().decode('utf-8', errors='replace').strip()
        err = stderr.read().decode('utf-8', errors='replace').strip()
        return out + ("\n[STDERR] " + err if err.strip() else "")
    finally:
        client.close()
```

## Bash Connection Helper

```bash
#!/bin/bash
VPS_KEY="/Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key"
VPS_HOST="<VPS_USER>@<VPS_IP_OR_HOSTNAME>"
SSH_OPTS="-i $VPS_KEY -o StrictHostKeyChecking=no -o IdentitiesOnly=yes"

vps_run() {
    ssh $SSH_OPTS $VPS_HOST "$@"
}

# Usage examples:
# vps_run "uptime"
# vps_run "df -h"
# vps_run "free -h"
```

---

## Important Security Notes

1. **Keep the private key safe** — Never share `vps_agent_key`. The public key (`vps_agent_key.pub`) is safe to share.
2. **Restrict SSH on VPS** — Ensure `PasswordAuthentication no` and `PermitRootLogin prohibit-password` are set.
3. **Rotate keys periodically** — Generate new keys every 6-12 months for security.
4. **Remove unused keys** — Periodically audit `/root/.ssh/authorized_keys` on the VPS and remove old/compromised keys.
5. **The connection files are stored locally** — Only accessible to your Kimi sessions on this machine.
6. **Session persistence** — The key and config are files on disk, not in memory. They survive app restarts, machine reboots, and new Kimi sessions.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Permission denied (publickey)` | Key not on VPS. Re-add the public key to `authorized_keys`. |
| `Connection refused` | VPS is down, SSH port blocked, or firewall issue. Check UFW/iptables. |
| `Connection timed out` | Network issue or VPS IP changed. Verify IP and network connectivity. |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Fix permissions: `chmod 600 vps_agent_key` |
| `Host key verification failed` | Use `-o StrictHostKeyChecking=no` or clear `~/.ssh/known_hosts`. |
| `No connection configured` (new session) | The `.vps_connection.json` file may have been deleted. Recreate it using the steps above. |

## Session FAQ

**Q: Do I need to do anything special when a new Kimi session starts?**
A: No. Just mention your VPS or server. The skill auto-triggers and the agent reads the saved config.

**Q: What if the key files are deleted?**
A: The agent will detect this and guide you through re-creating the key and config. You'll need to re-add the new public key to the VPS.

**Q: Can I use this from another machine?**
A: No. The private key is stored locally on this MacBook. For another machine, you'd need to either copy the key securely or generate a new one and add it to the VPS.

**Q: Is the SSH key safe?**
A: Yes. It's stored in your local workspace with `600` permissions (owner-read-only). The agent never exposes it in conversation.
