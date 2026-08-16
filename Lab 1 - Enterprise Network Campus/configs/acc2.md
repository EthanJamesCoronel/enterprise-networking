
```
# Hostname Configuration

hostname ACC2


# VLAN Configuration

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


# Interface and EtherChannel Configuration

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


# MSTP Configuration

spanning-tree mode mst

spanning-tree mst configuration
 name LAB1-ENTERPRISE
 revision 1
 instance 1 vlan 10,20,50
 instance 2 vlan 30,40,100
 

# Line Security Configuration

username admin privilege 15 secret LAB-ADMIN-PASSWORD

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
 

# Syslog Configuration

service timestamps log datetime msec localtime show-timezone
service sequence-numbers

logging host 192.168.40.18
logging source-interface Vlan100
logging trap informational
logging buffered 16384 informational

no logging console


# SNMP Configuration

ip access-list standard SNMP_MANAGERS
 permit 192.168.40.19 
 permit 192.168.40.20

snmp-server community NMS-LAB-ENTERPRISE ro SNMP_MANAGERS
snmp-server host 192.168.40.19 version 2c NMS-LAB-ENTERPRISE
snmp-server host 192.168.40.20 version 2c NMS-LAB-ENTERPRISE
snmp-server location Enterprise Campus
snmp-server contact Senior Network Administrator


# CDP and LLDP Configuration

lldp run

interface range Ethernet1/0-3
 no cdp enable
 no lldp transmit
 no lldp receive
 

# NTP Configuration

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.3 key 1 prefer
ntp server 192.168.254.4 key 1
ntp source vlan 100

ntp update-calendar


# Port Security Configuration

interface range Ethernet1/0-3
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict

interface Ethernet1/3
 switchport port-security maximum 8
 no switchport port-security mac-address sticky

errdisable recovery cause psecure-violation
errdisable recovery interval 300


# DHCP Snooping Configuration

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

interface range Port-channel1-2
 ip dhcp snooping trust
 
interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15

errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 300


# DAI Configuration

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
 

# Device Hardening Configuration

no ip http server
no ip http secure-server

line aux 0
 no exec
 transport input none
```