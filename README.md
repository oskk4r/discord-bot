# Secure Discord Bot on Proxmox LXC

Dokumentacja wdrożenia bezpiecznego, izolowanego bota Discord w kontenerze Debian 12 LXC na hoście Proxmox.

## 🏗️ Architektura i Sieć
- **Proxmox Host & Network**: 
  - Zewnętrzny interfejs: `vmbr0` (NAT)
  - Wewnętrzny, izolowany mostek dla kontenera: `vmbr1` (Podsieć: `10.0.0.0/24`)
- **Kontener LXC (ID: 100)**:
  - Adres IP: `10.0.0.50`
  - Brama domyślna: `10.0.0.1` (zdefiniowana w `100.conf`)
  - Firewall Proxmoxa (`pve-firewall`) wyłączony na interfejsie (`firewall=0`), a bezpieczeństwo i izolacja wejściowa obsłużone bezpośrednio regułami `iptables` na hoście.

## 🔒 Polityka Bezpieczeństwa (iptables na hoście)
- Ruch przychodzący z zewnątrz (`NEW`) do kontenera (`10.0.0.50`) jest całkowicie zablokowany.
- Kontener ma dozwolony ruch wychodzący (`ESTABLISHED, RELATED` oraz wyjście na świat przez NAT), dzięki czemu może pobierać pakiety (`apt update`) i łączyć się z API Discorda, pozostając w pełni odizolowanym od sieci lokalnej i internetu (brak dostępu z zewnątrz).

## 🐍 Środowisko Aplikacji
- **Katalog roboczy**: `/opt/discord-bot`
- **Wirtualne środowisko**: Python `venv` (`/opt/discord-bot/venv`)
- **Zainstalowane biblioteki**: `discord.py` (v2.7.1)
- **Plik startowy**: `main.py` (szablon bota z obsługą intents i komendą `!ping`)

## 🚀 Stan na dziś (Następne kroki)
- [x] Konfiguracja sieci, NAT i bezpiecznego routingu na Proxmoxie.
- [x] Stworzenie izolowanego kontenera LXC z Pythonem i Gitem.
- [x] Przygotowanie struktury bota i instalacja `discord.py`.
- [ ] **Jutro**: Konfiguracja uprawnień bota na Discordzie (Discord Developer Portal / Intents), wklejenie tokena i pierwsze uruchomienie testowe na żywo.
