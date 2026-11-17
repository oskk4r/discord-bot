# 🤖 Event & Voice Attendance Tracking Discord Bot (`ActivityBot`)

A 100% free, self-hosted Discord Automation Service engineered for tactical guild event logistics, real-time voice channel attendance tracking, schedule management, AFK/Deafened detection, automated monthly Excel (`.xlsx`) reporting, visual attendance charts, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py`, `matplotlib` & `openpyxl`) and deployed inside a strictly isolated **Debian 12 Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **Cloudflare WARP** architecture for encrypted outbound connectivity and full local network isolation.

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
 │  │  │ (discord.py/mpl/xlsx)│       │ (Encrypted Outbound WireGuard) │  │  │
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

### 1. **Full Event Lifecycle & Schedule Management**
* **Automated Start & Optional Auto-End (`/activity schedule`):** Administrators can schedule events with a `start_time` and an optional `end_time`. The bot automatically starts tracking at the exact start minute and—if `end_time` is configured—automatically concludes the session, saves metrics, and posts a summary without manual intervention.
* **Schedule Inspection & Cancellation (`/activity list` & `/activity cancel`):** Officers can view all upcoming scheduled events in a numbered queue and cancel any scheduled event on the fly—which simultaneously removes it from the bot's queue and deletes the linked native Discord event.
* **Overlapping Event Resolution:** If a previous event is still running when a new scheduled or manual event starts, the bot seamlessly closes and saves the active event before initializing the new session.
* **Clean 15-Minute Reminder Alerts:** Exactly 15 minutes before a scheduled event starts, the bot broadcasts a clean `@everyone` embed reminder in `#hello` alongside Direct Message (DM) alerts to guild members.
* **Native Discord Scheduled Events Integration:** Scheduling an event automatically creates a native server event in Discord's **Events** tab (`ShotCaller` voice channel) and synchronizes its lifecycle (`Scheduled` ➡️ `Active` ➡️ `Completed`).
* **Smart `ShotCaller Bot` Safety Trigger:** Monitors the presence of the external **`ShotCaller Bot`** inside the `ShotCaller` voice channel. If `ShotCaller Bot` participates in an active event and subsequently disconnects for more than **2 minutes**, `ActivityBot` automatically concludes the event, saves the metrics, and posts the recap. (If `ShotCaller Bot` never joined, the event remains open until manually ended or its scheduled `end_time`).

### 2. **Multi-Channel Architecture, Monthly Excel Exports & Resets**
* **Dedicated Channel Routing:**
  * `#hello`: Event announcements and 15-minute advance reminders.
  * `#activity`: Automated post-event participant recaps with channel breakdowns, unique participant counts, active minutes, attendance percentages, AFK/Deafened flags, and an embedded attendance bar chart.
  * `#ranking`: Dedicated leaderboard channel for `/ranking` queries and automated monthly podium announcements.
  * `#my-activity`: Dedicated self-service channel where members check their personal stats via `/myactivity`.
  * `#admin`: Officer channel receiving automated formatted Excel (`.xlsx`) monthly reports prior to history resets.
* **50-Event Capacity, Monthly Excel Export & Reset:** Stores up to **50 events** per cycle. On the **1st day of every month**, a background task automatically:
  1. Generates a styled multi-sheet Excel report (`Attendance_Report_YYYY_MM.xlsx`) containing a **Monthly Summary** tab and a detailed **Event Breakdown** tab, sending it to `#admin`.
  2. Calculates the Top 5 most active members from the previous month and publishes the monthly ranking in `#ranking`.
  3. Clears `events_history.json` to start the new month fresh.

### 3. **Real-Time Voice State Tracking, AFK Detection & Visual Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user joins, disconnects, and channel switches in real-time across all voice channels.
* **AFK / Deafened Audio Detection (Mic Mutes Ignored):** Monitors `self_deaf` and server `deaf` states without penalizing microphone mutes (`mute`), as members primarily listen to shotcallers during events. Any participant who spends **$\ge 80\%$** of their event session fully deafened is automatically flagged with `🔇 (Deafened/AFK)` in post-event summaries, `/lista`, and Excel exports.
* **True Event-Duration Percentages:** Computes each participant's attendance percentage relative to the **total duration of the event** (`0%–100%`), excluding Discord bots and utility accounts (such as `_Speaker` profiles).
* **Visual Bar Charts (`/chart` & Auto-Recaps):** Uses `matplotlib` in headless mode (`Agg`) to dynamically render high-DPI PNG bar charts showing unique participant counts alongside event names and dates (defaulting to the last 10 events). Charts are generated on demand via `/chart` and automatically attached below every post-event summary in `#activity`.

### 4. **Server-Wide Audits, Low-Attendance Reports & On-Demand Excel Export**
* **Full Server Attendance (`/frekwencja`):** Audits **every eligible member** on the Discord server (including members with `0` events attended) across recorded events, sorted by attendance frequency and cumulative voice minutes, with automatic embed pagination.
* **Low Attendance Audit (`/brak_frekwencji`):** Filters and lists all eligible server members whose attendance falls **below 50%** across recent events (defaulting to the last 10 events), making it effortless to spot inactive members.
* **Last 3 Events Breakdown (`/lista`):** Displays a detailed channel-by-channel breakdown of the 3 most recent events with total unique participant counts, start/end timestamps, minutes spent, attendance percentages, and deafened flags.
* **Personal Activity Profile (`/myactivity`):** Allows any server member to privately inspect their own participation history, total attended events, and cumulative voice minutes.
* **On-Demand Excel Export (`/export`):** Enables administrators to generate and download a formatted `.xlsx` spreadsheet of the current event history directly inside Discord at any time.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Frameworks:** Python 3.11, `discord.py 2.x`, `matplotlib` (Headless `Agg` backend), `openpyxl`, `asyncio`
* **Data Storage:** Local JSON persistence (`events_history.json`, `scheduled_events.json`) & In-Memory `.xlsx` Generation
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Proxmox VE Firewall, Local Gateway DNS Routing
* **Process Management:** Systemd daemonized services (`systemctl`)

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity start [name]` | Slash (`/`) | **Administrator** | Manually starts background voice tracking with a custom event name across all voice channels (auto-closing any currently active event). |
| `/activity end` | Slash (`/`) | **Administrator** | Concludes the active event, computes final durations, percentages, and deafened flags, saves to `events_history.json`, closes the native Discord event, and posts a recap + bar chart in `#activity`. |
| `/activity schedule [name] [start_time] [end_time]` | Slash (`/`) | **Administrator** | Schedules a future event (`YYYY-MM-DD HH:MM`), creates a native Discord event, triggers 15-minute reminders & DMs, auto-starts tracking, and optionally auto-ends at `end_time`. |
| `/activity list` | Slash (`/`) | **Administrator** | Displays a numbered list of all currently scheduled upcoming events with their start and auto-end times. |
| `/activity cancel [number]` | Slash (`/`) | **Administrator** | Cancels and removes a scheduled event by its index number from `/activity list` and deletes the linked native Discord event. |
| `/export` | Slash (`/`) | **Administrator** | Exports the current event history as a formatted, multi-sheet Excel (`.xlsx`) report directly in Discord. |
| `/frekwencja` | Slash (`/`) | Everyone | Lists **all server members** and displays how many recorded events each person attended alongside their total active minutes. |
| `/brak_frekwencji [events_count]` | Slash (`/`) | Everyone | Displays a filtered report of members with **less than 50% attendance** across the selected number of recent events (default 10, max 50). |
| `/lista` | Slash (`/`) | Everyone | Renders a summary of the last 3 events, displaying total unique participants and channel-by-channel breakdowns with durations, percentages, and deafened flags. |
| `/ranking [events_count]` | Slash (`/`) | Everyone | Generates a Top 10 leaderboard across the selected number of recent events (max 50), sorted by attendance count and total active time. |
| `/myactivity` | Slash (`/`) | Everyone | Displays your personal participation profile, total attended events, cumulative voice time, and recent event log. |
| `/chart [events_count]` | Slash (`/`) | Everyone | Generates and sends a visual PNG bar chart illustrating unique participant counts, event names, and dates across recent events (default 10). |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites

* Debian 12 OS inside Proxmox VE with Hypervisor Firewall enabled.
* Installed and connected Cloudflare WARP (`warp-cli connect`).
* **Server Members Intent** enabled in the Discord Developer Portal (required for `/frekwencja`, `/brak_frekwencji`, and DM reminders).
* Dedicated text channels created on the server: `#hello`, `#activity`, `#ranking`, `#my-activity`, and `#admin`.

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
pip install discord.py matplotlib openpyxl
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
4. **Ultra-Low Resource Footprint:** Even with headless chart generation (`matplotlib`) and in-memory Excel exports (`openpyxl`), the bot consumes `<0.4%` average CPU and minimal RAM on Proxmox VE, maintaining negligible power draw during 24/7 operation.
