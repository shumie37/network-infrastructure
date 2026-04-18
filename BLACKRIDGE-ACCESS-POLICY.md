# Blackridge Access Policy

## Purpose

This document defines the phase 1 logical access model for the Blackridge network architecture.

It is intentionally split into:

- a human-readable access matrix for architectural review
- an ordered ruleset suitable for implementation planning
- short service-intent notes for later technical rule definition

## Access matrix

| Source | Destination | Policy | Notes |
|---|---|---|---|
| `secure` | `infrastructure` | allow | full admin and service access |
| `secure` | `home` | allow | normal household service use is permitted |
| `secure` | `iot` | deny by default | prefer control through `infrastructure` and Home Assistant |
| `secure` | `guest` | deny | no meaningful use case |
| `secure` | internet | allow | normal outbound access |
| `vpn` | same as `secure` | same as `secure` | UniFi WireGuard full-tunnel remote access |
| `home` | `secure` | deny | protect admin enclave |
| `home` | `infrastructure` | allow by exception | approved services only |
| `home` | `iot` | allow | direct control is allowed |
| `home` | internet | allow | standard client internet access |
| `iot` | `secure` | deny | no admin access |
| `iot` | `home` | deny except return traffic | no lateral initiation |
| `iot` | `infrastructure` | allow by exception | DNS to Rainier and required service flows only |
| `iot` | internet | allow | normal internet access |
| `infrastructure` | `secure` | deny except return traffic | protect admin enclave |
| `infrastructure` | `home` | deny by default | no broad reach into user devices |
| `infrastructure` | `iot` | allow by exception | only specific hosts such as Rainier as needed |
| `infrastructure` | internet | allow | normal outbound access |
| `guest` | any internal network | deny | fully untrusted |
| `guest` | internet | allow | captive portal and internet only |
| `guest` | guest peers | deny | client isolation enabled |

## Zone-based policy table

This table is the intended phase 1 Zone-Based Firewall / Policy Table translation of the access model above.

Assumptions:

- rules are stateful, so established and related return traffic is not listed separately
- `Any` in `Src.` or `Dst.` means the whole zone unless a narrower host or object is named
- named destinations such as `Rainier DNS` or `Approved NAS Services` should be implemented as UniFi objects
- UniFi assigns rule IDs automatically; treat the `ID` column as controller-generated reference data rather than a user-controlled design field
- UniFi may represent the `guest` network as the built-in `Hotspot` zone in the Zone-Based Firewall UI; treat `guest` and `Hotspot` as equivalent for policy implementation unless UniFi changes that platform behavior

## Current live UniFi policy details

The controller screenshots captured on `2026-04-17` confirm the following live policy details as the current working source of truth:

- UniFi labels the infrastructure LAN as `Internal` in the policy UI
- masquerade NAT is active for the internal routed networks toward the internet uplinks
- `Home` allows DNS to `rainier` at `192.168.10.10`
- `Home` has explicit printer exceptions to the IoT printer at `192.168.40.10`
- `Secure` has explicit printer exceptions to the IoT printer at `192.168.40.10`
- `Rainier` at `192.168.10.10` has an explicit HTTPS allow to the IoT printer at `192.168.40.10`
- `Hotspot` is explicitly allowed DHCP, DHCPv6, and DNS to the gateway and blocked from internal zones

Current observed printer-related objects and targets:

| Object / Target | Value |
| --- | --- |
| Printer target | `192.168.40.10` |
| `Allow Printer (BONJ)` | `UDP 5353` |
| `Allow Printer (IPP)` | `TCP 631` |
| `Allow Printer (JETD)` | `TCP 9100` |
| `Allow Internal to Printer (HTTPS)` | `TCP 443` |

| Name                                                | Action  | IP Version | Protocol | Src. Zone        | Src.                                    | Src. Port | Dst. Zone        | Dst.                           | Dst. Port      | ID      |
| --------------------------------------------------- | ------- | ---------- | -------- | ---------------- | --------------------------------------- | --------- | ---------------- | ------------------------------ | -------------- | ------- |
| `Allow Secure to Infrastructure`                    | `Allow` | `Both`     | `All`    | `secure`         | `Any`                                   | `Any`     | `infrastructure` | `Any`                          | `Any`          | `10000` |
| `Allow Secure to Home`                              | `Allow` | `Both`     | `All`    | `secure`         | `Any`                                   | `Any`     | `home`           | `Any`                          | `Any`          | `10010` |
| `Block Secure to IoT`                               | `Block` | `Both`     | `All`    | `secure`         | `Any`                                   | `Any`     | `iot`            | `Any`                          | `Any`          | `10020` |
| `Block Secure to Guest`                             | `Block` | `Both`     | `All`    | `secure`         | `Any`                                   | `Any`     | `guest`          | `Any`                          | `Any`          | `10030` |
| `Allow Secure to Internet`                          | `Allow` | `Both`     | `All`    | `secure`         | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10040` |
| `Allow VPN to Infrastructure`                       | `Allow` | `Both`     | `All`    | `vpn`            | `Any`                                   | `Any`     | `infrastructure` | `Any`                          | `Any`          | `10100` |
| `Allow VPN to Home`                                 | `Allow` | `Both`     | `All`    | `vpn`            | `Any`                                   | `Any`     | `home`           | `Any`                          | `Any`          | `10110` |
| `Block VPN to IoT`                                  | `Block` | `Both`     | `All`    | `vpn`            | `Any`                                   | `Any`     | `iot`            | `Any`                          | `Any`          | `10120` |
| `Block VPN to Guest`                                | `Block` | `Both`     | `All`    | `vpn`            | `Any`                                   | `Any`     | `guest`          | `Any`                          | `Any`          | `10130` |
| `Allow VPN to Internet`                             | `Allow` | `Both`     | `All`    | `vpn`            | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10140` |
| `Block Home to Secure`                              | `Block` | `Both`     | `All`    | `home`           | `Any`                                   | `Any`     | `secure`         | `Any`                          | `Any`          | `10200` |
| `Allow Home to IoT`                                 | `Allow` | `Both`     | `All`    | `home`           | `Any`                                   | `Any`     | `iot`            | `Any`                          | `Any`          | `10210` |
| `Allow Home to Rainier DNS UDP`                     | `Allow` | `IPv4`     | `UDP`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `Rainier DNS`                  | `53`           | `10220` |
| `Allow Home to Rainier DNS TCP`                     | `Allow` | `IPv4`     | `TCP`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `Rainier DNS`                  | `53`           | `10221` |
| `Allow Home to Rainier HTTPS`                       | `Allow` | `IPv4`     | `TCP`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `192.168.10.10`                | `443`          | `10222` |
| `Allow Home to NAS SMB`                             | `Allow` | `Both`     | `TCP`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `192.168.10.30`                | `445`          | `UniFi Assigned` |
| `Allow Home to NAS Wake-on-LAN`                     | `Allow` | `Both`     | `UDP`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `192.168.10.30`                | `9`            | `UniFi Assigned` |
| `Block Home to Infrastructure Default`              | `Block` | `Both`     | `All`    | `home`           | `Any`                                   | `Any`     | `infrastructure` | `Any`                          | `Any`          | `10290` |
| `Allow Home to Internet`                            | `Allow` | `Both`     | `All`    | `home`           | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10299` |
| `Block IoT to Secure`                               | `Block` | `Both`     | `All`    | `iot`            | `Any`                                   | `Any`     | `secure`         | `Any`                          | `Any`          | `10300` |
| `Block IoT to Home`                                 | `Block` | `Both`     | `All`    | `iot`            | `Any`                                   | `Any`     | `home`           | `Any`                          | `Any`          | `10310` |
| `Allow IoT to Rainier DNS UDP`                      | `Allow` | `IPv4`     | `UDP`    | `iot`            | `Any`                                   | `Any`     | `infrastructure` | `Rainier DNS`                  | `53`           | `10320` |
| `Allow IoT to Rainier DNS TCP`                      | `Allow` | `IPv4`     | `TCP`    | `iot`            | `Any`                                   | `Any`     | `infrastructure` | `Rainier DNS`                  | `53`           | `10321` |
| `Deferred: IoT to Approved Infrastructure Integrations` | `Review Later` | `Both` | `All` | `iot` | `Any` | `Any` | `infrastructure` | `Approved Integration Targets` | `Object Ports` | `10330` |
| `Block IoT to Infrastructure Default`               | `Block` | `Both`     | `All`    | `iot`            | `Any`                                   | `Any`     | `infrastructure` | `Any`                          | `Any`          | `10390` |
| `Allow IoT to Internet`                             | `Allow` | `Both`     | `All`    | `iot`            | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10399` |
| `Block Infrastructure to Secure`                    | `Block` | `Both`     | `All`    | `infrastructure` | `Any`                                   | `Any`     | `secure`         | `Any`                          | `Any`          | `10400` |
| `Deferred: Infrastructure to Approved IoT Services` | `Review Later` | `Both` | `All` | `infrastructure` | `Rainier and Approved Automation Hosts` | `Any` | `iot` | `Approved IoT Targets` | `Object Ports` | `10410` |
| `Allow Rainier to HP Printer HTTPS`                 | `Allow` | `IPv4`     | `TCP`    | `infrastructure` | `192.168.10.10`                         | `Any`     | `iot`            | `192.168.40.10`                | `443`          | `10411` |
| `Block Infrastructure to Home`                      | `Block` | `Both`     | `All`    | `infrastructure` | `Any`                                   | `Any`     | `home`           | `Any`                          | `Any`          | `10420` |
| `Allow Infrastructure to Internet`                  | `Allow` | `Both`     | `All`    | `infrastructure` | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10499` |
| `Block Guest to Secure`                             | `Block` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `secure`         | `Any`                          | `Any`          | `10500` |
| `Block Guest to Infrastructure`                     | `Block` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `infrastructure` | `Any`                          | `Any`          | `10510` |
| `Block Guest to Home`                               | `Block` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `home`           | `Any`                          | `Any`          | `10520` |
| `Block Guest to IoT`                                | `Block` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `iot`            | `Any`                          | `Any`          | `10530` |
| `Block Guest to Guest`                              | `Block` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `guest`          | `Any`                          | `Any`          | `10540` |
| `Allow Guest to Internet`                           | `Allow` | `Both`     | `All`    | `guest`          | `Any`                                   | `Any`     | `external`       | `Any`                          | `Any`          | `10599` |

Deferred review note:

- `10330` and `10410` are intentionally not part of the active phase 1 rule set
- keep them out of UniFi until a specific local automation use case requires cross-zone Home Assistant or similar integration flows
- revisit them only when exact hosts, destinations, and ports are known

## Service intent notes

### `home` to `infrastructure`

Phase 1 keeps this narrow.

Allowed by design intent:

- DNS to Rainier
- NAS access only for explicitly approved user-facing file services
- HTTPS to Rainier for explicitly published user-facing reverse-proxied apps such as the printer Web UI
- no broad host reachability
- no admin surfaces

Current design assumption:

- Rainier is not broadly user-facing to `home`
- if additional user-facing infrastructure services are added later, they must be explicitly documented

### `iot` to `infrastructure`

Allowed by design intent:

- DNS to Rainier
- other tightly scoped flows only where required by documented integrations

Phase 1 default:

- DNS only unless a specific integration proves otherwise

### `infrastructure` to `iot`

Allowed by design intent:

- only specific infrastructure hosts may initiate as needed
- primary example is Rainier and Home Assistant related service flows
- Rainier may reach the HP printer at `192.168.40.10:443` so Caddy can publish the printer Web UI
- NAS and UniFi gear do not get broad IoT access

### Admin surfaces

Admin surfaces are `secure` and `vpn` only.

This includes:

- UniFi administration
- Rainier admin interfaces
- NAS administration
- Blackcomb administration
- SSH to infrastructure hosts

Published internal app note:

- `printer.blackridge.shumie.net` is intended to be reachable from `secure` and `home`
- `secure` already inherits this through broad access to `infrastructure`
- `home` requires an explicit `192.168.10.10:443` allow because `home -> infrastructure` remains deny-by-default

Phase 1 allows broad admin access from `secure` and `vpn` rather than service-by-service micro-segmentation.

### Discovery

Discovery is not a default cross-network capability.

Use explicit access first. Add discovery exceptions only for proven Apple-specific needs after testing.

## Ordered phase 1 ruleset

1. Allow `secure` to `infrastructure`.
2. Allow `secure` to `home`.
3. Deny `secure` to `iot` by default.
4. Deny `secure` to `guest`.
5. Allow `secure` to internet.
6. Apply `vpn` policy the same as `secure`.
7. Deny `home` to `secure`.
8. Allow `home` to `iot`.
9. Allow `home` to `infrastructure` on explicitly approved service ports only.
10. Allow `home` to internet.
11. Deny `iot` to `secure`.
12. Deny `iot` to `home` except established and related return traffic.
13. Allow `iot` to `infrastructure` only for approved services such as DNS to Rainier and tightly defined integration flows.
14. Allow `iot` to internet.
15. Deny `infrastructure` to `secure` except established and related return traffic.
16. Deny `infrastructure` to `home` by default.
17. Allow only specific infrastructure hosts, primarily Rainier and Home Assistant related services, to initiate to `iot` as required.
18. Allow `infrastructure` to internet.
19. Deny `guest` to all internal networks.
20. Allow `guest` to internet only.
21. Enable guest client isolation.
22. Restrict admin surfaces to `secure` and `vpn` only.
23. Enforce approved DNS paths: trusted internal networks use Rainier; guest uses gateway DNS with upstream DoH.
24. Do not enable broad cross-network discovery by default.

## Phase alignment

### Phase 1

- implement the zone model
- implement the VLAN and subnet plan
- move devices into target networks
- enforce baseline inter-zone rules
- implement `secure` with `WPA3-Enterprise` and per-device certificate identity
- keep admin access broad within `secure` and `vpn`
- keep technical service definitions minimal where the architectural intent is already clear

### Phase 2

- tighten admin surfaces further if desired
- consider IoT client isolation
- add narrowly scoped discovery exceptions only if testing proves a need
- refine service-level rule definitions where rollout experience justifies them

## Identity posture

`secure` is the only network that requires network-layer device identity in phase 1.

Implementation intent:

- one certificate-backed identity per approved admin device
- no shared Wi-Fi passphrase for the `secure` enclave
- certificate revocation replaces shared secret rotation as the normal offboarding path
- `vpn` remains equivalent in privilege posture to `secure`, but its authentication mechanism is governed separately through UniFi WireGuard peer management

## Implementation guidance

Translate these policy rules into UniFi firewall objects and rules after:

- VLANs are created
- SSIDs are mapped
- Rainier service ports are confirmed
- NAS service ports are confirmed
- migration sequencing is approved

This document is intentionally policy-first, not controller-click-path-first.
