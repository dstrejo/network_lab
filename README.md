# network_lab
Branch office and headquarter 

This is a lab from the Udemy course **Secure Networking - A Company Network Project on Open-Source** created by Seyed Farshid Miri. 
Note: The configuration of the lab is not complete and this is only a part of it.

Software used:
* GNS3: 2.2.33.1
* Linux Cumulus 4.19.149-1+cl4.3u1 (2021-01-28) x86_64 GNU/Linux
* Linux AlpineLinux-1 5.8.0-48-generic #54~20.04.1-Ubuntu
* openSUSE Leap 15.3-1
* pfSense 2.5.2
* Ubuntu Server 18.04.6
* Linux KaliLinuxCLI-1 5.8.0-48-generic #54~20.04.1-Ubuntu
* VirtualBox 7.0.18 r162988 (Qt5.15.2)
  
Objective: Configure a secure network utilizing open source equipment simulating the branch office and the headquarter where both LANs have to be connected. 
* Create the templates in GNS3 with the OS mentioned aboved for servers, switches, nodes and firewalls.
* On switches: Add virtual interface for managmenet VLAN, add VLANs, configure access ports, configuring bonding.
* Create bond interfaces on openSUSE via tool yast2 (GUI configuration). Add VLANs intefaces linked to bond0. Install and configure keepalived (VRRP).
* DHCP Server
* iptables -> translate to nftables
* Configure pfSense and VPNs -> GUI configuration

# GNS3 Open-Source Lab Configuration

## Necessary Configuration

### 1. Create the Templates
- Set up the templates in GNS3 with the mentioned OS for servers, switches, nodes, and firewalls.

### 2. Switch Configuration
- Add a virtual interface for the management VLAN.
- Add VLANs.
- Configure access ports.
- Configure bonding.

### 3. openSUSE Configuration
- Create bond interfaces via the tool `yast2`.
- Add VLAN interfaces linked to `bond0`.
- Install and configure `keepalived` (VRRP).

### 4. DHCP Server on openSUSE-Infra

## Example Configurations
```sh
--BS-LEAF-2--

*VLAN*
net show bridge vlan
net add bridge bridge vids 10,20,30,40
Change Default VLAN 1
net del bridge bridge vids 1
Change from Trunk to Access Port and Add to VLAN 10
net add interface swp3-5 bridge access 10

*Port Security*
net add interface swp3-5 stp bpduguard
net add interface swp3-5 stp portadminedge
VLAN Management, Virtual Interface, and Assigned to VLAN 40
net add vlan 40 ip address 20.10.40.101/24
net add vlan 40 vlan-id 40

*LACP*
net del bridge bridge ports swp2,6
net commit
net add bond bond1 bond slaves swp2,6
net add bond bond1 bond mode 802.3ad
net add bridge bridge ports bond1
Check interface status:
net show int all
Note: The bond won't come up until it is configured on the other side of the switch as well. Parameters like duplex and speed, among others, MUST be the same on both sides.

-- FFM-Spine-1 --
Basic Configuration same as for LEAF switches. 
net add hostname FFM-Spine-1
net add bridge bridge ports swp2,3,5
net add bridge bridge vids 10,20,30,40
net add vlan 40 ip address 10.10.40.111/24
*MLAG Configuration*
net del bridge bridge ports swp2,3,5
net commit
net add bond bond1 bond slaves swp2
net add bond bond2 bond slaves swp5
net add bond bond3 bond slaves swp3
net add bond bond4 bond slaves swp4
net add bond bond5 bond slaves swp6
net add clag peer sys-mac 44:38:39:BE:EF:AA interface swp1,15 primary backup-ip 30.30.30.2
CLAG IDs (must match on the other side of the bond)
net add bond bond1 clag id 1
net add bond bond2 clag id 2
net add bond bond3 clag id 3
net add bond bond4 clag id 4
net add bond bond5 clag id 5
net add bridge bridge ports bond4-5
net add bridge bridge ports bond1-3
net commit

-- Firewall configuration keepalived and iptables FW1 --

FFM-openSUSE-Fw1 Configuration

FFM-openSUSE-Fw1:/etc/sysconfig/network # cat /etc/keepalived/keepalived.conf
vrrp_instance internal {
    state MASTER
    interface bond1
    virtual_router_id 51
    priority 255
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 12345
    }
    virtual_ipaddress {
        10.10.10.1/24 dev vlan10
        10.10.20.1/24 dev vlan20
        10.10.30.1/24 dev vlan30
        10.10.40.1/24 dev vlan40
    }
}
FFM-openSUSE-Fw2 Configuration

FFM-openSUSE-Fw2:~ # cat /etc/keepalived/keepalived.conf
vrrp_instance internal {
    state BACKUP
    interface bond1
    virtual_router_id 51
    priority 254
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 12345
    }
    virtual_ipaddress {
        10.10.10.1/24 dev vlan10
        10.10.20.1/24 dev vlan20
        10.10.30.1/24 dev vlan30
        10.10.40.1/24 dev vlan40
    }
}

*iptables*

*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [212:21712]
-A INPUT -i vlan40 -p tcp -m tcp --dport 22 -m comment --comment "Allow SSH from management VLAN40 to FW1" -j ACCEPT
-A INPUT -i vlan10 -p icmp -m comment --comment "Allow ICMP traffic from VLAN10" -j ACCEPT
-A INPUT -i lo -m comment --comment "Allow loopback traffic" -j ACCEPT
-A INPUT -s 192.168.100.0/24 -i bond1 -m comment --comment "Allow all traffic to bond1 from 192.168.100.0/24" -j ACCEPT
-A INPUT -m conntrack --cstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i vlan20 -o eth1 -m comment --comment "Allow traffic from VLAN20 to internet" -j ACCEPT
-A FORWARD -m conntrack --cstate REALTED,ESTABLISHED -m comment --comment "Allow established traffic" -j ACCEPT
-A FORWARD -i vlan40 -o eth1 -p tcp -m multiport --dports 443,80 -m comment --comment "Allow management VLAN40 to access https/s" -j ACCEPT
-A FORWARD -i vlan40 -o eth1 -p udp -m --dport 53 -m comment --comment "Allow DNS from management VLAN40 to internet" -j ACCEPT
-A FORWARD -i vlan40 -p icmp -m comment --comment "Allow ICPM from management VLAN to anywhere" -j ACCEPT
-A FORWARD -i vlan40 -o vlan20 -m comment --comment "Alllow any traffic from VLAN40 to VLAN20" -j ACCEPT
-A FORWARD -i vlan40 -o vlan30 -p tcp -m multiport --dports 22,80 -m comment --comment "Alllow SSH/HTTP traffic from VLAN40 to VLAN30" -j ACCEPT
-A FORWARD -i vlan10 -o eth1 -p tcp -m multiport --dports 443,80 -m comment --comment "Allow management VLAN10 to access https/s" -j ACCEPT
-A FORWARD -i vlan10 -o eth1 -p udp -m --dport 53 -m comment --comment "Allow DNS from  VLAN10 to internet" -j ACCEPT
-A FORWARD -i vlan10 -o vlan20 -p tcp -m multiport --dports 445,139 -m comment --comment "Allow SMB from vlan 10 to VLAN20" -j ACCEPT
-A FORWARD -i vlan10 -o vlan20 -p udp -m multiport --dports 137,138 -m comment --comment "Alllow SMB/UDP from vlan 10 to VLAN20" -j ACCEPT
-A FORWARD -o vlan20 -p tcp -m multiport --dports 443,80 -m comment --comment "Alllow https/s from anywhere to VLAN20" -j ACCEPT
-A FORWARD -i vlan30 -o eth1 -p tcp -m multiport --dports 443,80 -m comment --comment "Alllow traffic from DMZ to https" -j ACCEPT
-A FORWARD -i vlan30 -o eth1 -p upd -m --dport 53 -m comment --comment "Alllow DNS traffic from DMZ to internet" -j ACCEPT
-A FORWARD -p tcp -d 10.10.30.201 --dport 80 -m state --cstate NEW;ESTABLISHED;RELATED -j ACCEPT
COMMIT
*nat
:PREROUTING ACCEPT [2504:329944]
:INPUT ACCEPT [1:60]
:OUTPUT ACCEPT [1260:101301]
:POSTROUTING ACCEPT [3:224]
-A POSTROUTING -o eth1 -m comment --comment "NAT traffic to internet" -j MASQUERADE
-A PREROUTING -p tcp -i eth1 --dport 80 -j DNAT --to-destination 10.10.30.201:80
COMMIT

*DCHP Server*
option domain-name-servers 8.8.8.8;
default-lease-time 14400;
ddns-update-style none;

subnet 10.10.10.0 255.255.255.0 {
	range 10.10.10.50 10.10.10.220;
	option router 10.10.10.1;
	default-lease-time 14400;
	max-lease-time 172800;	
}

subnet 10.10.40.0 255.255.255.0 {
	range 10.10.40.50 10.10.40.220;
	option router 10.10.40.1;
	default-lease-time 14400;
	max-lease-time 172800;	
}
