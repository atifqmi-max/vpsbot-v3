# StoneNodes VPS Manager Bot

A Discord bot that deploys and manages Docker-based VPS containers, with full
`systemctl` support and **direct root SSH access** (real IP, port, username,
and root password — works with Termius, PuTTY, or plain `ssh`).

---

## ✨ Features

- 🐉 1-click VPS deploy from Discord (`/deploy` or `/create`)
- 🖥 Real Ubuntu/Debian containers with full `systemd` support
- 🔑 Direct root SSH — real IPv4 + NAT port + root password sent to the user's DM
- 🔄 Start / stop / restart / reinstall / regen-ssh commands
- 📊 Live VPS performance stats
- ⏰ Optional auto-expiry / auto-suspend
- 🛡 Admin-only management commands (suspend, delete, list all, fix stuck containers)
- 🐧 Optional Pterodactyl panel integration

---

## 📋 Requirements

Before you start, you need:

- A Linux server (Ubuntu 22.04/24.04 recommended) with a **public IP address**
- **Docker** installed and running on that server
- **Python 3.10+**
- A **Discord Bot Token** (from the Discord Developer Portal)
- Root/sudo access on the server (needed to run privileged Docker containers)

---

## 🚀 Setup — Step by Step

### 1. Clone the repository

```bash
git clone https://github.com/atifqmi-max/vpsbot-v3.git
cd vpsbot-v3
```

### 2. Install Docker (skip if already installed)

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable docker
sudo systemctl start docker
```

Verify it works:

```bash
docker ps
```

### 3. Install Python & create a virtual environment

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

python3 -m venv venv
source venv/bin/activate
```

### 4. Install the Python dependencies

```bash
pip install -r requirements.txt
```

### 5. Create your Discord Bot

1. Go to https://discord.com/developers/applications
2. Click **New Application** → give it a name (e.g. `StoneNodes`)
3. Go to **Bot** → click **Add Bot**
4. Under **Privileged Gateway Intents**, enable:
   - Server Members Intent
   - Message Content Intent
5. Click **Reset Token** → copy the token (you'll need it in Step 7)
6. Go to **OAuth2 → URL Generator**:
   - Scopes: `bot`, `applications.commands`
   - Bot Permissions: `Send Messages`, `Embed Links`, `Use Slash Commands`
7. Open the generated URL and invite the bot to your Discord server

### 6. Configure environment variables

Copy the example env file:

```bash
cp .env.example .env
nano .env
```

Fill in at minimum:

```env
DISCORD_TOKEN=your_bot_token_here
ADMIN_ROLE_ID=your_admin_role_id
ADMIN_USER_IDS=your_discord_user_id
SERVER_IP=your_server_public_ip
```

> **How to find your server's public IP:**
> ```bash
> curl -4 ifconfig.me
> ```

### 7. Open the SSH port range in your firewall

Each deployed VPS gets a random port in the `SSH_PORT_START`–`SSH_PORT_END`
range (default `20000-29999`), forwarded to that container's SSH server.
Open this range so users can actually connect:

```bash
sudo ufw allow 20000:29999/tcp
```

If you're on a cloud provider (AWS, GCP, Azure, Contabo, Hetzner, etc.), also
open this port range in the provider's **Security Group / Firewall** panel.

### 8. Run the bot

```bash
python3 stonenodes_bot.py
```

If everything is set up correctly you'll see:

```
Database ready.
Starting StoneNodes VPS Manager...
```

### 9. Keep it running 24/7 (recommended)

Use `systemd` so the bot restarts automatically and survives reboots.

Create `/etc/systemd/system/stonenodes.service`:

```ini
[Unit]
Description=StoneNodes VPS Manager Bot
After=docker.service network.target
Requires=docker.service

[Service]
Type=simple
WorkingDirectory=/path/to/vpsbot-v3
ExecStart=/path/to/vpsbot-v3/venv/bin/python3 stonenodes_bot.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable stonenodes
sudo systemctl start stonenodes
sudo systemctl status stonenodes
```

View logs any time with:

```bash
journalctl -u stonenodes -f
```

---

## 💬 Commands

### User commands
| Command | Description |
|---|---|
| `/create` | Deploy a new VPS for yourself |
| `/deploy` | 1-click VPS deploy (admin picks the user) |
| `/start <id>` | Start a stopped VPS |
| `/stop <id>` | Stop a running VPS |
| `/restart <id>` | Restart a VPS |
| `/reinstall <id>` | Wipe and rebuild a VPS (same specs, new root password) |
| `/regen-ssh <id>` | Get a fresh tmate backup SSH link |
| `/vps-performance <id>` | Live CPU/RAM/disk stats |
| `/my-vps` | List all your VPS instances |
| `/commands` | Show all available commands |

### Admin commands
| Command | Description |
|---|---|
| `/admin-add-user` | Grant a user hosting access |
| `/admin-remove-user` | Revoke a user's hosting access |
| `/extend-vps` | Change or remove a VPS's auto-expiry |
| `/suspend-vps` | Suspend a VPS |
| `/unsuspend-vps` | Reactivate a suspended VPS |
| `/remove-vps` | Permanently delete a VPS |
| `/fix-vps` | Force-remove a stuck container |
| `/list-vps` | List every VPS on the node |
| `/node-stats` | Host server resource usage |
| `/check-network` | Diagnose SERVER_IP + SSH port setup |
| `/ptero-status` | Pterodactyl panel connection status |

---

## 🔐 What the user receives

When a VPS is deployed, the owner gets a DM like this:

```
⚡ Your VPS is Ready
An admin deployed a VPS for you!

Instance ID       vps-xxxxxxx
OS                Ubuntu 24.04
RAM / CPU         16g / 4 vCPU
Shared IPv4       <your SERVER_IP>
SSH Port (NAT)    <random port from your range>
Username          root

Root Password
<16-character random password>

SSH Command
ssh root@<SERVER_IP> -p <port>
```

This command connects with any SSH client — Termius, PuTTY, or the terminal —
because it's a real `openssh-server` running inside the container with a real
port forwarded from your host.

---

## 🧪 Verifying your setup after install

Run `/check-network` (admin only) any time after the bot is running. It checks:
- Docker socket is reachable
- Your `.env` `SERVER_IP` actually matches this machine's real public IP
- The start of your SSH port range is free to bind locally

It **cannot** check your cloud firewall from inside the server itself. After
running it, test from a **different machine** (your own laptop) with:

```bash
nc -zv <SERVER_IP> <port>
```

`succeeded` = port is open and reachable. `timed out` / `refused` = firewall
or security group is still blocking it — go back to Step 7.

Once a VPS is deployed, test the real SSH command from your own machine:

```bash
ssh root@<SERVER_IP> -p <port>
```

---

## 🛠 Troubleshooting

**"Docker socket not found" error**
Docker isn't running. Start it with `sudo systemctl start docker`.

**Users can't connect over SSH**
Double-check step 7 — the `SSH_PORT_START`–`SSH_PORT_END` range must be open
both in `ufw`/`iptables` **and** in your cloud provider's firewall.

**Bot doesn't respond to slash commands**
Slash commands can take up to an hour to sync globally the first time. Kick
and re-invite the bot, or wait for Discord to sync.

**"cgroupns not supported" warning in logs**
This is safe to ignore on older `docker-py` versions — the bot automatically
falls back to a compatible container config.

---

## 📄 License

For personal/internal use. Modify freely for your own hosting community.
