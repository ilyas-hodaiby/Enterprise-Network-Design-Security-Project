# Network Security Policy

This document defines the security controls implemented in the enterprise network and the reasoning behind each one.

---

## 1. Traffic Segmentation Policy (ACLs)

### Principle

The network follows the **principle of least privilege** — each department can only communicate with what it needs to do its job. All other traffic is denied by default.

### Access Matrix

| Source \ Destination | IT (10) | HR (20) | Finance (30) | Mgmt (40) | Guest (50) | Internet |
|---|---|---|---|---|---|---|
| **IT (10)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **HR (20)** | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Finance (30)** | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ |
| **Mgmt (40)** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| **Guest (50)** | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

### ACL Implementation

ACLs are applied **inbound on VLAN interfaces** at the Layer 3 switch — traffic is filtered as close to the source as possible, before it consumes any routing resources.

```bash
! Extended ACL 110 — Inter-VLAN traffic policy
ip access-list extended INTER_VLAN_POLICY

 ! IT can reach everything — no restrictions
 permit ip 192.168.10.0 0.0.0.255 any

 ! HR cannot reach Finance
 deny   ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255

 ! Finance cannot reach HR
 deny   ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255

 ! Guest cannot reach any internal subnet
 deny   ip 192.168.50.0 0.0.0.127 192.168.0.0 0.0.255.255

 ! Everyone else — permit
 permit ip any any

! Apply to each VLAN SVI inbound
interface Vlan20
 ip access-group INTER_VLAN_POLICY in
interface Vlan30
 ip access-group INTER_VLAN_POLICY in
interface Vlan50
 ip access-group INTER_VLAN_POLICY in
```

---

## 2. Device Management Security (SSH + Management VLAN)

### Policy

Network devices (routers, switches) can only be accessed via SSH. Telnet is disabled on all devices. SSH access is restricted to the Management VLAN (192.168.40.0/27) only.

### Why Management VLAN isolation matters

If an attacker compromises a device on VLAN 20 (HR), they cannot SSH into a core switch because ACL 99 on the VTY lines drops all connections from non-management IPs. The attacker would need to already be inside VLAN 40 to attempt device management — a much harder position to reach.

```bash
! SSH only — Telnet disabled
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0

! Source IP restriction — management VLAN only
access-list 99 permit 192.168.40.0 0.0.0.31
line vty 0 4
 access-class 99 in

! SSH v2, 2048-bit RSA key
ip ssh version 2
crypto key generate rsa modulus 2048
```

---

## 3. Layer 2 Security

### Port Security

Applied on all access ports to prevent unauthorised device connections.

```bash
interface range FastEthernet0/1 - 24
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
```

- **Maximum 2 MACs:** Allows a PC + IP phone per port without triggering a violation
- **Sticky MAC:** Automatically learns and locks to the first connected device's MAC address — no manual entry needed
- **Shutdown on violation:** Port is immediately disabled if a third MAC appears — requires manual `no shutdown` to re-enable (forces IT involvement, creating an audit trail)

### BPDU Guard

Prevents rogue switches from being connected to access ports and disrupting the spanning tree topology.

```bash
interface range FastEthernet0/1 - 24
 spanning-tree portfast
 spanning-tree bpduguard enable
```

If a device sending BPDUs (like a switch or a Linux machine running STP) connects to an access port, the port is immediately shut down (`err-disabled`). This prevents STP topology changes from being triggered by unauthorised devices.

### Root Guard

Applied on distribution switch uplinks to prevent any switch from advertising itself as a better root bridge and taking over STP.

```bash
interface GigabitEthernet0/1
 spanning-tree guard root
```

If a superior BPDU arrives on this port, it is placed in `root-inconsistent` state — the port stops forwarding but is not shut down.

### DHCP Snooping

Prevents rogue DHCP servers from handing out incorrect IP addresses (a common attack that leads to MITM via gateway spoofing).

```bash
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40,50

! Only the uplink to the legitimate DHCP server is trusted
interface GigabitEthernet0/1
 ip dhcp snooping trust
```

All access ports are untrusted by default — DHCP offers from those ports are dropped.

### Dynamic ARP Inspection (DAI)

Prevents ARP spoofing/poisoning attacks where an attacker sends fake ARP replies to redirect traffic through their machine.

```bash
ip arp inspection vlan 10,20,30,40,50

! Trusted ports (uplinks to switches with legitimate devices)
interface GigabitEthernet0/1
 ip arp inspection trust
```

DAI validates ARP packets against the DHCP snooping binding table — if the IP/MAC pair in an ARP reply doesn't match a legitimate DHCP lease, the packet is dropped.

---

## 4. Wireless Security

All corporate SSIDs use **WPA2-Enterprise** (802.1X) authentication:
- Users authenticate with their domain credentials — no shared PSK
- Each user gets a unique session key — compromise of one user doesn't expose others
- Guest SSID uses WPA2-PSK with a rotating weekly password — no internal network access

---

## 5. Summary — Security Controls by Layer

| OSI Layer | Control | Protects Against |
|---|---|---|
| Layer 2 | BPDU Guard | Rogue switch / STP disruption |
| Layer 2 | Root Guard | STP topology hijacking |
| Layer 2 | Port Security | Unauthorised device connection |
| Layer 2 | DHCP Snooping | Rogue DHCP server |
| Layer 2 | Dynamic ARP Inspection | ARP spoofing / MITM |
| Layer 3 | ACLs | Unauthorised inter-VLAN access |
| Layer 3 | Management ACL (99) | Unauthorised device management |
| Layer 3 | NAT/PAT | Internal IP concealment |
| Layer 7 | SSH v2 only | Credential eavesdropping (vs Telnet) |
| Wireless | WPA2-Enterprise | WiFi credential theft |
