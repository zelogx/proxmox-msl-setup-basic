# Architecture Overview

This page provides a high-level visual summary of the Zelogx™ MSL Setup architecture.  
For full details, refer to the main [README](../README.md).

---

## 1. Overall System Architecture
![Overview](./assets/zelogx-MSL-Setup.svg)

This diagram shows the three-tier structure (mainlan, vpndmz, isolated labs)  
and how Proxmox SDN and Pritunl combine to provide per-project L2/L3 isolation.

---

## 2. Network Segments
![Segments](./assets/zelogx-MSL-Setup-withID.svg)

Each project receives its own isolated /24 network,  
and VPN client pools are allocated per-project in /28 blocks.

---

## 3. VPN Traffic Flow
![VPN Flow](./assets/zelogx-MSL-Setup_VPNtraffic.svg)

This illustrates how VPN clients enter through Pritunl, traverse vpndmz,  
and reach only their assigned project network — with mainlan shielded by PVE Firewall.

## 4. VPN Traffic Flow

The following diagram summarizes how VPN clients traverse vpndmz and reach only their assigned project networks.

---

## Why MSL Setup Does **Not** Use VLANs or pfSense/OPNsense

Many Proxmox-based lab environments rely on a combination of VLANs and a
pfSense/OPNsense virtual firewall to perform inter-segment routing.
Zelogx™ MSL Setup intentionally avoids this approach for several critical reasons:

### 1. Bare-Metal Routing Is Significantly Faster
Proxmox routes packets directly in the host kernel, without passing through:

- a virtual NIC (virtio),
- a virtual switch,
- a virtualized firewall OS.

This results in dramatically lower latency and higher throughput compared to
pfSense/OPNsense running inside a VM.  
When each packet must traverse vNIC → hypervisor → guest OS → firewall rules → back to Proxmox,
performance degrades substantially — especially with many isolated project networks.

### 2. No Resource Overhead for a Firewall VM
A pfSense/OPNsense VM permanently consumes:

- CPU cores
- RAM
- virtual disks
- boot time / maintenance effort

MSL Setup eliminates this entirely.  
All routing and filtering happen inside the Proxmox bare-metal host using PVE-Firewall
and SDN, resulting in near-zero overhead.

### 3. Eliminates VLAN Management Complexity
VLAN-based isolation introduces operational friction:

- assigning VLAN IDs  
- tagging/untagging switch ports  
- ensuring correct VLAN propagation across physical switches  
- configuring VLANs inside each VM guest OS  

For multi-project labs, this becomes error-prone and unscalable.  
MSL Setup uses **pure virtual L2 segmentation (vnetpjxx)** with no dependency on
physical switches or guest OS VLAN configuration.

---

In short, MSL Setup chooses Proxmox SDN L2 isolation + bare-metal routing not because VLANs
or pfSense are inherently bad, but because they **do not scale**, **consume more resources**,
and **introduce unnecessary operational risk** for multi-project environments.

