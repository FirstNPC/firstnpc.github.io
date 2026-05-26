---
layout: page
title: "18.0 Network Monitoring: Examine the ARP Table"
permalink: /cptlabs/18-network-monitoring/
---

# Lab 42: Examine the ARP Table

**Module:** 18.0 Network Monitoring  
**Date completed:** 2026-05-25  
**Score:** Completed (Activity Results: success)  
**Time elapsed:** 00:30:12

## Objective

Use Cisco Packet Tracer simulation mode to observe how the Address Resolution Protocol (ARP) maps Layer 3 IP addresses to Layer 2 MAC addresses, and to examine how switches build their MAC address tables from that traffic. The lab is split into three parts: an ARP request on the local LAN, populating a switch MAC address table, and observing the ARP process when the destination is on a remote network.

## Skills shown

- Generating and capturing ARP requests and replies in simulation mode
- Reading PDU details at Layer 2 (source/destination MAC behavior)
- Interpreting `arp -a` and `arp -d` on a Windows-style end device
- Reading a switch MAC address table with `show mac-address-table`
- Reading a router ARP table with `show arp`
- Distinguishing local ARP resolution from default-gateway ARP resolution for remote destinations

## Topology

**Before:**

![Topology before the lab](/assets/images/cptlabs/mod18/lab42_topology-before.png)

**After:**

![Topology after the lab](/assets/images/cptlabs/mod18/lab42_topology-after.png)

Two LANs connected by a serial WAN link between Router0 and Router1:

- **10.10.10.0/24** — Router0 → Switch0 → Access Point → two wireless laptops (10.10.10.2, 10.10.10.3)
- **172.16.31.0/24** — Router1 → Switch1 → three PCs (172.16.31.2, 172.16.31.3, 172.16.31.4)

## Addressing table

| Device | Interface | MAC Address | Switch Interface |
|---|---|---|---|
| Router0 | G0/0 | 0001.6458.2501 | G0/1 |
| Router0 | S0/0/0 | N/A | N/A |
| Router1 | G0/0 | 00E0.F7B1.8901 | G0/1 |
| Router1 | S0/0/0 | N/A | N/A |
| 10.10.10.2 | Wireless | 0060.2F84.4AB6 | F0/2 |
| 10.10.10.3 | Wireless | 0060.4706.572B | F0/2 |
| 172.16.31.2 | F0 | 000C.85CC.1DA7 | F0/1 |
| 172.16.31.3 | F0 | 0060.7036.2849 | F0/2 |
| 172.16.31.4 | G0 | 0002.1640.8D75 | F0/3 |

## Configuration steps

The devices were already configured. The work in this lab was observation, capture, and verification, not building the network.

### Part 1: Examine an ARP request

1. Open the command prompt on **172.16.31.2** and run `arp -d` to clear the ARP cache.
2. Enter simulation mode and run `ping 172.16.31.3`. Two PDUs are generated: an ARP broadcast (because the destination MAC is unknown) and the ICMP echo that is waiting on the ARP result.
3. Click **Capture/Forward** once. The ARP PDU moves to Switch1; the ICMP PDU stays pending. Switch1 floods the ARP request out every active port except the one it arrived on.

![Switch1 flooding copies of the ARP broadcast](/assets/images/cptlabs/mod18/lab42_step1_f-copies-of-pdu.png)

4. Continue Capture/Forward until the ARP reply returns to 172.16.31.2, then switch back to Realtime so the ping can complete.
5. Run `arp -a` on 172.16.31.2 to confirm the new entry.

### Part 2: Examine a switch MAC address table

1. From 172.16.31.2, run `ping 172.16.31.4` to generate additional Layer 2 traffic on Switch1.
2. From 10.10.10.2, run `ping 10.10.10.3` to generate traffic on Switch0 (through the access point).
3. On Switch1, run `show mac-address-table`.
4. On Switch0, run `show mac-address-table`.

### Part 3: Examine the ARP process in remote communications

1. From 172.16.31.2, run `ping 10.10.10.1`. The destination is on a different network, so the host must ARP for its default gateway (Router1's G0/0) instead of for 10.10.10.1 directly.
2. Run `arp -a` to view the new entry, then `arp -d` to clear it.
3. Switch to simulation mode and repeat `ping 10.10.10.1`. Capture/Forward to the PDU at Switch1 and inspect the ARP request's target IP.
4. Switch back to Realtime, open Router1's CLI, enter privileged EXEC mode, and run `show mac-address-table` and `show arp`.

## Device running configs

### Switch0

```
Switch0>show mac-address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

   1    0001.6458.2501    DYNAMIC     Gig0/1
   1    0060.2f84.4ab6    DYNAMIC     Fa0/2
   1    0060.4706.572b    DYNAMIC     Fa0/2
Switch0>
```

Two MAC addresses are bound to Fa0/2 because the access point is the single device plugged into that port and it bridges multiple wireless clients. From Switch0's point of view, all wireless laptop traffic enters on Fa0/2.

### Switch1

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
Switch>
```

Entries match the addressing table: each PC on its own Fa port and Router1's G0/0 on Gig0/1.

### Router1

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
Router#
```

Router1's MAC address table is empty because a router is a Layer 3 device, not a Layer 2 switch. Its ARP table, however, has two entries: its own G0/0 interface and 172.16.31.2, which ARPed for the default gateway during Part 3.

### Host 172.16.31.2 (command prompt)

```
C:\>arp -d
C:\>ping 172.16.31.3

Pinging 172.16.31.3 with 32 bytes of data:

Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time<1ms TTL=128
Reply from 172.16.31.3: bytes=32 time=1ms TTL=128

Ping statistics for 172.16.31.3:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms

C:\>arp -a
  Internet Address      Physical Address      Type
  172.16.31.3           0060.7036.2849        dynamic

C:\>ping 172.16.31.4

Pinging 172.16.31.4 with 32 bytes of data:

Reply from 172.16.31.4: bytes=32 time<1ms TTL=128
Reply from 172.16.31.4: bytes=32 time<1ms TTL=128
Reply from 172.16.31.4: bytes=32 time=1ms TTL=128
Reply from 172.16.31.4: bytes=32 time<1ms TTL=128

Ping statistics for 172.16.31.4:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),

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

C:\>arp -d
C:\>ping 10.10.10.1

Pinging 10.10.10.1 with 32 bytes of data:

Reply from 10.10.10.1: bytes=32 time=17ms TTL=254
Reply from 10.10.10.1: bytes=32 time=16ms TTL=254
Reply from 10.10.10.1: bytes=32 time=11ms TTL=254
Reply from 10.10.10.1: bytes=32 time=8ms TTL=254
```

### Host 10.10.10.2 (command prompt)

```
C:\>ping 10.10.10.3

Pinging 10.10.10.3 with 32 bytes of data:

Reply from 10.10.10.3: bytes=32 time=59ms TTL=128
Reply from 10.10.10.3: bytes=32 time=44ms TTL=128
Reply from 10.10.10.3: bytes=32 time=29ms TTL=128
Reply from 10.10.10.3: bytes=32 time=28ms TTL=128

Ping statistics for 10.10.10.3:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 28ms, Maximum = 59ms, Average = 40ms
```

## Lab questions and answers

**Part 1 — ARP request to 172.16.31.3**

- *Is the destination MAC address listed in the addressing table?* Yes — `0060.7036.2849`, which matches 172.16.31.3.
- *How many copies of the PDU did Switch1 make?* Three. The ARP broadcast was flooded out every active port on Switch1 except the port it arrived on.
- *Which device accepted the PDU?* 172.16.31.3. The other recipients dropped the frame at Layer 3 because the target IP in the ARP request did not match theirs.
- *What happened to the source and destination MAC addresses in the reply?* They swapped. The reply is a unicast from 172.16.31.3 back to 172.16.31.2 with the original requester's MAC as the new destination.
- *How many copies of the PDU did the switch make during the ARP reply?* One. The reply is unicast, and Switch1 had already learned the requester's MAC on Fa0/1, so it forwarded out that single port.
- *Do the MAC addresses of the source and destination align with their IP addresses?* Yes — they match the addressing table.
- *After the ping completes, to what IP does the new `arp -a` entry correspond?* 172.16.31.3, the host that was just pinged.
- *When does an end device issue an ARP request?* Whenever it needs to send a frame to an IP on its local subnet and does not already have a valid ARP entry for that IP, or to its default gateway when the destination is on a remote subnet.

**Part 2 — populating the switch MAC address table**

- *How many replies were sent and received for `ping 10.10.10.3`?* Four sent, four received, 0% loss.
- *Do the Switch1 entries correspond to the addressing table?* Yes. Fa0/1 = 172.16.31.2, Fa0/2 = 172.16.31.3, Fa0/3 = 172.16.31.4, Gig0/1 = Router1 G0/0.
- *Do the Switch0 entries correspond to the addressing table?* Yes. Gig0/1 = Router0 G0/0, Fa0/2 = both wireless laptops.
- *Why are two MAC addresses associated with one port on Switch0?* The access point is a Layer 2 bridge between wireless and wired. Both wireless laptops reach Switch0 through the single AP, which is plugged into Fa0/2, so Switch0 learns both laptop MACs on that one port.

**Part 3 — ARP for a remote destination**

- *After `ping 10.10.10.1` succeeds, what is the new ARP entry on 172.16.31.2?* 172.16.31.1 (Router1's G0/0), not 10.10.10.1.
- *After clearing the cache and re-pinging in simulation mode, how many PDUs appear?* Two — an ARP broadcast and a pending ICMP echo, the same pattern as in Part 1.
- *What is the target IP in the ARP request seen at Switch1?* 172.16.31.1, the default gateway.
- *Why is the target IP not 10.10.10.1?* Because 10.10.10.1 is on a different subnet. A host only ARPs for addresses on its own local network. For anything off-subnet, it ARPs for its default gateway and lets the router handle the rest.
- *How many MAC addresses are in Router1's MAC address table? Why?* Zero. A router is a Layer 3 device and does not maintain a switching MAC address table the way a switch does — its forwarding decisions are made from the routing table and ARP table.
- *Is there an entry for 172.16.31.2 in Router1's ARP table?* Yes — `172.16.31.2 → 000C.85CC.1DA7` on G0/0, learned when the host ARPed for the gateway.
- *What happens to the first ping in the situation where the router responds to the ARP request?* The first ICMP echo can be dropped or delayed while the ARP exchange completes. In this run all four replies returned, but the first reply showed a noticeably higher round-trip time (17–18 ms vs. 1 ms for the rest), consistent with the round trip waiting on ARP resolution before it could be transmitted.

## Troubleshooting notes

No troubleshooting was required. The activity is observation-only on a pre-configured topology. The one thing worth watching is timing: if you switch to Realtime before the ARP exchange finishes in simulation mode, the ping may already have started retrying in the background and the PDU sequence in the simulator can get out of sync with what the command prompt shows.

## Check Results

![Activity Results: completed](/assets/images/cptlabs/mod18/lab42_check-results-summary.png)

## Reflection

The clearest takeaway from this lab is that ARP is strictly a local-subnet protocol. A host never ARPs for a remote IP directly; it ARPs for its default gateway and hands the packet off. That single rule explains almost everything observed in Part 3, including why Router1's ARP table contains the host's MAC even though the host was trying to reach a completely different network.

The Switch0 result was the other moment that made the broader picture click. Seeing two MAC addresses on Fa0/2 looks wrong at first glance, but it is exactly what should happen: the switch only sees what is directly plugged into its ports, and the access point is acting as a transparent bridge for everything behind it. The same principle applies in production with any Layer 2 device sitting between a switch and its endpoints, including unmanaged switches, IP phones with PC pass-through ports, and virtualization hosts.

Finally, the first-ping latency in Part 3 (17–18 ms versus 1 ms for the follow-up packets) is a useful real-world tell. A consistently slow first ping followed by fast subsequent pings is almost always ARP resolution, not a network problem.
