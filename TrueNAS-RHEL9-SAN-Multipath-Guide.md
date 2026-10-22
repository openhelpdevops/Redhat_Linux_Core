# TrueNAS and RHEL 9: four-path multipathing lab and production Fibre Channel reference

**Prepared:** 22 October 2026  
**Addressing revision:** your LAN gateway is `192.168.0.1`; management uses `192.168.0.0/24`, and the four isolated storage paths use separate `192.168.x.0/24` subnets.  
**Lab software:** TrueNAS Community Edition 25.10.7 and RHEL 9.x, x86_64  
**Style:** manual commands, GUI steps, expected results, colour Mermaid diagrams  
**Execution status:** documentation-based procedure; the VMs and physical SAN were not deployed or exercised by the author.

The Markdown includes self-contained Mermaid source. Use the ZIP bundle when you want the two JPG diagrams alongside the document: keep its `images` directory next to this file so the relative image links work in Git.

## 1. Read this distinction first

Your requested physical server design is valid:

**Two separate dual-port FC HBA cards = four FC initiator ports = four host WWPNs.**

However, a normal TrueNAS Community Edition VM and four virtual Ethernet adapters cannot reproduce those physical FC ports. The official TrueNAS documentation makes FC target functionality an Enterprise-licensed feature requiring suitable FC hardware. The version-specific UI and appliance support also matter. See [S1], [S2] and [S3].

This guide therefore supplies two clearly labelled routes:

| Route | What you actually build | Identifiers on the RHEL host | Result |
|---|---|---|---|
| **A: free VM lab — start here** | TrueNAS CE exports one iSCSI LUN over four isolated Ethernet paths to a second RHEL VM | One initiator IQN, four interface bindings, four sessions, one disk WWID | A working four-path DM Multipath exercise |
| **B: actual FC hardware** | Two physical dual-port FC HBA cards, two independent fabrics, FC switches, and a supported FC target such as licensed TrueNAS Enterprise | Four host WWPNs; one LUN WWID | Real FC multipathing; eight paths in the illustrated target-port layout |

**Route A teaches storage multipathing. It does not create four FC WWPNs or test FC zoning.** Renaming four virtual NICs to “HBA1/HBA2” would not change their protocol.

The free VM lab has one TrueNAS VM. Its four paths do not provide storage-controller HA, hypervisor HA or disaster recovery. Route B shows the separate production architecture.

### 1.1 What bonding does here

| Technology | Joins or manages | Linux result |
|---|---|---|
| Ethernet bonding | Ethernet NIC links | A network interface such as `bond0` |
| DM Multipath | Paths to the same block-storage LUN | A block device such as `/dev/mapper/truenas_lun01` |

Keep the four dedicated storage NICs separate in Route A. TrueNAS recommends multipath rather than LACP/LAGG for iSCSI [S4]. Ethernet bonds used for application or management traffic are a separate part of a production server.

### 1.2 Contents and order

1. Sections 2–4: understand the production FC topology and the VM equivalent.
2. Sections 5–7: create the virtual networks and install the two VMs.
3. Sections 8–10: configure RHEL networking and the TrueNAS block share.
4. Sections 11–15: bind four iSCSI paths, configure multipathing, and create a filesystem.
5. Sections 16–19: test failover, reboot persistence, troubleshooting, and shutdown.
6. Sections 20–23: actual FC hardware, Cisco MDS zoning examples, and RHEL FC setup.
7. Sections 24–26: interview reference, completion record, and sources.

## 2. Production reference: four FC ports and multiple FC switches

This is a storage-only view of a critical Linux server. Ethernet ToR switches, application bonds, client networks and management networks are omitted so they cannot be confused with FC switches.

<img width="3000" height="2600" alt="NetApp-FC-SAN-Physical-Cabling" src="https://github.com/user-attachments/assets/e93e36fc-c764-44a5-b2e7-6d0f8c470d1b" />


### 2.1 Physical inventory

| Item | Example quantity | Purpose |
|---|---:|---|
| Linux server | 1 shown | Runs the application and DM Multipath |
| Dual-port FC HBA card | 2 | Four host FC ports; card failure does not remove every path |
| FC fabric | 2 | Independent failure domains, called A and B |
| FC switch | 4 | Edge and core switch in each fabric |
| HA SAN controller | 2 | Each presents the same LUN through a port in each fabric |
| LUN | 1 shown | One logical disk with one persistent WWID |

For a small SAN, one switch per fabric is sufficient to illustrate dual-fabric redundancy. The expanded reference below uses two switches per fabric to show a genuine multi-switch layout. Additional switches do not, by themselves, create additional Linux SCSI paths.

```mermaid
flowchart TB
    M["RHEL: DM Multipath"]
    subgraph H1["PCIe HBA card 1"]
      P11["Port 1: A1"]
      P12["Port 2: B1"]
    end
    subgraph H2["PCIe HBA card 2"]
      P21["Port 1: A2"]
      P22["Port 2: B2"]
    end
    M --> P11
    M --> P12
    M --> P21
    M --> P22
    P11 --> AE["Fabric A: edge switch"]
    P21 --> AE
    AE -->|"ISL inside Fabric A"| AC["Fabric A: core switch"]
    P12 --> BE["Fabric B: edge switch"]
    P22 --> BE
    BE -->|"ISL inside Fabric B"| BC["Fabric B: core switch"]
    AC --> C1["SAN controller 1"]
    AC --> C2["SAN controller 2"]
    BC --> C1
    BC --> C2
    C1 --> L["Same LUN: one WWID"]
    C2 --> L
    classDef host fill:#dbeafe,stroke:#2563eb,color:#172554;
    classDef a fill:#dcfce7,stroke:#15803d,color:#14532d;
    classDef b fill:#f3e8ff,stroke:#7e22ce,color:#581c87;
    classDef disk fill:#ffedd5,stroke:#c2410c,color:#7c2d12;
    class M host;
    class P11,P21,AE,AC a;
    class P12,P22,BE,BC b;
    class C1,C2,L disk;
```

**An ISL is an inter-switch link inside one fabric. Fabric A and Fabric B are not joined in this reference.** FC switch meshing, when used, is designed within a fabric; it is not a replacement for keeping the two fabrics independent [S5, S15, S16].

### 2.2 Host cabling and host WWPN count

| Host card and port | Name in this guide | Fabric | Unique host WWPN |
|---|---|---|---|
| HBA card 1, port 1 | A1 | A | WWPN-A1 |
| HBA card 1, port 2 | B1 | B | WWPN-B1 |
| HBA card 2, port 1 | A2 | A | WWPN-A2 |
| HBA card 2, port 2 | B2 | B | WWPN-B2 |

The target has four target ports: controller 1/A, controller 1/B, controller 2/A and controller 2/B. These have their own target WWPNs; they are not extra host WWPNs.

### 2.3 Why this reference has eight paths, not four

Each of the four host ports can reach one target port on each controller in its fabric:

| Path | Host port | Switch route | Target port | Disk |
|---|---|---|---|---|
| 1 | A1 | A-edge → A-core | Controller 1/A | Same LUN |
| 2 | A1 | A-edge → A-core | Controller 2/A | Same LUN |
| 3 | A2 | A-edge → A-core | Controller 1/A | Same LUN |
| 4 | A2 | A-edge → A-core | Controller 2/A | Same LUN |
| 5 | B1 | B-edge → B-core | Controller 1/B | Same LUN |
| 6 | B1 | B-edge → B-core | Controller 2/B | Same LUN |
| 7 | B2 | B-edge → B-core | Controller 1/B | Same LUN |
| 8 | B2 | B-edge → B-core | Controller 2/B | Same LUN |

**Four host WWPNs and eight paths are consistent.** Path count depends on initiator ports, visible target ports and LUN mapping. A different zoning or target-port design can give a different count. The host OS and array must support the selected count.

### 2.4 Failure behaviour in this reference

| Failure | Remaining connectivity |
|---|---|
| HBA card 1 | HBA card 2 retains A2 and B2: four of eight paths |
| One FC edge/core switch or its only ISL in Fabric A | Fabric B retains four paths |
| One SAN controller | The other controller retains four paths |
| One host FC cable | Three initiator ports retain six paths |
| Entire server | Requires an application/OS cluster peer; multipathing cannot replace the server |
| Entire SAN array/site | Requires a separate replication/DR design |

These are topology counts, not promises of zero interruption. ALUA state, controller takeover and vendor timeout settings determine actual recovery behaviour. An HA server pair also needs quorum/fencing and controlled shared-disk access [S17].

## 3. Free VM lab: the four independent iSCSI paths you can build now

Run both VMs on the same VMware Workstation host or on the same VirtualBox host for this procedure. The four virtual storage networks are local to that hypervisor.

![Free TrueNAS and RHEL VM lab with four independent iSCSI paths to one LUN](images/truenas-rhel-four-path-lab.jpg)

```mermaid
flowchart TB
    R["RHEL VM: four separate storage NICs"]
    A1["A1 network: 192.168.110.0/24"]
    B1["B1 network: 192.168.120.0/24"]
    A2["A2 network: 192.168.111.0/24"]
    B2["B2 network: 192.168.121.0/24"]
    T["TrueNAS CE VM: one iSCSI target"]
    L["One extent and one 20 GiB LUN"]
    R --> A1 --> T
    R --> B1 --> T
    R --> A2 --> T
    R --> B2 --> T
    T --> L
    classDef host fill:#dbeafe,stroke:#2563eb,color:#172554;
    classDef a fill:#dcfce7,stroke:#15803d,color:#14532d;
    classDef b fill:#f3e8ff,stroke:#7e22ce,color:#581c87;
    classDef disk fill:#ffedd5,stroke:#c2410c,color:#7c2d12;
    class R host;
    class A1,A2 a;
    class B1,B2 b;
    class T,L disk;
```

The A/B labels are learning labels. These are four Ethernet virtual networks, not four FC switches. Four subnets avoid ambiguous routing and the TrueNAS restriction against separate interfaces on the same subnet [S4].

### 3.1 What corresponds and what does not

| Production FC concept | VM-lab counterpart | Difference |
|---|---|---|
| Four physical FC initiator ports | Four RHEL Ethernet vNICs | vNICs have MAC/IP addresses, not FC WWPNs |
| Two physical HBA cards | No equivalent physical card separation | All virtual NICs may depend on one host |
| A/B fabric connectivity | A1/A2 and B1/B2 logical path groups | Group outage is simulated by disabling two NICs |
| FC zoning | IQN ACL, CHAP and isolated networks | Different access-control mechanisms |
| FC initiator login | iSCSI session login | Different transport protocol |
| One LUN reached by several paths | One iSCSI LUN reached by four sessions | Same DM Multipath learning objective |
| Two HA SAN controllers | One TrueNAS VM | No controller failover exercise |
| Eight paths in Section 2 | Four paths in this lab | Each initiator connects to one target portal |

Do not create four separate zvols or four independent extents. Four different disks will have different WWIDs and must not be combined as if they were paths to one disk.

## 4. Addressing, names and VM sizing

**Your home LAN:** this revision uses your gateway `192.168.0.1` and assumes a LAN mask of `255.255.255.0` (`/24`). Check the router's actual mask before applying the examples. Both VMs use Adapter 1 for management and internet access on that LAN.

All lab addresses now begin with `192.168`. With a `/24` mask, the third number identifies the subnet: `192.168.0.x` is your home LAN, while `192.168.110.x`, `192.168.120.x`, `192.168.111.x` and `192.168.121.x` are four different storage subnets. Keep the storage subnets on their own host-only/internal virtual networks. TrueNAS requires separate subnets for these separate interfaces [S4].

You do not need extra routers or gateways such as `192.168.110.1` for this lab. RHEL and TrueNAS talk directly on each storage network. Set `192.168.0.1` as the management default gateway on both VMs; leave the gateway blank on every storage interface. The physical FC reference uses WWPNs for storage paths, so these IP changes apply to the Ethernet/iSCSI lab.

### 4.1 Host and service names

| Item | Value |
|---|---|
| TrueNAS VM name | `TrueNAS-SAN` |
| TrueNAS hostname | `truenas01.openhelp.net` |
| RHEL VM name | `RHEL-SAN01` |
| RHEL hostname | `rhel-san01.openhelp.net` |
| TrueNAS management IP | `192.168.0.80/24` |
| RHEL management IP | `192.168.0.81/24` |
| Management subnet / mask | `192.168.0.0/24` / `255.255.255.0` — assumed; verify on your router |
| Management gateway | `192.168.0.1` — your LAN router, used by Adapter 1 on both VMs |
| Management DNS | Your working LAN DNS; `192.168.0.10` only if your FreeIPA DNS actually uses it. Use `192.168.0.1` only if your router provides DNS forwarding. |
| RHEL initiator IQN | `iqn.2026-10.net.openhelp:rhel-san01` |
| TrueNAS target basename | `iqn.2026-10.net.openhelp:truenas01` |
| Target short name | `rhel-lun01` |
| Full target IQN | `iqn.2026-10.net.openhelp:truenas01:rhel-lun01` |
| Portal group description | `rhel-four-path-portals` |
| Extent | `rhel-lun01-extent` |
| Pool / zvol | `sanpool/rhel_lun01` |
| iSCSI LUN ID | `0` |
| Linux multipath alias | `truenas_lun01` |

The VM management addresses `192.168.0.80` and `192.168.0.81` are suggested static addresses, not addresses assigned automatically by this document. Confirm both are unused and reserve them or exclude them from your router's DHCP pool. Confirm that the four storage subnets do not overlap your LAN, VPN or existing virtual networks. IQN strings are identifiers; DNS resolution does not establish an iSCSI session. Copy the actual target IQN displayed by your TrueNAS system if it differs.

### 4.2 Four storage paths

| VM adapter | Path | VMware network | VirtualBox internal network | TrueNAS IP | RHEL IP | Mask |
|---|---|---|---|---|---|---|
| Adapter 2 | A1 | VMnet10 | SAN-A1 | `192.168.110.10` | `192.168.110.20` | `/24` |
| Adapter 3 | B1 | VMnet11 | SAN-B1 | `192.168.120.10` | `192.168.120.20` | `/24` |
| Adapter 4 | A2 | VMnet12 | SAN-A2 | `192.168.111.10` | `192.168.111.20` | `/24` |
| Adapter 5 | B2 | VMnet13 | SAN-B2 | `192.168.121.10` | `192.168.121.20` | `/24` |

Adapter 1 is management: TrueNAS `192.168.0.80/24`, RHEL `192.168.0.81/24`, and gateway `192.168.0.1` on both. Give storage NICs **no default gateway and no DNS settings**. They use directly connected routes. Keep MTU 1500 throughout this first exercise.

| Network | Subnet | Gateway | Connection type |
|---|---|---|---|
| Management / internet | `192.168.0.0/24` | `192.168.0.1` | Adapter 1, bridged to your home LAN |
| Storage A1 | `192.168.110.0/24` | None | Adapter 2, isolated VMnet10 / SAN-A1 |
| Storage B1 | `192.168.120.0/24` | None | Adapter 3, isolated VMnet11 / SAN-B1 |
| Storage A2 | `192.168.111.0/24` | None | Adapter 4, isolated VMnet12 / SAN-A2 |
| Storage B2 | `192.168.121.0/24` | None | Adapter 5, isolated VMnet13 / SAN-B2 |

### 4.3 VM resources

| Resource | TrueNAS-SAN | RHEL-SAN01 |
|---|---|---|
| vCPU | 2 | 2 |
| RAM | 8 GiB minimum; 16 GiB useful if available | 4 GiB |
| OS disk | 32 GiB | 40 GiB |
| Data virtual disks attached directly | Two empty 60 GiB disks | None: data arrives through iSCSI |
| Network adapters | 5 | 5 |
| ISO | TrueNAS CE 25.10.7, or the then-current General release | RHEL 9.x |

Create a mirror from the two TrueNAS data disks and a 20 GiB zvol. A mirror of two virtual disks on one physical laptop disk teaches pool configuration, but it does not survive loss of that laptop disk.

The TrueNAS software-status page listed 25.10.7 for General use when checked for this guide [S2]. Recheck it before downloading a later release. The official installation guide supports VM installation with a separate boot disk and data disk [S3].

## 5. Create the virtual storage networks

Choose **5A or 5B**, matching your hypervisor. Do not apply both procedures to one VM pair.

### 5A. VMware Workstation — GUI

1. Power off both lab VMs.
2. Open **Edit → Virtual Network Editor** and use **Change Settings** if required.
3. Add `VMnet10`, set it to **Host-only**, and set subnet `192.168.110.0`, mask `255.255.255.0`.
4. Disable the VMware DHCP service for this network. The VM addresses will be static.
5. Add the remaining networks:

| VMnet | Type | Subnet | DHCP |
|---|---|---|---|
| VMnet10 | Host-only | `192.168.110.0/24` | Off |
| VMnet11 | Host-only | `192.168.120.0/24` | Off |
| VMnet12 | Host-only | `192.168.111.0/24` | Off |
| VMnet13 | Host-only | `192.168.121.0/24` | Off |

6. A host virtual adapter is optional; the two VMs can communicate on each virtual network without an IP on the laptop itself.
7. In **both VMs**, configure Adapter 1 for your management network. Bridged networking gives the VMs management access on your LAN; select the actual host uplink rather than relying on automatic bridging when the host has several uplinks.
8. Add four **Custom: Specific virtual network** adapters according to Section 4.2.
9. Select **Connected** and **Connect at power on** for each adapter.
10. Record each adapter's MAC address. Match adapters to guest interface names by MAC, not by assuming the guest names follow the VM adapter order.

These host-only networks are not automatically available on another laptop. If the two VMs run on different hypervisor hosts, you need four reachable VLAN-backed networks or a separately designed routed setup; this guide assumes one hypervisor host.

### 5B. VirtualBox — GUI and host CLI

VirtualBox can expose additional adapters using `VBoxManage`; the ordinary GUI may show only the first four [S12]. Use the same host user account for both VMs' internal networks.

1. With each VM powered off, set Adapter 1 to **Bridged Adapter** and select the actual management uplink.
2. On the hypervisor host, create Adapter 2–5 attachments. The internal networks are created by using the same network names on the two VMs.
3. The commands below use the long-standing compact option spelling accepted by VirtualBox releases. Check `VBoxManage modifyvm --help` for your installed version if it requests the newer hyphenated spelling.

**Run on the VirtualBox host, not inside RHEL or TrueNAS:**

```bash
VBoxManage modifyvm "TrueNAS-SAN" --nic2 intnet --intnet2 SAN-A1 --nictype2 82540EM --cableconnected2 on
VBoxManage modifyvm "TrueNAS-SAN" --nic3 intnet --intnet3 SAN-B1 --nictype3 82540EM --cableconnected3 on
VBoxManage modifyvm "TrueNAS-SAN" --nic4 intnet --intnet4 SAN-A2 --nictype4 82540EM --cableconnected4 on
VBoxManage modifyvm "TrueNAS-SAN" --nic5 intnet --intnet5 SAN-B2 --nictype5 82540EM --cableconnected5 on

VBoxManage modifyvm "RHEL-SAN01" --nic2 intnet --intnet2 SAN-A1 --nictype2 82540EM --cableconnected2 on
VBoxManage modifyvm "RHEL-SAN01" --nic3 intnet --intnet3 SAN-B1 --nictype3 82540EM --cableconnected3 on
VBoxManage modifyvm "RHEL-SAN01" --nic4 intnet --intnet4 SAN-A2 --nictype4 82540EM --cableconnected4 on
VBoxManage modifyvm "RHEL-SAN01" --nic5 intnet --intnet5 SAN-B2 --nictype5 82540EM --cableconnected5 on

VBoxManage showvminfo "TrueNAS-SAN"
VBoxManage showvminfo "RHEL-SAN01"
```

Expected: five enabled adapters in each VM; the four storage adapters use the same four internal-network names on both VMs. Do not enable DHCP on these internal networks.

## 6. Install TrueNAS and set management access

**Location: TrueNAS VM console and TrueNAS web UI.**

1. Download the installer from the official [TrueNAS Community Edition download page](https://www.truenas.com/download-truenas-community-edition/). Verify the published checksum.
2. Boot `TrueNAS-SAN` from the ISO and select **Install/Upgrade**.
3. Select only the 32 GiB OS disk. Identify disks by their configured size and virtual-disk identity. The installer erases the selected disk.
4. Complete the local administrator setup offered by that release. Record the actual administrator username; do not assume that it is always `root` or `admin`.
5. Select the boot option appropriate to the VM firmware, finish installation, remove the ISO and reboot.
6. In the console setup menu, identify Adapter 1 by its MAC and configure management address `192.168.0.80/24`.
7. Set the management IPv4 default gateway to `192.168.0.1`. Configure your working LAN DNS server; use `192.168.0.10` only if that is your actual FreeIPA DNS address. Do not put gateways on the storage interfaces.
8. Open `https://192.168.0.80` from the workstation and log in. A fresh installation may present its initial self-signed certificate.
9. Set hostname `truenas01`, domain `openhelp.net`, time zone and time synchronisation under the applicable system/network settings.
10. Record the installed release. Use documentation for that stable release; generic TrueNAS documentation URLs may open development documentation unless you select the correct version.

## 7. Install RHEL 9 and identify its NICs

**Location: RHEL VM console; root shell after installation.**

1. Install RHEL 9.x on the 40 GiB virtual OS disk.
2. Configure only the management interface during installation: address `192.168.0.81`, prefix `24` (mask `255.255.255.0`), gateway `192.168.0.1`, and your working LAN DNS. Reserve this address or exclude it from DHCP before using it.
3. Register the VM with your available Red Hat entitlement if it is not already registered.
4. Set the hostname and inspect the NIC inventory:

```bash
hostnamectl set-hostname rhel-san01.openhelp.net
cat /etc/redhat-release
ip -br link
ip -br address
nmcli -f DEVICE,TYPE,STATE,CONNECTION device status
nmcli -f GENERAL.DEVICE,GENERAL.HWADDR device show
```

Match each guest MAC to the hypervisor adapter inventory. Example names below are **not guaranteed**:

| Purpose | Example VMware guest name | Example VirtualBox guest name | Your actual name |
|---|---|---|---|
| Management | ens160 | enp0s3 | Fill in |
| A1 | ens192 | enp0s8 | Fill in |
| B1 | ens224 | enp0s9 | Fill in |
| A2 | ens256 | enp0s10 | Fill in |
| B2 | ens288 | enp0s16 | Fill in |

Confirm package repositories:

```bash
subscription-manager identity
dnf repolist
```

If the VM is registered but the standard RHEL 9 x86_64 repositories are disabled, enable them:

```bash
subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms --enable=rhel-9-for-x86_64-appstream-rpms
```

Install the tools used by the guide:

```bash
dnf install -y iscsi-initiator-utils device-mapper-multipath lvm2 xfsprogs lsscsi nmap-ncat fio
```

No data disk needs to be attached directly to this RHEL VM. Its new data disk will arrive through iSCSI.

## 8. Configure RHEL's four independent storage NICs

**Location: RHEL root shell.** Keep management connectivity on Adapter 1.

Set these variables to your actual interface names. Re-enter them if you open a new shell before finishing the NIC/interface-binding steps.

```bash
SAN_A1_NIC=ens192
SAN_B1_NIC=ens224
SAN_A2_NIC=ens256
SAN_B2_NIC=ens288
```

For VirtualBox, these might instead be `enp0s8`, `enp0s9`, `enp0s10` and `enp0s16`; use the MAC mapping, not the example order.

Check for existing connection profiles:

```bash
nmcli -f NAME,UUID,TYPE,DEVICE connection show
```

The next commands create profiles on **new, dedicated storage NICs**. If NetworkManager already created an automatic profile for one of those NICs, disable autoconnect on that specific old storage profile first:

```bash
nmcli connection modify "OLD_STORAGE_PROFILE_NAME" connection.autoconnect no
```

Substitute the exact old profile name; leave the management profile alone.

Create and activate the four storage profiles:

```bash
nmcli connection add type ethernet ifname "$SAN_A1_NIC" con-name san-a1 ipv4.method manual ipv4.addresses 192.168.110.20/24 ipv4.never-default yes ipv6.method disabled connection.autoconnect yes
nmcli connection add type ethernet ifname "$SAN_B1_NIC" con-name san-b1 ipv4.method manual ipv4.addresses 192.168.120.20/24 ipv4.never-default yes ipv6.method disabled connection.autoconnect yes
nmcli connection add type ethernet ifname "$SAN_A2_NIC" con-name san-a2 ipv4.method manual ipv4.addresses 192.168.111.20/24 ipv4.never-default yes ipv6.method disabled connection.autoconnect yes
nmcli connection add type ethernet ifname "$SAN_B2_NIC" con-name san-b2 ipv4.method manual ipv4.addresses 192.168.121.20/24 ipv4.never-default yes ipv6.method disabled connection.autoconnect yes

nmcli connection up san-a1
nmcli connection up san-b1
nmcli connection up san-a2
nmcli connection up san-b2

ip -br address
ip route
```

Expected: four directly connected storage routes and one IPv4 default route through `192.168.0.1` on the management interface. With the example VMware interface names, the routes should include the following; actual metrics and output order can differ:

```text
default via 192.168.0.1 dev ens160
192.168.0.0/24 dev ens160 src 192.168.0.81
192.168.110.0/24 dev ens192 src 192.168.110.20
192.168.120.0/24 dev ens224 src 192.168.120.20
192.168.111.0/24 dev ens256 src 192.168.111.20
192.168.121.0/24 dev ens288 src 192.168.121.20
```

No storage route should use `via 192.168.0.1`. No `bond0` is created for these storage interfaces.

## 9. Configure TrueNAS storage networking and pool

**Location: TrueNAS web UI.**

### 9.1 Set the four storage addresses

1. Open **Network → Interfaces**.
2. Match Adapter 2 to its physical interface by MAC address; choose **Edit**.
3. Turn off DHCP for that storage NIC.
4. Under aliases/IP addresses, add `192.168.110.10` with prefix length `24`.
5. Repeat for Adapter 3/B1 = `192.168.120.10/24`, Adapter 4/A2 = `192.168.111.10/24`, and Adapter 5/B2 = `192.168.121.10/24`.
6. Keep MTU 1500. Set no storage default gateway.
7. Apply the pending network changes, test connectivity, and save them permanently before the rollback timer expires.

### 9.2 Create the mirror pool

1. Open **Storage → Create Pool** or **Storage → Pools → Add**, according to the release's screen layout.
2. Pool name: `sanpool`.
3. Choose only the two empty 60 GiB data disks.
4. Create one **Mirror** data vdev with those disks.
5. Confirm the disk erasure for those two lab disks and create the pool.
6. Confirm that the pool is healthy. The OS boot disk must not be selected for this pool.

### 9.3 Create one zvol

1. Open **Datasets**, select `sanpool`, and choose **Add Zvol**.
2. Name: `rhel_lun01`.
3. Size: `20 GiB`.
4. For this small exercise, use a non-sparse/reserved zvol if the GUI offers the choice. Leave other storage properties at their normal defaults.
5. Save. Confirm the object is a **zvol**, not a regular dataset or SMB share.

The following screen labels can move between stable releases. The required objects are one pool, one zvol, one extent, one target, one portal group and one LUN association [S6, S7].

## 10. Configure one TrueNAS iSCSI target through four portals

**Location: TrueNAS web UI.** Set up the target before logging in from RHEL.

### 10.1 Global target basename

1. Open **Shares → Block Shares (iSCSI)** and its configuration screen.
2. Open **Global Target Configuration**, or the iSCSI service's edit/settings screen if that release puts the field there.
3. On this fresh lab installation, set basename to `iqn.2026-10.net.openhelp:truenas01`.
4. Save. This guide's target IQN will then be `iqn.2026-10.net.openhelp:truenas01:rhel-lun01`.

If you retain the default basename, record the resulting full IQN and use it in every RHEL target command. Do not change an existing production system's basename just to match an example.

### 10.2 Authorized Access / CHAP

1. Open **Authorized Access → Add**.
2. Group ID: `1`.
3. User: `rhel-san01`.
4. Secret: choose your own CHAP secret meeting the GUI's length requirements; use the same value later in RHEL. Common TrueNAS releases require 12–16 characters.
5. For one-way CHAP, leave peer user/secret empty.
6. Save. Do not put your real secret in this Markdown file or a Git repository.

CHAP authenticates the initiator; it does not encrypt the iSCSI data stream.

### 10.3 Initiator group

1. Open **Initiators → Add**.
2. Description: `rhel-san01-only`.
3. Disable **Allow All Initiators**, if selected.
4. Add initiator IQN `iqn.2026-10.net.openhelp:rhel-san01`.
5. Save and record the group ID.

### 10.4 One portal group with four listen addresses

1. Open **Portals → Add**.
2. Description: `rhel-four-path-portals`.
3. Add all four listen addresses, TCP port `3260`:

| Listen IP | Purpose |
|---|---|
| `192.168.110.10` | A1 |
| `192.168.120.10` | B1 |
| `192.168.111.10` | A2 |
| `192.168.121.10` | B2 |

4. Do not use `0.0.0.0` for this lab; the explicit addresses avoid listening on management.
5. If the release offers a separate **Discovery Authentication Method**, leave discovery authentication at **None** for this isolated lab. Session authentication on the target group will still be CHAP.
6. Save and record the portal group ID.

### 10.5 Device extent

1. Open **Extents → Add**.
2. Name: `rhel-lun01-extent`.
3. Type: **Device**.
4. Device: the zvol `sanpool/rhel_lun01`.
5. Keep the generated serial/NAA and the default logical block-size setting. Read-only should be off.
6. Save. Keep that serial/NAA stable; it contributes to the disk identity seen by clients.

### 10.6 Target and access group

1. Open **Targets → Add**.
2. Short name: `rhel-lun01`.
3. Mode: **iSCSI** where a mode selector is present.
4. Authorized networks: add `192.168.110.0/24`, `192.168.120.0/24`, `192.168.111.0/24` and `192.168.121.0/24`.
5. Add a target group using the portal group from 10.4 and initiator group from 10.3.
6. Authentication method: **CHAP**. Authentication/Authorized Access group: `1`.
7. Save and verify the full target IQN.

Discovery authentication and session authentication are different settings. A discovery listing can be visible while access to the actual LUN still requires CHAP and the correct initiator IQN.

### 10.7 Associate the target and extent

1. Open **Associated Targets → Add** or the target-to-extent association screen.
2. Target: `rhel-lun01`.
3. LUN ID: `0`.
4. Extent: `rhel-lun01-extent`.
5. Save.
6. Start the **iSCSI** service and enable its **Start Automatically** option.

Expected final inventory:

| Object | Count |
|---|---:|
| Target IQN | 1 |
| Portal group | 1 |
| Listen addresses in that group | 4 |
| Zvol / extent / LUN | 1 / 1 / 1 |
| RHEL initiator IQN allowed | 1 |

## 11. Verify the four IP paths before iSCSI login

**Location: RHEL root shell.**

```bash
ping -c 3 -I 192.168.110.20 192.168.110.10
ping -c 3 -I 192.168.120.20 192.168.120.10
ping -c 3 -I 192.168.111.20 192.168.111.10
ping -c 3 -I 192.168.121.20 192.168.121.10

ip route get 192.168.110.10
ip route get 192.168.120.10
ip route get 192.168.111.10
ip route get 192.168.121.10

nc -zv -w 3 192.168.110.10 3260
nc -zv -w 3 192.168.120.10 3260
nc -zv -w 3 192.168.111.10 3260
nc -zv -w 3 192.168.121.10 3260
```

Expected: each route uses its own storage NIC and source IP; TCP 3260 is reachable on all four target addresses.

Keep SELinux enforcing and keep the RHEL firewall enabled. RHEL is the initiator: it does not require a new inbound TCP 3260 listener rule. Target-side ACLs and any intermediate firewall must permit the actual sessions. Do not disable the firewall globally to hide a routing or portal problem.

## 12. Configure RHEL iSCSI identity, CHAP and interface bindings

**Location: RHEL root shell; fresh lab VM with no existing iSCSI workloads.**

### 12.1 Set the initiator identity

```bash
install -d -m 700 /root/san-lab-backup
cp -a /etc/iscsi/initiatorname.iscsi /root/san-lab-backup/initiatorname.iscsi.before-lab
cp -a /etc/iscsi/iscsid.conf /root/san-lab-backup/iscsid.conf.before-lab

printf '%s\n' 'InitiatorName=iqn.2026-10.net.openhelp:rhel-san01' > /etc/iscsi/initiatorname.iscsi
cat /etc/iscsi/initiatorname.iscsi
```

This IQN must exactly match the TrueNAS initiator ACL. Keep it unique if you clone the VM.

### 12.2 Set default session authentication for new node records

```bash
vi /etc/iscsi/iscsid.conf
```

Find and edit/uncomment the corresponding existing lines. Do not leave conflicting duplicate settings:

```ini
node.session.auth.authmethod = CHAP
node.session.auth.username = rhel-san01
node.session.auth.password = REPLACE_WITH_YOUR_TRUENAS_CHAP_SECRET
node.startup = manual
```

The password placeholder must be replaced in the local configuration file with the secret you chose in TrueNAS. The guide does not supply a reusable password. Leave discovery CHAP settings unset because discovery authentication was not enabled on the target.

```bash
chmod 600 /etc/iscsi/iscsid.conf
systemctl enable --now iscsid
systemctl enable iscsi
systemctl restart iscsid
```

`iscsid` runs the initiator daemon. Enabling `iscsi` supplies the automatic-session restore service for later reboots [S8].

### 12.3 Create four software-iSCSI iface records

Re-enter the four actual NIC names from Section 8 if you opened a new shell.

```bash
iscsiadm -m iface -I path-a1 --op=new
iscsiadm -m iface -I path-a1 --op=update -n iface.transport_name -v tcp
iscsiadm -m iface -I path-a1 --op=update -n iface.net_ifacename -v "$SAN_A1_NIC"

iscsiadm -m iface -I path-b1 --op=new
iscsiadm -m iface -I path-b1 --op=update -n iface.transport_name -v tcp
iscsiadm -m iface -I path-b1 --op=update -n iface.net_ifacename -v "$SAN_B1_NIC"

iscsiadm -m iface -I path-a2 --op=new
iscsiadm -m iface -I path-a2 --op=update -n iface.transport_name -v tcp
iscsiadm -m iface -I path-a2 --op=update -n iface.net_ifacename -v "$SAN_A2_NIC"

iscsiadm -m iface -I path-b2 --op=new
iscsiadm -m iface -I path-b2 --op=update -n iface.transport_name -v tcp
iscsiadm -m iface -I path-b2 --op=update -n iface.net_ifacename -v "$SAN_B2_NIC"

iscsiadm -m iface
```

Expected: four named iface records using transport `tcp` and four different `net_ifacename` values. If a `path-*` record already exists, inspect and update it instead of repeatedly creating it.

An iSCSI **iface** is an initiator binding record. It is not a Linux bond, an FC HBA or a new network interface. Software-iSCSI binding is documented by the upstream Open-iSCSI project [S9].

## 13. Create exactly four node records and log in

### 13.1 Set the target IQN

**Location: RHEL root shell.** Use the actual full IQN from TrueNAS:

```bash
TARGET_IQN='iqn.2026-10.net.openhelp:truenas01:rhel-lun01'
```

Optional read-only discovery, without creating extra node records:

```bash
iscsiadm -m discoverydb -t sendtargets -p 192.168.110.10:3260 -I path-a1 --op=nonpersistent --discover
```

Expected: the target IQN and possibly all four advertised portals. This command does not populate the node database. Discovery behaviour and static node creation are described in Open-iSCSI's documentation [S9].

### 13.2 Create the four selected portal/interface combinations

Static node records are used deliberately. Repeating ordinary discovery through four interfaces can create unwanted cross-interface records, while this lab needs exactly four selected sessions.

```bash
iscsiadm -m node --op=new -T "$TARGET_IQN" -p 192.168.110.10:3260 -I path-a1
iscsiadm -m node --op=new -T "$TARGET_IQN" -p 192.168.120.10:3260 -I path-b1
iscsiadm -m node --op=new -T "$TARGET_IQN" -p 192.168.111.10:3260 -I path-a2
iscsiadm -m node --op=new -T "$TARGET_IQN" -p 192.168.121.10:3260 -I path-b2
```

These new records inherit the CHAP settings you edited before creating them. Editing `iscsid.conf` later does not reliably update already-created node records; update those records explicitly if correcting credentials.

### 13.3 Prepare multipathing without claiming the OS disk

This starter configuration is **only for this new lab VM with a local OS disk**. It temporarily excludes every disk until the intended LUN's WWID is identified.

```bash
mpathconf --enable
cp -a /etc/multipath.conf /root/san-lab-backup/multipath.conf.before-lab

cat > /etc/multipath.conf <<'EOF'
defaults {
    find_multipaths strict
    user_friendly_names no
}
blacklist {
    wwid ".*"
}
EOF

multipath -t > /dev/null
systemctl enable --now multipathd
multipathd reconfigure
```

Do not replace the configuration of a running production host with this lab-only starting file.

### 13.4 Log in through the four paths

```bash
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.110.10:3260 -I path-a1 --login
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.120.10:3260 -I path-b1 --login
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.111.10:3260 -I path-a2 --login
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.121.10:3260 -I path-b2 --login

iscsiadm -m session
iscsiadm -m session -P 3
lsscsi -t
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL,WWN,MOUNTPOINTS
```

Expected: four sessions to the same target IQN, one through each portal. Each session exposes LUN 0. The kernel may show four new `/dev/sdX` path devices of 20 GiB each.

**Those four devices are four views of the same 20 GiB disk. They do not provide 80 GiB. Do not format any of them individually.**

## 14. Admit only the intended LUN into DM Multipath

### 14.1 Identify its four SCSI paths and their WWID

**Location: RHEL root shell.** Read `iscsiadm -m session -P 3` and record the four attached SCSI disk names for the target and LUN 0.

The following names are an example only. Replace them with the four actual new iSCSI path devices. These commands are read-only:

```bash
/usr/lib/udev/scsi_id --whitelisted --device=/dev/sdb
/usr/lib/udev/scsi_id --whitelisted --device=/dev/sdc
/usr/lib/udev/scsi_id --whitelisted --device=/dev/sdd
/usr/lib/udev/scsi_id --whitelisted --device=/dev/sde
```

**All four outputs must be the same non-empty WWID.** If they differ, stop here and correct the TrueNAS target/extent/LUN associations. Never force unrelated disks into one map by changing their identities.

Record the actual LUN identity, substituting a verified path device:

```bash
SAN_WWID=$(/usr/lib/udev/scsi_id --whitelisted --device=/dev/sdb)
printf 'Intended SAN LUN WWID: %s\n' "$SAN_WWID"
```

Continue only after visually verifying that it is non-empty and is the identity returned by all four target-LUN paths.

### 14.2 Create a LUN-specific configuration

The heredoc below intentionally expands the **verified** `SAN_WWID` variable into the configuration. It is not an example WWID to paste blindly.

```bash
cat > /etc/multipath.conf <<EOF
defaults {
    find_multipaths strict
    user_friendly_names no
}

blacklist {
    wwid ".*"
}

blacklist_exceptions {
    wwid "$SAN_WWID"
}

multipaths {
    multipath {
        wwid "$SAN_WWID"
        alias truenas_lun01
        path_grouping_policy failover
        path_selector "service-time 0"
        failback manual
        no_path_retry fail
    }
}
EOF

multipath -a "$SAN_WWID"
multipath -t > /dev/null
multipathd reconfigure
udevadm settle
multipath -ll truenas_lun01
multipathd show paths
```

Why these settings are used:

| Setting | Purpose in this lab |
|---|---|
| `find_multipaths strict` | Only accept a LUN registered in the WWID file |
| WWID blacklist plus exact exception | Keep this exercise limited to the intended data LUN |
| `alias truenas_lun01` | Give the map a readable name |
| `path_grouping_policy failover` | Use one path group at a time; make failover easy to observe |
| `failback manual` | Avoid an automatic switch back to an earlier path during the test |
| `no_path_retry fail` | Fail I/O rather than queue forever if every path disappears |

The blacklist/exception mechanism and validation command are documented by Red Hat [S10, S11]. These choices are a lab policy, not a generic replacement for an enterprise array's ALUA/path-policy requirements.

### 14.3 Expected storage inventory

| Linux object | Expected count | Size |
|---|---:|---:|
| Local OS disk | 1 | 40 GiB |
| Raw iSCSI SCSI path devices | 4 | Each reports the same 20 GiB LUN |
| Multipath map `truenas_lun01` | 1 | 20 GiB |

In `multipath -ll`, expect four path entries. Healthy paths normally include `ready` and `running`; the DM state `active` on a path does not, by itself, mean that every path is currently carrying application I/O. The selected **path group** and grouping policy determine that.

```mermaid
flowchart TB
    P1["Path A1: one SCSI device"]
    P2["Path B1: one SCSI device"]
    P3["Path A2: one SCSI device"]
    P4["Path B2: one SCSI device"]
    P1 --> M["One map: truenas_lun01"]
    P2 --> M
    P3 --> M
    P4 --> M
    M --> V["LVM volume"]
    V --> X["One XFS filesystem"]
    classDef a fill:#dcfce7,stroke:#15803d,color:#14532d;
    classDef b fill:#f3e8ff,stroke:#7e22ce,color:#581c87;
    classDef map fill:#dbeafe,stroke:#2563eb,color:#172554;
    classDef data fill:#ffedd5,stroke:#c2410c,color:#7c2d12;
    class P1,P3 a;
    class P2,P4 b;
    class M map;
    class V,X data;
```

## 15. Create LVM and a filesystem on the mapped LUN

**Location: RHEL root shell.** This section intentionally writes a PV, VG, LV and filesystem onto the **new, empty lab LUN**. Verify its identity and size before continuing. The target must not contain data you need.

### 15.1 Inspect before writing

```bash
multipath -ll truenas_lun01
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/mapper/truenas_lun01
wipefs --no-act /dev/mapper/truenas_lun01
```

Expected: the intended 20 GiB map; no existing filesystem/PV signatures on this new zvol. A signature on a supposedly empty LUN is a reason to investigate, not to add a force option.

### 15.2 Initialise only the multipath map

```bash
pvcreate /dev/mapper/truenas_lun01
vgcreate vg_sanlab /dev/mapper/truenas_lun01
lvcreate -L 10G -n lv_data vg_sanlab
mkfs.xfs /dev/vg_sanlab/lv_data
mkdir -p /srv/san-lab
mount /dev/vg_sanlab/lv_data /srv/san-lab

pvs -o pv_name,vg_name,pv_uuid
vgs
lvs
findmnt /srv/san-lab
df -hT /srv/san-lab
```

The PV should use the multipath map, not a single `/dev/sdX` component. Some tools display the mapper device's resolved name rather than its alias; verify its WWID/tree.

RHEL 9 may use the LVM devices file. `pvcreate` normally registers a new PV automatically. If an already-created PV is missing because its mapper device is absent from that file, inspect `lvmdevices`, then add only the correct mapped device as needed:

```bash
lvmdevices
```

Conditional repair for that specific situation:

```bash
lvmdevices --adddev /dev/mapper/truenas_lun01
pvs
```

Do not add the four component paths separately, and do not globally exclude the OS disk from LVM [S13].

### 15.3 Persist the filesystem mount

```bash
blkid /dev/vg_sanlab/lv_data
cp -a /etc/fstab /root/san-lab-backup/fstab.before-lab
vi /etc/fstab
```

Add one line, using the actual **filesystem UUID** from `blkid`:

```fstab
UUID=REPLACE_WITH_ACTUAL_FILESYSTEM_UUID /srv/san-lab xfs defaults,_netdev,x-systemd.requires=iscsi.service,x-systemd.requires=multipathd.service 0 0
```

The filesystem UUID, disk WWID and initiator IQN are three different identifiers. `_netdev` tells systemd this mount depends on network storage; the explicit service dependencies ensure the iSCSI and multipath services are started for it [S8, S14].

```bash
systemctl daemon-reload
findmnt --verify --verbose
```

When no application or test is using the mount, verify the fstab entry itself:

```bash
umount /srv/san-lab
mount /srv/san-lab
findmnt /srv/san-lab
```

Do not add `nofail` merely to conceal a storage problem. In a real application deployment, service ordering must also prevent the application from writing to an unmounted local directory.

## 16. Persist the four sessions and verify a reboot

**Location: RHEL root shell.** Re-enter the actual IQN if this is a new shell:

```bash
TARGET_IQN='iqn.2026-10.net.openhelp:truenas01:rhel-lun01'

iscsiadm -m node -T "$TARGET_IQN" -p 192.168.110.10:3260 -I path-a1 --op=update -n node.startup -v automatic
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.120.10:3260 -I path-b1 --op=update -n node.startup -v automatic
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.111.10:3260 -I path-a2 --op=update -n node.startup -v automatic
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.121.10:3260 -I path-b2 --op=update -n node.startup -v automatic

systemctl enable iscsid iscsi multipathd NetworkManager-wait-online.service
iscsiadm -m node -P 1
```

Verify there are only the four intended automatic node records for this target. Start the restore unit so its normal boot/stop ordering is established:

```bash
systemctl start iscsi
```

For this local-OS data-LUN lab, the root filesystem is not on iSCSI. Keep its boot configuration local. If multipath settings were already embedded in your initramfs, or your vendor requires their inclusion, regenerate the current kernel's image after the final configuration:

```bash
dracut --force --add multipath
```

This is not a SAN-boot setup; do not add iSCSI-root kernel parameters [S10].

Write a persistent test file:

```bash
printf 'TrueNAS four-path test\n' > /srv/san-lab/persistence-test.txt
sync
cat /srv/san-lab/persistence-test.txt
```

Leave TrueNAS running and reboot RHEL:

```bash
systemctl reboot
```

After reconnecting to RHEL management:

```bash
systemctl is-active iscsid multipathd
iscsiadm -m session
multipath -ll truenas_lun01
findmnt /srv/san-lab
cat /srv/san-lab/persistence-test.txt
```

Expected: four sessions restored, one map with four paths, filesystem mounted and file readable. `iscsi.service` is commonly an `active (exited)` restore unit; it is not a second long-running initiator daemon.

## 17. Test path failover with ongoing I/O

Use three SSH sessions to RHEL through its **management IP**. Restore all four paths and verify them before beginning each test. Never disable the management NIC to test a storage path.

### 17.1 Terminal 1 — observe the multipath map

```bash
watch -n 2 'multipath -ll truenas_lun01'
```

### 17.2 Terminal 2 — write to a dedicated file

```bash
fio --name=san-path-failover --filename=/srv/san-lab/fio-path-test.bin --size=256M --rw=write --bs=128k --direct=1 --ioengine=libaio --iodepth=4 --numjobs=1 --time_based --runtime=180 --group_reporting
```

This writes a file in the mounted test filesystem, not a raw disk. The test checks I/O continuity/error reporting; it is not a production performance benchmark or a comprehensive integrity test.

### 17.3 Terminal 3 — fail A1 only

```bash
nmcli connection down san-a1
```

Observe the path status. iSCSI failure detection is not necessarily immediate. Allow the negotiated transport/multipath timeouts to run; do not assume a fixed recovery time from a virtual lab.

Expected: A1 becomes unavailable; the map remains accessible through surviving paths. `fio` should complete with `err=0` after any temporary pause. Record the actual result rather than assuming success.

Restore A1:

```bash
nmcli connection up san-a1
iscsiadm -m session
multipath -ll truenas_lun01
```

The existing iSCSI session should recover. If its node is genuinely logged out rather than recovering, log in only that selected node again:

```bash
TARGET_IQN='iqn.2026-10.net.openhelp:truenas01:rhel-lun01'
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.110.10:3260 -I path-a1 --login
```

An “already present” session message means it is already logged in; do not repeatedly create new records.

### 17.4 Repeat for B1, A2 and B2

Test one profile at a time using the following pairs. Verify recovery to four paths before moving to the next pair:

```bash
nmcli connection down san-b1
nmcli connection up san-b1
```

```bash
nmcli connection down san-a2
nmcli connection up san-a2
```

```bash
nmcli connection down san-b2
nmcli connection up san-b2
```

Leave time between each down/up operation to observe failure and recovery; these are individual manual actions, not a block to run as an instant down/up loop.

### 17.5 Simulate loss of the A path group

After restoring four healthy paths, run a new `fio` test, then disable both A profiles:

```bash
nmcli connection down san-a1
nmcli connection down san-a2
```

Expected: the B1 and B2 sessions keep the LUN reachable. This is a logical approximation of losing Fabric A connectivity; it does not test a physical FC switch or the FC protocol.

Restore and verify:

```bash
nmcli connection up san-a1
nmcli connection up san-a2
iscsiadm -m session
multipath -ll truenas_lun01
```

Do not disconnect every path during the normal mounted-filesystem test. This lab deliberately uses `no_path_retry fail`, so loss of every path can return I/O errors to XFS and applications.

### 17.6 Optional integrity comparison

With all paths healthy, create a fixed file and record its checksum outside the SAN mount:

```bash
dd if=/dev/urandom of=/srv/san-lab/integrity-test.bin bs=1M count=32 conv=fsync status=progress
sha256sum /srv/san-lab/integrity-test.bin > /root/san-lab-integrity.sha256
```

Keep this file separate from the `fio` workload file. After the one-path and two-path tests, and again after a RHEL reboot:

```bash
sha256sum -c /root/san-lab-integrity.sha256
```

Expected: `OK`. This confirms that the fixed file remains readable with its original content; it is not proof against every possible storage failure.

## 18. Troubleshooting

| Symptom | Most likely checks |
|---|---|
| One storage IP cannot be reached | MAC-to-interface mapping, VMnet/internal-network names, cable-connected setting, `/24` mask |
| Ping works but TCP 3260 fails | iSCSI service, explicit portal listener, firewall between endpoints |
| Discovery sees a target but login is denied | Initiator IQN ACL, target group CHAP settings, secret in the selected node record |
| Login reports target not found | Wrong full IQN, target basename mismatch, target/extent association |
| Only one or two sessions | Missing node records, wrong iface binding, unreachable portal |
| Four sessions but different disk WWIDs | Different extents/LUNs were exported; correct TrueNAS mappings |
| Four path disks but no map | Intended WWID missing from exception or WWID file; inspect strict policy and logs |
| One map has four paths but writes use one | Expected with this lab's `failover` grouping policy |
| A recovered path is not carrying writes | `failback manual` does not automatically switch back; healthy standby paths are normal |
| Mount works until reboot | Automatic node flags, persistent NIC profiles, iSCSI service, multipath config, fstab UUID/dependencies |
| LVM reports duplicate PVs | Check that the PV uses the multipath map and inspect the devices file; do not initialise component paths |
| Four vNICs do not increase laptop throughput | Paths share the hypervisor, physical storage and CPU; this is a failover lab |
| `/sys/class/fc_host` is empty | Expected in the pure iSCSI VM lab; no FC HBAs were passed through |

Useful read-only commands on RHEL:

```bash
ip -br address
ip route
nmcli -f NAME,DEVICE,STATE connection show --active
iscsiadm -m iface
iscsiadm -m node -P 1
iscsiadm -m session -P 3
multipath -ll
multipathd show paths
multipath -t > /dev/null
lsscsi -t
lsblk -o NAME,SIZE,TYPE,FSTYPE,WWN,MOUNTPOINTS
journalctl -u iscsid -u iscsi -u multipathd -b --no-pager
journalctl -k -b --no-pager
```

To correct the CHAP secret for one existing node, use an interactive editor for that node's configuration through your normal administrative process, or update its individual setting with `iscsiadm`. Do not dump credentials into a support ticket. The following example reads the new secret without placing the literal value in shell history; the secret can still be briefly present in process arguments while the update runs:

```bash
TARGET_IQN='iqn.2026-10.net.openhelp:truenas01:rhel-lun01'
read -r -s -p 'CHAP secret: ' SAN_CHAP_SECRET
printf '\n'
iscsiadm -m node -T "$TARGET_IQN" -p 192.168.110.10:3260 -I path-a1 --op=update -n node.session.auth.password -v "$SAN_CHAP_SECRET"
unset SAN_CHAP_SECRET
```

Repeat the selected-node update for other affected records only. Changes are used for a new login; do not log out a mounted production LUN to test credentials.

## 19. Safe shutdown and startup of the lab

### 19.1 Shutdown

1. Stop `fio` and any process using `/srv/san-lab`.
2. On RHEL, flush and unmount:

```bash
sync
umount /srv/san-lab
findmnt /srv/san-lab
```

`findmnt` should return no mount for that path. If unmount reports “busy”, identify the user/process and stop it; do not force the unmount.

3. Shut down the RHEL VM through the OS:

```bash
systemctl poweroff
```

4. Wait until it is powered off.
5. Use the TrueNAS UI's **Shut Down** action and wait for TrueNAS to finish.
6. Then shut down the hypervisor host if required.

### 19.2 Startup

1. Start the hypervisor host and its virtual networks.
2. Start TrueNAS first. Confirm pool health and the iSCSI service.
3. Start RHEL.
4. Verify four sessions, one mapped LUN, the mount and the persistence/integrity files using Sections 16–17.

## 20. Route B: requirements for actual four-port FC

Use this route only when you actually have FC hardware and the target's supported FC capability. It is separate from the free iSCSI VM lab above.

| Requirement | What it means |
|---|---|
| Two host dual-port FC HBAs | Two independent cards, compatible drivers/firmware, four usable ports |
| Supported presentation to RHEL | Physical RHEL server, or a supported type-1/KVM PCI-passthrough design presenting the real HBA ports to the RHEL VM |
| FC target | Supported TrueNAS Enterprise FC configuration and licence, or another supported SAN array |
| FC target adapters | Suitable target-mode hardware on the storage side, separate from the RHEL initiator HBAs |
| Two fabrics | A/B switch networks with supported optics, speeds, cabling, firmware and ISLs |
| HA array for controller tests | A supported dual-controller platform; two ordinary TrueNAS CE VMs do not create this |

For a VM, ordinary emulated Ethernet cards cannot satisfy the HBA requirement. Actual passthrough requires physical hardware ownership, IOMMU support and a supported hypervisor/platform combination. Do not assume that adding adapters in VMware Workstation or VirtualBox creates PCI FC devices.

A TrueNAS disk-controller HBA used to attach local SAS disks is also a different role from an FC target adapter. Passing through a SAS disk HBA does not enable the licensed FC target feature.

### 20.1 Present the real HBAs to a RHEL VM

For a supported ESXi DirectPath or KVM/VFIO deployment:

1. Check the server, HBA and hypervisor interoperability requirements.
2. Enable hardware IOMMU/VT-d/AMD-Vi support as required by that platform.
3. Allocate the two complete FC HBA cards/functions to the RHEL VM according to the hypervisor's PCI-passthrough procedure. Both ports/functions of each chosen adapter must be accounted for.
4. Keep the host from using those same passed-through FC ports for its own mounted storage.
5. Boot RHEL and verify the actual PCI devices and four FC hosts as below.

The exact passthrough screens and IOMMU groups depend on the physical host. This guide does not claim a virtual-only substitute or invent a universal host-device assignment command.

For a **supported VMware ESXi/vSphere platform**, the concrete UI workflow is [S18]:

1. Shut down the RHEL VM and identify the physical HBAs by PCI address and adapter description. Confirm these adapters are dedicated to this VM and are not serving the ESXi host's mounted datastores.
2. In vSphere, select the ESXi host, then **Configure → Hardware → PCI Devices**. In the standalone Host Client, the equivalent location is usually **Host → Manage → Hardware → PCI Devices**.
3. Show all PCI devices, select the HBA functions to be assigned, and choose **Toggle Passthrough**. Account for both ports/functions of both cards; the number of PCI functions depends on the adapter.
4. Reboot the ESXi host if its UI reports that activation requires a reboot, following the host's normal maintenance procedure.
5. Edit the powered-off RHEL VM: **Virtual Hardware → Add New Device → PCI Device**. Add the selected supported HBA functions.
6. Set/reserve the VM's full configured guest memory as required by DirectPath I/O and confirm the devices are available for assignment.
7. Power on the VM. Verify four FC ports in RHEL rather than assuming that four configured virtual Ethernet NICs satisfy this requirement.

Directly assigned hardware ties the VM to those physical devices. Check that platform's supported migration/HA behaviour; do not assume an ordinary virtual-NIC failover model applies to passthrough HBAs.

### 20.2 Collect the four real host WWPNs

**Location: RHEL physical host or the RHEL VM after successful hardware passthrough.**

```bash
dnf install -y pciutils device-mapper-multipath lsscsi
lspci -nnk
ls /sys/class/fc_host/
grep -H . /sys/class/fc_host/host*/port_name
grep -H . /sys/class/fc_host/host*/node_name
grep -H . /sys/class/fc_host/host*/port_state
grep -H . /sys/class/fc_host/host*/speed
```

`port_name` is the local port's WWPN. `node_name` is its WWNN. The four host WWPNs are not the disk WWID. Record the actual values and the physical card/port labels; OS host numbers such as `host2` are discovery identifiers, not printed PCIe card labels [S12A].

## 21. FC switch cabling and zoning example

This section uses **four Cisco MDS switches** as a concrete example. The same separation of fabrics applies to other vendors, but their CLI is different. The example assumes new dedicated lab VSANs with no existing workloads. An established production SAN must incorporate the new zones into its existing active zoning configuration, not replace it with this small example.

### 21.1 Cable map

| Source | Destination | Switch port role |
|---|---|---|
| Host HBA1/P1, A1 | A-EDGE `fc1/1` | F port |
| Host HBA2/P1, A2 | A-EDGE `fc1/2` | F port |
| A-EDGE `fc1/47` | A-CORE `fc1/47` | E-port ISL inside Fabric A |
| SAN controller 1/A | A-CORE `fc1/10` | F port |
| SAN controller 2/A | A-CORE `fc1/11` | F port |
| Host HBA1/P2, B1 | B-EDGE `fc1/1` | F port |
| Host HBA2/P2, B2 | B-EDGE `fc1/2` | F port |
| B-EDGE `fc1/47` | B-CORE `fc1/47` | E-port ISL inside Fabric B |
| SAN controller 1/B | B-CORE `fc1/10` | F port |
| SAN controller 2/B | B-CORE `fc1/11` | F port |

Port numbers are examples, not a requirement to buy a switch with a specific number of ports. Use licensed, enabled ports that exist on your chosen model. Choose compatible FC optics/fibre and speeds; these are FC F/E ports, not Ethernet LACP ports.

The example uses VSAN `10` for Fabric A and VSAN `20` for Fabric B. These are FC fabric isolation identifiers; they are not Ethernet VLAN IDs or IP subnets.

### 21.2 Configure A-EDGE

**Location: A-EDGE Cisco MDS CLI.** The switch should be in its ordinary switching role, not an NPV-only configuration.

```text
configure terminal
vsan database
  vsan 10 name FABRIC_A
  vsan 10 interface fc1/1
  vsan 10 interface fc1/2
exit
interface fc1/1
  switchport mode F
  no shutdown
exit
interface fc1/2
  switchport mode F
  no shutdown
exit
interface fc1/47
  switchport mode E
  switchport trunk mode on
  switchport trunk allowed vsan 10
  no shutdown
exit
end
copy running-config startup-config
```

### 21.3 Configure A-CORE

**Location: A-CORE Cisco MDS CLI.**

```text
configure terminal
vsan database
  vsan 10 name FABRIC_A
  vsan 10 interface fc1/10
  vsan 10 interface fc1/11
exit
interface fc1/10
  switchport mode F
  no shutdown
exit
interface fc1/11
  switchport mode F
  no shutdown
exit
interface fc1/47
  switchport mode E
  switchport trunk mode on
  switchport trunk allowed vsan 10
  no shutdown
exit
end
copy running-config startup-config
```

### 21.4 Configure B-EDGE

**Location: B-EDGE Cisco MDS CLI.**

```text
configure terminal
vsan database
  vsan 20 name FABRIC_B
  vsan 20 interface fc1/1
  vsan 20 interface fc1/2
exit
interface fc1/1
  switchport mode F
  no shutdown
exit
interface fc1/2
  switchport mode F
  no shutdown
exit
interface fc1/47
  switchport mode E
  switchport trunk mode on
  switchport trunk allowed vsan 20
  no shutdown
exit
end
copy running-config startup-config
```

### 21.5 Configure B-CORE

**Location: B-CORE Cisco MDS CLI.**

```text
configure terminal
vsan database
  vsan 20 name FABRIC_B
  vsan 20 interface fc1/10
  vsan 20 interface fc1/11
exit
interface fc1/10
  switchport mode F
  no shutdown
exit
interface fc1/11
  switchport mode F
  no shutdown
exit
interface fc1/47
  switchport mode E
  switchport trunk mode on
  switchport trunk allowed vsan 20
  no shutdown
exit
end
copy running-config startup-config
```

Verify the ISL and fabric membership before zoning. Each switch in a fabric must have a valid unique domain ID; inspect and resolve any domain/VSAN/port-license conflict using the switch's release-specific procedure. An E/TE link must be operational and carry the intended VSAN [S15].

### 21.6 Example WWPN worksheet — replace every sample with real values

The sample strings below are illustrative identifiers, not values to deploy unchanged:

| Name | Example WWPN |
|---|---|
| Host A1, HBA1/P1 | `10:00:00:10:9b:11:01:01` |
| Host B1, HBA1/P2 | `10:00:00:10:9b:11:01:02` |
| Host A2, HBA2/P1 | `10:00:00:10:9b:22:02:01` |
| Host B2, HBA2/P2 | `10:00:00:10:9b:22:02:02` |
| Controller 1/A | `20:00:00:25:b5:aa:00:01` |
| Controller 1/B | `20:00:00:25:b5:aa:00:02` |
| Controller 2/A | `20:00:00:25:b5:bb:00:01` |
| Controller 2/B | `20:00:00:25:b5:bb:00:02` |

### 21.7 Fabric A zoning — two single-initiator zones

**Location: A-EDGE CLI, for the new isolated VSAN 10 only.** Replace all member WWPNs with your recorded values. Each zone below contains one host initiator port and the two target ports in Fabric A.

```text
configure terminal
zone name Z_A_HOST_A1 vsan 10
  member pwwn 10:00:00:10:9b:11:01:01
  member pwwn 20:00:00:25:b5:aa:00:01
  member pwwn 20:00:00:25:b5:bb:00:01
exit
zone name Z_A_HOST_A2 vsan 10
  member pwwn 10:00:00:10:9b:22:02:01
  member pwwn 20:00:00:25:b5:aa:00:01
  member pwwn 20:00:00:25:b5:bb:00:01
exit
zoneset name ZS_FABRIC_A vsan 10
  member Z_A_HOST_A1
  member Z_A_HOST_A2
exit
zoneset activate name ZS_FABRIC_A vsan 10
end
copy running-config startup-config
```

If that VSAN uses enhanced zoning, follow its zone-session/commit workflow as required by the installed NX-OS release. Do not mix a new basic-zoning procedure into a fabric whose operational zoning policy is different [S16].

### 21.8 Fabric B zoning

**Location: B-EDGE CLI, for the new isolated VSAN 20 only.**

```text
configure terminal
zone name Z_B_HOST_B1 vsan 20
  member pwwn 10:00:00:10:9b:11:01:02
  member pwwn 20:00:00:25:b5:aa:00:02
  member pwwn 20:00:00:25:b5:bb:00:02
exit
zone name Z_B_HOST_B2 vsan 20
  member pwwn 10:00:00:10:9b:22:02:02
  member pwwn 20:00:00:25:b5:aa:00:02
  member pwwn 20:00:00:25:b5:bb:00:02
exit
zoneset name ZS_FABRIC_B vsan 20
  member Z_B_HOST_B1
  member Z_B_HOST_B2
exit
zoneset activate name ZS_FABRIC_B vsan 20
end
copy running-config startup-config
```

### 21.9 Verify zoning and logins

On both Fabric A switches:

```text
show interface brief
show vsan
show fcdomain vsan 10
show flogi database vsan 10
show fcns database vsan 10
show zoneset active vsan 10
```

On both Fabric B switches:

```text
show interface brief
show vsan
show fcdomain vsan 20
show flogi database vsan 20
show fcns database vsan 20
show zoneset active vsan 20
```

Expected per fabric: two host initiator WWPNs and two target WWPNs in the FC name-server view, correct active zones, and an operational intra-fabric ISL. FLOGI tables show locally attached logins, so edge and core switches need not show identical local entries. Active zoning distributes through the fabric; verify it on both switches [S16].

## 22. Actual TrueNAS Enterprise FC target and LUN presentation

**Location: supported, FC-licensed TrueNAS Enterprise storage system. This section cannot be completed in the free CE VM lab.**

1. Verify the installed Enterprise licence includes FC and the platform/adapter combination is supported.
2. Verify target FC ports are connected as recorded in the worksheet, with each controller attached to both fabrics.
3. Create the intended pool and one zvol using the appliance's supported storage layout. On an HA appliance, configure the HA pair through its supported procedure; do not independently import the same pool into two ordinary VMs.
4. Open the block-share configuration and create one device extent for that zvol. Keep its generated serial/NAA stable.
5. Create a target with mode **Fibre Channel**; select **Both** only if you intentionally need both protocols and the vendor supports the access pattern.
6. Associate the extent with the target as one LUN. Use the same LUN identity across the selected target ports.
7. In the release's **Fibre Channel Ports** or wizard protocol-options screen, select/configure the FC target port mappings for the selected target. Depending on the supported release, the UI may use virtual FC target ports/NPIV. Record the **actual presented target WWPNs**; use those actual WWPNs in switch zoning.
8. If the appliance's supported HA setup requires ALUA, enable it in the global block-target configuration according to the TrueNAS Enterprise procedure.
9. Start the block-target service. The service may retain an iSCSI-labelled name in the UI even when the configured target mode is FC.
10. Confirm all four intended host initiator WWPNs are permitted by the target's supported access/mapping controls and the SAN zoning.
11. Recheck the FC switch name-server entries after target ports and any virtual target ports are created.

FC authorization does not use the iSCSI IQN/CHAP steps from Route A. Zoning and target-side LUN access must both be correct. TrueNAS release-specific FC screens differ, so the exact target-port mapping controls must be taken from the stable release's supported FC guide [S1].

## 23. Actual RHEL FC multipathing

This section assumes the real four FC ports are visible in RHEL, supported drivers/firmware are loaded, both fabrics are zoned and the same LUN is presented through both controllers. A single-controller target will not produce the reference eight-path result.

### 23.1 Initialise the host with the vendor's supported policy

**Location: RHEL root shell.** For a new FC host:

```bash
dnf install -y device-mapper-multipath lsscsi
mpathconf --enable
vi /etc/multipath.conf
```

Apply the actual array's RHEL configuration for path grouping, ALUA, retry/failure policy and timeouts. Keep its supported built-in hardware-table settings unless the vendor requires an override. Do not paste the single-controller iSCSI lab's `failover` policy over an enterprise array configuration.

For a new host, a normal defaults starting point is:

```conf
defaults {
    find_multipaths on
    user_friendly_names no
}
```

Merge this with the vendor's configuration and intended device exclusions; preserve an existing production host's settings.

```bash
multipath -t > /dev/null
systemctl enable --now multipathd
multipathd reconfigure
```

### 23.2 Scan only the four actual FC hosts

First identify the real host numbers:

```bash
ls /sys/class/fc_host/
grep -H . /sys/class/fc_host/host*/port_name
grep -H . /sys/class/fc_host/host*/port_state
```

The example below assumes the four FC ports are `host2`, `host3`, `host4` and `host5`. Substitute your real FC host numbers:

```bash
echo '- - -' > /sys/class/scsi_host/host2/scan
echo '- - -' > /sys/class/scsi_host/host3/scan
echo '- - -' > /sys/class/scsi_host/host4/scan
echo '- - -' > /sys/class/scsi_host/host5/scan
udevadm settle
multipath -ll
lsscsi -t
```

Scanning discovers paths. It does not authorise a host at the array or change a switch's zoning. Do not use a fabric-reset/`issue_lip` operation as a routine substitute for investigating a missing LUN [S12A].

### 23.3 Verify the actual path count

```bash
multipathd show paths
multipath -ll
lsblk -o NAME,SIZE,TYPE,WWN,FSTYPE,MOUNTPOINTS
```

For the exact reference layout in Section 2, expect:

| Item | Count |
|---|---:|
| Local FC initiator ports / host WWPNs | 4 |
| Visible FC target ports in Fabric A | 2 |
| Visible FC target ports in Fabric B | 2 |
| Paths to the presented LUN | 8 |
| Multipath map for that LUN | 1 |

Some paths can be active-optimized and others non-optimized on an ALUA array. Healthy non-optimized paths are not automatically faults. Confirm the expected vendor state and path count before deploying the application.

Use the mapper device or an LVM volume built on it for storage, following the vendor and application deployment procedure. For a new empty non-clustered test LUN, Section 15 illustrates the Linux PV/VG/LV/filesystem sequence; substitute the actual FC LUN map. Never initialise an existing production LUN from that example.

### 23.4 FC failover commissioning

Run controlled application I/O and inspect `multipath -ll` while testing one failure at a time:

1. Disable one **host-facing** FC switch port for the test server; restore and verify it.
2. Test loss of one host HBA's connectivity; restore and verify it.
3. Test the chosen Fabric A switch/ISL failure in a scheduled SAN test; verify Fabric B continues serving the LUN.
4. Use the storage vendor's supported controller-takeover procedure for controller failover.
5. Restore the complete configuration and verify all intended paths, application health and logs.

Changing a core target port or ISL can affect other hosts. Those actions require the actual SAN's commissioning/change procedure; a generic server-local link test is not permission to interrupt a shared bank fabric.

## 24. Interview-ready explanations

| Question | Concrete answer |
|---|---|
| Are FC HBA ports bonded? | No. In this FC design, DM Multipath manages storage paths; Ethernet bonding manages a different type of interface. |
| Two dual-port HBAs means how many host WWPNs? | Four FC ports have four port WWPNs. WWNNs are different identifiers. |
| Does four WWPNs mean four paths? | No. The host-to-target port combinations and LUN visibility determine path count. |
| Why are there two fabrics? | A failure in one fabric can leave connectivity through the other. |
| Why four FC switches in the expanded drawing? | Each fabric has an edge/core pair. Only switches within the same fabric have an ISL. |
| Does an extra switch double the path count? | No. A route through an additional FC switch is not another initiator-to-target SCSI path. |
| How can Linux show four 20 GiB disks but only 20 GiB usable? | The four SCSI devices are paths to one LUN. The mapper device represents that one disk. |
| Is WWID the same as WWPN? | No. WWID identifies the LUN; WWPN identifies an FC port. |
| Is IQN a virtual WWPN? | No. IQN is an iSCSI node name; the transport and login model differ. |
| Do we format every path? | No. Format the intended mapper device or an LV on it once. |
| Can the free VM lab test FC zoning? | No. It tests iSCSI session paths and Linux DM Multipath. |
| Is one TrueNAS VM HA because it has four NICs? | No. Paths can fail independently, but the VM/storage host is still shared. |
| Can both ordinary RHEL servers mount the same XFS LUN simultaneously? | Only one node may mount it in an ordinary active/passive design. Concurrent shared access needs a supported cluster/storage design with locking and fencing. |

## 25. Completion record

Fill this in while performing the lab. Blank entries are not claimed test results.

| Check | Expected | Observed / date |
|---|---|---|
| TrueNAS stable version | Record the installed release | |
| RHEL version | RHEL 9.x | |
| Management LAN / gateway | TrueNAS `192.168.0.80`, RHEL `192.168.0.81`, default gateway `192.168.0.1`, working LAN DNS | |
| Four dedicated storage networks | `192.168.110.0/24`, `192.168.120.0/24`, `192.168.111.0/24`, `192.168.121.0/24`; correct adapter/MAC/IP mapping, no storage gateways | |
| Target objects | One target, one extent, one LUN, four portals | |
| CHAP login | Correct credentials and initiator ACL | |
| iSCSI sessions | Four selected sessions | |
| LUN WWIDs | All four component paths match | |
| Mapper device | One 20 GiB map with four paths | |
| LVM and XFS | Mapped LUN used once | |
| RHEL reboot | Sessions/map/mount restored | |
| A1 failure and recovery | I/O completes, path returns | |
| B1 failure and recovery | I/O completes, path returns | |
| A2 failure and recovery | I/O completes, path returns | |
| B2 failure and recovery | I/O completes, path returns | |
| Both A paths unavailable | B paths retain access | |
| Fixed-file checksum after reboot | OK | |
| Safe shutdown/startup | RHEL before TrueNAS on shutdown; TrueNAS first on startup | |

For actual FC commissioning, record all four real host WWPNs, all visible target WWPNs, active zone sets, the LUN WWID, supported path count and measured failure results separately.

## 26. Official sources and document scope

Sources were checked on 22 October 2026. Select the stable product version in vendor documentation; some unversioned TrueNAS URLs currently show development documentation. The practical lab is written for the 25.10 generation and RHEL 9; minor UI labels and hardware-specific procedures can differ.

- **[S1] TrueNAS FC support and setup:** [TrueNAS 25.10: Setting Up Fibre Channel](https://www.truenas.com/docs/scale/25.10/scaletutorials/shares/iscsi/settingupfibrechannel/) and [TrueNAS target API: FC requires Enterprise licensing and suitable hardware](https://api.truenas.com/v27.0/api_methods_iscsi.target.create.html).
- **[S2] TrueNAS release selection:** [TrueNAS Software Status](https://www.truenas.com/docs/softwarestatus/).
- **[S3] TrueNAS VM installation and sizing:** [Installing TrueNAS](https://www.truenas.com/docs/scale/gettingstarted/install/installingscale/).
- **[S4] TrueNAS networking:** [Networking Recommendations](https://www.truenas.com/docs/solutions/optimizations/networking/), including MPIO and interface subnet guidance.
- **[S5] Production FC redundancy:** [NetApp dual-fabric zoning recommendations](https://docs.netapp.com/us-en/ontap/san-config/fc-fcoe-recommended-zoning-configuration.html) and [Dell example with two dual-port host HBAs](https://infohub.delltechnologies.com/en-us/l/oracle-database-19c-best-practices-on-dell-poweredge-servers-and-powerstore-t-model-arrays/esxi-host-connectivity/).
- **[S6] TrueNAS target objects:** [TrueNAS 25.10: Adding iSCSI Block Shares](https://www.truenas.com/docs/scale/25.10/scaletutorials/shares/iscsi/addingiscsishares/).
- **[S7] Stable-generation UI reference:** [TrueNAS 25.10 tutorials](https://www.truenas.com/docs/scale/25.10/scaletutorials/scaletutorialsprint/).
- **[S8] RHEL initiator setup:** [RHEL 9: Configuring an iSCSI initiator](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_storage_devices/configuring-an-iscsi-initiator_managing-storage-devices).
- **[S9] Upstream interface/node semantics:** [Open-iSCSI README](https://github.com/open-iscsi/open-iscsi/blob/master/README) and [iscsiadm source manual](https://github.com/open-iscsi/open-iscsi/blob/master/usr/iscsiadm.8).
- **[S10] RHEL multipath setup:** [RHEL 9: Configuring DM Multipath](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_device_mapper_multipath/configuring-dm-multipath_configuring-device-mapper-multipath).
- **[S11] Intended-device selection:** [RHEL 9: Preventing devices from multipathing](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_device_mapper_multipath/preventing-devices-from-multipathing_configuring-device-mapper-multipath).
- **[S12] VirtualBox adapters:** [VirtualBox command-line networking overview](https://www.oracle.com/technical-resources/articles/it-infrastructure/admin-manage-vbox-cli.html) and [VirtualBox 7.2 User Guide](https://docs.oracle.com/en/virtualization/virtualbox/7.2/user/EN-VBOX-7-2-USER.pdf).
- **[S12A] RHEL FC devices:** [RHEL 9: Using Fibre Channel devices](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_storage_devices/using-fibre-channel-devices_managing-storage-devices).
- **[S13] LVM device selection:** [RHEL 9: Limiting LVM device visibility and usage](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/limiting-lvm-device-visibility-and-usage_configuring-and-managing-logical-volumes).
- **[S14] Mount ordering:** [systemd.mount manual](https://www.freedesktop.org/software/systemd/man/latest/systemd.mount.html).
- **[S15] Cisco MDS ISLs/VSAN trunks:** [MDS 9.x: Configuring Trunking](https://www.cisco.com/c/en/us/td/docs/dcn/mds9000/sw/9x/configuration/interfaces/cisco-mds-9000-nx-os-interfaces-configuration-guide-9x/configuring_trunking.html).
- **[S16] Cisco MDS zoning:** [MDS 9.x: Configuring and Managing Zones](https://www.cisco.com/c/en/us/td/docs/dcn/mds9000/sw/9x/configuration/fabric/cisco-mds-9000-nx-os-fabric-configuration-guide-9x/configuring_and_managing_zones.html).
- **[S17] Shared storage and fencing:** [RHEL 9 HA overview](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_and_managing_high_availability_clusters/assembly_overview-of-high-availability-configuring-and-managing-high-availability-clusters) and [Red Hat fencing requirements](https://access.redhat.com/articles/2881341).
- **[S18] Actual hardware assignment to a VM:** [Broadcom: Configuring VMDirectPath I/O passthrough](https://knowledge.broadcom.com/external/article/309986/configuring-vmdirectpath-io-passthrough.html) and [VMDirectPath platform/device requirements](https://knowledge.broadcom.com/external/article/312208/).

This is an original learning procedure and production reference, not a record of a particular bank's deployed design or a vendor-certified implementation. Real FC support, controller recovery policy, switch licensing, firmware compatibility and application clustering must match the chosen equipment. Keep all credentials and actual production inventory out of a public Git repository.
