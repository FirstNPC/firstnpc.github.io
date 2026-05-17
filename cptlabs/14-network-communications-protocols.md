---
layout: page
title: "14.0: Network Communications Protocols"
permalink: /cptlabs/14-network-communications-protocols/
---

## Lab 17: Packet Tracer - Configure DHCP on a Wireless Router

**Module:** 14 | **Completed:** May 16, 2026 | **Score:** 9/9 (9/9 items)

**Skills demonstrated:** wireless router web administration, DHCP server configuration, LAN IP addressing, DHCP client configuration, lease renewal, ipconfig verification, ping connectivity testing

---

### Objective
Build a small home network of three PCs served by a single wireless router, move the router's LAN IP address and DHCP scope onto a specified network range, and confirm that every PC obtains its addressing automatically through DHCP.

---

### Network Topology
**Before** - the DHCP Enabled Router placed on the workspace before any PCs were connected: ![Topology before](/assets/images/cptlabs/mod14/lab17_topology-before.png)
**After** - three PCs connected, all link lights green: ![Topology after](/assets/images/cptlabs/mod14/lab17_topology-after.png)
*Each PC connects to an Ethernet port on the wireless router with a straight-through copper cable. The wireless router is both the default gateway and the DHCP server for the whole LAN — every address the three PCs use originates from this one device, which is what makes a single scope change ripple out to every client.*

---

### DHCP Configuration Plan
The lab moves the network off the router's factory defaults and onto a chosen range. Because a DHCP scope has to sit on the router's own subnet, changing the Router IP relocates the scope along with it.

| Setting | Default | Configured |
|:--|:--|:--|
| Router LAN IP | 192.168.0.1 | 192.168.5.1 |
| Subnet mask | 255.255.255.0 | 255.255.255.0 |
| DHCP server | Enabled | Enabled |
| DHCP start address | 192.168.0.100 | 192.168.5.126 |
| Maximum number of users | 50 | 75 |
| Resulting address pool | 192.168.0.100 to 192.168.0.149 | 192.168.5.126 to 192.168.5.200 |

---

### Configuration Steps
**Step 1 - Set up the topology**
Placed three generic PCs and one wireless router on the workspace, then connected each PC's Ethernet port to a LAN port on the router with straight-through copper cables. The link lights start amber while the ports negotiate and turn green once the links are up.

**Step 2 - Observe the default DHCP settings**
Set PC0's Desktop > IP Configuration to DHCP. It immediately leased 192.168.0.100 with a default gateway of 192.168.0.1 — confirmation that the router's DHCP server was active out of the box.
![PC0 leases an address on the default network](/assets/images/cptlabs/mod14/lab17-02_pc0-dhcp-toggled-to-get-ip.png)
Browsed to 192.168.0.1, logged in with the username admin and password admin, and reviewed the Basic Setup page: DHCP enabled, scope starting at 192.168.0.100, capacity for 50 clients.
![Router Basic Setup page showing factory defaults](/assets/images/cptlabs/mod14/lab17-05_pc0-access-router-settings.png)

**Step 3 - Change the Router IP**
In Router IP Settings, changed the IP address to 192.168.5.1 and saved.
![Router IP changed to 192.168.5.1](/assets/images/cptlabs/mod14/lab17-06_pc0-change-ip-192-168-5-1.png)
Saving the change moved the router off its old address, so the link lights turned amber and the PCs lost connectivity.
![Link lights amber after the router IP change](/assets/images/cptlabs/mod14/lab17-08_color-of-turning-amber-due-to-network-changes.png)
Renewed PC0's lease by toggling IP Configuration to Static and back to DHCP, then reconnected to the GUI at the new address.
![Logging back in to the router at 192.168.5.1](/assets/images/cptlabs/mod14/lab17-13_login-to-new-gateway-address.png)

**Step 4 - Change the DHCP range**
With the scope now on the 192.168.5.0 network, changed the DHCP Starting IP Address to 192.168.5.126 and the Maximum Number of Users to 75, then saved.
![DHCP scope set to start at 192.168.5.126 for 75 users](/assets/images/cptlabs/mod14/lab17-14_change-dhcp-range.png)
Renewed PC0's lease again; it picked up the first address in the new scope, 192.168.5.126.
![PC0 leases 192.168.5.126](/assets/images/cptlabs/mod14/lab17-19_pc0-toggle-static-dhcp.png)

**Step 5 - Enable DHCP on the other PCs**
Set PC1 and PC2 to DHCP. They leased the next two addresses in the scope: PC1 took 192.168.5.127 and PC2 took 192.168.5.128.
![PC1 leases 192.168.5.127](/assets/images/cptlabs/mod14/lab17-20_pc1-toggle-static-dhcp.png)
![PC2 leases 192.168.5.128](/assets/images/cptlabs/mod14/lab17-21_pc2-toggle-static-dhcp.png)

**Step 6 - Verify connectivity**
From PC2's Command Prompt, ran `ipconfig` to confirm the leased address, then pinged the router and both other PCs. The output is shown in the Verification section below.

---

### Device Configuration
A wireless router in Packet Tracer is administered through its web GUI rather than a command-line interface, so there is no `show running-config` to capture. The router's final Basic Setup state and each client's leased addressing are recorded below.

**Wireless Router - Basic Setup (final state)**
```
Local Router IP Address ....... 192.168.5.1
Subnet Mask ................... 255.255.255.0
DHCP Server ................... Enabled
Start IP Address .............. 192.168.5.126
Maximum Number of Users ....... 75
DHCP Address Range ............ 192.168.5.126 to 192.168.5.200
```

**Client leased addressing**

| Device | IPv4 Address | Subnet Mask | Default Gateway |
|:--|:--|:--|:--|
| PC0 | 192.168.5.126 | 255.255.255.0 | 192.168.5.1 |
| PC1 | 192.168.5.127 | 255.255.255.0 | 192.168.5.1 |
| PC2 | 192.168.5.128 | 255.255.255.0 | 192.168.5.1 |

---

### Verification
From PC2's Command Prompt, `ipconfig` confirmed the leased address and all three pings succeeded with no loss:

```
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Link-local IPv6 Address.........: FE80::201:97FF:FED7:AC73
   IPv4 Address....................: 192.168.5.128
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: 192.168.5.1

C:\>ping 192.168.5.1

Pinging 192.168.5.1 with 32 bytes of data:

Reply from 192.168.5.1: bytes=32 time<1ms TTL=255
Reply from 192.168.5.1: bytes=32 time<1ms TTL=255
Reply from 192.168.5.1: bytes=32 time<1ms TTL=255
Reply from 192.168.5.1: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.5.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\>ping 192.168.5.126

Reply from 192.168.5.126: bytes=32 time<1ms TTL=128
Reply from 192.168.5.126: bytes=32 time<1ms TTL=128
Reply from 192.168.5.126: bytes=32 time<1ms TTL=128
Reply from 192.168.5.126: bytes=32 time=9ms TTL=128

Ping statistics for 192.168.5.126:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\>ping 192.168.5.127

Reply from 192.168.5.127: bytes=32 time<1ms TTL=128
Reply from 192.168.5.127: bytes=32 time<1ms TTL=128
Reply from 192.168.5.127: bytes=32 time<1ms TTL=128
Reply from 192.168.5.127: bytes=32 time<1ms TTL=128

Ping statistics for 192.168.5.127:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

Every ping returned four of four replies with 0% loss. The router replies at TTL 255 and the PCs reply at TTL 128 — these are simply the two stacks' default starting TTLs. Because all four devices sit on the same 192.168.5.0/24 segment, nothing is routed and the TTL is never decremented in transit.

---

### Troubleshooting
**Problem:** After each router-side change — first the Router IP, then the DHCP scope — the three link lights turned amber and the PCs lost connectivity. A PC does not automatically move to a new network; it holds its existing lease.

**What I tried:** Checked PC0's IP Configuration after the Router IP change and saw it still held a 192.168.0.x address from the old network, which is why it could not reach the router at 192.168.5.1.

**Resolution:** Renewed each PC's lease by toggling IP Configuration from DHCP to Static and back to DHCP. Each PC then requested a fresh address on the 192.168.5.0/24 network and the link lights returned to green. This renewal step is expected after any addressing change, not a fault.

---

### Check Results
![Assessment summary](/assets/images/cptlabs/mod14/lab17_check-results-summary.png)
![Assessment items detail](/assets/images/cptlabs/mod14/lab17_check-results-items.png)
Final score: **9/9** (9 of 9 assessment items correct). The graded items covered the router's DHCP pool — default gateway, maximum users, and start IP address — and the IP address and subnet mask leased by each of PC0, PC1, and PC2.

---

### Reflection
**What I learned:** How DHCP is delivered by a SOHO wireless router. The router runs a DHCP server bound to its own LAN subnet, so moving the router's IP address moves the entire scope with it. I also saw that a client holds its existing lease until it is renewed, which is why the lease has to be refreshed when the network underneath it changes.

**What was challenging:** Keeping track of when each PC needed its lease renewed. After both the Router IP change and the DHCP scope change, the link lights turned amber and the PCs kept stale addresses until I toggled each one between Static and DHCP.

**What I would do differently:** Renew every PC's lease immediately after each router-side change rather than going back to restore connectivity afterward. I would also record the default settings before editing them so I keep a clean before-and-after reference.

[Back to home](/)
