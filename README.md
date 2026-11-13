# 🤖 Event & Voice Attendance Tracking Discord Bot

A 100% free, self-hosted Discord Automation Service engineered for real-time voice channel attendance tracking, voice engagement analytics, automated event archiving, guild roster management, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py`) and deployed inside a strictly isolated **Debian Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **Cloudflare WARP** architecture for encrypted outbound connectivity and full local network isolation. All insights, stats, and rankings are handled natively inside Discord via modern Slash Commands and Rich Embeds.

---

## 📐 Infrastructure & Zero-Trust Network Architecture

To guarantee complete protection of the host's local home network (LAN/Wi-Fi) without relying on dedicated external hardware firewalls or router port forwarding, the infrastructure utilizes software firewalls combined with secure Cloudflare WARP routing:

```
 ┌───────────────────────────────────────────────────────────────────────────┐
 │                         PROXMOX VE HYPERVISOR                             │
 │                  (L2/L3 VM Firewall: LAN Subnet Dropped)                  │
 │                                                                           │
 │  ┌─────────────────────────────────────────────────────────────────────┐  │
 │  │                   Debian 12 VM (/opt/discord-bot)                   │  │
 │  │               (UFW Host Firewall: Strict Allowlist)                 │  │
 │  │                                                                     │  │
 │  │  ┌──────────────────────┐       ┌────────────────────────────────┐  │  │
 │  │  │ Discord Bot Service  │ ────► │ Cloudflare WARP Client Tunnel  │  │  │
 │  │  │ (Async discord.py)   │       │ (Encrypted Outbound WireGuard) │  │  │
 │  │  └──────────┬───────────┘       └────────────────────────────────┘  │  │
 │  │             │                                                       │  │
 │  │             ▼ (Local JSON Persistence)                              │  │
 │  │  ┌──────────────────────┐                                           │  │
 │  │  │ events_history.json  │                                           │  │
 │  │  └──────────────────────┘                                           │  │
 │  └─────────────────────────────────────────────────────────────────────┘  │
 └───────────────────────────────────────────────────────────────────────────┘
                                                      │ Encrypted Gateway
                                                      ▼
                                        ┌───────────────────────────┐
                                        │        Discord API        │
                                        │  (Zero Open Router Ports) │
                                        └───────────────────────────┘
```

### Isolation Model
* **Outbound Traffic (`Cloudflare WARP`):** All outgoing gateway traffic from the `discord.py` daemon is routed through an encrypted WireGuard tunnel (`CloudflareWARP`), masking the home IP and ensuring resilient connectivity.
* **Hypervisor-Level LAN Isolation (`Proxmox Firewall`):** Egress traffic from the VM interface is explicitly filtered at the hypervisor bridge level (`vmbr0`). Outbound packets targeting the local private subnet (`192.168.0.0/24`) are dropped (`DROP`), permitting only default gateway routing (`192.168.0.1`) for WAN/DNS resolution.
* **Guest OS Hardening (`UFW`):** Default `DENY` policy on all incoming interfaces, with a strict IP-restricted allowlist permitting SSH administration (`22/tcp`) exclusively from the primary workstation.

---

## 🌟 Key Technical Features

### 1. **Real-Time Voice State Tracking & Attendance Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user connection events, disconnects, and channel shifts in real-time between `/activity_start` and `/activity_end`.
* **Duration & Percentage Computation:** Calculates exact time spent by each participant on voice channels relative to total event duration, yielding precise attendance percentages (0–100%).
* **Historical Persistence & Auto-Rotation:** Saves structured event metrics to local persistent storage (`events_history.json`), maintaining a rolling cap of the 10 most recent events to prevent unbounded storage growth.

### 2. **Rich Discord Visualizations (Embeds & Party Grouping)**
* **Organized Voice Party Breakdown (`/lista`):** Automatically groups participants by voice channels (e.g., `Party 1`, `Party 2`, `ShotCaller`), displaying individual durations and active percentages in a clean Discord Embed.
* **Global Activity Ranking (`/ranking`):** Aggregates total time spent across the last 10 stored events, generating a podium-style leaderboard (`👑`, `🥈`, `🥉`) of the most active guild members.

### 3. **Low-Footprint Hardware Optimization**
* Engineered specifically for energy-efficient, 24/7 operation on compact x86 hardware (Lenovo ThinkCentre Tiny / Proxmox VE), keeping idle power draw minimal (~6–12W) while maintaining sub-second response times.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Framework:** Python 3.11, `discord.py 2.x`, `asyncio`
* **Data Storage:** Local JSON persistence (`events_history.json` with rolling 10-event limit)
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Proxmox VE Firewall, Linux `UFW`
* **Process Management:** Systemd daemonized services (`systemctl`)

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity_start [name]` | Slash (`/`) | **Administrator** | Initializes background session tracking. Immediately captures existing voice members and begins logging duration for all users until the session ends. |
| `/activity_end` | Slash (`/`) | **Administrator** | Concludes the active event session, computes relative attendance percentages, commits results to local history storage, and clears active memory. |
| `/lista` | Slash (`/`) | Everyone | Renders a clean Discord Embed showing the latest event breakdown, grouped by voice channels (Party 1, Party 2, etc.) with individual times and percentages. |
| `/ranking` | Slash (`/`) | Everyone | Generates a global leaderboard summarizing total active time across the last 10 stored events. |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites
* Debian 12 OS inside Proxmox VE with Hypervisor Firewall enabled.
* Active Cloudflare WARP connection (`warp-cli connect`).

### 1. Directory Structure Setup
```bash
sudo mkdir -p /opt/discord-bot
cd /opt/discord-bot
```

### 2. Virtual Environment & Dependencies
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install discord.py python-dotenv
```

### 3. Environment Configuration (`.env`)
Create a `.env` file in `/opt/discord-bot`:
```env
DISCORD_TOKEN=your_bot_token_here
```

### 4. Host Firewall Hardening (`UFW`)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from <ADMIN_WORKSTATION_IP> to any port 22 proto tcp
sudo ufw enable
```

### 5. Running as a Systemd Background Service (24/7 Operation)
Create a systemd service file at `/etc/systemd/system/discord-bot.service`:
```ini
[Unit]
Description=Discord Attendance Bot
After=network.target

[Service]
User=root
WorkingDirectory=/opt/discord-bot
ExecStart=/opt/discord-bot/venv/bin/python3 /opt/discord-bot/main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable discord-bot
sudo systemctl start discord-bot
```

---

## 🔐 Security Considerations

1. **Multi-Layer Network Isolation:** Combines Proxmox VE hypervisor-level egress filtering (blocking access to the local `192.168.0.0/24` subnet) with Debian's host-based `UFW` firewall.
2. **Zero Open Ports & Zero Cost:** No external domains, paid hosting, or public tunnels (`cloudflared`) are used. All features operate 100% for free 24/7 with outbound traffic secured via Cloudflare WARP.
3. **Token & Credential Protection:** `.env` files and session history are strictly excluded from version control via `.gitignore`.
4. **Data Integrity:** Graceful handling of storage files prevents daemon crashes during unexpected machine shutdowns.
