---
layout: page
title: "13.0: Network Addressing Schemes"
permalink: /cptlabs/13-network-addressing-schemes/
---

## Lab 15: Packet Tracer - Subnet an IPv4 Network

**Module:** 13 | **Completed:** May 14, 2026 | **Score:** 23/23 (23/23 items)

**Skills demonstrated:** Fixed-length subnet design from a /24, binary-to-dotted-decimal mask conversion, Cisco IOS router CLI (interface IP, hostname, enable secret, console line login), switch SVI and `ip default-gateway` configuration, end-host static IP assignment, `%IP-4-DUPADDR` interpretation, ping-based connectivity verification

---

### Objective

Subnet `192.168.0.0/24` into multiple equal-size subnets that satisfy the host counts for two LANs while reserving room for future growth, then build out the network by configuring the customer router, both LAN switches, and both end hosts so that every device communicates end to end.

---

### Network Topology

**Before** — customer-side interfaces administratively down, no addressing yet on CustomerRouter, LAN-A Switch, LAN-B Switch, PC-A, or PC-B (red indicators on every customer link):

![Topology before customer-side configuration](/assets/images/cptlabs/mod13/lab15_topology-before.png)

**After** — every customer interface up and addressed, all link indicators green:

![Topology after customer-side configuration](/assets/images/cptlabs/mod13/lab15_topology-after.png)

*CustomerRouter joins LAN-A (50 hosts) and LAN-B (40 hosts) to ISPRouter over the 209.165.201.0/30 WAN link. ISP side (209.165.200.224/27) was pre-configured. Customer LANs were carved out of 192.168.0.0/24 using a /26 mask.*

---

### Subnet Design

**Requirements:** 50+ hosts in LAN-A, 40+ hosts in LAN-B, at least four subnets total (two in service plus two reserved for growth), single fixed-length mask across the whole block (no VLSM).

**Mask comparison:**

| Prefix | Dotted decimal | Subnets | Usable hosts | Verdict |
|:--|:--|:-:|:-:|:--|
| /25 | 255.255.255.128 | 2 | 126 | Hosts OK, too few subnets |
| **/26** | **255.255.255.192** | **4** | **62** | **Meets both** |
| /27 | 255.255.255.224 | 8 | 30 | Too few hosts |
| /28 | 255.255.255.240 | 16 | 14 | Too few hosts |
| /29 | 255.255.255.248 | 32 | 6 | Too few hosts |
| /30 | 255.255.255.252 | 64 | 2 | Too few hosts |

Borrowing 2 bits from the host portion of `/24` produces `/26`, the only mask that satisfies both the 50-host minimum and the 4-subnet minimum simultaneously.

**Derived subnets from 192.168.0.0/26:**

| # | Subnet | Usable range | Broadcast | Assignment |
|:-:|:--|:--|:--|:--|
| 1 | 192.168.0.0/26 | .1 to .62 | .63 | LAN-A |
| 2 | 192.168.0.64/26 | .65 to .126 | .127 | LAN-B |
| 3 | 192.168.0.128/26 | .129 to .190 | .191 | Reserved for future expansion |
| 4 | 192.168.0.192/26 | .193 to .254 | .255 | Reserved for future expansion |

**Resulting addressing table:**

| Device | Interface | IP Address | Subnet Mask | Default Gateway |
|:--|:--|:--|:--|:--|
| CustomerRouter | G0/0 | 192.168.0.1 | 255.255.255.192 | N/A |
| CustomerRouter | G0/1 | 192.168.0.65 | 255.255.255.192 | N/A |
| CustomerRouter | S0/1/0 | 209.165.201.2 | 255.255.255.252 | N/A |
| LAN-A Switch | VLAN1 | 192.168.0.2 | 255.255.255.192 | 192.168.0.1 |
| LAN-B Switch | VLAN1 | 192.168.0.66 | 255.255.255.192 | 192.168.0.65 |
| PC-A | NIC | 192.168.0.62 | 255.255.255.192 | 192.168.0.1 |
| PC-B | NIC | 192.168.0.126 | 255.255.255.192 | 192.168.0.65 |

First host in each subnet went to the router interface, second host went to the switch's VLAN1 SVI, and the last usable host went to the PC, per the lab's assignment rules.

---

### Configuration Steps

**Step 1 — Configure CustomerRouter**

Entered privileged EXEC then global config. Set the G0/0 interface to `192.168.0.1 255.255.255.192` and brought it up with `no shutdown`. Set the G0/1 interface to `192.168.0.65 255.255.255.192` and brought it up. Set the hostname to `CustomerRouter`, the enable secret to `Class123`, and configured the console line with a password and `login`. Saved the running config to startup with `copy running-config startup-config`.

**Step 2 — Configure LAN-A Switch**

Entered the VLAN1 SVI with `interface vlan 1`, set `ip address 192.168.0.2 255.255.255.192`, and brought it up with `no shutdown`. Exited the SVI and set `ip default-gateway 192.168.0.1` so the switch's own management traffic could reach beyond the LAN.

**Step 3 — Configure LAN-B Switch**

Same pattern as LAN-A but with the LAN-B addressing. SVI got `192.168.0.66 255.255.255.192`, and the default gateway pointed at the router's G0/1 interface at `192.168.0.65`.

**Step 4 — Configure PC-A and PC-B**

Used the Desktop → IP Configuration tool on each PC and selected Static. PC-A took `192.168.0.62 / 255.255.255.192` with gateway `192.168.0.1` (last usable host in LAN-A pointing at the first usable host as gateway). PC-B took `192.168.0.126 / 255.255.255.192` with gateway `192.168.0.65`.

**Step 5 — Verify with ping**

From PC-A's command prompt, pinged its own gateway, the LAN-B gateway across the router, and finally PC-B. All three targets replied. From PC-B, pinged its own gateway and confirmed replies.

---

### Device Configuration

**CustomerRouter — configuration session**

```
Router>en
Router#conf t
Router(config)#int g0/0
Router(config-if)#ip address 192.168.0.1 255.255.255.192
Router(config-if)#no shutdown
%LINK-5-CHANGED: Interface GigabitEthernet0/0, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0, changed state to up
Router(config-if)#exit
Router(config)#int g0/1
Router(config-if)#ip address 192.168.0.65 255.255.255.192
Router(config-if)#no shutdown
%LINK-5-CHANGED: Interface GigabitEthernet0/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
%IP-4-DUPADDR: Duplicate address 192.168.0.65 on GigabitEthernet0/1, sourced by 0050.0F33.E320
Router(config-if)#exit
Router(config)#hostname CustomerRouter
CustomerRouter(config)#enable secret Class123
CustomerRouter(config)#line con 0
CustomerRouter(config-line)#password Class123
CustomerRouter(config-line)#login
CustomerRouter(config-line)#end
CustomerRouter#copy running-config startup-config
Destination filename [startup-config]?
Building configuration...
[OK]
```

**LAN-A Switch — configuration session**

```
Switch>en
Switch#conf t
Switch(config)#interface vlan 1
Switch(config-if)#ip address 192.168.0.2 255.255.255.192
Switch(config-if)#no shutdown
%LINK-3-UPDOWN: Interface Vlan1, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan1, changed state to up
Switch(config-if)#exit
Switch(config)#ip default-gateway 192.168.0.1
```

**LAN-B Switch — configuration session**

```
Switch>en
Switch#conf t
Switch(config)#interface vlan 1
Switch(config-if)#ip address 192.168.0.66 255.255.255.192
Switch(config-if)#no shutdown
%LINK-3-UPDOWN: Interface Vlan1, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan1, changed state to up
Switch(config-if)#exit
Switch(config)#ip default-gateway 192.168.0.65
```

---

### Verification

**PC-A — ping to local gateway, remote gateway, and PC-B**

```
C:\>ping 192.168.0.1

Pinging 192.168.0.1 with 32 bytes of data:
Reply from 192.168.0.1: bytes=32 time<1ms TTL=255
Reply from 192.168.0.1: bytes=32 time<1ms TTL=255
Reply from 192.168.0.1: bytes=32 time<1ms TTL=255
Reply from 192.168.0.1: bytes=32 time<1ms TTL=255

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\>ping 192.168.0.65

Pinging 192.168.0.65 with 32 bytes of data:
Reply from 192.168.0.65: bytes=32 time=10ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\>ping 192.168.0.126

Pinging 192.168.0.126 with 32 bytes of data:
Request timed out.
Reply from 192.168.0.126: bytes=32 time<1ms TTL=127
Reply from 192.168.0.126: bytes=32 time<1ms TTL=127
Reply from 192.168.0.126: bytes=32 time<1ms TTL=127

Packets: Sent = 4, Received = 3, Lost = 1 (25% loss)
```

The single timeout on the first PC-A to PC-B ping is expected first-packet ARP resolution latency, not a connectivity fault. The remaining three replies, plus the TTL drop from 255 to 127 in the response, confirm the packets are crossing the router as designed.

**PC-B — ping to local gateway**

```
C:\>ping 192.168.0.65

Pinging 192.168.0.65 with 32 bytes of data:
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255
Reply from 192.168.0.65: bytes=32 time<1ms TTL=255

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

---

### Troubleshooting

**Problem:** While bringing up CustomerRouter's G0/1 interface, the console printed `%IP-4-DUPADDR: Duplicate address 192.168.0.65 on GigabitEthernet0/1, sourced by 0050.0F33.E320`. Some other device on the LAN-B segment was already claiming the router's planned gateway address.

**What I tried:** Cross-referenced the offending MAC `0050.0F33.E320` against each end host. PC-B's IP Configuration panel showed a link-local IPv6 address of `FE80::250:FFF:FE33:E320`, which derives from that same MAC. So PC-B was the source of the conflict, meaning PC-B had been given the LAN-B gateway IP (`192.168.0.65`) by mistake instead of its assigned host IP (`192.168.0.126`).

**Resolution:** Opened PC-B's Desktop → IP Configuration, set the IPv4 address back to `192.168.0.126`, kept the subnet mask at `255.255.255.192`, and confirmed the default gateway was `192.168.0.65`. The duplicate-address warning did not reappear, and all subsequent pings succeeded.

---

### Understanding the `%IP-4-DUPADDR` Message

Address conflict warnings like this one are easy to gloss over inside the boot output, but the message itself contains everything needed to find the offending device. Decoding it is worth doing once.

**The analogy.** Every device on a network needs a unique IP, the same way every house on a street needs a unique number. If two houses both claim "123 Main Street," the mail carrier has no way to know which one to deliver to. Two devices claiming `192.168.0.65` produces the same kind of routing ambiguity: packets bound for that IP might land at the router or at the rogue PC depending on which entry the upstream switch's MAC table has cached at the moment. The symptom looks like flaky, intermittent connectivity rather than a clean failure.

**The message format.** Cisco IOS log messages always follow the same shape. Reading them piece by piece turns a wall of text into actionable information:

| Piece | Meaning |
|:--|:--|
| `%` | System message marker |
| `IP` | The subsystem reporting (the IP stack) |
| `4` | Severity level. 4 is Warning |
| `DUPADDR` | Mnemonic for "duplicate address" |
| `192.168.0.65` | The contested IP |
| `GigabitEthernet0/1` | The local interface that detected the conflict |
| `0050.0F33.E320` | The MAC address of the other device claiming that IP |

The last field is the critical one for troubleshooting. The router is essentially saying, "this is not me, it is that device over there, and here is its fingerprint."

**How the router detected it.** As soon as a Cisco interface comes up, it sends a gratuitous ARP on the segment: a broadcast that asks, in effect, "is anyone already using this address?" If a device replies, the router logs the conflict and includes the responder's MAC. In this lab, PC-B raised its hand because it had been given `.65` instead of its assigned `.126`.

**Why this matters in production.** Address conflicts are one of the harder real-world bugs to track down by symptom alone, because the end-user experience is "the internet feels broken" rather than "device X is down." `%IP-4-DUPADDR` short-circuits the whole hunt: it names the contested IP, the segment where the fight is happening, and the MAC fingerprint of the offending device. Match the MAC to a device and the diagnosis is complete.

---

### Check Results

![Check Results summary screen](/assets/images/cptlabs/mod13/lab15_check-results-summary.png)

![Check Results assessment items detail](/assets/images/cptlabs/mod13/lab15_check-results-items.png)

Final score: **23/23** across IP addressing (15/15), Physical port status (5/5), and Other items including console login, console password, enable secret, and hostname (3/3).

---

### Reflection

**What I learned:** Bit-borrowing is a clean trade-off. Each bit borrowed from the host portion doubles the subnet count and halves the hosts per subnet. The dotted-decimal mask values follow directly from the place value of each borrowed bit (128, 192, 224, 240, 248, 252), so once that sequence is in muscle memory, sizing a subnet plan against requirements becomes a one-step calculation rather than a lookup. The `%IP-4-DUPADDR` message was also useful exposure to a real diagnostic that exists in production gear, not just lab gear.

**What was challenging:** Catching the duplicate-address warning. It flashes by during interface bring-up and is easy to scroll past. Linking the offending MAC back to a specific device required pulling up PC-B's IP configuration panel and reading the IPv6 link-local, since the duplicate message only reports the MAC.

**What I would do differently:** Configure the router interfaces fully before touching the PCs, and run `show ip interface brief` after every `no shutdown` to confirm each interface is up cleanly. That sequencing eliminates any window where a PC and its gateway can briefly share an address.

[Back to home](/)
