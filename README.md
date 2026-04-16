# 🖥️ Proxmox Homelab

Documentation of my virtualization lab built on Proxmox VE, designed to practice system administration, networking, and security.

---

## Hardware

| Component | Details |
|---|---|
| Motherboard | Dell 0FDY5C |
| CPU | Intel Core i7-6700 3.40 GHz (4 cores / 8 threads, turbo up to 4.00 GHz) |
| RAM | 32 GB DDR4 (max. 64 GB) |
| Storage | 1 TB SATA 2.5" SSD |
| PSU | 180 W |
| GPU | Intel HD Graphics 530 (integrated) |

---

## Base Software

| Software | Version | Notes |
|---|---|---|
| Proxmox VE | 9.1.1 | Main hypervisor |
| Debian | Trixie (13) | Proxmox underlying OS |
| OPNsense | 26.1.2 | Virtual firewall/router |
| Windows Server 2022 | Standard Evaluation | Domain Controller (DC01) |
| Windows 11 Pro | 24H2 | Domain client (PC-GARCIA) |
| Ubuntu Server | 24.04 LTS | Web Server (DMZ) + App/File Server (Corporate) |
| Kali Linux | 2025.1 | Security Lab (VLAN 30) |

---

## Lab Architecture

```
Internet
    │
    ▼
[OPNsense VM] ── Central Firewall/Router (192.168.1.38 WAN)
    │
    ├── vmbr10 ── VLAN 10 · DMZ          (10.10.10.0/24)
    │               └── WEBSERVER-DMZ (Nginx reverse proxy) → 10.10.10.10
    │
    ├── vmbr20 ── VLAN 20 · Corporate    (10.20.20.0/24)
    │               ├── DC01 - Windows Server 2022 (AD + DNS) → 10.20.20.10
    │               ├── PC-GARCIA - Windows 11 Pro → 10.20.20.21
    │               └── APPSERVER-CORP (Flask + PostgreSQL + Samba) → 10.20.20.20
    │
    └── vmbr30 ── VLAN 30 · Security Lab (10.30.30.0/24)
                    └── KALI - Kali Linux → 10.30.30.10
```

### Traffic flow — Web request end to end

```
Client (internet or internal)
    │  GET /api/usuarios
    ▼
Nginx DMZ (10.10.10.10)          ← only exposed point
    │  reverse proxy → 10.20.20.20:5000
    ▼
Flask APPSERVER-CORP (10.20.20.20:5000)
    │  SELECT * FROM usuarios
    ▼
PostgreSQL (localhost:5432)
```

### Why this architecture?

- Kali (VLAN 30) cannot directly attack the Active Directory (VLAN 20) unless the pentesting rule is explicitly enabled
- The Web Server (VLAN 10) is exposed but isolated from internal networks
- The App Server (VLAN 20) is never directly reachable from outside — only Nginx can talk to it
- All internet-bound traffic passes through and is filtered by OPNsense

---

## Virtual Networking in Proxmox

| Bridge | Type | Purpose |
|---|---|---|
| `vmbr0` | WAN | Connects OPNsense to the physical network/router. IP: `192.168.1.37/24` |
| `vmbr10` | Internal | DMZ — exists only inside Proxmox |
| `vmbr20` | Internal | Corporate network — exists only inside Proxmox |
| `vmbr30` | Internal | Security Lab — exists only inside Proxmox |

---

## OPNsense Network Interfaces

| Interface | Maps to | IP | Role |
|---|---|---|---|
| vtnet0 | vmbr0 | 192.168.1.38/24 (DHCP) | WAN |
| vtnet1 | vmbr10 | 10.10.10.1/24 | LAN (DMZ) |
| vtnet2 | vmbr20 | 10.20.20.1/24 | OPT1 (Corporate) |
| vtnet3 | vmbr30 | 10.30.30.1/24 | OPT2 (Security Lab) |

### DHCP Ranges

| Network | Range |
|---|---|
| DMZ (10.10.10.0/24) | 10.10.10.100 – 10.10.10.200 |
| Corporate (10.20.20.0/24) | 10.20.20.100 – 10.20.20.200 |
| Security Lab (10.30.30.0/24) | 10.30.30.100 – 10.30.30.200 |

---

## Firewall Rules

### OPT1 — Corporate (VLAN 20)

| Action | Source | Destination | Description |
|---|---|---|---|
| Pass | OPT1 net | LAN net | Corporate to DMZ |
| Pass | OPT1 net | any | Corporate to internet |

### OPT2 — Security Lab (VLAN 30)

| Action | Source | Destination | Description |
|---|---|---|---|
| Pass | OPT2 net | any | Security Lab to internet (includes DMZ) |
| Pass *(disabled)* | OPT2 net | OPT1 net | **PENTESTING ONLY** — enable only during controlled exercises |

### LAN — DMZ (VLAN 10)

| Action | Source | Destination | Port | Description |
|---|---|---|---|---|
| Pass | 10.10.10.10 | 10.20.20.20 | 5000 | Nginx to Flask (reverse proxy) |

### WAN

| Action | Source | Destination | Port | Description |
|---|---|---|---|---|
| Pass | 192.168.1.0/24 | WAN address | 443 | Allow UI access from management network |
| Pass | any | WAN address | 80 | HTTP to WEBSERVER-DMZ |
| Pass | 192.168.1.0/24 | WAN address | 2210 | SSH to WEBSERVER-DMZ |
| Pass | 192.168.1.0/24 | WAN address | 2220 | SSH to APPSERVER-CORP |

### NAT — Destination NAT (Port Forwarding)

| Interface | Destination port | Redirect to | Description |
|---|---|---|---|
| WAN | 2210 | 10.10.10.10:22 | SSH to WEBSERVER-DMZ |
| WAN | 2220 | 10.20.20.20:22 | SSH to APPSERVER-CORP |
| WAN | 80 | 10.10.10.10:80 | HTTP to WEBSERVER-DMZ |

> **Known issue:** OPNsense NAT port forward for port 80 is configured but external HTTP access is not working yet. Internal reverse proxy (Nginx → Flask) is fully functional.

---

## Active Directory — DC01

### Domain info

| Parameter | Value |
|---|---|
| Domain | `mandanga.local` |
| Forest Functional Level | Windows Server 2022 |
| Domain Controller | DC01 |
| DC IP | 10.20.20.10 (static) |
| DNS | DC01 (127.0.0.1 self) |

### DC01 VM specs

| Parameter | Value |
|---|---|
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 60 GB |
| Network | vmbr20 (VLAN 20 Corporate) |
| OS | Windows Server 2022 Standard Evaluation (Desktop Experience) |

### OU Structure

```
mandanga.local
├── Workstations      ← domain-joined client machines
├── Servers           ← member servers
├── Users_LAB         ← lab user accounts
└── Groups_LAB        ← lab security groups
```

### Users

| Username | Full name | OU | Groups |
|---|---|---|---|
| jgarcia | Juan Garcia | Users_LAB | IT_Admins |

### Groups

| Group | Type | Scope | Members |
|---|---|---|---|
| IT_Admins | Security | Global | jgarcia |

### GPOs

| GPO | Linked to | Description |
|---|---|---|
| Administracion Remota - Workstations | Workstations | Enables WinRM remote management rules on domain workstations |
| Mapeo Unidad Red | Workstations | Maps network drive Z: to \\10.20.20.20\compartido on login ← PENDING |

---

## Client — PC-GARCIA

| Parameter | Value |
|---|---|
| OS | Windows 11 Pro |
| Hostname | PC-GARCIA |
| IP | 10.20.20.21 (static) |
| DNS | 10.20.20.10 (DC01) |
| Domain | mandanga.local ✅ |
| RAM | 4 GB |
| Disk | 100 GB |
| Network | vmbr20 (VLAN 20 Corporate) |

---

## Web Stack — WEBSERVER-DMZ + APPSERVER-CORP

### WEBSERVER-DMZ (Nginx)

| Parameter | Value |
|---|---|
| IP | 10.10.10.10 (static) |
| RAM | 1 GB |
| Disk | 20 GB |
| Network | vmbr10 (DMZ) |
| OS | Ubuntu Server 24.04 LTS |
| Role | Nginx reverse proxy |
| SSH access | `ssh admin@192.168.1.38 -p 2210` |

### APPSERVER-CORP (Flask + PostgreSQL + Samba)

| Parameter | Value |
|---|---|
| IP | 10.20.20.20 (static) |
| RAM | 1 GB |
| Disk | 20 GB |
| Network | vmbr20 (Corporate) |
| OS | Ubuntu Server 24.04 LTS |
| Role | Flask API + PostgreSQL + Samba File Server |
| SSH access | `ssh admin-app@192.168.1.38 -p 2220` |
| Domain | mandanga.local ✅ (joined via net ads join) |

### API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | /api/usuarios | Returns all users as JSON |
| POST | /api/usuarios | Creates a new user |

### Samba File Server

| Parameter | Value |
|---|---|
| Share | `\\10.20.20.20\compartido` |
| Path | `/srv/samba/compartido` |
| Access | Domain users (`usuarios del dominio`) |

#### smb.conf

```ini
[global]
    workgroup = MANDANGA
    server string = File Server
    security = ADS
    realm = MANDANGA.LOCAL
    winbind use default domain = yes
    winbind enum users = yes
    winbind enum groups = yes
    idmap config * : backend = tdb
    idmap config * : range = 3000-7999
    idmap config MANDANGA : backend = rid
    idmap config MANDANGA : range = 10000-19999

[compartido]
    path = /srv/samba/compartido
    browseable = yes
    writeable = yes
    guest ok = no
    valid users = @"usuarios del dominio"
```

> **Note:** Group name must match the AD language. In Spanish AD installations, use `@"usuarios del dominio"` — not `@"Domain Users"`. The domain prefix (`MANDANGA+`) is not needed since it's already defined in `[global]`.

---

## Security Lab — Kali Linux

| Parameter | Value |
|---|---|
| IP | 10.30.30.10 (static) |
| RAM | 2 GB |
| Disk | 30 GB |
| Network | vmbr30 (Security Lab) |
| OS | Kali Linux 2025.1 |

### Attacks demonstrated

| Technique | Tool | Target | Result |
|---|---|---|---|
| NTDS dump | NetExec (nxc) | DC01 (10.20.20.10) | All domain hashes extracted |
| SMB enumeration | nxc smb | DC01 | Domain info, shares enumerated |

### Key findings

- `krbtgt` hash extracted → Golden Ticket attack possible
- All domain user hashes (NT) extracted via `nxc smb --ntds`
- WinRM accessible on DC01 port 5985

> **Note:** LLMNR/NBT-NS attacks (Responder) require being on the same network segment as the target — they do not cross VLANs. Kali is in VLAN 30, Corporate network is VLAN 20.

---

## Build Phases

- [x] **Phase 1** — Proxmox VE 9.1.1 installation
- [x] **Phase 2** — Repository configuration (no-subscription)
- [x] **Phase 3** — Virtual network bridges (vmbr0, vmbr10, vmbr20, vmbr30)
- [x] **Phase 4** — OPNsense installation and configuration
- [x] **Phase 5** — VLAN 20: Windows Server + Active Directory
  - [x] DC01 promoted to Domain Controller (mandanga.local)
  - [x] OU structure, users and groups created
  - [x] Windows 11 Pro client (PC-GARCIA) joined to domain
  - [x] GPO: Remote management enabled on workstations
- [x] **Phase 6** — Web stack: Nginx (DMZ) + Flask + PostgreSQL (Corporate)
  - [x] Reverse proxy Nginx → Flask working internally
  - [ ] External HTTP access via OPNsense port forward ← pending fix
- [x] **Phase 7** — VLAN 30: Kali Linux
  - [x] Kali installed and operational
  - [x] NTDS dump via NetExec demonstrated
- [x] **Phase 8** — Samba File Server on APPSERVER-CORP
  - [x] APPSERVER-CORP joined to mandanga.local domain
  - [x] Samba share accessible by domain users
  - [ ] GPO: Auto-map Z: drive on login ← pending
- [ ] **Phase 9** — Docker / Kubernetes node
- [ ] **Phase 10** — Site-to-site VPN with Azure + Azure AD Connect

---

## VM Resource Planning

| VM | RAM | Disk | VLAN | Status |
|---|---|---|---|---|
| OPNsense | 2 GB | 20 GB | WAN + all | ✅ Running |
| DC01 - Windows Server (AD+DNS) | 4 GB | 60 GB | 20 | ✅ Running |
| PC-GARCIA - Windows 11 Pro | 4 GB | 100 GB | 20 | ✅ Domain joined |
| WEBSERVER-DMZ (Nginx) | 1 GB | 20 GB | 10 | ✅ Running |
| APPSERVER-CORP (Flask + PostgreSQL + Samba) | 1 GB | 20 GB | 20 | ✅ Running |
| KALI | 2 GB | 30 GB | 30 | ✅ Running |
| **Total used** | **14 GB / 32 GB** | **250 GB / 1 TB** | — | — |

---

## Installation Notes

### Proxmox — Post-install repository fix

```bash
echo "" > /etc/apt/sources.list.d/pve-enterprise.sources
echo "" > /etc/apt/sources.list.d/ceph.sources
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt dist-upgrade -y
```

### OPNsense — WAN management access

1. Disable **Block private networks** and **Block bogon networks** on WAN interface
2. Create firewall rule: Pass TCP from `192.168.1.0/24` to WAN address port 443

### Windows Server — VirtIO drivers

```
Disk:    viostor → 2k22 → amd64
Network: NetKVM  → 2k22 → amd64
Balloon: Balloon → 2k22 → amd64
```

### Windows 11 — Bypassing internet requirement during OOBE

```cmd
taskkill /F /IM oobenetworkconnectionflow.exe
```

### Samba — joining Linux to AD domain

```bash
# Install required packages
sudo apt install samba winbind libnss-winbind libpam-winbind krb5-user -y

# Join the domain
sudo net ads join -U Administrador

# Enable and start services
sudo systemctl enable winbind smbd
sudo systemctl restart winbind smbd

# Verify domain join
sudo net ads info
sudo wbinfo -u
sudo wbinfo -g
```

> **Important:** Set DNS to DC IP (`10.20.20.10`) in `/etc/resolv.conf` before joining the domain. systemd-resolved (`127.0.0.53`) will prevent the join.

### Testing Samba access

```bash
# From APPSERVER-CORP
smbclient //localhost/compartido -U jgarcia%password

# From Windows client
net use \\10.20.20.20\compartido /user:MANDANGA\jgarcia password
```

### NTDS dump with NetExec

```bash
nxc smb 10.20.20.10 -u 'Administrador' -p 'Password' --ntds
```

---

*Documentation in progress — updated as the lab evolves.*
