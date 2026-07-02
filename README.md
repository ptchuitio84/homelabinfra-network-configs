# homelabinfra-network-configs

Network device configuration files for the NNT homelab — running configs and pre-change backups for the three-switch VRRP stack. These files are both the audit trail and the recovery point. When a change is enforced via Ansible, a timestamped backup is committed here before any change is applied.

---

## Device Inventory

| File | Device | Model | Role | Mgmt IP |
|---|---|---|---|---|
| `cisco_catalyst_3750g_sw-core-01.txt` | sw-core-01 | Cisco Catalyst 3750G | L3 VRRP Backup (priority 90) | 10.0.100.251 |
| `arista_7050sx_sw-core-02.eos` | sw-core-02 | Arista 7050SX-64 | L3 VRRP Master (priority 110) | 10.0.100.252 |
| `arista_7048t_sw-core-03.eos` | sw-core-03 | Arista 7048T-A | L3 VRRP Secondary (priority 100) | 10.0.100.253 |

**Note on filename vs hostname:** The Cisco file is named `cisco_catalyst_3750g_sw-core-01.txt` but the running config hostname is `sw-core-01`. The file naming convention reflects the physical asset tag; the hostname reflects the network role.

---

## Architecture

```
                    WAN (Meraki MX64)
                   /        |         \
                  /         |          \
    sw-core-01 (3750G)  sw-core-02 (7050SX)  sw-core-03 (7048T)
    VRRP Backup (90)    VRRP Master (110)    VRRP Secondary (100)
    Uplink: VLAN 501    Uplink: VLAN 502     Uplink: VLAN 503, VLAN 100
    DHCP Server                │
         │                     │
         └─────────────────────┘
              Inter-switch trunks (STP active/blocked)
```

**L3 redundancy:** VRRP on each SVI. Virtual IP is the gateway for each VLAN. If the master fails, secondary takes over within 3 seconds. No routing change needed on hosts.

**L2 redundancy:** Spanning Tree (STP) prevents loops. sw-core-02 (7050SX) is STP root for all VLANs. Single path active at a time; blocked ports activate only on failure.

**DHCP:** The Cisco 3750G (`sw-core-01`) is the DHCP server for all subnets. All SVIs on the Arista switches relay DHCP requests to `10.0.100.251`. One DHCP server, one place to manage leases.

---

## VRRP Priority Design

| Switch | Priority | State | Preempt |
|---|---|---|---|
| sw-core-02 (7050SX) | 110 | Master | Yes |
| sw-core-03 (7048T) | 100 | Secondary | Yes |
| sw-core-01 (3750G) | 90 | Backup | Yes |

Preemption is enabled — if the master comes back online after a failure, it reclaims Master role automatically. The 7050SX is the highest-priority node because it has the fastest CPU and lowest forwarding latency.

**IP scheme for VRRP SVIs:**
- Large subnets (192.168.0.0/23): `.1` = VIP, `.251/.252/.253` = switch SVIs
- Small subnets (10.100.100.0/24): `.1` = VIP, `.2/.3/.4` = switch SVIs

---

## Config File Format

**Cisco IOS (`.txt`):**
Standard IOS running-config format. Lines beginning with `!` are comments. Applied via `cisco.ios.ios_config src:` with `match: none` in Ansible (IOS reformats lines on write — text-match comparison doesn't work idempotently).

**Arista EOS (`.eos`):**
EOS running-config format. Applied via `arista.eos.eos_config src:` in Ansible. Syntax is similar to IOS with EOS-specific extensions.

---

## Backup Process

Pre-enforcement backups are stored in `backups/`. Naming convention:

```
backups/<hostname>_pre-enforce_<YYYY-MM-DD>_<HHMM>.txt
```

Example: `backups/sw-core-02.homelab.local_pre-enforce_2026-04-12_1430.txt`

**When backups are created:**
The Ansible enforcement pipelines (`lab-jkn-net-enforce-sw-core-01/02`) fetch the running config before applying any changes and commit it here. This gives a point-in-time snapshot of exactly what was on the device before the enforcement run.

**Using a backup for rollback:**

```bash
# 1. Identify the backup
ls backups/sw-core-02*

# 2. Copy to the main config file
cp backups/sw-core-02.homelab.local_pre-enforce_2026-04-12_1430.txt \
   arista_7050sx_sw-core-02.eos

# 3. Commit and push
git commit -am "rollback: restore sw-core-02 to 2026-04-12 pre-enforce state"
git push

# 4. Trigger the enforcement pipeline — it will push the restored config
```

---

## How Configs Are Enforced

Enforcement is Ansible-driven, Jenkins-triggered:

| Pipeline | Device | Playbook |
|---|---|---|
| `lab-jkn-net-enforce-sw-core-01` | sw-core-01 (Cisco) | `enforce_baseline_sw-core-01.yml` |
| `lab-jkn-net-enforce-sw-core-02` | sw-core-02 (Arista) | `enforce_baseline_sw-core-02.yml` |

**Pipeline flow:**
1. Dry-run: shows what would change (no device modification)
2. 24-hour approval gate (manual review required)
3. Enforce: pushes config to device

**Cisco note:** SSH to sw-core-01 must originate from `ansible-ctl-01` (192.168.1.31). The Cisco 3750G runs IOS 12.2 — older KEX algorithms that modern OpenSSH on macOS refuses. The Ansible control node has legacy cipher support configured.

---

## Adding or Modifying a Config

1. Edit the relevant config file (`*.txt` or `*.eos`)
2. Commit and push
3. Run the enforcement pipeline for the affected device (dry-run first)
4. Review the dry-run diff — confirm only the intended lines changed
5. Approve the gate and enforce

For changes that affect all three switches (e.g., a new VLAN), update all three files before pushing.

---

## Known Issues

- **Cisco 3750G EOL** — Hardware and IOS 12.2 are both end-of-life. No security patches. Replacement candidate when Arista inventory is complete.
- **Filename vs hostname mismatch** — Cisco file is `sw-core-01.txt` but device hostname is `sw-core-01`. Not renaming (git history tracking); mismatch documented here.
- **No automated config pull** — Configs are updated manually or via the enforcement pipeline. No scheduled "pull from device and commit" automation exists. Drift between device and repo is possible between enforcement runs.
- **PoE TPLink dual-home** — STP bpdufilter is OFF on TPLink-facing ports. If STP is misconfigured, this can create a loop. Validate STP state before any trunk changes on those ports.
