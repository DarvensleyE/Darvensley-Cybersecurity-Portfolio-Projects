# Phase 3: Wazuh SIEM Setup

## Objective
Deploy Wazuh (manager, indexer, dashboard) on a Proxmox VM to serve as the lab's SIEM.

## What this covers
- Wazuh all-in-one installation
- Diagnosing and resolving a failed `wazuh-manager` service
- Live disk and LVM resizing on a running VM
- Verifying all three Wazuh services and accessing the dashboard

## Steps

### Install Wazuh
Ran Wazuh's official all-in-one installation script on a fresh Ubuntu VM in Proxmox.

### Hit a failure: dashboard stuck on "server is not ready yet"
The Wazuh dashboard sat unresponsive after install. Checked service status directly:
```bash
systemctl status wazuh-manager
```
Result: `failed`.

### Diagnose
```bash
journalctl -u wazuh-manager -e --no-pager
df -h
```
The logs and disk check pointed to the same root cause: **the VM's disk was completely full.** The Wazuh indexer (OpenSearch-based) is disk-intensive, and the VM had been under-provisioned relative to that requirement.

Checked where the space had gone with a layered search:
```bash
du -sh /var/log/* 2>/dev/null | sort -rh | head -10
sudo du -sh /* 2>/dev/null | sort -rh | head -15
sudo du -sh /var/* 2>/dev/null | sort -rh | head -15
```
This confirmed `/var` was consuming the bulk of the disk — consistent with the indexer's data directory.

### Fix: resize the disk live
1. In Proxmox: VM → `Hardware` → disk → `Resize`, increased the disk to 64GB total.
2. Inside the guest, checked the new partition layout:
   ```bash
   lsblk
   ```
3. The partition (`sda3`) had already auto-expanded; the LVM logical volume had not. Extended it:
   ```bash
   sudo pvresize /dev/sda3
   sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
   sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
   ```
4. Confirmed with `df -h` that the filesystem now reflected the full 64GB.

### Restart and verify
```bash
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```
All three came back `active (running)`.

### Confirm the dashboard is reachable
```bash
sudo ss -tulnp | grep 443
```
Confirmed the dashboard was listening, then reached it at `https://<wazuh-vm-ip>` with the default `admin` credentials generated at install time (retrievable via `wazuh-install-files/wazuh-passwords.txt` if not saved during setup).

## Key takeaway
Provisioning a SIEM VM's disk generously from the start (50GB+) avoids this failure entirely — but diagnosing it live, from service logs down to a disk-usage breakdown, down to a live LVM resize without reinstalling anything, was itself a valuable exercise in root-cause troubleshooting under a broken service.

## Next
→ [Phase 4: Connecting It All Together](https://github.com/DarvensleyE/Projects/blob/main/Isolated%20SOC%20Home%20Lab/Documents/Connecting-it-Together.md)
