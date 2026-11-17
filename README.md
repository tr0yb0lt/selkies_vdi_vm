# VCF 9.1.1 Selkies Host & Bastion Specification (`SELKIES_HOST.md`)

## 1. Overview & Purpose
This repository (`photon-selkies-host`) contains the infrastructure-as-code, host bootstrap scripts, firewall policies, and custom container image definitions to turn a **VMware Photon OS 5.0** VM into an isolated **Selkies VDI Host and Network Bastion** for the **VMware Cloud Foundation (VCF) 9.1.1** lab.

### Access Model Transformation
* **Before:** All employees required personal Tailscale clients to route directly into the VCF 9.1.1 lab subnet, creating onboarding friction, direct L3 exposure of management interfaces, and untracked lab logins.
* **Now:** Tailscale is **restricted strictly to Lab Administrators** for backend host maintenance. Standard employees have zero direct VPN/Tailscale access to the VCF network; instead, they connect over HTTPS/WebRTC to ephemeral **Selkies** desktop containers hosted on this Photon OS bastion, reducing onboarding time to zero.

---

## 2. Host Sizing & Network Topology (VCF 9.1.1)

### 2.1 VM Hardware Specifications
* **Guest OS:** VMware Photon OS 5.0 (x86_64)
* **Compute:** 16 vCPU, 32–64 GB RAM (sized for 8–10 concurrent Selkies desktop containers at ~2.5 GB RAM each)
* **Storage:** 150 GB thin-provisioned vSAN disk (mounted to `/var/lib/docker` and `/var/lib/postgresql`)
* **Network Interfaces:**
  * `eth0`: Connected to Lab VCF 9.1.1 Segment (reaches SDDC Manager, vCenter, NSX Manager, VCF Operations, VCF Automation, and VKS Supervisor clusters).
  * `tailscale0`: Virtual interface restricted via Tailscale ACLs to **Lab Administrators only** for SSH (`TCP 22`) and host-level troubleshooting.

### 2.2 Traffic Flow Diagram
```text
[ Standard Employees ]                       [ Lab Administrators Only ]
(Browser Only - No Tailscale)                    (Admin Tailscale ACL)
             │                                              │
             │ TCP 3000 (Portal UI)                         │ tailscale0 (SSH :22)
             │ TCP/UDP 10000-10100 (Selkies WebRTC/WSS)     │
             ▼                                              ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    VMware Photon OS 5.0 Bastion Host                      │
│                                                                           │
│  [ iptables Firewall ]                                                    │
│   ├── ALLOW TCP 3000 & TCP/UDP 10000:10100 (User VDI & Portal Traffic)    │
│   ├── ALLOW SSH :22 ONLY on tailscale0 (Admin Management)                 │
│   └── DROP Direct User Routing to VCF Subnets (Isolated inside Docker)    │
│                                                                           │
│  [ Docker Engine (/var/run/docker.sock) ]                                 │
│   ├── Container: postgres:16-alpine (:5432 localhost only)                │
│   └── Containers: selkies-vcf9 (:10000-10100 -> internal :8080)       │
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
* Installs `docker`, `docker-compose`, `iptables`, `curl`, `jq`, `git`, and `tailscale`.
* Enables and starts `docker.service` and `tailscaled.service`.

### 4.2 Kernel & Memory Tuning for Selkies (`02-sysctl-tuning.sh`)
Selkies streams a full Linux desktop and Chromium browser over WebRTC/WebSockets. Running multiple VCF 9.1.1 management UIs simultaneously (vSphere Client, NSX Manager, VCF Operations) is memory-intensive.
* **Host Kernel Limits (`/etc/sysctl.d/99-selkies.conf`):**
  * `vm.max_map_count = 262144` (prevents Chromium/Electron renderer crashes inside containers).
  * `net.core.rmem_max = 26214400` and `net.core.wmem_max = 26214400` (increases UDP receive/send buffers for smooth 60fps WebRTC video transport).
* **Container Runtime Flags (Enforced when spawned):**
  * `--shm-size=2g`: Mandatory 2 GB `/dev/shm` allocation per container so Chromium tabs do not crash when rendering complex NSX topology maps or vCenter inventories.
  * `--rm`: Automatically destroys the container filesystem upon stop/expiration.

### 4.3 Firewall & Access Lockdown (`03-firewall-rules.sh`)
Photon OS defaults to a restrictive firewall. The script writes persistent rules to `/etc/systemd/scripts/ip4save`:
1. **Loopback & Docker Bridge:** Allow all traffic on `lo` and `docker0`.
2. **Admin SSH Access:** Allow `TCP 22` **only** via `-i tailscale0`. Drop SSH attempts from standard LAN interfaces if required by security policy.
3. **Web Portal Port:** Allow `TCP 3000` (or `443` if fronted by TLS proxy) for the Next.js VDI Broker dashboard.
4. **Selkies Dynamic Port Pool:** Allow `TCP 10000:10100` and `UDP 10000:10100` for active Selkies WebRTC/WebSocket desktop streams.

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
- [ ] Photon OS 5.0 VM boots on VCF 9.1.1 and reaches internal management IPs.
- [ ] Tailscale is authenticated and restricted via ACLs to Lab Admins only.
- [ ] `bootstrap-photon.sh` completes cleanly and `sysctl` / `iptables` rules persist across reboots.
- [ ] Manual test container (`docker run --rm -d --shm-size=2g -p 10001:8080 selkies-vcf9:latest`) streams a desktop in the browser and resolves internal VCF 9.1.1 UIs without client-side Tailscale installed.