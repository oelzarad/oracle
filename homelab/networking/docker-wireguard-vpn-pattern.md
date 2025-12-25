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

### Kill-Switch Implementation – Ordering Caveat

When adding the VPN kill-switch rule to `wg0.conf`, the initial implementation used:


Using **`-I` (insert)** places the rule at the **top** of the chain, which caused:

- The REJECT rule to run **before** the RFC1918 ACCEPT rules
- All local and Docker-internal traffic to be blocked
- Result: Radarr, Sonarr, NZB apps, etc. were no longer reachable

#### Correct Implementation

The kill-switch must be **appended**, not inserted, so that it executes **after** LAN-allowed ACCEPT rules.

The corrected rule uses **`-A` (append)**:


#### Why Order Matters

Order of rules in `OUTPUT` is evaluated **top-down**.  
If the kill-switch is first, it catches *all* traffic and prevents later ACCEPT rules from matching.  
Appending instead ensures:

1️⃣ Allow LAN  
2️⃣ Allow Docker internal  
3️⃣ Block anything not going through `wg0`

#### Verification Checklist

To confirm correct order:

docker exec -it wireguard sh -lc 'iptables -S OUTPUT'

Expected:
-A OUTPUT -d 192.168.0.0/16 -j ACCEPT
-A OUTPUT -d 10.0.0.0/8 -j ACCEPT
-A OUTPUT -d 172.16.0.0/12 -j ACCEPT
-A OUTPUT ! -o wg0 -m addrtype ! --dst-type LOCAL -j REJECT


If the REJECT line appears **above** the ACCEPT lines → the kill-switch will block all apps.


### Kill-Switch Attempt and Why It Was Removed

We experimented with adding a global OUTPUT kill-switch rule:

```bash
iptables -A OUTPUT ! -o wg0 -m addrtype ! --dst-type LOCAL -j REJECT
```

Although this successfully blocked non-VPN internet traffic, it also had an unintended side effect:

WireGuard’s own control traffic to the VPN endpoint (public IP on eth0) was blocked.

Once those packets were rejected, the tunnel stopped working.

As a result, Radarr/Sonarr lost all internet connectivity even though the containers were running.

Because this setup runs on Docker Desktop (macOS) with LinuxServer.io WireGuard and nftables under the hood, implementing a robust kill-switch without breaking the tunnel would require more complex, endpoint-specific firewall rules.

For now, the kill-switch rule has been removed, and we rely on:

Full-tunnel routing via WireGuard

RFC1918 route exceptions for LAN/Docker access

This keeps the configuration simple and reliable while still ensuring that application traffic is routed through the VPN under normal operation.