---
layout: post
title: Network in OpenStack
category: OpenStack
tags: [OpenStack]
---

Reference:  https://docs.openstack.org/neutron/rocky/admin/index.html   

#1, review OSI model 7 layer 
Open Systems Interconnection 
Application - Layer 7
Presentation - Layer 6
Session - Layer 5
Transport - Layer 4 - TCP 
Network   - Layer 3 - IP 
data link - layer 2 - Ethernet 
physical  - layer 1 - MAC(48 bits hexadecimal) is hardcoded in each NIC

##1.1 promiscuous mode 
By default a NIC discard an ethernet frame if MAC address does not match. 
When NIC is configured in promiscuous mode, it passes all Ethernet frames to the OS even if the MAC does not match

To know how many NIC on local host, and their corresponding MAC 
[root@controller log]# ip link show
......
2: eno49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether ec:b1:d7:a4:be:b0 brd ff:ff:ff:ff:ff:ff

##1.2 switch 
is a box of networking hardware with a large number of ports that forward Ethernet frames from one connected host to another.  Switch broadcast the unknown MAC address to all ports, it learns the mapping between MAC and ports by observing the traffic, and maintains the mapping in FIB – Forwarding Information Base, or forwarding table. 
##1.3 VLAN
Enables a single switch to act as if it was multiple independent switches. 2 hosts that are connected to the same switch but on different VLANs does not see each other   
Access Port:  carry traffic for only one VLAN  
Trunk Port: carry traffic for more than one VLAN   
##1.4 ARP
While NIC use MAC addresses to address network hosts, TCP/IP applications use IP addresses.   
ARP - Address Resolution Protocol bridges the gap between Ethernet and IP by translating IP to MAC   
IP addresses are broken up into two parts: a network number and a host identifier.   
A netmask indicates how many of the bits in 32-bite IP addresses make up the network number.   
There are 2 syntaxes for expressing a netmask:   
•	Dotted quad   
•	Classless inter-domain routing (CIDR)  
Consider an IP address of 192.0.2.5, where the first 24 bits of the address are the network number. In dotted quad notation,   
the netmask would be written as 255.255.255.0. CIDR notation includes both the IP address and netmask, and this example would be written as 192.0.2.5/24. 
```
[root@controller log]# ip a
......
2: eno49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether ec:b1:d7:a4:be:b0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.151.115/24 brd 192.168.151.255 scope global noprefixroute eno49
......
[root@controller log]# arping -I  eno49 192.168.151.116    #initiate an ARP request manually 
ARPING 192.168.151.116 from 192.168.151.115 eno49
Unicast reply from 192.168.151.116 [EC:B1:D7:A5:CA:D0]  0.847ms
[root@controller log]# arp -n      #ARP cache that contains the mapping of IP addresses to MAC address. 
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.151.100          ether   00:00:0c:07:ac:97   C                     eno49
192.168.151.251          ether   00:24:98:27:18:00   C                     eno49
192.168.151.116          ether   ec:b1:d7:a5:ca:d0   C                     eno49
```
##1.5 DHCP 
Hosts connected to a network use Dynamic Host Configuration Protocol to dynamically obtain IP addresses.   
DHCP clients locate the DHCP server by sending a UDP packet from port 68 to address 255.255.255.255 on port 67.   
DHCP server must be on the same local network as the client, or the server will not receive the broadcast.   
The DHCP server responds by sending a UDP packet from port 67 to port 68 on the client.   

Openstack uses a third program dnsmasq to implement the DHCP server

##1.6 IP - Layer3, network layer 
Internet Protocol specifies how to route packets between hosts that are connected to different networks.  
IP relies on special nework hosts called routers or gateways. 
A router has multiple IP addresses, one for each of the network it is conencted to. 

##1.7 TCP/UDP/ICMP - layer 4, transportation layer 




#2, 

# Q & A 
## Q1 - list all NICs on a host 

## when a host has multiple NICs, 
   
## How do I know what type of ethernet controllers available on my node, which link are they, MAC, IP 
```
[root@controller nova]# lspci | egrep -i --color 'network|ethernet'
06:00.0 Ethernet controller: Broadcom Inc. and subsidiaries BCM57840 NetXtreme II 10/20-Gigabit Ethernet (rev 11)
06:00.1 Ethernet controller: Broadcom Inc. and subsidiaries BCM57840 NetXtreme II 10/20-Gigabit Ethernet (rev 11)
87:00.0 Ethernet controller: Intel Corporation I350 Gigabit Backplane Connection (rev 01)
87:00.1 Ethernet controller: Intel Corporation I350 Gigabit Backplane Connection (rev 01)
87:00.2 Ethernet controller: Intel Corporation I350 Gigabit Backplane Connection (rev 01)
87:00.3 Ethernet controller: Intel Corporation I350 Gigabit Backplane Connection (rev 01)

cd /sys/bus/pci/devices
[root@controller devices]# ls -ltr ../../../devices/pci0000:80/0000:80:03.0/0000:87:00.3/net
total 0
drwxr-xr-x 5 root root 0 Jan 29 10:28 ens2f3
[root@kos2000 devices]# ls -ltr ../../../devices/pci0000:00/0000:00:02.0/0000:06:00.1/net
total 0
drwxr-xr-x 5 root root 0 Jan 29 10:28 eno50

# see all configured network devices
[root@controller devices]# ip link show
......
2: eno49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether ec:b1:d7:a4:be:b0 brd ff:ff:ff:ff:ff:ff
3: eno50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether ec:b1:d7:a4:be:b8 brd ff:ff:ff:ff:ff:ff
4: ens2f0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 5c:b9:01:8c:15:34 brd ff:ff:ff:ff:ff:ff
5: ens2f1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 5c:b9:01:8c:15:35 brd ff:ff:ff:ff:ff:ff
6: ens2f2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 5c:b9:01:8c:15:36 brd ff:ff:ff:ff:ff:ff
7: ens2f3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether 5c:b9:01:8c:15:37 brd ff:ff:ff:ff:ff:ff
......
so there are 6 NICs in total on this node, mapping between the physical ethernet controller and network devices output by ip link show: 
06:00.0 Ethernet controller: Broadcom Inc. -- 2: eno49
06:00.1 Ethernet controller: Broadcom Inc. -- 3: eno50
87:00.0 Ethernet controller:  -- 4: ens2f0
87:00.1 Ethernet controller:  -- 5: ens2f1
87:00.2 Ethernet controller:  -- 6: ens2f2
87:00.3 Ethernet controller:  -- 7: ens2f3
```





