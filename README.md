# 🤖 Event & Voice Attendance Tracking Discord Bot (`ActivityBot`)

A 100% free, self-hosted Discord Automation Service engineered for real-time voice channel attendance tracking, automated event lifecycle management, native Discord Scheduled Events integration, full-server attendance auditing, personal activity profiles, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py`) and deployed inside a strictly isolated **Debian 12 Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **Cloudflare WARP** architecture for encrypted outbound connectivity and full local network isolation. All insights, recaps, and leaderboards are handled natively inside Discord via modern Slash Commands and Rich Embeds.

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
 │  │  ┌─────────────────────────────────┐                                │  │
 │  │  │ events_history.json             │                                │  │
 │  │  │ scheduled_events.json           │                                │  │
 │  │  └─────────────────────────────────┘                                │  │
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

* **Outbound Traffic (`Cloudflare WARP`):** All outgoing gateway traffic from the `discord.py` daemon is routed through an active encrypted WireGuard tunnel (`warp-svc`, `warp=on`), masking the home IP and ensuring resilient, encrypted connectivity.
* **Hypervisor-Level LAN Isolation (`Proxmox Firewall`):** Egress traffic from the VM interface is explicitly filtered at the hypervisor bridge level (`vmbr0`). Outbound packets targeting the local private subnet (`192.168.0.0/24`) are dropped (`DROP`), permitting only default gateway routing for WAN/DNS resolution.

---

## 🌟 Key Technical Features

### 1. **Full Event Lifecycle Automation & Native Discord Events**
* **Automated Start & Optional Auto-End (`/activity schedule`):** Administrators can schedule events with a `start_time` and an optional `end_time`. The bot automatically starts tracking at the exact start minute and—if `end_time` is configured—automatically concludes the session, saves metrics, and posts a summary without manual intervention.
* **Native Discord Scheduled Events Integration:** Scheduling an event automatically creates a native server event in Discord's **Events** tab and synchronizes its lifecycle (`Scheduled` ➡️ `Active` ➡️ `Completed`).
* **15-Minute Multi-Channel Alerts:** Exactly 15 minutes before an event starts, the bot broadcasts an `@everyone` embed in the announcement channel (`#hello`) and sends Direct Message (DM) reminders to guild members.

### 2. **Real-Time Voice State Tracking & Fair Percentage Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user joins, disconnects, and channel switches in real-time across all voice channels.
* **True Event-Duration Percentages:** Computes each participant's attendance percentage relative to the **total duration of the event** (`0%–100%`), rather than channel headcount, ensuring fair and accurate participation metrics.
* **Smart User Filtering:** Automatically excludes Discord bots and non-human utility accounts (such as `Speaker` profiles).
* **Rolling 10-Event Persistence:** Stores structured event metrics (`start_time`, `end_time`, channel breakdowns) in `events_history.json`, maintaining a rolling window of the 10 most recent events.

### 3. **Automated Post-Event Channel Recaps**
* **Instant Summary Broadcasts:** Whenever an event concludes (either manually via `/activity end` or automatically at `end_time`), the bot posts a clean, formatted summary embed in `#hello` listing all voice channels and their participants.

### 4. **Server-Wide Attendance Audits & Leaderboards**
* **Full Server Attendance (`/frekwencja`):** Audits **every eligible member** on the Discord server (including members with `0` events attended) across the last 10 recorded events, sorted by attendance frequency and cumulative voice minutes, with automatic embed pagination for large rosters.
* **Last 3 Events Breakdown (`/lista`):** Displays a detailed channel-by-channel breakdown of the 3 most recent events with exact start/end timestamps, minutes spent, and event-duration attendance percentages.
* **Top 5 Leaderboard (`/ranking`):** Aggregates attendance frequency and total active voice time across a customizable number of recent events (up to 10) to generate a Top 5 podium leaderboard (`🏆`).
* **Personal Activity Profile (`/myactivity`):** Allows any server member to privately inspect their own participation history, total attended events, and cumulative voice minutes.

### 5. **Administrative JSON Backup & Export (`/export`)**
* **Data Portability:** Enables administrators to export the complete historical database (`events_history.json`) directly as a downloadable JSON attachment within Discord.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Framework:** Python 3.11, `discord.py 2.x`, `asyncio`
* **Data Storage:** Local JSON persistence (`events_history.json`, `scheduled_events.json`)
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Proxmox VE Firewall, Local Gateway DNS Routing
* **Process Management:** Systemd daemonized services (`systemctl`)

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity start [name]` | Slash (`/`) | **Administrator** | Manually starts background voice tracking with a custom event name, immediately logging all users currently in voice channels. |
| `/activity end` | Slash (`/`) | **Administrator** | Concludes the active event, computes final durations and percentages, saves to `events_history.json`, closes the native Discord event, and posts a participant recap in `#hello`. |
| `/activity schedule [name] [start_time] [end_time]` | Slash (`/`) | **Administrator** | Schedules a future event (`YYYY-MM-DD HH:MM`), creates a native Discord event, triggers 15-minute `@everyone` & DM reminders, auto-starts tracking, and optionally auto-ends at `end_time`. |
| `/export` | Slash (`/`) | **Administrator** | Exports the full event history as a downloadable JSON file for backups or external analysis. |
| `/frekwencja` | Slash (`/`) | Everyone | Lists **all server members** and displays how many of the last 10 events each person attended alongside their total active minutes. |
| `/lista` | Slash (`/`) | Everyone | Renders a summary of the last 3 events, grouped by voice channels with time ranges, individual durations, and event-duration percentages. |
| `/ranking [events_count]` | Slash (`/`) | Everyone | Generates a Top 5 leaderboard across the selected number of recent events (max 10), sorted by attendance count and total active time. |
| `/myactivity` | Slash (`/`) | Everyone | Displays your personal participation profile, total attended events, cumulative voice time, and recent event log. |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites

* Debian 12 OS inside Proxmox VE with Hypervisor Firewall enabled.
* Installed and connected Cloudflare WARP (`warp-cli connect`).
* **Server Members Intent** enabled in the Discord Developer Portal (required for `/frekwencja` and DM reminders).

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

### 3. Running as a Systemd Background Service (24/7 Operation)

Create a systemd service file at `/etc/systemd/system/discord-bot.service`:

```ini
[Unit]
Description=Discord Voice Attendance Bot
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

Enable, start, or restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable discord-bot
sudo systemctl restart discord-bot
```

---

## 🔐 Security & Resource Efficiency

1. **Encrypted Tunneling (`Cloudflare WARP`):** All traffic is routed securely through WireGuard (`warp=on`), ensuring total privacy and masking the host's public IP address.
2. **Multi-Layer Network Isolation:** Combines Proxmox VE hypervisor-level egress filtering (dropping access to local subnets) with local gateway controls.
3. **Zero Open Ports & Zero Cost:** No external domains, paid hosting, or public inbound ports are required.
4. **Ultra-Low Resource Footprint:** Consumes `<0.1%` average CPU and `~40 MB` RAM, maintaining negligible power draw (`~6–12W` idle on compact x86 hardware) during 24/7 operation.
