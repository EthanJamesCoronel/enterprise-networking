
#### Layer 2 Design

##### VLAN Configuration

###### Command

```
show vlan brief
```

###### Description and Screenshots

The `show vlan brief` output on `ACC1` confirms that all required VLANs were successfully created on `ACC1`. These include the four departmental VLANs (Finance, Marketing, HR and IT), the dedicated Printer VLAN, the Management VLAN, the Native VLAN, and the VLAN assigned to unused switchports.

**Note**: All switches have identical VLAN databases. The VLAN database of `ACC1` represents the other switches'. Likewise, all ports have not been assigned to their respective VLAN/s yet.

![acc1_vlan_database.png]("C:\..\verification\screenshots\1 - Layer 2 Design\acc1_vlan_database.png")

##### Switchports and EtherChannel

###### Command

```
show interfaces status
show interfaces trunk
show etherchannel summary
show ip interface brief
```

###### Description and Screenshots

The `show interface status` output on `ACC2` confirms that its `Ethernet1/0-3` interfaces were configured as access ports and were now assigned to their respective departmental VLANs. The `Ethernet1/3` interface was configured as a PortFast Trunk port to accommodate a VMware ESXi host with VMs for the NMS, the Syslog server, and the HP JetAdvantage server. The `Ethernet1/1` interface was also placed in the Unused Ports VLAN and administratively shut down as a device hardening measure.

![[acc2_interface_status.png]]

The `show interfaces trunk` output likewise confirms that the departmental VLANs, the printer VLAN and the management VLAN were allowed, and that the native VLAN was set to VLAN 2151.

The `show etherchannel summary` output confirms that the LACP bundles were up. The trunk links from the access-layer switches to the distribution-layer switches were configured as LACP bundles for redundancy and increased uplink bandwidth.

![[acc2_trunk_interfaces.png]]

![[acc2_etherchannel.png]]

In addition, the `show ip interface brief` output confirms that the management SVI for `ACC2` was successfully configured and operational. This was created for the purpose of remote device management.

![[acc2_management_svi.png]]


##### MSTP

###### Commands

```
# DIS1 and DIS2

show spanning-tree mst configuration
show spanning-tree mst
```

###### Description and Screenshots

The outputs confirm that `DIS1` serves as the root bridge for MSTI1, and `DIS2` for MSTI2. MSTI1 was also mapped to VLANs 10, 30 and 50, and MSTI2 to VLANs 20, 40 and 100. All other VLANs were mapped to the default MST instance (MSTI0).

![[dis1_mst_config.png]]

![[dis2_mst_config.png]]


![[dis1_mst_instances.png]]

![[dis2_mst_instances.png]]

---
#### Layer 3 Design

##### Interfaces Configuration

###### Commands

```
R1, R2, ABR1, ASBR1

show ip interface brief
```

###### Description

The outputs of `show ip interface brief` confirm that the IP address assignment for each physical interface of `R1`, `R2`, `ABR1` and `ASBR1` was successfully applied.

**Note**: Router-on-a-stick (ROAS) subinterfaces were intentionally omitted at this stage because they form part of the VRRP gateway configuration presented in the following subsection. As such, the physical trunk interface (`Ethernet0/0`) of `R1` and `R2` appears as `unassigned` in the output.

![[r1_interface_status.png]]

![[r2_interface_status.png]]

![[abr1_interface_status.png]]

![[asbr1_interface_status.png]]


##### VRRP

###### Commands

```
show vrrp brief
```

###### Description and Screenshots

The outputs of `show vrrp brief` confirm that `R1` serves as Master for VLANs 10, 30 and 50, and `R2` serves as Master for VLANs 20, 40 and 100. The other router assumes the Backup role for each VRRP, providing first-hop redundancy.

![[r1_vrrp_status.png]]

![[r2_vrrp_status.png]]


##### OSPF

###### Commands

```
# R1, R2, ABR1, ASBR1

show ip ospf neighbor

# ABR1

show ip ospf interface brief
show ip ospf database
show ip route ospf
```

###### Description and Screenshots

The outputs for `show ip ospf neighbor`, `show ip ospf database` and `show ip route ospf` confirm that OSPF neighbor adjacencies were established successfully, and that routes to the departmental VLANs, the printer VLAN and the management VLAN were propagated across the OSPF domain. 

The `show ip ospf neighbor` output also confirms that the network type of the OSPF-enabled interfaces was successfully changed to `point-to-point`, as indicated by the `-` in the `State` column. This eliminates unnecessary DR/BDR elections and prevents the propagation of Type 2 LSAs to conserve hardware resources and bandwidth.

The RID assignments are as follows: `1.1.1.1` for `R1`, `2.2.2.2` for `R2`, `3.3.3.3` for `ABR1`, and `4.4.4.4` for `ASBR1`.

![[r1_ospf_neighbors.png]]

![[r2_ospf_neighbors.png]]

![[abr1_ospf_neighbors.png]]

![[asbr1_ospf_neighbors.png]]


![[abr1_ospf_interfaces.png]]


![[abr1_ospf_database.png]]


![[abr1_ospf_routes.png]]

---
#### Management Plane

##### User Account Creation

###### Commands

```
show running-config | section username
```

###### Description and Screenshots

The outputs for `show running-config | section username` in `R1` and `R2` confirm that the username and password were successfully configured, that the password hash was stored using Type 9 authentication (`secret 9`), and that the network administrator was granted Level 15 privileges, enabling full control over the network devices.

**Note**: Even though the senior network administrator uses the same password across all network devices, the scrypt algorithm adds a random salt that causes the password hash to be unique on each one. This thwarts rainbow table attacks, which involve digests pre-computed using hashing algorithms that do not implement randomized salts.

![[r1_user_creation.png]]

![[r2_user_creation.png]]

##### Line Security

###### Commands

```
show running-config | section line

show ip ssh
```

###### Description and Screenshots

The output for `show running-config | section line` on `R1` confirms that line security was successfully configured on the console and VTY lines, and that Telnet was disabled, leaving only SSH for remote device management. 

![[r1_line_security_config.png]]

The output for `show ip ssh` proves that SSH 2.0 was successfully configured, and that 2048-bit keys were successfully generated.

![[r1_ssh_info.png]]

Upon saving the line security configuration to the startup configuration, a user login prompt appeared after the console access timeout timer expired, confirming that device authentication was set up correctly.

![[r1_console_auth_prompt.png]]


##### Syslog

###### Commands

```
show running-config | include logging

show logging last 5
```

###### Description and Screenshots

The `show running-config | include logging` output on `R1` confirms the successful application of the Syslog configuration on `R1`. The Syslog configuration is uniform across all network devices.

![[r1_syslog_config.png]]

The `show logging last 5` output displays the five most recent Syslog messages stored in the Syslog buffer of `R1`.

![[r1_syslog_buffer.png]]

**Note**: Screenshots on the Syslog server itself are not included because setting up a Syslog server is beyond the scope of this lab.


##### SNMP

###### Commands

```
show running-config | section snmp

show snmp
```

###### Description and Screenshots

The `show running-config | section snmp` output on `ACC1` confirms that SNMPv2c was configured successfully. 

![[acc1_snmp_config.png]]

The `show snmp` output on `ACC1` confirms that the SNMPv2c was operational when SNMP traps were temporarily enabled, having sent 16 SNMP Trap messages as of this screenshot. SNMP Traps were disabled since configuration of an NMS is outside the scope of this lab.

![[acc1_snmp_stats.png]]

**Note**: The configuration and the output of this command are uniform across all network devices.

##### CDP and LLDP

###### Commands

```
# ABR1

show lldp neighbors
show cdp neighbor

# ACC1, ACC2

show running-config | exclude switchport|ip dhcp

# ASBR1

show running-config | exclude ip 
```

###### Description and Screenshots

The `show lldp neighbors` output on `ABR1` confirm that LLDP was successfully configured, and that its neighbors could be discovered. Even though CDP is already enabled by default on all Cisco network devices, the `show cdp neighbors` output was entered for good measure to display its list of neighbors.

![[abr1_lldp_neighbors.png]]

![[abr1_cdp_neighbors.png]]

The outputs of `show running-config | exclude switchport|ip dhcp` on `ACC1` and `ACC2` and `show running-config | exclude ip ` on `ASBR1` likewise confirm that CDP and LLDP were disabled on their untrusted interfaces. 

![[acc1_acc_ports_cdp_lldp_disabled.png]]

![[asbr1_interface_cdp_lldp_disabled.png]]

---
#### Infrastructure Services

##### DHCP

###### Commands

```
# R1
show running-config | section ip dhcp
show ip dhcp binding

# FINANCE-1

ip addr show eth0
ip route show default

# R2
show running-config | include dhcp
```

###### Description and Screenshots

The `show running-config | section dhcp` output on `R1` confirms that the exempted IPv4 address ranges and DHCP pools for the departmental VLANs were successfully created.

![[r1_dhcp_config.png|366]]

The `show ip dhcp binding` output on `R1` and the `ip addr show eth0` and `ip route show default` outputs on `FINANCE-1`, which is an Alpine Linux VM residing at VLAN 10, show the successful configuration of an IPv4 address (`192.168.10.11/24`) and default gateway (`192.168.10.1`) on the aforementioned end-user host.

![[r1_dhcp_binding.png]]

![[finance1_network_info.png]]

On the other hand, the `show running-config | include dhcp` output on `R2` also confirms that the DHCP service was successfully disabled as a hardening measure since the LAN only has one DHCP server.

![[r2_dhcp_config.png|265]]

##### NAT/PAT

###### Commands

```
# ASBR1

show running-config | section interface

show running-config | include ip nat inside source

show ip nat translations
```

###### Description and Screenshots

The `show running-config | section interface` and `show running-config | include ip nat inside source` outputs on `ASBR1` confirm that the NAT/PAT configurations were successfully applied.

![[asbr1_pat_interfaces.png]]

![[asbr1_pat_config.png]]

The `show ip nat translations` output on `ASBR1` confirms that the ICMP Echo packets originating from `FINANCE-1` (`192.168.10.11`) were being translated into the public IP address (`203.0.113.2`) while it was pinging the ISP's router, as indicated by `icmp` in the `Pro` (Protocol) column.

![[asbr1_nat_translations.png]]


##### NTP

###### Commands

```
show running-config | section ntp

show ntp status

show ntp associations
```

###### Description and Screenshots

The `show running-config | include ntp` output confirms the successful configuration of NTP on `ASBR1`. Three public Philippine-based NTP Pool servers were specified as the NTP servers with which `ASBR1` would synchronize its time.

![[asbr1_ntp_config.png]]

To enable DNS resolution, the DNS server of Cloudflare (`1.1.1.1`) and Google (`8.8.8.8`). This is necessary because the fully-qualified domain names (FQDNs) of the NTP Pool servers were configured instead of their IP addresses.

![[asbr1_dns_addresses.png]]

The `show ntp associations` and `show ntp status` outputs on `ASBR1` confirm that it synchronized its time with the NTP server at `162.159.200.123`. Two more redundant NTP servers (`222.127.4.114` and `8.220.135.46`) were listed as candidates.

![[asbr1_ntp_associations.png]]

![[asbr1_ntp_status.png]]

The `show ntp associations` and `show ntp status` outputs on `DIS1` confirm that it synchronized its time with `R1` (`192.168.254.3`). Note that, at the time of this screenshot, `R2` was disabled, thus showing the `.TIME.` value of the `ref clock` column on the first command.

![[dis1_ntp_associations.png]]

![[dis1_ntp_status.png]]

---
#### Network Security

##### Port Security

###### Commands

```
# ACC1

show running-config | section interface Ethernet1

show port-security interface Ethernet1/0

# ACC2

show running-config | section interface Ethernet1/3
```

###### Description and Screenshots

The `show running-config | section interface Ethernet1` output on `ACC1` confirms that the Port Security configurations were successfully applied to the user-facing interfaces. 

![[acc1_port_security_config.png]]

The `show port-security interface Ethernet1/0` output confirms that `ACC1` automatically learned the MAC address of the `FINANCE-1` host. ****

![[acc1_port_security_interface.png]]

Additionally, the `show port-security address` output confirms that `ACC1` learned the MAC addresses of the `FINANCE-1` and `MARKETING-1` hosts on VLAN 10 and VLAN 20 respectively.

![[acc1_port_security_addresses.png]]

On `ACC2`, the `show running-config | section interface Ethernet1/3` output confirms that the unique Port Security configuration was successfully applied. The eight-MAC address limit was configured to enable the access port to accommodate the MAC addresses of multiple VMs in the VMware ESXi host.

![[acc2_port_security_interface.png]]

##### DHCP Snooping

###### Commands

```
# ACC1
show ip dhcp snooping

# R1
show ip dhcp binding
```

###### Description and Screenshots

The `show running-config | include dhcp snooping` output on `ACC1` confirms that the successful configuration of DHCP Snooping.

![[acc1_dhcp_snooping_config.png]]

The `show ip dhcp binding` output on `R1` confirms DHCP leases to `FINANCE-1` and `MARKETING-1`. However, despite the successful configuration of DHCP Snooping on `ACC1`, and the aforementioned DHCP leases, a possible platform limitation on the Cisco IOL image in CML-Free prevented the DHCP Snooping binding table from being populated.

![[r1_dhcp_leases.png]]


##### Dynamic ARP Inspection (DAI)

###### Commands

```
# ACC1

show ip arp inspection

show ip arp inspection interfaces

show arp access-list
```

###### Description and Screenshots

The partial `show ip arp inspection` output on `ACC1` confirms the successful configuration of DAI. The ARP ACL, which contains IP-to-MAC bindings for static hosts, was also successfully applied to VLAN 40. The remainder of the command's output was omitted due to IOL platform limitations causing the DHCP Snooping binding table, on which DAI relies to validate ARP traffic, to remain empty despite successful DHCP leases as mentioned in the previous subsection.

![[acc1_dai_config.png]]

The `show ip arp inspection interfaces` output confirms that all uplink interfaces were configured as trusted ports while user-facing access ports remain untrusted. The ARP rate-limit configuration was also successfully applied to the access ports, limiting inbound ARP traffic to fifteen (15) packets per a burst interval of one (1) second.

![[acc1_dai_interfaces.png]]

The `show arp access-list` output confirms the successful configuration of IP-to-MAC bindings for static hosts, ensuring that they are still able to reply to ARP requests. Note, however, that the original `remark` commands were rejected due to IOL platform limitations.

![[acc1_arp_acl_hosts.png]]


##### BPDU Guard

###### Commands

```
# ACC1

show spanning-tree summary
```

###### Description and Screenshots

The `show spanning-tree summary` output on `ACC1` confirms that BPDU Guard was enabled globally by default. PortFast and BPDU Guard were automatically disabled upon configuration of the member interfaces of the LACP trunks.

![[acc1_bpdu_guard_config.png]]


##### Automatic Err-Disable Recovery

###### Commands

```
# ACC1

show errdisable recovery
```

###### Description and Screenshots

The `show errdisable recovery` on `ACC1` confirms that automatic err-disable recovery was enabled for DAI, DHCP rate-limit and Port Security violations, with a recovery interval of three hundred (300) seconds.

![[acc1_errdisable_recovery_config.png]]


##### Access Control Lists (ACLs)

###### Commands

```
show ip access-lists

show running-config | section Ethernet0/0.
```

###### Description and Screenshots

The `show ip access-lists` output on `R1` confirms that the ACLs were successfully configured. The entries disallow each departmental VLAN from accessing hosts on the IT Department (`192.168.40.0/24`), the management VLAN (`192.168.100.0/24`) and the loopback address subnet (`192.168.254.0/24`), as well as other departments. The departmental VLANs were permitted to access an SMB server (`192.168.40.21:445`) residing in the IT Department. The same ACLs were also configured on `R2`.

**Note**: Similar to the ARP ACL, the `remark` statements were not applied to the ACLs even though the IOL router image did not explicitly reject the commands.

![[r1_acls_config.png]]

The `show running-config | section Ethernet0/0.` output on `R1` confirms that the ACLs for VLANs 10, 20 and 30 were successfully applied on the `Ethernet0/0.10`, `Ethernet0/0.20` and `Ethernet0/0.30` subinterfaces respectively. 

![[r1_acls_subinterfaces.png]]

Despite `FINANCE-1` and `MARKETING-1` having been leased IP addresses by the DHCP server, what appeared to be a bug on the IOL images involving DHCP Snooping and DAI caused the ICMP Echo packets from `FINANCE-1` to be dropped when pinging its default gateway. Upon temporarily disabling DHCP Snooping and DAI, `FINANCE-1` could ping `R1`. 

![[finance1_ping_success.png]]

Likewise, the `show ip access-lists VLAN_10_INBOUND` output confirms that the ACLs are able to restrict user traffic between departmental VLANs as intended. Here, `FINANCE-1` attempted to ping `MARKETING-1` (`192.168.20.11`) over 500 times but the `VLAN_10_INBOUND ACL` dropped 520 ICMP Echo packets as indicated in the `(520 matches)` keyword.

![[finance1_ping_fail.png]]

![[r1_acl_hit_counter.png]]

On `ASBR1`, the `show running-config | section interface Ethernet0/1` output confirmed that the `BLOCK_BOGON_TRAFFIC ACL` was applied successfully.

![[asbr1_acl_interface.png]]

Verification of the `BLOCK_BOGON_TRAFFIC` ACL was also attempted after pinging the WAN-facing interface `Ethernet0/1` of `ASBR1` from a temporarily directly connected Alpine Linux VM for demonstration purposes. Using a bogon address as a source address when pinging from the ISP router was not allowed. An APIPA address (`169.254.1.1`) was set as the source address instead of a valid RFC 1918 address, as shown in the following tcpdump output.

![[finance1_tcpdump.png]]

The hit counter for the `deny 0.0.0.0 0.255.255.255` entry incremented instead.

![[asbr1_bogon_acl_hit_counter.png]]

**Note**: Due to the five-active node limit of CML-Free, verification of the accessibility of the SMB server in the IT Department from VLANs 10, 20 and 30 was deliberately omitted even though it was included in the network topology diagram.


##### Device Hardening

###### Commands

```
show running-config | include http

show running-config | section line aux

show running-config | section interface Ethernet0/0.




show running-config | include ip http

show running-config | exclude switchport|ip dhcp

```

###### Description and Screenshots


***Disable HTTP and HTTPS servers***

The `show running-config | include ip http` output on `ACC1` and `R1` confirms that the HTTP and HTTPS servers were successfully disabled. The same configuration was applied to all other routers and switches.

![[acc1_http_srv_disabled.png]]

![[r1_http_srv_disabled.png]]


***Disable AUX port***

The `show running-config | section line aux` output on `ACC1` and `R1` confirms that the AUX port was successfully locked down. The same configuration was applied to all other routers and switches.

![[acc1_aux_port_disabled.png]]

![[r1_aux_port_disabled.png]]


***Disable Telnet***

The `show running-config | section line vty` output on `ACC1` and `R1` confirms that only SSH was running as the CLI-based remote device management protocol. SSH 2.0 was configured as shown in the output of `show ip ssh` in the Line Security subsection of the Management Plane section above.

![[r1_vty_telnet_disabled.png]]
