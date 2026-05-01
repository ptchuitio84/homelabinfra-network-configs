# homelabinfra-network-configs

Network device configuration files for the NNT homelab switch stack.

## Devices
| Device | Model | Role | IP | EOS/IOS |
|--------|-------|------|----|---------|
| swplnet251 | Cisco Catalyst 3750G | VRRP backup (priority 90) | 10.100.100.251 | IOS 12.x |
| swplnet252 | Arista 7050SX-64 | VRRP master (priority 110) | 10.100.100.252 | EOS 4.28.12M |
| swplnet253 | Arista 7048T-A | VRRP secondary (priority 100) | 10.100.100.253 | EOS 4.12.x |

## Meraki
- Device: MX64 | Serial: Q2KN-NDUF-T88L | WAN1: 184.57.35.211
- Org ID: **1679044** (numeric API ID — NOT the URL slug LiZFhb)
- Always query `/api/v1/organizations` to get numeric ID — never use URL slug

## Key Gotchas
- **Cisco SSH:** Must run from ans001 (hmvlapans001, 10.10.1.31). Mac OpenSSH is too new for IOS 12.2 KEX algorithms.
- **ios_config idempotency:** Use `match: none`. IOS reformats config lines on write — text comparison always shows drift even with no real changes.
- **Arista eAPI:** Preferred transport over SSH for EOS automation. Enable with `management api http-commands / no shutdown`.

## Ansible Collections
- Cisco IOS: `cisco.ios`
- Arista EOS: `arista.eos`
- Meraki: `cisco.meraki` or `uri` module against REST API
