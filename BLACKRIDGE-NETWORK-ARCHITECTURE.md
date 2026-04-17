# Blackridge Network Architecture

## Purpose

`blackridge` is the network infrastructure layer that shared infrastructure systems such as `rainier` and `blackcomb` sit on top of.

This document defines the target-state network architecture for the Blackridge site. It is intended to be the source of truth for:

- network zones
- VLAN and subnet assignments
- SSID mapping
- host placement
- security boundaries
- DNS and DHCP behavior
- remote access posture
- wired port design
- rollout sequencing

This is a greenfield target design, not a description of the current flat-network state.

## Design principles

- separate network-layer identity from server and service identity
- keep the high-privilege admin enclave very small
- prefer explicit access over broad discovery
- use simple, memorable VLAN and subnet numbering
- centralize DNS on `rainier` for all trusted networks
- keep guest fully untrusted and isolated
- keep infrastructure single-homed and policy-controlled
- stage advanced features only after the base design is stable

## Network zones

### `secure`

High-privilege admin enclave.

Intended use:

- administrative devices only
- management access to infrastructure and admin surfaces
- normal access to user-facing `home` services

Current intended members:

- MacBook (LS)
- iPad (LS)

### `infrastructure`

Shared back-end systems and network gear.

Intended use:

- UniFi gateway, switch, and APs
- `rainier`
- `blackcomb`
- NAS
- future service hosts and fixed infrastructure systems

This network is wired-only in phase 1.

### `home`

Normal household and personal device network.

Intended use:

- non-admin Macs
- iPhones
- iPads not assigned to `secure`
- Apple TV
- speakers
- printer
- WiiM
- similar user-facing household devices

This network is for convenience and normal use, not infrastructure administration.

### `iot`

Restricted IoT device network.

Intended use:

- Nest
- Govee lights
- Pura devices
- similar automation and appliance endpoints

Phase 1 keeps IoT simple: normal internet access, no broad access into other internal networks, and no client isolation inside the IoT network yet.

### `hotspot`

Fully untrusted guest network.

Intended use:

- transient guest devices only

Policy objective:

- internet only
- no internal access
- client isolation enabled
- captive portal enabled

### `vpn`

Remote-access network zone.

Intended use:

- UniFi WireGuard remote clients only

Policy objective:

- same privilege posture as `secure`
- own dedicated subnet for routing and documentation clarity
- full-tunnel remote access
- admin-grade access limited to approved users and devices

## Friendly SSID mapping

Internal network names remain functional. User-facing SSID names are branded.

| SSID | Internal network | Notes |
|---|---|---|
| `Blackridge` | `secure` | WPA3-Enterprise with `802.1X` / `EAP-TLS` |
| `ShuMK` | `home` | WPA3-Personal in phase 1 |
| `ShuMK IoT` | `iot` | WPA2/WPA3 transition mode in phase 1 |
| `ShuMK Guest` | `hotspot` | WPA2/WPA3 transition mode with captive portal |

`infrastructure` has no Wi-Fi SSID in phase 1.

## VLAN and subnet plan

| VLAN | Network | Subnet | Gateway |
|---|---|---|---|
| 1 | `infrastructure` | `192.168.10.0/24` | `192.168.10.1` |
| 2 | `secure` | `192.168.20.0/24` | `192.168.20.1` |
| 3 | `home` | `192.168.30.0/24` | `192.168.30.1` |
| 4 | `iot` | `192.168.40.0/24` | `192.168.40.1` |
| 5 | `hotspot` | `192.168.50.0/24` | `192.168.50.1` |
| 6 | `vpn` | `192.168.60.0/24` | `192.168.60.1` |

`vpn` is a routed remote-access network, not an SSID-backed LAN.

## Addressing conventions

### Global conventions

- gateway is `.1` in every subnet
- infrastructure network gear occupies low sequential addresses under `.10`
- non-network infrastructure hosts use `.10`, `.20`, `.30`, etc.
- no static client IPs on `secure`
- `home` and `iot` prefer dynamic addressing unless a later policy requires reservations

### Infrastructure host breakout

| IP | Host |
|---|---|
| `192.168.10.1` | Blackridge Gateway |
| `192.168.10.2` | Blackridge Switch |
| `192.168.10.3` | Blackridge AP (Office) |
| `192.168.10.4` | Blackridge AP (Living Room) |
| `192.168.10.5` | reserved for future network gear |
| `192.168.10.6` | reserved for future network gear |
| `192.168.10.7` | reserved for future network gear |
| `192.168.10.8` | reserved for future network gear |
| `192.168.10.9` | reserved for future network gear |
| `192.168.10.10` | Rainier |
| `192.168.10.20` | Blackcomb |
| `192.168.10.30` | NAS |

## DHCP and DNS model

### DHCP

- the UniFi gateway provides DHCP for every network
- `infrastructure` uses static or reserved addresses as needed
- `secure` uses a small DHCP scope with no client-side static IPs
- `home`, `iot`, and `hotspot` use normal DHCP pools
- `vpn` receives gateway-provided addressing within the `192.168.60.0/24` subnet

### DNS

`rainier` is the required DNS server for all trusted internal networks:

- `secure`
- `infrastructure`
- `home`
- `iot`
- `vpn`

`hotspot` is the only exception:

- clients use the gateway as DNS
- the gateway resolves upstream using DoH

### DNS enforcement

Policy should enforce standard DNS to the approved resolver only:

- internal trusted networks should be blocked from sending port 53 DNS to anything except `rainier`
- `hotspot` should be blocked from internal DNS entirely
- blocking all encrypted DNS variants perfectly is out of scope for phase 1

Phase 1 expectation:

- `rainier` is the required and documented resolver for trusted networks
- firewall policy must enforce standard DNS to `rainier`
- encrypted DNS bypass is treated as an accepted residual risk in phase 1, not as permitted design intent
- if trusted Apple clients show material DNS bypass behavior, add targeted platform controls or tighter egress policy in a later phase

## Authentication and Wi-Fi posture

### Phase 1 authentication

- `secure`: WPA3-Enterprise with `802.1X` / `EAP-TLS`
- `home`: WPA3-Personal
- `iot`: WPA2/WPA3 transition mode
- `hotspot`: WPA2/WPA3 transition mode with captive portal

`secure` uses per-device certificate identity rather than a shared Wi-Fi password.

Rationale:

- `secure` is the highest-privilege enclave
- device-level certificate identity supports revocation and attribution
- a shared WPA3-Personal passphrase is not an acceptable long-term control for the admin enclave

Implementation model for `secure`:

- UniFi SSID backed by external RADIUS
- FreeRADIUS on `rainier`
- internal CA on `rainier` for RADIUS and client certificates
- one Apple Wi-Fi profile per device
- TLS compatibility floor of `1.2` with `1.3` preferred upper bound

### Phase 1 Wi-Fi tuning posture

Keep Wi-Fi optimization conservative at launch:

- minimum RSSI: off
- band steering: off
- fast roaming: off
- all SSIDs broadcast normally
- allow UniFi to determine radio behavior initially

Tune later from observed client behavior after VLAN, DHCP, and firewall policy are stable.

### Authentication rollout posture

- `secure` is intended to launch on `WPA3-Enterprise`, not as a temporary WPA3-Personal network
- `home`, `iot`, and `hotspot` remain on simpler authentication models in phase 1
- if `secure` needs temporary fallback during cutover, that fallback should be treated as a migration exception and not as the documented target state

## Device placement

### `secure`

- MacBook (LS)
- iPad (LS)

### `infrastructure`

- Blackridge Gateway
- Blackridge Switch
- Blackridge AP (Office)
- Blackridge AP (Living Room)
- Rainier
- Blackcomb
- NAS

### `home`

- iPhone (LS)
- iPhone (MK)
- iPad (MK)
- Mac 1f:ff
- MacBook Air (Oracle)
- Apple TV (Living Room)
- TV (Living Room)
- Speaker (Bedroom)
- Speaker (Living Room)
- Speaker (Office)
- Printer (Office)
- WiiM Pro (Living Room)
- similar general personal and household devices

### `iot`

- Nest Thermostat
- Light (Bedroom)
- Light (Column)
- Light (Living Room)
- Lights (Office)
- Pura (Bedroom)
- Pura (Entryway)
- Pura (Office)
- similar IoT endpoints

### `hotspot`

- transient guest devices only

### `vpn`

- UniFi WireGuard remote clients
- same policy posture as `secure`

## Routing model

- inter-VLAN routing is enabled on the gateway
- `hotspot` is the only network treated as fully isolated in practice
- all other networks are routable, with boundaries enforced by firewall policy
- `vpn` is full-tunnel and follows `secure` policy

## Wired port and trunk design

### Topology intent

- gateway to switch: trunk
- switch to APs: trunks
- infrastructure hosts: access ports on `infrastructure`
- default client-facing wired ports: access ports on `home`

### Trunk policy

#### Gateway to switch

- link type: trunk
- native / management network: `infrastructure`
- allowed VLANs: `2`, `3`, `4`, `5`

#### Switch to APs

- link type: trunk
- native / management network: `infrastructure`
- allowed VLANs: `2`, `3`, `4`, `5`

This allows each AP to present `secure`, `home`, `iot`, and `hotspot` SSIDs while remaining managed on `infrastructure`.

### Access port policy

- Rainier: access `infrastructure`
- Blackcomb: access `infrastructure`
- NAS: access `infrastructure`
- unused switch ports: enabled, default access `home`
- in-wall AP passthrough ports: default access `home` unless intentionally reassigned

### Recommended USW Lite 16 PoE port map

| Port | Role | Profile |
|---|---|---|
| 1 | uplink to gateway | trunk, native `infrastructure`, allow VLANs `10/20/30/40/50` |
| 2 | AP Living Room | trunk, native `infrastructure`, allow VLANs `10/20/30/40/50` |
| 3 | AP Office | trunk, native `infrastructure`, allow VLANs `10/20/30/40/50` |
| 4 | Rainier | access `infrastructure` |
| 5 | Blackcomb | access `infrastructure` |
| 6 | NAS | access `infrastructure` |
| 7-16 | general wired clients | enabled, access `home` |

## Discovery model

Discovery is not treated as a general cross-network feature.

Default posture:

- use explicit access first
- avoid broad cross-network discovery
- add only narrow exceptions later if testing proves Apple-specific discovery needs require them

## Remote access model

### Technology

- UniFi WireGuard server on the UniFi gateway

### Posture

- remote-user access only
- site-to-site VPN is out of scope for phase 1
- full-tunnel
- DNS handed out as `rainier`
- same privilege posture as `secure`
- explicit per-device peer configuration
- standard WireGuard clients rather than WiFiman / Teleport dependency

## Rollout model

### Phase 1: target build

Objectives:

- create all target VLANs and subnets
- re-IP infrastructure into the new plan immediately
- deploy target SSIDs
- move devices directly into their target networks
- stand up WireGuard using the `vpn` network model
- apply baseline firewall boundaries

Recommended migration order:

1. infrastructure first
2. SSIDs and VLAN mapping
3. clients into target networks
4. VPN / WireGuard

### Phase 2: hardening

Objectives:

- refine `secure` operations and certificate lifecycle once RADIUS is stable
- tighten admin-surface exposure further if needed
- evaluate IoT client isolation
- add service-level tightening where rollout experience justifies it
- add discovery exceptions only if proven necessary by testing

## Out of scope for this document

- credential values
- exact NAS service port numbers
- exact Rainier service port numbers beyond current design intent
- site-to-site VPN
- controller click-path instructions
