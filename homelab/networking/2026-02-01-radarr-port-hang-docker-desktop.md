# Radarr LAN Access Hang - Docker Desktop Port Forwarding Wedge

Date: 2026-02-01

## Summary
Radarr was healthy inside the WireGuard network namespace, but LAN access to the published port hung indefinitely. Root cause was Docker Desktop port-forwarding being wedged.

## Symptoms
- Browser and curl to host LAN IP or 127.0.0.1:7878 connected, then hung with no response.
- Service responded normally from inside the WireGuard container namespace.

## Diagnostics
- Inside namespace: `docker exec wireguard sh -lc 'curl -sS --max-time 5 http://127.0.0.1:7878'` returned HTML.
- Host: `curl -sv --max-time 5 http://127.0.0.1:7878` connected but timed out with 0 bytes.

## Root Cause
Docker Desktop networking (userland proxy / VM port-forwarding layer) was wedged. Container-level networking and VPN routing were healthy.

## Fix
Restart Docker Desktop (full engine restart, not just containers). After restart, Radarr was reachable on port 7878 from the LAN.

## Next Steps
- If this recurs, restart Docker Desktop before deeper VPN or firewall debugging.
- Keep wg0.conf LAN exclusions in place for shared network namespace containers.
