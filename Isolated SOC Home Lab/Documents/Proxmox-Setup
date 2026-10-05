# Phase 1: Proxmox Setup

## Objective
Stand up a Proxmox VE hypervisor on dedicated hardware to host the lab's virtual machines, starting with the Wazuh SIEM VM.

## What this covers
- Base Proxmox VE installation
- Initial network bridge configuration
- Hardware/sizing considerations for the VMs it would host

## Steps

### Install Proxmox VE
Installed Proxmox VE on a dedicated machine, separate from the OPNsense firewall hardware. Standard installer, default partitioning.

### Initial network bridge
Proxmox creates a default bridge, `vmbr0`, during installation, bound to the host's physical NIC. At this stage, `vmbr0` was configured on the home network, since OPNsense didn't exist yet in the topology — this gets revisited once the lab network is introduced.

### Planning VM sizing
Before deploying Wazuh (see [Phase 3](./03-wazuh-siem-setup.md)), sized the VM with **50GB+ disk and 4GB+ RAM** as a baseline — Wazuh's indexer component (OpenSearch-based) is disk-intensive, and under-provisioning here causes real failures later.

## Key takeaway
Proxmox itself was the easy part — a standard hypervisor install. The real complexity came later, when its networking had to be re-pointed at a new, isolated subnet once OPNsense was introduced (see [Phase 4](./04-connecting-it-together.md)).

## Next
→ [Phase 2: OPNsense Setup](./02-opnsense-setup.md)
