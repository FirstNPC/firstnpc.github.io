---
layout: page
title: "12.0: Cabling"
permalink: /cptlabs/12-cabling/
---

## Lab 9: Packet Tracer - Connect the Physical Layer

**Module:** 12 | **Completed:** May 11, 2026 | **Score:** 51/51 (51/51 items)

**Skills demonstrated:** Router hardware identification, expansion module selection, cable type selection (straight-through, cross-over, fiber, serial DCE), powering devices off for non-hot-swappable installs, `show ip interface brief` verification, wireless and cellular endpoint configuration

---

### Objective

Identify the physical interfaces and management ports on a Cisco router, select expansion modules to add interface capacity, and cable all devices in a multi-segment network so every PC, laptop, tablet, and access point can reach the network's web server.

---

### Network Topology

**Before** — all devices placed but no cables attached, switches not yet wired:

![Topology before cabling](/assets/images/cptlabs/mod12/lab09_topology-before.png)

**After** — every device cabled with the correct medium and the serial WAN link in place:

![Topology after cabling](/assets/images/cptlabs/mod12/lab09_topology-after.png)

*East and West routers joined by a serial link. East fronts PC1-PC3 directly through an HWIC-4ESW switch module, plus Switch1 (PC4-PC6) and Switch4 over GigabitEthernet. Switch4 crosses over to Switch3, which trunks to Switch2 over fiber. Switch2 hosts PC7-PC9 and the wireless access point that serves the Laptop. The TabletPC reaches the network over either Wi-Fi via the access point or cellular via Cell Tower0.*

---

### Configuration Steps

**Step 1 — Examine East's hardware**

Opened the Physical tab on the East router and zoomed in. Identified the management ports (Console, Auxiliary, USB Console), 2 GigabitEthernet (LAN) interfaces, 2 Serial (WAN) interfaces, and 2 empty HWIC expansion slots. On the CLI tab, ran `show ip interface brief` and confirmed that only the four physical interfaces (plus the virtual `Vlan1`) were listed at this stage. Checked default bandwidth values with `show interface gigabitethernet 0/0` (1,000,000 Kbit) and `show interface serial 0/0/0` (1544 Kbit). The serial bandwidth value is a routing-protocol input for path cost calculation, not the actual provider-negotiated line speed.

**Step 2 — Select and install expansion modules**

Selected `HWIC-4ESW` for East to host PC1, PC2, and PC3 directly without buying a separate switch. The module supports up to 4 hosts. Selected `PT-SWITCH-NM-1FGE` for Switch2 to provide the Gigabit fiber uplink to Switch3. First attempt to drag the HWIC into East returned "Cannot add a module when the power is on" because these chassis are not hot-swappable. Clicked the chassis power switch to power East off, dragged the module into an empty HWIC slot, then powered the router back on. Repeated the same off-install-on sequence for Switch2 with the fiber module in the rightmost empty slot. Confirmed the new interface as `GigabitEthernet5/1` using `show ip interface brief` on Switch2.

**Step 3 — Cable the devices**

Cabled all device-to-device links per the lab table. Used **straight-through copper** for every PC-to-switch run, every PC-to-router-switchport run, and the router-to-switch uplinks (East to Switch1, East to Switch4). Used **cross-over copper** for Switch4 to Switch3 since both endpoints are switches. Used **fiber** for the Switch3 to Switch2 trunk via the newly installed `PT-SWITCH-NM-1FGE` modules on each side. Used a **serial DCE** cable for East to West, connecting the DCE end at East first so East provides the clock for the link.

**Step 4 — Verify interface state on East**

Ran `show ip interface brief` on East to confirm that the configured interfaces had come up. `GigabitEthernet0/0`, `GigabitEthernet0/1`, and `Serial0/0/0` were all up/up. `Serial0/0/1` stayed down because nothing was connected to it. `FastEthernet0/1/3` showed link up but line protocol down for the same reason: the port on the EtherSwitch module was enabled but unused.

**Step 5 — Bring up the wireless and cellular endpoints**

On the Laptop's Config tab, set the `Wireless0` interface to On. Within a few seconds the laptop associated with the access point and DHCP delivered an address. Opened the Web Browser from the Desktop tab and loaded `www.cisco.srv` to confirm reachability all the way through the access point, Switch2, the fiber trunk, Switch3, and over to the web server. Repeated on the TabletPC over `Wireless0`, then disabled `Wireless0` and enabled `3G/4G Cell1` to confirm reachability over the cellular path as well. Both interfaces were not enabled simultaneously, since that can cause routing ambiguity on the device when reaching the same resource.

---

### Verification

**East — show ip interface brief (after cabling)**

```
Interface              IP-Address  OK? Method Status Protocol
GigabitEthernet0/0     172.30.1.1  YES manual up     up
GigabitEthernet0/1     172.31.1.1  YES manual up     up
Serial0/0/0            10.10.10.1  YES manual up     up
Serial0/0/1            unassigned  YES unset  down   down
FastEthernet0/1/0      unassigned  YES unset  up     up
FastEthernet0/1/1      unassigned  YES unset  up     up
FastEthernet0/1/2      unassigned  YES unset  up     up
FastEthernet0/1/3      unassigned  YES unset  up     down
Vlan1                  172.29.1.1  YES manual up     up
```

---

### Troubleshooting

**Problem:** First attempt to insert the `HWIC-4ESW` into East returned "Cannot add a module when the power is on."

**What I tried:** Re-read the lab note. Confirmed these chassis are not hot-swappable, so the device must be powered off before any module is added or removed.

**Resolution:** Clicked the chassis power switch on East's Physical tab to power it off, dragged the module into the empty HWIC slot, then clicked the power switch again to bring it back up. Followed the same off-install-on sequence for the fiber module on Switch2.

---

### Check Results

![Check Results summary screen](/assets/images/cptlabs/mod12/lab09_check-results-summary.png)

![Check Results assessment items detail](/assets/images/cptlabs/mod12/lab09_check-results-items.png)

Final score: **51/51** across 51 assessment items, covering Connect Devices (35/35) and Physical (16/16) components.

---

### Reflection

**What I learned:** Module choice and chassis layout are part of the topology, not separate planning concerns. The `FastEthernet0/1/x` interfaces on East simply do not exist as software entities until the `HWIC-4ESW` is installed and the router is powered back on. `show ip interface brief` is also a fast sanity check after a module install since the new slot and interface name appear in the output as soon as the device finishes booting.

**What was challenging:** Keeping the cable type rules straight across many links. Switch-to-switch over copper takes cross-over, PC-to-switch or PC-to-router-switchport takes straight-through, fiber is its own cable, and the serial link needs the DCE end attached first at the clocking side. Each of these silently produces a down or non-scoring link if mismatched, which is the same failure mode that shows up in production with the wrong patch cable.

**What I would do differently:** Run `show ip interface brief` after each cable insertion instead of waiting until the end. Catching a down link immediately is much faster than walking the whole topology back to find the bad cable.

[Back to home](/)
