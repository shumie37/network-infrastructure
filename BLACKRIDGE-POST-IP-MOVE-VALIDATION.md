# Blackridge Post-IP-Move Validation

## Purpose

This document is the validation plan to use after core infrastructure hosts have already been moved to their target addresses.

Current assumed target state:

- `rainier` = `192.168.10.10`
- `blackcomb` = `192.168.10.20`
- `NAS` = `192.168.10.30`

This plan focuses on dependency reconciliation rather than host migration.

## Current validated state

As of the latest remediation pass on `rainier`:

- AdGuard DNS is healthy on `192.168.10.10:53`
- FreeRADIUS is healthy on `192.168.10.10` and serving:
  - `CN=radius.blackridge.shumie.net`
  - SAN `DNS:radius.blackridge.shumie.net`
  - SAN `IP:192.168.10.10`
  - validity through `2026-07-16`
- `step-ca` is healthy and its configured extra hostname/IP includes `192.168.10.10`
- Caddy upstreams have been updated to the new infrastructure addresses
- `dns.blackridge.shumie.net`, `nas.blackridge.shumie.net`, `luna-admin.blackridge.shumie.net`, and `ha.blackridge.shumie.net` are serving Let's Encrypt certificates
- no core dependency should still rely on `192.168.3.0/24`

## Validation goals

1. Confirm all core services are healthy on the new infrastructure addresses.
2. Confirm UniFi now points to the new infrastructure addresses everywhere it matters.
3. Confirm trusted clients are using the intended DNS and authentication paths.
4. Confirm reverse proxy and internal service publishing are aligned to the new network.
5. Remove stale references to the old infrastructure subnet once the new path is verified.

## Step 1: Confirm core host reachability

From an approved admin device on `secure`, verify:

- [ ] `rainier` reachable at `192.168.10.10`
- [ ] `blackcomb` reachable at `192.168.10.20`
- [ ] `NAS` reachable at `192.168.10.30`
- [ ] UniFi gateway, switch, and APs reachable on their target infrastructure addresses

If any of these fail, stop and restore basic management reachability before validating higher-level services.

## Step 2: Validate DNS on `rainier`

Expected target:

- trusted networks use `192.168.10.10` as DNS
- `guest` / `Hotspot` continues to use gateway DNS behavior

Checks:

- [ ] AdGuard / DNS service on `rainier` is listening on `192.168.10.10`
- [ ] name resolution works from `secure`
- [ ] name resolution works from `home`
- [ ] name resolution works from `iot`
- [ ] UniFi network settings for trusted networks point to `192.168.10.10`
- [ ] firewall rules allow DNS traffic to `192.168.10.10`
- [ ] no active trusted-network DNS object still points to the old infrastructure subnet

UniFi object / policy checks:

- [ ] `Rainier DNS` object = `192.168.10.10`
- [ ] `Home -> Rainier DNS` rules point to `192.168.10.10`
- [ ] `IoT -> Rainier DNS` rules point to `192.168.10.10`

## Step 3: Validate RADIUS on `rainier`

Expected target:

- UniFi external RADIUS server points to `192.168.10.10`

Checks:

- [ ] FreeRADIUS is healthy on `rainier`
- [ ] UniFi RADIUS server object points to `192.168.10.10`
- [ ] ports `1812/1813` are reachable from the UniFi infrastructure path
- [ ] `Blackridge` secure SSID still authenticates successfully
- [ ] FreeRADIUS logs show successful requests from the new network path
- [ ] RADIUS server certificate SAN includes `192.168.10.10`
- [ ] RADIUS server certificate validity window is longer than the temporary bootstrap leaf

## Step 4: Validate secure Wi-Fi access

Checks:

- [ ] at least one admin Apple device can join `Blackridge`
- [ ] device gets a `192.168.20.x` address
- [ ] DNS on that device resolves through `192.168.10.10`
- [ ] UniFi admin is reachable from that device
- [ ] `rainier`, `blackcomb`, and NAS admin surfaces are reachable from that device

## Step 5: Validate reverse proxy and service publishing

Expected target:

- internal reverse proxy and service dependencies no longer rely on old infrastructure addresses

Checks:

- [ ] Caddy or equivalent reverse proxy reaches its upstreams on the new `192.168.10.x` addresses
- [ ] internal service hostnames resolve correctly
- [ ] admin apps behind the reverse proxy are reachable from `secure`
- [ ] user-facing internal apps intended for `home` are still reachable where permitted
- [ ] no hard-coded old infrastructure IPs remain in reverse proxy upstream definitions or local firewall allowlists
- [ ] public hostnames intended for public TLS are serving Let's Encrypt certificates

## Step 6: Validate firewall and object consistency

Checks:

- [ ] `rainier` UFW no longer contains `192.168.3.0/24` legacy LAN rules
- [ ] `rainier` UFW no longer contains `192.168.2.0/24` Teleport-era rules
- [ ] `NAS` object = `192.168.10.30`
- [ ] `Home -> NAS SMB` rule points to `192.168.10.30`
- [ ] `Home -> NAS Wake-on-LAN` rule points to `192.168.10.30`
- [ ] `Home -> Rainier HTTPS` rule points to `192.168.10.10:443`
- [ ] `Rainier -> HP Printer HTTPS` rule points to `192.168.40.10:443`
- [ ] `Secure -> Infrastructure` rule still grants the intended broad admin path
- [ ] `Home -> Secure` remains blocked
- [ ] `IoT -> Secure` remains blocked
- [ ] `Guest` / `Hotspot` remains isolated from internal zones
- [ ] deferred Home Assistant / automation rules remain absent unless intentionally introduced

Current `rainier` UFW target state:

| Service | Ports | Allowed source networks | Notes |
| --- | --- | --- | --- |
| DNS | `53/tcp`, `53/udp` | `Infrastructure`, `Secure`, `Home`, `IoT`, `VPN` | `Hotspot` remains excluded |
| SSH | `22/tcp` | `Infrastructure`, `Secure`, `VPN` | admin surfaces only |
| Reverse proxy / web | `80/tcp`, `443/tcp` | `Infrastructure`, `Secure`, `Home`, `VPN` | supports admin and approved internal apps |
| Home Assistant direct | `8123/tcp` | `Infrastructure`, `Secure`, `VPN` | kept off `Home`, `IoT`, and `Hotspot` |
| MQTT | `1883/tcp` | `Infrastructure` | internal automation only |
| RADIUS | `1812/udp`, `1813/udp` | `Infrastructure` | UniFi infrastructure path only |
| Internal container paths | `3000/tcp`, `8123/tcp`, `53/tcp`, `53/udp` | Docker bridge networks | required for local reverse proxy and container DNS behavior |

Target published internal app path:

- `printer.blackridge.shumie.net` terminates on `rainier` via Caddy and reverse proxies to `https://192.168.40.10`
- expected consumers are `secure` and `home`

## Step 7: Validate WireGuard / VPN

Checks:

- [ ] one WireGuard client can connect successfully
- [ ] client receives a `192.168.60.x` address
- [ ] VPN client DNS resolves through `192.168.10.10`
- [ ] VPN client has the intended `secure`-equivalent administrative reach

## Step 8: Remove stale references

Only after all previous checks pass:

- [ ] remove stale UniFi objects pointing at old infrastructure IPs
- [ ] remove stale firewall references to the old infrastructure subnet
- [ ] remove stale DNS or reverse proxy references to the old infrastructure subnet
- [ ] update documentation if any real-world IPs differ from the planned values

## Stop conditions

Stop and correct the issue before continuing if any of these occur:

- [ ] DNS on `192.168.10.10` is not stable
- [ ] RADIUS on `192.168.10.10` fails or becomes intermittent
- [ ] secure SSID authentication fails
- [ ] reverse proxy upstreams fail after the IP move
- [ ] admin access from `secure` is degraded
- [ ] stale old-subnet references are still required for core functionality

## Completion criteria

The post-IP-move phase is complete when:

- [ ] DNS works from all intended trusted zones via `192.168.10.10`
- [ ] RADIUS works via `192.168.10.10`
- [ ] reverse proxy and service publishing work on the new infrastructure addresses
- [ ] Let's Encrypt coverage matches the intended public hostnames
- [ ] the local CA and RADIUS leaf reflect `192.168.10.10`
- [ ] UniFi firewall and object references match the new infrastructure addresses
- [ ] no operational dependency remains on the old infrastructure subnet
