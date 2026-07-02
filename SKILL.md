---
name: mo-vps-connector
description: >
  Create and manage a stable SSH connection to a remote Linux VPS from any Kimi session.
  The SSH key and connection config are stored PERSISTENTLY in the workspace and survive
  across sessions. No need to regenerate keys for each new session. Any future Kimi session
  on this machine can connect automatically using the saved config. Includes tmux support
  for persistent terminal sessions. Triggered by: "connect to my VPS", "setup VPS connection",
  "stable SSH to server", "VPS access", "connect to remote server", "SSH to VPS",
  "run command on server", "tmux on VPS", "persistent SSH session".
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
"Start a tmux session on my VPS"
```

The skill auto-triggers from the description. The agent will:
1. Read `.vps_connection.json` from the workspace
2. Use the saved `vps_agent_key` private key
3. Connect to your VPS automatically
4. Execute the requested commands

---

## New Session Trigger Guide (If Auto-Trigger Fails)

**Problem:** In a new Kimi session, the agent has **zero memory** of previous sessions. It doesn't know your VPS exists or that there's a saved SSH key. The skill only activates if the right keywords are in the description.

**Solution:** Use one of these exact phrases to trigger the skill:

| Phrase to Say | What Happens |
|---------------|-------------|
| `"Connect to my VPS"` | Skill auto-triggers. Agent reads `.vps_connection.json` and connects. |
| `"SSH to my server"` | Skill auto-triggers. Agent uses saved key to connect. |
| `"Run command on my VPS"` | Skill auto-triggers. Agent connects and runs the command. |
| `"Use the VPS connector skill"` | Explicitly asks for the skill. Agent loads `mo-vps-connector`. |
| `"Check my server at 187.124.2.26"` | Skill triggers on "server" + IP. Agent finds the saved config. |

**If the skill STILL doesn't trigger**, tell the agent explicitly:

```
"Use the mo-vps-connector skill. The connection config is at 
/Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json
and the SSH key is at /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key"
```

This forces the agent to read the skill and use the saved files.

**Important:** Make sure you're in the **Mnemosyne workspace** (or the workspace where the files were saved). The agent only has access to the current workspace. If you switched workspaces, the files won't be found.

**To check which workspace you're in:**
```bash
pwd
ls -la /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json 2>/dev/null || echo "Config not found in this workspace"
```

**If the config is missing**, the agent will guide you through re-creating it (one-time setup again).

---

## SSH + tmux: Persistent Terminal Sessions

tmux lets you start long-running processes on the VPS that survive SSH disconnections. Perfect for:
- Long-running scripts or backups
- Keeping a terminal session alive while you disconnect
- Running multiple tasks in parallel within one SSH session
- Avoiding "command killed when SSH disconnects" issues

### Quick tmux Commands

```bash
# Start a new tmux session (named "kimi-work")
tmux new-session -d -s kimi-work

# Attach to the session (interact with it)
tmux attach -t kimi-work

# Run a command inside the tmux session without attaching
tmux send-keys -t kimi-work "apt update && apt upgrade -y" C-m

# List all tmux sessions
tmux ls

# Detach from a session (keep it running in background)
# Press Ctrl+B then D (inside tmux)
# Or: tmux detach

# Kill a session
tmux kill-session -t kimi-work
```

### tmux Session with Persistent Command

```bash
# Create a session and run a command that persists after disconnect
ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  root@187.124.2.26 "tmux new-session -d -s backup-session 'bash -c \"echo backup started at \$(date) > /tmp/backup.log && sleep 3600\"'"

# Check if the session is still running
ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  root@187.124.2.26 "tmux ls"

# Kill the session when done
ssh -i /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/vps_agent_key \
  -o StrictHostKeyChecking=no -o IdentitiesOnly=yes \
  root@187.124.2.26 "tmux kill-session -t backup-session"
```

### tmux Windows (Tabs)

```bash
# Create a new window in the session
tmux new-window -t kimi-work -n logs

# Send a command to a specific window
tmux send-keys -t kimi-work:logs "tail -f /var/log/syslog" C-m

# Switch between windows (when attached)
# Ctrl+B then 0, 1, 2... (window number)
# Ctrl+B then N (next window)
# Ctrl+B then P (previous window)
```

### tmux Panes (Split Screen)

```bash
# Split horizontally (two panes side by side)
tmux split-window -h -t kimi-work

# Split vertically (two panes stacked)
tmux split-window -v -t kimi-work

# Send command to a specific pane (0 = left/top, 1 = right/bottom)
tmux send-keys -t kimi-work.0 "htop" C-m
tmux send-keys -t kimi-work.1 "tail -f /var/log/nginx/access.log" C-m
```

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
| `tmux: command not found` | Install tmux: `apt install tmux -y` |
| `session not found` | The tmux session name doesn't exist. Use `tmux ls` to list active sessions. |

## Session FAQ

**Q: The skill didn't trigger in my new session. What do I do?**
A: New Kimi sessions have no memory. Say explicitly: `"Use the mo-vps-connector skill to connect to my VPS"` or `"Connect to my VPS using the saved config at /Users/mohamedfathy/Documents/Kimi/Workspaces/Mnemosyne/.vps_connection.json"`. This forces the agent to load the skill and read the config.

**Q: I get "Config not found" even though the skill triggered.**
A: You might be in a different workspace. Check with `pwd`. The config is saved in the **Mnemosyne** workspace. If you switched workspaces, either switch back or tell the agent the full path to the config file.

**Q: Do I need to do anything special when a new Kimi session starts?**
A: No. Just mention your VPS or server. The skill auto-triggers and the agent reads the saved config.

**Q: What if the key files are deleted?**
A: The agent will detect this and guide you through re-creating the key and config. You'll need to re-add the new public key to the VPS.

**Q: Can I use this from another machine?**
A: No. The private key is stored locally on this MacBook. For another machine, you'd need to either copy the key securely or generate a new one and add it to the VPS.

**Q: Is the SSH key safe?**
A: Yes. It's stored in your local workspace with `600` permissions (owner-read-only). The agent never exposes it in conversation.

**Q: Can I keep a tmux session running after the Kimi session ends?**
A: Yes! That's the whole point of tmux. Start a tmux session with a command, disconnect, and it keeps running. Check on it in the next Kimi session.

**Q: How do I check what tmux sessions are running on my VPS?**
A: Just say "List tmux sessions on my VPS" — the agent will run `tmux ls` for you.
