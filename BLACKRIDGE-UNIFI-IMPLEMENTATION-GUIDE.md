# Blackridge UniFi Implementation Guide

## Purpose

This document defines the recommended step-by-step UniFi implementation sequence for rebuilding the Blackridge network with minimal downtime.

It assumes:

- UniFi is the control plane for gateway, switching, APs, Wi-Fi, DHCP, VLANs, firewall policy, and Teleport
- configuration is performed primarily through the UniFi WebUI
- the current network is still serving live devices
- the target architecture is defined in:
  - `BLACKRIDGE-NETWORK-ARCHITECTURE.md`
  - `BLACKRIDGE-ACCESS-POLICY.md`

UniFi feature alignment for this guide:

- use UniFi `Networks` for VLAN-backed routed networks
- use UniFi `WiFi` for SSID mapping
- use UniFi `Profiles` for switch-port profiles
- use UniFi external RADIUS server configuration for `ShuMK Secure`
- use UniFi Zone-Based Firewall / Policy Engine constructs where available instead of older per-interface firewall rule style
- do not use UniFi gateway-hosted DNS record features as the authoritative DNS model for trusted networks, because trusted clients are intended to use `rainier` directly

Primary objective:

- keep the current flat network working until each replacement network function is ready

Secondary objective:

- avoid management lockout while moving infrastructure into the new `infrastructure` VLAN

## Core rollout principles

1. Build the new network in parallel before moving clients.
2. Keep at least one known-good admin path alive at all times.
3. Move admin devices first, infrastructure second, general clients third, low-trust clients last.
4. Change one control plane at a time:
   - networks and VLANs
   - switch port profiles
   - firewall policy
   - Wi-Fi SSIDs
   - infrastructure host addressing
5. Do not retire the legacy LAN or legacy SSID until the replacement path is validated.

## Required prep before touching UniFi

### Admin safety requirements

- have one primary admin Mac on Ethernet if possible
- have one second admin device available on the old network
- know the local recovery path for the UniFi gateway and switch
- have current UniFi admin credentials verified
- confirm you can reach:
  - UniFi controller
  - gateway
  - `rainier`
  - NAS

### Backups and exports

Before making any changes in UniFi:

1. Export or download the UniFi Network backup.
2. Take screenshots or notes of:
   - current networks
   - current Wi-Fi settings
   - current firewall rules
   - current switch port profiles
   - current DHCP reservations
3. Record the current working subnet, gateway, and DNS path.
4. Save current static IP settings for:
   - `rainier`
   - `blackcomb`
   - NAS
   - UniFi gateway
   - UniFi switch
   - UniFi APs
5. Confirm the UniFi Network version and UI generation in use.
6. Confirm whether Zone-Based Firewall is already enabled, available for new policy creation, or still pending migration in the current controller.

### External dependencies to prepare first

Before cutover day, prepare these while the current network remains untouched:

1. Stand up and validate FreeRADIUS on `rainier`.
2. Stand up and validate the internal CA on `rainier`.
3. Generate and test at least one working `secure` Apple Wi-Fi profile.
4. Confirm `rainier` can serve DNS for the future trusted networks.
5. Confirm `rainier` firewall rules will allow:
   - DNS from trusted VLANs
   - RADIUS from UniFi infrastructure
6. Confirm the `ShuMK Secure` RADIUS path using the current UniFi UI model:
   - external RADIUS server object
   - WPA3-Enterprise SSID binding
   - successful join from one admin device

Do not start the UniFi rebuild until the `secure` authentication stack is already working.

## Recommended implementation order

## Phase 0: Freeze and naming

Goal:

- avoid avoidable churn during the rebuild

Steps:

1. Freeze non-essential changes to `rainier`, `blackcomb`, NAS, and UniFi.
2. Confirm final names for:
   - networks
   - VLAN IDs
   - SSIDs
   - port profiles
3. Confirm final addressing plan:
   - `secure`: `192.168.10.0/24`
   - `infrastructure`: `192.168.20.0/24`
   - `home`: `192.168.30.0/24`
   - `iot`: `192.168.40.0/24`
   - `guest`: `192.168.50.0/24`
   - `vpn`: `192.168.60.0/24`

Validation checkpoint:

- nothing has changed in live traffic yet

## Phase 1: Build the new UniFi networks in parallel

Goal:

- create the target networks and DHCP scopes without cutting over clients

UniFi WebUI area:

- `Settings` -> `Networks`

Steps:

1. Leave the current default or legacy LAN in place.
2. Create the new routed networks:
   - `secure` / VLAN `10`
   - `infrastructure` / VLAN `20`
   - `home` / VLAN `30`
   - `iot` / VLAN `40`
   - `guest` / VLAN `50`
3. Create the `vpn` remote-access network definition if UniFi requires it separately.
4. Enable DHCP for each network per the target design.
5. Set DNS server behavior:
   - trusted networks point to `rainier`
   - `guest` uses gateway DNS
6. If the controller exposes mDNS or discovery helpers per network, leave them off by default for phase 1 unless a later validation step proves they are required.
7. Do not create UniFi gateway-local DNS records as a substitute for the trusted-network DNS model.
8. Do not remove or repurpose the current live LAN yet.

Validation checkpoint:

- UniFi shows all new networks created
- no production host has been moved yet
- legacy network still works unchanged

## Phase 2: Create the target SSIDs before migrating clients

Goal:

- make the new wireless destinations available while the old SSID remains live

UniFi WebUI area:

- `Settings` -> `WiFi`

Steps:

1. Leave the old Wi-Fi network active.
2. Create:
   - `ShuMK Secure`
   - `ShuMK`
   - `ShuMK IoT`
   - `ShuMK Guest`
3. Bind each SSID to its new network:
   - `ShuMK Secure` -> `secure`
   - `ShuMK` -> `home`
   - `ShuMK IoT` -> `iot`
   - `ShuMK Guest` -> `guest`
4. Configure authentication:
   - `ShuMK Secure`: `WPA3-Enterprise` with external RADIUS
   - `ShuMK`: `WPA3-Personal`
   - `ShuMK IoT`: `WPA2/WPA3` transition
   - `ShuMK Guest`: guest posture per controller capabilities
5. Keep guest portal and hotspot complexity minimal until basic segmentation is validated.
6. Keep tuning conservative:
   - band steering off
   - fast roaming off
   - minimum RSSI off

Validation checkpoint:

- new SSIDs are broadcast
- old SSID still exists
- no broad client migration has happened yet

## Phase 3: Build and validate `secure` first

Goal:

- establish the new management enclave before touching infrastructure addressing

Why first:

- once `secure` works, it becomes the clean admin path for the rest of the migration

Steps:

1. In UniFi, configure the external RADIUS profile for `ShuMK Secure`.
2. Confirm the RADIUS server points at `rainier`.
3. Install the Apple `.mobileconfig` on one admin Mac.
4. Join `ShuMK Secure`.
5. Verify:
   - the device gets an address in `192.168.10.0/24`
   - DNS uses `rainier`
   - UniFi admin is reachable
   - `rainier`, NAS, and `blackcomb` admin surfaces are reachable
6. Confirm the UniFi controller still shows the RADIUS server object and SSID binding exactly as intended after provisioning.
7. Repeat for the second admin device only after the first is proven.

Validation checkpoint:

- at least one admin device has stable access on `secure`
- old admin path still exists as fallback

## Phase 4: Create port profiles before moving infrastructure

Goal:

- avoid ad hoc switch-port edits during live host moves

UniFi WebUI area:

- `Settings` -> `Profiles` -> `Switch Ports`

Create profiles for at least:

- `Infrastructure Native`
- `Home Native`
- `IoT Native`
- `Guest Native`
- trunk profile for AP uplinks

Expected intent:

- AP uplinks should carry:
  - native `infrastructure`
  - tagged `secure`, `home`, `iot`, `guest`
- fixed infrastructure hosts should land on `infrastructure`
- user wired ports should land on the correct access VLAN

Validation checkpoint:

- all needed profiles exist before any port is reassigned

## Phase 5: Move network gear management to `infrastructure`

Goal:

- get the UniFi control plane into its final management network with minimal lockout risk

Highest-risk step:

- changing management VLANs for switch and APs can temporarily disconnect them

Safe sequence:

1. Change one AP uplink or one switch uplink only after confirming the upstream trunk profile is correct.
2. Re-provision one AP first, not all APs at once.
3. Confirm the AP comes back online in UniFi.
4. Repeat for the second AP.
5. Only after AP uplinks are stable, move switch management into `infrastructure`.
6. Finally confirm gateway, switch, and APs are all reachable from `secure`.

Important:

- do not batch-change all AP or switch uplink ports simultaneously
- if one device fails to return, stop and fix that device before continuing

Validation checkpoint:

- gateway, switch, and APs all show healthy
- infrastructure management addresses align with `192.168.20.0/24`
- `secure` clients can still administer UniFi

## Phase 6: Move fixed infrastructure hosts

Goal:

- move `rainier`, `blackcomb`, and NAS into `infrastructure` after the control plane is stable

Recommended order:

1. NAS
2. `blackcomb`
3. `rainier`

Why `rainier` last:

- it is your DNS and RADIUS dependency
- moving it too early increases cutover risk for everything else

Steps per host:

1. Update the switch port profile for the host’s wired port.
2. Move the host to its target `192.168.20.x` address.
3. Confirm:
   - ping from `secure`
   - DNS reachability if relevant
   - admin interface reachability
4. Proceed to the next host only after the previous one is fully healthy.

Special caution for `rainier`:

Before moving `rainier`, confirm:

- DNS configuration is ready for trusted VLANs
- RADIUS is reachable from UniFi infrastructure
- firewall rules on `rainier` allow DNS and RADIUS from the new source networks

Validation checkpoint:

- all infrastructure hosts are reachable on `192.168.20.0/24`
- `secure` still has stable admin access

## Phase 7: Apply firewall policy in stages

Goal:

- enforce segmentation without accidentally cutting off required dependencies

UniFi WebUI area:

- prefer UniFi `Zones`, `Policy Table`, or current Zone-Based Firewall UI when available
- if the controller is still on the older model, use the closest `Traffic & Firewall Rules` workflow without changing the logical order below

Policy model expectation:

- define the architectural zones clearly before writing inter-zone policy
- keep `Gateway` handling conservative because overblocking there can break DHCP, DNS, and controller reachability
- treat UniFi automatic migration from legacy firewall rules as something to review carefully, not blindly trust during a live rebuild

Recommended order:

1. Define or verify the zone model that matches:
   - `secure`
   - `infrastructure`
   - `home`
   - `iot`
   - `guest`
   - `vpn`
   - `external`
2. Add rules for `guest` isolation first.
3. Add explicit allows for trusted DNS to `rainier`.
4. Add `home` -> `infrastructure` narrow allows.
5. Add `iot` -> `infrastructure` narrow allows.
6. Add deny rules protecting `secure`.
7. Add `infrastructure` -> `home` and `infrastructure` -> `iot` restrictions last.

Best practice:

- introduce rules with logging where practical
- after each rule group, validate from a test client before adding the next group
- avoid broad `Gateway` denies early in the rollout

Validation checkpoint:

- DNS works from trusted networks
- admin surfaces are reachable only from `secure` and `vpn`
- guest has internet only

## Phase 8: Migrate general client networks

Goal:

- move normal users and devices only after infrastructure and policy are stable

Recommended order:

1. `home`
2. `iot`
3. `guest`

### `home`

Steps:

1. Join a few representative devices to `ShuMK`.
2. Confirm:
   - correct DHCP scope
   - internet access
   - DNS to `rainier`
   - allowed user-facing services still work
   - no admin surfaces are exposed unintentionally
3. Move the rest of `home` devices in batches.

### `iot`

Steps:

1. Move a small number of low-risk devices first.
2. Confirm internet, DNS, and required integrations.
3. Move the rest in batches.
4. Leave unusual or fragile IoT devices for the end.
5. Only add discovery or multicast exceptions after proving a real need.

### `guest`

Steps:

1. Enable captive portal and isolation only after validation.
2. Confirm no internal access.

Validation checkpoint:

- new SSIDs work for their target populations
- old flat-network SSID is still available as the rollback path until final cutover

## Phase 9: Retire the legacy flat network

Goal:

- remove fallback only after the replacement is proven

Do this only after:

- `secure` is stable
- infrastructure is fully moved
- trusted DNS path works
- firewall rules are validated
- representative `home` and `iot` devices are working

Steps:

1. Confirm no required device remains on the old network.
2. Disable the legacy SSID.
3. Remove the legacy flat LAN only after a quiet observation window.
4. Clean up stale DHCP reservations, old Wi-Fi entries, and temporary firewall exceptions.

## Rollback rules

If a phase fails:

1. Stop after the first failed dependency.
2. Restore the last known-good path before attempting another change.
3. Prefer re-enabling the old SSID or old port profile over improvising a new workaround.
4. Never continue client migration while admin access is degraded.

Immediate rollback triggers:

- UniFi gateway or switch unreachable
- APs fail to return after profile change
- `rainier` DNS unavailable to trusted networks
- `secure` admin access lost
- RADIUS unstable for the `secure` SSID

## Practical cutover checklist

### Day before

- validate UniFi backup
- validate FreeRADIUS and CA
- validate one `secure` profile on one admin device
- prepare all VLANs, SSIDs, and port profiles in documentation

### Cutover start

- keep old LAN and old SSID alive
- build new networks
- build new SSIDs
- validate `secure`

### Mid cutover

- move APs and switch management carefully
- move infrastructure hosts
- apply firewall policy in stages

### Late cutover

- migrate `home`
- migrate `iot`
- validate `guest`

### Finalization

- remove old SSID
- remove old flat LAN
- clean temporary exceptions
- update documentation with any real-world deviations
