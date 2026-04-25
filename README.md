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
| Ubuntu Server | 24.04 LTS | Web Server (DMZ) + App/File Server + Docker/K3s host |
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
    │               └── DOCKER-CORP (Docker + K3s) → 10.20.20.30
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

## OPNsense

### Network Interfaces

| Interface | Maps to | IP | Role |
|---|---|---|---|
| vtnet0 | vmbr0 | 192.168.1.38/24 (DHCP) | WAN |
| vtnet1 | vmbr10 | 10.10.10.1/24 | LAN (DMZ) |
| vtnet2 | vmbr20 | 10.20.20.1/24 | OPT1 (Corporate) |
| vtnet3 | vmbr30 | 10.30.30.1/24 | OPT2 (Security Lab) |

### Firewall Rules Summary

| Interface | Action | Source | Destination | Notes |
|---|---|---|---|---|
| OPT1 | Pass | OPT1 net | any | Corporate internet access |
| OPT2 | Pass | OPT2 net | any | Security Lab internet access |
| OPT2 | Pass *(disabled)* | OPT2 net | OPT1 net | Pentesting only |
| LAN | Pass | 10.10.10.10 | 10.20.20.20:5000 | Nginx → Flask |
| WAN | Pass | 192.168.1.0/24 | WAN:443 | OPNsense UI access |

### NAT Port Forwarding

| Port | Redirects to | Description |
|---|---|---|
| 2210 | 10.10.10.10:22 | SSH to WEBSERVER-DMZ |
| 2220 | 10.20.20.20:22 | SSH to APPSERVER-CORP |
| 80 | 10.10.10.10:80 | HTTP to WEBSERVER-DMZ *(external access pending fix)* |

---

## Active Directory — DC01

| Parameter | Value |
|---|---|
| Domain | `mandanga.local` |
| DC IP | 10.20.20.10 (static) |
| OS | Windows Server 2022 Standard Evaluation |

### OU Structure

```
mandanga.local
├── Workstations
├── Servers
├── Users_LAB
└── Groups_LAB
```

### Users & Groups

| Username | OU | Groups |
|---|---|---|
| jgarcia (Juan Garcia) | Users_LAB | IT_Admins |

### GPOs

| GPO | Linked to | Description |
|---|---|---|
| Administracion Remota - Workstations | Workstations | Enables WinRM firewall rules |
| Mapeo Unidad Red | Workstations + Users_LAB | Auto-maps `Z:` → `\\10.20.20.20\compartido` on login |

> **Note:** Drive mapping GPOs must be linked to both the computer OU and the user OU. User Configuration GPOs apply based on where the **user** object is in AD.

---

## Web Stack

### WEBSERVER-DMZ (Nginx)

| Parameter | Value |
|---|---|
| IP | 10.10.10.10 |
| Role | Nginx reverse proxy → 10.20.20.20:5000 |
| SSH | `ssh admin@192.168.1.38 -p 2210` |

### APPSERVER-CORP (Flask + PostgreSQL + Samba)

| Parameter | Value |
|---|---|
| IP | 10.20.20.20 |
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

> In Spanish AD installations group names are localized. Use `@"usuarios del dominio"` not `@"Domain Users"`.

---

## Docker & Kubernetes — DOCKER-CORP

### VM specs

| Parameter | Value |
|---|---|
| IP | 10.20.20.30 (static) |
| RAM | 2 GB |
| Disk | 20 GB |
| OS | Ubuntu Server 24.04 LTS |

---

### Docker

Docker solves the **"it works on my machine"** problem. It packages an application with all its dependencies into an image that runs identically anywhere.

| Concept | Description |
|---|---|
| **Image** | Immutable template built from a Dockerfile |
| **Container** | Running instance of an image. Ephemeral — data is lost when removed |
| **Volume** | Host directory mounted inside a container. Data persists after container deletion |
| **Dockerfile** | Recipe defining how to build a custom image |
| **Docker Compose** | Manages multi-container stacks via a single YAML file |

#### Key Docker commands

```bash
docker images                          # list images
docker build -t my-image .             # build from Dockerfile
docker run -d -p 8080:80 nginx         # run container in background
docker run -d -p 8080:80 \
  -v ~/html:/usr/share/nginx/html \
  --name my-nginx nginx                # run with volume
docker ps / docker ps -a               # list running / all containers
docker stop my-nginx && docker rm my-nginx
docker logs my-nginx
docker exec -it my-nginx bash          # shell inside container
docker compose up -d                   # start compose stack
docker compose down                    # stop compose stack
```

#### Volumes — persistence across container restarts

```bash
# Changes to ~/web-data on the host are immediately visible inside the container
docker run -d -p 8080:80 \
  -v ~/web-data:/usr/share/nginx/html \
  --name web nginx
```

```
HOST ~/web-data  ←→  CONTAINER /usr/share/nginx/html
```

#### Project 1 — Nginx with HTML volume

```bash
mkdir ~/web-data
echo "<h1>Hello from the host</h1>" > ~/web-data/index.html
chmod 755 ~/web-data && chmod 644 ~/web-data/index.html
docker run -d -p 8080:80 -v ~/web-data:/usr/share/nginx/html --name web nginx
# http://10.20.20.30:8080
```

#### Project 2 — Nginx + Portainer (Docker Compose)

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
      - /var/run/docker.sock:/var/run/docker.sock  # gives Portainer full Docker control
      - portainer_data:/data

volumes:
  portainer_data:
```

Portainer UI at: `http://10.20.20.30:9000`

#### Project 3 — Custom Flask API + PostgreSQL (Docker Compose)

**`~/flask-app/Dockerfile`:**

```dockerfile
FROM python:3.11-slim    # base image from Docker Hub
WORKDIR /app             # working directory inside container
COPY requirements.txt .  # copy dependency list
RUN pip install -r requirements.txt  # install dependencies
COPY app.py .            # copy application code
CMD ["python", "app.py"] # startup command
```

**`~/flask-app/docker-compose.yml`:**

```yaml
services:
  flask:
    build: .             # builds image from local Dockerfile
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://flaskuser:flask1234@db/appdb
    depends_on:
      - db               # waits for PostgreSQL before starting

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=flaskuser
      - POSTGRES_PASSWORD=flask1234
    volumes:
      - postgres_data:/var/lib/postgresql/data  # data persists on host

volumes:
  postgres_data:
```

Flask API at: `http://10.20.20.30:5000`

---

### Kubernetes (K3s)

Kubernetes solves the problem of **operating containers at scale in production**. While Docker runs containers and Compose manages multi-container stacks, Kubernetes adds:

- **Self-healing** — automatically restarts failed containers
- **Scaling** — runs multiple replicas, adds more under load
- **Zero-downtime updates** — rolling updates without service interruption
- **Service discovery** — stable endpoints regardless of Pod IP changes

```
Developer local  →  Docker
Small stack      →  Docker Compose
Production/scale →  Kubernetes
```

K3s is a lightweight Kubernetes distribution — full Kubernetes functionality with lower resource requirements. Ideal for labs and edge deployments.

#### Core concepts

| Concept | Description |
|---|---|
| **Pod** | Smallest unit — one or more containers sharing IP and storage |
| **Deployment** | Declares desired state: "always run N replicas of this Pod" |
| **Service** | Stable IP/port endpoint in front of a group of Pods |
| **Node** | A server running Pods |
| **Cluster** | Set of Nodes managed by Kubernetes |

#### Why Services are necessary

Pod IPs are ephemeral — they change every time a Pod is recreated. A Service provides a stable endpoint:

```
Without Service:  Client → 10.42.0.11 (Pod IP) → changes on restart ❌
With Service:     Client → Service (stable IP:port) → Pod (any IP)   ✅
```

#### Installation

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

> K3s uses **containerd** as its container runtime, independent of Docker. Images must be imported into containerd if internet access is restricted:
> ```bash
> sudo docker save nginx | sudo k3s ctr images import -
> ```

#### Key kubectl commands

```bash
kubectl get nodes                      # list cluster nodes
kubectl get pods                       # list pods in default namespace
kubectl get pods -n kube-system        # list system pods
kubectl get services                   # list services
kubectl apply -f manifest.yaml         # create/update resources from file
kubectl delete pod my-pod              # delete a pod
kubectl describe pod my-pod            # detailed pod info and events
kubectl logs my-pod                    # view pod logs
kubectl exec -it my-pod -- bash        # shell inside pod
```

#### Pod manifest — `nginx-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mi-nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: Never    # use local image, do not pull from registry
    ports:
    - containerPort: 80
```

#### Deployment manifest — `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2                 # always maintain 2 running pods
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

#### Service manifest — `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx               # targets all pods with label app=nginx
  type: NodePort             # exposes service on node port
  ports:
  - port: 80                 # service port
    targetPort: 80           # container port
    nodePort: 30080          # external port on node (accessible from outside)
```

Access at: `http://10.20.20.30:30080`

#### Self-healing demonstration

```bash
# Delete one of the 2 replica pods
kubectl delete pod nginx-deployment-xxxxx

# Kubernetes immediately creates a replacement to maintain 2 replicas
kubectl get pods
# New pod appears within seconds — service never interrupted
```

---

## Security Lab — Kali Linux

| Parameter | Value |
|---|---|
| IP | 10.30.30.10 |
| OS | Kali Linux 2025.1 |

### Attacks demonstrated

| Technique | Tool | Target | Result |
|---|---|---|---|
| NTDS dump | NetExec (nxc) | DC01 10.20.20.10 | All domain hashes extracted |

```bash
nxc smb 10.20.20.10 -u 'Administrador' -p 'Password' --ntds
```

> LLMNR/NBT-NS attacks (Responder) require same network segment. Kali (VLAN 30) cannot reach AD (VLAN 20) broadcast traffic.

---

## Build Phases

- [x] **Phase 1** — Proxmox VE 9.1.1 installation
- [x] **Phase 2** — Repository configuration
- [x] **Phase 3** — Virtual network bridges
- [x] **Phase 4** — OPNsense: interfaces, DHCP, firewall, NAT
- [x] **Phase 5** — Active Directory (DC01 + PC-GARCIA + GPOs)
- [x] **Phase 6** — Web stack: Nginx DMZ + Flask + PostgreSQL
- [x] **Phase 7** — Kali Linux security lab
- [x] **Phase 8** — Samba File Server integrated with AD
- [x] **Phase 9** — Docker on DOCKER-CORP
  - [x] Nginx with persistent volume
  - [x] Docker Compose: Nginx + Portainer
  - [x] Custom Dockerfile: Flask API image
  - [x] Docker Compose: Flask + PostgreSQL
- [x] **Phase 10** — Kubernetes K3s on DOCKER-CORP
  - [x] K3s installed (single-node cluster)
  - [x] Pod deployed and accessible
  - [x] Service (NodePort) exposing Pod externally
  - [x] Deployment with 2 replicas — self-healing demonstrated
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
| DOCKER-CORP (Docker + K3s) | 2 GB | 20 GB | 20 | ✅ Running |
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
Disk:    viostor → 2k22 → amd64
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

### K3s — Installation

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

> K3s uses containerd, not Docker. Import existing Docker images with:
> ```bash
> sudo docker save <image> | sudo k3s ctr images import -
> ```
> Add `imagePullPolicy: Never` to manifests when using local images.

### Samba — Joining Ubuntu to AD domain

```bash
sudo apt install samba winbind libnss-winbind libpam-winbind krb5-user -y
# Set /etc/resolv.conf nameserver to DC IP (10.20.20.10) before joining
sudo net ads join -U Administrador
sudo systemctl enable winbind smbd
sudo systemctl restart winbind smbd
sudo net ads info       # verify domain join
sudo wbinfo -u          # list domain users
sudo wbinfo -g          # list domain groups
```

---

*Documentation in progress — updated as the lab evolves.*
