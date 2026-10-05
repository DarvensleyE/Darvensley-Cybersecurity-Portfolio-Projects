# Phase 4: Connecting It All Together

## Objective
Physically and logically connect OPNsense, Proxmox, and the management desktop into one working, isolated lab segment — and verify the isolation actually holds.

## What this covers
- Physical cabling topology
- Resolving a static-IP subnet mismatch on Proxmox
- Setting up dual-homed management access from a desktop
- Verifying connectivity, DHCP, and isolation

## Steps

### Cable the topology
```
Home Router ──┬── Main Desktop (primary NIC, normal internet)
              └── OPNsense WAN (em0)

OPNsense LAN (re0) ── Switch ──┬── Proxmox
                                └── Main Desktop (USB-Ethernet dongle)
```
- Home router's LAN port → OPNsense WAN (`em0`)
- OPNsense LAN (`re0`) → switch
- Proxmox → switch
- USB-Ethernet adapter on the main desktop → switch (second NIC, for lab access)
- Main desktop's primary NIC/WiFi stays on the home router, untouched

### Fix: Proxmox's static IP no longer matched the new subnet
Proxmox had been statically configured on the home network before OPNsense existed. After cabling it into the new lab switch, it was unreachable — its address belonged to a different subnet than OPNsense's LAN.

**Resolution (full detail in [Phase 1](https://github.com/DarvensleyE/Projects/blob/main/Isolated%20SOC%20Home%20Lab/Documents/Proxmox-Setup.md)):**
1. Temporarily matched the management NIC to Proxmox's old subnet to reach it one last time.
2. Reconfigured `vmbr0` to a static address on the new lab subnet (`192.168.50.246/24`, gateway `192.168.50.1`), chosen outside OPNsense's DHCP pool.
3. Reached Proxmox going forward at its new address.

### Set up dual-homed management access
Rather than dedicating a separate machine to managing OPNsense and Proxmox, added a USB-Ethernet adapter to the main desktop:
- **Primary NIC/WiFi** → home router → normal internet, unaffected
- **USB dongle** → lab switch → direct access to OPNsense's GUI and Proxmox's GUI

Verified with `ipconfig` (Windows) that both adapters held addresses in their respective, correct subnets — and that the dongle's address fell within OPNsense's DHCP pool.

### Verify connectivity
- `ping 192.168.50.1` (OPNsense LAN) — confirmed reachable from the dongle
- `ping 192.168.50.246` (Proxmox) — confirmed reachable
- Checked `Interfaces > Diagnostics > ARP Table` in OPNsense to confirm it could see Proxmox even though Proxmox uses a static address rather than a DHCP lease
- Checked `Services > Kea DHCP > Leases` to confirm the dongle had received a proper lease

### Verify isolation
- Confirmed no `Firewall > NAT > Port Forward` rules exist from WAN — nothing in the lab is reachable from the internet
- Confirmed the lab segment has working outbound internet access through OPNsense's NAT (verified via Proxmox reaching package repositories, and the Wazuh VM completing its install)
- Confirmed a static IP alone does not equal internet exposure — exposure only happens via explicit port-forwarding, which was deliberately not configured

## Key takeaway
Getting each component working individually (Proxmox, OPNsense, Wazuh) was the easier part. The real integration work — and the real troubleshooting — happened here, at the seams: cabling, subnetting, and making sure a device configured *before* the firewall existed could correctly rejoin the network *after* it did.

## Result
A working, isolated SOC lab: OPNsense as the segmentation boundary, Proxmox hosting Wazuh behind it, and reliable management access from a single desktop — with zero impact on the home network throughout the entire build.
