# Runbook 004 — Docker Engine and Compose on Herd OS

**Project:** Personal AI OS / Herd OS  
**Date:** 2026-11-01  
**Status:** Planned; preflight verified, installation not yet performed.

## Objective

Install Docker Engine and the Docker Compose plugin on the Ubuntu Lightsail server, verify the runtime, and prepare for a private OpenClaw deployment.

## Verified starting state

| Property | Observed |
| --- | --- |
| Instance | `herd-os-gateway` |
| OS | Ubuntu 24.04 LTS (Noble) |
| CPU architecture | `x86_64` (amd64) |
| Docker | Not installed |
| RAM / swap | ~2 GB RAM, 2 GB swap |
| Remote access | SSH over Tailscale (`100.108.177.8`) |
| Lightsail public inbound ports | None |

## Architectural decisions

### Portability: Lightsail today, Hetzner tomorrow

**We want Herd OS to be portable between AWS Lightsail and Hetzner.** Packaging the application in containers and defining services with a version-controlled Compose file makes the application runtime easier to reproduce on another Ubuntu VPS. This is not complete portability by itself: persistent volumes, secrets, DNS, networking, backups, CPU architecture, and host provisioning must also be migrated. Avoid AWS-specific runtime dependencies unless justified.

### Security: published container ports

**Docker-published ports can bypass some host firewall rules (notably UFW's usual INPUT-chain rules).** Do not assume that enabling UFW makes `docker run -p 3000:3000` private. For OpenClaw, prefer no host port publication when feasible; otherwise bind deliberately to `127.0.0.1` and access via an authenticated tunnel or carefully configured private proxy. Do not bind the gateway to `0.0.0.0` by default. Lightsail's network firewall is an additional control but is not a substitute for secure container networking.

## Architecture

```text
Mac (Tailscale) 100.85.130.7
       |
       | encrypted tailnet / private SSH
       v
AWS Lightsail: herd-os-gateway (Ubuntu 24.04, x86_64)
+-------------------------------------------------------+
| Tailscale: 100.108.177.8                               |
| SSH: OpenSSH (reachable via tailnet)                   |
| Docker Engine (TO INSTALL)                            |
|   +------------------------------------------------+  |
|   | Docker Compose (TO CONFIGURE)                  |  |
|   | OpenClaw container (FUTURE)                    |  |
|   | Persistent application data (FUTURE)           |  |
|   +------------------------------------------------+  |
| Public inbound ports: none                            |
+-------------------------------------------------------+
             |
             +--> Outbound connections for updates/APIs
```

## Installation (official Docker APT repository)

Execute **on the Ubuntu server**, not on the Mac. This follows Docker's Ubuntu installation method; review the official instructions before execution.

### 1. Remove conflicting distro packages (if present)

```bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc
```

> On this fresh server, Docker is not installed. The removal step is included for reproducibility, not because conflicting packages were observed.

### 2. Add Docker's repository signing key

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 3. Configure the repository

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

### 4. Install Engine, CLI, Compose and supporting plugins

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 5. Verify the daemon and CLI

```bash
sudo systemctl is-active docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Expected: daemon `active`, Docker client/server version output, Compose version, and a successful `hello-world` message. **Do not mark complete until verified.**

### 6. Check service persistence

```bash
sudo systemctl is-enabled docker
sudo systemctl is-enabled containerd
```

On standard Ubuntu Docker packages, services normally start on boot. Verify rather than assume.

## Security considerations before deploying OpenClaw

- Avoid adding `ubuntu` to the `docker` group casually: access to the Docker daemon effectively grants root-level control of the host. Use `sudo docker` initially.
- Avoid publishing container ports to all interfaces. Explicitly review every `ports:` entry in Compose.
- Store secrets outside Git; restrict permissions; plan rotation and backups.
- Review Docker daemon logs and disk usage (`docker system df`) as part of operational monitoring.
- Do not deploy OpenClaw until its binding, authentication, persistence, and upgrade strategy are defined.
- 2 GB RAM is a starting point for a lightweight API-backed gateway, not local model inference; monitor usage and upgrade if necessary.

## Next verification and documentation

1. Complete Docker installation and capture actual versions/output.
2. Define `compose.yaml` and persistent data directories.
3. Verify that no application ports are exposed publicly.
4. Document OpenClaw configuration, backup/restore, updates, and rollback in Runbook 005.

## Official references

- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker and UFW networking caveat: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- Docker daemon attack surface: https://docs.docker.com/engine/security/#docker-daemon-attack-surface
