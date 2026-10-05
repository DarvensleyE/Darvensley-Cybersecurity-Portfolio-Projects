# Phase 2: OPNsense Setup

## Objective
Install and configure OPNsense as a dedicated firewall, creating a physically isolated network segment for the lab — separate from the home network.

## What this covers
- OPNsense installation
- Interface assignment (WAN/LAN)
- WAN and LAN configuration
- DHCP (Kea) setup for the lab subnet
- Setup wizard, admin hardening, and default firewall rule review

## Steps

### Install OPNsense
1. Flashed the OPNsense installer image to a USB stick.
2. Booted a dedicated mini PC (with two NICs — Intel `em0`, Realtek `re0`) from the USB stick.
3. Installed OPNsense to the internal drive, replacing any existing OS.

### Assign interfaces
From the console menu:
- `em0` (Intel) → **WAN** — plugged into the home router
- `re0` (Realtek) → **LAN** — plugged into the lab switch

> Used manual assignment rather than auto-detect — auto-detect requires unplugging and replugging the cable *while the prompt is active* to register a link-state change; a cable already connected beforehand won't trigger it.

### Configure WAN
In `Interfaces > WAN`:
- **IPv4 Configuration Type:** DHCP (downstream of the home router, which assigns the address)
- Unchecked "Block private networks" (otherwise OPNsense rejects the private IP the home router hands it)

### Configure LAN
In `Interfaces > LAN`:
- **IPv4 Configuration Type:** Static
- **IPv4 Address:** `192.168.50.1/24` — deliberately distinct from the home network's range (`192.168.1.x`) to avoid address collisions

### Enable DHCP (Kea)
In `Services > Kea DHCPv4 > [LAN]`:
- Enabled the DHCP server
- Pool: `192.168.50.10`–`192.168.50.245`
- Reserved `.1`–`.9` for the gateway and static addresses
- **Deployment type:** Standalone (no HA failover partner)

### Run the setup wizard
`System > Setup Wizard` — hostname, DNS (Cloudflare `1.1.1.1`), time zone, WAN/LAN confirmation. On the general options page:
- Optimize for Multiwan: left checked
- Automatic DHCP/DNS registration: checked
- Optimize for IPsec: unchecked (not using IPsec)

### Harden admin access
Set a strong root password under `System > Access > Users` before continuing.

### Review defaults
- Confirmed `Firewall > Rules > LAN` ships with a default allow-all outbound rule — fine as a starting point, revisited later for lab/home segmentation.
- Noted `System > Firmware > Updates` for periodic patching, especially relevant since this box fronts a future malware-analysis segment.

## Key takeaway
The single setting that prevented the most headaches: changing OPNsense's default LAN subnet away from `192.168.1.x` before touching anything else, since that's the range most home routers already use.

## Next
→ [Phase 3: Wazuh SIEM Setup](https://github.com/DarvensleyE/Projects/blob/main/Isolated%20SOC%20Home%20Lab/Documents/Wazuh-SIEM-Setup.md)
