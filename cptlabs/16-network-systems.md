---
layout: page
title: "16.0: Network Systems"
permalink: /cptlabs/16-network-systems/
---

## Lab 29: Packet Tracer - VLSM Design and Implementation Practice

**Module:** 16 | **Completed:** May 23, 2026 | **Score:** 30/30 (13/13 items)

**Skills demonstrated:** VLSM subnet design, /24 address space planning, largest-first subnet allocation, /26 /27 /28 /29 /30 mask selection, router LAN and WAN interface configuration, switch SVI and default gateway configuration, EIGRP neighbor convergence, CDP-based interface identification, routing table inspection, troubleshooting subnet-mask mismatches, recovering from an interface overlap error, end-to-end ping verification

---

### Objective

Design a VLSM addressing scheme for the 10.11.48.0/24 network so that four LAN subnets and one WAN link can coexist without overlap or wasted address space. Sizes vary by room: Room-407 needs 60 hosts, Room-279 needs 30, Room-114 needs 14, Room-312 needs 6, and the Branch1 to Branch2 serial link needs 2. After designing the scheme, apply the addressing to Branch1's two LAN interfaces, the Room-312 management switch, and PC-D, then verify reachability across the WAN.

---

### Network Topology

**Before** — baseline topology as delivered, with some link arrows red because LAN interfaces still need addressing or correction: ![Topology before configuration](/assets/images/cptlabs/mod16/lab29_topology-before.png)

**After** — the same topology after Branch1, the Room-312 switch, and PC-D were addressed and EIGRP reconverged: ![Topology after configuration](/assets/images/cptlabs/mod16/lab29_topology-after.png)

*Branch1 routes between Room-114 (14 hosts) on G0/0 and Room-279 (30 hosts) on G0/1. Branch2 routes between Room-312 (6 hosts) and Room-407 (60 hosts). The two routers connect over a serial WAN that only needs two usable addresses. Branch2 is locked in this activity variant, so the deployed WAN subnet (10.11.48.120/30) had to be matched on Branch1 rather than reassigned to 10.11.48.128/30 as the largest-first design would otherwise place it.*

---

### VLSM Addressing Plan

The plan allocates subnets largest-first inside 10.11.48.0/24. Room-407 takes the /26 block at the start, Room-279 the /27 block after it, then Room-114 at /28 and Room-312 at /29, leaving a /30 for the WAN. Total usable host space consumed is 132 addresses out of 256.

| Subnet | Hosts Needed | Network/CIDR | Mask | First Usable | Broadcast |
|:--|:--|:--|:--|:--|:--|
| Room-407 LAN | 60 | 10.11.48.0/26 | 255.255.255.192 | 10.11.48.1 | 10.11.48.63 |
| Room-279 LAN | 30 | 10.11.48.64/27 | 255.255.255.224 | 10.11.48.65 | 10.11.48.95 |
| Room-114 LAN | 14 | 10.11.48.96/28 | 255.255.255.240 | 10.11.48.97 | 10.11.48.111 |
| Room-312 LAN | 6 | 10.11.48.112/29 | 255.255.255.248 | 10.11.48.113 | 10.11.48.119 |
| Branch1–Branch2 WAN | 2 | 10.11.48.120/30 | 255.255.255.252 | 10.11.48.121 | 10.11.48.123 |

---

### Addressing Table

The router LAN interfaces take the first usable address in each subnet, the Branch1 side of the WAN takes the first usable address and Branch2 the last, the switch SVI takes the second usable address, and the host PCs take the last usable address.

| Device | Interface | Address | Mask | Gateway |
|:--|:--|:--|:--|:--|
| Branch1 | G0/0 (Room-114) | 10.11.48.97 | 255.255.255.240 | N/A |
| Branch1 | G0/1 (Room-279) | 10.11.48.65 | 255.255.255.224 | N/A |
| Branch1 | S0/0/0 | 10.11.48.121 | 255.255.255.252 | N/A |
| Branch2 | G0/0 (Room-407) | 10.11.48.1 | 255.255.255.192 | N/A |
| Branch2 | G0/1 (Room-312) | 10.11.48.113 | 255.255.255.248 | N/A |
| Branch2 | S0/0/0 | 10.11.48.122 | 255.255.255.252 | N/A |
| Room-312 SW | VLAN 1 | 10.11.48.114 | 255.255.255.248 | 10.11.48.113 |
| PC-D (Room-407) | NIC | 10.11.48.62 | 255.255.255.192 | 10.11.48.1 |

---

### Configuration Steps

**Step 1 — Identify which room sits on which Branch1 interface**
Branch2 is pre-configured and locked in this variant, so the LAN-to-router pairings on Branch2 are accepted as given. For Branch1, `show cdp neighbors` confirmed that G0/0 connects to the Room-114 switch and G0/1 to the Room-279 switch. This is the prerequisite for choosing the correct subnet for each interface — getting the pairing wrong would produce a working interface in the wrong network.

**Step 2 — Configure Branch1 LAN interfaces**
Assigned G0/0 to 10.11.48.97/28 (Room-114, first usable in the /28) and G0/1 to 10.11.48.65/27 (Room-279, first usable in the /27). The S0/0/0 WAN interface was already configured at 10.11.48.121/30, matching the locked Branch2 side at 10.11.48.122/30, so it was left in place once the WAN subnet was confirmed from Branch1's routing table and the live EIGRP adjacency.

**Step 3 — Configure the Room-312 management switch**
Brought the VLAN 1 SVI up with 10.11.48.114/29 — the second usable address in the Room-312 /29 — and set `ip default-gateway 10.11.48.113` so the switch can reply to management traffic that arrives from off-subnet sources such as Branch1 or other LANs.

**Step 4 — Configure PC-D**
PC-D physically lives in Room-407 behind Branch2, so its addressing belongs to the /26 block, not the /29 Room-312 block. Set the NIC to 10.11.48.62/26 with a default gateway of 10.11.48.1 — last usable address in the /26 for the host, first usable for the gateway.

**Step 5 — Verify connectivity**
From Branch1: pings to 10.11.48.122 (Branch2 WAN) returned 5/5, and `show ip route` showed EIGRP routes for the remote LANs via 10.11.48.122 on Serial0/0/0. From Room-312: pings to its own gateway 10.11.48.113, to 10.11.48.121 across the WAN, and to the other LAN gateways all succeeded. The reachability across the WAN confirmed that EIGRP had reconverged after the Branch1 LAN changes.

---

### Device Configuration

**Branch1 — LAN configuration session**

Branch1's G0/1 already carried an address before the planned reconfiguration, which caused an overlap error on the first attempt to add the same range to G0/0. See Troubleshooting below — the working sequence was to clear G0/1 first, configure G0/0, then re-address G0/1.

```
Branch1>enable
Branch1#configure terminal
Branch1(config)#interface GigabitEthernet0/1
Branch1(config-if)#no ip address
Branch1(config-if)#exit
Branch1(config)#interface GigabitEthernet0/0
Branch1(config-if)#ip address 10.11.48.97 255.255.255.240
Branch1(config-if)#no shutdown
Branch1(config-if)#exit
Branch1(config)#interface GigabitEthernet0/1
Branch1(config-if)#ip address 10.11.48.65 255.255.255.224
Branch1(config-if)#no shutdown
Branch1(config-if)#exit
Branch1(config)#end
Branch1#copy running-config startup-config
```

**Branch1 — verification**

```
Branch1#show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     10.11.48.97     YES manual up                    up
GigabitEthernet0/1     10.11.48.65     YES manual up                    up
Serial0/0/0            10.11.48.121    YES manual up                    up
Serial0/0/1            unassigned      YES unset  administratively down down
Vlan1                  unassigned      YES unset  administratively down down

Branch1#show cdp neighbors
Device ID    Local Intrfce   Holdtme    Capability  Platform   Port ID
Branch2      Ser 0/0/0       169        R           C1900      Ser 0/0/0
Room-114     Gig 0/0         153        S           2960       Gig 0/1
Room-279     Gig 0/1         153        S           2960       Gig 0/1
```

**Room-312 switch — configuration session**

```
Room-312>enable
Room-312#configure terminal
Room-312(config)#interface vlan 1
Room-312(config-if)#ip address 10.11.48.114 255.255.255.248
Room-312(config-if)#no shutdown
Room-312(config-if)#exit
Room-312(config)#ip default-gateway 10.11.48.113
Room-312(config)#end
Room-312#copy running-config startup-config
```

**PC-D — IP Configuration**

PC-D is set statically through the Desktop > IP Configuration utility rather than the CLI.

| Field | Value |
|:--|:--|
| IPv4 Address | 10.11.48.62 |
| Subnet Mask | 255.255.255.192 |
| Default Gateway | 10.11.48.1 |

![PC-D IP Configuration after correction](/assets/images/cptlabs/mod16/lab29_pc-d-config.png)

---

### Troubleshooting

**Problem 1 — Misidentified which router served which rooms.** The first attempt at Branch1 used 10.11.48.1/26 on G0/0 (Room-407 subnet) and 10.11.48.97/28 on G0/1 (Room-114 subnet), based on an assumption that Branch1 hosted Room-407 and Room-114. The Check Results panel flagged both interfaces incorrect even though every other detail looked right.

**What I tried:** Cross-referenced the topology diagram with `show cdp neighbors` on Branch1. CDP returned Room-114 on Gig 0/0 and Room-279 on Gig 0/1 — Branch1 served Room-114 and Room-279, not Room-407. Room-407 was actually behind Branch2.

**Resolution:** Reassigned Branch1 G0/0 to 10.11.48.97/28 (Room-114) and G0/1 to 10.11.48.65/27 (Room-279), and moved PC-D into the Room-407 /26 subnet at 10.11.48.62 with gateway 10.11.48.1. The takeaway is that the room-to-router pairing has to be confirmed from the topology or CDP before any addressing is applied — visual proximity in the diagram is not a reliable signal.

**Problem 2 — Overlap error when re-addressing Branch1 G0/0.** Applying `ip address 10.11.48.97 255.255.255.240` to G0/0 returned `% 10.11.48.96 overlaps with GigabitEthernet0/1`, because G0/1 was still holding the old /28 address.

**What I tried:** Confirmed from `show running-config` that G0/1 still had 10.11.48.97/28 left over from the earlier configuration. IOS refuses to let two interfaces sit in overlapping subnets.

**Resolution:** Removed the address from G0/1 first with `no ip address`, then configured G0/0 with the /28 address, then re-addressed G0/1 with the /27 Room-279 address. The takeaway is that when swapping subnets between interfaces, the source interface has to be cleared before the destination can claim its range.

**Problem 3 — Subnet-mask mismatch with the locked Branch2 router.** The lab instructions specified a /28 mask for Room-312, but the locked Branch2 router's G0/1 was actually configured with a /29 mask. PC-D and the Room-312 switch initially used /28, which made the gateway look reachable from PC-D's perspective but invisible from Branch2's — Branch2 considered .126 to be outside its LAN, so no ARP reply ever returned and every ping timed out.

**What I tried:** Inspected the EIGRP-learned routes in Branch1's table. The advertised prefix from Branch2 was `10.11.48.112/29`, not `/28`. That confirmed the deployed mask was /29, regardless of what the instructions said.

**Resolution:** Reconfigured the Room-312 switch VLAN 1 interface and PC-D's NIC to use 255.255.255.248 (/29). After the change the gateway pings succeeded and EIGRP-learned reachability was complete. The deeper takeaway is that the *deployed* network governs connectivity, even if a written instruction disagrees — and reading the routing table is faster than re-reading the spec.

---

### Check Results

![Activity results summary](/assets/images/cptlabs/mod16/lab29_check-results-summary.png)
![Assessment items detail](/assets/images/cptlabs/mod16/lab29_check-results-items.png)

Final score: **30/30 (13/13 items)**, with every assessment item passing. The score breaks down across three graded components — Default Gateway Configuration 5/5, Device Interface Configuration 3/3, and VLSM Addressing Implementation 22/22.

---

### Reflection

**What I learned:** A VLSM design is only half the work — the design has to be matched to what is actually deployed before it produces a working network. The graded scheme here happens to follow the same largest-first allocation I would have chosen, but Branch2 was locked with a /29 on Room-312 where the instructions called for /28, and the WAN sat on /30 from .120 rather than .128. The shortest path to a correct configuration was to read the routing table and CDP output from Branch1, treat that as ground truth, and build outward from there. CDP in particular makes the room-to-interface mapping unambiguous in a few seconds.

**What was challenging:** Untangling three overlapping mistakes at once: a wrong room-to-router mental model, an interface-overlap error blocking the re-address, and a /28-versus-/29 mask mismatch that produced silent ping failures with no obvious error message. None of those failures had a noisy console message pointing at the cause — the first showed up only in the Check Results panel, the second as a one-line IOS warning, and the third as plain timeouts.

**What I would do differently:** Read the routing table first, before touching any interface. `show ip route` and `show cdp neighbors` together reveal which subnets are deployed, on which masks, and which interfaces they sit on — answering every question the design phase needs before the configuration phase starts. I would also document the deployed mask of each pre-configured interface alongside the design table, so any discrepancy between "what the instructions said" and "what the lab actually built" is visible at a glance instead of discovered through failed pings.

[Back to home](/)
