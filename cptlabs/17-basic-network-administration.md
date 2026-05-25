---
layout: page
title: "17.0: Basic Network Administration"
permalink: /cptlabs/17-basic-network-administration/
---

## Lab 40: Packet Tracer - Configure Secure Passwords and SSH

**Module:** 17 | **Completed:** May 25, 2026 | **Score:** 213/213 (67/67 items)

**Skills demonstrated:** Router and switch device hardening, service password encryption, minimum password length policy, RSA key generation, SSH (vs Telnet) on VTY lines, local user authentication, login attempt blocking, EXEC timeout, shutting down unused switch ports with `interface range`, default gateway on a Layer 2 switch, end-to-end SSH verification from a client

---

### Objective

Prepare router RTA and switch SW1 for deployment by enabling a layered set of security measures: encrypted passwords, a strong secret, restricted DNS behavior, an RSA key pair, SSH-only VTY access with local authentication, login-attempt blocking, EXEC timeout, and disabled unused ports on the switch. Verify the hardening by establishing an SSH session from PCA to RTA.

---

### Network Topology

**Before** — devices placed but interfaces not yet configured:

![Topology before configuration](/assets/images/cptlabs/mod17/lab40_topology-before.png)

**After** — RTA G0/0 and SW1 VLAN 1 brought up, PCA reaching RTA over the network:

![Topology after configuration](/assets/images/cptlabs/mod17/lab40_topology-after.png)

*Three-device segment: PCA (172.16.1.10/24) and RTA G0/0 (172.16.1.1/24) connected through SW1, with SW1's management interface on VLAN 1 (172.16.1.2/24) and its default gateway pointing at RTA.*

---

### Configuration Steps

**Step 1 — Address PCA**

Configured PCA's NIC with IP 172.16.1.10, mask 255.255.255.0, and default gateway 172.16.1.1 from the Desktop > IP Configuration window.

**Step 2 — Console into RTA and set the hostname and interface**

Opened the Terminal on PCA, pressed Enter to reach the router CLI, and entered privileged EXEC with `enable`. From global config (`configure terminal`) set `hostname RTA`, then on `interface GigabitEthernet0/0` assigned `ip address 172.16.1.1 255.255.255.0` and brought it up with `no shutdown`.

**Step 3 — Apply password and DNS hardening on RTA**

Enabled service-wide password encryption with `service password-encryption`. Set the minimum length policy with `security password min-length 10`. Disabled DNS lookups (`no ip domain-lookup`) so mistyped commands would not trigger long DNS resolution attempts. Set the domain name to `CCNA.com` (case-sensitive — this matters both for the RSA key name and for the Packet Tracer scoring engine).

**Step 4 — Create a local user and generate RSA keys**

Created a local user with `username firstnpc secret firstnpcspassword`. The `secret` keyword stores the password as an MD5 hash rather than plain text. Then ran `crypto key generate rsa` and selected a 1024-bit modulus when prompted. The router named the key `RTA.CCNA.com` from the hostname and domain set in Step 3, and SSH 1.99 was enabled automatically once the key existed.

**Step 5 — Lock down login attempts and VTY lines**

Set `login block-for 180 attempts 4 within 120` so that four failed logins inside a two-minute window lock anyone out for three minutes. On `line vty 0 4`, restricted the transport to SSH (`transport input ssh`), pointed authentication at the local user database (`login local`), and set `exec-timeout 6` to drop idle sessions after six minutes.

**Step 6 — Save, then add the missing enable secret**

Saved with `copy running-config startup-config`. On review noticed the enable secret had not been set, re-entered config, and added `enable secret networkplus` before saving a second time.

**Step 7 — Mirror the hardening on SW1**

Opened the SW1 CLI. Set the hostname, brought up the management SVI with `interface vlan 1` and `ip address 172.16.1.2 255.255.255.0`, then configured the Layer 2 switch's default gateway with `ip default-gateway 172.16.1.1`. Shut down all unused access ports in a single command with `interface range f0/2-24, g0/2` followed by `shutdown`. Applied the same hardening sequence as RTA: `service password-encryption`, `enable secret class`, `no ip domain-lookup`, `ip domain-name CCNA.com`, RSA key generation at 1024 bits, `username admin secret adminspassword`, and `line vty 0 15` with `transport input ssh`, `login local`, and `exec-timeout 6`. The SW1 enable secret was set to `class` and the RTA enable secret to `networkplus` — deliberately different values across the two devices so that compromise of one secret does not extend privileged access to the other. The same reasoning applies to the local user accounts: `firstnpc` on RTA, `admin` on SW1. Saved with `copy run start`.

**Step 8 — Verify SSH from PCA**

From the PCA command prompt, first checked the Packet Tracer SSH client's syntax with the help flag:

```
C:\> ssh /?
Packet Tracer PC SSH
Usage: SSH -l username target
```

Then established the SSH session into RTA using the local username:

```
C:\> ssh -l firstnpc 172.16.1.1
Password:
RTA>
```

The router accepted the local credentials and dropped into user EXEC, confirming that the VTY lines were reachable, the local-auth path was working, and Telnet was no longer accepted.

---

### Device Configuration

> The CLI capture for this lab recorded the live configuration session rather than `show running-config` output. The blocks below are reconstructed from the commands entered and reflect what the running-config should contain. If you want the verbatim output, re-open the .pka, run `show run` on each device, and paste it in place of these blocks.

**Router RTA — reconstructed running-config (key sections)**

```
service password-encryption
!
hostname RTA
!
enable secret 5 <hash>
!
security passwords min-length 10
!
ip domain-name CCNA.com
no ip domain-lookup
!
username firstnpc secret 5 <hash>
!
login block-for 180 attempts 4 within 120
!
interface GigabitEthernet0/0
 ip address 172.16.1.1 255.255.255.0
 no shutdown
!
line vty 0 4
 exec-timeout 6 0
 login local
 transport input ssh
!
end
```

**Switch SW1 — reconstructed running-config (key sections)**

```
service password-encryption
!
hostname SW1
!
enable secret 5 <hash>
!
ip default-gateway 172.16.1.1
ip domain-name CCNA.com
no ip domain-lookup
!
username admin secret 5 <hash>
!
interface range FastEthernet0/2 - 24 , GigabitEthernet0/2
 shutdown
!
interface Vlan1
 ip address 172.16.1.2 255.255.255.0
 no shutdown
!
line vty 0 15
 exec-timeout 6 0
 login local
 transport input ssh
!
end
```

---

### Troubleshooting

**Problem 1:** Typed `conf t` at the user EXEC prompt (`Router>`) instead of from privileged EXEC.

**What happened:** IOS returned `% Invalid input detected at '^' marker.` because `configure terminal` is only available after `enable`.

**Resolution:** Ran `enable` to enter privileged EXEC, then `configure terminal` worked normally.

---

**Problem 2:** Took the lab directions' example syntax (`username any_user secret any_password`) too literally and typed `username firstnpc ccna.com secret firstnpcspassword`, treating the domain as a separate argument.

**What happened:** IOS rejected it with `% Invalid input detected at '^' marker.` The caret pointed at `ccna.com` because `username` expects one word for the user, then keywords like `secret` or `privilege`.

**Resolution:** Re-entered as `username firstnpc secret firstnpcspassword`. The username does not include the domain — the domain name is set globally with `ip domain-name` and is only used by the device to label the RSA key and accept SSH connections.

---

**Problem 3:** Saved the running-config to NVRAM before realizing the enable secret had never been set on RTA.

**What happened:** None of the configured commands had set an enable secret, so privileged EXEC was unprotected on RTA even after `copy run start`.

**Resolution:** Re-entered global config and ran `enable secret networkplus`, then saved again. On a production device this matters more than in the lab — without an enable secret, anyone with VTY access can drop straight into privileged EXEC.

---

### Check Results

![Check Results summary screen](/assets/images/cptlabs/mod17/lab40_check-results-summary.png)

![Check Results assessment items detail](/assets/images/cptlabs/mod17/lab40_check-results-items.png)

Final score: **213/213** across 67 assessment items covering Basic Security Configuration (33/33), Configuration Management (12/12), Default Gateway Configuration (15/15), Device Hardening Configuration (93/93), Device Interface Configuration (40/40), Hostname Configuration (10/10), and IPv4 Host Address Configuration (10/10).

---

### Reflection

**What I learned:** SSH on Cisco IOS is not a single command — it is the byproduct of four things being in place at the same time: a hostname and domain name (so the device can name the key), an RSA key pair of sufficient size, local credentials (or AAA), and VTY lines that explicitly accept `transport input ssh` with `login local`. Removing any one of those breaks SSH access. Also reinforced that `enable secret` and `username … secret …` use a hashed password, while `service password-encryption` only obfuscates plain-text passwords already in the config — the two are not equivalent.

**What was challenging:** Tracking the order of operations across two devices. The hardening commands are the same on RTA and SW1, but the prerequisites differ: SW1 needs a default gateway for its management VLAN to reach anything off-subnet, the unused-port shutdown is switch-specific, and the VTY range is `0 4` on the router versus `0 15` on the switch. Easy to mirror one for the other without thinking.

**What I would do differently:** Build a hardening checklist and work it top-down rather than following the lab steps in order. The lab walks through the commands in a sequence that made sense for teaching, but it leaves the enable secret until late and mixes interface configuration in with security configuration. A checklist organized by security domain (authentication, then access control, then logging, then interface state) catches the missing-enable-secret gap before saving rather than after.

[Back to home](/)
