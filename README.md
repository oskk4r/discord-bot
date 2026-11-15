# 🤖 Event & Voice Attendance Tracking Discord Bot (`ActivityBot`)

A 100% free, self-hosted Discord Automation Service engineered for real-time voice channel attendance tracking, automated event lifecycle management, interactive RSVP alerts, visual attendance charts, automated monthly leaderboards, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py` & `matplotlib`) and deployed inside a strictly isolated **Debian 12 Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **Cloudflare WARP** architecture for encrypted outbound connectivity and full local network isolation.

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
 │  │  │ (discord.py + mpl)   │       │ (Encrypted Outbound WireGuard) │  │  │
 │  │  └──────────┬───────────┘       └────────────────────────────────┘  │  │
 │  │             │                                                       │  │
 │  │             ▼ (Local JSON Persistence)                              │  │
 │  │  ┌─────────────────────────────────┐                                │  │
 │  │  │ events_history.json (Max 50)    │                                │  │
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

### 1. **Full Event Lifecycle Automation & Interactive RSVP**
* **Automated Start & Optional Auto-End (`/activity schedule`):** Administrators can schedule events with a `start_time` and an optional `end_time`. The bot automatically starts tracking at the exact start minute and—if `end_time` is configured—automatically concludes the session, saves metrics, and posts a summary without manual intervention.
* **Interactive RSVP Alerts:** Exactly 15 minutes before a scheduled event starts, the bot broadcasts an `@everyone` embed in `#hello` equipped with live interactive buttons (**`I'll be there!`** / **`Can't make it`**) that dynamically tally participant responses, alongside Direct Message (DM) reminders.
* **Native Discord Scheduled Events Integration:** Scheduling an event automatically creates a native server event in Discord's **Events** tab (`ShotCaller` voice channel) and synchronizes its lifecycle (`Scheduled` ➡️ `Active` ➡️ `Completed`).
* **Smart `ShotCaller Bot` Safety Trigger:** Monitors the presence of the external **`ShotCaller Bot`** inside the `ShotCaller` voice channel. If `ShotCaller Bot` participates in an active event and subsequently disconnects for more than **2 minutes**, `ActivityBot` automatically concludes the event, saves the metrics, and posts the recap. (If `ShotCaller Bot` never joined, the event remains open until manually ended or its scheduled `end_time`).

### 2. **Multi-Channel Architecture & Automated Monthly Resets**
* **Dedicated Channel Routing:**
  * `#hello`: Event announcements, 15-minute warnings, and interactive RSVP buttons.
  * `#activity`: Automated post-event participant recaps with channel breakdowns, active minutes, and attendance percentages.
  * `#ranking`: Dedicated leaderboard channel for `/ranking` queries and automated monthly podium announcements.
  * `#my-activity`: Dedicated self-service channel where members check their personal stats via `/myactivity`.
* **50-Event Capacity & Monthly Leaderboard Reset:** Stores up to **50 events** per cycle. On the **1st day of every month**, a background task automatically calculates the Top 5 most active members from the previous month, publishes the monthly ranking in `#ranking` (e.g., *Monthly Ranking — November 2026*), and clears `events_history.json` to start the new month fresh.

### 3. **Real-Time Voice State Tracking & Visual Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user joins, disconnects, and channel switches in real-time across all voice channels.
* **True Event-Duration Percentages:** Computes each participant's attendance percentage relative to the **total duration of the event** (`0%–100%`), excluding Discord bots and utility accounts (such as `_Speaker` profiles).
* **Visual Bar Charts (`/chart`):** Uses `matplotlib` in headless mode (`Agg`) to dynamically render high-DPI PNG bar charts comparing unique participant counts across recent events.

### 4. **Server-Wide Audits & Data Export**
* **Full Server Attendance (`/frekwencja`):** Audits **every eligible member** on the Discord server (including members with `0` events attended) across recorded events, sorted by attendance frequency and cumulative voice minutes, with automatic embed pagination.
* **Last 3 Events Breakdown (`/lista`):** Displays a detailed channel-by-channel breakdown of the 3 most recent events with exact start/end timestamps, minutes spent, and attendance percentages.
* **Personal Activity Profile (`/myactivity`):** Allows any server member to privately inspect their own participation history, total attended events, and cumulative voice minutes.
* **Administrative JSON Export (`/export`):** Enables administrators to download the complete `events_history.json` database directly inside Discord.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Frameworks:** Python 3.11, `discord.py 2.x`, `matplotlib` (Headless `Agg` backend), `asyncio`
* **Data Storage:** Local JSON persistence (`events_history.json`, `scheduled_events.json`)
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Proxmox VE Firewall, Local Gateway DNS Routing
* **Process Management:** Systemd daemonized services (`systemctl`)

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity start [name]` | Slash (`/`) | **Administrator** | Manually starts background voice tracking with a custom event name across all voice channels. |
| `/activity end` | Slash (`/`) | **Administrator** | Concludes the active event, computes final durations and percentages, saves to `events_history.json`, closes the native Discord event, and posts a participant recap in `#activity`. |
| `/activity schedule [name] [start_time] [end_time]` | Slash (`/`) | **Administrator** | Schedules a future event (`YYYY-MM-DD HH:MM`), creates a native Discord event, triggers 15-minute RSVP alerts & DMs, auto-starts tracking, and optionally auto-ends at `end_time`. |
| `/export` | Slash (`/`) | **Administrator** | Exports the full event history as a downloadable JSON file for backups or external analysis. |
| `/frekwencja` | Slash (`/`) | Everyone | Lists **all server members** and displays how many recorded events each person attended alongside their total active minutes. |
| `/lista` | Slash (`/`) | Everyone | Renders a summary of the last 3 events, grouped by voice channels with time ranges, individual durations, and event-duration percentages. |
| `/ranking [events_count]` | Slash (`/`) | Everyone | Generates a Top 10 leaderboard across the selected number of recent events (max 50), sorted by attendance count and total active time. |
| `/myactivity` | Slash (`/`) | Everyone | Displays your personal participation profile, total attended events, cumulative voice time, and recent event log. |
| `/chart [events_count]` | Slash (`/`) | Everyone | Generates and sends a visual PNG bar chart illustrating unique participant counts across recent events. |

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
pip install discord.py matplotlib
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
4. **Ultra-Low Resource Footprint:** Even with headless chart generation (`matplotlib`), the bot consumes `<0.3%` average CPU and minimal RAM on Proxmox VE, maintaining negligible power draw during 24/7 operation.
