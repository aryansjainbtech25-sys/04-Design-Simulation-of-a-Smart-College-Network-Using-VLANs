# Design & Simulation of a Smart College Network Using VLANs

## Project Overview
A secure, segmented college campus network using VLANs, Router-on-a-Stick, ACLs, and real-traffic analysis with Wireshark.

- **Switch:** College Core Switch 2960-24TT (Star Topology)
- **Router:** College Core Router ISR4331 (Router-on-a-Stick)
- **Simulation Tool:** Cisco Packet Tracer 8.x
- **Analysis Tool:** Wireshark 4.x on Linux (HP Pavilion Laptop)
- **Host Interface:** `wlo1` (Linux Wireless) via Mobile Hotspot `172.16.222.0/24`

---

## Day 2: VLAN & IP Addressing Plan

| VLAN ID | Name | Network | Subnet Mask | Gateway (Router Subinterface) | Device - IP - Switch Port |
|---|---|---|---|---|---|
| 10 | STUDENTS | 192.168.10.0/24 | 255.255.255.0 | 192.168.10.1 (g0/0/0.10) | Student PC - 192.168.10.10 - Fa0/1 |
| 20 | FACULTY | 192.168.20.0/24 | 255.255.255.0 | 192.168.20.1 (g0/0/0.20) | Faculty PC - 192.168.20.10 - Fa0/2 |
| 30 | ADMIN | 192.168.30.0/24 | 255.255.255.0 | 192.168.30.1 (g0/0/0.30) | Admin PC - 192.168.30.10 - Fa0/3, Server - 192.168.30.100 - Fa0/5 |
| 40 | GUESTS | 192.168.40.0/24 | 255.255.255.0 | 192.168.40.1 (g0/0/0.40) | Guest PC - 192.168.40.10 - Fa0/4 |

### Switch VLAN Verification
```
Switch#show vlan brief
10 STUDENTS active Fa0/1
20 FACULTY active Fa0/2
30 ADMIN active Fa0/3, Fa0/5
40 GUESTS active Fa0/4
```

## Day 3: Trunk & Inter-VLAN Routing

- Switch: `interface Gig0/1` -> `switchport mode trunk` -> `switchport trunk allowed vlan 10,20,30,40`
- Router: Physical `g0/0/0` up, no IP. 4 subinterfaces:

```
interface g0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface g0/0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

interface g0/0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0

interface g0/0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
```

Router Verification:
```
GigabitEthernet0/0/0 unassigned up up
Gig0/0/0.10 192.168.10.1 up up
Gig0/0/0.20 192.168.20.1 up up
Gig0/0/0.30 192.168.30.1 up up
Gig0/0/0.40 192.168.40.1 up up
```

---

## Day 4: Connectivity Test Results

### A. Gateway Tests - PASS

| # | Source | Destination | Command | Result |
|---|---|---|---|---|
| 1 | Student PC (10.10) | 192.168.10.1 | `ping 192.168.10.1` | ✅ Reply |
| 2 | Faculty PC (20.10) | 192.168.20.1 | `ping 192.168.20.1` | ✅ Reply |
| 3 | Admin PC (30.10) | 192.168.30.1 | `ping 192.168.30.1` | ✅ Reply |
| 4 | College Server (30.100) | 192.168.30.1 | `ping 192.168.30.1` | ✅ Reply |
| 5 | Guest PC (40.10) | 192.168.40.1 | `ping 192.168.40.1` | ✅ Reply |

### B. Same-VLAN Test

| # | Source | Destination | Type | Result | Explanation |
|---|---|---|---|---|---|
| 6 | Admin PC (30.10) | Server (30.100) | Same-VLAN VLAN 30 | ✅ Reply | Both in VLAN 30 (Fa0/3, Fa0/5), Layer 2 switched locally. Fixed earlier cabling mismatch where cables were on Fa0/6, Fa0/8 (VLAN 1). |

### C. Inter-VLAN Tests - After Router-on-a-Stick

| # | Source | Destination | Type | Result | Explanation |
|---|---|---|---|---|---|
| 7 | Student (10.10) | Faculty (20.10) | 10->20 | ✅ Reply | Tagged 10 -> Router g0/0/0.10 -> Routing table -> Tagged 20 -> Fa0/2 |
| 8 | Student (10.10) | Admin (30.10) | 10->30 | ✅ Reply | Router pings 100% `!!!!!` |
| 9 | Student (10.10) | Guest (40.10) | 10->40 | ✅ Reply | Allowed, will be filtered later |
| 10 | Faculty (20.10) | Server (30.100) | 20->30 | ✅ Reply | Faculty access to College Server |
| 11 | Guest (40.10) | Student (10.10) | 40->10 | ✅ Reply | Reverse path works |
| 12 | Admin (30.10) | Guest (40.10) | 30->40 | ✅ Reply | Admin manages Guest |

### Simulation Mode Observations

**Without ARP cache (after `arp -d` on PC + `clear arp-cache` on Router):**
- Admin PC wants to ping Student (different subnet) -> ARPs for Gateway 30.1, not 10.10
- ARP Request: Who has 192.168.30.1? Tell 192.168.30.10
- ARP Reply: 192.168.30.1 is at Router MAC
- ICMP Echo Request -> Router -> Router ARPs for 192.168.10.10 -> ICMP Reply
- This is why filtering only ARP+ICMP and clearing cache is needed to see full flow.

**With ARP cache:** Only ICMP seen, because MACs cached. STP/DTP are background control protocols.

### Troubleshooting Applied (Day 4)

| Issue | Root Cause | Fix |
|---|---|---|
| Same-VLAN ping timeout | Wired to Fa0/6, Fa0/8 but VLAN assigned to Fa0/3, Fa0/5 | Enabled Port Labels, rewired to Fa0/1-5 matching `show vlan brief` |
| Router interface flapping | Gig0/1 not trunk | `switchport mode trunk` on Switch |
| `show interfaces trunk` invalid on Router | Command is Switch-only | Used `show ip interface brief` on Router |
| Simulation only ICMP, no ARP | ARP cache populated | `arp -d` + `clear arp-cache`, filter ARP+ICMP only |
| Simulation only ARP, no ICMP | Simple PDU expired after ARP for gateway | Used ping command inside Simulation Mode, not Simple PDU, with Auto Capture |

---

## Day 5 - Task 1: ACL Configuration

### Policy Implemented

| Source | Destination | Expected | ACL |
|---|---|---|---|
| Students | College learning server (30.100) | Allow | STUDENT-ACL permit host |
| Students | Administration network (30.0/24) | Deny | STUDENT-ACL deny 30.0 |
| Guests | Internal college networks (10,20,30) | Deny | GUEST-ACL deny 10,20,30 |
| Faculty | Approved college resources | Allow | No ACL - full access |

### ACL Configuration (Router)

```
ip access-list extended STUDENT-ACL
 remark Allow Students to College Server only
 permit ip 192.168.10.0 0.0.0.255 host 192.168.30.100
 remark Deny Students to rest of Admin network
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 remark Allow everything else
 permit ip 192.168.10.0 0.0.0.255 any

ip access-list extended GUEST-ACL
 remark Deny Guests to all Internal College Networks
 deny ip 192.168.40.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.30.0 0.0.0.255
 remark Allow Guests to Internet only
 permit ip 192.168.40.0 0.0.0.255 any

interface g0/0/0.10
 ip access-group STUDENT-ACL in

interface g0/0/0.40
 ip access-group GUEST-ACL in
```

Verification:
```
show access-lists
show ip interface g0/0/0.10 | include access list -> Inbound is STUDENT-ACL
show ip interface g0/0/0.40 | include access list -> Inbound is GUEST-ACL
```

### ACL Test Results - Deliverable

| # | Source | Destination | Expected | Actual | Reason |
|---|---|---|---|---|---|
| 1 | Student 10.10 | Server 30.100 | Allow | ✅ Reply | Rule 10: permit host 30.100 |
| 2 | Student 10.10 | Admin PC 30.10 | Deny | ❌ Timeout | Rule 20: deny 10.0 -> 30.0 |
| 3 | Student 10.10 | Faculty 20.10 | Allow | ✅ Reply | Rule 30: permit any |
| 4 | Guest 40.10 | Student 10.10 | Deny | ❌ Timeout | GUEST-ACL deny 40 -> 10 |
| 5 | Guest 40.10 | Faculty 20.10 | Deny | ❌ Timeout | GUEST-ACL deny 40 -> 20 |
| 6 | Guest 40.10 | Server 30.100 | Deny | ❌ Timeout | GUEST-ACL deny 40 -> 30 |
| 7 | Faculty 20.10 | Server 30.100 | Allow | ✅ Reply | No ACL on Faculty subinterface |

**Order Matters:** Host permit must be before network deny, otherwise Server would be blocked.

---

## Day 5 - Task 2: Wireshark Packet Analysis

### Important Limitation
> Wireshark captures real traffic on real interfaces (wlo1, eth0). It does NOT automatically capture Packet Tracer simulated traffic. Packet Tracer traffic is internal to the simulator. To bridge them you need Cloud device. Therefore, I used Packet Tracer Simulation Mode for virtual packets and Wireshark separately for real packets from my test environment. Both show same protocols (ARP, ICMP, DNS, TCP).

### Test Environment (Real)
- **OS:** Linux - HP Pavilion Laptop 16-af0xxx (aryansjain18@aryansjain18-HP-Pavilion-Laptop-16-af0xxx)
- **Interface:** `wlo1` (Linux name for Wi-Fi, equivalent to Wi-Fi on Windows)
- **Connection:** Mobile Hotspot
- **Host IP:** `172.16.222.235/24` on wlo1
- **Gateway (Phone):** `172.16.222.1` (Android 12+ hotspot uses 172.16.x.x range, not 192.168.43.x)
- **Verification:** `ip a | grep wlo1` and `ip r | grep wlo1`

### Capture Procedure
1. Install: `sudo apt install wireshark -y` + `sudo usermod -a -G wireshark $USER`
2. Start Wireshark, select interface `wlo1`, start capture
3. Generate traffic:
   ```
   ping -c 4 172.16.222.1
   ping -c 4 8.8.8.8
   nslookup google.com
   # Browse google.com
   ```
4. Stop capture, Save as `Day5_Wireshark_wlo1_172.16.pcapng`

### Display Filters Used

| Filter | What it displays | Observation on Hotspot (wlo1) |
|---|---|---|
| `arp` | Address Resolution Protocol | `Who has 172.16.222.1? Tell 172.16.222.235` + Reply with Phone MAC (02:xx:xx) |
| `icmp` | ICMP traffic, ping | Type 8 Echo Request to 172.16.222.1, Type 0 Echo Reply. External 8.8.8.8 may be blocked by carrier (Jio/Airtel) - gateway ping used as proof |
| `dns` | DNS queries/responses | Query A google.com, Response 142.250.x.x - proves DNS via mobile data |
| `tcp` | TCP packets | TCP 3-way handshake SYN, SYN-ACK, ACK to 142.250.x.x:443 (HTTPS) |
| `udp` | UDP packets | UDP port 53 for DNS, QUIC for Chrome |

### Packet Details (ICMP Example)
- Frame: Timestamp, size on wlo1
- Ethernet: Src = Laptop MAC (wlo1), Dst = Phone Hotspot MAC
- IP: Src = 172.16.222.235, Dst = 172.16.222.1
- ICMP: Type 8 = Echo Request, Type 0 = Echo Reply, Code 0
