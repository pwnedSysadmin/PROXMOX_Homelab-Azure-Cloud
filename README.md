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
| Ubuntu Server | 24.04 LTS | Web Server (DMZ) + App/File Server (Corporate) + Docker host |
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
    │               ├── APPSERVER-CORP (Flask + PostgreSQL + Samba) → 10.20.20.20
    │               └── DOCKER-CORP (Docker host) → 10.20.20.30
    │
    └── vmbr30 ── VLAN 30 · Security Lab (10.30.30.0/24)
                    └── KALI - Kali Linux → 10.30.30.10
```

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

### WAN — Port Forwarding (NAT + Firewall rules)

| Port | Redirects to | Description |
|---|---|---|
| 443 | OPNsense UI | Management access from 192.168.1.0/24 |
| 80 | 10.10.10.10:80 | HTTP to WEBSERVER-DMZ *(external access pending fix)* |
| 2210 | 10.10.10.10:22 | SSH to WEBSERVER-DMZ |
| 2220 | 10.20.20.20:22 | SSH to APPSERVER-CORP |

---

## Active Directory — DC01

### Domain info

| Parameter | Value |
|---|---|
| Domain | `mandanga.local` |
| Domain Controller | DC01 |
| DC IP | 10.20.20.10 (static) |
| DNS | DC01 self (127.0.0.1) |

### OU Structure

```
mandanga.local
├── Workstations      ← domain-joined client machines
├── Servers           ← member servers
├── Users_LAB         ← lab user accounts
└── Groups_LAB        ← lab security groups
```

### Users & Groups

| Username | OU | Groups |
|---|---|---|
| jgarcia (Juan Garcia) | Users_LAB | IT_Admins |

### GPOs

| GPO | Linked to | Description |
|---|---|---|
| Administracion Remota - Workstations | Workstations | Enables WinRM firewall rules on domain workstations |
| Mapeo Unidad Red | Workstations + Users_LAB | Auto-maps `Z:` drive to `\\10.20.20.20\compartido` on login |

> **Note:** Drive mapping GPOs are linked to both the computer OU (Workstations) and the user OU (Users_LAB) because User Configuration GPOs apply based on where the **user** object is in AD, not the computer.

---

## Client — PC-GARCIA

| Parameter | Value |
|---|---|
| OS | Windows 11 Pro |
| Hostname | PC-GARCIA |
| IP | 10.20.20.21 (static) |
| DNS | 10.20.20.10 (DC01) |
| Domain | mandanga.local ✅ |

---

## Web Stack

### WEBSERVER-DMZ (Nginx reverse proxy)

| Parameter | Value |
|---|---|
| IP | 10.10.10.10 |
| OS | Ubuntu Server 24.04 LTS |
| Role | Nginx reverse proxy → forwards to Flask on 10.20.20.20:5000 |
| SSH | `ssh admin@192.168.1.38 -p 2210` |

### APPSERVER-CORP (Flask + PostgreSQL + Samba)

| Parameter | Value |
|---|---|
| IP | 10.20.20.20 |
| OS | Ubuntu Server 24.04 LTS |
| Role | Flask API + PostgreSQL + Samba File Server |
| SSH | `ssh admin-app@192.168.1.38 -p 2220` |
| Domain | mandanga.local ✅ |

#### Samba configuration

```ini
[global]
    workgroup = MANDANGA
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

> **Note:** In Spanish AD installations group names are localized. Use `@"usuarios del dominio"` — not `@"Domain Users"`. The domain prefix is not needed since it's defined in `[global]`.

---

## Docker — DOCKER-CORP

### VM specs

| Parameter | Value |
|---|---|
| IP | 10.20.20.30 (static) |
| RAM | 2 GB |
| Disk | 20 GB |
| OS | Ubuntu Server 24.04 LTS |
| Network | vmbr20 (Corporate) |

### Docker concepts

| Concept | Description |
|---|---|
| **Image** | Immutable template — like an ISO. Built from a Dockerfile |
| **Container** | Running instance of an image. Ephemeral — deleted when stopped and removed |
| **Volume** | Persistent storage mounted from the host into a container. Survives container deletion |
| **Dockerfile** | Recipe that defines how to build a custom image |
| **Docker Compose** | Tool to define and manage multi-container stacks via a YAML file |

### Key Docker commands

```bash
# Images
docker images                          # list downloaded images
docker build -t my-image .             # build image from Dockerfile in current dir
docker pull nginx                      # download image from Docker Hub

# Containers
docker run -d -p 8080:80 nginx         # run container in background, map ports
docker run -d -p 8080:80 \
  -v ~/html:/usr/share/nginx/html \
  --name my-nginx nginx                # run with volume and custom name
docker ps                              # list running containers
docker ps -a                           # list all containers including stopped
docker stop my-nginx                   # stop container gracefully
docker rm my-nginx                     # remove container
docker logs my-nginx                   # view container logs
docker exec -it my-nginx bash          # open shell inside container

# Docker Compose
docker compose up -d                   # start all services in background
docker compose down                    # stop and remove all services
docker compose logs                    # view logs of all services
```

### Volumes — why they matter

Containers are **ephemeral** — any data written inside a container is lost when the container is removed. Volumes solve this by mounting a host directory inside the container:

```bash
docker run -d -p 8080:80 \
  -v ~/web-data:/usr/share/nginx/html \
  --name my-nginx nginx
```

```
HOST                         CONTAINER
~/web-data          ←→      /usr/share/nginx/html
```

Changes to `~/web-data` on the host are immediately visible inside the container and vice versa. Data persists even if the container is deleted.

### Project 1 — Nginx with HTML volume

```bash
mkdir ~/web-data
echo "<h1>Hello from the host</h1>" > ~/web-data/index.html
chmod 755 ~/web-data
chmod 644 ~/web-data/index.html
docker run -d -p 8080:80 -v ~/web-data:/usr/share/nginx/html --name web nginx
```

Access at: `http://10.20.20.30:8080`

### Project 2 — Nginx + Portainer (Docker Compose)

**`~/mi-stack/docker-compose.yml`:**

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html

  portainer:
    image: portainer/portainer-ce
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Portainer mounts `/var/run/docker.sock` — the Docker daemon socket — which gives it full control over the Docker host from inside the container. This is a common pattern but grants elevated privileges.

Access Portainer at: `http://10.20.20.30:9000`

### Project 3 — Custom Flask API + PostgreSQL (Docker Compose)

**`~/flask-app/Dockerfile`:**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

The Dockerfile defines 5 build steps:
1. Start from the official `python:3.11-slim` base image
2. Set `/app` as the working directory inside the container
3. Copy `requirements.txt` into the container
4. Install Python dependencies with pip
5. Copy the application code and define the startup command

**`~/flask-app/requirements.txt`:**

```
flask
psycopg2-binary
flask-sqlalchemy
```

**`~/flask-app/docker-compose.yml`:**

```yaml
services:
  flask:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://flaskuser:flask1234@db/appdb
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=flaskuser
      - POSTGRES_PASSWORD=flask1234
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Key concepts in this compose file:
- `build: .` — builds a custom image from the local Dockerfile instead of pulling from Docker Hub
- `environment` — passes environment variables into the container (database credentials, connection strings)
- `depends_on: db` — Flask waits for PostgreSQL to start before launching
- `postgres_data` volume — PostgreSQL data persists on the host even if the container is deleted

Access Flask API at: `http://10.20.20.30:5000`

---

## Security Lab — Kali Linux

| Parameter | Value |
|---|---|
| IP | 10.30.30.10 (static) |
| OS | Kali Linux 2025.1 |
| Network | vmbr30 (Security Lab) |

### Attacks demonstrated

| Technique | Tool | Target | Result |
|---|---|---|---|
| NTDS dump | NetExec (nxc) | DC01 10.20.20.10 | All domain hashes extracted |
| SMB enumeration | nxc smb | DC01 | Domain info enumerated |

```bash
# Dump all NTDS hashes
nxc smb 10.20.20.10 -u 'Administrador' -p 'Password' --ntds
```

> **Note:** LLMNR/NBT-NS attacks (Responder) require being on the same network segment. Kali is in VLAN 30, AD is in VLAN 20 — broadcast traffic does not cross VLANs.

---

## Build Phases

- [x] **Phase 1** — Proxmox VE 9.1.1 installation
- [x] **Phase 2** — Repository configuration (no-subscription)
- [x] **Phase 3** — Virtual network bridges (vmbr0–vmbr30)
- [x] **Phase 4** — OPNsense: interfaces, DHCP, firewall, NAT
- [x] **Phase 5** — Active Directory (DC01 + PC-GARCIA + GPOs)
- [x] **Phase 6** — Web stack: Nginx DMZ + Flask + PostgreSQL
- [x] **Phase 7** — Kali Linux security lab
- [x] **Phase 8** — Samba File Server integrated with AD
- [x] **Phase 9** — Docker on DOCKER-CORP
  - [x] Docker installed and verified
  - [x] Nginx container with persistent volume
  - [x] Docker Compose: Nginx + Portainer stack
  - [x] Custom Dockerfile: Flask API image
  - [x] Docker Compose: Flask + PostgreSQL stack
- [ ] **Phase 10** — Kubernetes (K3s)
- [ ] **Phase 11** — Azure VPN + Azure AD Connect

---

## VM Resource Planning

| VM | RAM | Disk | VLAN | Status |
|---|---|---|---|---|
| OPNsense | 2 GB | 20 GB | WAN + all | ✅ Running |
| DC01 - Windows Server 2022 | 4 GB | 60 GB | 20 | ✅ Running |
| PC-GARCIA - Windows 11 Pro | 4 GB | 100 GB | 20 | ✅ Domain joined |
| WEBSERVER-DMZ (Nginx) | 1 GB | 20 GB | 10 | ✅ Running |
| APPSERVER-CORP (Flask + PostgreSQL + Samba) | 1 GB | 20 GB | 20 | ✅ Running |
| DOCKER-CORP | 2 GB | 20 GB | 20 | ✅ Running |
| KALI | 2 GB | 30 GB | 30 | ✅ Running |
| **Total used** | **16 GB / 32 GB** | **270 GB / 1 TB** | — | — |

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
2. Add firewall rule: Pass TCP from `192.168.1.0/24` to WAN address port 443

### Windows — VirtIO drivers

```
Disk:    viostor → 2k22 → amd64  (or w11 for Windows 11)
Network: NetKVM  → 2k22 → amd64
Balloon: Balloon → 2k22 → amd64
```

### Windows 11 — Bypassing internet requirement during OOBE

```cmd
taskkill /F /IM oobenetworkconnectionflow.exe
```

### Docker — Installation on Ubuntu

```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

### Samba — Joining Ubuntu to AD domain

```bash
sudo apt install samba winbind libnss-winbind libpam-winbind krb5-user -y
# Set DNS to DC IP in /etc/resolv.conf before joining
sudo net ads join -U Administrador
sudo systemctl enable winbind smbd
sudo systemctl restart winbind smbd
```

---

*Documentation in progress — updated as the lab evolves.*
