# Day 1–2: Active Directory Domain Setup

## Objective

Build a functional enterprise-style Active Directory environment from scratch — domain controller, DNS, organizational units, and a tiered user structure — to serve as the foundation for all subsequent security lab work.

---

## Why This Matters

Active Directory is the identity backbone of almost every enterprise Windows environment. If you understand AD, you understand how organizations authenticate users, delegate authority, and control access to resources. It's also the primary target in most real-world attacks — compromising AD means compromising everything.

Building it from scratch (rather than using a pre-built VM) ensures every configuration is understood, not just inherited.

---

## Environment

| Machine | Role | IP |
|---|---|---|
| DC01 | Domain Controller, DNS Server | 192.168.1.10 |
| WIN11-WS01 | Domain Workstation | 192.168.1.20 |

**Network:** VirtualBox Internal Network — no external internet access, fully isolated  
**Domain:** homelab.local

---

## What Was Built

### 1. Domain Controller Promotion

Installed the Active Directory Domain Services (AD DS) role on Windows Server 2022, then promoted the server to a Domain Controller for a new forest: `homelab.local`.

**Why a new forest rather than joining an existing domain:** A forest represents a complete security boundary. Building a new one means building all the trust infrastructure from the ground up — this is what you'd do deploying AD for a new organization.

During promotion, DNS was installed automatically and configured to use DC01 as the authoritative resolver for `homelab.local`. All machines on the internal network point to `192.168.1.10` as their DNS server — this is how domain-joined machines resolve names like `dc01.homelab.local` and authenticate against the domain.

---

### 2. Organizational Unit Structure

Three OUs were created to reflect a tiered organizational structure:

```
homelab.local
├── IT Department
├── HR Department
└── Service Accounts
```

**Why OUs matter for security:** OUs aren't just folders. They're the containers that Group Policy links to. By separating users into OUs by department or function, you can apply different security policies to different groups — stricter controls on HR (sensitive data), tighter restrictions on service accounts (no interactive logon), etc. This is the principle of least privilege applied at the structural level.

---

### 3. User Accounts — Tiered Structure

Three accounts were created representing different access tiers found in real environments:

| Account | OU | Type | Purpose |
|---|---|---|---|
| jolaes | HR Department | Standard user | Normal employee — no admin rights |
| itadmin | IT Department | Admin account | IT administrator — elevated privileges |
| sqlservice | Service Accounts | Service account | Background service — no interactive logon |

**Why three tiers matter:**

- **Standard users** have access only to what they need. If a standard user account is compromised, the attacker's foothold is limited.
- **Admin accounts** are separate from day-to-day accounts. Admins should log in as standard users for normal tasks and only elevate when necessary. This is why `itadmin` is a *separate account* — not just jolaes with admin rights added.
- **Service accounts** are high-value targets. They often run with elevated privileges and may have weak or static passwords. Isolating them in their own OU allows separate policy enforcement and makes them easier to audit.

---

### 4. Domain Join — WIN11-WS01

Joined the Windows 11 workstation to the `homelab.local` domain by pointing its DNS to `192.168.1.10` and running the domain join wizard.

After joining, domain user accounts (jolaes, itadmin) can authenticate against this machine — credentials are validated by DC01, not stored locally.

**Why domain join matters for security monitoring:** Once a machine is domain-joined, authentication events flow through the Domain Controller. When jolaes logs into WIN11-WS01, DC01 records it. This is why the DC is the primary source of authentication telemetry — it sees every login across every domain-joined machine.

---

## Key Concepts

**Forest vs Domain vs OU:** A forest is the top-level security boundary. A domain is inside a forest. OUs are inside a domain. Trust relationships exist between domains and forests — not between OUs.

**DNS and AD are inseparable:** AD uses DNS to locate domain controllers (via SRV records). If DNS breaks, domain authentication breaks. This is why the first thing you configure on a DC is DNS, and why all domain clients must point to the DC as their DNS resolver.

**Kerberos vs NTLM:** Domain authentication in modern Windows uses Kerberos by default. A client requests a Ticket Granting Ticket (TGT) from the DC's KDC service. The TGT is used to request service tickets for specific resources. NTLM is the fallback protocol — weaker, and explicitly disabled in the GPO hardening phase.

---

## Commands Used

```powershell
# Install AD DS role (run on DC01 before promotion)
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to Domain Controller (new forest)
Install-ADDSForest -DomainName "homelab.local" -InstallDns

# Create Organizational Units
New-ADOrganizationalUnit -Name "IT Department" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "HR Department" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=homelab,DC=local"

# Create users
New-ADUser -Name "jolaes" -SamAccountName "jolaes" `
  -Path "OU=HR Department,DC=homelab,DC=local" -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force)

New-ADUser -Name "itadmin" -SamAccountName "itadmin" `
  -Path "OU=IT Department,DC=homelab,DC=local" -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "AdminPass456!" -AsPlainText -Force)

New-ADUser -Name "sqlservice" -SamAccountName "sqlservice" `
  -Path "OU=Service Accounts,DC=homelab,DC=local" -Enabled $true `
  -AccountPassword (ConvertTo-SecureString "ServicePass789!" -AsPlainText -Force)

# Verify domain controller status
Get-ADDomainController

# Verify users created correctly
Get-ADUser -Filter * -Properties * | Select-Object Name, DistinguishedName, Enabled
```

---

## Verification

After configuration, confirmed:
- `homelab.local` forest exists and DC01 holds all FSMO roles
- Three OUs visible in Active Directory Users and Computers (ADUC)
- All three user accounts created, enabled, and placed in correct OUs
- WIN11-WS01 appears in the Computers container in ADUC
- Domain logon works from WIN11-WS01 using `homelab\jolaes`

---

## What This Enables

This AD foundation is required for everything that follows:
- Group Policy applies to OUs — so policy enforcement requires the OU structure
- Wazuh agents register to the domain — so agent deployment requires the domain to exist
- Authentication attack simulation (password spraying, Kerberoasting) requires real domain users and a functional KDC

Without this foundation, none of the security monitoring or attack simulation phases work.
