# Design & Simulation of a Smart College Network Using VLANs

## Project Overview
A secure, segmented college campus network using VLANs, Router-on-a-Stick, ACLs, and real-traffic analysis with Wireshark.

- **Switch:** College Core Switch 2960-24TT (Star Topology)
- **Router:** College Core Router ISR4331 (Router-on-a-Stick)
- **Simulation Tool:** Cisco Packet Tracer 8.x
- **Analysis Tool:** Wireshark 4.x on Linux (HP Pavilion Laptop 16-af0xxx)
- **Host Interface:** `wlo1` (Linux Wireless) via Mobile Hotspot `172.16.222.0/24`
- **IP Method:** Static IP (No DHCP server)

---

## Day 2: VLAN & IP Addressing Plan

| VLAN ID | Name | Network | Gateway (Router Subinterface) | Device - IP - Switch Port |
|---|---|---|---|---|
| 10 | STUDENTS | 192.168.10.0/24 | 192.168.10.1 (g0/0/0.10) | Student PC - 192.168.10.10 - Fa0/1 |
| 20 | FACULTY | 192.168.20.0/24 | 192.168.20.1 (g0/0/0.20) | Faculty PC - 192.168.20.10 - Fa0/2 |
| 30 | ADMIN | 192.168.30.0/24 | 192.168.30.1 (g0/0/0.30) | Admin PC - 192.168.30.10 - Fa0/3, Server - 192.168.30.100 - Fa0/5 |
| 40 | GUESTS | 192.168.40.0/24 | 192.168.40.1 (g0/0/0.40) | Guest PC - 192.168.40.10 - Fa0/4 |

Mask: 255.255.255.0 for all. DNS: 8.8.8.8

## Day 3: Trunk & Inter-VLAN Routing

- Switch: `interface Gig0/1` -> `switchport mode trunk` -> allowed vlan 10,20,30,40
- Router: Physical `g0/0/0` up, no IP. 4 subinterfaces with `encapsulation dot1Q`

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

## Day 4: Connectivity Tests (Before ACL)

Gateway tests: All 5 devices ping their gateway - PASS
Same-VLAN: Admin 30.10 -> Server 30.100 - PASS (Fixed wiring to Fa0/3, Fa0/5)
Inter-VLAN: Student->Faculty, Student->Admin, Guest->Student etc - All PASS after Router-on-a-Stick

**Simulation Note:** Inter-VLAN PC first ARPs for its Gateway (e.g., Who has 192.168.10.1?), not destination. To see ARP+ICMP, clear cache: `arp -d` on PC + `clear arp-cache` on Router, filter only ARP+ICMP in Simulation Mode.

---

## Day 5 - Task 1: ACL Configuration (45 mins)

### Policy
- Students -> College Server (30.100) = Allow
- Students -> Admin network (30.0/24) = Deny (Server host exception)
- Guests -> Internal networks (10,20,30) = Deny
- Faculty -> All = Allow

### Configuration

```
ip access-list extended STUDENT-ACL
 remark Allow Students to College Server only
 permit ip 192.168.10.0 0.0.0.255 host 192.168.30.100
 remark Deny Students to rest of Admin network
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any

ip access-list extended GUEST-ACL
 deny ip 192.168.40.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.40.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.40.0 0.0.0.255 any

interface g0/0/0.10
 ip access-group STUDENT-ACL in
interface g0/0/0.40
 ip access-group GUEST-ACL in
```

Verification: `show access-lists` + `show ip interface g0/0/0.10 | include access list` + `show ip interface g0/0/0.40 | include access list`

### ACL Test Results

| # | Source | Destination | Expected | Actual | Reason |
|---|---|---|---|---|---|
| 1 | Student 10.10 | Server 30.100 | Allow | Reply | Rule 10 permit host |
| 2 | Student 10.10 | Admin PC 30.10 | Deny | Timeout | Rule 20 deny 30.0 |
| 3 | Student 10.10 | Faculty 20.10 | Allow | Reply | Rule 30 permit any |
| 4 | Guest 40.10 | Student 10.10 | Deny | Timeout | GUEST-ACL deny 40->10 |
| 5 | Guest 40.10 | Faculty 20.10 | Deny | Timeout | deny 40->20 |
| 6 | Guest 40.10 | Server 30.100 | Deny | Timeout | deny 40->30 |
| 7 | Faculty 20.10 | Server 30.100 | Allow | Reply | No ACL on Faculty |

**Order Matters:** Host permit must be before network deny.

---

## Day 5 - Task 2: Wireshark Packet Analysis (45 mins)

### Limitation
> Wireshark cannot capture Packet Tracer internal simulated traffic. Packet Tracer Simulation Mode is used for virtual packets. Wireshark is used separately for real traffic from host test environment (wlo1). Both show same protocols.

### Real Environment
- OS: Linux HP Pavilion 16-af0xxx, user aryansjain18
- Interface: `wlo1` (Linux Wireless LAN, equivalent to Wi-Fi on Windows)
- Connection: Mobile Hotspot
- Host IP: `172.16.222.235/24` on wlo1 (`ip a | grep wlo1`)
- Gateway (Phone): `172.16.222.1` (`ip r | grep wlo1` shows default via)
- Note: Android 12+ hotspot uses 172.16.x.x, not 192.168.43.x

### Capture Steps
1. Install: `sudo apt install wireshark -y` + `sudo usermod -a -G wireshark $USER` (re-login) or `sudo wireshark`
2. Select `wlo1`, start capture
3. Generate:
   ```
   ping -c 4 172.16.222.1
   ping -c 4 8.8.8.8
   nslookup google.com
   # browse google.com
   ```
4. Stop, Save as `Day5_Wireshark_wlo1_172.16.pcapng`

### Filters

| Filter | Displays | Observation |
|---|---|---|
| `arp` | ARP | Who has 172.16.222.1? Tell 172.16.222.235 + Phone MAC |
| `icmp` | ICMP ping | Type 8 Request to 172.16.222.1, Type 0 Reply. External 8.8.8.8 may be blocked by carrier, so gateway ping used |
| `dns` | DNS | Query A google.com, Response 142.250.x.x |
| `tcp` | TCP | SYN, SYN-ACK, ACK to 142.250.x.x:443 |
| `udp` | UDP | UDP 53 for DNS |

Deliverable: `.pcapng` + screenshots of filters + packet details (Ethernet, IP, ICMP layers expanded)

---

## Day 5 - Task 3: Test the Complete Network (30 mins)

### IP Addressing Note
> **No DHCP implemented.** All PCs and Server use Static IP addressing manually configured (IP, Mask, Gateway, DNS). Therefore, "Student receives valid address" is verified via `ipconfig` showing static config, not DHCP lease.

### Final Test Results Table - Deliverable

| Test | Command / Check | Expected result | Actual Result | Status | Evidence |
|---|---|---|---|---|---|
| **Student PC receives a valid address** | `ipconfig` on Student PC | Valid IP in 192.168.10.0/24, Mask 255.255.255.0, Gateway 192.168.10.1 | Static: 192.168.10.10/24, GW 192.168.10.1, DNS 8.8.8.8 - Manually configured | **Pass** | `27_Student_IPConfig_Static.png` |
| **Student reaches its gateway** | `ping 192.168.10.1` from Student | Reply | Reply from 192.168.10.1 bytes=32 TTL=255 | **Pass** | `28_Student_Gateway_Ping.png` |
| **Permitted student-to-server communication** | `ping 192.168.30.100` from Student | Pass - STUDENT-ACL Rule 10 permit host 30.100 | Reply from 192.168.30.100 | **Pass** | `29_Student_Server_Allow.png` |
| **Student attempts to access admin devices** | `ping 192.168.30.10` from Student | **Blocked** - STUDENT-ACL Rule 20 deny 30.0 | Request timed out / Destination unreachable. Hit count on `show access-lists` increases | **Blocked (Expected)** | `30_Student_Admin_Blocked.png` |
| **Guest attempts to access internal resources** | `ping 192.168.10.10` `20.10` `30.100` from Guest | **Blocked** - GUEST-ACL deny 10,20,30 | All 3: Request timed out. Hits on GUEST-ACL 10,20,30 | **Blocked (Expected)** | `31_Guest_Internal_Blocked.png` |
| **Faculty accesses permitted resources** | `ping 192.168.30.100` from Faculty | Pass - No ACL on g0/0/0.20 | Reply from 192.168.30.100 | **Pass** | `32_Faculty_Server_Allow.png` |

## Troubleshooting Log

| Issue | Cause | Fix |
|---|---|---|
| Same-VLAN timeout | Cables on Fa0/6, Fa0/8 not Fa0/3, Fa0/5 | Port labels enabled, rewired to match show vlan brief |
| Router flapping | Gig0/1 not trunk | switchport mode trunk |
| show interfaces trunk invalid on Router | Switch-only command | Used show ip interface brief on Router |
| Simulation only ICMP | ARP cache cached | arp -d + clear arp-cache + filter ARP+ICMP |
| Simulation only ARP | Simple PDU expired after gateway ARP | Used ping inside Simulation Mode, not Simple PDU |
| Wireshark no Wi-Fi | Linux uses wlo1 not Wi-Fi | Used wlo1 interface |
| Wireshark permission | Not in wireshark group | usermod -a -G wireshark + re-login or sudo wireshark |
| Hotspot IP 172.16.222.x not 192.168.43.x | Android 12+ hotspot range | Documented as observation |
