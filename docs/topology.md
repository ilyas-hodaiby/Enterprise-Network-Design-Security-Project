# Network Topology — Floor by Floor Breakdown

This document describes the physical and logical layout of the enterprise network, floor by floor.

---

## Building Overview

| Floor | Department(s) | VLAN(s) | Approx. Users | Access Switches |
|---|---|---|---|---|
| Floor 1 | IT Department | VLAN 10 | 20 | 2× access switches |
| Floor 2 | HR Department | VLAN 20 | 25 | 2× access switches |
| Floor 3 | Finance | VLAN 30 | 20 | 2× access switches |
| Floor 4 | Management / Executive | VLAN 40 | 15 | 1× access switch |
| Floor 5 | General Staff / Guest WiFi | VLAN 10, 50 | 20 | 2× access switches |

---

## Core Infrastructure

### Edge Router
- **Location:** Server room, Floor 1
- **Role:** Internet gateway, NAT/PAT, OSPF edge, DHCP server
- **Connects to:** ISP (WAN) and Core Layer 3 switch (LAN)
- **Key config:** NAT overload on WAN interface, OSPF Area 0 on LAN interface

### Core Layer 3 Switches (HSRP Pair)
- **Location:** Main technical room, Floor 1
- **Role:** Inter-VLAN routing, OSPF backbone, HSRP gateways for all VLANs
- **Active switch:** Priority 110, preempt enabled
- **Standby switch:** Priority 100, preempt enabled
- **Virtual IPs:** One per VLAN (e.g. 192.168.10.254 for VLAN 10)
- **Uplink to edge router:** Routed link (not trunk), OSPF neighbour

---

## Floor 1 — IT Department

```
Core L3 Switch
      │
      │ Trunk (VLANs 10, 40)
      │
Distribution Switch FL1
      │               │
      │ Trunk         │ Trunk
      │               │
Access SW FL1-A    Access SW FL1-B
  (ports 1-24)      (ports 1-24)
PCs, Printers      PCs, AP
VLAN 10            VLAN 10
```

- 20 workstations — VLAN 10
- 1 network printer — static IP 192.168.10.200
- 1 wireless AP — static IP 192.168.10.210, broadcasts SSID "Enterprise-IT"
- Technical room houses patch panel and distribution switch
- IT VLAN has access to all other VLANs (admin privilege via ACL)

---

## Floor 2 — HR Department

```
Core L3 Switch
      │
      │ Trunk (VLANs 20, 40)
      │
Distribution Switch FL2
      │               │
Access SW FL2-A    Access SW FL2-B
VLAN 20            VLAN 20
PCs, Printer       PCs, AP
```

- 25 workstations — VLAN 20
- 1 network printer — static IP 192.168.20.200
- 1 wireless AP — static IP 192.168.20.210, SSID "Enterprise-HR"
- ACL blocks HR from reaching Finance (VLAN 30) in both directions

---

## Floor 3 — Finance Department

```
Core L3 Switch
      │
      │ Trunk (VLANs 30, 40)
      │
Distribution Switch FL3
      │               │
Access SW FL3-A    Access SW FL3-B
VLAN 30            VLAN 30
PCs, Printer       PCs, AP
```

- 20 workstations — VLAN 30
- 1 network printer — static IP 192.168.30.200
- 1 wireless AP — VLAN 30 only (no guest WiFi on Finance floor)
- Most restricted VLAN — ACL denies access from all non-IT VLANs
- Finance can initiate traffic to internet (via NAT) but not to HR or General Staff

---

## Floor 4 — Management / Executive

```
Core L3 Switch
      │
      │ Trunk (VLAN 40)
      │
Distribution Switch FL4
      │
Access SW FL4
VLAN 40
PCs, AP
```

- 15 workstations — VLAN 40
- 1 wireless AP — SSID "Enterprise-Exec"
- VLAN 40 is also the management VLAN — only source allowed to SSH to network devices
- ACL 99 on all device VTY lines: only 192.168.40.0/27 can SSH in

---

## Floor 5 — General Staff + Guest WiFi

```
Core L3 Switch
      │
      │ Trunk (VLANs 10, 50)
      │
Distribution Switch FL5
      │               │
Access SW FL5-A    Access SW FL5-B
VLAN 10            VLAN 50 (Guest)
Staff PCs          AP (Guest SSID)
```

- 20 general staff workstations — VLAN 10
- 1 AP broadcasting two SSIDs:
  - "Enterprise-Corp" → VLAN 10 (staff)
  - "Enterprise-Guest" → VLAN 50 (guest — internet only)
- Guest VLAN 50 is blocked from all internal subnets via ACL
- Guest VLAN uses a separate DHCP pool with a 4-hour lease

---

## Trunk Links Summary

All inter-switch links are configured as 802.1Q trunks, carrying only the VLANs needed on each link (not all VLANs — native VLAN pruned for security).

| Link | VLANs Allowed |
|---|---|
| Core → Distribution FL1 | 10, 40 |
| Core → Distribution FL2 | 20, 40 |
| Core → Distribution FL3 | 30, 40 |
| Core → Distribution FL4 | 40 |
| Core → Distribution FL5 | 10, 50 |
| Distribution → Access (all floors) | Floor-specific VLANs only |

Restricting allowed VLANs on each trunk prevents traffic from one department from ever reaching a switch it has no business reaching — even before ACLs.
