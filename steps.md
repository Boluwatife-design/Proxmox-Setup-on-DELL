# Proxmox VE Reinstall and Network Setup Runbook

Dell PowerEdge R240 | Proxmox VE 9.2 (Debian 13 "trixie") | LACP trunk with tagged VLANs


## 1. Summary

A Dell PowerEdge R240 that previously ran as the "2nd Proxmox Server" (hosting a Secondary DNS VM) was wiped and reinstalled with a fresh Proxmox VE. After the install, the server had link but no connectivity. The cause was a mismatch between the switch port (an LACP port-channel trunk) and the new server network config (a plain single port). The issue was fixed by configuring an LACP bond with a VLAN-aware bridge on the server, matching the switch.

## 2. Hardware

| Item | Value |
|---|---|
| Model | Dell PowerEdge R240 |
| CPU | Intel Xeon E-2224, 4 cores / 4 threads (no hyperthreading) |
| RAM | 16 GB (about 13 GB usable for VMs) |
| Storage | 1 x 931 GB SATA HDD, non-RAID (no redundancy) |
| NICs | 2 x 1 GbE (`nic0`, `nic1`); only `nic1` cabled |
| Management | iDRAC (IP unknown at start; checked in System Setup, F2) |

## 3. Placeholders used in this document

| Placeholder | Meaning |
|---|---|
| `<MGMT_IP>/24` | Server management IP |
| `<GATEWAY_IP>` | Default gateway on the management VLAN |
| `<DNS_IP>` | DNS server |
| `<MGMT_VLAN>` | VLAN ID used to tag the management address |
| `<VM_VLAN>` | Additional VLAN carried for VMs |
| `<SWITCH_PORT>` | Switch port the server is cabled to |

## 4. Installation steps

### 4.1 Prepare installation media
1. Download the Proxmox VE ISO from proxmox.com/downloads.
2. Write it to a USB stick with **Rufus in DD Image mode** (or balenaEtcher).
3. A stick that still contained a Windows installer failed to boot with `Could not locate 'efi\boot\bootx64.efi'`. Rewriting the stick with the Proxmox ISO fixed this.

### 4.2 Check before wiping
- Confirm with the owner that nothing on the server is needed.
- Optional read-only inspection: boot the installer in **Debug mode**, then run `lsblk`, `vgchange -ay`, `lvs` and mount `/dev/pve/root` read-only.
- The old install had dropped into emergency mode waiting for a missing disk mounted at `/mnt/restore`, which suggests an external backup drive existed.

### 4.3 BIOS checks (F2, System Setup)
- Virtualization Technology: **Enabled**
- Boot mode: **UEFI**
- Disk: one 931 GB SATA drive in Non-RAID mode

### 4.4 Boot and install
1. Plug in monitor (VGA), keyboard and USB stick.
2. Press **F11** at the Dell logo, choose **One-shot UEFI Boot Menu**, pick the USB.
3. Choose **Install Proxmox VE (Graphical)**.
4. Target disk: the single 931 GB disk, filesystem **ext4** (ZFS mirroring needs two disks).
5. Network screen: select the NIC that shows a green link dot, then enter hostname (no spaces), `<MGMT_IP>/24`, `<GATEWAY_IP>` and `<DNS_IP>`. Tick **Pin network interface names**.
6. Finish, remove the USB, reboot, and open `https://<MGMT_IP>:8006`.

## 5. Problem: link up but no connectivity

### Symptoms
- Web interface reachable only over a direct laptop cable.
- `ip a`: `nic1` and `vmbr0` showed `NO-CARRIER` until a cable was connected, then `UP`.
- `ping <GATEWAY_IP>` failed with `Destination Host Unreachable`.
- `ip neigh` showed the gateway as `FAILED` (ARP got no answer).
- `ip route` was correct (`default via <GATEWAY_IP> dev vmbr0`).

### Cause
The switch port was part of an **LACP port-channel** carrying tagged VLANs. The old Proxmox install used a matching bond. The new install used a plain port, so the switch **suspended** the port: the link light stayed on, but no traffic passed.

### Key concept: access vs trunk

| | Access port | Trunk port |
|---|---|---|
| Carries | One VLAN | Several VLANs |
| VLAN tags | Added by the switch | Added by the server |
| Server config | Plain bridge, IP on `vmbr0` | VLAN-aware bridge, IP on `vmbr0.<VLAN>` |

**Rule: the server config and the switch port must always match. Change both sides together.**

### Interim fix
The network team changed the port to **access**, and the plain config worked:

```
auto vmbr0
iface vmbr0 inet static
    address <MGMT_IP>/24
    gateway <GATEWAY_IP>
    bridge-ports nic1
    bridge-stp off
    bridge-fd 0
```

### Final fix: LACP bond, VLAN-aware bridge, tagged management IP
The switch port was set back to **trunk, port-channel, LACP active, VLANs allowed**. The server config became:

```
auto lo
iface lo inet loopback

iface nic0 inet manual
iface nic1 inet manual

auto bond0
iface bond0 inet manual
    bond-slaves nic1
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer2+3
    bond-lacp-rate 1

auto vmbr0
iface vmbr0 inet manual
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids <MGMT_VLAN> <VM_VLAN>

auto vmbr0.<MGMT_VLAN>
iface vmbr0.<MGMT_VLAN> inet static
    address <MGMT_IP>/24
    gateway <GATEWAY_IP>

source /etc/network/interfaces.d/*
```

What each part does:
- `bond0` with mode `802.3ad` is LACP, the "handshake" the switch port-channel expects.
- `vmbr0` is a virtual switch. `bridge-vlan-aware yes` lets it handle VLAN tags, and `bridge-vids` lists the allowed VLANs.
- `vmbr0.<MGMT_VLAN>` is the host's own address, tagged with that VLAN. The interface name sets the tag, so check that it matches the VLAN the network team intends.
- VMs pick their VLAN with **VM → Hardware → Network Device → VLAN Tag**.

### Safe change procedure
Do network changes at the **server console**, not the web Shell, because the connection drops.

```
cp /etc/network/interfaces /etc/network/interfaces.access-working
(sleep 300; cp /etc/network/interfaces.access-working /etc/network/interfaces; ifreload -a) &
nano /etc/network/interfaces
ifreload -a
```

Verify, then cancel the auto-undo:

```
cat /proc/net/bonding/bond0     # MII Status: up, Partner Mac not all zeros
ip a show vmbr0.<MGMT_VLAN>
ping -c 3 <GATEWAY_IP>
kill %1
```

## 6. Problem: web interface blocked from the office network and VPN

Browsing to the server showed a FortiGate page: **"FortiGuard Intrusion Prevention - Access Blocked", category "Newly Observed Domain"**.

- Cause: the firewall web filter blocks addresses it has not seen before.
- Fix (network team): exempt `<MGMT_IP>` (TCP 8006) from the web filter.
- Note: `<MGMT_IP>` is a private address and is not reachable from a phone hotspot without the company VPN. Never expose port 8006 to the internet.


## 7. Giving an external user limited access

1. **Datacenter → Permissions → Pools**: create a pool for the project.
2. **Users → Add**: use realm `pve` (not `pam`), strong password, expiry date, 2FA.
3. **Permissions → Add**:

| Path | Role |
|---|---|
| `/pool/<pool>` | PVEVMAdmin |
| `/storage/local-lvm` | PVEDatastoreUser |
| `/storage/local` | PVEDatastoreUser |
| `/sdn/zones/localnetwork` | PVESDNUser |

4. Do not grant Administrator or any role on `/`. Do not share root or Shell access.
5. Access only through the company VPN. Remove the user when the project ends.

## 10. Incident note and lessons learned

- **Lost workload:** the Secondary DNS VM that ran on this server was lost in the wipe. It needs to be rebuilt, and the team should confirm whether a backup exists (possibly on the missing `/mnt/restore` disk).
- **Single disk, no redundancy:** schedule VM backups to separate storage.
- **Document switch settings** (port, port-channel, LACP mode, VLAN tagging) in the asset inventory. Missing network documentation caused most of the downtime.
- **Before wiping a server:** record its network setup (bond, VLANs, switch port) and its VMs.
- **Inventory sheets:** keep credentials out of them. Use a password manager.


