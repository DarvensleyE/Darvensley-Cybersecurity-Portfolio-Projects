# Setup Guide

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the full topology and hardware list before starting.

## 1. Install OPNsense

1. Flash the OPNsense installer image to a USB stick.
2. Boot the mini PC from the USB stick and follow the installer to write OPNsense to the internal drive. This replaces any existing OS.
3. On first boot, the console will prompt for interface assignment.

## 2. Assign interfaces

Identify your two NICs (e.g. `em0` for Intel, `re0` for Realtek) from the console menu.

- Assign the NIC connected to your **home router** as **WAN**.
- Assign the NIC connected to your **switch** as **LAN**.

If using auto-detect, plug/unplug the relevant cable *while the prompt is active* — auto-detect looks for a link state change, not just an existing connection.

## 3. Configure WAN

In the OPNsense GUI (`Interfaces > WAN`):

- **IPv4 Configuration Type:** DHCP (since it's plugged into your home router, which assigns it an address like any other device)
- Leave MAC/MTU fields default
- Uncheck "Block private networks" — otherwise OPNsense will reject the private IP your home router hands it

> Only use Static or PPPoE here if OPNsense is plugged **directly** into your ISP's modem — check your ISP's connection type first.

## 4. Configure LAN

In `Interfaces > LAN`:

- **IPv4 Configuration Type:** Static
- **IPv4 Address:** Choose a subnet **distinct from your home network**, e.g. `192.168.50.1/24`. If your home router uses `192.168.1.x`, do not reuse that range — it causes address collisions.

## 5. Enable DHCP on LAN

In `Services > Kea DHCPv4 > [LAN]` (or `Services > DHCPv4 > [LAN]` on older versions):

- Enable the DHCP server
- Set a range within your LAN subnet, e.g. `192.168.50.10`–`192.168.50.245`
- Leave `.1`–`.9` free for the gateway and any static addresses
- **Deployment type:** Standalone (unless running a second OPNsense box for HA failover)

Any device plugged into the switch now gets an address automatically from this pool.

## 6. Run the setup wizard

`System > Setup Wizard` — set hostname, DNS (e.g. Cloudflare `1.1.1.1`), time zone, and confirm WAN/LAN. On the general options page:

- **Optimize for Multiwan:** leave checked (harmless with a single WAN)
- **Automatic DHCP/DNS registration:** checked — lets you reach devices by hostname later
- **Optimize for IPsec:** leave unchecked unless using IPsec VPN

Set a strong root password under `System > Access > Users` before going further.

**Check for firmware updates:** `System > Firmware > Updates`. Worth checking periodically, especially since this box sits directly in front of a malware analysis lab — keep it patched.

**Review the default LAN firewall rule:** `Firewall > Rules > LAN` ships with a default rule allowing all outbound LAN traffic. Fine to start with — this is the rule to revisit later when segmenting home and lab traffic (see `ARCHITECTURE.md`).

## 7. Cable everything up

- Home router LAN port → OPNsense WAN (`em0`)
- OPNsense LAN (`re0`) → Switch port 1
- Proxmox → Switch port 2
- USB-Ethernet dongle (on your main desktop) → Switch port 3
- Your main desktop's regular NIC/WiFi stays on the home router, untouched

This gives the desktop a dual-homed connection: normal internet on one interface, lab-network access on the other.

## 8. Verify GUI access

From the desktop's dongle connection, browse to `https://192.168.50.1` (your LAN IP). Accept the self-signed certificate warning and log in with `root` and your new password.

## 9. Point Proxmox at the lab network

If Proxmox was previously configured with a **static IP on a different subnet** (e.g. from before OPNsense existed), it won't be reachable on the new network until reconfigured.

**If you can reach Proxmox's old address:**
1. Temporarily set your dongle's IP to match Proxmox's *old* subnet.
2. Log into Proxmox's web UI (`https://<old-ip>:8006`) via that connection.
3. Go to node → `Network`, edit `vmbr0`.
4. Set a static IP **outside the DHCP pool**, e.g. `192.168.50.246/24`, gateway `192.168.50.1`.
5. Apply the configuration.
6. Switch your dongle back to automatic DHCP.
7. Reach Proxmox going forward at its new address.

> Note: the Proxmox GUI's bridge editor does not always expose an explicit "DHCP" toggle — a static IP in the correct subnet is the reliable option.

## 10. Install Wazuh on a Proxmox VM

Follow Wazuh's official all-in-one installation script on a fresh VM with **at least 50GB disk / 4GB RAM**.

If `wazuh-manager` fails to start:

```bash
journalctl -u wazuh-manager -e --no-pager
df -h
```

A full disk is a common cause. If so, resize the VM's disk:

1. In Proxmox: select VM → `Hardware` → disk → `Resize`, add space (e.g. to 64GB total).
2. Inside the guest, check the layout:
   ```bash
   lsblk
   ```
3. If the partition auto-expanded but the LVM volume hasn't:
   ```bash
   sudo pvresize /dev/sda3
   sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
   sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
   ```
4. Confirm space with `df -h`, then:
   ```bash
   sudo systemctl start wazuh-manager
   sudo systemctl status wazuh-manager
   ```

Once `wazuh-manager`, `wazuh-indexer`, and `wazuh-dashboard` all show `active (running)`, the dashboard should load at `https://<wazuh-vm-ip>`.

**Default login:** username `admin`, password generated at install time. Retrieve it if lost:
```bash
sudo tar -xvf wazuh-install-files.tar
cat wazuh-install-files/wazuh-passwords.txt
```

## Troubleshooting

**Confirming OPNsense can see a device (e.g. Proxmox) even without a DHCP lease:**
Statically-configured devices won't show up under DHCP leases since they never requested an address. To confirm OPNsense sees them anyway:
- `Interfaces > Diagnostics > ARP Table` — lists every device OPNsense has recently communicated with on the LAN, regardless of DHCP vs static.
- `Interfaces > Diagnostics > Ping` — ping the device's IP directly from OPNsense.

**Checking what's actually been handed a DHCP lease:**
`Services > Kea DHCP > Leases` shows every device that has requested and received an address from the pool, along with hostname and MAC.

**Wazuh dashboard won't load at all:**
1. Confirm all three services are up:
   ```bash
   sudo systemctl status wazuh-manager
   sudo systemctl status wazuh-indexer
   sudo systemctl status wazuh-dashboard
   ```
   All three must show `active (running)`.
2. Confirm the dashboard is actually listening:
   ```bash
   sudo ss -tulnp | grep 443
   ```
3. Check dashboard logs directly if still stuck:
   ```bash
   sudo journalctl -u wazuh-dashboard -e --no-pager
   ```
4. Double-check you're using `https://`, not `http://`.
5. Try an incognito/private browser window to rule out a cached redirect or a missed certificate-warning page.

## Security notes

- Nothing on the lab segment is reachable from the internet by default — OPNsense blocks all unsolicited inbound WAN traffic unless a `Firewall > NAT > Port Forward` rule explicitly allows it. None is created in this guide.
- Powering OPNsense off does not erase configuration — all settings persist to disk and reload automatically on next boot.
- A static IP address does **not** expose a device to the internet; exposure only happens via explicit port-forwarding, unrelated to whether an address is static or DHCP-assigned.
