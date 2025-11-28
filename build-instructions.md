# Network Diagram

docs/assets/zelog-MSL-Setup-withID.svg

# Requirements: Proxmox Installation

- The system must be updated to PVE 9.0.11.
- It is recommended (but not mandatory) to disable the Enterprise repository and enable the No-subscription repositories (ceph.sources and proxmox.sources).

# Network Design and Segmentation

The following network design does not have to be implemented exactly as written.  
It is fine as long as you can guarantee that the IP ranges you choose do not overlap with existing ones.

---

### a. MainLAN (vmbr0 existing): 192.168.77.0/24, GW: .254

This is the home-lab / household network.  
Examples of devices connected here:

- CentOS Stream 10 server (192.168.77.1) hosting zelogx web server, Nextcloud, Samba  
- Personal OpenVPN / WireGuard  
- Unbound DNS  
- Alexa devices, TV, PlayStation, home PCs, smartphones  
- Internet router  

The “Pritunl mainlan-side IP” must exist within this network range.

Most consumer routers only allow port-forwarding to LAN-side IPs →  
Therefore, Pritunl should ideally be placed under the home LAN segment.

---

### b. Proxmox PVE mainlan IP  
192.168.77.2  
This becomes the **static-route next-hop** when configuring the internet router.

---

### c. vpndmzvn (new): 192.168.80.0/24, GW: 192.168.80.1

This is the DMZ network that Pritunl uses to route VPN clients into each project VLAN.  
A /30 network is technically sufficient, but here we allocate /24 for simplicity.

---

### d. IP range for VPN clients: 192.168.81.0/24

You can split this range into OpenVPN and WireGuard blocks.  
Example:

- 192.168.81.2–126 (/25) → OpenVPN
- 192.168.81.129–254 (/25) → WireGuard

This range is further divided into /28 blocks, one per project segment.

Maximum clients per PJ = 13.

Suitable for offshore distributed development or multi-member teams.

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

---

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

---

### e. Number of project segments (PJs)

8 PJs.  
Must be a power of two (2, 4, 8, 16…).

---

### f. Project network ranges (new): 172.16.16.0/20

This block is divided into subnets for each PJ based on the total number of PJs.

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

---

### g. Pritunl mainlan-side IP  
192.168.77.9  
Used as the destination IP for port-forwarding on the home router.

---

### h. Pritunl vpndmzvn-side IP  
192.168.80.2  
Subnet used by Pritunl when routing clients into project networks.  
A /30 is enough, but /24 is used here.

---

### i. UDP Port Numbers for OpenVPN and WireGuard

You must configure port-forwarding on the main router.

OpenVPN: UDP 11856–11863 (*1) → 192.168.77.9  
WireGuard: UDP 15952–15959 (*1) → 192.168.77.9  

*1: Arbitrary UDP ports.  
Range = (Number of PJs = 8) × (Protocols = 2)

Details:

| PJ | OVPN(udp) | WG(udp) |
|---:|-----------:|---------:|
|01|11856|15952|
|02|11857|15953|
|03|11858|15954|
|04|11859|15955|
|05|11860|15956|
|06|11861|15957|
|07|11862|15958|
|08|11863|15959|

---

### Static routes (Home Router)

```
route 192.168.80.0/24 via 192.168.77.2
route 172.16.16.0/20 via 192.168.77.2
```

---

# Adding Proxmox SDN Bridges

## Create SDN Zones

Datacenter → SDN → Zones → Add → Simple

- ID: **vpndmz** (for VPN ingress)
- ID: **devpj** (for project LANs)

---

## Create VNets (Virtual Layer-2 Segments)

### DMZ VNet (vpn-dmz-vnet)

Datacenter → SDN → VNets → Create

- ID: **vpndmzvn**
- Alias: vpn-dmz-vnet
- Zone: vpndmz
- VLAN/Tag: *unset (Simple zone requires none)*

**Subnets**

- Subnet: 192.168.80.0/24  
- GW: 192.168.80.1

---

### Development LAN VNets

Datacenter → SDN → VNets → Create

Zone: **devpj**

| VNet | Subnet | GW |
|-------|--------------|-------------|
| vnetpj01 | 172.16.16.0/24 | 172.16.16.254 |
| vnetpj02 | 172.16.17.0/24 | 172.16.17.254 |
| vnetpj03 | 172.16.18.0/24 | 172.16.18.254 |
| vnetpj04 | 172.16.19.0/24 | 172.16.19.254 |
| vnetpj05 | 172.16.20.0/24 | 172.16.20.254 |
| vnetpj06 | 172.16.21.0/24 | 172.16.21.254 |
| vnetpj07 | 172.16.22.0/24 | 172.16.22.254 |
| vnetpj08 | 172.16.23.0/24 | 172.16.23.254 |

---

### Apply SDN Configuration

Datacenter → SDN → **Apply**

---

# Datacenter Firewall Configuration

GUI → Datacenter → Firewall → IPSet

```
[IPSET all_private_ip]
10.0.0.0/8
127.0.0.0/8
172.16.0.0/12
192.168.0.0/16

[IPSET devpjs]
172.16.16.0/20

[IPSET mainlan]
192.168.77.0/24

[IPSET vpn_guest_pool]
192.168.81.0/24
```

---

### Enable Datacenter-Level Firewall

GUI → Datacenter → Firewall → Options

- Firewall: **Yes**
- Input policy: **DROP**
- Output policy: **ACCEPT**
- Forward policy: **ACCEPT**

Also enable Firewall: Yes at Datacenter and Host levels.

> Important:  
> If nested PVE instances are running inside this host, you will lose access to nested-VMs unless you disable the MAC Filter at  
> **VM → Firewall → Options → MAC Filter = No**.

# Blocking Access from Development LAN Gateways to Proxmox

Host-level firewall must block access from project LANs to the Proxmox host itself.

GUI → \<HOST\> → Firewall → Options  
- Firewall: **Yes**  
- nftables: **Yes**

GUI → \<HOST\> → Firewall → Firewall

Rules:

| ✓ | Chain   | Action | Macro | Protocol | Source              | S.Port | Destination         | D.Port | Log |
|---|---------|--------|--------|----------|----------------------|--------|----------------------|--------|------|
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj08-all    | -      | +sdn/vnetpj08-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj07-all    | -      | +sdn/vnetpj07-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj06-all    | -      | +sdn/vnetpj06-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj05-all    | -      | +sdn/vnetpj05-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj04-all    | -      | +sdn/vnetpj04-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj03-all    | -      | +sdn/vnetpj03-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj02-all    | -      | +sdn/vnetpj02-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | -        | +sdn/vnetpj01-all    | -      | +sdn/vnetpj01-all    | -      | nolog |
| ✓ | FORWARD | ACCEPT | -      | tcp      | +dc/devpjs           | -      | 192.168.77.1         | 53     | nolog |
| ✓ | FORWARD | ACCEPT | -      | udp      | +dc/devpjs           | -      | 192.168.77.1         | 53     | nolog |
| ✓ | FORWARD | DROP   | -      | -        | +dc/devpjs           | -      | +dc/all_private_ip   | -      | nolog |

---

# Placing the Pritunl VM on vpndmzvn

NIC Configuration:
- **NIC1 (vmbr0)** → IP: 192.168.77.9  
- **NIC2 (vpndmzvn)** → IP: 192.168.80.2

This provides routing + firewall separation.  
Ports required:
- WireGuard: **15952/UDP**
- OpenVPN: **11856/UDP**

The VM internally manages:

- OpenVPN client pool: **192.168.81.0/24**
- WireGuard client pool: **192.168.81.128/24**

---

# Install Ubuntu 24.04 (Minimal)

Take a snapshot first.

```
sudo apt update -y
sudo apt upgrade -y
sudo shutdown -h 0
```

Take another snapshot.

---

# Install Pritunl and Dependencies

Reference:  
https://docs.pritunl.com/kb/vpn/getting-started/installation

Add package repositories:

```
sudo tee /etc/apt/sources.list.d/mongodb-org.list << EOF
deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse
EOF
```

```
sudo tee /etc/apt/sources.list.d/openvpn.list << EOF
deb [ signed-by=/usr/share/keyrings/openvpn-repo.gpg ] https://build.openvpn.net/debian/openvpn/stable noble main
EOF
```

```
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb [ signed-by=/usr/share/keyrings/pritunl.gpg ] https://repo.pritunl.com/stable/apt noble main
EOF
```

Install dependencies:

```
sudo apt --assume-yes install gnupg
```

Import keys:

```
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc \
 | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor --yes
```

```
curl -fsSL https://swupdate.openvpn.net/repos/repo-public.gpg \
 | sudo gpg -o /usr/share/keyrings/openvpn-repo.gpg --dearmor --yes
```

```
curl -fsSL https://raw.githubusercontent.com/pritunl/pgp/master/pritunl_repo_pub.asc \
 | sudo gpg -o /usr/share/keyrings/pritunl.gpg --dearmor --yes
```

Install packages:

```
sudo apt update
sudo apt --assume-yes install pritunl openvpn mongodb-org wireguard wireguard-tools
```

Disable UFW:

```
sudo ufw disable
```

Start and enable services:

```
sudo systemctl start pritunl mongod
sudo systemctl enable pritunl mongod
```

---

# Increase File Descriptor Limits

```
sudo sh -c 'echo "* hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "* soft nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root soft nofile 64000" >> /etc/security/limits.conf'
```

---

# Bind MongoDB to Localhost

```
sudo sed -i "s/^ *bindIp: .*$/  bindIp: 127.0.0.1/" /etc/mongod.conf
sudo systemctl restart mongod
sudo ss -ltnp | grep 27017
```

---

# Check Pritunl Ports

```
ss -ltnp | egrep '9700|7774|7775|27017'
```

---

# Get Setup Key

```
sudo pritunl setup-key
```

Example output:

```
b41ac4f73e034262a504d0b1bed96d37
```

---

# Access the Web UI

Open browser:

```
https://192.168.77.9
```

Enter the setup key.

---

# Get Default Admin Password

```
sudo pritunl default-password
```

Example:

```
username: "pritunl"
password: "xxxxxxxxxxx"
```

Login to the Web UI.

Enter:
- Public IP (or FQDN)
- Admin username
- Admin password

---

# Pritunl GUI Setup

## Create Organization

GUI:  
Users → Add Organization  
Name example: **TestPrj1**

---

## Add User

GUI:  
Users → Add

- Name: testuser1  
- Organization: TestPrj1  
- Email: (optional)  
- PIN: (optional)

---

## Create Server

Servers → Add Server

- Name: **Server01**
- Port: **11856** (OpenVPN)
- Protocol: **udp**
- Enable WireGuard: **ON**
- WG Port: **15952** (WireGuard)
- DNS Server: 1.1.1.1
- Virtual Network: **192.168.81.0/28** (OpenVPN pool)
- Virtual WG Network: **192.168.81.128/28** (WireGuard pool)

### Advanced Settings

- Restrict Routing (Split Tunnel): **ON**  
  → Only routes defined under *Server → Routes* will be routed through VPN.
- Inter-Client Routing: **OFF**  
  → Enforces user isolation.
- Bind Address: **192.168.77.9**

---

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

---

## Virtual Network List (OpenVPN)

Same as in previous tables.

## Virtual Network List (WireGuard)

Same as in previous tables.

---

# Development LAN VNets

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

---

# Server Routes

Delete:

```
0.0.0.0/0
```

Add:

```
172.16.16.0/24 … 172.16.31.0/24 (each /24 individually, NAT = OFF)
```

(*Do NOT add the entire /20.*)

---

# Create Organization and Attach to Server

Login to Pritunl:

```
https://<pritunl-ip>/
```

User: admin username  
Pass: admin password  

Users → Add Organization  
Name: pj01, pj02, … pj08 (*recommended*)

> Do NOT use company names  
> Project-level Organizations help clean access control

Attach Org to server:

Servers → Attach Organization  
Select the Org

---

# Change Pritunl GUI Listen Address

```
sudo vi /etc/pritunl.conf
```

Change:

```
"bind_addr": "192.168.77.9"
```

(from 0.0.0.0)

---

# Prevent SSH Access From 192.168.80.1 to Pritunl Host

```
sudo vi /etc/ssh/sshd_config
```

Add:

```
ListenAddress 192.168.77.9
```

Restart:

```
systemctl restart ssh
reboot
```

---

# Ensure No Processes Listen on 0.0.0.0

```
sudo ss -lntp | grep '0\.0\.0\.0'
```

# When PJ VMs Cannot Ping the Gateway (172.16.1X.254)

On Proxmox host (pve1):

GUI → Firewall → Rules → **Add**

Add this rule at the *top*:

- Type: **in**
- Action: **ACCEPT**
- Protocol: **icmp**

This allows ICMP Echo Requests from PJ networks to the gateway.

---

# Connectivity Test (3 Steps)

1. **77-network → PJ**  
   - ping/traceroute → should work

2. **PJ → 77-network**  
   - ping → fails (expected)  
   - DNS → works

3. **VPN Client → PJ**  
   - ssh / http → should work

---

# Development VMs Must Have Firewall + Security Group Enabled

VM → Firewall → insert Security Group  
- Security Group: **pj-dev**  
- Enable: **checked**

VM → Firewall → Options  
- Firewall: **Yes**

---

# When Client Config Uses Local Address Instead of Public IP

Pritunl GUI has no UI for “Sync Address”.  
Modify MongoDB directly:

```
sudo pritunl mongodb
use pritunl
db.hosts.updateMany({}, {$set: {sync_address: "pritunl.zelogx.com"}})
db.hosts.updateMany({}, {$set: {public_address: "ma.zelogx.com"}})
quit
sudo systemctl restart pritunl
```

---

# Prevent VPN Clients From Using Pritunl for DNS

Disable DNS route distribution:

```
sudo pritunl set vpn.dns_route false
sudo systemctl restart pritunl
```

---

# Disable NAT and Use Pure Routing (Per-Server)

## 1. Get SID (Server ID)

```
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.find({}, {name:1}).forEach(d => print(d.name, d._id.toHexString()))
'
```

Example output:

```
Server01 68e9993dd42cf24361d6cf31
Server02 68e9b49376a47b6c2b180a3b
...
Server08 68f16dea3b853aa0ced86a47
```

---

## 2. Check NAT Status

```
sid="68e9993dd42cf24361d6cf31"
sudo mongosh --quiet "mongodb://127.0.0.1:27017/pritunl" --eval '
const sid=ObjectId("'$sid'");
db.servers.find({_id:sid},{name:1,network:1,routes:1}).forEach(doc=>printjson(doc));
'
```

Example (NAT=true):

```
{
  _id: ObjectId('68e9993dd42cf24361d6cf31'),
  name: 'Server01',
  network: '192.168.81.0/28',
  routes: [
    {
      network: '172.16.16.0/24',
      nat: true,
      ...
    }
  ]
}
```

---

## 3. Disable NAT for All Servers

```
sudo mongosh "mongodb://127.0.0.1:27017/pritunl" --eval '
db.servers.updateMany(
  {},
  { $set: { "routes.$[].nat": false } }
);
'
```

---

# Check iptables NAT Rules

```
sudo iptables -t nat -S | grep -i pritunl
```

If **MASQUERADE disappears**, routing mode is enabled.

```
sudo systemctl restart pritunl
```

---

# Return Route for VPN Client Pool

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

# Reduce CPU Usage When Idle

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

---

# Pritunl Organizations

> **Do NOT use real company names** when creating Organizations.  
> Each Organization is assigned entirely to a specific PJ-server.  
> Pritunl cannot assign users individually — assignment is Org-level only.

GUI: Users → Add Organization  
Name: any (recommended: 1 Org per PJ, e.g., pj01, pj02 …)

---

# One-to-One Mapping for This Setup

Each Pritunl VPN server corresponds 1:1 with a development network:

| VPN Server | Development Network |
|------------|----------------------|
| Server01 | vnetpj01 |
| Server02 | vnetpj02 |
| … | … |
| Server08 | vnetpj08 |

---

# Example Organization–User Mapping

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

### Not possible  
Assign only **UserAA + UserBA** to Server01.

### Possible  
- Entire OrgA  
- Entire OrgB  
- Or both

This is why PJ-based Orgs are strongly recommended.

