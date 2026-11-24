# Ampchor Stack

A self-hosted media automation stack running as **rootless and rootful Podman containers** managed through **systemd Quadlet** on **RHEL 10**, with all download traffic isolated behind a **VPN gateway container**. Built by hand — every container, network, storage, and security decision made and documented from first principles — as an exercise in Linux systems administration, container orchestration, and network security.

> This project is for educational and self-hosting purposes, demonstrating infrastructure and sysadmin skills. Users are responsible for complying with applicable laws.

---

## What this demonstrates

- **Podman / Quadlet** — containers defined as declarative `.container` / `.network` units, generated into systemd services (no Docker, no Compose)
- **Rootless *and* rootful containers** — understanding when each is required, and how they differ (user vs. system systemd, mailbox locations, networking, device access)
- **systemd service management** — unit files, `daemon-reload`, lingering for rootless persistence across reboots, service dependencies, auto-start targets
- **VPN gateway isolation** — a dedicated WireGuard VPN container; the download client shares that container's network namespace (`Network=container:gluetun`) so it has **no network path except through the VPN**, with a kill-switch that drops its connectivity if the tunnel fails
- **SELinux** — per-container private labels (`:Z`) vs. shared labels (`:z`), relabeling, troubleshooting `container_file_t` context mismatches
- **firewalld** — per-service port management, masquerade/NAT, zone configuration
- **Container networking** — a shared user-defined network so containers resolve one another by name; and a container-namespace-shared network for VPN routing
- **Storage architecture** — a single shared data root enabling instant **hardlinks** (atomic moves, no duplicate disk usage) between downloads and media libraries
- **NFS / Samba** — exporting the media library over the network and mounting it on Linux and macOS clients
- **Automation** — scheduled cron-based cleanup for disk management
- **Secrets management** — credentials kept in a gitignored `.env`, never committed; `.env.example` provided as a template

---

## Architecture

```
                         ┌───────────────────────────────┐
                         │         RHEL 10 server         │
                         │                                │
  Indexers ◄────────────►│  Prowlarr ──┬──► Sonarr        │  rootless
                         │             └──► Radarr        │  (user systemd)
                         │                   │            │
                         │                 Bazarr         │
                         │   FlareSolverr (Cloudflare)    │
                         │                   │            │
                         │        ┌──────────┴─────────┐  │
                         │        │    shared /data     │  │  single-root
                         │        │  tv / movies /      │  │  (hardlinks)
                         │        │  downloads          │  │
                         │        └──────────┬─────────┘  │
                         │                   │            │
                         │   ┌───────────────┴─────────┐  │  rootful
     VPN provider ◄──────┼───┤  VPN gateway (WireGuard)│  │  (system systemd)
                         │   │     └── download client  │  │  kill-switch:
                         │   │   (Network=container:vpn)│  │  no VPN = no net
                         │   └─────────────────────────┘  │
                         └───────────────┬────────────────┘
                                         │ Samba / NFS
                                         ▼
                               Ampchor (desktop player,
                               Mac / Linux, reads /data)
```

---

## Stack

| Service | Role | Mode | Network |
|---|---|---|---|
| Prowlarr | Indexer manager | rootless | `media.network` |
| Sonarr | TV management | rootless | `media.network` |
| Radarr | Movie management | rootless | `media.network` |
| Bazarr | Subtitle management | rootless | `media.network` |
| FlareSolverr | Cloudflare solver | rootless | `media.network` |
| VPN gateway | WireGuard tunnel | rootful | host + published ports |
| Download client | Download management | rootful | `container:<vpn>` (VPN-only) |

---

## Repository layout

```
ampchor-stack/
├── quadlets/        # Quadlet unit files (.container, .network)
├── docs/            # Per-service build logs (how each was built + debugged)
├── scripts/         # Maintenance scripts (scheduled cleanup, etc.)
├── .env.example     # Secrets template (real .env is gitignored)
├── .gitignore
└── README.md
```

The `docs/` folder contains a build log for each service — not just the final config, but the reasoning behind each setting and the real problems hit and solved along the way.

---

## Notable problems solved

- **VPN tunnel passing no traffic** — the WireGuard tunnel completed its handshake but moved zero packets. Worked through a chain of causes: a missing kernel firewall-compatibility module on RHEL 10, rootless device-access limits on the tunnel device (resolved by running the VPN container rootful), and finally a dual-stack (IPv4+IPv6) key on an IPv4-only-capable host — regenerating the key IPv4-only restored traffic.

- **Download client losing connectivity after tunnel resets** — the client would stall indefinitely after the VPN silently renegotiated, because it held stale connections. Fixed by binding the client to the VPN tunnel interface (`tun0`) so it rebinds when the tunnel changes instead of hanging.

- **SELinux label collision** — a container mounting a shared parent directory relabeled it with its own private SELinux category, breaking database access for the other containers that shared it. Fixed by mounting each app's own config subdirectory with a private `:Z` label.

- **Hardlink-aware storage** — rejected the common split-mount layout (`/tv` + `/downloads`) that silently breaks hardlinks, in favor of a single `/data` root so every container sees one filesystem.

- **Rootless service persistence** — rootless user services stop when the user logs out; enabling lingering (`loginctl enable-linger`) keeps the stack running across reboots without a logged-in session.

- **Rootless → rootful migration** — moving the VPN container between modes left a stale unit behind, producing a port-conflict restart loop; resolved by removing the duplicate unit.

---

## Roadmap

Planned evolution to demonstrate cloud-engineering progression. The current build proves the fundamentals (containers, networking, security) without a cloud abstracting them away; each phase lifts that into cloud-native tooling.

- **Phase 1 — Infrastructure as Code (Terraform):** provision the environment on Azure (VM, network, NSGs, storage) with remote state, replacing manual setup with reproducible code.
- **Phase 2 — Reverse proxy & TLS (Caddy):** consolidate services behind a single entry point with automatic HTTPS, closing individual app ports.
- **Phase 3 — CI/CD (GitHub Actions):** `terraform plan` on pull request, `apply` on merge — GitOps deployment.
- **Phase 4 — Kubernetes (AKS):** translate the Quadlet units into Kubernetes manifests (Deployments, Services, PVCs, Secrets).
- **Phase 5 — Observability (Prometheus / Grafana):** metrics and dashboards for the running services.

---

## Tech

RHEL 10 · Podman · Quadlet · systemd · SELinux · firewalld · WireGuard · NFS · Samba · Bash/cron
