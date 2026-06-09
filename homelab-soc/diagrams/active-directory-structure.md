# Active Directory Structure

## Domain: homelab.local

```
homelab.local (Forest Root Domain)
│
├── [Domain Controllers]
│   └── DC01.homelab.local (Windows Server 2022)
│       ├── FSMO Roles: Schema Master, Domain Naming, RID, PDC, Infrastructure
│       └── DNS: Authoritative for homelab.local
│
├── OU: IT Department
│   └── 👤 itadmin
│       ├── Type: Admin account
│       ├── Member of: Domain Admins
│       └── Purpose: IT administrative tasks
│
├── OU: HR Department  ← GPO: HR-Security-Policy applied here
│   └── 👤 jolaes
│       ├── Type: Standard user
│       ├── Member of: Domain Users
│       └── Purpose: Normal employee simulation, attack target
│
├── OU: Service Accounts
│   └── 👤 sqlservice
│       ├── Type: Service account
│       ├── SPN: registered (Kerberoasting target — Day 4)
│       └── Purpose: Background service simulation
│
└── [Computers]
    └── WIN11-WS01 (Windows 11 Enterprise, 192.168.1.20)
```

---

## User Account Tiers

### Tier 1 — Standard Users
`jolaes` — Normal employee account. No administrative rights. Subject to all HR Department GPO restrictions. Primary account used for attack simulations (password spray target).

### Tier 2 — Admin Accounts
`itadmin` — Separate administrative account. Elevated privileges for IT management tasks. **Not used for day-to-day tasks** — this separation is the principle of least privilege in practice. If jolaes's account is compromised, the attacker doesn't automatically get admin access.

### Tier 3 — Service Accounts
`sqlservice` — Non-interactive service account. Registered with a Service Principal Name (SPN) to support Kerberos service ticket requests. This makes it a target for Kerberoasting attacks (Day 4 — Impacket GetUserSPNs). Service accounts often run with elevated privileges and may have long-lived static passwords — making them high-value targets.

---

## GPO: HR-Security-Policy

Linked to the HR Department OU. Applies to all users and computers in that container.

| Category | Key Settings |
|---|---|
| Password Policy | Min 10 chars, complexity, 90-day max age, 24 history |
| Account Lockout | 5 attempts, 30-minute lockout, 30-minute reset |
| Audit Policy | Success + Failure on Logon, Account Mgmt, Object Access, Policy Change, Privilege Use, Process Tracking |
| App Restrictions | CMD disabled, Registry Editor disabled, USB Autoplay disabled |
| Security Options | Screen lock 300s, NTLMv2 only, Admin renamed, Last username hidden, Anonymous enum blocked |

**GPO policy application order:** Local → Site → Domain → **OU** (HR-Security-Policy)  
OU-linked GPOs have highest precedence — settings here override any conflicting domain-level policy.

---

## Why This Structure Enables Security Monitoring

The OU structure is what makes targeted GPO application possible. By placing jolaes in the HR OU:
- The audit policy GPO applies — Windows writes Event IDs 4625, 4740, etc. to the Security Event Log
- The Wazuh agent on DC01 picks up those events and forwards them to the SIEM
- The SIEM can alert on authentication failures against that account

Without the OU structure and the linked audit policy, there would be no events to monitor.
