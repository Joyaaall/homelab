# Home Server Infrastructure

**Last audited:** October 7, 2026

This repository documents how my Raspberry Pi home server is organized, how its services work together, and where application data is stored.

## Overview

The server is a **Raspberry Pi 5 with 8 GB of RAM** running **Ubuntu 24.04 LTS**. Docker and Docker Compose are used to run most applications as isolated containers.

The infrastructure is divided into six main layers:

1. **Hardware** — Raspberry Pi, system SSD, and bulk-storage HDD.
2. **Operating system** — Ubuntu Server and host-level services.
3. **Access and ingress** — Tailscale and SSH for administration, plus a managed tunnel for selected applications.
4. **Application platform** — Docker and separate Compose stacks.
5. **Storage** — local SSD/HDD storage and cloud-backed media mounts.
6. **Monitoring and recovery** — logs, health checks, temperature monitoring, and storage recovery automation.

```mermaid
flowchart TD
    User[User devices]
    VPN[Tailscale private network]
    Host[Raspberry Pi 5<br/>Ubuntu Server]
    Docker[Docker and Docker Compose]

    Dashboard[Glance dashboard]
    Logs[Dozzle log viewer]
    Media[Media stack]
    Photos[Immich photo stack]
    Automation[n8n automation stack]
    Sync[Syncthing]
    Vault[Vaultwarden]

    Databases[PostgreSQL and Valkey]
    SSD[System SSD]
    HDD[Bulk-storage HDD]
    Cloud[Cloud-backed media mounts]
    Monitor[Health and temperature monitoring]
    Watchdog[Storage watchdog]

    User --> VPN
    VPN --> Host
    Host --> Docker
    Host --> Monitor

    Docker --> Dashboard
    Docker --> Logs
    Docker --> Media
    Docker --> Photos
    Docker --> Automation
    Docker --> Sync
    Docker --> Vault

    Photos --> Databases
    Automation --> Databases

    Docker --> SSD
    Media --> HDD
    Media --> Cloud
    Photos --> HDD
    Sync --> HDD
    Host --> Watchdog
    Watchdog --> Cloud
```

## Hardware Layout

| Component | Role |
| --- | --- |
| Raspberry Pi 5 | Runs Ubuntu, Docker, networking, and monitoring services |
| 4-core ARM64 processor | Handles application and container workloads |
| 8 GB RAM | Shared between the operating system and containers |
| Approximately 256 GB SSD | Stores Ubuntu, Docker, application configuration, and databases |
| Approximately 500 GB HDD | Stores larger media, photo, and synchronized files |

The SSD is the primary boot and application drive. During the latest audit, the SSD was approximately **30% used** and the HDD was approximately **35% used**. The HDD provides additional capacity for data that does not need SSD-level performance.

## Host Operating System

The base system runs:

| Component | Version or purpose |
| --- | --- |
| Ubuntu Server | Ubuntu 24.04.4 LTS |
| Linux kernel | Raspberry Pi Linux 6.8.0 build |
| CPU architecture | ARM64 / `aarch64` |
| Docker Engine | Version 29.8.0 |
| Docker Compose | Version 5.5.1 |
| Tailscale | Private networking between approved devices |
| SSH | Command-line administration through the private network |

Ubuntu manages hardware, storage, networking, Docker, and system services. Applications are kept inside containers instead of being installed directly on the host whenever possible.

## How Applications Are Organized

Applications are separated into Docker Compose stacks. Each stack contains services that belong together and normally has its own Docker network and persistent data directories.

This structure provides several benefits:

- One application can be restarted without restarting the entire server.
- Dependencies from different applications do not conflict.
- Application configuration can be backed up separately.
- A broken update is easier to isolate and roll back.
- Services within a stack can communicate using Docker networking.

The server had **20 running containers** during the latest audit.

## Dashboard and Logs

### Glance

Glance provides a simple homepage for viewing links and useful server information from one place. It acts as the entry point to the other self-hosted applications.

```text
User device
    |
Tailscale
    |
Glance dashboard
    |
Other self-hosted services
```

### Dozzle

Dozzle provides a lightweight browser-based view of Docker logs. It supports day-to-day troubleshooting without requiring a full metrics and log-aggregation platform.

## Media Stack

The media services are connected as a workflow rather than operating independently.

| Service | Role |
| --- | --- |
| Jellyfin | Plays and organizes the final media library |
| Jellyseerr | Provides a user-friendly interface for requesting content |
| Sonarr | Manages television-series downloads and organization |
| Radarr | Manages movie downloads and organization |
| Bazarr | Finds and manages subtitles |
| Prowlarr | Manages indexer connections for Sonarr and Radarr |
| qBittorrent | Handles downloads requested by the management services |
| FlareSolverr | Supports compatible indexer access when required |
| Plezy Relay | Provides an additional supporting relay service |

### Media request flow

```mermaid
flowchart LR
    User[User request]
    Request[Jellyseerr]
    Manager[Sonarr or Radarr]
    Indexer[Prowlarr]
    Download[qBittorrent]
    Storage[Media storage]
    Player[Jellyfin]

    User --> Request
    Request --> Manager
    Manager --> Indexer
    Manager --> Download
    Download --> Storage
    Storage --> Player
```

A request begins in Jellyseerr. Sonarr or Radarr processes it, Prowlarr helps locate a suitable source, and qBittorrent downloads the file. The completed file is organized on bulk storage and then made available through Jellyfin. Bazarr handles subtitles separately.

## Photo Management Stack

Immich provides private photo and video management. Its components are separated by responsibility:

| Service | Role |
| --- | --- |
| Immich Server | Main web interface and application API |
| Immich Machine Learning | Facial recognition and intelligent search processing |
| Immich PostgreSQL | Stores application metadata |
| Valkey | Provides caching and background-job coordination |

```text
User uploads photo
        |
        v
Immich Server
   |          |
   v          v
PostgreSQL   Valkey
   |
   v
Photo storage
   |
   v
Machine-learning processing
```

The database stores metadata, while the original photo and video files are kept on persistent storage. Machine-learning work runs in its own container so it can be managed independently from the main server.

## Automation Stack

n8n runs automated workflows and integrations.

| Service | Role |
| --- | --- |
| n8n | Workflow editor, scheduler, and execution service |
| n8n Task Runner | Executes supported workflow tasks separately |
| PostgreSQL | Stores workflows, execution history, and application data |

```text
Trigger or schedule
       |
       v
      n8n
   |       |
   v       v
Task Runner  PostgreSQL
```

Keeping the task runner and database separate makes the automation platform easier to maintain and allows each component to have a focused role.

## File Synchronization

Syncthing synchronizes selected files between approved devices. It provides direct device-to-device synchronization without requiring a third-party storage provider for every transfer.

Its configuration remains on persistent storage so device relationships and synchronization settings survive container replacement.

## Password Management

Vaultwarden provides a lightweight, self-hosted password-vault service compatible with Bitwarden clients.

The application runs in its own container and stores its persistent vault data separately from the container image. The container can therefore be upgraded or recreated without deleting its stored data.

## Docker Networking

Docker networks separate application groups from one another.

For example:

- Media-management services share a network so they can communicate internally.
- Immich services share a dedicated network with their database and cache.
- n8n connects to its task runner and database through application networks.
- Unrelated applications do not need to share the same internal network.

Containers refer to one another using service names instead of fixed container addresses. Docker handles the internal service discovery.

## Storage Structure

Persistent data is divided by purpose:

### System SSD

The SSD contains:

- Ubuntu Server
- Docker and container images
- Application configuration
- Databases
- Frequently accessed application data
- Logs and system packages

### Bulk-storage HDD

The HDD contains larger files such as:

- Media libraries
- Photo and video files
- Downloaded content
- Synchronized files

### Cloud-backed media mounts

Read-through media mounts extend the local library with cloud-backed storage. Host-level mount services make the remote libraries available to the media stack, while a storage watchdog checks that the mounts remain usable and refreshes dependent services after recovery.

Container images are replaceable, but configuration, databases, and user files are not. Those persistent data categories require backups.

## Access and Ingress

Tailscale connects approved devices to the server through a private network. SSH and administrative web interfaces can be accessed through this private connection.

The general access path is:

```text
Approved device
      |
      v
Tailscale private network
      |
      v
Raspberry Pi server
      |
      v
Docker application
```

This approach avoids making administrative access directly dependent on the local network and reduces the need to expose management services publicly. A managed tunnel client also runs on the host for selected application traffic; administrative access remains separated from that application-ingress path.

## Monitoring

The server currently uses several lightweight monitoring methods:

- Docker reports whether containers are running.
- Supported containers provide health-check results.
- Dozzle provides a lightweight view of container logs.
- A user service monitors SSD temperature.
- A host timer checks cloud-backed storage mounts and coordinates recovery.
- Storage usage is checked during infrastructure audits.
- System and container updates are reviewed as part of maintenance.

During the latest audit:

| Check | Result |
| --- | --- |
| Docker | Active |
| Tailscale | Active |
| SSH | Active |
| Managed application tunnel | Active |
| SSD temperature monitor | Active |
| Storage watchdog | Active; latest run successful |
| Containers | 20 running |
| Configured container health checks | 7 healthy, 0 unhealthy |
| System SSD usage | 30% |
| Bulk-storage HDD usage | 35% |
| Host temperature | Approximately 54°C |
| Ubuntu package updates | Pending review |

## Maintenance Workflow

Regular maintenance includes:

1. Check that important containers are running and healthy.
2. Review available Ubuntu security updates.
3. Review container-image updates and release notes.
4. Check storage usage and disk health.
5. Confirm that the cloud-backed mounts and storage watchdog are healthy.
6. Confirm that the SSD temperature monitor is active.
7. Back up application configuration and databases.
8. Test that important backups can actually be restored.
9. Remove unused images and volumes only after confirming they are unnecessary.

## Recovery Model

This is a single-server setup rather than a high-availability cluster. If the Raspberry Pi, system SSD, or operating system fails, multiple applications may become unavailable together.

Recovery therefore depends on preserving:

- Docker Compose definitions
- Environment configuration and secrets
- Application configuration directories
- PostgreSQL database backups
- Vaultwarden data
- Important photo, media, and synchronized files
- Storage-mount definitions and recovery automation

With those items available, the operating system and Docker can be reinstalled, the application stacks recreated, and persistent data restored.

## Design Principles

The infrastructure follows a few simple principles:

- **Containerize applications** to keep dependencies isolated.
- **Separate services by stack** so each application can be managed independently.
- **Use private networking** for administration.
- **Separate bulk files from system data** to use storage efficiently.
- **Keep application data persistent** even when containers are replaced.
- **Monitor basic health and temperature** without requiring a heavy monitoring platform.
- **Document how components connect** so the system can be maintained and rebuilt later.
