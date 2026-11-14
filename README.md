# 🤖 Event & Voice Attendance Tracking Discord Bot

A 100% free, self-hosted Discord Automation Service engineered for real-time voice channel attendance tracking, voice engagement analytics, automated event archiving, guild roster management, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py`) and deployed inside a strictly isolated **Debian Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **Cloudflare WARP** architecture for encrypted outbound connectivity and full local network isolation. All insights, stats, and rankings are handled natively inside Discord via modern Slash Commands and Rich Embeds.

---

## 📐 Infrastructure & Zero-Trust Network Architecture

To guarantee complete protection of the host's local home network (LAN/Wi-Fi) without relying on dedicated external hardware firewalls or router port forwarding, the infrastructure utilizes Proxmox firewall rules combined with a secure Cloudflare WARP encrypted WireGuard tunnel:

```
 ┌───────────────────────────────────────────────────────────────────────────┐
 │                         PROXMOX VE HYPERVISOR                             │
 │                  (L2/L3 VM Firewall: LAN Subnet Dropped)                  │
 │                                                                           │
 │  ┌─────────────────────────────────────────────────────────────────────┐  │
 │  │                   Debian 12 VM (/opt/discord-bot)                   │  │
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
* **Outbound Traffic (`Cloudflare WARP`):** All outgoing gateway traffic from the `discord.py` daemon is routed through an active encrypted WireGuard tunnel (`warp-svc`), masking the home IP, providing `warp=on` security, and ensuring resilient connectivity.
* **Hypervisor-Level LAN Isolation (`Proxmox Firewall`):** Egress traffic from the VM interface is explicitly filtered at the hypervisor bridge level (`vmbr0`). Outbound packets targeting the local private subnet (`192.168.0.0/24`) are dropped (`DROP`), permitting only default gateway routing for WAN/DNS resolution.

---

## 🌟 Key Technical Features

### 1. **Real-Time Voice State Tracking & Attendance Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user connection events, disconnects, and channel shifts in real-time between `/activity start` and `/activity end`.
* **Duration & Percentage Computation:** Calculates exact time spent by each participant on voice channels relative to total event duration, yielding precise attendance percentages (0–100%) while automatically filtering out bot accounts and non-human users (such as `Speaker` profiles).
* **Historical Persistence & Auto-Rotation:** Saves structured event metrics to local persistent storage (`events_history.json`), maintaining a rolling cap of the 10 most recent events. When an 11th event is recorded, the oldest is automatically purged.

### 2. **Rich Discord Visualizations (Embeds & Party Grouping)**
* **Organized Voice Party Breakdown (`/lista`):** Automatically groups participants by voice channels (e.g., `Party 1`, `Party 2`, `ShotCaller`), displaying the 3 most recent events with individual durations and active percentages in a clean Discord Embed.
* **Global Activity Ranking (`/ranking`):** Aggregates participation count and total time spent across a selectable number of recent stored events, generating a podium-style leaderboard (`🏆`) sorted first by frequency of attendance and second by total accumulated active time (Top 5).

### 3. **Low-Footprint Hardware Optimization**
* Engineered specifically for energy-efficient, 24/7 operation on compact x86 hardware (Lenovo ThinkCentre Tiny / Proxmox VE), keeping idle power draw minimal (~6–12W) while maintaining sub-second response times.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Framework:** Python 3.11, `discord.py 2.x`, `asyncio`
* **Data Storage:** Local JSON persistence (`events_history.json` with rolling 10-event cap)
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Proxmox VE Firewall, Local Gateway DNS Routing
* **Process Management:** Systemd daemonized services (`systemctl`)

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity start [name]` | Slash (`/`) | **Administrator** | Initializes background session tracking with a custom event name. Immediately captures existing voice members across all channels and begins logging duration for valid users until the session ends. |
| `/activity end` | Slash (`/`) | **Administrator** | Concludes the active event session, computes relative attendance percentages, commits results to local history storage (rolling 10-event cap), and clears active memory. |
| `/lista` | Slash (`/`) | Everyone | Renders a clean Discord Embed showing the breakdown of the last 3 events, grouped by voice channels with individual times, minutes, and percentages. |
| `/ranking [events_count]` | Slash (`/`) | Everyone | Generates a Top 5 leaderboard sorted primarily by event participation frequency and secondarily by total active time across the specified number of recent events. |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites
* Debian 12 OS inside Proxmox VE with Hypervisor Firewall enabled.
* Installed and connected Cloudflare WARP (`warp-cli connect`).

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

### 4. Running as a Systemd Background Service (24/7 Operation)
Create a systemd service file at `/etc/systemd/system/discord-bot.service`:
```ini
[Unit]
Description=Discord Attendance Bot
After=network.target warp-svc.service

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

1. **Encrypted Tunneling (`Cloudflare WARP`):** All traffic is routed securely through WireGuard (`warp=on`), ensuring total privacy and bypassing local ISP restrictions.
2. **Multi-Layer Network Isolation:** Combines Proxmox VE hypervisor-level egress filtering (blocking access to unauthorized subnets) with local gateway controls.
3. **Zero Open Ports & Zero Cost:** No external domains, paid hosting, or public tunnels are required. All features operate 100% for free 24/7.
4. **Token & Credential Protection:** `.env` files and session history are strictly excluded from version control via `.gitignore`.
