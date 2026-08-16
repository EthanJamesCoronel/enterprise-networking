##### Layer 2 Design


###### *VLAN Configuration*

```
# DIS1, DIS2, ACC1, ACC2

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
 
vlan 2151
 name NATIVE_VLAN
 
vtp mode off
```


###### *Trunk and Access Ports, EtherChannel and SVI Configuration*

```
## ACC1

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
 
interface vlan 100
 ip address 192.168.100.6 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
 

## ACC2

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
 switchport trunk allowed vlan 40,100bg
 switchport native vlan 2151
 switchport nonegotiate
 spanning-tree portfast trunk

interface vlan 100
 ip address 192.168.100.7 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
 
 
## DIS1

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

interface vlan 100
 ip address 192.168.100.4 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
 
 
## DIS2

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
 
interface vlan100
 ip address 192.168.100.5 255.255.255.0
 no shutdown
ip default-gateway 192.168.100.1
```




###### Multiple Spanning Tree Protocol (MSTP)

```
## DIS1

spanning-tree mode mst
spanning-tree mst 1 priority 4096
spanning-tree mst 2 priority 8192

spanning-tree mst configuration
 name ENTERPRISE-CAMPUS-LAB
 revision 1
 instance 1 vlan 10,30,50
 instance 2 vlan 20,40,100

 
## DIS2

spanning-tree mode mst
spanning-tree mst 1 priority 8192
spanning-tree mst 2 priority 4096

spanning-tree mst configuration
 name ENTERPRISE-CAMPUS-LAB
 revision 1
 instance 1 vlan 10,30,50
 instance 2 vlan 20,40,100


## ACC1 and ACC2

spanning-tree mode mst

spanning-tree mst configuration
 name ENTERPRISE-CAMPUS-LAB
 revision 1
 instance 1 vlan 10,30,50
 instance 2 vlan 20,40,100
 
---
VERIFICATION

# View MSTIs

show spanning-tree mst

# View MST configuration

spanning-tree mst configuration
```

---
##### Layer 3 Design


###### *Interface Configuration*

```
## R1

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

 
## R2 

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

 
## ABR1

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


###### *Virtual Router Redundancy Protocol (VRRP)*

```
## R1 

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
 vrrp 50 ip 192.168.50.1
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
 
## R2

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
 vrrp 50 ip 192.168.50.1
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
 
---
VERIFICATION

show vrrp
```

###### *Open Shortest Path First (OSPF)*

```
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


## R2

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


## ASBR1

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

----
VERIFICATION

show ip ospf neighbor
show ip ospf interface brief
show ip ospf database
```

---
##### Network Security


###### *Access Control Lists (ACLs)*

```
## R1, R2

ip access-list extended VLAN_10_INBOUND  
 remark Allow access to SMB server at IT Department and printer; deny access to all other IT hosts and management plane  
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.40.21 eq 445  
 permit ip 192.168.10.0 0.0.0.255 host 192.168.50.4  
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
 deny ip 192.168.20.0 0.0.0.255 host 192.168.50.4  
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
 deny ip 192.168.30.0 0.0.0.255 host 192.168.50.4  
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


## ASBR1

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

-----
VERIFICATION

show ip interface Ethernet0/0.10
show ip access-lists
```

###### *Device Hardening*

```
----
HYPERTEXT TRANSFER PROTOCOL (HTTP)

# ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1, ASBR1

no ip http server
no ip http secure-server

----
DISABLE AUX PORT

# ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1, ASBR1

line aux 0
 no exec
 transport input none

----
TELNET

# ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1, ASBR1

line vty 0 4
 transport input ssh


CISCO DISCOVERY PROTOCOL (CDP)
 
## ACC1 and ACC2

interface range Ethernet1/0-3
 no cdp enable

## ASBR1

interface Ethernet0/1
 no cdp enable

----
DYNAMIC HOST CONFIGURATION PROTOCOL (DHCP)

# R2 

no service dhcp
```


---
##### Management Plane


###### *User Account Creation*

```
## ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1

username admin privilege 15 secret LAB-ADMIN-PASSWORD
```

###### *Line Security*

```
## ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1 and ASBR1

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

###### *Syslog*

```
# R1, R2, ABR1, ASBR1

service timestamps log datetime msec localtime show-timezone
service sequence-numbers

logging host 192.168.40.18
logging source-interface Loopback0
logging trap informational
logging buffered 16384 informational

no logging console
```

###### *SNMPv2c

```
## ACC1, ACC2, DIS1, DIS2, R1, R2, ABR1, ASBR1

ip access-list standard SNMP_MANAGERS
 permit 192.168.40.19 
 permit 192.168.40.20

snmp-server community NMS-LAB-ENTERPRISE ro SNMP_MANAGERS
snmp-server host 192.168.40.19 version 2c NMS-LAB-ENTERPRISE
snmp-server host 192.168.40.20 version 2c NMS-LAB-ENTERPRISE
snmp-server location Enterprise Campus
snmp-server contact Senior Network Administrator

-----
VERIFICATION

show running-config | section snmp
show snmp
```


###### *CDP and LLDP*

```
## ACC1, ACC2

lldp run

interface range Ethernet1/0-3
 no cdp enable
 no lldp transmit
 no lldp receive

## DIS1, DIS2, R1, R2, ABR1

lldp run

## ASBR1

lldp run

interface Ethernet0/1
 no cdp enable
 no lldp transmit
 no lldp receive
```


---
##### Infrastructure Services

###### *Dynamic Host Configuration Protocol (DHCP)*

```
## R1

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
ip dhcp excluded-address 192.168.40.1 192.168.40.50

ip dhcp pool VLAN_10_FINANCE
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12 0

ip dhcp pool VLAN_20_MARKETING
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12 0

ip dhcp pool VLAN_30_HR
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12 0

ip dhcp pool VLAN_40_IT
 network 192.168.40.0 255.255.255.0
 default-router 192.168.40.1
 dns-server 8.8.8.8
 domain-name example.com
 lease 0 12 0

ip dhcp conflict logging


## R2

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


###### *Port Address Translation (PAT)*

```
# ASBR1

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


###### *Network Time Protocol (NTP)*

```
## ASBR1

ip name-server 8.8.8.8
ip name-server 1.1.1.1

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 0.ph.pool.ntp.org prefer
ntp server 1.ph.pool.ntp.org
ntp server 2.ph.pool.ntp.org

ntp update-calendar

## ABR1

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.1 key 1

ntp source Loopback0
ntp update-calendar


## R1, R2

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.2 key 1 prefer
ntp server 192.168.254.1 key 1

ntp source Loopback0
ntp update-calendar


## ACC1, ACC2, DIS1, DIS2

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.3 key 1 prefer
ntp server 192.168.254.4 key 1
ntp source vlan 100

ntp update-calendar

----
VERIFICATION

show ntp status
show ntp associations
```


---
##### Network Security


###### *Port Security*

```
## ACC1, ACC2

interface range Ethernet1/0-3
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

errdisable recovery cause psecure-violation
errdisable recovery interval 300

# ACC2

interface Ethernet1/3
 switchport port-security maximum 8
 no switchport port-security mac-address sticky
```

###### *DHCP Snooping*

```
## ACC1, ACC2

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50,100
no ip dhcp snooping information option

interface range Ethernet0/0-1
 ip dhcp snooping trust

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15

errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 300
```

###### *Dynamic ARP Inspection (DAI)*

```
## ACC1, ACC2

ip arp inspection vlan 10,20,30,40,50,100
ip arp inspection validate src-mac dst-mac ip

interface range Ethernet0/0-1
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

