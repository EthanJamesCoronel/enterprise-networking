

#### Overview

This lab simulates a simplified enterprise campus network consisting of two routers, two distribution-layer switches, and two access-layer switches with multiple VLANs for various departments. The objective was to create a Layer 2 and Layer 3 network architecture that implements robust network security and high availability.

The lab was built in Cisco Modeling Labs (CML) for the purpose of gaining hands-on experience with campus network design, network security and troubleshooting.


#### Topology

![[network_topology.png]]

The lab topology consists of the following:

1. Two access-layer switches (`ACC1` and `ACC2`)

2. Two distribution-layer switches (`DIS1` and `DIS2`)

3. Two routers providing first-hop redundancy (`R1` and `R2`)

4. One area border routers (ABR) implementing a multi-area OSPF domain (`ABR1`)

5. One autonomous system border router (ASBR) providing Internet connectivity (`ASBR1`)


#### Assumptions

1. The enterprise campus network consists of four departments: Finance, Marketing, HR and IT.

2. The network uses IPv4 exclusively.

3. Area 0 extends beyond the scope of this lab.

4. Internet connectivity is provided by a single ISP.

5. Open-standard protocols are used as much as possible for interoperability with non-Cisco network devices.


#### Design Objectives

#### Layer 2

- Configure VLAN segmentation.

- Implement Layer 2 scalability using MSTP.

- Aggregate redundant uplinks using EtherChannel.

#### Layer 3

- Configure inter-VLAN routing.

- Implement dynamic routing using OSPF.

- Provide default gateway redundancy using VRRP.

- Enable Internet connectivity using NAT/PAT.

#### Network Security

- Secure the Layer 2 and Layer 3 infrastructure.

- Harden network devices.

#### Management Plane

- Configure secure local device authentication.

- Facilitate secure remote device management using SSH.

- Restrict management access only to authorized hosts using ACLs.

- Implement centralized logging using Syslog.

- Enable remote monitoring of device health and performance using SNMP.

- Simplify neighbor discovery and troubleshooting with CDP and LLDP.

#### Infrastructure Services

- Facilitate automatic network configuration in end hosts using DHCP.

- Enable automatic time synchronization using NTP.


### VLAN and IP Addressing Plan

| VLAN | Department   | Subnet             | Default Gateway |
| ---- | ------------ | ------------------ | --------------- |
| 10   | Finance      | `192.168.10.0/24`  | `192.168.10.1`  |
| 20   | Marketing    | `192.168.20.0/24`  | `192.168.20.1`  |
| 30   | HR           | `192.168.30.0/24`  | `192.168.30.1`  |
| 40   | IT           | `192.168.40.0/24`  | `192.168.40.1`  |
| 50   | Printers     | `192.168.50.0/24`  | `192.168.50.1`  |
| 100  | Management   | `192.168.100.0/24` | `192.168.100.1` |
| 2063 | Unused Ports | -                  | -               |
| 2151 | Native VLAN  | -                  | -               |

`/32` router loopback addresses were also assigned the `192.168.254.0/24` subnet.



### Technologies Used

- **Layer 2**: VLANs, MSTP, EtherChannel (LACP)

- **Layer 3**: VRRP, OSPF

- **Management Plane**: SSH, Syslog, SNMP, CDP, LLDP

- **Network Security**: DHCP Snooping, DAI, ACLs, HMAC Authentication

- **Infrastructure Services**: DHCP, NAT/PAT, NTP



### Layer 2 Design


#### VLAN Configuration

Six VLANs were created to logically segment the enterprise campus into four departments: VLAN 10 (Finance), VLAN 20 (Marketing), VLAN 30 (HR) and VLAN 40 (IT). VLAN 50 was created as a dedicated printer VLAN as a security best practice to isolate network printers from user workstations, limiting opportunities for lateral movement in the event that a printer is compromised. VLAN 100 (Management) was also created to provide dedicated VLANs for printers and remote device management.  Each VLAN was assigned a descriptive name that corresponds to its associated department or function. 

All unused switchports were placed in VLAN 2063 (Unused Ports) and administratively shut down.

In addition, VTP was disabled on all switches because VLANs were intentionally managed locally on each switch, eliminating the risk of unintended VLAN database modifications caused by VTP advertisements while reducing the switch's attack surface.

<details>
	
<summary>DIS1, DIS2, ACC1 and ACC2</summary>


```cisco
vlan 10
 name FINANCE
vlan 20
 name MARKETING
vlan 30
 name HR
vlan 40
 name IT
vlan 50
 name PRINTERS
vlan 100
 name MANAGEMENT_VLAN
vlan 2063
 name UNUSED_PORTS

vtp mode off
```
</details>


#### Interfaces and EtherChannel Configuration

Access ports were configured on `ACC1` and `ACC2` to connect end-user hosts such as workstations, laptops and printers to the network. PortFast and BPDU Guard were enabled globally so that all interfaces operating as access ports are protected by default. 

Because BPDU Guard was enabled only on PortFast-enabled interfaces, PortFast was explicitly disabled on all but one trunk port to ensure that inter-switch links continued participating in Spanning Tree operations without being subject to BPDU Guard. One host-facing interface in `ACC2` was configured as a PortFast-enabled trunk port connected to a VMware ESXi host where the NMS, Syslog server, and HP JetAdvantage printer server are hosted.

The trunk links were also manually configured to permit only the VLANs required by the design, preventing reliance on the default behavior of forwarding all VLANs. Automatic trunk negotiation via DTP was disabled to reduce the attack surface of the switches and prevent unauthorized trunk formation.

**Note**: The IOL images of CML-Free require the `switchport trunk encapsulation dot1q` command for the interface to become a trunk port. In modern Catalyst switches, this command is no longer necessary because ISL is no longer supported.

EtherChannel was implemented using the Link Aggregation Control Protocol (LACP) on all access-distribution uplinks. Two physical interfaces were bundled on each port channel uplink to provide both redundancy and increased bandwidth. Fast LACP timers were configured to enable rapid detection of failed member links and accelerate convergence time.

LACP was selected over the Port Aggregation Protocol (PAgP) because it is an IEEE standard and supports interoperability between Cisco switches and non-Cisco switches. Although this lab uses Cisco devices exclusively, the use of LACP reflects enterprise network environments where multi-vendor network infrastructures may be present.

<details>
	
<summary>ACC1</summary>

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default

interface range Ethernet0/0-1
 description Link to DIS1
 channel-group 1 mode active
 
interface range Ethernet0/2-3
 description Link to DIS2
 channel-group 2 mode active
 
interface Port-channel1
 description LACP bundle to DIS1

interface Port-channel2
 description LACP bundle to DIS2
 
interface range Port-channel1-2
 no spanning-tree portfast
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,100
 switchport native vlan 2151
 switchport nonegotiate

interface range Ethernet1/0-1
 description Link to VLAN 10 (Finance) Hosts
 switchport mode access
 switchport access vlan 10
 
interface range Ethernet1/2-3
 description Link to VLAN 20 (Marketing) Hosts
 switchport mode access
 switchport access vlan 20

interface Vlan100
 ip address 192.168.100.6 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
```
</details>

<details>

<summary>ACC2</summary>

```cisco
spanning-tree portfast default
spanning-tree portfast bpduguard default

interface range Ethernet0/0-1
 description Link to DIS2
 channel-group 1 mode active
 
interface range Ethernet0/2-3
 description Link to DIS1
 channel-group 2 mode active

interface Port-channel1
 description LACP bundle to DIS2

interface Port-channel2
 description LACP bundle to DIS1
 
interface range Port-channel1-2
 no spanning-tree portfast
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,100
 switchport native vlan 2151
 switchport nonegotiate
 
interface Ethernet1/0
 description Link to VLAN 30 (HR) Hosts
 switchport mode access
 switchport access vlan 30
 
interface Ethernet1/1
 description UNUSED PORT
 switchport mode access
 switchport access vlan 2063
 shutdown
 
interface Ethernet1/2
 description Link to VLAN 40 (IT) Hosts
 switchport mode access
 switchport access vlan 40

interface Ethernet1/3
 description VMware ESXi (NMS, Syslog, HP JetAdvantage)
 switchport trunk encapsulation dot1q
 switchport mode trunk 
 switchport trunk allowed vlan 40,100
 switchport native vlan 2151
 spanning-tree portfast trunk
 switchport nonegotiate

interface Vlan100
 ip address 192.168.100.7 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
```
</details>

<details>
	<summary>DIS1</summary>
	
```cisco
interface range Ethernet0/0-1
 description Link to ACC1
 channel-group 1 mode active
 
interface range Ethernet0/2-3
 description Link to ACC2
 channel-group 2 mode active
 
interface Port-channel1
 description LACP bundle to ACC1
 
interface Port-channel2
 description LACP bundle to ACC2
 
interface range Port-channel1-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,100
 switchport native vlan 2151
 switchport nonegotiate

interface Ethernet1/0
 description Link to R1
interface Ethernet1/1
 description Link to DIS2
 
interface range Ethernet1/0-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,100
 switchport native vlan 2151
 switchport nonegotiate

interface range Ethernet1/2-3
 description UNUSED PORTS
 switchport mode access
 switchport access vlan 2063
 shutdown

interface Vlan100
 ip address 192.168.100.4 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
```
	
</details>

<details>

<summary>DIS2</summary>
	
```cisco
interface range Ethernet0/0-1
 description Link to ACC2
 channel-group 1 mode active
 
interface range Ethernet0/2-3
 description Link to ACC1
 channel-group 2 mode active
 
interface Port-channel1
 description LACP bundle to ACC2

interface Port-channel2
 description LACP bundle to ACC1
 
interface range Port-channel1-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,100
 switchport native vlan 2151
 switchport nonegotiate
 
interface Vlan100
 ip address 192.168.100.5 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
```
</details>


#### MSTP Configuration

Although only four departmental VLANs were configured in this lab, the Multiple Spanning Tree Protocol (MSTP) was chosen to simulate the design of a larger enterprise campus network. MSTP enables multiple VLANs to be mapped to a single Spanning Tree instance, improving scalability in network environments with tens, hundreds or even thousands of VLANs.

In contrast, RPVST+ maintains a separate Spanning Tree instance for each VLAN, which results in higher CPU, memory and control plane overhead as the number of VLANs in the network grows. 

Although the scalability benefits are negligible in this scenario, the lab demonstrates VLAN-to-instance mapping and traffic engineering by assigning different root bridges to separate MST instances.

<details>
<summary>DIS1</summary>
	
```cisco
spanning-tree mode mst
spanning-tree mst 1 priority 4096
spanning-tree mst 2 priority 8192

spanning-tree mst configuration
 name LAB1-ENTERPRISE
 revision 1
 instance 1 vlan 10,20,50
 instance 2 vlan 30,40,100
```
</details>

<details>
<summary>DIS2</summary>
	
```cisco
spanning-tree mode mst
spanning-tree mst 1 priority 8192
spanning-tree mst 2 priority 4096

spanning-tree mst configuration
 name LAB1-ENTERPRISE
 revision 1
 instance 1 vlan 10,20,50
 instance 2 vlan 30,40,100
```
</details>

<details>
<summary>ACC1 and ACC2</summary>

```cisco
spanning-tree mode mst

spanning-tree mst configuration
 name LAB1-ENTERPRISE
 revision 1
 instance 1 vlan 10,20,50
 instance 2 vlan 30,40,100
```
</details>


---
### Layer 3 Design


#### Interface Configuration

Operationally relevant interface descriptions were configured to simplify management and troubleshooting by clearly identifying the purpose of each interface. All unused router interfaces were left administratively disabled for security purposes.

Point-to-point links between routers will use the `192.168.255.0/24` address space but be configured with `/31` IPv4 addresses for the purpose of IPv4 address conservation.

<details>
<summary>R1</summary>
	
```cisco
interface range Ethernet0/0-2
 no shutdown

interface Ethernet0/1
 description Link to ABR1 (3.3.3.3)
 ip address 192.168.255.0 255.255.255.254

interface Ethernet0/2
 description Link to R2 (2.2.2.2)
 ip address 192.168.255.4 255.255.255.254

interface Ethernet0/3
 description UNUSED PORT

interface Loopback0
 ip address 192.168.254.3 255.255.255.255
```
</details>

<details>
<summary>R2</summary>
	
```cisco
interface range Ethernet0/0-2
 no shutdown

interface Ethernet0/1
 description Link to ABR1 (3.3.3.3)
 ip address 192.168.255.2 255.255.255.254

interface Ethernet0/2
 description Link to R1 (1.1.1.1)
 ip address 192.168.255.5 255.255.255.254

interface Ethernet0/3
 description UNUSED PORT

interface Loopback0
 ip address 192.168.254.4 255.255.255.255
```
</details>




<details>
<summary>ABR1</summary>
	
```cisco
## ABR1

interface range Ethernet0/0-2
 no shutdown

interface Ethernet0/0
 description Link to R1 (1.1.1.1)
 ip address 192.168.255.1 255.255.255.254

interface Ethernet0/1
 description Link to R2 (2.2.2.2)
 ip address 192.168.255.3 255.255.255.254

interface Ethernet0/2
 description Link to ABR2 (4.4.4.4)
 ip address 192.168.255.6 255.255.255.254

interface Loopback0
 ip address 192.168.254.2 255.255.255.255
```
</details>


<details>
<summary>ASBR1</summary>

```cisco
## ASBR1

interface range Ethernet0/0-1
 no shutdown

interface Ethernet0/0
 description Link to ABR1 (3.3.3.3)
 ip address 192.168.255.7 255.255.255.254

interface Ethernet0/1
 description Link to ISP
 ip address 203.0.113.2 255.255.255.0

interface range Ethernet0/2-3
 description UNUSED PORT

interface Loopback0
 ip address 192.168.254.1 255.255.255.255
```
</details>


#### VRRP

Virtual Router Redundancy Protocol (VRRP) was configured on `R1` and `R2` to provide default gateway redundancy and load sharing for all VLANs. `R1` operates as the active router for VLANs 10, 30 and 50, while `R2` operates as the active router for VLANs 20, 40 and 100. This distributes default gateway traffic across both routers under normal operating conditions. Object tracking via IP SLA was considered but could not be implemented because of feature limitations in the IOL images of CML-Free. VRRP advertisement timers were also decreased from the default value of one (1) second to two hundred fifty (250) milliseconds to speed up gateway failover.

**Note**: Because the IOL images of CML-Free only support legacy plaintext authentication for VRRP, it was not configured on both routers as it provides negligible security benefits.


<details>
<summary>R1</summary>

```cisco
interface Ethernet0/0.10
 description Link to VLAN 10 (Finance)
 encapsulation dot1q 10
 ip address 192.168.10.2 255.255.255.0
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 110
 vrrp 10 preempt
 vrrp 10 timers advertise 250

interface Ethernet0/0.20
 description Link to VLAN 20 (Marketing)
 encapsulation dot1q 20
 ip address 192.168.20.2 255.255.255.0
 vrrp 20 ip 192.168.20.1
 vrrp 20 priority 90
 vrrp 20 preempt
 vrrp 20 timers advertise 250
  
interface Ethernet0/0.30
 description Link to VLAN 30 (HR)
 encapsulation dot1q 30
 ip address 192.168.30.2 255.255.255.0
 vrrp 30 ip 192.168.30.1
 vrrp 30 priority 110
 vrrp 30 preempt
 vrrp 30 timers advertise 250

interface Ethernet0/0.40
 description Link to VLAN 40 (IT)
 encapsulation dot1q 40
 ip address 192.168.40.2 255.255.255.0
 vrrp 40 ip 192.168.40.1
 vrrp 40 priority 90
 vrrp 40 preempt
 vrrp 40 timers advertise 250

interface Ethernet0/0.50
 description Link to VLAN 50 (Printers)
 encapsulation dot1q 50
 ip address 192.168.50.2 255.255.255.0
 vrrp 50 ip 192.168.50.1 255.255.255.0
 vrrp 50 priority 110
 vrrp 50 preempt
 vrrp 50 timers advertise 250

interface Ethernet0/0.100
 description Management VLAN
 encapsulation dot1q 100
 ip address 192.168.100.2 255.255.255.0
 vrrp 100 ip 192.168.100.1
 vrrp 100 priority 90
 vrrp 100 preempt
 vrrp 100 timers advertise 250
```
</details>


<details>
<summary>R2</summary>

```cisco
interface Ethernet0/0.10
 description Link to VLAN 10 (Finance)
 encapsulation dot1q 10
 ip address 192.168.10.3 255.255.255.0
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 90
 vrrp 10 preempt
 vrrp 10 timers advertise 250

interface Ethernet0/0.20
 description Link to VLAN 20 (Marketing)
 encapsulation dot1q 20
 ip address 192.168.20.3 255.255.255.0
 vrrp 20 ip 192.168.20.1
 vrrp 20 priority 110
 vrrp 20 preempt
 vrrp 20 timers advertise 250

interface Ethernet0/0.30
 description Link to VLAN 30 (HR)
 encapsulation dot1q 30
 ip address 192.168.30.3 255.255.255.0
 vrrp 30 ip 192.168.30.1
 vrrp 30 priority 90
 vrrp 30 preempt
 vrrp 30 timers advertise 250

interface Ethernet0/0.40
 description Link to VLAN 40 (IT)
 encapsulation dot1q 40
 ip address 192.168.40.3 255.255.255.0
 vrrp 40 ip 192.168.40.1
 vrrp 40 priority 110
 vrrp 40 preempt
 vrrp 40 timers advertise 250

interface Ethernet0/0.50
 description Link to VLAN 50 (Printers)
 encapsulation dot1q 50
 ip address 192.168.50.3 255.255.255.0
 vrrp 50 ip 192.168.50.1 255.255.255.0
 vrrp 50 priority 90
 vrrp 50 preempt
 vrrp 50 timers advertise 250
 
interface Ethernet0/0.100
 description Management VLAN
 encapsulation dot1q 100
 ip address 192.168.100.3 255.255.255.0
 vrrp 100 ip 192.168.100.1
 vrrp 100 priority 110
 vrrp 100 preempt
 vrrp 100 timers advertise 250
```
</details>


#### OSPF

Open Shortest Path First (OSPF) was configured on all routers to facilitate dynamic routing. 

This lab involves a highly simplified OSPF domain with just two areas while abstracting the rest away. Area 0 serves as the backbone area and is where `ASBR1` resides. `ABR1`, which sits between Area 0 and Area 1, is directly connected to `ASBR1`.

OSPF was enabled at the interface level (`ip ospf <pid> area <area-id>`) on all interfaces facing routers and loopback interfaces instead of the traditional `network` command to ensure that only the IP address configured on the specific interface would participate in OSPF operations. 

Additional features and optimizations were implemented to ensure high availability, security and optimization of OSPF operations.

1. **Fast Failure Detection**. All OSPF-enabled interfaces within both areas were configured with a Hello timer of one (1) second. Instead of specifying the Dead timer in seconds, a “multiplier” was configured instead, allowing routers to detect neighbor failures within approximately two hundred fifty (250) milliseconds. This enables rapid convergence. 

2. **Authentication**. Because null authentication provides no security, and Type 1 authentication exposes passwords as plaintext strings, Type 2 authentication via HMAC-SHA-512 was implemented in all OSPF-enabled interfaces. For this lab, static per-area keys were chosen for the following reasons:

	**Simplicity**. While per-link keys and periodic key rotation offer robust security, they also add administrative overhead. This lab prioritizes ease of configuration and troubleshooting.
	  
	**Operational reality**. Most production networks use a single key per area or for the entire OSPF domain. In high-security network environments, network engineers may opt to configure different keys per link. It is also best practice to implement key rotation by configuring multiple keys per keychain and setting their send and accept lifetimes.

3. **TTL Security**. Generalized TTL Security Mechanism (GTSM) was enabled on all OSPF-enabled interfaces to ensure that OSPF packets are accepted only from directly connected neighbors, providing an additional layer of protection to the OSPF domain by thwarting spoofed OSPF messages from multiple hops away.

4. **Optimization**. The network type of all OSPF-enabled interfaces was changed to `point-to-point` to eliminate Type 2 LSA overhead, conserving processing resources and bandwidth. Because Ethernet uses the `broadcast` network type, DR/BDR elections would take place even in physical point-to-point links, causing Type 2 LSAs to be flooded out of active interfaces unnecessarily. Changing the network type to `point-to-point` suppresses DR/BDR elections on Ethernet links.


<details>
<summary>R1</summary>
	
```cisco
## R1 

key chain OSPF_AREA1_AUTH
 key 1
  key-string LAB-OSPF-AREA1-PASSWORD
  cryptographic-algorithm hmac-sha-512

router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2

interface Ethernet0/0.10
 ip ospf 1 area 1
interface Ethernet0/0.20
 ip ospf 1 area 1
interface Ethernet0/0.30
 ip ospf 1 area 1
interface Ethernet0/0.40
 ip ospf 1 area 1
interface Ethernet0/0.50
 ip ospf 1 area 1
interface Ethernet0/0.100
 ip ospf 1 area 1

interface range Ethernet0/1-2
 ip ospf hello-interval 1
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf authentication key-chain OSPF_AREA1_AUTH
 ip ospf ttl-security hops 1
 ip ospf network point-to-point
 ip ospf 1 area 1
 
interface Loopback0
 ip ospf 1 area 1
```
</details>


<details>
<summary>R2</summary>

```cisco
key chain OSPF_AREA1_AUTH
 key 1
  key-string LAB-OSPF-AREA1-PASSWORD
  cryptographic-algorithm hmac-sha-512

router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2

interface Ethernet0/0.10
 ip ospf 1 area 1
interface Ethernet0/0.20
 ip ospf 1 area 1
interface Ethernet0/0.30
 ip ospf 1 area 1
interface Ethernet0/0.40
 ip ospf 1 area 1
interface Ethernet0/0.50
 ip ospf 1 area 1
interface Ethernet0/0.100
 ip ospf 1 area 1

interface range Ethernet0/1-2
 ip ospf hello-interval 1
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf authentication key-chain OSPF_AREA1_AUTH
 ip ospf ttl-security hops 1
 ip ospf network point-to-point
 ip ospf 1 area 1

interface Loopback0
 ip ospf 1 area 1
```
</details>

<details>
<summary>ABR1</summary>

```cisco
## ABR1

key chain OSPF_AREA0_AUTH
 key 1
  key-string LAB-OSPF-AREA0-PASSWORD
  cryptographic-algorithm hmac-sha-512

key chain OSPF_AREA1_AUTH
 key 1
  key-string LAB-OSPF-AREA1-PASSWORD
  cryptographic-algorithm hmac-sha-512

router ospf 1
 router-id 3.3.3.3
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2

interface range Ethernet0/0-2
 ip ospf hello-interval 1
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf network point-to-point
 ip ospf ttl-security hops 1

interface range Ethernet0/0-1
 ip ospf authentication key-chain OSPF_AREA1_AUTH
 ip ospf 1 area 1

interface Ethernet0/2
 ip ospf authentication key-chain OSPF_AREA0_AUTH
 ip ospf 1 area 0

interface Loopback0
 ip ospf 1 area 0
```
</details>

<details>
<summary>ASBR1</summary>

```cisco
key chain OSPF_AREA0_AUTH
 key 1
  key-string LAB-OSPF-AREA0-PASSWORD
  cryptographic-algorithm hmac-sha-512

ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1

router ospf 1
 router-id 4.4.4.4
 passive-interface default
 no passive-interface Ethernet0/0
 default-information originate

interface Ethernet0/0
 ip ospf hello-interval 1
 ip ospf dead-interval minimal hello-multiplier 4
 ip ospf authentication key-chain OSPF_AREA0_AUTH
 ip ospf ttl-security hops 1
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Loopback0
 ip ospf 1 area 0
```
</details>


---
### Management Plane


#### Management Interface Configuration

VLAN 100 was created for the remote management of network devices. The `192.168.100.0/24` subnet was reserved for Layer 2 management IP addresses. SVIs were configured on all switches.

Loopback addresses, which were assigned the `192.168.254.0/24` subnet, were configured on all routers to ensure that they are reachable by other services without relying on the availability of physical interfaces.

##### User Account Creation

One senior network administrator manages the enterprise campus network. An administrative account conferred with Level 15 privileges was created to facilitate local device authentication for management purposes.

The password configured below was a relatively short, non-complex string with only upper-case letters and two hyphens. This was done for simplicity. In production, a strong password, preferably involving a combination of upper- and lower-case letters, numbers and special characters, should always be configured and renewed periodically in accordance with the organization’s password policy.
  
Cisco IOS stores passwords in plaintext if the `password` keyword is used. When the `secret` keyword is used to set passwords instead, they will be encrypted depending on the IOS version and platform. The IOL images in CML-Free default to Type 9 authentication, which implements the cryptographically robust scrypt algorithm.


<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>
	
```cisco
username admin privilege 15 secret LAB-ADMIN-PASSWORD
```
</details>


#### Line Security

The console line for each router and switch was secured by prompting users to authenticate using locally stored credentials.  

Day-to-day management of the routers and switches is done remotely via Secure Shell (SSH). SSH was selected over Telnet because the latter does not encrypt credentials and commands sent over the wire. While modern cryptographic algorithms such as ECDSA and Ed25519 generate stronger but less computationally expensive keys, RSA was selected because it remains the most widely supported algorithm for SSH key generation across Cisco platforms, maximizing interoperability with legacy devices. An ACL was also created and applied to all VTY lines to ensure that only authorized clients from the IT department could access the network devices over SSH.

Console access via the console line (`line console 0`) and SSH (`line vty 0 4`) was configured to automatically time out after ten (10) minutes of idle time to deprive any unauthorized user of the opportunity to make any changes or view sensitive information such as password hashes and neighbor adjacencies if the senior network administrator is away from their workstation. 

<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>
	
```cisco
ip domain name example.com

crypto key generate rsa modulus 2048

ip ssh version 2

ip access-list standard SSH_AUTH_HOSTS
 remark Authorized SSH clients
 permit 192.168.40.0 0.0.0.255

line console 0
 login local
 exec-timeout 10 0
 logging synchronous

line vty 0 4
 access-class SSH_AUTH_HOSTS in
 transport input ssh
 login local
 exec-timeout 10 0
```
</details>


#### Syslog

All network devices in the enterprise campus network were configured to send Syslog messages with severity levels 0 through 6 (emergencies through informational) to a remote Syslog server. 

The loopback interface of each router and management SVI of each switch were configured as the source interface for all Syslog messages to ensure that they originate from a stable IP address, regardless of the operational status of individual physical interfaces. This simplifies device identification and log correlation at the centralized Syslog server at `192.168.40.18`. Console logging was disabled to enhance readability of interactive management sessions.

A 16 KB local Syslog buffer was configured on all routers and switches to retain recent log messages for troubleshooting purposes in the event that the centralized Syslog server becomes temporarily unavailable. This complements, rather than replaces, centralized log collection.

Timestamps and sequence numbers were also enabled to improve event correlation, troubleshooting, and forensic analysis by providing accurate temporal information and unique identifiers for individual log messages.

<details>
	<summary>ACC1, ACC2, DIS1 and DIS2</summary>

```cisco
service timestamps log datetime msec localtime show-timezone
service sequence-numbers

logging host 192.168.40.18
logging source-interface Vlan100
logging trap informational
logging buffered 16384 informational

no logging console
```
</details>


<details>
<summary>R1, R2, ABR1 and ASBR1</summary>
	
```cisco
service timestamps log datetime msec localtime show-timezone
service sequence-numbers

logging host 192.168.40.18
logging source-interface Loopback0
logging trap informational
logging buffered 16384 informational

no logging console
```
</details>


#### SNMP

Simple Network Management Protocol (SNMP) was configured to enable network and system administrators to monitor device performance and status remotely and respond to any issues or anomalies promptly when they receive alerts in the Network Management System (NMS).

All routers and switches were configured as managed devices to allow a centralized NMS to monitor their status and performance using SNMPv2c. Read-only access to the Management Information Base (MIB) was granted. 

SNMPv2c was selected because SNMPv3 is not supported by the IOL router and switch images of CML-Free. In production, SNMPv3 is highly recommended due to its support for authentication and encryption. An ACL was also configured to permit SNMP server messages only from authorized hosts running the NMS.

SNMP Traps were disabled since configuring an NMS is outside the scope of the lab. However, the feature was temporarily enabled to check whether SNMP Trap messages were being sent out.

<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>
	
```cisco
ip access-list standard SNMP_MANAGERS
 permit 192.168.40.19 
 permit 192.168.40.20

snmp-server community NMS-LAB-ENTERPRISE ro SNMP_MANAGERS
snmp-server host 192.168.40.19 version 2c NMS-LAB-ENTERPRISE
snmp-server host 192.168.40.20 version 2c NMS-LAB-ENTERPRISE
snmp-server location Enterprise Campus
snmp-server contact Senior Network Administrator
```
</details>


#### CDP and LLDP

Link-Layer Discovery Protocol (LLDP) was enabled as the primary neighbor discovery protocol to support interoperability with non-Cisco devices. Cisco Discovery Protocol (CDP) was retained only on trusted internal infrastructure links for operational visibility of Cisco devices. Both protocols were disabled on user-facing access ports of `ACC1` and `ACC2` and on the WAN-facing interface of `ASBR1` to eliminate the exposure of device information to untrusted devices and external networks.

CDP and LLDP timers were left at their default values because neighbor discovery is not a convergence-critical function in this lab environment. The default advertisement and hold timers are sufficient for topology discovery and troubleshooting.


<details>
<summary>ACC1 and ACC2</summary>

```cisco
## ACC1, ACC2

lldp run

interface range Ethernet1/0-3
 no cdp enable
 no lldp transmit
 no lldp receive
```
</details>

<details>
<summary>DIS1, DIS2, R1, R2, ABR1</summary>

```cisco
lldp run
```
</details>


<details>
<summary>ASBR1</summary>
	
```
lldp run

interface Ethernet0/1
 no cdp enable
 no lldp transmit
 no lldp receive
```
</details>


### Infrastructure Services

#### DHCP

Dynamic Host Configuration Protocol (DHCP) was configured to facilitate automatic provisioning of network information for hosts such as the IP address, subnet mask, default gateway address, DNS server addresses, enterprise domain name, and more.

`R1` is configured as the DHCP server for the hosts of all VLANs. Ten (10) addresses were excluded from the DHCP pool for VLANs 10, 20 and 30, and fifty (50) for that of VLAN 40, to be reserved for servers and other static hosts. The lease period of the IP addresses is twelve (12) hours. No DHCP pools were created for the printer and management VLANs since all IP addresses were statically configured. Likewise, the default gateway for each VLAN is the VIP address in each subinterface in the VRRP configuration.

`R2`, on the other hand, was not configured as a second DHCP server because the DHCP server program of Cisco routers does not have automatic binding table synchronization and coordinated failover with `R1`, which may cause duplicate IP address assignments. As such, the DHCP server was disabled on `R2` for hardening purposes, and the user-facing subinterfaces act as a DHCP relay agent pointing to the loopback address of `R1`.

In modern enterprise networks, DHCP servers are centralized on Active Directory or Linux servers because they allow for ease of management and troubleshooting and have other useful features such as automatic failover, load balancing, and support for Dynamic DNS.

**Note**: Because the enterprise campus network is IPv4-only, DHCPv6 configuration is outside the scope of this lab.

<details>
<summary>R1</summary>
	
```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
ip dhcp excluded-address 192.168.40.1 192.168.40.50

ip dhcp pool VLAN_10_FINANCE
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12

ip dhcp pool VLAN_20_MARKETING
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12

ip dhcp pool VLAN_30_HR
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12

ip dhcp pool VLAN_40_IT
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12

ip dhcp conflict logging
```
</details>

<details>
<summary>R2</summary>

```cisco
no service dhcp

interface Ethernet0/0.10
 ip helper-address 192.168.254.3
interface Ethernet0/0.20
 ip helper-address 192.168.254.3
interface Ethernet0/0.30
 ip helper-address 192.168.254.3
interface Ethernet0/0.40
 ip helper-address 192.168.254.3
```
</details>


#### NAT/PAT

`ASBR1` serves as the edge router of the enterprise network, being directly connected to an ISP router. As such, it was configured to perform Port Address Translation (PAT) to provide Internet connectivity to all internal hosts. 

An ACL was created to identify the internal subnets whose private IP addresses are to be translated to the public IP address of `203.0.113.2/24` and their respective port numbers. An entry was created for the printer subnet to permit outbound connectivity for critical functions such as firmware updates, certificate validation and time synchronization. In a production environment, outbound traffic from the printer VLAN would be typically restricted further by firewall policies to OEM-approved services only.

<details>
<summary>ASBR1</summary>

```cisco
ip access-list standard NAT_HOSTS
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255
 permit 192.168.40.0 0.0.0.255
 permit 192.168.50.0 0.0.0.255

interface Ethernet0/0
 ip nat inside

interface Ethernet0/1
 ip nat outside
 
ip nat inside source list NAT_HOSTS interface e0/1 overload
```
</details>


#### NTP

Network Time Protocol (NTP) was configured on all routers and switches to enable them to synchronize their clocks with that of upper-strata NTP servers. 
  
`ASBR1` synchronizes its time directly with several NTP servers in the Philippine zone of the NTP Pool project for redundancy. Instead of statically configuring the IP addresses of the NTP servers, their FQDNs were configured instead. The DNS servers of Cloudflare (`1.1.1.1`) and Google (`8.8.8.8`) were set to facilitate DNS resolution.

NTP Pool was selected because many production networks rely on it for synchronization of time with public NTP servers. All other devices synchronize their time with `ASBR1`. 

NTP authentication via HMAC-SHA-256 was also enabled to ensure the integrity of NTP messages. Incorrect or maliciously altered system time can cause failures in time-dependent security mechanisms, including certificate validation. 


<details>
<summary>ASBR1</summary>
	
```cisco
ip name-server 8.8.8.8
ip name-server 1.1.1.1

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 0.ph.pool.ntp.org prefer
ntp server 1.ph.pool.ntp.org
ntp server 2.ph.pool.ntp.org

ntp update-calendar
```
</details>

<details>
<summary>ABR1</summary>

```cisco
ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.1 key 1
ntp source Loopback0

ntp update-calendar
```
</details>

<details>
<summary>R1 and R2</summary>
	
```cisco
ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.2 key 1 prefer
ntp server 192.168.254.1 key 1
ntp source Loopback0

ntp update-calendar
```
</details>

<details>
<summary>ACC1, ACC2, DIS1 and DIS2</summary>

```cisco
ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.3 key 1 prefer
ntp server 192.168.254.4 key 1
ntp source vlan 100

ntp update-calendar
```
</details>


---
### Network Security

#### Layer 2 Security

##### Port Security

Port Security was enabled on all access ports of `ACC1` and `ACC2` as a lightweight edge security measure.

Port Security was configured differently on the `Ethernet1/3` interface of `ACC2` because the VMware ESXi host connected to it would have multiple VMs. Each VM has its own virtual network interface card (vNIC), with its own MAC address. To accommodate multiple VMs, the maximum MAC addresses allowed was increased to eight (8), and sticky MAC address learning was disabled due to the administrative overhead of managing learned MAC addresses across multiple VMs, some of which might be set up temporarily.

<details>
<summary>ACC1 and ACC2</summary>

```cisco
interface range Ethernet1/0-3
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

errdisable recovery cause psecure-violation
errdisable recovery interval 300
```
</details>

<details>
<summary>ACC2</summary>
	
```cisco
interface Ethernet1/3
 switchport port-security maximum 8
 no switchport port-security mac-address sticky
```
</details>


##### DHCP Snooping

DHCP Snooping was enabled on `ACC1` and `ACC2` to prevent rogue DHCP servers and DHCP starvation attacks. All uplink interfaces were configured as trusted interfaces while keeping all host-facing ports untrusted. Only DHCP traffic from hosts in the departmental VLANs would be inspected.

Additionally, DHCP rate-limiting was implemented to mitigate DHCP starvation attacks, in which an attacker attempts to exhaust the pool of available addresses by generating large numbers of bogus DHCP requests. In this lab, each host-facing access port was rate-limited to only fifteen (15) packets per second to thwart DHCP message floods. DHCP Snooping also provides the binding database required by Dynamic ARP Inspection (DAI).


<details>
	<summary>ACC1 and ACC2</summary>

```cisco
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

interface range Port-channel1-2
 ip dhcp snooping trust
 
interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15

errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 300
```
</details>


##### *Dynamic ARP Inspection (DAI)*

Dynamic ARP Inspection (DAI) was configured to validate source and destination MAC addresses and IP-to-MAC mappings on `ACC1` and `ACC2` in VLANs 10, 20, 30, 40, 50 and 100 to prevent ARP cache poisoning attacks. Without DAI, an attacker can send Gratuitous ARP (GARP) messages or ARP replies using a spoofed IP address. This could cause traffic bound for a legitimate device to be redirected through the attacker’s machine, enabling a man-in-the-middle attack.

In addition, just like in DHCP Snooping, all host-facing untrusted interfaces were rate-limited to just fifteen (15) ARP packets per second to accommodate normal host behavior while restricting abnormally high rates of ARP traffic generated by malicious end-user devices. An ARP ACL was created for static hosts in the network. For simplicity, only four devices in the IT Department (VLAN 40) were added.

<details>
<summary>ACC1 and ACC2</summary>

```cisco
ip arp inspection vlan 10,20,30,40,50,100
ip arp inspection validate src-mac dst-mac ip
ip arp inspection filter STATIC_HOSTS vlan 40

interface range Port-channel1-2
 ip arp inspection trust

interface range Ethernet1/0-3
 ip arp inspection limit rate 15 

errdisable recovery cause arp-inspection
errdisable recovery interval 300

arp access-list STATIC_HOSTS

 remark SYSLOG SERVER
 permit ip host 192.168.40.18 mac host 0050.56b3.4018
 
 remark NMS-1
 permit ip host 192.168.40.19 mac host 0050.56b3.4019
 
 remark NMS-2
 permit ip host 192.168.40.20 mac host 0050.56b3.4020
 
 remark SMB SERVER
 permit ip host 192.168.40.21 mac host 0050.56b3.4021
```
</details>


##### BPDU Guard

BPDU Guard was enabled alongside PortFast on all ports by default but disabled on trunk ports manually. Without it, a rogue switch plugged into the network may inadvertently trigger a root bridge election, possibly causing that switch to become a suboptimal root bridge.

##### Automatic Err-Disable Recovery

Automatic err-disable recovery was enabled for Dynamic ARP Inspection events, DHCP Snooping rate-limiting events, and BPDU Guard violations with a recovery interval of three hundred (300) seconds. This allows ports disabled by anomalous ARP and DHCP packet floods to recover automatically while maintaining protection against ARP, DHCP and STP attacks.

##### Unsupported Features

It is worth mentioning that IP Source Guard (IPSG), Storm Control and MACsec were considered as Layer 2 security features in this lab as they are configured in most high-security production networks. However, since CML-Free was used, the IOL L2 switch image does not support these features. 


#### Layer 3 Security

##### Access Control Lists (ACLs)

ACLs were configured in multiple areas of the network for traffic filtering, infrastructure protection, management access control, and feature-specific operations such as NAT/PAT and DAI. The ACLs discussed in this subsection focus on security-related traffic filtering operations.
  
Inter-VLAN communications were restricted via ACLs to ensure that hosts in VLANs 10, 20, 30, 40 and 50 cannot access one another, thwarting lateral movement attempts. Access from VLANs 10, 20, 30 and 50 to the management VLAN and router loopback subnets was likewise restricted. However, hosts in VLAN 40 (IT) were permitted to access hosts in all other VLANs to facilitate administrative functions such as remote troubleshooting, software deployment, system maintenance and end user support performed by helpdesk technicians and system administrators.

Hosts from all departmental VLANs were granted access to an SMB server in the IT Department at `192.168.40.21` dedicated to software management. However, in many real-life enterprise environments, shared infrastructure services such as software deployment, file servers and patch management servers are placed in a dedicated server VLAN that must be accessible across multiple VLANs.

Likewise, certain IPv4 address spaces such as the RFC 1918 and APIPA addresses should never originate from the Internet. Packets arriving from the Internet bearing source addresses belonging to these address spaces, commonly known as *bogons*, were filtered using an inbound ACL that contains a non-exhaustive list of bogon spaces and applied to the WAN-facing interface of `ASBR1`. This protects the enterprise network from IP spoofing attacks.

<details>
<summary>R1 and R2</summary>

```cisco
ip access-list extended VLAN_10_INBOUND  
 remark Allow access to SMB server at IT Department; deny access to all other IT hosts and management plane  
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.21 eq 445 
 deny ip 192.168.10.0 0.0.0.255 192.168.40.0 0.0.0.255  
 deny ip 192.168.10.0 0.0.0.255 192.168.100.0 0.0.0.255  
 deny ip 192.168.10.0 0.0.0.255 192.168.254.0 0.0.0.255

 remark Isolate Finance Department from all other departments  
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255  
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255

 remark Permit all other traffic  
 permit ip any any

ip access-list extended VLAN_20_INBOUND  
 remark Restrict access to IT Department (except SMB server) and management plane  
 permit tcp 192.168.20.0 0.0.0.255 host 192.168.40.21 eq 445  
 deny ip 192.168.20.0 0.0.0.255 192.168.40.0 0.0.0.255  
 deny ip 192.168.20.0 0.0.0.255 192.168.100.0 0.0.0.255  
 deny ip 192.168.20.0 0.0.0.255 192.168.254.0 0.0.0.255

 remark Isolate Marketing Department from all other departments  
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255  
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255

 remark Permit all other traffic  
 permit ip any any

ip access-list extended VLAN_30_INBOUND  
 remark Restrict access to IT Department (except SMB server) and management plane  
 permit tcp 192.168.30.0 0.0.0.255 host 192.168.40.21 eq 445
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255  
 deny ip 192.168.30.0 0.0.0.255 192.168.100.0 0.0.0.255  
 deny ip 192.168.30.0 0.0.0.255 192.168.254.0 0.0.0.255

 remark Isolate HR Department from all other departments  
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255  
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255

 remark Permit all other traffic  
 permit ip any any

interface Ethernet0/0.10  
 ip access-group VLAN_10_INBOUND in  
interface Ethernet0/0.20  
 ip access-group VLAN_20_INBOUND in  
interface Ethernet0/0.30  
 ip access-group VLAN_30_INBOUND in
```
</details>


<details>
<summary>ASBR1</summary>


```cisco
ip access-list extended BLOCK_BOGON_TRAFFIC
 remark Block bogon traffic originating from Internet
 deny ip 0.0.0.0 0.255.255.255 any log
 deny ip 10.0.0.0 0.255.255.255 any log
 deny ip 127.0.0.0 0.255.255.255 any log
 deny ip 169.254.0.0 0.0.255.255 any log
 deny ip 172.16.0.0 0.15.255.255 any log
 deny ip 192.168.0.0 0.0.255.255 any log
 deny ip 224.0.0.0 31.255.255.255 any log
 permit ip any any

interface Ethernet0/1
 ip access-group BLOCK_BOGON_TRAFFIC in
```
</details>


##### Device Hardening

Device hardening involves disabling unused services and components as they enlarge the attack surface of the network device being secured. This adheres to the *principle of least functionality*, which holds that only services essential to the intended function and configuration of the network device shall remain enabled.

Cisco routers and switches have a few services and components that are enabled by default. However, they are typically never used in most modern production networks, with some of these services having been deprecated or considered obsolete. Earlier, VTP was disabled on all switches, and CDP and LLDP were disabled on the untrusted interfaces of `ACC1`, `ACC2` and `ASBR1`. Unused ports were also disabled on routers and switches. Below are services and components that were disabled in observance of enterprise hardening measures:

1. **Disable HTTP and HTTPS servers**. The HTTP and HTTPS servers in the routers and switches were disabled because web-based management is not required in this environment. Network administrators are assumed to manage such devices exclusively through SSH. Cisco IOS and IOS XE web management interfaces have historically been associated with several high-severity vulnerabilities, with some leading to device compromise and remote code execution.


<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>

```cisco
no ip http server
no ip http secure-server
```
</details>

2. **Disable AUX port.** The AUX port was historically used to access a device's console over a modem line. While Plain Old Telephone Service (POTS) lines are now considered obsolete in most modern production networks, the AUX port nevertheless represents a high-risk attack vector in case malicious actors have physical access to the devices. `no exec` prevents the device from spawning a CLI shell on the AUX line.

<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>


```cisco
line aux 0
 no exec
 transport input none
```
</details>

3. **Disable Telnet**. In this lab scenario, SSH is exclusively used for remote device management. Because Telnet does not ship with built-in encryption, the service was disabled in favor of its encrypted counterpart. This was configured earlier on all network devices in the Line Security subsection.

<details>
<summary>ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1</summary>

```cisco
line vty 0 4
 transport input ssh
```
</details>
