# Architecture

## Topology

```
Internet ── Home Router ──┬── Gaming/Main Desktop (normal internet, unaffected)
                           │
                           └── OPNsense WAN (em0)
                                     │
                                OPNsense LAN (re0)
                                     │
                                  Switch
                                ┌────┴────┐
                            Proxmox    USB-Ethernet dongle
                            (Wazuh)    (on main desktop, for GUI access)
```

The home router keeps handling the regular network exactly as before. OPNsense sits off to the side as its own router/firewall for a completely separate lab segment. Outbound traffic from the lab goes through OPNsense's NAT; nothing from the internet, and nothing from the home network, can reach the lab unless a rule explicitly allows it.

## Hardware

| Item | Purpose | Notes |
|---|---|---|
| Mini PC with 2 NICs | Runs OPNsense | Intel NICs generally have the best FreeBSD driver support |
| Proxmox server | Hosts Wazuh and lab VMs | Existing or dedicated machine |
| Managed/smart switch (e.g. TP-Link TL-SG108E) | Connects OPNsense LAN to lab devices | VLAN-capable, for future segmentation |
| USB-to-Ethernet adapter | Second NIC for the main desktop | Lets you reach the lab GUI without losing normal internet |

**Minimum specs — OPNsense:** 4GB RAM, 16–32GB SSD.
**Minimum specs — Wazuh VM:** 4GB+ RAM, 50GB+ disk (the indexer component is disk-intensive; under-provisioning causes failed services).

## Network design

| Zone | Subnet | Role |
|---|---|---|
| Home network | `192.168.1.x` (example) | Unchanged, handled by the existing home router |
| Lab network | `192.168.50.0/24` | OPNsense LAN, hands out `.10`–`.245` via DHCP |

The lab subnet is deliberately **distinct** from the home router's range to avoid address collisions. `.1`–`.9` are reserved for the gateway and any statically-addressed devices (e.g. Proxmox at `.246`).

## Why two physical NICs

OPNsense needs a genuinely separate WAN and LAN interface to create an isolation boundary — one facing the home router, one facing the lab switch. This is what makes the separation physical rather than just a firewall-rule convention.

## Why a dual-homed management desktop

Rather than dedicating a separate machine to managing OPNsense and Proxmox, the main desktop carries a second network leg (a USB-Ethernet dongle) that plugs into the lab switch. This keeps the desktop on the home network for normal use via its primary NIC, while giving it direct access to the lab subnet for GUI administration via the dongle — no cable-swapping required.

## Design decisions and tradeoffs

- **OPNsense is not the home router.** It sits beside the existing router rather than replacing it, keeping the blast radius of any mistake limited to the lab segment. Becoming the primary router/firewall is a deliberate future step, once more comfortable with the rule engine.
- **NAT + default-deny inbound does most of the isolation work.** No port-forwarding rules exist from WAN to the lab, so nothing in the lab is reachable from the internet.
- **VLANs vs. physical separation.** The current design uses physical separation (two NICs, one switch dedicated to the lab) rather than VLANs on a shared switch. This is simpler and arguably safer while learning, at the cost of needing dedicated switch hardware. VLAN segmentation is a planned next step for merging lab and home traffic onto the same physical firewall.

## Planned evolution

Future state, once VLANs are introduced:

```
                         OPNsense firewall
                         (rule: block lab → home)
                        ┌───────┴───────┐
                   VLAN 10, Home   VLAN 20, Lab
                   (trusted)       (isolated)
```

Same physical switch and firewall, but home and lab traffic travel on separate tagged VLANs, with an explicit firewall rule preventing the lab VLAN from reaching the home VLAN.
