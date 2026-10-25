# Design & Simulation of a Smart College Network Using VLANs

## Network Design
- **Switch:** College Core Switch 2960-24TT
- **Router:** College Core Router ISR4331
- **Topology:** Star - All PCs and Server connected to Switch, Router connected via Gig0/1 to Gig0/0/0

### VLAN & IP Addressing Plan (Day 2)

| VLAN ID | Name | Network | Subnet Mask | Gateway (Router Subinterface) | Devices |
|---|---|---|---|---|---|
| 10 | STUDENTS | 192.168.10.0/24 | 255.255.255.0 | 192.168.10.1 (g0/0/0.10) | Student PC - 192.168.10.10 - Fa0/1 |
| 20 | FACULTY | 192.168.20.0/24 | 255.255.255.0 | 192.168.20.1 (g0/0/0.20) | Faculty PC - 192.168.20.10 - Fa0/2 |
| 30 | ADMIN | 192.168.30.0/24 | 255.255.255.0 | 192.168.30.1 (g0/0/0.30) | Admin PC - 192.168.30.10 - Fa0/3, Server - 192.168.30.100 - Fa0/5 |
| 40 | GUESTS | 192.168.40.0/24 | 255.255.255.0 | 192.168.40.1 (g0/0/0.40) | Guest PC - 192.168.40.10 - Fa0/4 |

### Trunk Configuration (Day 3)
- Switch: Gig0/1 -> `switchport mode trunk`, allows VLAN 1,10,20,30,40
- Router: Gig0/0/0 physical up, 4 subinterfaces with `encapsulation dot1Q <vlan-id>` + IP

Router Verification:
```
GigabitEthernet0/0/0 unassigned up up
Gig0/0/0.10 192.168.10.1 up up
Gig0/0/0.20 192.168.20.1 up up
Gig0/0/0.30 192.168.30.1 up up
Gig0/0/0.40 192.168.40.1 up up
```

---

## Day 4 - Connectivity Test Results

### A. Gateway Tests - Expected: SUCCESS

| # | Source | Destination | Command | Actual | Status |
|---|---|---|---|---|---|
| 1 | Student PC (10.10) | 192.168.10.1 | `ping 192.168.10.1` | Reply | ✅ PASS |
| 2 | Faculty PC (20.10) | 192.168.20.1 | `ping 192.168.20.1` | Reply | ✅ PASS |
| 3 | Admin PC (30.10) | 192.168.30.1 | `ping 192.168.30.1` | Reply | ✅ PASS |
| 4 | College Server (30.100) | 192.168.30.1 | `ping 192.168.30.1` | Reply | ✅ PASS |
| 5 | Guest PC (40.10) | 192.168.40.1 | `ping 192.168.40.1` | Reply | ✅ PASS |

### B. Same-VLAN Test - Expected: SUCCESS

| # | Source | Destination | Type | Actual | Status | Explanation |
|---|---|---|---|---|---|---|
| 6 | Admin PC (30.10) | Server (30.100) | Same-VLAN (VLAN 30) | Reply | ✅ PASS | Both in VLAN 30 (Fa0/3, Fa0/5), switched locally at Layer 2, no router needed. Verified after fixing wiring to match `show vlan brief`. |

### C. Inter-VLAN Tests - Expected: SUCCESS after Router-on-a-Stick

| # | Source | Destination | Type | Actual | Status | Explanation |
|---|---|---|---|---|---|---|
| 7 | Student (10.10) | Faculty (20.10) | Inter-VLAN 10->20 | Reply | ✅ PASS | Packet: PC -> Fa0/1 (VLAN10) -> Trunk Gig0/1 tagged 10 -> Router g0/0/0.10 (10.1) -> Routing table -> g0/0/0.20 tagged 20 -> Fa0/2 |
| 8 | Student (10.10) | Admin (30.10) | Inter-VLAN 10->30 | Reply | ✅ PASS | Verified via Router pings: `!!!!! 100%` success. Requires `arp -d` before test to clear stale cache. |
| 9 | Student (10.10) | Guest (40.10) | Inter-VLAN 10->40 | Reply | ✅ PASS | Currently allowed. Can be blocked later with ACL on Router if Guest isolation policy required. |
| 10 | Faculty (20.10) | Server (30.100) | Inter-VLAN 20->30 | Reply | ✅ PASS | Faculty accessing College Server resource in ADMIN VLAN. |
| 11 | Guest (40.10) | Student (10.10) | Inter-VLAN 40->10 | Reply | ✅ PASS | Reverse direction also works, proving full inter-VLAN routing. |
| 12 | Admin (30.10) | Guest (40.10) | Inter-VLAN 30->40 | Reply | ✅ PASS | Admin can manage Guest VLAN. |

### Troubleshooting Applied

| Issue Encountered | Root Cause | Fix Applied |
|---|---|---|
| Initial `ping 192.168.30.100` timeout for same VLAN | Cabling mismatch: Assigned VLAN to Fa0/3, Fa0/5 but physical cables were on Fa0/6, Fa0/8 (VLAN 1) | Enabled port labels (`Options > Preferences > Always Show Port Labels`), rewired to Fa0/1-5 exactly matching `show vlan brief` |
| Router interface flapping down/up | Switchport Gig0/1 was still access, not trunk | Configured `interface Gig0/1` -> `switchport mode trunk` on Switch |
| `show interfaces trunk` invalid on Router | Command is Switch-only | Ran `show ip interface brief` on Router for subinterfaces, `show interfaces trunk` on Switch |
| Simulation shows only ICMP, STP, DTP, no ARP | ARP cache already populated from previous pings | Cleared ARP: `arp -d` on PCs + `clear arp-cache` on Router, filtered only ARP+ICMP in Simulation |

### Simulation Mode Observations

1. **Without ARP cache (after clear):**
   - Step 1: Admin PC sends ARP Broadcast - Who has 192.168.10.10?
   - Step 2: Switch floods to Router trunk
   - Step 3: Router replies with ARP Reply (Gateway MAC)
   - Step 4: ICMP Echo Request tagged VLAN 30 -> Router -> VLAN 10
   - Step 5: ICMP Echo Reply returns

2. **With ARP cache (normal):**
   - Only ICMP Request / Reply observed, because MACs already known. This is why STP/DTP appear - they are background control protocols.
