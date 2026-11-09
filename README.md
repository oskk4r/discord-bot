[readme_md.md](https://github.com/user-attachments/files/34604367/readme_md.md)

# 🤖 Event & Voice Attendance Tracking Discord Bot

An enterprise-grade, self-hosted Discord Automation Service engineered for real-time channel attendance tracking, automated event archiving, and secure system operations. 

Built with **Python 3.11** (`discord.py`) and deployed inside an isolated **Debian Linux Virtual Machine** managed by **Proxmox VE**, utilizing **Cloudflare WARP encrypted network tunneling** for outbound connectivity bypass and network security.

---

## 📐 Infrastructure & Architecture Architecture Overview

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
                                                    │ Ssl/TLS (Port 443)
                                                    ▼
                                           ┌─────────────────┐
                                           │ Discord API /   │
                                           │ Gateway Engine  │
                                           └─────────────────┘
```

---

## 🌟 Key Technical Features

### 1. **Voice Channel State Capturing & Event Tracking**
* **Dynamic Event Initiation:** Captures full guild voice state snapshots on demand.
* **Granular Member Resolution:** Maps channel names to active member display names across all server channels in real-time.
* **Persistent JSON Storage:** Saves event metadata (`name`, `key`, `timestamp`, `party_data`) with rotating history management (maintaining latest 10 runs to minimize memory and I/O overhead).

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

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity_start [name]` | Slash (`/`) | **Administrator** | Captures all active voice channels and stores current users under a date-stamped event key (e.g., `Wargames_10_11_2026`). |
| `/activity_end` | Slash (`/`) | **Administrator** | Ends the active event session, commits data to `events_history.json`, and cleans session memory. |
| `!lista` | Text (`!`) | Everyone | Renders a formatted breakdown of the 3 most recent events and recorded attendance lists. |

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
