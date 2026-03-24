# Enterprise Network Design & Security Project

A simulated enterprise network project demonstrating design, routing, and security implementation.

## Project Overview
This project was completed as part of an academic program at ENSET University. It focuses on the design, implementation, and simulation of a secure enterprise network for a medium-sized organization.

The network was designed to support **100 employees distributed across five floors**, ensuring scalability, availability, and security.

## 🧠 Design Approach

- The network is segmented using VLANs based on departments
- Inter-VLAN routing is handled through Layer 3 devices
- OSPF is used for dynamic routing across the network
- ACLs are implemented to restrict communication between sensitive departments
- Redundancy is achieved using HSRP and EtherChannel
  
## Network Architecture
- Five-floor enterprise building
- Dedicated technical room on each floor
- Wired and wireless connectivity
- Network printers and access points on each floor

## Technologies & Protocols Used
- VLANs, Trunking
- VTP, STP
- NAT & PAT
- VLSM & CIDR
- DHCP
- OSPF Routing
- EtherChannel
- HSRP (High Availability)
- Access Control Lists (ACLs)
- Network security best practices

## 🛠️ Tools Used

- Cisco Packet Tracer
- Network simulation and testing environment

## Project Objectives
- Design a scalable and secure enterprise network
- Ensure high availability and fault tolerance
- Segment departments using VLANs
- Implement secure routing and traffic control
- Simulate a real-world enterprise environment

## Skills Demonstrated
- Network design & architecture
- Routing and switching
- Network security fundamentals
- Troubleshooting & simulation
- Team collaboration

## Academic Context
- Institution: ENSET Mohammedia  
- Duration: 2 weeks  
- Type: Academic / Practical Project

## 📸 Network Topology

![Network Topology](network-topology.png.jpeg)

## ⚙️ Configuration Examples

### OSPF, ACL & VLAN Configuration (Core Devices)

```bash
! ===== OSPF =====
! Dynamic routing configuration

router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0

! ===== ACL =====
! Traffic filtering between departments

access-list 100 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 100 permit ip any any

! ===== VLAN =====
! Network segmentation

vlan 10
 name IT

vlan 20
 name HR

vlan 30
 name Finance

! ===== DHCP =====
! Automatic IP address assignment for clients

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8

ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
