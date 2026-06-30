---
name: mo-vps-connector
description: >
  Create and manage a stable SSH connection to a remote Linux VPS from any Kimi session.
  Stores the private key and connection parameters securely in the workspace so future sessions
  can connect without re-entering credentials. Triggered by: "connect to my VPS",
  "setup VPS connection", "stable SSH to server", "VPS access", "connect to remote server".
---

# mo-VPS Connector

Create and manage a stable SSH connection to your remote Linux VPS.

## Quick Start

### 1. Check if a connection already exists

```bash
cat /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json 2>/dev/null || echo "No connection configured"
```

### 2. Create a new connection (if not exists)

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -f /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key -N '' -C 'kimi-agent-access'

# Read the public key
cat /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key.pub
```

Then add the public key to the VPS:
```bash
# On the VPS:
echo '<PASTE_PUBLIC_KEY_HERE>' >> /root/.ssh/authorized_keys
sort -u /root/.ssh/authorized_keys -o /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 3. Save connection config

```bash
cat > /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json << 'EOF'
{
  "host": "187.124.2.26",
  "user": "root",
  "key_path": "/Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key",
  "ssh_command": "ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key -o StrictHostKeyChecking=no -o IdentitiesOnly=yes root@187.124.2.26"
}
EOF
```

### 4. Test the connection

```bash
ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  root@187.124.2.26 "uptime"
```

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
VPS_HOST="root@187.124.2.26"
SSH_OPTS="-i $VPS_KEY -o StrictHostKeyChecking=no -o IdentitiesOnly=yes"

vps_run() {
    ssh $SSH_OPTS $VPS_HOST "$@"
}

# Usage examples:
# vps_run "uptime"
# vps_run "df -h"
# vps_run "free -h"
```

## Important Security Notes

1. **Keep the private key safe** — Never share `vps_agent_key`. The public key (`vps_agent_key.pub`) is safe to share.
2. **Restrict SSH on VPS** — Ensure `PasswordAuthentication no` and `PermitRootLogin prohibit-password` are set.
3. **Rotate keys periodically** — Generate new keys every 6-12 months.
4. **Remove unused keys** — Periodically audit `/root/.ssh/authorized_keys` on the VPS.
5. **The connection file is stored locally** — Only accessible to your Kimi sessions on this machine.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Permission denied (publickey)` | Key not on VPS. Re-add the public key to `authorized_keys`. |
| `Connection refused` | VPS is down, SSH port blocked, or firewall issue. Check UFW/iptables. |
| `Connection timed out` | Network issue or VPS IP changed. Verify IP and network connectivity. |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Fix permissions: `chmod 600 vps_agent_key` |
| `Host key verification failed` | Use `-o StrictHostKeyChecking=no` or clear `~/.ssh/known_hosts`. |
