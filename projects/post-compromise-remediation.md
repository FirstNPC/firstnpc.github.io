---
layout: page
title: "The Clean Log and the Smoking Gun: An AiTM Compromise Investigation"
permalink: /projects/post-compromise-remediation/
---

# The Clean Log and the Smoking Gun: An AiTM Compromise Investigation

A user account at a small business client was compromised through a successful phishing attempt. I reset the password and revoked sessions on first detection, which is the textbook starting move. This case study covers everything that came after, because a password reset alone leaves most attacker persistence intact, and because the actual compromise vector turned out to be more interesting than the surface symptoms suggested.

## Background

| Item | Detail |
|---|---|
| Environment | Microsoft 365 tenant, Exchange Online, Entra ID |
| MFA status before incident | Not configured for the affected user |
| Tenant licensing | Exchange Online Plan 1 (no Conditional Access available) |

The tenant had Security Defaults disabled, meaning no baseline MFA was being enforced for any user.

[![Entra admin center showing Security defaults set to Disabled](/assets/post-compromise-remediation/00-security-defaults-disabled.png)](/assets/post-compromise-remediation/00-security-defaults-disabled.png){:target="_blank"}

This is the precondition that allowed the compromise. With Security Defaults disabled and no Conditional Access policies (the tenant was on Exchange Online Plan 1, which does not include Entra ID P1), every account in the tenant was protected only by a password.

## The Symptom That Prompted Investigation

The user reported being able to send emails but not receive them. A review of the user's mailbox rules in Outlook on the Web showed why.

[![Outlook rules showing a suspicious rule named "." that marks all incoming messages as read and moves them to Conversation History](/assets/post-compromise-remediation/01-outlook-rules.png)](/assets/post-compromise-remediation/01-outlook-rules.png){:target="_blank"}

A rule named "." (a single period) had been created on the mailbox. It marked every incoming message as Read and moved it to the Conversation History folder. This is a common attacker tactic. It hides reply emails from the user, especially replies to phishing messages the attacker is sending from the compromised mailbox, while leaving the inbox apparently intact. The unusual rule name and behavior were the first clear indicator of compromise.

## Why the Initial Response Was Not Enough

My password reset and session revocation were correct first moves, but they only address part of the problem. A password reset closes the front door. It does not address:

- Authentication methods the attacker registered for themselves (their own phone number, their own authenticator app, or a FIDO key)
- Inbox rules silently hiding or forwarding mail
- Native mailbox forwarding configured at the Exchange level
- OAuth app consents granted to malicious third-party apps, which survive password resets
- Registered devices the attacker enrolled
- Group memberships modified to expand attacker reach

Each of these can re-establish access or exfiltrate mail without ever needing the password again. As it turned out, the actual compromise vector was something a password reset cannot touch at all.

## What Actually Happened: Reconstructing the Compromise

Microsoft 365 keeps two separate logs of every sign-in to a user's account, and reading both of them was where the real story emerged.

### The Two Sign-In Logs

**Interactive sign-in:** the user actively logs in. Types their password, completes any MFA prompt, and gets a session.

**Non-interactive sign-in:** an app reauthenticates silently in the background using a token issued from a previous interactive sign-in. No password, no prompt, no user involvement.

Both logs matter because a clever attacker can steal a session token and keep using it without ever logging in interactively again. If you only review the interactive log, the attack is invisible.

### How I Pulled the Logs

In the Entra admin center under Sign-in events, I filtered to the affected user and applied a 30-day date range with Status set to Success, then exported each tab as a CSV.

[![Sign-in events page filtered to the affected user, showing the User sign-ins (interactive) tab](/assets/post-compromise-remediation/02-interactive-signins.png)](/assets/post-compromise-remediation/02-interactive-signins.png){:target="_blank"}

The interactive tab returned 58 entries over 30 days, all from the user's expected IP ranges. Nothing visibly wrong.

[![Sign-in events page filtered to the affected user, showing the User sign-ins (non-interactive) tab](/assets/post-compromise-remediation/03-noninteractive-signins.png)](/assets/post-compromise-remediation/03-noninteractive-signins.png){:target="_blank"}

The non-interactive tab returned 3,603 entries over the same period. The entries from commercial datacenter IP ranges were the smoking gun.

### The Smoking Gun in the Data

Cross-referencing the two CSVs side by side gave me the exact moment of compromise.

[![Side-by-side view of the interactive and non-interactive sign-in CSVs in Excel, with the AiTM moment highlighted: an interactive sign-in at 2026-04-16T17:25:22Z from a Mozilla browser, followed one second later by a non-interactive sign-in from the same IP with user agent "node"](/assets/post-compromise-remediation/04-csv-evidence.png){:width="100%"}](/assets/post-compromise-remediation/04-csv-evidence.png){:target="_blank"}

The two highlighted rows are one second apart, from the same IP, with completely different user agents. The interactive entry shows a normal Mozilla Windows browser. The non-interactive entry shows the literal string `node`, which is what Node.js scripts identify as. No human switches from a browser to a script in one second.

This is the AiTM (Adversary in the Middle) signature. AiTM phishing works by standing up a real-time proxy that sits between the user and the real Microsoft login page. The user's actual credentials authenticate successfully against Microsoft, so the login works and the user lands in their inbox. But during that handoff, the attacker captures the session token Microsoft issued. From that moment on, the attacker can replay that token from anywhere in the world, and Microsoft will accept it as a valid signed-in session with no further authentication required.

The interactive entry was the proxy server impersonating the user's browser to Microsoft. The non-interactive entry was the attacker's automation taking over the session one second later.

### How Long They Were Inside

The earliest hostile non-interactive sign-in was 4/16 at 17:25:23 UTC. The latest was 4/30 at 14:53:26 UTC, shortly before remediation began. That is a 14-day window.

For 14 days, scripts on two commercial datacenter servers (one in Los Angeles, one in Utah, both on commercial VPS hosting ASNs) were continuously reading the mailbox through the Microsoft 365 API at a rate of roughly 100 requests per day. The user noticed nothing because the attacker traffic was invisible to their normal experience.

The compromise was only detected because the attacker's inbox rule interfered with the user's ability to see incoming mail, prompting them to ask for help.

This is the real cost of not having MFA enabled. Not the moment of the breach, but the silent occupancy that follows it.

## What Was Remediated and How

Once it was clear the compromise vector was a stolen session token rather than a stolen password, the remediation priorities shifted. A password reset alone is meaningless against an attacker using a token, because the token survives password resets.

The full remediation, in order:

### 1. Password reset

On first detection of suspicious activity, I reset the user's password to a new strong value.

### 2. Sessions revoked

I revoked all active sign-in sessions through the Entra admin center, invalidating any session tokens issued before the reset.

### 3. Authentication methods cleared

Entra admin center: Users > [user] > Authentication methods. I removed every method present and started clean. The attacker had registered a phone number that did not match the user's known number.

### 4. Inbox rules wiped

Connected to Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline

# Check native mailbox forwarding
Get-Mailbox user@domain.com | Select ForwardingAddress, ForwardingSmtpAddress, DeliverToMailboxAndForward

# List all inbox rules
Get-InboxRule -Mailbox user@domain.com | Format-List Name, Enabled, ForwardTo, RedirectTo, DeleteMessage, MoveToFolder
```

Rather than auditing each rule's logic for subtle malicious behavior, I wiped them all:

```powershell
Get-InboxRule -Mailbox user@domain.com | Remove-InboxRule -Confirm:$false
```

The user recreated their two legitimate rules from scratch.

### 5. Tenant-level forwarding posture verified

```powershell
Get-HostedOutboundSpamFilterPolicy | Select Name, AutoForwardingMode
```

`AutoForwardingMode` was set to `Automatic`, the current Microsoft default that blocks external forwarding.

### 6. OAuth app consents reviewed

Entra admin center: Users > [user] > Applications. I reviewed each application the user had granted consent to and revoked anything unfamiliar. OAuth consents are the persistence path that survives password changes most cleanly, so this is the step most often skipped and most often the cause of "we got hit again two weeks later."

### 7. Registered devices removed

Removed every device under the user's profile. The user re-registered their phone and laptop on next sign-in.

### 8. Group memberships compared to baseline

No anomalies found.

### 9. Final session revocation

A second session revocation after all other cleanup steps, to invalidate any tokens still held by the attacker.

### 10. Security Defaults recommended and enabled

I recommended Security Defaults to the business owner with a plain-language explanation of what changes for end users (a one-time Authenticator setup, then prompts only on new devices and locations) and what protection it provides (Microsoft reports a 99.9% reduction in account compromise for accounts protected by MFA). The owner approved, and I enabled Security Defaults at the tenant level.

This was the realistic baseline given the tenant's licensing. Conditional Access requires Entra ID P1, which this tenant did not have, so the longer-term recommendation was an upgrade to Microsoft 365 Business Premium for finer-grained policies (trusted-location bypass, phishing-resistant authentication strength, session controls).

### 11. Clean MFA registration for the affected user

After Security Defaults was enabled, I walked the affected user through a clean MFA registration:

- Sent the user to `https://aka.ms/mfasetup` from a clean device
- Pre-coached registration of Microsoft Authenticator (push with number matching) as primary
- Phone as backup only

## A Side Issue Surfaced During Cleanup

During the registration flow, the user hit an error when trying to open an encrypted message:

[![Troubleshooting details showing AADSTS500014 error: the service principal for resource 'https://aadrm.com' is disabled](/assets/post-compromise-remediation/05-aadsts500014-error.png)](/assets/post-compromise-remediation/05-aadsts500014-error.png){:target="_blank"}

This is the Microsoft Rights Management Services service principal being disabled at the tenant level, unrelated to the compromise but exposed by it. I re-enabled it via:

Entra admin center > Enterprise Applications > change the filter to **All Applications** (the default filter hides it) > search "Microsoft Rights Management Services" (App ID `00000012-0000-0000-c000-000000000000`) > Properties > Enabled for users to sign-in: Yes.

## Takeaways

1. **Password reset and session revoke are step one, not the whole job.** The attacker persistence layer lives below the password.
2. **Modern compromises hide in non-interactive logs.** AiTM token theft produces a clean interactive log because the user actually authenticated. The attacker's footprint shows up only in non-interactive sign-ins, where token replay traffic from commercial datacenter ASNs with `node` user agents stands out clearly.
3. **The cost of no MFA is not the breach moment, it is the silent occupancy.** This attacker was inside the mailbox for 14 days before detection, scraping data through the Graph API at roughly 100 requests per day.
4. **OAuth consents are the most-missed persistence vector.** This belongs as a permanent step in any incident runbook.
5. **Licensing dictates the hardening path.** Without Entra ID P1, Conditional Access is not an option. Security Defaults plus clean MFA registration is the realistic baseline for tenants on Exchange Online Plan 1 or similar.

## Tools and Surfaces Used

- Entra admin center (Authentication methods, Enterprise Applications, Sign-in logs, Properties)
- Exchange Online PowerShell (`Get-Mailbox`, `Get-InboxRule`, `Get-HostedOutboundSpamFilterPolicy`, `Remove-InboxRule`)
- Outlook on the Web (rule audit)
- Microsoft Authenticator (re-registration)
- Excel (cross-referencing exported sign-in logs)

[Back to home](/)
