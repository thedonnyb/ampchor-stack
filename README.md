# ampchor-stack

A self-hosted media automation stack running as **rootless Podman containers** managed through **systemd Quadlet** on **RHEL 10**. Built by hand — every container, network, and storage decision made and documented from first principles — as an exercise in Linux systems administration, container orchestration, and network configuration.

> This project is for educational and self-hosting purposes, demonstrating infrastructure and sysadmin skills. Users are responsible for complying with applicable laws.

---

## What this demonstrates

- **Podman / Quadlet** — containers defined as declarative `.container` / `.network` units, generated into systemd services (no Docker, no Compose)
- **Rootless containers** — understanding user vs. system systemd, mailbox locations, and rootless networking
- **systemd service management** — unit files, `daemon-reload`, lingering for rootless persistence across reboots, auto-start targets
- **SELinux** — per-container private labels (`:Z`) vs. shared labels (`:z`), relabeling, troubleshooting `container_file_t` context mismatches
- **firewalld** — per-service port management and zone configuration
- **Container networking** — a shared user-defined network so containers resolve one another by name
- **Storage architecture** — a single shared data root enabling instant **hardlinks** (atomic moves, no duplicate disk usage) between downloads and media libraries
- **NFS / Samba** — exporting the media library over the network and mounting it on Linux and macOS clients
- **Secrets management** — credentials kept in a gitignored `.env`, never committed; `.env.example` provided as a template

---

## Architecture

```
                         ┌─────────────────────────────┐
                         │        RHEL 10 server        │
                         │                              │
  Indexers ◄────────────►│  Prowlarr ──┬──► Sonarr      │   rootless
                         │             └──► Radarr      │   (user systemd)
                         │                   │          │
                         │                 Bazarr       │
                         │                   │          │
                         │        ┌──────────┴────────┐ │
                         │        │   shared /data     │ │  single-root
                         │        │  tv / movies /     │ │  (hardlinks)
                         │        │  downloads         │ │
                         │        └────────────────────┘ │
                         │                              │
                         │   FlareSolverr (Cloudflare)  │  rootless
                         └──────────────┬───────────────┘
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

---

## Repository layout

```
ampchor-stack/
├── quadlets/        # Quadlet unit files (.container, .network)
├── docs/            # Per-service build logs (how each was built + debugged)
├── .env.example     # Secrets template (real .env is gitignored)
├── .gitignore
└── README.md
```

The `docs/` folder contains a build log for each service — not just the final config, but the reasoning behind each setting and the real problems hit and solved along the way.

---

## Notable problems solved

- **SELinux label collision** — a container mounting a shared parent directory relabeled it with its own private SELinux category, breaking database access for the other containers that shared it. Fixed by mounting each app's own config subdirectory with a private `:Z` label.
- **Hardlink-aware storage** — rejected the common split-mount layout (`/tv` + `/downloads`) that silently breaks hardlinks, in favor of a single `/data` root so every container sees one filesystem.
- **Rootless service persistence** — rootless user services stop when the user logs out; enabling lingering (`loginctl enable-linger`) keeps the stack running across reboots without a logged-in session.

---

## Tech

RHEL 10 · Podman · Quadlet · systemd · SELinux · firewalld · NFS · Samba
