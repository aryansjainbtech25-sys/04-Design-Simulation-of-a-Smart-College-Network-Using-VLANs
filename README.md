# 04 Design & Simulation of a Smart College Network Using VLANs

## Address Configurations

### VLAN Addressing Table

| Department | VLAN ID | Network address | Default gateway |
|---|---:|---|---|
| Students | 10 | 192.168.10.0/24 | 192.168.10.1 |
| Faculty | 20 | 192.168.20.0/24 | 192.168.20.1 |
| Administration | 30 | 192.168.30.0/24 | 192.168.30.1 |
| Guests | 40 | 192.168.40.0/24 | 192.168.40.1 |

### IP Addressing Table

| Device | IP address | Subnet mask | Gateway |
|---|---|---|---|
| Student PC | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| Faculty PC | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| Admin PC | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| Guest PC | 192.168.40.10 | 255.255.255.0 | 192.168.40.1 |

## Tests:
- **Day 2:** Intra-VLAN ping (Admin 192.168.30.10 -> Server 192.168.30.100): SUCCESS
- **Day 2:** Inter-VLAN ping (Admin 192.168.30.10 -> Student 192.168.10.10): FAILED (Expected, router not configured yet)

## Fix Applied
- **Day 2:** Rewired Fa0/1-5 to match VLAN config + set Gig0/1 as trunk.
