---
layout: page
title: "11.0: Network Topologies"
permalink: /cptlabs/11-network-topologies/
---

## Lab 4: Packet Tracer - Configure Initial Router Settings

**Module:** 11 | **Completed:** May 10, 2026 | **Score:** 80/80 (10/10 items)

**Skills demonstrated:** Router CLI navigation, hostname configuration, MOTD banner, password encryption, enable secret vs enable password, console line login, saving configuration to NVRAM

---

### Objective

Configure a Cisco router by setting secure and plain-text passwords, adding a login warning banner, and verifying then saving the running configuration to NVRAM.

---

### Network Topology

**Before** — PCA and R1 present, no connection established:

![Topology before console connection](/assets/images/cptlabs/mod11/lab04_topology-before.png)

**After** — PCA connected to R1 via console cable (RS-232 to Console):

![Topology after console connection](/assets/images/cptlabs/mod11/lab04_topology-after.png)

*Single Cisco 1941 router (R1) with 2 GigabitEthernet interfaces, 2 Serial interfaces, and 4 FastEthernet switch ports. Configuration performed via console cable from PCA.*

---

### Configuration Steps

**Step 1 — Verify the default configuration**

Connected PCA to R1 via console cable (RS-232 to Console). Opened Terminal on PCA and pressed Enter to reach the router CLI. Entered privileged EXEC mode with `enable`. Ran `show running-config` to examine the default state. The router had no hostname, no passwords, and 2 GigabitEthernet, 2 Serial, and 4 FastEthernet interfaces. Ran `show startup-config` which returned "not present" because nothing had been saved to NVRAM yet.

**Step 2 — Set the hostname**

Entered global configuration mode with `config t`. Set the hostname to R1 using `hostname R1`.

**Step 3 — Configure the MOTD banner**

Configured the MOTD banner with `banner motd "Unauthorized access is strictly prohibited."` to warn anyone attempting unauthorized access.

**Step 4 — Configure passwords**

Enabled password encryption with `service password-encryption`. Set the unencrypted enable password to `cisco` and the encrypted enable secret to `itsasecret`. Configured the console line with `line console 0`, set the password to `letmein`, and enabled login with the `login` command.

**Step 5 — Verify and save**

Exited config mode and logged out to verify the settings. The MOTD banner appeared on reconnect and the console password prompt was presented. Re-entered privileged EXEC mode using `itsasecret`. Only the enable secret worked at this point because when both `enable password` and `enable secret` are configured, the secret always takes precedence. Saved the running config to NVRAM with `copy running-config startup-config` (shortest form: `copy run start`).

---

### Device Configuration

**Router R1 — show running-config**

```
version 15.1
no service timestamps log datetime msec
no service timestamps debug datetime msec
service password-encryption
!
hostname R1
!
enable secret 5 $1$mERr$ILwq/b7kc.7X/ejA4Aosn0
enable password 7 0822455D0A16
!
ip cef
no ipv6 cef
!
license udi pid CISCO1941/K9 sn FTX152459PZ
!
spanning-tree mode pvst
!
interface GigabitEthernet0/0
 no ip address
 duplex auto
 speed auto
 shutdown
!
interface GigabitEthernet0/1
 no ip address
 duplex auto
 speed auto
 shutdown
!
interface Serial0/0/0
 no ip address
 clock rate 2000000
 shutdown
!
interface Serial0/0/1
 no ip address
 clock rate 2000000
 shutdown
!
interface FastEthernet0/1/0
 switchport mode access
 switchport nonegotiate
 shutdown
!
interface Vlan1
 no ip address
 shutdown
!
banner motd ^CUnauthorized access is strictly prohibited.^C
!
line con 0
 password 7 082D495A041C0C19
 login
!
line vty 0 4
 login
!
end
```

---

### Troubleshooting

**Problem:** Typed "prohitbited" instead of "prohibited" in the MOTD banner.

**What I tried:** Proceeded with the lab since it did not affect functionality or the Check Results score.

**Resolution:** To correct it, re-enter the full banner command:

```
banner motd "Unauthorized access is strictly prohibited."
```

---

### Check Results

![Check Results summary screen](/assets/images/cptlabs/mod11/lab04_check-results-summary.png)

![Check Results assessment items detail](/assets/images/cptlabs/mod11/lab04_check-results-items.png)

Final score: **80/80** across 10 assessment items covering Basic Security Configuration, Configuration Management, Device Connection, and Hostname Configuration.

---

### Reflection

**What I learned:** The difference between `enable password` and `enable secret`. When both are configured, the secret always overrides the plain-text password. Also learned that `service password-encryption` encrypts existing and future plain-text passwords in the config file, though this encryption is considered weak and mainly deters casual viewing rather than serious attacks.

**What was challenging:** Remembering to add the `login` command under `line console 0`. Without it, the console password is set but never actually prompted, which the lab specifically flags as a common mistake.

**What I would do differently:** Proofread banner text before pressing Enter. There is no quick inline edit for banners. You have to re-enter the full `banner motd` command to overwrite it.

[Back to home](/)
