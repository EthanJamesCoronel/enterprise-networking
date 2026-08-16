
```
# Hostname Configuration

hostname DIS2


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


# MSTP Configuration

spanning-tree mode mst
spanning-tree mst 1 priority 8192
spanning-tree mst 2 priority 4096

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


# LLDP Configuration

lldp run


# NTP Configuration

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 192.168.254.3 key 1 prefer
ntp server 192.168.254.4 key 1
ntp source vlan 100

ntp update-calendar


# DHCP Snooping Configuration

ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

interface range Port-channel1-2
 ip dhcp snooping trust
interface range Ethernet1/0-1
 ip dhcp snooping trust
 

# Device Hardening Configuration

no ip http server
no ip http secure-server

line aux 0
 no exec
 transport input none
```