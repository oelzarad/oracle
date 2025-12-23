# Docker + WireGuard VPN Pattern (LinuxServer.io)
**Split Tunneling + Kill Switch + Docker Desktop (macOS)**

This document captures a **working, production-safe pattern** for routing Docker containers through a WireGuard VPN using the LinuxServer.io image.

It includes:
- Split tunneling (LAN access preserved)
- A true VPN kill switch (no leaks)
- Docker Desktop quirks and fixes
- Verified commands and behaviors

This file is intended to be **used directly by Codex CLI** as reference context.

---

## Goals

- Force specific containers (Radarr, Sonarr, etc.) through a VPN
- Prevent traffic leaks if VPN drops
- Maintain access to Web UIs from the LAN
- Keep the setup **simple and Docker-native**

Non-goals:
- macvlan
- host-level firewalling
- per-container routing rules

---

## Architecture

**Key idea:**  
One container owns the network namespace; other containers share it.

## Troubleshooting and Operational Notes

## Docker Desktop Port-Forwarding Failure Mode

During troubleshooting, all container-level diagnostics indicated a healthy system:

- Services were listening on the expected ports
- Applications responded correctly from inside the container namespace
- WireGuard routing and firewall rules were correct
- Docker reported ports as successfully published

However, connections from the host to published ports would:
- Successfully establish a TCP connection
- Hang indefinitely without returning any data

This behavior was caused by Docker Desktop’s networking stack becoming wedged
(userland proxy / VM networking layer), not by container configuration.

### Resolution

A full restart of the Docker Engine (Docker Desktop) immediately resolved the issue.

Restarting individual containers or Docker Compose projects was **not sufficient**.

### Diagnostic Indicator

If all of the following are true:
- `curl` connects to a published port but times out
- The service responds correctly from inside the container
- VPN routing appears correct

Then the recommended action is:

> Restart Docker Desktop before continuing troubleshooting.

This issue has been observed in setups involving:
- `network_mode: service:<container>`
- VPN containers (WireGuard/OpenVPN)
- User-defined bridge networks
- Docker Desktop on macOS


## Manual `wg0.conf` Modifications (LinuxServer WireGuard)

This setup requires **manual edits** to the WireGuard configuration file:


These changes are **not optional** and are **not generated automatically** by Docker or the LinuxServer image.

### Why Manual Changes Are Needed

When containers share the WireGuard container’s network namespace
(`network_mode: service:wireguard`), Docker does not preserve local
(LAN / Docker bridge) access by default.

Without manual routing and firewall rules:
- Local Web UIs become unreachable
- LAN services (NAS, download clients) may break
- Traffic may leak if the VPN drops

### What Was Added

The following logic was added manually to `wg0.conf`:

- Explicit routing exclusions for all RFC1918 networks  
  (`192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`)
- Matching exclusions in **both**:
  - the main routing table
  - WireGuard’s policy routing table (`51820`)
- OUTPUT firewall rules to allow local traffic
- A final OUTPUT rule enforcing a VPN **kill switch**

### Important Constraints

- `PostUp` and `PreDown` **must be single-line commands**
  (LinuxServer.io requirement)
- These rules live **inside `wg0.conf`**, not Docker Compose
- Restarting containers alone does **not** apply changes  
  → the WireGuard container must be restarted

### Operational Reminder

Any time `wg0.conf` is modified:
1. Restart the WireGuard container
2. Verify routes and rules were applied
3. Confirm local access and VPN egress

Failure to maintain these manual rules can result in:
- Loss of local access
- Silent traffic leaks
- Misleading symptoms during debugging

