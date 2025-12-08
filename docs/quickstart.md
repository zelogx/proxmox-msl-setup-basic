# Quickstart

All open-source components --- reproducible setup from scratch.
For the full specification, refer to the main [README](../README.md).

## Requirements

-   One Proxmox VE 9.0+ host
-   Internet router (for port forwarding VPN traffic)
-   Static IP (for Pritunl)

## Network Design Considerations

You will need to provide the following network addresses, which must be configured appropriately.
If your environment has no additional subnets other than the one connected to Proxmox VE, you can generally keep the example values below as-is — except for (a) and (b), which should be set according to your actual network to avoid conflicts.

> Details are described in README.md

![Zelogx MSL Setup Network Overview](./docs/assets/zelogx-MSL-Setup-withID.svg)

### a. MainLan (existing vmbr0): (e.g., 192.168.77.0/24 GW: .254)
### b. Proxmox PVE’s mainlan IP: (e.g., 192.168.77.7)
### c. vpndmzvn (new): (e.g., 192.168.80.0/24 GW: 192.168.80.1)
### d. Client-distributed IPs: (e.g., 192.168.81.0/24)
### e. Number of isolated development segments (number of projects) to create: (e.g., 8)
### f. Network address assigned to each project (vnetpjxx) (new): (e.g., 172.16.16.0/20)
### g. Pritunl mainlan-side IP: (e.g., 192.168.77.10)
### h. Pritunl vpndmzvn-side IP: (e.g., 192.168.80.2)
### i. UDP ports: 

## Installation (Proxmox VE 9.0)

Please follow [build-instructions.md](../build-instructions.md)
