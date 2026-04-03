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
| OPNsense | Latest | Virtual firewall/router (pfSense alternative) |

---

## Lab Architecture

```
Internet
    │
    ▼
[OPNsense VM] ── Central Firewall/Router
    │
    ├── vmbr10 ── VLAN 10 · DMZ
    │               └── Web Server (HTTP/HTTPS)
    │
    ├── vmbr20 ── VLAN 20 · Corporate Network
    │               ├── Windows Server (Active Directory + DNS)
    │               ├── Linux Server (File Server + DNS)
    │               └── Windows Client
    │
    └── vmbr30 ── VLAN 30 · Security Lab
                    ├── Kali Linux
                    └── Vulnerable Machine
```

### Why this architecture?

OPNsense acts as the **inter-VLAN router/firewall**: it is the only VM with interfaces on all network segments. Traffic between VLANs can only flow if OPNsense rules explicitly allow it, which ensures:

- Kali (VLAN 30) cannot directly attack the Active Directory (VLAN 20)
- The Web Server (VLAN 10) is exposed to the internet but isolated from internal networks
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

## Build Phases

- [x] **Phase 1** — Proxmox VE 9.1.1 installation
- [x] **Phase 2** — Repository configuration (no-subscription)
- [x] **Phase 3** — Virtual network bridges created (vmbr0, vmbr10, vmbr20, vmbr30)
- [ ] **Phase 4** — OPNsense installation and configuration
- [ ] **Phase 5** — VLAN 20: Windows Server + Active Directory + domain-joined client
- [ ] **Phase 6** — VLAN 10: Web Server in DMZ
- [ ] **Phase 7** — VLAN 30: Kali Linux + vulnerable machine
- [ ] **Phase 8** — Site-to-site VPN with Azure (optional)

---

## VM Resource Planning

| VM | RAM | Disk | VLAN |
|---|---|---|---|
| OPNsense | 2 GB | 20 GB | WAN + all |
| Windows Server (AD+DNS) | 4 GB | 60 GB | 20 |
| Linux Server (File+DNS) | 2 GB | 40 GB | 20 |
| Windows Client | 2 GB | 40 GB | 20 |
| Web Server (Linux) | 1 GB | 20 GB | 10 |
| Kali Linux | 2 GB | 30 GB | 30 |
| **Total** | **13 GB / 32 GB** | **210 GB / 1 TB** | — |

---

## Installation Notes

### Post-install repository fix

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

---

*Documentation in progress — updated as the lab evolves.*
