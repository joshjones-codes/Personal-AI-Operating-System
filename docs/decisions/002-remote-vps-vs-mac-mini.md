# ADR-002: Cloud-first always-on execution over a dedicated Mac mini

- **Status:** Accepted for MVP
- **Date:** 2026-10-31
- **Decision owners:** Herd project

## Context

The assistant must accept instructions and complete cloud-accessible work while the user's MacBook is asleep, disconnected, or off. The user travels and wants low upfront cost, remote management, and a platform for learning cloud operations. Some future tasks—Apple Photos, local apps, and local herdr sessions—depend on macOS and cannot run on a Linux VPS alone.

## Decision

Run the **primary OpenClaw Gateway on an always-on Linux VPS**. Treat the MacBook as an **optional connected node**, available only when powered on and reachable. Do not purchase a dedicated Mac mini for the MVP.

## Why

1. Cloud execution remains available independently of the laptop.
2. A VPS avoids hardware purchase, local power/network dependencies, and physical maintenance.
3. Linux deployment provides practical experience in networking, service supervision, secrets, patching, and backups.
4. We can scale cloud workers separately from local macOS capabilities.

## Alternatives considered

- **Dedicated Mac mini at home:** Better for unattended macOS-native automation and local Photos access, but has upfront cost and relies on home power and internet.
- **MacBook-only Gateway:** Cheapest but fails the explicit requirement to work when the laptop is off.
- **Hybrid VPS + Mac mini:** Possible later if macOS-native workloads become frequent enough to justify dedicated hardware.

## Boundaries

- **Works with Mac off:** Telegram messages, cloud-hosted repositories and workers, scheduled cloud tasks, cloud APIs, notifications.
- **Does not work with Mac off:** Reading local-only files, manipulating the local Photos library, or controlling local terminal panes. Such jobs must wait for the Mac to reconnect or be moved to a continuously available host.
- A VPS is not automatically highly available: provider outages, restarts, and deployment failures still matter.

## Operational baseline

- Non-root service identity; SSH keys; minimal firewall exposure.
- Private administration path (for example, Tailscale or SSH tunnel); no unauthenticated public gateway endpoint.
- Persistent volume and tested backups; system service restart policy.
- Structured task records, health checks, logs, and notification on failure.
- Human approval for deletion, purchases, infrastructure mutation, and publishing.

## MVP acceptance criteria

- Close/power off the MacBook; issue a request from the phone; receive a cloud-generated response.
- Reboot the VPS; Gateway and persisted task data recover.
- An unavailable Mac-dependent task fails clearly or queues safely rather than silently pretending success.

## Revisit if

- macOS-only tasks become the dominant workload.
- An always-on local Mac is needed for Photos, GUI, or local app automation.
- Hosting economics or privacy requirements favor owned hardware.
