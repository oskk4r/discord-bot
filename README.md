
# 🤖 Event & Voice Attendance Tracking Discord Bot

An enterprise-grade, self-hosted Discord Automation Service engineered for real-time channel attendance tracking, voice engagement analytics, automated event archiving, and secure system operations. 

Built with **Python 3.11** (`discord.py`) and deployed inside an isolated **Debian Linux Virtual Machine** managed by **Proxmox VE**, utilizing **Cloudflare WARP encrypted network tunneling** for outbound connectivity bypass and network security.

---

## 📐 Infrastructure & Architecture Overview

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                        PROXMOX VE HYPERVISOR                           │
 │                                                                        │
 │  ┌──────────────────────────────────────────────────────────────────┐  │
 │  │                  Debian VM (/opt/discord-bot)                    │  │
 │  │                                                                  │  │
 │  │  ┌─────────────────────┐       ┌──────────────────────────────┐  │  │
 │  │  │ Python 3.11 Service │ ────► │ Cloudflare WARP Client Tunnel│  │  │
 │  │  │ (Async discord.py)   │       │ (Encrypted Outbound WireGuard)│  │  │
 │  │  └─────────────────────┘       └──────────────┬───────────────┘  │  │
 │  └───────────────────────────────────────────────┼──────────────────┘  │
 └──────────────────────────────────────────────────┼─────────────────────┘
                                                    │ SSL/TLS (Port 443)
                                                    ▼
                                           ┌─────────────────┐
                                           │ Discord API /   │
                                           │ Gateway Engine  │
                                           └─────────────────┘
```

---

## 🌟 Key Technical Features

### 1. **Real-Time Voice State Tracking & Attendance Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user connection events, disconnects, and channel shifts in real-time between `/activity_start` and `/activity_end`.
* **Duration & Percentage Computation:** Calculates exact time spent by each participant on voice channels relative to total event duration, yielding precise attendance percentages (0–100%).
* **Historical Persistence & Auto-Rotation:** Saves structured event metrics (`name`, `key`, `duration_minutes`, `attendance`) to JSON. Maintains a strict rolling cap of the **10 most recent events** to prevent unbounded storage expansion and excessive memory consumption.

### 2. **Slash Command & Event Synchronization**
* Integrates both native **Discord Application Slash Commands** (`/activity_start`, `/activity_end`) and legacy prefix handling (`!lista`).
* Strict Role-Based Access Control (RBAC) ensuring administrative-only execution for privileged management routines.

### 3. **Infrastructure Security & Network Bypassing**
* **No Open Inbound Ports:** Operates entirely behind outbound-only network routing, eliminating exposure to public internet attacks (Zero-Trust Security Alignment).
* **Cloudflare WARP Tunnel Integration:** Employs WireGuard-based tunneling (`warp-client`) to bypass upstream ISP network restrictions and guarantee 99.9% gateway uptime.
* **Environment Secret Isolation:** Strict separation of authentication variables (`.env`) from codebase runtime.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Framework:** Python 3.11, `discord.py 2.x`, `asyncio`
* **Configuration Management:** `python-dotenv`
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Linux Virtual Networking
* **Process Management:** `screen` / `systemd` daemonized services

---

## 🔄 Architecture Evolution: From Static Snapshots to Real-Time Tracking

Initially, the tracking architecture captured a **single static snapshot** at the exact moment `/activity_start` was issued. This approach had a structural limitation: users joining mid-event or leaving before `/activity_end` were either missed or falsely represented.

### The Refactored Real-Time Pipeline
To ensure true event analytics, the system was refactored to an **event-driven state-machine**:

1. **State Initialization (`/activity_start`):** Captures current timestamps and maps all active channel members to dynamic in-memory session timers (`active_sessions`).
2. **Asynchronous Telemetry (`on_voice_state_update`):** 
   * On channel join/leave/move events, the bot calculates elapsed session time and accumulates total voice seconds per user (`user_seconds`).
3. **Analytics & Aggregation (`/activity_end`):**
   * Finalizes active timers, computes total event duration $\Delta t$, and evaluates relative attendance percentage per participant:
     $$\text{Attendance (\%)} = \min\left(100, \left(\frac{\text{User Voice Seconds}}{\Delta t}\right) \times 100\right)$$
   * Sorts participants by active engagement level and writes the top historical metrics to structured persistence (`events_history.json`).

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity_start [name]` | Slash (`/`) | **Administrator** | Initializes background session tracking. Immediately captures existing voice members and begins logging duration for all users until the session ends. |
| `/activity_end` | Slash (`/`) | **Administrator** | Concludes the active event session, computes relative attendance percentages for each user, commits results to `events_history.json` (max 10 records), and clears active memory. |
| `!lista` | Text (`!`) | Everyone | Renders a formatted breakdown of the 3 most recent events, including total event duration (in minutes) and individual attendance percentages. |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites
* Debian 12 OS inside Proxmox VE.
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

### 4. Running as a Daemon Service (Background Persistence)
To launch the service in a persistent virtual terminal session:
```bash
screen -S discord-bot
source venv/bin/activate
python3 -u main.py
```
*Detach from session:* Press `Ctrl + A`, followed by `D`.

---

## 🔐 Security Considerations

1. **Token Protection:** `.env` files are excluded from version control via `.gitignore`.
2. **Network Footprint:** Does not expose host IP or open external listening sockets (DNAT/Port Forwarding disabled).
3. **Data Integrity:** Graceful handling of corrupted/missing JSON files prevents daemon crashes during unexpected machine shutdowns.
