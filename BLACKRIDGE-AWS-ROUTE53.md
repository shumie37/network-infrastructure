# Blackridge AWS and Route 53

## Purpose

This document is the durable source of truth for Blackridge AWS usage that affects network infrastructure.

Initial scope:

- Route 53 authoritative DNS for `shumie.net`
- Dynamic DNS design for the Blackridge VPN endpoint
- IAM policy for the Route 53 updater running on `rainier`

## AWS role in Blackridge

AWS is used for public DNS authority, not as the primary runtime platform for Blackridge services.

Current intended model:

- `Route 53` is authoritative for `shumie.net`
- `rainier` performs approved DNS automation against Route 53
- the UniFi gateway remains the VPN endpoint

## Route 53 hosted zone

- Zone: `shumie.net`
- Hosted zone ID: `Z1XIAIXXJ9KA3O`

## VPN Dynamic DNS target

Blackridge remote admin access should use:

- `vpn.blackridge.shumie.net`

Target design:

- the UniFi gateway terminates the WireGuard VPN
- `rainier` runs a small DDNS updater
- the updater checks the current public WAN IP
- the updater updates `vpn.blackridge.shumie.net` in Route 53

This is preferred over:

- pointing the UniFi gateway itself at `rainier` for DNS just to support DDNS
- running a separate DynDNS provider stack on `rainier`

## Implemented updater

Chosen implementation:

- container image: `crazymax/ddns-route53:2.15.0`
- runtime host: `rainier`
- schedule: every 5 minutes
- record type: `A`
- TTL: `60`

Current live implementation notes:

- the updater runs as a dedicated Docker Compose service on `rainier`
- it reuses the existing local-only AWS Route 53 credentials already present on `rainier` for nginx / Let's Encrypt DNS-01
- it updates only `vpn.blackridge.shumie.net`

Validated on `2026-04-21`:

- updater detected WAN IPv4 `23.252.53.58`
- Route 53 update succeeded
- `vpn.blackridge.shumie.net` resolved publicly to `23.252.53.58`

## AWS IAM policy

This is the current source-of-truth IAM policy for the Route 53 updater path:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Route53ListZones",
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListHostedZonesByName"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Route53ChangeSpecificZone",
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/Z1XIAIXXJ9KA3O"
    },
    {
      "Sid": "Route53GetChange",
      "Effect": "Allow",
      "Action": [
        "route53:GetChange"
      ],
      "Resource": "arn:aws:route53:::change/*"
    }
  ]
}
```

## Credential handling

Keep AWS credentials:

- local-only
- off Git
- scoped to the minimum Route 53 permissions above

Do not commit:

- access keys
- secret access keys
- `.env` files containing AWS secrets
- local credential files

## Durable implementation direction

Planned durable implementation:

1. keep `Route 53` authoritative for `shumie.net`
2. create `vpn.blackridge.shumie.net`
3. run a small updater on `rainier`
4. update the record only when the WAN IP changes
5. use `vpn.blackridge.shumie.net:51820` as the WireGuard client endpoint

Preferred updater host:

- `rainier`

Preferred updater style:

- dedicated Docker container on `rainier`
- `crazymax/ddns-route53`
- not a full DynDNS provider stack

## Validation expectations

When the DDNS updater is implemented, validate:

- `vpn.blackridge.shumie.net` resolves to the current public WAN IP
- the record updates after a WAN IP change
- WireGuard clients can connect using the DNS name
- VPN clients still use `192.168.10.10` as DNS after tunnel establishment

## Notes

- This document is the durable home for future AWS network-infrastructure documentation.
- Host-specific runtime details belong in `rainier-infra`.
- Blackridge-wide AWS design and intent belong here.
