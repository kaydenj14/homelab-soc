# Day 2: Group Policy Hardening

## Objective

Author and deploy a Group Policy Object (GPO) that enforces security controls on the HR Department OU — covering password policy, account lockout, audit logging, application restrictions, and security options — aligned with CIS Windows security benchmarks.

---

## Why This Matters

Group Policy is how organizations enforce security standards at scale. Without GPOs, every machine is configured manually and inconsistently. With GPOs, you define the control once, link it to an OU, and every machine and user in that container inherits it automatically — and keeps inheriting it even if someone tries to change settings locally.

This is also the foundation of security monitoring. Audit policies defined in GPO are what cause Windows to *write* security events to the Event Log in the first place. Without audit policy, there are no Event IDs 4625 or 4740 — and the SIEM has nothing to collect.

---

## GPO Created

**Name:** HR-Security-Policy  
**Linked to:** HR Department OU  
**Scope:** All users and computers in the HR Department OU

---

## Configuration Details

### Password Policy

| Setting | Value | Why |
|---|---|---|
| Minimum password length | 10 characters | Short passwords are trivially brute-forced offline |
| Password complexity | Enabled | Prevents passwords like `password123` — requires uppercase, lowercase, number, and symbol |
| Maximum password age | 90 days | Limits how long a stolen credential remains valid |
| Password history | 24 passwords | Prevents cycling back to old passwords after a forced reset |

**Why these numbers:** CIS Benchmark Level 1 for Windows recommends minimum 14 characters, but 10 is the floor that covers most brute force scenarios while remaining practical for users. The 90-day rotation is a balance — shorter windows increase administrative overhead, but static passwords create long exposure windows after a breach.

---

### Account Lockout Policy

| Setting | Value | Why |
|---|---|---|
| Lockout threshold | 5 failed attempts | Low enough to stop brute force, high enough to survive a typo |
| Lockout duration | 30 minutes | Slows down automated attacks without requiring admin intervention |
| Reset counter after | 30 minutes | Failed attempt counter resets, preventing perpetual lockout |

**Why this matters for detection:** Every lockout generates Event ID 4740 on the Domain Controller. This is a primary indicator of a brute force attack in progress. Combined with Event ID 4625 (individual failed logon), account lockout telemetry is how you detect and investigate password spray campaigns.

---

### Audit Policy

These settings control which events Windows writes to the Security Event Log. Each category maps to specific Event IDs that feed into the SIEM.

| Audit Category | Setting | Key Event IDs Generated |
|---|---|---|
| Logon Events | Success + Failure | 4624 (success), 4625 (failure), 4634 (logoff) |
| Account Management | Success + Failure | 4720 (account created), 4740 (lockout) |
| Object Access | Success + Failure | File and registry access events |
| Policy Change | Success + Failure | 4719 (policy changed) |
| Privilege Use | Success + Failure | 4672 (special privileges assigned) |
| Process Tracking | Success + Failure | 4688 (new process created) — catches command execution |

**Why auditing both success and failure:** Failure events catch attacks in progress. Success events catch attacks that succeeded. You need both. A SOC analyst who only sees failures misses successful intrusions; one who only sees successes misses the reconnaissance phase.

---

### Application Restrictions

| Setting | Value | Why |
|---|---|---|
| Command Prompt | Disabled | Prevents users from running `cmd.exe` — a common lateral movement tool |
| Registry Editor | Disabled | Blocks `regedit.exe` — prevents persistence via registry run keys |
| USB Autoplay | Disabled | Prevents malicious USB devices from auto-executing payloads |

**Why restrict command prompt for HR:** HR users have no legitimate need for a command line. Disabling it reduces the attack surface for malware that drops a payload and tries to execute it via `cmd.exe`. It also prevents an insider threat from running enumeration tools.

**What this doesn't stop:** A motivated attacker with admin rights or a custom payload that doesn't use `cmd.exe`. These restrictions are speed bumps, not walls — their value is in raising the cost and complexity of attacks, and in generating alerts when bypass attempts occur.

---

### Security Options

| Setting | Value | Why |
|---|---|---|
| Screen lock | 300 seconds (5 minutes) | Idle sessions are physical security risks |
| NTLM authentication | NTLMv2 only | NTLMv1 is cryptographically broken and vulnerable to relay attacks |
| Administrator account | Renamed | Default Administrator account is targeted by automated attacks |
| Last username | Hidden from login screen | Prevents username enumeration from the physical machine |
| Anonymous enumeration | Blocked | Prevents unauthenticated users from listing shares, users, or policies via null sessions |

**Why NTLMv2 specifically:** NTLMv1 uses DES encryption and can be cracked offline in seconds with modern hardware. NTLMv2 uses HMAC-MD5 which is harder (though not uncrackable). The right answer is Kerberos everywhere, but NTLMv2 enforcement ensures the weakest protocol is off the table.

**Why renaming Administrator:** Automated attack tools (and most penetration testing frameworks) attempt `Administrator` as a default username. Renaming it forces attackers to first enumerate valid usernames before they can attempt credential attacks.

---

## Policy Application

GPOs apply in a specific order: **Local → Site → Domain → OU** (LSDOU). OU-linked GPOs have the highest precedence. This means `HR-Security-Policy` overrides any conflicting settings from domain-level GPOs.

To apply the policy immediately on a target machine without waiting for the background refresh interval:

```cmd
gpupdate /force
```

To verify which policies are applied to a specific machine and confirm no conflicts:

```cmd
gpresult /r
```

To see the full applied policy in detail (useful for troubleshooting):

```cmd
gpresult /h C:\policy-report.html
```

---

## Verification

After linking the GPO and running `gpupdate /force` on WIN11-WS01:

1. **Password policy confirmed:** Attempted to set a short password on `jolaes` — rejected
2. **Lockout confirmed:** Failed logon 5 times with wrong password — account locked, Event ID 4740 generated on DC01
3. **Audit policy confirmed:** Failed logon attempts generated Event ID 4625 in DC01 Security Event Log
4. **CMD restriction confirmed:** Logged in as jolaes on WIN11-WS01 — Command Prompt blocked

---

## Key Concepts

**LSDOU precedence:** When multiple GPOs conflict, the last one to apply wins. OU GPOs apply last, so they have the final say. This is why linking the security GPO directly to the HR Department OU — rather than at the domain level — ensures it takes precedence.

**GPO vs local policy:** Local security policy (managed via `secpol.msc`) is the lowest precedence. Any domain GPO overrides it. This is why a user can't change their own screen lock timeout on a domain-joined machine — the GPO setting wins.

**Audit policy is the foundation of detection:** Every SIEM alert depends on the endpoint generating the event first. If audit policy doesn't enable the relevant category, the event never gets written, the agent never collects it, and the SIEM never sees it. Audit policy configuration is the first step in any detection engineering workflow.

**Why this links directly to SOC work:** In a real SOC, one of the first things an analyst does when investigating an alert is check whether the relevant audit categories were enabled at the time of the incident. If they weren't, there may be gaps in the forensic timeline. Understanding audit policy is understanding your own visibility limitations.
