# VCF 9.1.1 Selkies Host & Bastion Specification (`SELKIES_HOST.md`)

## 1. Overview & Purpose
This repository (`photon-selkies-host`) contains the infrastructure-as-code, host bootstrap scripts, firewall policies, and custom container image definitions to turn a **VMware Photon OS 5.0** VM into an isolated **Selkies VDI Host and Network Bastion** for the **VMware Cloud Foundation (VCF) 9.1.1** lab.

### Access Model Transformation
* **Before:** All employees required personal Tailscale clients to route directly into the VCF 9.1.1 lab subnet, creating direct L3 exposure of management interfaces, complex subnet routing, and untracked lab logins.
* **Now:** Clients connect **via Tailscale** (`tailscale0`) to reach the VDI Portal and ephemeral **Selkies** desktop containers over HTTPS/WebRTC on this Photon OS bastion. Standard employees still have **zero direct L3/subnet routing** to the VCF 9.1.1 management network—all VCF access remains isolated inside the Selkies containers, while Tailscale ACLs separate standard VDI users (Portal + Selkies ports only) from Lab Administrators (SSH `TCP 22`).

---

## 2. Host Sizing & Network Topology (VCF 9.1.1)

### 2.1 VM Hardware Specifications
* **Guest OS:** VMware Photon OS 5.0 (x86_64)
* **Compute:** 16 vCPU, 32–64 GB RAM (sized for 8–10 concurrent Selkies desktop containers at ~2.5 GB RAM each)
* **Storage:** 150 GB thin-provisioned vSAN disk (mounted to `/var/lib/docker` and `/var/lib/postgresql`)
* **Network Interfaces:**
  * `eth0`: Connected to Lab VCF 9.1.1 Segment (reaches SDDC Manager, vCenter, NSX Manager, VCF Operations, VCF Automation, and VKS Supervisor clusters). Inbound VDI, Portal, and SSH traffic is dropped on `eth0`.
  * `tailscale0`: Virtual interface used for all inbound client and admin access, governed by Tailscale ACLs:
    * **Standard Users (Clients):** Permitted to reach the VDI Portal (`TCP 3000` / `443`) and Selkies WebRTC/WSS streams (`TCP/UDP 10000–10100`) only. No VCF subnet routes are advertised to clients.
    * **Lab Administrators:** Permitted SSH (`TCP 22`) and host-level troubleshooting in addition to VDI access.

### 2.2 Traffic Flow Diagram
```text
[ Standard Employees / Clients ]             [ Lab Administrators Only ]
(Browser via Tailscale Client)                  (Admin Tailscale ACL)
             │                                              │
             │ tailscale0:                                  │ tailscale0:
             │  - TCP 3000/443 (Portal UI)                  │  - SSH (:22)
             │  - TCP/UDP 10000-10100 (Selkies WebRTC/WSS)  │
             ▼                                              ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    VMware Photon OS 5.0 Bastion Host                      │
│                                                                           │
│  [ iptables Firewall ]                                                    │
│   ├── ALLOW TCP 3000/443 & TCP/UDP 10000:10100 ONLY on tailscale0 (VDI)   │
│   ├── ALLOW SSH :22 ONLY on tailscale0 (Admin Management via ACL)         │
│   ├── DROP Inbound VDI / Portal / SSH Traffic on eth0                     │
│   └── DROP Direct User Routing to VCF Subnets (Isolated inside Docker)    │
│                                                                           │
│  [ Docker Engine (/var/run/docker.sock) ]                                 │
│   ├── Container: postgres:16-alpine (:5432 localhost only)                │
│   └── Containers: selkies-vcf9 (:10000-10100 -> internal :8080)           │
└───────────────────────────────────────┬───────────────────────────────────┘
                                        │ Routed via Docker Bridge -> eth0
                                        ▼
                 ┌─────────────────────────────────────────────┐
                 │       VCF 9.1.1 Internal Lab Network        │
                 │  (SDDC Mgr, vCenter, NSX, VKS, VCF Ops/Auto)│
                 └─────────────────────────────────────────────┘
```

---

## 3. Repository File Structure (`photon-selkies-host`)

```text
photon-selkies-host/
├── README.md
├── bootstrap-photon.sh          # Master setup script for fresh Photon OS 5.0 VM
├── scripts/
│   ├── 01-packages.sh           # tdnf package installation (docker, iptables, tailscale, jq)
│   ├── 02-sysctl-tuning.sh      # Kernel shared memory & WebRTC UDP buffer tuning
│   └── 03-firewall-rules.sh     # Photon OS iptables lockdown & port allowances
├── selkies-image/
│   ├── Dockerfile.vcf-jumpbox   # Custom Selkies desktop image for VCF 9.1.1
│   ├── bookmarks.html           # Pre-configured Chromium bookmarks for VCF UIs
│   └── certs/                   # VCF 9.1.1 Root CA certificates
└── docker-compose.infra.yml     # Boots local PostgreSQL 16 & builds Selkies image
```

---

## 4. Photon OS Bootstrap & Tuning Requirements

### 4.1 Package & Service Baseline (`01-packages.sh`)
* Updates Photon OS packages via `tdnf update -y`.
* Installs **Docker Engine** (`docker`, `docker-compose`) directly inside the Photon OS VM to build and host the Selkies desktop containers, along with `iptables`, `curl`, `jq`, `git`, and `tailscale`.
* Enables and starts `docker.service` and `tailscaled.service`.

### 4.2 Kernel & Memory Tuning for Selkies (`02-sysctl-tuning.sh`)
Selkies streams a full Linux desktop and Chromium browser over WebRTC/WebSockets. Running multiple VCF 9.1.1 management UIs simultaneously (vSphere Client, NSX Manager, VCF Operations) is memory-intensive.
* **Host Kernel Limits (`/etc/sysctl.d/99-selkies.conf`):**
  * `vm.max_map_count = 262144` (prevents Chromium/Electron renderer crashes inside containers).
  * `net.core.rmem_max = 26214400` and `net.core.wmem_max = 26214400` (increases UDP receive/send buffers for smooth 60fps WebRTC video transport).
* **Container Runtime Flags (Enforced when spawned on Docker):**
  * `--shm-size=2g`: Mandatory 2 GB `/dev/shm` allocation per container so Chromium tabs do not crash when rendering complex NSX topology maps or vCenter inventories.
  * `--rm`: Automatically destroys the container filesystem upon stop/expiration.
  * `-p <TAILSCALE_IP>:<PORT>:8080` (recommended): Binds published Selkies container ports directly to the `tailscale0` interface IP (or enforces `DOCKER-USER` rules) so Docker's NAT does not expose containers on `eth0`.

### 4.3 Firewall & Access Lockdown (`03-firewall-rules.sh`)
Photon OS defaults to a restrictive firewall. The script writes persistent rules to `/etc/systemd/scripts/ip4save`:
1. **Loopback & Docker Bridge:** Allow all traffic on `lo` and `docker0`.
2. **Admin SSH Access (Tailscale Only):** Allow `TCP 22` **only** via `-i tailscale0` (further restricted to Lab Administrators via Tailscale ACLs).
3. **Web Portal Port (Tailscale Only):** Allow `TCP 3000` (or `443` if fronted by TLS proxy) **only** via `-i tailscale0` for the Next.js VDI Broker dashboard.
4. **Selkies Dynamic Port Pool (Tailscale Only):** Allow `TCP 10000:10100` and `UDP 10000:10100` **only** via `-i tailscale0` for active Selkies WebRTC/WebSocket desktop streams, and enforce `DOCKER-USER` rules so Docker-published ports are only reachable from `tailscale0`.
5. **External Interface Lockdown:** Drop unsolicited inbound VDI, Portal, and SSH traffic on `eth0` so all client and admin access must traverse `tailscale0`.

---

## 5. Custom Lab Selkies Image (`Dockerfile.vcf-jumpbox`)

To achieve **zero onboarding time** for employees, the base `ghcr.io/selkies-project/selkies/desktop:main-ubuntu26.04` image is extended with specific tools so users never have to configure their workspace manually:

1. **Pre-Trusted VCF 9.1.1 Certificates:**
   * Injects the lab Root CA certificates into the container OS trust store (`/usr/local/share/ca-certificates`) and Chromium's NSS database (`~/.pki/nssdb`) so users never encounter SSL/TLS security warnings when opening vCenter, SDDC Manager, or NSX.
2. **Pre-Populated Browser Bookmarks:**
   * Ships a managed Chromium bookmarks bar containing one-click links to VCF SDDC Manager, vCenter Server, NSX Manager, VCF Operations, and VCF Automation portals.
3. **Pre-Installed VCF 9.1.1 CLI Tooling:**
   * Bundles `vcf-cli`, `kubectl`, `govc`, `powercli`, `ssh`, `curl`, `jq`, and `yq` in `/usr/local/bin` for immediate terminal troubleshooting.

---

## 6. Verification Checklist (Phase 1 Completion Criteria)
- [ ] Photon OS 5.0 VM boots on VCF 9.1.1, reaches internal management IPs, and has `docker.service` running.
- [ ] Tailscale is authenticated and ACLs permit VDI Clients to reach `TCP 3000`/`443` and `TCP/UDP 10000–10100`, while restricting `SSH (TCP 22)` to Lab Admins only (with no VCF subnet routes advertised to clients).
- [ ] `bootstrap-photon.sh` completes cleanly and `sysctl` / `iptables` rules persist across reboots.
- [ ] Manual test container (`docker run --rm -d --shm-size=2g -p <TAILSCALE_IP>:10001:8080 selkies-vcf9:latest`) streams a desktop in the browser over Tailscale (`tailscale0`) and resolves internal VCF 9.1.1 UIs, while remaining unreachable directly over `eth0`.