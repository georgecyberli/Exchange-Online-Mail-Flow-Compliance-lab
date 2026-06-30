# Microsoft Exchange Online — Mail Flow, Security & Compliance Lab

**Topics:** Exchange Online, Mail Flow Rules, Message Trace, SPF/DKIM/DMARC, Defender for Office 365, Microsoft Purview, eDiscovery, Mailbox Administration

![Cloud](https://img.shields.io/badge/Cloud-Microsoft%20365-0078D4?style=flat-square)
![Focus](https://img.shields.io/badge/Focus-Mail%20Flow%20%26%20Security-red?style=flat-square)
![Focus](https://img.shields.io/badge/Focus-Compliance-green?style=flat-square)
![Practice](https://img.shields.io/badge/Practice-Email%20Authentication-blue?style=flat-square)
![Output](https://img.shields.io/badge/Output-Runbooks-grey?style=flat-square)

A hands-on lab series demonstrating enterprise email administration using Microsoft Exchange Online, built as part of a self-directed sysadmin training program. Each lab simulates real-world scenarios found in healthcare and enterprise IT environments, progressing from foundational mailbox management through mail flow security, compliance, and admin operations.

**Environment:** Microsoft 365 E5 Trial | Exchange Admin Center | Microsoft Defender for Office 365 | Microsoft Purview, fully isolated from production

---

## Overview

| # | Lab Series | Skills Demonstrated |
|---|-----|----------------------|
| 1 | [Foundations](#phase-1--foundations) | Mailbox types, delegation, groups, quotas, archiving |
| 2 | [Mail Flow](#phase-2--mail-flow) | Transport rules, message trace, anti-spam, SPF/DKIM/DMARC, forwarding |
| 3 | [Security & Compliance](#phase-3--security--compliance) | Anti-phishing, Safe Links/Attachments, retention, eDiscovery |
| 4 | [Admin Operations](#phase-4--admin-operations) | Bulk delegation auditing, OOF automation, migration, audit logs |

---

## Phase 1 — Foundations

**Objective:** Establish the foundational mailbox and permission structures every Exchange environment is built on, before introducing mail flow logic or security controls.

### What I Built

- Created all four Exchange mailbox types — user, shared, room, and equipment — through both the modern Exchange Admin Center wizard and PowerShell
- Configured Full Access, Send As, and Send on Behalf permissions on a shared mailbox, then deliberately tested the boundary between them by attempting to open a mailbox with Send As alone
- Built Distribution Groups, Mail-Enabled Security Groups, and Microsoft 365 Groups, validating each through the distinct PowerShell cmdlets required to query them (`Get-DistributionGroup` vs `Get-UnifiedGroup`)
- Set custom storage quotas on both a user mailbox and a shared mailbox, discovering that E5 licensing ships with significantly higher default quotas (98/99/100 GB) than the documented 50 GB baseline
- Enabled an archive mailbox and applied Litigation Hold, then hit and documented a genuine licensing wall — Litigation Hold on a shared mailbox requires Exchange Online Plan 2

### Key Concepts

- Full Access and Send As are independent permissions — Full Access does not grant the ability to send as the mailbox, and Send As does not grant the ability to open it; OWA in particular requires Full Access before a Send As permission can be exercised through the interface
- Litigation Hold on shared mailboxes is licensing-gated in a way that user mailboxes are not, which directly motivated the Purview eDiscovery Hold approach used later in Phase 3
- Quota enforcement requires explicitly setting `UseDatabaseQuotaDefaults $false` — without it, Exchange silently ignores any custom quota values applied

### Skills Demonstrated

`Mailbox Provisioning` `Permission Delegation` `Distribution & M365 Groups` `Storage Quota Management` `Litigation Hold`

---

## Phase 2 — Mail Flow

**Objective:** Build and validate the mail flow control layer — transport rules, delivery troubleshooting, spam filtering, and email authentication — that governs how messages move through the organization.

### What I Built

- Created three transport rules covering attachment blocking, disclaimer injection, and keyword-based redirection, then validated all three using real test emails, including an actual `.exe` attachment that triggered a genuine NDR
- Investigated every test message using both the EAC message trace GUI and the PowerShell `Get-MessageTraceV2`/`Get-MessageTraceDetailV2` cmdlets, reading the full event timeline (Receive → Transport rule → Deliver/Fail) for each
- Built a custom anti-spam policy with a tightened bulk email threshold and explicit allowed/blocked sender entries, then compared it directly against the tenant default policy via PowerShell
- Verified SPF, DKIM, and DMARC end-to-end — generated and enabled DKIM signing keys, confirmed the SPF hard-fail record, discovered no DMARC record exists by default on a `.onmicrosoft.com` domain, then validated the full authentication chain (`spf=pass`, `dkim=pass`, `dmarc=bestguesspass`) by inspecting real headers on an external test email
- Configured both mailbox-level forwarding (with copy retention) and conditional mail-flow-rule-based forwarding, confirming the security warning Exchange surfaces about external auto-forwarding as a data exfiltration vector

### Key Concepts

- Multiple transport rules can fire on the same message in priority order — a single email was observed receiving both a disclaimer and a redirect action sequentially, confirmed in the message trace event log
- `Get-MessageTrace` and `Get-MessageTraceDetail` are deprecated; the V2 cmdlets are required from now on, and discovering this via a live deprecation warning is itself a realistic troubleshooting moment
- DKIM only signs outbound mail to external recipients — internal-to-internal messages within the same tenant correctly show `dkim=none`, which is expected behavior, not a misconfiguration
- Auto-forwarding to external addresses is blocked by Exchange Online's default outbound spam policy specifically because it is a known account-takeover and data-exfiltration pattern

### Skills Demonstrated

`Transport Rules` `Message Trace` `Anti-Spam Policy Tuning` `SPF / DKIM / DMARC` `Mailbox & Rule-Based Forwarding`

---

## Phase 3 — Security & Compliance

**Objective:** Layer Microsoft Defender for Office 365 and Microsoft Purview controls on top of the existing mail flow foundation to protect against impersonation, malicious content, and to support legal/compliance discovery workflows.

### What I Built

- Configured a custom anti-phishing policy with user and domain impersonation protection, mailbox intelligence, and spoof intelligence, correctly distinguishing the policy's scope (who it applies to) from its protection targets (who it defends against impersonation)
- Built Safe Attachments and Safe Links policies, including discovering and explaining why a test malicious attachment bypassed detection when OWA silently converted it to a OneDrive link instead of a true attachment
- Created two retention policies via Microsoft Purview — an org-wide 7-year Exchange policy and a precisely-scoped 5-year policy targeting a single shared mailbox — confirming both via `Connect-IPPSSession` and `Get-RetentionCompliancePolicy`
- Ran a Microsoft Purview Content Search across two mailboxes and recovered the exact keyword-flagged email from Phase 2's transport rule lab, found independently in two locations due to the forwarding configuration set up earlier
- Resolved the Phase 1 Litigation Hold licensing limitation by creating a full eDiscovery case and applying a Purview eDiscovery Hold to the same shared mailbox — achieving equivalent legal preservation without the Exchange Online Plan 2 requirement

### Key Concepts

- Anti-phishing impersonation protection requires understanding two separate scope dimensions in the policy wizard — "who is covered by this policy" and "who attackers might impersonate" are configured on different screens and are easy to conflate
- Safe Links can operate in API-only checking mode without visually rewriting URLs, providing identical protection with a cleaner user experience — confirmed by testing that protection remained active even when no URL rewrite was visible in the message
- Purview eDiscovery Hold is the practical, license-friendly alternative to Exchange Litigation Hold for shared mailboxes, and ties directly into a full case/search/hold/export discovery workflow rather than being an isolated mailbox setting

### Skills Demonstrated

`Anti-Phishing Policy` `Safe Links & Safe Attachments` `Microsoft Purview Retention` `Content Search` `eDiscovery Case Management`

---

## Phase 4 — Admin Operations

**Objective:** Apply the prior three phases of foundational, mail flow, and security knowledge to day-to-day admin operations — bulk auditing, lifecycle-driven mailbox actions, and the platform limitations that surface during real trial-tenant administration.

### What I Built

- Audited shared mailbox permissions across the entire tenant in a single PowerShell pipeline rather than checking mailboxes individually, then performed a full onboard/verify/offboard/verify delegation cycle on a newly created shared mailbox
- Investigated a genuine cloud-only Exchange Online platform limitation when attempting to create a custom Email Address Policy with recipient filtering — diagnosed the actual supported parameter set via `Get-Command -Syntax` rather than guessing, and documented why this on-premises/hybrid capability isn't fully exposed in a cloud-only tenant
- Configured admin-initiated Out-of-Office replies on a user's behalf, including both immediate and scheduled auto-reply states, validating the external-audience reply against a real personal email account
- Walked the full mailbox migration batch wizard across all five available migration types (Remote move, Cutover, Cross tenant, Google Workspace, IMAP) and documented the complete Cutover migration prerequisite checklist
- Hit a real Unified Audit Log search failure tied to expiring trial compliance licensing, and used it to demonstrate the distinction between mailbox audit *logging* (confirmed still active via `AuditEnabled`) and audit log *search* (confirmed blocked independently)

### Key Concepts

- Bulk operations require referencing `PrimarySmtpAddress` rather than `Identity` inside a loop — ambiguous display names across object types will otherwise throw "doesn't represent a unique recipient" errors at scale
- Not every Exchange Online cmdlet exposes full functionality in a cloud-only tenant; `Get-Command -Syntax` is the correct diagnostic step before assuming a syntax error when a cmdlet's parameter set behaves unexpectedly
- Audit logging and audit log search are independently failing dependencies — a tenant can be actively recording every mailbox access event while simultaneously being unable to search or retrieve that data due to a licensing gap

### Skills Demonstrated

`Bulk Permission Auditing` `Platform Limitation Diagnosis` `Admin-Initiated Mailbox Actions` `Migration Planning` `Audit Log Architecture`

---

## Tech Stack

`Exchange Online` `Exchange Admin Center` `Microsoft Defender for Office 365` `Microsoft Purview` `Exchange Online PowerShell` `SPF / DKIM / DMARC` `eDiscovery` `Connect-IPPSSession`
