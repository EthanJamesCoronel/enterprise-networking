
```
# Hostname Configuration

hostname ASBR1


# Interface Configuration

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


# OSPF Configuration

key chain OSPF_AREA0_AUTH
 key 1
  key-string LAB-OSPF-AREA0-PASSWORD
  cryptographic-algorithm hmac-sha-512

ip route 0.0.0.0 0.0.0.0 203.0.113.1

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


# CDP and LLDP Configuration

lldp run

interface Ethernet0/1
 no cdp enable
 no lldp transmit
 no lldp receive

lldp run

interface Ethernet0/1
 no cdp enable
 no lldp transmit
 no lldp receive
 
 
# NAT/PAT Configuration

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


# NTP Configuration

ip name-server 8.8.8.8
ip name-server 1.1.1.1

ntp authenticate
ntp authentication-key 1 hmac-sha2-256 LAB-NTP-AUTHENTICATION
ntp trusted-key 1

ntp server 0.ph.pool.ntp.org prefer
ntp server 1.ph.pool.ntp.org
ntp server 2.ph.pool.ntp.org

ntp update-calendar


# Bogon Filter Configuration

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



# Device Hardening

no ip http server
no ip http secure-server

line aux 0
 no exec
 transport input none
```