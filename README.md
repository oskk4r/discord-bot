# 🤖 Event & Voice Attendance Tracking Discord Bot & Guild Management System

An enterprise-grade, self-hosted Discord Automation Service and Web Dashboard engineered for real-time channel attendance tracking, voice engagement analytics, automated event archiving, guild roster management, and hardened Zero-Trust system operations.

Built with **Python 3.11** (`discord.py` + `FastAPI`) and deployed inside a strictly isolated **Debian Linux Virtual Machine** managed by **Proxmox VE**, utilizing a **hybrid Cloudflare WARP + Cloudflare Tunnel (`cloudflared`) architecture** for encrypted outbound connectivity, secure inbound web exposure, and full local network isolation.

---

## 📐 Infrastructure & Zero-Trust Network Architecture

To guarantee complete protection of the host's local home network (LAN/Wi-Fi) without relying on dedicated external hardware firewalls or router port forwarding, the infrastructure utilizes a dual-layer software firewall combined with split Cloudflare tunneling:

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
 │  │  └──────────┬───────────┘       └───────────────┬────────────────┘  │  │
 │  │             │                                   │                   │  │
 │  │             ▼ (Shared SQLite WAL DB)            │                   │  │
 │  │  ┌──────────────────────┐       ┌───────────────┼────────────────┐  │  │
 │  │  │ Web Roster Dashboard │ ◄──── │ Cloudflare Tunnel (cloudflared)│  │  │
 │  │  │ (FastAPI / Uvicorn)  │       │ (Zero-Trust Inbound Reverse Proxy)│ │
 │  │  └──────────────────────┘       └───────────────┬────────────────┘  │  │
 │  └─────────────────────────────────────────────────┼───────────────────┘  │
 └────────────────────────────────────────────────────┼──────────────────────┘
                                                      │ SSL/TLS (Port 443)
                                                      ▼
                                        ┌───────────────────────────┐
                                        │ Discord API & Public Web  │
                                        │ (Zero Open Router Ports)  │
                                        └───────────────────────────┘
```

### Hybrid Tunneling & Isolation Model
* **Outbound Bot Traffic (`Cloudflare WARP`):** All outgoing gateway traffic from the `discord.py` daemon is routed through an encrypted WireGuard tunnel (`CloudflareWARP`), masking the home IP and ensuring resilient connectivity.
* **Inbound Web Dashboard (`Cloudflare Tunnel / cloudflared`):** The upcoming FastAPI web portal is exposed securely over HTTPS via an outbound-only `cloudflared` daemon. Zero inbound ports (no DNAT/Port Forwarding) are opened on the home router.
* **Hypervisor-Level LAN Isolation (`Proxmox Firewall`):** Egress traffic from the VM interface is explicitly filtered at the hypervisor bridge level (`vmbr0`). Outbound packets targeting the local private subnet (`192.168.0.0/24`) are dropped (`DROP`), permitting only default gateway routing (`192.168.0.1`) for WAN/DNS resolution.
* **Guest OS Hardening (`UFW`):** Default `DENY` policy on all incoming interfaces, with a strict IP-restricted allowlist permitting SSH administration (`22/tcp`) exclusively from the primary workstation.

---

## 🌟 Key Technical Features

### 1. **Real-Time Voice State Tracking & Attendance Analytics**
* **Event-Driven Listener (`on_voice_state_update`):** Dynamically tracks user connection events, disconnects, and channel shifts in real-time between `/activity_start` and `/activity_end`.
* **Duration & Percentage Computation:** Calculates exact time spent by each participant on voice channels relative to total event duration, yielding precise attendance percentages (0–100%).
* **Historical Persistence & Auto-Rotation:** Saves structured event metrics (`name`, `key`, `duration_minutes`, `attendance`) to persistent storage, maintaining rolling caps to prevent unbounded storage growth on low-power hardware.

### 2. **Slash Command & Event Synchronization**
* Integrates both native **Discord Application Slash Commands** (`/activity_start`, `/activity_end`) and legacy prefix handling (`!lista`).
* Strict Role-Based Access Control (RBAC) ensuring administrative-only execution for privileged management routines.

### 3. **Low-Footprint Hardware Optimization**
* Engineered specifically for energy-efficient, 24/7 operation on compact x86 hardware (Lenovo ThinkCentre Tiny / Proxmox VE), keeping idle power draw minimal (~6–12W) while maintaining sub-second response times.

---

## 🗺️ Roadmap & Future Vision (Guild Management Suite)

The project is actively evolving from a standalone attendance tracker into a comprehensive, lightweight **Guild Management & Roster Platform**:

* [ ] **SQLite (WAL Mode) Database Migration:** Transitioning from flat JSON storage to a concurrent SQLite database with Write-Ahead Logging enabled, allowing simultaneous read/write operations between the Discord bot and the web backend.
* [ ] **Guild Roster & Static Party Web Dashboard (`FastAPI` + `Tailwind CSS`):**
  * Public/authenticated web portal displaying active guild members, character classes, specializations, and assigned raid groups (*Static Parties*).
  * Live integration with Discord attendance metrics—displaying each player's historical event participation percentage directly alongside their character card.
* [ ] **In-Discord Character & Absence Management:**
  * Slash commands (`/character add`, `/static assign`) allowing players to register and update their in-game characters directly via Discord.
  * Automated AFK/Leave of Absence tracking (`/absence`) that updates roster availability on the web dashboard and prevents unfair attendance penalties.
* [ ] **Discord OAuth2 & Cloudflare Zero Trust Access:** Seamless authentication for guild officers and members viewing protected roster analytics.
* [ ] **Lightweight Audio / Soundboard Integration:** Low-CPU guild soundboard or stream player utilizing external audio offloading (Lavalink) to preserve host energy efficiency.

---

## 🛠️ Stack & Technologies Used

* **Language & Core Framework:** Python 3.11, `discord.py 2.x`, `asyncio`
* **Web Backend (Planned/In-Progress):** `FastAPI`, `Uvicorn`, `Jinja2`, `aiosqlite`
* **Configuration Management:** `python-dotenv`
* **Virtualization & OS:** Proxmox VE, Debian GNU/Linux 12 (Bookworm)
* **Networking & Security:** Cloudflare WARP (`warp-cli`), Cloudflare Tunnel (`cloudflared`), Proxmox VE Firewall, Linux `UFW`
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
   * Sorts participants by active engagement level and writes the top historical metrics to structured persistence.

---

## 📊 Command Reference

| Command | Type | Permissions | Description |
| :--- | :--- | :--- | :--- |
| `/activity_start [name]` | Slash (`/`) | **Administrator** | Initializes background session tracking. Immediately captures existing voice members and begins logging duration for all users until the session ends. |
| `/activity_end` | Slash (`/`) | **Administrator** | Concludes the active event session, computes relative attendance percentages for each user, commits results to storage, and clears active memory. |
| `!lista` | Text (`!`) | Everyone | Renders a formatted breakdown of the 3 most recent events, including total event duration (in minutes) and individual attendance percentages. |

---

## 🚀 Deployment & Local Environment Setup

### Prerequisites
* Debian 12 OS inside Proxmox VE with Hypervisor Firewall enabled.
* Active Cloudflare WARP connection (`warp-cli connect`) & `cloudflared` daemon installed.

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

### 5. Running as a Daemon Service (Background Persistence)
To launch the service in a persistent virtual terminal session:
```bash
screen -S discord-bot
source venv/bin/activate
python3 -u main.py
```
*Detach from session:* Press `Ctrl + A`, followed by `D`.

---

## 🔐 Security Considerations

1. **Multi-Layer Network Isolation:** Combines Proxmox VE hypervisor-level egress filtering (blocking access to the local `192.168.0.0/24` subnet) with Debian's host-based `UFW` firewall.
2. **Zero Open Ports:** No port forwarding or DNAT rules exist on the perimeter router. Outbound traffic traverses Cloudflare WARP, while inbound web traffic will be brokered strictly via Cloudflare Tunnel (`cloudflared`).
3. **Token & Credential Protection:** `.env` files, database files, and Cloudflare tunnel credentials (`.json`/`.pem`) are strictly excluded from version control via `.gitignore`.
4. **Data Integrity:** Graceful handling of corrupted/missing storage files prevents daemon crashes during unexpected machine shutdowns.
