
```
# Hostname Configuration

hostname ABR1


# Interface Configuration

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
 

# OSPF Configuration

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
logging source-interface Loopback0
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

ntp server 192.168.254.1 key 1
ntp source Loopback0

ntp update-calendar


# Device Hardening Configuration

no ip http server
no ip http secure-server

line aux 0
 no exec
 transport input none
```