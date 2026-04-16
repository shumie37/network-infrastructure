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
| `vpn` | same as `secure` | same as `secure` | UniFi Teleport full-tunnel remote access |
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

## Service intent notes

### `home` to `infrastructure`

Phase 1 keeps this narrow.

Allowed by design intent:

- DNS to Rainier
- NAS access only for explicitly approved user-facing file services
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
- NAS and UniFi gear do not get broad IoT access

### Admin surfaces

Admin surfaces are `secure` and `vpn` only.

This includes:

- UniFi administration
- Rainier admin interfaces
- NAS administration
- Blackcomb administration
- SSH to infrastructure hosts

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
- `vpn` remains equivalent in privilege posture to `secure`, but its authentication mechanism is governed separately by the remote-access platform

## Implementation guidance

Translate these policy rules into UniFi firewall objects and rules after:

- VLANs are created
- SSIDs are mapped
- Rainier service ports are confirmed
- NAS service ports are confirmed
- migration sequencing is approved

This document is intentionally policy-first, not controller-click-path-first.
