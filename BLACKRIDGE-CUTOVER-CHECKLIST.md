# Blackridge Cutover Checklist

## Purpose

This checklist is the execution companion to the Blackridge architecture, access policy, and UniFi implementation guide.

Use it during the actual rebuild to minimize downtime and avoid skipping validation steps.

## Day 0: Before Cutover

### 1. Freeze and backup

- [ ] Pause non-essential changes to UniFi, `rainier`, `blackcomb`, and NAS.
- [ ] Export a fresh UniFi Network backup.
- [ ] Save screenshots of:
  - [ ] current networks
  - [ ] current Wi-Fi settings
  - [ ] current Zone-Based Firewall / Policy Table
  - [ ] current switch port profiles
  - [ ] current DHCP reservations
- [ ] Confirm current admin login works for UniFi.

### 2. Confirm source-of-truth values

- [ ] `infrastructure` = `192.168.10.0/24`
- [ ] `secure` = `192.168.20.0/24`
- [ ] `home` = `192.168.30.0/24`
- [ ] `iot` = `192.168.40.0/24`
- [ ] `hotspot` = `192.168.50.0/24`
- [ ] `vpn` = `192.168.60.0/24`
- [ ] `rainier` target IP = `192.168.10.10`
- [ ] `blackcomb` target IP = `192.168.10.20`
- [ ] `NAS` target IP = `192.168.10.30`

### 3. Confirm admin safety paths

- [ ] One primary admin Mac available on Ethernet if possible.
- [ ] One secondary admin device still able to use the old network.
- [ ] Local recovery path known for gateway, switch, and APs.
- [ ] Current static IP details recorded for gateway, switch, APs, `rainier`, `blackcomb`, and NAS.

### 4. Confirm `secure` dependencies are already working

- [ ] FreeRADIUS is healthy on `rainier`.
- [ ] Internal CA is healthy on `rainier`.
- [ ] At least one admin Apple device can join `Blackridge`.
- [ ] RADIUS and DNS are reachable from the expected networks.
- [ ] Apple Wi-Fi profiles are ready for the admin devices needed during cutover.

### 5. Confirm VPN readiness

- [ ] UniFi gateway supports the intended WireGuard workflow.
- [ ] Public IP or upstream UDP port-forward requirement for `51820` is understood.
- [ ] One test WireGuard peer can be created when needed.

## Day 1: Build in Parallel

### 1. Create networks

- [ ] Leave the current legacy LAN in place.
- [ ] Create or confirm `infrastructure` VLAN `1`.
- [ ] Create or confirm `secure` VLAN `2`.
- [ ] Create or confirm `home` VLAN `3`.
- [ ] Create or confirm `iot` VLAN `4`.
- [ ] Create or confirm `hotspot` VLAN `5`.
- [ ] Create or confirm `vpn` network `6` if UniFi requires separate definition.
- [ ] Enable DHCP on all required networks.
- [ ] Set trusted-network DNS to `rainier`.
- [ ] Set `hotspot` DNS to gateway behavior.
- [ ] Do not remove the legacy LAN.

### 2. Create SSIDs

- [ ] Leave the old Wi-Fi SSID(s) active.
- [ ] Create `Blackridge` -> `secure`
- [ ] Create `ShuMK` -> `home`
- [ ] Create `ShuMK IoT` -> `iot`
- [ ] Create `ShuMK Guest` -> `hotspot`
- [ ] `Blackridge` uses `WPA3-Enterprise` with external RADIUS.
- [ ] `ShuMK` uses `WPA3-Personal`.
- [ ] `ShuMK IoT` uses `WPA2/WPA3` transition mode.
- [ ] `ShuMK Guest` uses hotspot / guest posture per UniFi capabilities.
- [ ] Leave tuning conservative:
  - [ ] band steering off
  - [ ] fast roaming off
  - [ ] minimum RSSI off

### 3. Validate `secure`

- [ ] Join one admin Mac to `Blackridge`.
- [ ] Confirm it gets `192.168.20.x`.
- [ ] Confirm DNS resolves through `rainier`.
- [ ] Confirm UniFi admin is reachable.
- [ ] Confirm `rainier`, `blackcomb`, and NAS are reachable from the admin device.
- [ ] Only after success, join the second admin device.

## Day 1: Build Port Profiles and Remote Access

### 4. Create switch port profiles

- [ ] Create `Infrastructure Native`
- [ ] Create `Home Native`
- [ ] Create `IoT Native`
- [ ] Create `Hotspot Native`
- [ ] Create AP trunk profile
- [ ] Confirm AP trunk intent:
  - [ ] native `infrastructure`
  - [ ] tagged `secure`, `home`, `iot`, `hotspot`

### 5. Stand up WireGuard

- [ ] Enable the UniFi WireGuard server.
- [ ] Bind or align it with the `vpn` model.
- [ ] Create one test admin peer.
- [ ] Import it into a standard WireGuard client.
- [ ] Confirm:
  - [ ] client gets `192.168.60.x`
  - [ ] full-tunnel behavior works
  - [ ] DNS uses `rainier`
  - [ ] admin surfaces are reachable

## Day 1: Move Infrastructure Carefully

### 6. Move AP and switch management

- [ ] Change one AP uplink only.
- [ ] Confirm the AP reprovisions and returns.
- [ ] Repeat for the next AP.
- [ ] Only after APs are healthy, move switch management.
- [ ] Confirm gateway, switch, and APs are reachable from `secure`.

### 7. Move fixed infrastructure hosts

Recommended order:

1. NAS
2. `blackcomb`
3. `rainier`

For each host:

- [ ] Update the switch port profile.
- [ ] Move the host to its target `192.168.10.x` address.
- [ ] Confirm reachability from `secure`.
- [ ] Confirm any service-specific health checks pass.

Before moving `rainier`:

- [ ] DNS is ready for trusted networks.
- [ ] RADIUS is ready for UniFi infrastructure.
- [ ] `rainier` firewall rules allow DNS and RADIUS from the new networks.

## Day 1: Apply Firewall / Policy

### 8. Apply zone-based rules in stages

- [ ] Confirm `hotspot` is represented as `Hotspot` in UniFi policy views.
- [ ] Add or verify `hotspot` isolation rules.
- [ ] Add or verify trusted DNS rules to `192.168.10.10`.
- [ ] Add or verify `home` -> NAS rules:
  - [ ] `TCP 445` to `192.168.10.30`
  - [ ] `UDP 9` to `192.168.10.30`
- [ ] Add or verify `secure` broad admin-path rules.
- [ ] Add or verify `vpn` rules equivalent to `secure`.
- [ ] Add or verify broad deny rules protecting `secure`.
- [ ] Leave these deferred:
  - [ ] `IoT -> Approved Infrastructure Integrations`
  - [ ] `Infrastructure -> Approved IoT Services`

### 9. Validate policy behavior

- [ ] `home` can reach internet
- [ ] `home` can reach NAS only on approved ports
- [ ] `home` cannot reach `secure`
- [ ] `iot` can reach DNS and internet
- [ ] `iot` cannot reach `secure`
- [ ] `hotspot` has internet only
- [ ] `hotspot` cannot reach internal zones
- [ ] `secure` can administer infrastructure
- [ ] `vpn` has the same intended admin posture as `secure`

## Day 1: Migrate Clients

### 10. Migrate `home`

- [ ] Move a small test set first.
- [ ] Confirm DHCP, DNS, internet, and NAS access.
- [ ] Move the rest in batches.

### 11. Migrate `iot`

- [ ] Move a low-risk test set first.
- [ ] Confirm DHCP, DNS, and internet.
- [ ] Move the rest in batches.
- [ ] Leave fragile devices for the end.

### 12. Validate `hotspot`

- [ ] Confirm captive portal behavior if enabled.
- [ ] Confirm isolation.
- [ ] Confirm internet only.

## Finalization

### 13. Retire the legacy network

Only do this after the new environment is stable.

- [ ] Confirm no required device remains on the old LAN or SSID.
- [ ] Disable the old SSID.
- [ ] Remove the legacy LAN only after a quiet observation window.
- [ ] Remove temporary exceptions and stale DHCP reservations.

### 14. Post-cutover review

- [ ] Record any real-world deviations from the docs.
- [ ] Decide whether any discovery exceptions are truly needed.
- [ ] Decide whether any IoT automation rules need to be introduced later.
- [ ] Re-export UniFi backup after stabilization.

## Stop Conditions

If any of these occur, stop and restore the last known-good path:

- [ ] gateway unreachable
- [ ] switch unreachable
- [ ] AP does not return after profile change
- [ ] `secure` admin access lost
- [ ] `rainier` DNS unavailable
- [ ] RADIUS unstable
- [ ] unexpected cross-zone exposure appears
