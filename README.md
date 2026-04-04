
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

---

## Lab Architecture

```
Internet
    │
    ▼
[OPNsense VM] ── Central Firewall/Router (192.168.1.38 WAN)
    │
    ├── vmbr10 ── VLAN 10 · DMZ          (10.10.10.0/24)
    │               └── Web Server (HTTP/HTTPS)
    │
    ├── vmbr20 ── VLAN 20 · Corporate    (10.20.20.0/24)
    │               ├── DC01 - Windows Server 2022 (AD + DNS) → 10.20.20.10
    │               ├── Linux Server (File Server + DNS)
    │               └── Windows Client (domain-joined)
    │
    └── vmbr30 ── VLAN 30 · Security Lab (10.30.30.0/24)
                    ├── Kali Linux
                    └── Vulnerable Machine
```

### Why this architecture?

OPNsense acts as the **inter-VLAN router/firewall**: it is the only VM with interfaces on all network segments. Traffic between VLANs can only flow if OPNsense rules explicitly allow it, which ensures:

- Kali (VLAN 30) cannot directly attack the Active Directory (VLAN 20) unless the pentesting rule is explicitly enabled
- The Web Server (VLAN 10) is exposed but isolated from internal networks
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
| Pass *(disabled)* | OPT2 net | OPT1 net | **PENTESTING ONLY** — Kali to Corporate. Enable only during controlled exercises |

### WAN

| Action | Source | Destination | Port | Description |
|---|---|---|---|---|
| Pass | 192.168.1.0/24 | WAN address | 443 | Allow UI access from management network |

> **Note:** Block private networks and Block bogon networks are disabled on the WAN interface because the management network (192.168.1.0/24) shares the same segment as the WAN in this lab setup.

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

---

## Build Phases

- [x] **Phase 1** — Proxmox VE 9.1.1 installation
- [x] **Phase 2** — Repository configuration (no-subscription)
- [x] **Phase 3** — Virtual network bridges created (vmbr0, vmbr10, vmbr20, vmbr30)
- [x] **Phase 4** — OPNsense installation and configuration
  - [x] Interface assignment (WAN/LAN/OPT1/OPT2)
  - [x] IP addressing per VLAN
  - [x] DHCP server per VLAN
  - [x] Firewall rules (inter-VLAN + internet access)
- [x] **Phase 5** — VLAN 20: Windows Server + Active Directory
  - [x] Windows Server 2022 installation with VirtIO drivers
  - [x] Static IP configuration (10.20.20.10)
  - [x] AD DS + DNS roles installed
  - [x] Promoted to Domain Controller (mandanga.local)
  - [x] OU structure created
  - [x] Test user and group created
- [ ] **Phase 5b** — Domain-joined Windows 11 client
- [ ] **Phase 6** — VLAN 10: Web Server in DMZ
- [ ] **Phase 7** — VLAN 30: Kali Linux + vulnerable machine
- [ ] **Phase 8** — Site-to-site VPN with Azure (optional)

---

---

## Installation Notes

### Proxmox — Post-install repository fix

Proxmox VE 9 ships with enterprise repositories enabled by default (requires a paid subscription). For a no-cost lab environment, these must be disabled:

```bash
# Disable enterprise repos (PVE 9 uses .sources format in addition to .list)
echo "" > /etc/apt/sources.list.d/pve-enterprise.sources
echo "" > /etc/apt/sources.list.d/ceph.sources

# Add free no-subscription repo (trixie = Debian 13 = PVE 9)
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# Update system
apt update && apt dist-upgrade -y
```

> **Note:** In PVE 9, repos use the `.sources` format in addition to the classic `.list` format. Simply commenting out lines inside `.sources` files causes a `Malformed entry` error — emptying the files entirely is the correct approach.

### OPNsense — WAN management access

By default OPNsense blocks access to the web UI from the WAN interface. In this lab the management network shares the same segment as the WAN (`192.168.1.0/24`), so two changes are required:

1. **Disable "Block private networks"** and **"Block bogon networks"** on the WAN interface (`Interfaces → WAN`)
2. **Create a firewall rule** on WAN allowing TCP from `192.168.1.0/24` to WAN address port 443

### Windows Server — VirtIO drivers

Windows does not include VirtIO drivers by default. Without them, the installer cannot detect the virtual disk. During installation click **Load driver** and browse to the VirtIO ISO:

- Disk driver: `viostor → 2k22 → amd64`
- Network driver: `NetKVM → 2k22 → amd64`
- Memory balloon: `Balloon → 2k22 → amd64`

VirtIO ISO download:
```
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

---

*Documentation in progress — updated as the lab evolves.*
