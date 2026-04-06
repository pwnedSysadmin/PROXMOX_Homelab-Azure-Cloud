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
| Windows 10 Pro | 22H2 | Domain client (PC-jgarcia) |
| Ubuntu Server | 24.04 LTS | Web Server (DMZ) + App Server (Corporate) |

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
    │               ├── PC-jgarcia - Windows 10 Pro → 10.20.20.21
    │               └── APPSERVER-CORP (Flask + PostgreSQL) → 10.20.20.20
    │
    └── vmbr30 ── VLAN 30 · Security Lab (10.30.30.0/24)
                    ├── Kali Linux                     ← PENDING
                    └── Vulnerable Machine             ← PENDING
```

### Traffic flow — Web request end to end

```
Client (internet or internal)
    │
    │  GET /api/usuarios
    ▼
Nginx DMZ (10.10.10.10)          ← only exposed point
    │
    │  reverse proxy → 10.20.20.20:5000
    ▼
Flask APPSERVER-CORP (10.20.20.20:5000)    ← never directly exposed
    │
    │  SELECT * FROM usuarios
    ▼
PostgreSQL (localhost:5432)
    │
    ▼
JSON response back to client
```

### Why this architecture?

OPNsense acts as the **inter-VLAN router/firewall**: it is the only VM with interfaces on all network segments. Traffic between VLANs can only flow if OPNsense rules explicitly allow it, which ensures:

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
| Pass *(disabled)* | OPT2 net | OPT1 net | **PENTESTING ONLY** — Kali to Corporate. Enable only during controlled exercises |

### LAN — DMZ (VLAN 10)

| Action | Source | Destination | Port | Description |
|---|---|---|---|---|
| Pass | 10.10.10.10 | 10.20.20.20 | 5000 | Nginx to Flask (reverse proxy) |

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

## Web Stack — Phase 6

### WEBSERVER-DMZ (Nginx)

| Parameter | Value |
|---|---|
| IP | 10.10.10.10 (static) |
| RAM | 1 GB |
| Disk | 20 GB |
| Network | vmbr10 (DMZ) |
| OS | Ubuntu Server 24.04 LTS |
| Role | Nginx reverse proxy — forwards traffic to APPSERVER-CORP |

### APPSERVER-CORP (Flask + PostgreSQL)

| Parameter | Value |
|---|---|
| IP | 10.20.20.20 (static) |
| RAM | 1 GB |
| Disk | 20 GB |
| Network | vmbr20 (Corporate) |
| OS | Ubuntu Server 24.04 LTS |
| Role | Flask API + PostgreSQL database |

### Stack

| Component | Purpose |
|---|---|
| **Nginx** | Reverse proxy — receives external requests, forwards to Flask |
| **Flask** | Python web framework — handles API logic |
| **SQLAlchemy** | ORM — Python interface to PostgreSQL |
| **psycopg2** | PostgreSQL driver for Python |
| **PostgreSQL** | Relational database |

### API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/usuarios` | Returns all users as JSON |
| POST | `/api/usuarios` | Creates a new user |

### PostgreSQL setup

```bash
sudo -i -u postgres psql

CREATE DATABASE appdb;
CREATE USER flaskuser WITH PASSWORD 'flask1234';
GRANT ALL PRIVILEGES ON DATABASE appdb TO flaskuser;
\c appdb
GRANT ALL ON SCHEMA public TO flaskuser;
\q
```

### Flask app — app.py

```python
from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://flaskuser:flask1234@localhost/appdb'
db = SQLAlchemy(app)

class Usuario(db.Model):
    __tablename__ = 'usuarios'
    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), nullable=False)

@app.route('/api/usuarios', methods=['GET'])
def get_usuarios():
    usuarios = Usuario.query.all()
    return jsonify([{'id': u.id, 'nombre': u.nombre, 'email': u.email} for u in usuarios])

@app.route('/api/usuarios', methods=['POST'])
def create_usuario():
    data = request.get_json()
    nuevo = Usuario(nombre=data['nombre'], email=data['email'])
    db.session.add(nuevo)
    db.session.commit()
    return jsonify({'mensaje': 'Usuario creado', 'id': nuevo.id}), 201

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(host='0.0.0.0', port=5000)
```

### Flask systemd service — /etc/systemd/system/flask-app.service

```ini
[Unit]
Description=Flask App
After=network.target postgresql.service

[Service]
User=root
WorkingDirectory=/home/admin-app
Environment="PATH=/home/admin-app/venv/bin"
ExecStart=/home/admin-app/venv/bin/python3 /home/admin-app/app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable flask-app
systemctl start flask-app
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
  - [x] Test user (jgarcia) and group (IT_Admins) created
  - [x] Windows 10 Pro client (PC-jgarcia) joined to domain
- [ ] **Phase 6** — VLAN 10: Web Server in DMZ + App Server in Corporate
  - [x] WEBSERVER-DMZ (Ubuntu + Nginx) installed — 10.10.10.10
  - [x] APPSERVER-CORP (Ubuntu + Flask + PostgreSQL) installed — 10.20.20.20
  - [x] PostgreSQL database and user created
  - [x] Flask API running as systemd service
  - [x] API tested — GET and POST endpoints working
  - [ ] Nginx configured as reverse proxy to Flask
  - [ ] OPNsense rule: DMZ → Corporate port 5000
- [ ] **Phase 7** — VLAN 30: Kali Linux + vulnerable machine
- [ ] **Phase 8** — Linux File Server (VLAN 20)
- [ ] **Phase 9** — Docker / Kubernetes node
- [ ] **Phase 10** — Site-to-site VPN with Azure + Azure AD Connect

---

## VM Resource Planning

| VM | RAM | Disk | VLAN | Status |
|---|---|---|---|---|
| OPNsense | 2 GB | 20 GB | WAN + all | ✅ Running |
| DC01 - Windows Server (AD+DNS) | 4 GB | 60 GB | 20 | ✅ Running |
| PC-jgarcia - Windows 10 Pro | 4 GB | 64 GB | 20 | ✅ Domain joined |
| WEBSERVER-DMZ (Nginx) | 1 GB | 20 GB | 10 | ✅ Running |
| APPSERVER-CORP (Flask + PostgreSQL) | 1 GB | 20 GB | 20 | ✅ Running |
| Kali Linux | 2 GB | 30 GB | 30 | ⏳ Pending |
| Vulnerable Machine | 1 GB | 20 GB | 30 | ⏳ Pending |
| **Total used** | **14 GB / 32 GB** | **234 GB / 1 TB** | — | — |

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

Windows does not include VirtIO drivers by default. Without them, the installer cannot detect the virtual disk or network adapter. During installation click **Load driver** and browse to the VirtIO ISO:

- Disk driver: `viostor → 2k22 → amd64`
- Network driver: `NetKVM → 2k22 → amd64`
- Memory balloon: `Balloon → 2k22 → amd64`

Same process for Windows 10 client, using `w10` folder instead of `2k22`.

VirtIO ISO download:
```
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

### Windows 10 — Bypassing internet requirement during OOBE

Windows 10/11 setup may require an internet connection during OOBE. To bypass:

```cmd
# Open CMD with Shift+F10 during setup, then run:
taskkill /F /IM oobenetworkconnectionflow.exe
```

### Extending C: drive after Proxmox disk resize

After resizing the disk in Proxmox, recovery partitions may block direct extension of C:. Remove them first:

```cmd
diskpart
select disk 0
select partition 4
delete partition override
select partition 5
delete partition override
select partition 3
extend
exit
```

### Ubuntu Server — Python virtual environment setup

Always use a virtual environment to isolate app dependencies:

```bash
python3 -m venv venv
source venv/bin/activate
pip install flask psycopg2-binary flask-sqlalchemy
```

### Testing the API from Windows (PowerShell)

```powershell
# GET — list all users
Invoke-RestMethod -Uri "http://10.20.20.20:5000/api/usuarios" -Method GET

# POST — create a user
Invoke-RestMethod -Uri "http://10.20.20.20:5000/api/usuarios" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"nombre":"Juan Garcia","email":"jgarcia@mandanga.local"}'
```
