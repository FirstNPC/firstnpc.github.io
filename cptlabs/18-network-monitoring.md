---
layout: page
title: "18.0: Network Monitoring"
permalink: /cptlabs/18-network-monitoring/
---

## Lab 42: Packet Tracer - Examine the ARP Table

**Module:** 18 | **Completed:** May 25, 2026 | **Score:** Completed (00:30:12 elapsed)

**Skills demonstrated:** ARP request and reply capture in simulation mode, PDU inspection at Layer 2, end-device ARP cache management with `arp -a` and `arp -d`, switch MAC address table reading with `show mac-address-table`, router ARP table reading with `show arp`, distinguishing local-subnet ARP resolution from default-gateway ARP resolution for remote destinations

---

### Objective

Use Cisco Packet Tracer simulation mode to observe how the Address Resolution Protocol maps Layer 3 IP addresses to Layer 2 MAC addresses, and how switches build their MAC address tables from that traffic. The lab is split into three parts: an ARP request on the local LAN, populating a switch MAC address table, and observing the ARP process when the destination is on a remote network.

---

### Network Topology

**Before** — pre-configured two-LAN topology with all devices placed and addressed:

![Topology before](/assets/images/cptlabs/mod18/lab42_topology-before.png)

**After** — same topology, with link lights green after the observation traffic ran:

![Topology after](/assets/images/cptlabs/mod18/lab42_topology-after.png)

*Two LANs connected by a serial WAN link between Router0 and Router1. The 10.10.10.0/24 LAN sits behind Router0 with Switch0, an access point, and two wireless laptops. The 172.16.31.0/24 LAN sits behind Router1 with Switch1 and three wired PCs. Devices were already configured; the work in this lab was observation, capture, and verification.*

---

### Addressing Table

| Device | Interface | MAC Address | Switch Interface |
|:--|:--|:--|:--|
| Router0 | G0/0 | 0001.6458.2501 | G0/1 |
| Router0 | S0/0/0 | N/A | N/A |
| Router1 | G0/0 | 00E0.F7B1.8901 | G0/1 |
| Router1 | S0/0/0 | N/A | N/A |
| 10.10.10.2 | Wireless | 0060.2F84.4AB6 | F0/2 |
| 10.10.10.3 | Wireless | 0060.4706.572B | F0/2 |
| 172.16.31.2 | F0 | 000C.85CC.1DA7 | F0/1 |
| 172.16.31.3 | F0 | 0060.7036.2849 | F0/2 |
| 172.16.31.4 | G0 | 0002.1640.8D75 | F0/3 |

---

### Configuration Steps

**Step 1 — Clear the ARP cache on 172.16.31.2 and trigger an ARP request**

Opened the command prompt on 172.16.31.2 and ran `arp -d` to clear the ARP cache. Entered simulation mode and ran `ping 172.16.31.3`. Two PDUs were generated: an ARP broadcast (because the destination MAC was unknown) and the ICMP echo waiting on the ARP result.

**Step 2 — Step through the ARP broadcast at Switch1**

Clicked Capture/Forward once. The ARP PDU moved to Switch1 and the ICMP PDU stayed pending. Switch1 flooded the ARP request out every active port except the one it arrived on, producing three copies of the broadcast frame.

![Switch1 flooding copies of the ARP broadcast](/assets/images/cptlabs/mod18/lab42_step1_f-copies-of-pdu.png)

Inspecting the PDU at Layer 2 showed the destination MAC as the broadcast address (`FFFF.FFFF.FFFF`). Only 172.16.31.3 accepted the frame — the other hosts dropped it at Layer 3 because the target IP in the ARP request did not match theirs.

**Step 3 — Step through the ARP reply back to 172.16.31.2**

Continued Capture/Forward until the ARP reply returned to 172.16.31.2. The reply is a unicast: source and destination MAC addresses swapped from the request, with the original requester's MAC now in the destination field. Switch1 made only one copy of the reply, because by that point it had already learned the requester's MAC on Fa0/1 and forwarded out that single port. Switched back to Realtime so the pending ICMP echo could complete.

**Step 4 — Verify the new ARP entry on the host**

Ran `arp -a` on 172.16.31.2. A single entry now mapped 172.16.31.3 to its physical address `0060.7036.2849`, matching the addressing table.

**Step 5 — Generate additional traffic to populate the switch tables**

From 172.16.31.2, ran `ping 172.16.31.4` to put Switch1's Fa0/3 endpoint into the table. From 10.10.10.2, ran `ping 10.10.10.3` — four sent, four received, 0% loss — to put both wireless laptop MACs into Switch0 via the access point.

**Step 6 — Read each switch's MAC address table**

On Switch1, ran `show mac-address-table`. The four expected entries appeared: each of the three wired PCs on its own Fa port, and Router1's G0/0 MAC on Gig0/1. On Switch0, ran the same command. Three entries appeared: Router0's G0/0 MAC on Gig0/1, and both wireless laptop MACs on Fa0/2. Two MAC addresses were bound to a single port because the access point is the only device physically connected to Fa0/2, and it bridges all wireless client traffic onto that port.

**Step 7 — Trigger an ARP exchange for a remote destination**

From 172.16.31.2, ran `ping 10.10.10.1`. The destination is on a different subnet, so the host had to ARP for its default gateway (Router1's G0/0 at 172.16.31.1) instead of for 10.10.10.1 directly. Confirmed with `arp -a` that the new entry was for 172.16.31.1, not 10.10.10.1. Cleared the cache with `arp -d`, switched to simulation mode, and repeated the ping. Two PDUs appeared, the same pattern as Step 1. Capture/Forward to the PDU at Switch1 confirmed the target IP in the ARP request was 172.16.31.1.

**Step 8 — Read the router's MAC and ARP tables**

Switched back to Realtime, opened Router1's CLI, and entered privileged EXEC. `show mac-address-table` returned an empty table — a router is a Layer 3 device and does not maintain a switching MAC address table the way a switch does. `show arp`, on the other hand, showed two entries: Router1's own G0/0 interface and 172.16.31.2, which had ARPed for the gateway during Step 7.

---

### Device Configuration

**Switch0 — MAC address table**

```
Switch0>show mac-address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

   1    0001.6458.2501    DYNAMIC     Gig0/1
   1    0060.2f84.4ab6    DYNAMIC     Fa0/2
   1    0060.4706.572b    DYNAMIC     Fa0/2
```

**Switch1 — MAC address table**

```
Switch>show mac-address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

   1    0002.1640.8d75    DYNAMIC     Fa0/3
   1    000c.85cc.1da7    DYNAMIC     Fa0/1
   1    0060.7036.2849    DYNAMIC     Fa0/2
   1    00e0.f7b1.8901    DYNAMIC     Gig0/1
```

**Router1 — MAC and ARP tables**

```
Router>en
Router#show mac-address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

Router#show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  172.16.31.1             -   00E0.F7B1.8901  ARPA   GigabitEthernet0/0
Internet  172.16.31.2             1   000C.85CC.1DA7  ARPA   GigabitEthernet0/0
```

**Host 172.16.31.2 — command prompt session**

```
C:\>arp -d
C:\>ping 172.16.31.3

Pinging 172.16.31.3 with 32 bytes of data:

Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time=1ms TTL=128

C:\>arp -a
  Internet Address      Physical Address      Type
  172.16.31.3           0060.7036.2849        dynamic

C:\>ping 10.10.10.1

Pinging 10.10.10.1 with 32 bytes of data:

Reply from 10.10.10.1: bytes=32 time=18ms TTL=254
Reply from 10.10.10.1: bytes=32 time=17ms TTL=254
Reply from 10.10.10.1: bytes=32 time=1ms TTL=254
Reply from 10.10.10.1: bytes=32 time=1ms TTL=254

C:\>arp -a
  Internet Address      Physical Address      Type
  172.16.31.1           00e0.f7b1.8901        dynamic
  172.16.31.3           0060.7036.2849        dynamic
  172.16.31.4           0002.1640.8d75        dynamic
```

**Host 10.10.10.2 — command prompt session**

```
C:\>ping 10.10.10.3

Pinging 10.10.10.3 with 32 bytes of data:

Reply from 10.10.10.3: bytes=32 time=59ms TTL=128
Reply from 10.10.10.3: bytes=32 time=44ms TTL=128
Reply from 10.10.10.3: bytes=32 time=29ms TTL=128
Reply from 10.10.10.3: bytes=32 time=28ms TTL=128
```

---

### Troubleshooting

No troubleshooting was required. The activity runs on a pre-configured topology and the work is observation rather than configuration. The one timing detail worth watching: if Realtime mode resumes before the ARP exchange finishes in simulation mode, the ping in the command prompt may already have started retrying in the background, and the PDU sequence in the simulator can fall out of sync with what the host shows. Switching back to Realtime only after the ARP reply has returned to the requester keeps the two views consistent.

---

### Check Results

![Check Results: Activity completed](/assets/images/cptlabs/mod18/lab42_check-results-summary.png)

Activity Results screen confirming the lab completed with 30 minutes 12 seconds elapsed.

---

### Reflection

**What I learned:** ARP is strictly a local-subnet protocol. A host never ARPs for a remote IP directly — it ARPs for its default gateway and hands the packet off, and the router resolves the next hop on the other side. That single rule explains almost everything in Part 3, including why Router1's ARP table ends up holding the requesting host's MAC even though the host was trying to reach a completely different network. The Switch0 result reinforced a separate principle: a switch only sees what is directly plugged into its ports, so any Layer 2 bridge sitting between the switch and its endpoints (an access point here, but the same is true of unmanaged switches, IP phones with PC pass-through ports, or virtualization hosts) will show up as multiple MAC addresses sharing one port.

**What was challenging:** Keeping the simulation-mode PDU sequence aligned with the realtime command-prompt output. The lab depends on stepping through PDUs one Capture/Forward click at a time, and it is easy to switch back to Realtime a click too early and miss the ARP reply step entirely. The questions about copy counts (three copies of the broadcast, one copy of the reply) only make sense if every step is observed in order.

**What I would do differently:** Treat the first-ping latency in Part 3 as the takeaway rather than a footnote. The host pinged 10.10.10.1 and the first two replies came back at 17–18 ms while the next two were 1 ms each. That latency drop is ARP resolution completing — the first ICMP echo had to wait for the ARP exchange before it could be transmitted. In production, a consistently slow first ping followed by fast subsequent pings is almost always ARP, not a network problem, and the lab is the cleanest demonstration of that pattern I will ever see in a controlled environment.

[Back to home](/)
