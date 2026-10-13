# Server constraints (redacted)

**Source:** `/Users/vernikr/server-spec.md`
**Purpose:** capture infrastructure facts that affect the logging design without copying credentials or
private operational configuration into this repository.

## Available capacity

- Ubuntu 24.04 LTS on x86_64.
- 1 vCPU, 1 GB RAM, 2 GB swap, and a 20 GB SSD.
- Existing systemd, journald, `git`, `curl`, `jq`, `rg`, `uv`, Python 3.12, and common build utilities.
- The server already runs a primary VPN and a backup proxy/VPN service.
- A lightweight server-side task already runs there and is bounded by systemd memory limits.

## Hard boundaries

- Do not change or rebind the existing VPN/proxy ports.
- Do not stop, disable, or modify the VPN/proxy systemd units or their configuration directories.
- Do not modify firewall or forwarding rules as part of this project unless a later, explicit deployment
  decision authorizes it and preserves all existing access paths.
- Avoid unneeded reboots.
- Keep any new background service within the existing safety envelope: roughly 250–350 MB RAM, with an
  explicit `MemoryMax` and bounded disk behavior.
- Use `uv` virtual environments for Python services; do not install Python packages globally.

## Existing operational patterns

- `journald` already captures stdout/stderr for systemd services and has a bounded system journal.
- One existing project is deployed with `rsync` from the workstation rather than a Git checkout.
- Other projects use GitHub release/deploy tooling with health checks and rollback behavior.
- The server is a plausible collection/query point for cloud agents, but it is not a reason to make every
  client network-dependent.

## Design consequences

1. Prefer a small collector or query layer over a heavy observability stack.
2. Reuse journald for the first server runtime where possible; avoid duplicating every event into multiple
   always-on daemons before measuring the need.
3. Use append-only, bounded, rotated storage and define retention before enabling broad collection.
4. Keep authentication and network exposure separate from the event schema.
5. Make the remote path optional for offline/restricted clients such as Figma plugins.
6. Measure CPU, RAM, disk growth, and delivery backlog with the first real adapter before onboarding more
   projects.

## Credential handling

The source specification contains an SSH private key and operational access details. They were used only
as authorized context and are intentionally not copied here. Any future deployment must read credentials
from the existing private environment or an explicit secret mechanism, never from this repository.
