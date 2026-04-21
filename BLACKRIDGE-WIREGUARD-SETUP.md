# Blackridge WireGuard Setup

## Purpose

This runbook defines the exact WireGuard setup for Blackridge remote admin access.

This is now the confirmed Blackridge remote-admin VPN path.

Use it instead of Teleport when VPN clients must use `rainier` for DNS.

## Confirmed working state

- VPN type: `WireGuard Server` on the UniFi gateway
- VPN network: `192.168.60.0/24`
- Gateway / server address: `192.168.60.1`
- Client DNS: `192.168.10.10`
- Client posture: full tunnel
- Privilege model: same as `secure`
- Intended users: admin devices only

Validated outcome:

- WireGuard works with the Blackridge UniFi gateway
- VPN clients successfully resolve through `rainier` at `192.168.10.10`
- this avoids the Teleport DNS limitation for admin clients

## Why WireGuard

- Teleport is convenient, but DNS behavior is tied to the UniFi gateway path.
- Blackridge requires VPN clients to resolve through `rainier`.
- WireGuard gives explicit client configuration and predictable DNS behavior.

## UniFi setup

UniFi WebUI area:

- `Settings` -> `VPN`
- on some builds: `Teleport & VPN`

### 1. Create the WireGuard server

Set:

- `Type`: `WireGuard Server`
- `Name`: `Blackridge WireGuard`
- `Port`: `51820`
- `VPN Network`: `192.168.60.0/24`

If UniFi asks for a gateway or server address, use:

- `192.168.60.1`

If UniFi offers DNS server fields, set:

- primary DNS: `192.168.10.10`

If UniFi offers split-tunnel vs full-tunnel, choose:

- full tunnel

Reason:

- keeps remote admin behavior simple
- ensures DNS resolution goes to `rainier`
- matches the documented `vpn` zone posture

### 2. Create the first peer

Create one test peer first.

Suggested name:

- `lane-mba`

Keep this as an admin-only client.

Export:

- QR code for mobile
- config file for desktop/laptop

### 3. Client config expectations

The WireGuard client config should effectively include:

- a client address inside `192.168.60.0/24`
- `DNS = 192.168.10.10`
- `AllowedIPs = 0.0.0.0/0, ::/0` for full tunnel
- endpoint = your public IP or DDNS name with UDP `51820`

If UniFi generates the config automatically, verify these values before relying on it.

Confirmed behavior:

- the generated configuration can be used successfully for a macOS client
- DNS testing succeeded against `192.168.10.10`

## Upstream network requirements

WireGuard needs UDP `51820` to reach the UniFi gateway WAN interface.

If the UniFi gateway has the public IP directly:

- no upstream port forward is needed

If the UniFi gateway is behind another router:

- forward UDP `51820` to the UniFi gateway WAN IP

## Validation checklist

### 1. Tunnel health

- [x] client connects successfully
- [x] client receives a `192.168.60.x` address

### 2. DNS

- [x] client resolves names through `192.168.10.10`
- [x] internal names like `rainier.blackridge.shumie.net` resolve correctly
- [x] `dns.blackridge.shumie.net` resolves correctly

### 3. Admin reachability

- [ ] UniFi admin is reachable
- [x] `rainier` is reachable at `192.168.10.10`
- [ ] `blackcomb` is reachable at `192.168.10.20`
- [ ] NAS is reachable at `192.168.10.30`

### 4. Policy posture

- [ ] VPN client can reach `infrastructure`
- [ ] VPN client can reach `home`
- [ ] VPN client cannot broadly reach `iot`
- [ ] VPN client cannot reach `hotspot`

## Recommended first test commands

From the connected VPN client:

```bash
dig @192.168.10.10 rainier.blackridge.shumie.net +short
dig @192.168.10.10 dns.blackridge.shumie.net +short
ping 192.168.10.10
ping 192.168.10.20
ping 192.168.10.30
```

## Rollback

If WireGuard does not work:

1. keep Teleport available temporarily
2. disable the new peer
3. verify upstream UDP `51820`
4. verify the generated client config actually includes `DNS = 192.168.10.10`
5. verify UniFi firewall policy still treats `vpn` as admin-only

## Operational note

Treat WireGuard as the primary admin VPN.

Teleport can remain available as a convenience fallback, but the documented Blackridge target state is:

- WireGuard for durable remote admin
- `rainier` as VPN DNS

For the planned public endpoint name and Route 53 DDNS model, see:

- `BLACKRIDGE-AWS-ROUTE53.md`
