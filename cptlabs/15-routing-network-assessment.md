---
layout: page
title: "15.0: Secure Network Devices"
permalink: /cptlabs/15-secure-network-devices/
---

## Lab 21: Packet Tracer - Secure Network Devices

**Module:** 15 | **Completed:** May 17, 2026 | **Score:** 98/98 (52/52 items)

**Skills demonstrated:** SSH configuration, RSA key generation, VTY line hardening, local user authentication, console and privileged EXEC password security, password encryption and minimum-length policy, brute-force login lockout, EXEC session timeouts, MOTD banners, switch SVI management addressing, unused port shutdown

---

### Objective

Configure a router (RTR-A) and a switch (SW-1) to meet a defined set of device-hardening requirements. The work covers securing administrative access over SSH, enforcing a password-strength policy, protecting the login process against brute-force attempts, disabling unused switch ports, and giving the switch a management interface that stays reachable from both LANs. A few best practices are intentionally simplified to keep the activity focused.

---

### Network Topology

**Before** — baseline topology as delivered, before any security configuration: ![Topology before configuration](/assets/images/cptlabs/mod15/lab21_topology-before.png)

**After** — the same topology after RTR-A and SW-1 were hardened: ![Topology after configuration](/assets/images/cptlabs/mod15/lab21_topology-after.png)

*RTR-A routes between two LANs — 192.168.1.0/24, where PC and Laptop attach through SW-1, and 192.168.2.0/24, where Remote PC attaches through SW-2. Because this is a device-hardening lab, the physical topology does not change between the before and after captures; all of the work lives in the running configuration. SW-2 is an unmanaged switch in the path to the remote LAN and is outside this activity's configuration scope.*

---

### Addressing Table

Completing the addressing table is the first task in the activity. The three blank gateway fields are filled with the router interface that serves each device's subnet. The switch SVI also needs a gateway so its replies can be routed back to hosts on the remote LAN.

| Device | Interface | Address | Mask | Gateway |
|:--|:--|:--|:--|:--|
| RTR-A | G0/0/0 | 192.168.1.1 | 255.255.255.0 | N/A |
| RTR-A | G0/0/1 | 192.168.2.1 | 255.255.255.0 | N/A |
| SW-1 | SVI (VLAN 1) | 192.168.1.254 | 255.255.255.0 | 192.168.1.1 |
| PC | NIC | 192.168.1.2 | 255.255.255.0 | 192.168.1.1 |
| Laptop | NIC | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| Remote PC | NIC | 192.168.2.10 | 255.255.255.0 | 192.168.2.1 |

---

### Configuration Steps

**Step 1 — Router baseline hardening**
Disabled DNS lookups so a mistyped command would not stall while the router tried to resolve it as a hostname, set the device hostname to RTR-A, and enforced a ten-character minimum on any newly created password.

**Step 2 — Console and privileged EXEC security**
Set a ten-character console password and a seven-minute idle timeout on the console line, configured an encrypted privileged EXEC password with `enable secret`, added a MOTD banner warning against unauthorized access, and enabled `service password-encryption` so any remaining plaintext passwords in the configuration are obscured.

**Step 3 — SSH and remote administration**
Created the NETadmin account with an encrypted secret, set the `security.com` domain name, and generated a 1024-bit RSA key pair to enable SSH. The VTY lines were then restricted to SSH only, set to authenticate against the local user database, and given the same seven-minute idle timeout.

**Step 4 — Brute-force login protection**
Configured the router to block further login attempts for 45 seconds if three failures occur within a 100-second window, which slows down password-guessing attacks against the management interfaces.

**Step 5 — Disabling unused switch ports**
Administratively shut down every switch port not in use, leaving only the uplink to RTR-A and the two host ports active. See Troubleshooting below — the interface range used here initially caught a live host port.

**Step 6 — Switch management interface**
Assigned 192.168.1.254/24 to the VLAN 1 SVI and brought it up, then set a default gateway of 192.168.1.1. The default gateway is what lets the SVI answer pings from Remote PC on the 192.168.2.0/24 LAN, since that reply has to be routed off the local subnet.

**Step 7 — Switch privileged EXEC and SSH**
Configured the switch with an encrypted privileged EXEC password, the `security.com` domain name, the SW-1 hostname, and a 1024-bit RSA key pair. The NETadmin account was created and the VTY lines were restricted to SSH with local authentication, so only that account can open a management session.

---

### Device Configuration

**RTR-A — configuration session**

RTR-A's G0/0/0 and G0/0/1 interface addresses were pre-configured in the activity file per the addressing table; the session below is the security configuration that was added. Boot output has been trimmed.

```
Router>enable
Router#configure terminal
Router(config)#no ip domain-lookup
Router(config)#hostname RTR-A
RTR-A(config)#security passwords min-length 10
RTR-A(config)#line con 0
RTR-A(config-line)#password @Cons1234!
RTR-A(config-line)#exec-timeout 7 0
RTR-A(config-line)#exit
RTR-A(config)#enable secret @Cons1234!
RTR-A(config)#banner motd *Unauthorized access prohibited.*
RTR-A(config)#service password-encryption
RTR-A(config)#username NETadmin secret LogAdmin!9
RTR-A(config)#ip domain-name security.com
RTR-A(config)#crypto key generate rsa
The name for the keys will be: RTR-A.security.com
How many bits in the modulus [512]: 1024
% Generating 1024 bit RSA keys, keys will be non-exportable...[OK]

RTR-A(config)#line vty 0 4
%SSH-5-ENABLED: SSH 1.99 has been enabled
RTR-A(config-line)#exec-timeout 7 0
RTR-A(config-line)#transport input ssh
RTR-A(config-line)#login local
RTR-A(config-line)#exit
RTR-A(config)#login block-for 45 attempts 3 within 100
```

**SW-1 — configuration session**

Boot output trimmed; the repeated link-state messages from the port shutdown are summarized in brackets.

```
Switch>enable
Switch#configure terminal
Switch(config)#interface range fa0/1, fa0/3-24, g0/2
Switch(config-if-range)#shutdown
[output trimmed — Fa0/1, Fa0/3 through Fa0/24, and G0/2 changed state to administratively down]
Switch(config-if-range)#interface fa0/10
Switch(config-if)#no shutdown
%LINK-5-CHANGED: Interface FastEthernet0/10, changed state to up
Switch(config-if)#exit
Switch(config)#interface vlan 1
Switch(config-if)#ip address 192.168.1.254 255.255.255.0
Switch(config-if)#no shutdown
%LINK-5-CHANGED: Interface Vlan1, changed state to up
Switch(config-if)#exit
Switch(config)#enable secret @Cons1234!
Switch(config)#ip domain-name security.com
Switch(config)#hostname SW-1
SW-1(config)#crypto key generate rsa
The name for the keys will be: SW-1.security.com
How many bits in the modulus [512]: 1024
% Generating 1024 bit RSA keys, keys will be non-exportable...[OK]

SW-1(config)#username NETadmin secret LogAdmin!9
%SSH-5-ENABLED: SSH 1.99 has been enabled
SW-1(config)#line vty 0 4
SW-1(config-line)#login local
SW-1(config-line)#transport input ssh
SW-1(config-line)#exit
SW-1(config)#ip default-gateway 192.168.1.1
```

The three active ports on SW-1 — the G0/1 uplink to RTR-A and the two host ports Fa0/2 and Fa0/10 — were left enabled. Every other port was administratively shut.

---

### Troubleshooting

**Problem:** The interface range used to disable unused ports, `fa0/1, fa0/3-24, g0/2`, also caught FastEthernet0/10, which is one of the live host ports. The console immediately reported `FastEthernet0/10, changed state to administratively down`, which would have knocked a host off the network.

**What I tried:** Compared the range against the topology and the boot-time link-up messages, which showed Fa0/2, Fa0/10, and G0/1 as the connected ports. Fa0/10 fell inside the `fa0/3-24` portion of the range and was shut along with the genuinely unused ports.

**Resolution:** Re-entered interface configuration for Fa0/10 specifically and ran `no shutdown` to bring it back up. The fix reinforces the lesson: when shutting unused ports with a range, the connected ports have to be excluded explicitly rather than assumed to fall outside a broad span.

---

### Check Results

![Activity results summary](/assets/images/cptlabs/mod15/lab21_check-results-summary.png)
![Assessment items detail](/assets/images/cptlabs/mod15/lab21_check-results-items.png)

Final score: **98/98 (52/52 items)**, with every assessment item passing. The score breaks down across five graded components — Basic Device Configuration 10/10, Basic Switch Configuration 37/37, Enhance Session Security 15/15, Enhanced Password Security 6/6, and SSH Configuration 30/30.

---

### Reflection

**What I learned:** Securing a device is a layered job rather than a single setting. SSH on its own is not "secure remote access" — it needs a domain name and an RSA key pair to function at all, the VTY lines have to be told to accept only SSH, and they need a real authentication source behind them, which is the local user database here. Wrapping that in a password-length policy, encrypted secrets, idle timeouts, and a login-attempt lockout is what actually hardens the management plane. Order matters too: the hostname and domain name must already exist before `crypto key generate rsa`, because the key pair is named after them.

**What was challenging:** The interface range for shutting unused ports. Writing one broad range is tempting, but `fa0/3-24` quietly included the live Fa0/10 host port and took it offline. Catching that from the console messages and correcting it was the main hurdle, and a good reminder to confirm which ports are actually connected before disabling a span.

**What I would do differently:** Identify and note the connected ports first, then build the shutdown range around them so no live port is ever caught — shutting `fa0/1, fa0/3-9, fa0/11-24, g0/2` directly instead of disabling Fa0/10 and undoing it. I would also finish by saving the running configuration to startup with `copy running-config startup-config` so the work survives a reload.

[Back to home](/)
