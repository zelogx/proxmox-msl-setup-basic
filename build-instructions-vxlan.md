# Multiverse Secure Lab(MSL) Setup for Proxmox by Zelogx™ - The Multi-tenant Enabler Step by step building guide

[![GitHub Discussions](https://img.shields.io/badge/GitHub-Discussions-181717?logo=github)](https://github.com/zelogx/msl-setup/discussions)
[![Ofiicial Site](https://img.shields.io/badge/Official-Site-blue)](https://www.zelogx.com)
[![Release Notes](https://img.shields.io/badge/Release-notes-green)](https://www.zelogx.com/documents/release-notes/)

## Table of Contents

- [Requirements](#requirements)
- [Overview](#overview)
- [Network Topology Diagram](#network-topology-diagram)
- [Network Design / Segment Design](#network-design--segment-design)
  - [a. MainLAN (existing vmbr0): 192.168.77.0/24  GW: .254](#a-mainlan-existing-vmbr0-19216877024--gw-254)
  - [b. Proxmox PVE MainLAN IP: 192.168.77.2](#b-proxmox-pve-mainlan-ip-192168772)
  - [c. vpndmzvn (new): 192.168.80.0/24  GW: 192.168.80.1](#c-vpndmzvn-new-19216880024--gw-192168801)
  - [d. IP range distributed to VPN clients: 192.168.81.0/24](#d-ip-range-distributed-to-vpn-clients-19216881024)
  - [List of IP address range and number of clients per PJ (OpenVPN)](#list-of-ip-address-range-and-number-of-clients-per-pj-openvpn)
  - [List of IP address range and number of clients per PJ (WireGuard)](#list-of-ip-address-range-and-number-of-clients-per-pj-wireguard)
  - [e. Number of isolated development segments (number of projects): 8](#e-number-of-isolated-development-segments-number-of-projects-8)
  - [f. Network address assigned to each project (vnetpjxx) (new): 172.16.16.0/20 : Project segment](#f-network-address-assigned-to-each-project-vnetpjxx-new-1721616020--project-segment)
  - [g. Pritunl mainlan-side IP](#g-pritunl-mainlan-side-ip)
  - [h. Pritunl vpndmzvn-side IP](#h-pritunl-vpndmzvn-side-ip)
  - [i. UDP Port Numbers for OpenVPN and WireGuard](#i-udp-port-numbers-for-openvpn-and-wireguard)
- [Main Router Configuration Changes](#main-router-configuration-changes)
  - [Configure port forwarding (for OpenVPN and WireGuard × number of projects)](#configure-port-forwarding-for-openvpn-and-wireguard-%C3%97-number-of-projects)
  - [Static Route](#static-route)
- [Proxmox SDN Configuration](#proxmox-sdn-configuration)
  - [Create SDN Zones](#create-sdn-zones)
  - [Create VNets (Virtual Layer-2 Segments)](#create-vnets-virtual-layer-2-segments)
  - [Create IPSet](#create-ipset)
- [Proxmox Firewall Configuration](#proxmox-firewall-configuration)
- [Create VXLAN gateways and VPN return routes](#create-vxlan-gateways-and-vpn-return-routes)
  - [Create the following files](#create-the-following-files)
  - [Reboot](#reboot)
- [Configure VXLAN gateway redundancy](#configure-vxlan-gateway-redundancy)
  - [Overview](#overview-1)
  - [Create the VXLAN gateway creation script for MASTER/BACKUP nodes](#create-the-vxlan-gateway-creation-script-for-masterbackup-nodes)
  - [Test](#test)
- [Create the Pritunl VM](#create-the-pritunl-vm)
  - [Install Ubuntu 24.04 (Minimal)](#install-ubuntu-2404-minimal)
- [Install Pritunl and Dependencies](#install-pritunl-and-dependencies)
  - [Get Setup Key](#get-setup-key)
  - [Access the Web UI](#access-the-web-ui)
  - [Get Default Admin Password](#get-default-admin-password)
  - [Pritunl GUI Setup](#pritunl-gui-setup)
- [Port Table](#port-table)
  - [Create Org & Attach to Server](#create-org--attach-to-server)
  - [Pritunl VM Hardening](#pritunl-vm-hardening)
  - [Additional configurations](#additional-configurations)
- [Create a Self-Care Portal](#create-a-self-care-portal)
  - [Create Group, Pool, and User](#create-group-pool-and-user)
  - [Assign Permissions and Roles to the Group](#assign-permissions-and-roles-to-the-group)
  - [Add Resources to the Pool](#add-resources-to-the-pool)
  - [Allow VPN Users to Access the Proxmox Dashboard](#allow-vpn-users-to-access-the-proxmox-dashboard)
- [Important security note: Always enable MFA for new users](#important-security-note-always-enable-mfa-for-new-users)
- [Known Issues](#known-issues)
- [Day-to-Day Operations](#day-to-day-operations)
  - [When `pj01admin` Creates a VM](#when-pj01admin-creates-a-vm)
  - [During VM Installation](#during-vm-installation)

## Requirements
- Proxmox must already be installed.
- Proxmox VE must be updated to version 9.0.11 or later. Operation on earlier versions has not been verified.
- Proxmox can be either a single-node setup or a multi-node cluster configuration.

## Overview
The following outlines the tasks that will be carried out.

- Configure one static route on the router and set up port forwarding  
  (number of projects × 2).

- Proxmox SDN configuration
  - Create Proxmox SDN VXLAN Zones equal to the number of projects (tenants), plus an additional zone used as the routing path that allows VPN users to access the tenants.
  - Create Proxmox SDN VNets (equivalent to bridges or network segments) inside the zones created above.
  - Assign an IP address and gateway to each VNet. At this point, the creation of networks isolated per project (tenant) is complete.
  - Configure firewall rules between each segment created above.
  - VXLAN gateways are not created automatically, so they must be created using a gateway setup script.

- Configure VXLAN gateway redundancy
  - Overview – In a multi-node cluster configuration, this improves availability.
  - Install and configure keepalived
  - Create a VXLAN gateway creation script for MASTER/BACKUP nodes
  - Test
  
- Build the Pritunl VM
  - Deploy the Pritunl VM and verify network connectivity. After confirming connectivity, create a Server in Pritunl for each project (tenant).
  - Create Organizations in Pritunl and attach them to the corresponding project (tenant) Servers.

- Create a self-care portal.

## Network Topology Diagram
The network that will be created is shown below.

The network addresses / CIDR values that must be determined are labeled **a–i** in the diagram.

![Network Diagram](./docs/assets/zelogx-MSL-Setup-withID.svg)

In a three-node cluster configuration, the network will look like the following:

![Cluster Network Diagram](./docs/assets/zelogx-MSL-Setup-cluster2.jpg)

## Network Design / Segment Design

It is sufficient to confirm that the network addresses / CIDR ranges defined below are **not already in use elsewhere**.

---

### a. MainLAN (existing vmbr0): 192.168.77.0/24  GW: .254
→ Home lab and household network.  
The main server (.1) runs web server, Nextcloud, Samba, personal OpenVPN/WireGuard, Unbound DNS, etc.  
It also includes various home devices such as Alexa, TVs, PS5, the internet router, family PCs, smartphones, and more.

The "Pritunl MainLAN-side IP" configured later **must be within this IP range**.

Many internet routers only support port forwarding to IP addresses within the LAN segment.  
Therefore, it is preferable that this system is connected directly to the LAN under the internet router.

### b. Proxmox PVE MainLAN IP: 192.168.77.2
This IP address will be used as the **destination IP** when adding a static route on the internet router.

### c. vpndmzvn (new): 192.168.80.0/24  GW: 192.168.80.1
This network provides the routing path that allows VPN clients to access each development project subnet.

At minimum, a **/30 network** is required.

### d. IP range distributed to VPN clients: 192.168.81.0/24
This IP range is assigned dynamically to VPN clients when they connect.

This range is further divided between **WireGuard** and **OpenVPN**.

Example:

- OpenVPN  : 192.168.81.002–126 /25  
- WireGuard: 192.168.81.129–254 /25  

This range is then further subdivided based on the **number of isolated development segments to be created**, resulting in **/28 networks**.

Each project can therefore support **up to 13 VPN clients**.

For offshore distributed development or larger teams, it may be preferable to allocate a larger range.

---

### List of IP address range and number of clients per PJ (OpenVPN)

| PJ  | Subnet          | IP Range                     | # Clients |
|---: |-----------------|------------------------------|----------:|
| 1   | 192.168.81.0/28 | 192.168.81.2–192.168.81.14   | 13 |
| 2   | 192.168.81.16/28 | 192.168.81.18–192.168.81.30 | 13 |
| 3   | 192.168.81.32/28 | 192.168.81.34–192.168.81.46 | 13 |
| 4   | 192.168.81.48/28 | 192.168.81.50–192.168.81.62 | 13 |
| 5   | 192.168.81.64/28 | 192.168.81.66–192.168.81.78 | 13 |
| 6   | 192.168.81.80/28 | 192.168.81.82–192.168.81.94 | 13 |
| 7   | 192.168.81.96/28 | 192.168.81.98–192.168.81.110 | 13 |
| 8   | 192.168.81.112/28 | 192.168.81.114–192.168.81.126 | 13 |

### List of IP address range and number of clients per PJ (WireGuard)

| PJ  | Subnet          | IP Range                     | # Clients |
|---: |-----------------|------------------------------|----------:|
| 1   | 192.168.81.128/28 | 192.168.81.130–192.168.81.142 | 13 |
| 2   | 192.168.81.144/28 | 192.168.81.146–192.168.81.158 | 13 |
| 3   | 192.168.81.160/28 | 192.168.81.162–192.168.81.174 | 13 |
| 4   | 192.168.81.176/28 | 192.168.81.178–192.168.81.190 | 13 |
| 5   | 192.168.81.192/28 | 192.168.81.194–192.168.81.206 | 13 |
| 6   | 192.168.81.208/28 | 192.168.81.210–192.168.81.222 | 13 |
| 7   | 192.168.81.224/28 | 192.168.81.226–192.168.81.238 | 13 |
| 8   | 192.168.81.240/28 | 192.168.81.242–192.168.81.254 | 13 |

### e. Number of isolated development segments (number of projects): 8  
(Must be at least 2 and must be a power of two: 2, 4, 8, 16, etc.)

### f. Network address assigned to each project (vnetpjxx) (new): 172.16.16.0/20 : Project segment

- This IP range will be divided according to the **number of isolated development segments to be created**.

- Example:  
  If the network address assigned to vnetpjxx is **172.16.16.0/20** and the number of development segments to create is **8**, the network will be divided as follows.

Example for 8 PJs:

| VNet ID | Subnet           | GW |
|---------|------------------|------------|
| vnetpj01 | 172.16.16.0/24 | 172.16.16.254 |
| vnetpj02 | 172.16.17.0/24 | 172.16.17.254 |
| vnetpj03 | 172.16.18.0/24 | 172.16.18.254 |
| vnetpj04 | 172.16.19.0/24 | 172.16.19.254 |
| vnetpj05 | 172.16.20.0/24 | 172.16.20.254 |
| vnetpj06 | 172.16.21.0/24 | 172.16.21.254 |
| vnetpj07 | 172.16.22.0/24 | 172.16.22.254 |
| vnetpj08 | 172.16.23.0/24 | 172.16.23.254 |

VMs within each vnetpjxx segment can communicate freely.  
Access control for these VMs is enforced via **Security Groups** (SG).  
This corresponds to Pritunl Organizations (Org).

### g. Pritunl mainlan-side IP  
192.168.77.9  
Used as the destination IP for port-forwarding on the home router.

### h. Pritunl vpndmzvn-side IP  
192.168.80.2  
Subnet used by Pritunl when routing clients into project networks.  
A /30 is enough, but /24 is used here.

### i. UDP Port Numbers for OpenVPN and WireGuard

---

## Main Router Configuration Changes

### Configure port forwarding (for OpenVPN and WireGuard × number of projects)

OpenVPN    UDP 11856–11863 (*1) → Pritunl MainLAN-side IP (192.168.77.9)  
WireGuard  UDP 15952–15959 (*1) → Pritunl MainLAN-side IP (192.168.77.9)

*1: Arbitrary UDP ports.  
The number of ports required equals the number of isolated development segments to be created  
(8) × VPN protocols (OpenVPN + WireGuard = 2).

Breakdown:

| pj | OVPN (udp) | WG (udp) |
|---:|-----------:|---------:|
| 01 | 11856 | 15952 |
| 02 | 11857 | 15953 |
| 03 | 11858 | 15954 |
| 04 | 11859 | 15955 |
| 05 | 11860 | 15956 |
| 06 | 11861 | 15957 |
| 07 | 11862 | 15958 |
| 08 | 11863 | 15959 |

### Static Route
  route 172.16.16.0/20 via 192.168.77.2     <---- route (f) via (b)
---

## Proxmox SDN Configuration

### Create SDN Zones

Datacenter → SDN → Zones → Add → VXLAN

- ID: **vpndmz** (for VPN ingress)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj01** (for project01 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj02** (for project02 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj03** (for project03 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj04** (for project04 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj05** (for project05 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj06** (for project06 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj07** (for project07 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**
- ID: **devpj08** (for project08 LANs)
  - Peer Address List: Refer to **How to Configure the Peer Address List**

#### How to Configure the Peer Address List
If your Proxmox nodes are:

- **Single-node setup** (including non-clustered environments)  
  Peer Address List: `<Proxmox node1 IP>`

- **Three-node setup**  
  Peer Address List: `<Proxmox node1 IP>,<Proxmox node2 IP>,<Proxmox node3 IP>`

Example: if you have three nodes with the following IP addresses:

- Proxmox node1 IP: `192.168.77.1`
- Proxmox node2 IP: `192.168.77.2`
- Proxmox node3 IP: `192.168.77.3`

Then the Peer Address List should be:

Peer Address List: `192.168.77.1, 192.168.77.2, 192.168.77.3`

---

### Create VNets (Virtual Layer-2 Segments)

#### DMZ VNet (vpn-dmz-vnet)

- Datacenter → SDN → VNets → Create
  - ID: **vpndmzvn**
  - Alias: vpn-dmz-vnet
  - Zone: vpndmz
  - Tag: 10000

- **Subnets**
  - Subnet: 192.168.80.0/24  
  - GW: 192.168.80.1

#### Create Development (Tenant) LAN VNets
- Create Vnet
  - Datacenter → SDN → VNets → Create
  - See table below for VNet and Zone
- Add Subnet
  - Datacenter → SDN → VNets → <click vnet> → Subnets → Create 
  - See table below for VNet, Subnet and GW

| VNet     | Zone    | Tag   | Subnet         | GW |
|----------|---------|-------|----------------|---------------|
| vnetpj01 | devpj01 | 10001 | 172.16.16.0/24 | 172.16.16.254 |
| vnetpj02 | devpj02 | 10002 | 172.16.17.0/24 | 172.16.17.254 |
| vnetpj03 | devpj03 | 10003 | 172.16.18.0/24 | 172.16.18.254 |
| vnetpj04 | devpj04 | 10004 | 172.16.19.0/24 | 172.16.19.254 |
| vnetpj05 | devpj05 | 10005 | 172.16.20.0/24 | 172.16.20.254 |
| vnetpj06 | devpj06 | 10006 | 172.16.21.0/24 | 172.16.21.254 |
| vnetpj07 | devpj07 | 10007 | 172.16.22.0/24 | 172.16.22.254 |
| vnetpj08 | devpj08 | 10008 | 172.16.23.0/24 | 172.16.23.254 |

> Modified in v1.1 to separate zone for each project. It is better if you use pool base ACL.

#### Apply SDN Configuration

Datacenter → SDN → **Apply**

---

### Create IPSet
Create the following IPSet entries in advance to make management easier.
- GUI → Datacenter → Firewall → IPSet

- [IPSET all_private_ip] # all_private_ip
  - 10.0.0.0/8
  - 127.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16

- [IPSET devpjs] # All development project networks
  - 172.16.16.0/20

- [IPSET MainLAN] # Main LAN Network
  - 192.168.77.0/24

- [IPSET vpn_guest_pool] # VPN Guest Pool
  - 192.168.81.0/24

- [IPSET vxlan_peers] # Cluster VXLAN peer nodes
  - 192.168.77.1
  - 192.168.77.2
  - 192.168.77.3

- [IPSET ceph_peers] # ceph peer nodes if you use ceph
  - 192.168.77.1
  - 192.168.77.2
  - 192.168.77.3

---

## Proxmox Firewall Configuration

Datacenter-level firewall must block access from project LANs to the Proxmox host itself.

GUI → Datacenter → Firewall → Firewall

Rules:

```
# On  Type  Action  Macro  Protocol  Source             S.Port  Destination       D.Port  Log level
# -------------------------------------------------------------------------------------------------
✓    In      ACCEPT  -      vrrp    +dc/vxlan_peers      -       224.0.0.18         -       nolog
✓    In      ACCEPT  -      udp     +dc/vxlan_peers      -       +dc/vxlan_peers    4789    nolog
✓    In      ACCEPT  -      tcp     +dc/all_private_ip   -       -                  8006    nolog
✓    In      ACCEPT  -      tcp     +dc/all_private_ip   -       -                  22      nolog
✓    In      ACCEPT  -      tcp     +dc/all_private_ip   -       -                  3128    nolog
✓    In      ACCEPT  -      tcp     +dc/ceph_peers       -       +dc/ceph_peers     3300    nolog *2
✓    In      ACCEPT  -      tcp     +dc/ceph_peers       -       +dc/ceph_peers     6789    nolog *2
✓    In      ACCEPT  -      tcp     +dc/ceph_peers       -       +dc/ceph_peers     6800:7300 nolog *2
✓    In      ACCEPT  -      -       +MainLAN             -       -                  -       nolog *1
✓    In      ACCEPT  -      ICMP    +vpndmzvn-no-gateway -       +vpndmzvn-gateway  -       nolog *1
✓    In      ACCEPT  -      ICMP    +vpndmzvn-no-gateway -       +devpjs            -       nolog *1
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj08-all    -       +sdn/vnetpj08-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj07-all    -       +sdn/vnetpj07-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj06-all    -       +sdn/vnetpj06-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj05-all    -       +sdn/vnetpj05-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj04-all    -       +sdn/vnetpj04-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj03-all    -       +sdn/vnetpj03-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj02-all    -       +sdn/vnetpj02-all  -       nolog
✓    FORWARD ACCEPT  -      -       +sdn/vnetpj01-all    -       +sdn/vnetpj01-all  -       nolog
✓    FORWARD ACCEPT  -      tcp     +dc/devpjs           -       192.168.77.1       53      nolog
✓    FORWARD ACCEPT  -      udp     +dc/devpjs           -       192.168.77.1       53      nolog
✓    FORWARD DROP    -      -       +dc/devpjs           -       +dc/all_private_ip -       nolog
*1:Option: for reachability test from Pritunl to GWs
*2:Option: Required only when using Ceph storage
```

- GUI → Datacenter → Firewall → Options
  - Firewall: Yes
  - Input policy: DROP
  - Output policy: ACCEPT
  - Forward policy: ACCEPT

- GUI → \<HOST> → Firewall → Options
  - Firewall: Yes
  - nftables: Yes

> Important:
> When Datacenter Firewall is set to **Yes**:
> 1. If you are accessing Proxmox via SSH from a segment different from `vmbr0`, you may get disconnected. It is recommended to configure the firewall rules correctly before enabling it.
> 2. If you have nested PVE, access to VMs running on the nested PVE may stop working. In that case, disable **MAC Filter** in the nested PVE VM's **Firewall Options**.

## Create VXLAN gateways and VPN return routes
> This step is required only for a single-node setup.  
> In a cluster configuration, this will be handled later in the section **"Configure VXLAN gateway redundancy"**.

Since VXLAN gateways are not created automatically, they must be created using a gateway creation script.  
The return route for the VPN is also defined here.

### Create the following files
```bash
cat << 'EOF' > /etc/network/if-up.d/mslsetup-vxlan-gw
#!/bin/bash
case "${IFACE:-}" in
    vnetpj01) ip addr replace 172.16.16.254/24 dev vnetpj01 || true ;;
    vnetpj02) ip addr replace 172.16.17.254/24 dev vnetpj02 || true ;;
    vnetpj03) ip addr replace 172.16.18.254/24 dev vnetpj03 || true ;;
    vnetpj04) ip addr replace 172.16.19.254/24 dev vnetpj04 || true ;;
    vnetpj05) ip addr replace 172.16.20.254/24 dev vnetpj05 || true ;;
    vnetpj06) ip addr replace 172.16.21.254/24 dev vnetpj06 || true ;;
    vnetpj07) ip addr replace 172.16.22.254/24 dev vnetpj07 || true ;;
    vnetpj08) ip addr replace 172.16.23.254/24 dev vnetpj08 || true ;;
    vpndmzvn) ip addr replace 192.168.80.1/24 dev vpndmzvn || true; ip route replace 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn ;;
    *)
        exit 0
        ;;
esac
exit 0
EOF
chmod +x /etc/network/if-up.d/mslsetup-vxlan-gw

cat << 'EOF' > /etc/network/if-down.d/mslsetup-vxlan-gw
#!/bin/bash
case "${IFACE:-}" in
    vpndmzvn) ip route del 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn 2>/dev/null || true; ip addr del 192.168.80.1/24 dev vpndmzvn || true ;;
    vnetpj01) ip addr del 172.16.16.254/24 dev vnetpj01 || true ;;
    vnetpj02) ip addr del 172.16.17.254/24 dev vnetpj02 || true ;;
    vnetpj03) ip addr del 172.16.18.254/24 dev vnetpj03 || true ;;
    vnetpj04) ip addr del 172.16.19.254/24 dev vnetpj04 || true ;;
    vnetpj05) ip addr del 172.16.20.254/24 dev vnetpj05 || true ;;
    vnetpj06) ip addr del 172.16.21.254/24 dev vnetpj06 || true ;;
    vnetpj07) ip addr del 172.16.22.254/24 dev vnetpj07 || true ;;
    vnetpj08) ip addr del 172.16.23.254/24 dev vnetpj08 || true ;;
    *)
        exit 0
        ;;
esac
exit 0
EOF
chmod +x /etc/network/if-down.d/mslsetup-vxlan-gw
```

### Reboot
```bash
reboot
```

## Configure VXLAN gateway redundancy
### Overview
From the VXLAN networks, access to external networks is possible only through the node where the VXLAN gateways have been created.

If the node hosting the VXLAN gateways goes down, access to external networks will be lost.  
To avoid this, this step configures the environment so that VXLAN gateways are automatically created on another node when the current node fails.

In this guide, `keepalived` is used to determine the MASTER node, and VXLAN gateways are created only on the MASTER node.

#### Install keepalived and arping
Run the following on all Proxmox nodes:

```bash
apt update -y
apt install -y keepalived arping
```

#### Configure keepalived
Example configuration for Proxmox node1

Prepare one virtual IP address that will float across Proxmox nodes 1–3, and assign it here.
```bash
cat << 'EOF' > /etc/keepalived/keepalived.conf
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_gw_reachability {
    script "/usr/bin/ping -c 1 -W 1 <MainLAN gateway IP>"
    interval 2
    weight -20
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface vmbr0
    virtual_router_id 100
    priority 100
    advert_int 1
    nopreempt

    authentication {
        auth_type PASS
        auth_pass <any password>
    }

    virtual_ipaddress {
        <virtual IP on vmbr0>/<CIDR> dev vmbr0
    }

    track_script {
        check_gw_reachability
    }

    notify_master "/usr/local/bin/msl-vip-hook.sh master"
    notify_backup "/usr/local/bin/msl-vip-hook.sh backup"
    notify_stop   "/usr/local/bin/msl-vip-hook.sh stop"
    notify_fault  "/usr/local/bin/msl-vip-hook.sh fault"
}
EOF
```
#### Example configuration for Proxmox node2

Use the same configuration as above, except set priority to 80.

#### Example configuration for Proxmox node3

Use the same configuration as above, except set priority to 60.

#### Change the destination of the router's static route to the virtual IP
```
route 172.16.16.0/20 via 192.168.77.4     <---- route (f) via (the virtual IP defined in keepalived.conf)
```

### Create the VXLAN gateway creation script for MASTER/BACKUP nodes
Create following scripts on all nodes in Proxmox cluster
```bash
cat << 'EOF' > /usr/local/bin/msl-vip-hook.sh
#!/bin/bash
set -euo pipefail

ACTION="${1:-}"

log() {
    logger "$*"
}

case "$ACTION" in
    master)
        log "notify_master start"
        ip addr replace 172.16.16.254/24 dev vnetpj01 || true
        ip addr replace 172.16.17.254/24 dev vnetpj02 || true
        ip addr replace 172.16.18.254/24 dev vnetpj03 || true
        ip addr replace 172.16.19.254/24 dev vnetpj04 || true
        ip addr replace 172.16.20.254/24 dev vnetpj05 || true
        ip addr replace 172.16.21.254/24 dev vnetpj06 || true
        ip addr replace 172.16.22.254/24 dev vnetpj07 || true
        ip addr replace 172.16.23.254/24 dev vnetpj08 || true
        ip addr replace 192.168.80.1/24 dev vpndmzvn || true
        ip route replace 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
        # Better run arping here
        log "Became MASTER: VIP/GW attached and GARP completed"
        ;;
    backup|fault|stop)
        log "notify_${ACTION} start"
        ip route del 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
        ip addr del 172.16.16.254/24 dev vnetpj01 || true
        ip addr del 172.16.17.254/24 dev vnetpj02 || true
        ip addr del 172.16.18.254/24 dev vnetpj03 || true
        ip addr del 172.16.19.254/24 dev vnetpj04 || true
        ip addr del 172.16.20.254/24 dev vnetpj05 || true
        ip addr del 172.16.21.254/24 dev vnetpj06 || true
        ip addr del 172.16.22.254/24 dev vnetpj07 || true
        ip addr del 172.16.23.254/24 dev vnetpj08 || true
        ip addr del 192.168.80.1/24 dev vpndmzvn || true
        log "Became ${ACTION}: GW detached"
        ;;
    *)
        log "Unknown action: $ACTION"
        exit 1
        ;;
esac
EOF
chmod +x /usr/local/bin/msl-vip-hook.sh
```

### Test
Verify the following using commands such as ip a and ip r:

- VXLAN gateways are configured on only one node
- The VPN return route is configured only on that node

If the VXLAN gateways are attached correctly, the output should look similar to the following.
On nodes where the VXLAN gateways are not attached, nothing should be displayed.

- Example on a node where the VXLAN gateways are correctly attached
```bash
root@pve1:~# ip a| grep -E 'vnetpj|vpndmz' | grep inet
    inet 172.16.16.254/24 scope global vnetpj01
    inet 172.16.17.254/24 scope global vnetpj02
    inet 172.16.18.254/24 scope global vnetpj03
    inet 172.16.19.254/24 scope global vnetpj04
    inet 172.16.20.254/24 scope global vnetpj05
    inet 172.16.21.254/24 scope global vnetpj06
    inet 172.16.22.254/24 scope global vnetpj07
    inet 172.16.23.254/24 scope global vnetpj08
    inet 192.168.80.1/24 scope global vpndmzvn
```
```bash
root@pve1:~# ip r|grep vpndmz|grep -v link
192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
```

- Example on a node where the VXLAN gateways are not attached
```bash
root@pve2:~# ip a| grep -E 'vnetpj|vpndmz' | grep inet
root@pve2:~# ip r|grep vpndmz|grep -v link
```
> Shut down one node at a time and confirm again with the commands above that the VXLAN gateways move to another node.


---

## Create the Pritunl VM
- NIC1 connected bridge: vmbr0    (IP: 192.168.77.9)
- NIC2 connected bridge: vpndmzvn (IP: 192.168.80.2)

### Install Ubuntu 24.04 (Minimal)

```
sudo apt update -y
sudo apt upgrade -y
sudo shutdown -h 0
```
Take a snapshot.

---

# Install Pritunl and Dependencies

Reference:  
https://docs.pritunl.com/kb/vpn/getting-started/installation

```
# Add package repositories:

sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF

sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF

sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF

# Install dependencies:
sudo apt --assume-yes install gnupg

# Import keys:

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc \
 | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes

curl -fsSL https://swupdate.openvpn.net/repos/repo-public.gpg \
 | sudo gpg -o /usr/share/keyrings/openvpn-repo.gpg --dearmor --yes

curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc \
 | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes

# Install packages:

sudo apt update
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools

# Disable UFW:

sudo ufw disable

# Start and enable services:

sudo systemctl start pritunl mongod
sudo systemctl enable pritunl mongod

# Increase File Descriptor Limits

sudo sh -c 'echo "* hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "* soft nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root soft nofile 64000" >> /etc/security/limits.conf'

# Bind MongoDB to Localhost
sudo sed -i "s/^ *bindIp: .*$/  bindIp: 127.0.0.1/" /etc/mongod.conf
sudo systemctl restart mongod
sudo ss -ltnp | grep 27017

# Check Pritunl Ports
ss -ltnp | egrep '9700|7774|7775|27017'
```

---

### Get Setup Key

```
sudo pritunl setup-key
```

Example output:

```
b41ac4f73e034262a504d0b1bed96d37
```

---

### Access the Web UI

Open browser:

```
https://192.168.77.9
```

Enter the setup key.

---

### Get Default Admin Password

```
sudo pritunl default-password
```

Example:

```
username: "pritunl"
password: "xxxxxxxxxxx"
```

### Pritunl GUI Setup

#### Login to the Web UI.

Enter:
- Public IP (or FQDN)
- Admin username
- Admin password


#### Create Organization

GUI:  
Users → Add Organization  
Name example: **TestPrj1**

#### Add User

GUI:  
Users → Add

- Name: testuser1  
- Organization: TestPrj1  
- Email: (optional)  
- PIN: (optional)

---

#### Create Server

- Servers → Add Server
  - Name: **Server01**
  - Port: **11856** (See OpenVPN port in Port Table below) 
  - Protocol: **udp**
  - Enable WireGuard: **ON**
    - WG Port: **15952** (See WireGuard port in Port Table below) 
  - DNS Server: 1.1.1.1
  - Virtual Network: **192.168.81.0/28** (See Virtual Network List (OpenVPN))
  - Virtual WG Network: **192.168.81.128/28** (Virtual Network List (WireGuard))
  - Advanced Settings
    - Restrict Routing (Split Tunnel): **ON**  
      → Only routes defined under *Server → Routes* will be routed through VPN.
    - Inter-Client Routing: **OFF**  
      → Enforces user isolation.
    - Bind Address: **192.168.77.9**


## Port Table

| PJ | OVPN(udp) | WG(udp) |
|---:|-----------:|--------:|
|01|11856|15952|
|02|11857|15953|
|03|11858|15954|
|04|11859|15955|
|05|11860|15956|
|06|11861|15957|
|07|11862|15958|
|08|11863|15959|

- Virtual Network List (OpenVPN)

| No. | Subnets           | IP range               | # of clients per tenant |
| --- | ----------------- | ----------------------------- |-:|
| 1   | 192.168.81.0/28   | 192.168.81.2–192.168.81.14    |13|
| 2   | 192.168.81.16/28  | 192.168.81.18–192.168.81.30   |13|
| 3   | 192.168.81.32/28  | 192.168.81.34–192.168.81.46   |13|
| 4   | 192.168.81.48/28  | 192.168.81.50–192.168.81.62   |13|
| 5   | 192.168.81.64/28  | 192.168.81.66–192.168.81.78   |13|
| 6   | 192.168.81.80/28  | 192.168.81.82–192.168.81.94   |13|
| 7   | 192.168.81.96/28  | 192.168.81.98–192.168.81.110  |13|
| 8   | 192.168.81.112/28 | 192.168.81.114–192.168.81.126 |13|

- Virtual Network List (WireGuard)

| No. | Subnets           | IP Range               | # of clients per tenant|
| --- | ----------------- | ----------------------------- |-:|
| 1   | 192.168.81.128/28 | 192.168.81.130–192.168.81.142 |13|
| 2   | 192.168.81.144/28 | 192.168.81.146–192.168.81.158 |13|
| 3   | 192.168.81.160/28 | 192.168.81.162–192.168.81.174 |13|
| 4   | 192.168.81.176/28 | 192.168.81.178–192.168.81.190 |13|
| 5   | 192.168.81.192/28 | 192.168.81.194–192.168.81.206 |13|
| 6   | 192.168.81.208/28 | 192.168.81.210–192.168.81.222 |13|
| 7   | 192.168.81.224/28 | 192.168.81.226–192.168.81.238 |13|
| 8   | 192.168.81.240/28 | 192.168.81.242–192.168.81.254 |13|


- Tenant LAN VNets

| VNet | Subnet | GW |
|------|------------------|-----------------|
| vnetpj01 | 172.16.16.0/24 | 172.16.16.254 |
| vnetpj02 | 172.16.17.0/24 | 172.16.17.254 |
| vnetpj03 | 172.16.18.0/24 | 172.16.18.254 |
| vnetpj04 | 172.16.19.0/24 | 172.16.19.254 |
| vnetpj05 | 172.16.20.0/24 | 172.16.20.254 |
| vnetpj06 | 172.16.21.0/24 | 172.16.21.254 |
| vnetpj07 | 172.16.22.0/24 | 172.16.22.254 |
| vnetpj08 | 172.16.23.0/24 | 172.16.23.254 |


#### Create Org & Attach to Server

- Users → Add Organization  
- Name: Any organization name  
  (Example: create pj01, pj02, ... pj08) (*1)

> *1 It is preferable to maintain a one-to-one relationship with the PJ ID.  
> Specifying the actual company name of the users belonging to the organization is not recommended.  
> Using the project name here generally makes operations easier.

- Create Organizations per project → Attach them to the corresponding server  
  (user access enable/disable can be controlled here).
  - Servers → Attach Organization
  - Select an organization

### Pritunl VM Hardening

#### Change the Pritunl GUI Listen Address
- Change the listen IP so that VPN users cannot access the Pritunl VM GUI.

```bash
sudo vi /etc/pritunl.conf
    "bind_addr": "192.168.77.9",  <---- change from 0.0.0.0
```

#### Change sshd Listen Address to Prevent SSH Access from 192.168.80.1
```bash
sudo vi /etc/ssh/sshd_config
ListenAddress 192.168.77.9    <------ add this line

systemctl restart ssh
reboot
```

#### Verify that No Process Is Listening on 0.0.0.0
```bash
sudo ss -lntp | grep '0\.0\.0\.0'
```

#### If PJVM Cannot Ping the Gateway (172.16.1X.254)
- Node (pve1) → Firewall → Rules → Add the following rule at the top:

  - Type: in
  - Action: ACCEPT
  - Protocol: icmp

#### Connectivity Test (3 Steps)

- 77 network → PJ: ping / traceroute (should succeed)
- PJ → 77 network: ping (should fail / DNS should succeed)
- VPN client → PJ: ssh / http (should succeed)

#### Enable Firewall and Apply Security Group for Development VMs

- VM → Firewall → Insert: Security Group
  - Security Group: pj-dev
  - Enable: checked

- VM → Firewall → Options
  - Firewall: Yes


#### The Client Configuration Shows a Local Address as the Connection Target

There is no UI for **Sync Address**, so set it directly in MongoDB:

```bash
sudo pritunl mongodb   # open the mongo shell (or mongosh)
use pritunl
db.hosts.updateMany({}, {$set: {sync_address: "pritunl.zelogx.com"}})
db.hosts.updateMany({}, {$set: {public_address: "ma.zelogx.com"}})  # keep them consistent just in case
quit
sudo systemctl restart pritunl
```


#### Clients Attempt to Resolve DNS Through the VPN

Stop distributing the DNS IP to clients:

```bash
sudo pritunl set vpn.dns_route false
sudo systemctl restart pritunl
```

#### Disable NAT and Use Pure Routing (for the specific server)

If VPN clients access development servers through NAT, all connections will appear to originate from the same user on the server side.  
This makes proper auditing impossible.

- First, check the **Server ID (SID)** for each server:

```bash
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.find({}, {name:1}).forEach(d => print(d.name, d._id.toHexString()))
'
Server01 68e9993dd42cf24361d6cf31
Server02 68e9b49376a47b6c2b180a3b
Server03 68f16ccc3b853aa0ced8694b
Server04 68f16d093b853aa0ced86980
Server05 68f16d553b853aa0ced869be
Server06 68f16d863b853aa0ced869eb
Server07 68f16db73b853aa0ced86a1a
Server08 68f16dea3b853aa0ced86a47
```

-- Check the current NAT setting  
  sid="68e9993dd42cf24361d6cf31"   # Server01 ID

```bash
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
const sid=ObjectId("'$sid'");
db.servers.find({_id:sid},{name:1,network:1,routes:1}).forEach(doc=>printjson(doc));
'
{
  _id: ObjectId('68e9993dd42cf24361d6cf31'),
  name: 'Server01',
  network: '192.168.81.0/28',
  routes: [
    {
      network: '172.16.16.0/24',
      comment: null,
      metric: null,
      nat: true,
      nat_interface: null,
      nat_netmap: null,
      advertise: false,
      vpc_region: null,
      vpc_id: null,
      net_gateway: false,
      server_link: false
    }
  ]
}
```

- Set `nat` to `false` for all SIDs

```bash
sudo mongosh "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.updateMany(
  {},
  { $set: { "routes.$[].nat": false } }
);
'
```

- Check the NAT setting again  
  sid="68e9993dd42cf24361d6cf31"   # Server01 ID

```bash
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
const sid=ObjectId("'$sid'");
db.servers.find({_id:sid},{name:1,network:1,routes:1}).forEach(doc=>printjson(doc));
'
{
  _id: ObjectId('68e9993dd42cf24361d6cf31'),
  name: 'Server01',
  network: '192.168.81.0/28',
  routes: [
    {
      network: '172.16.16.0/24',
      comment: null,
      metric: null,
      nat: true,
      nat_interface: null,
      nat_netmap: null,
      advertise: false,
      vpc_region: null,
      vpc_id: null,
      net_gateway: false,
      server_link: false
    }
  ]
}
```

- Set `nat` to `false` for all SIDs at once

```bash
sudo mongosh "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.updateMany(
  {},
  { $set: { "routes.$[].nat": false } }
);
'
```

- Check iptables NAT Rules

```
sudo iptables -t nat -S | grep -i pritunl
```

If **MASQUERADE disappears**, routing mode is enabled.

```
sudo systemctl restart pritunl
```

---

#### Return Route for VPN Client Pool

The VPN Client Pool (192.168.81.0/24) is not visible from pve1.

With NAT enabled:  
Traffic is NATed to 192.168.80.2 → no route is needed.

With NAT disabled:  
A return route must be added.

On **pve1**:

```
ip route add 192.168.81.0/24 via 192.168.80.2
```

Persist route:

Edit:

```
vi /etc/network/interfaces.d/sdn
```

Add:

```
auto vpndmzvn
iface vpndmzvn
        address 192.168.80.1/24
        bridge_ports none
        bridge_stp off
        bridge_fd 0
        alias vpn-dmz-vnet
        ip-forward on
        post-up ip route add 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
        pre-down ip route del 192.168.81.0/24 via 192.168.80.2 dev vpndmzvn
```

---

### Additional configurations
- Reduce CPU Usage When Idle

Edit:

```
sudo vi /etc/pritunl.conf
```

Add:

```
"mongodb_poll_interval": 30,
```

Restart:

```
sudo systemctl restart pritunl
```

- Pritunl Organizations

> **Do NOT use real company names** when creating Organizations.  
> Each Organization is assigned entirely to a specific PJ-server.  
> Pritunl cannot assign users individually — assignment is Org-level only.

GUI: Users → Add Organization  
Name: any (recommended: 1 Org per PJ, e.g., pj01, pj02 …)

- One-to-One Mapping for This Setup

Each Pritunl VPN server corresponds 1:1 with a development network:

| VPN Server | Development Network |
|------------|----------------------|
| Server01 | vnetpj01 |
| Server02 | vnetpj02 |
| … | … |
| Server08 | vnetpj08 |

- Example Organization–User Mapping

| Org | User |
|-----|------|
| OrgA | UserAA |
| OrgA | UserAB |
| OrgA | UserAC |
| OrgB | UserBA |
| OrgB | UserBB |
| OrgB | UserBC |

Pritunl assigns Orgs → Servers.

Therefore:

- Not possible  
  - Assign only **UserAA + UserBA** to Server01.

- Possible  
  - Entire OrgA  
  - Entire OrgB  
  - Or both

This is why PJ-based Orgs are strongly recommended.

---

## Create a Self-Care Portal

- The following procedure grants VPN users access to the Proxmox dashboard so they can operate VMs.
- If VPN users should not be allowed to operate VMs, this step is not required.

> Note:  
> Because the zone creation method needs to be slightly changed,  
> if only one `devpj` zone has been created, you must create `devpjXX` zones equal to the number of projects.

- Goal for this section:
  - Enable login to the Proxmox dashboard using `pj01Admin@pve`.

- After logging in:
  - Only VMs belonging to PJ01 are visible.
  - Only PJ01 VMs can be started, stopped, opened via console, have their settings modified, snapshots created, and backups performed.
  - New VMs for PJ01 can be created and deleted.
  - VMs, storage, and node settings for other projects in the Datacenter cannot be accessed.

- The following steps allow PJ01 VPN users to create VMs inside PJ01.

---

### Create Group, Pool, and User

- Create a Pool
  - Datacenter → Permissions → Pool → [Create]
  - Name: `pj01`

- Create a Group
  - Datacenter → Permissions → Groups → [Create]
  - Name: `Pj01Admins`

- Create a User
  - Datacenter → Permissions → Users → [Create]
  - UserName: `pj01Admin`
  - Realm: `Proxmox VE authentication server`
  - Group: `Pj01Admins`

---

### Assign Permissions and Roles to the Group

- Datacenter → Permissions → [Add]
  - Path: `/pool/pj01`
  - Group: `Pj01Admins`
  - Role: `PVEAdmins`

---

### Add Resources to the Pool

Add resources to the resource pool (`PJ01`).

- Assign existing VMs
  - Datacenter → pj01 → Members → [Add] → Virtual Machine

- Assign storage
  - Datacenter → pj01 → Members → [Add] → Storage

- Assign SDN Zone network
  - Datacenter → PVE (node) → devpjXX → [Permissions] → [Add] → [Group Permission]
    - Group: `Pj01Admins`
    - Role: `PVEAdmin`

---

### Allow VPN Users to Access the Proxmox Dashboard

- Add the following **Node-level firewall rule**

| ✓ | Chain | Action | Macro | Protocol | Source              | S.Port | Destination           | D.Port | Log   |
|---|-------|--------|-------|----------|---------------------|--------|-----------------------|------:|-------|
| ✓ | in    | ACCEPT | -     | tcp      | +dc/vpn_guest_pool  | -      | +sdn/vnetpjXX-gateway | 8006  | nolog |

`XX`: 01 – NUM_PJ

---
## Important security note: Always enable MFA for new users

VPN credentials stolen from a single compromised PC have been the entry point in several ransomware incidents in Japan in 2025–2026. MFA alone cannot eliminate risk, but it significantly raises the bar for unauthorized access. Treat MFA as mandatory when onboarding new users.

Proxmox has built-in support for multi-factor authentication (MFA).
For example, if you want to use Google Authenticator, you can configure it as follows:

- Go to: **Proxmox → Datacenter → Permissions → Two Factor → [Add] → TOTP**
- Select the user you want to enable MFA for; a QR code and SECRET will be displayed.
- On your smartphone, open **Google Authenticator** and scan the QR code or type the SECRET.
- Enter the 6-digit code shown in Google Authenticator into the Proxmox dialog and click **[Add]**.

Pritunl also has built-in support for multi-factor authentication (MFA):

- Go to: **Pritunl → Servers → [Select server] → [Stop]**
- Then: **Pritunl → Servers → [Select server] → [Settings]**
- Check **“Enable Google Authenticator”** and click **[Save]**.
- Start the server again: **Pritunl → Servers → [Select server] → [Start]**.
- Go to: **Pritunl → Users → [Select user]**, then click the **QR-code icon**.
- Provide the client with the VPN profile and the QR code (or TOTP secret) securely.

With MFA enabled, stolen IDs and passwords alone are no longer enough
to log in. When you onboard a new user, treat MFA as *mandatory*, not
optional.

> Note  
> The Proxmox and Pritunl MFA settings described here do **not** provide
> perfect protection against ransomware. If a client PC is fully compromised,
> even a combination of VDI, EDR, DLP, UTM and other layered defenses cannot
> realistically reduce the risk to zero.  
> However, enabling MFA as described above can help reduce the blast radius
> of an incident and make it significantly harder for attackers to perform
> unauthorized logins and lateral movement.

## Known Issues

- **Quotas cannot be set.** Snapshots and backups can be created without limits.
- **Per-user Proxmox dashboard access control is difficult.**  
  → Pritunl does not allow per-user IP assignment.  
  → Workaround: Only share credentials with specific users (operational control).
- **Audit trails** → Available through Proxmox logs.
- **Error message after VM deletion:**  
  `Permission check failed (/vms/101, VM.Audit) (403)`  
  → This is harmless. Reported on Proxmox forum:  
  [https://forum.proxmox.com/threads/pve-9-0-11-pool-based-rbac-%E2%80%93-gui-shows-permission-check-failed-vms-101-vm-audit-after-successful-vm-delete.178222/](https://forum.proxmox.com/threads/pve-9-0-11-pool-based-rbac-%E2%80%93-gui-shows-permission-check-failed-vms-101-vm-audit-after-successful-vm-delete.178222/)

---

## Day-to-Day Operations

### When `pj01admin` Creates a VM

- **VMID:**  
  An available VMID is automatically assigned, but there is no way to create per-project VMID pools.  
  → Node administrators must provide guidance to avoid management issues later.

- **VM Name:**  
  Cannot be restricted. Node administrators should instruct users to follow naming conventions (e.g., prefix with `pj01`).

- **CPU / Memory:**  
  Cannot be limited.

- **NIC:**  
  With the workaround, the NIC will be automatically assigned to `vnetpj01`.

- **Disk:**  
  Most storage is assigned, so ISO and VM disk storage should be available.  
  Node administrators should provide guidance on where to place different resources.

---

### During VM Installation

VPN users must be informed in advance of the following for `vnetpj01`:

- Available IP address range
- Gateway
- DNS server

