# Wazuh Detection Pipeline

How a real attack becomes a SIEM alert — the complete path from attacker action to dashboard visibility.

---

## End-to-End Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 1 — ATTACK                                                      │
│                                                                       │
│  Attacker on WIN11-WS01 runs:                                        │
│  runas /user:homelab\jolaes cmd  (wrong password)                    │
│                                                                       │
│  WIN11-WS01 sends Kerberos AS-REQ to DC01 (192.168.1.10)            │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 2 — EVENT GENERATION (DC01)                                    │
│                                                                       │
│  DC01 validates credentials → fails (wrong password)                 │
│  Windows writes to Security Event Log:                               │
│                                                                       │
│  Event ID: 4625 — An account failed to log on                        │
│  subStatus: 0xc000006a (wrong password, valid username)              │
│  targetUserName: jolaes                                              │
│  workstationName: WIN11-WS01                                         │
│                                                                       │
│  ⚠ This only happens because Audit Logon Events is enabled in GPO   │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 3 — AGENT COLLECTION (Wazuh Agent on DC01)                    │
│                                                                       │
│  Wazuh Agent monitors Windows Event Log in real time                 │
│  Detects new Security event → reads and parses it                   │
│  Forwards raw event to Wazuh Manager at 192.168.1.30:1514           │
│  Transport: encrypted over TCP                                        │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 4 — RULE EVALUATION (Wazuh Manager)                           │
│                                                                       │
│  Manager receives event → evaluates against ruleset                  │
│  Matches Rule 60122: "Windows logon failure"                         │
│  Assigns: level 5, description, event fields                         │
│                                                                       │
│  [Day 4] Custom Rule 100002 also evaluates:                         │
│  If Rule 60122 fires 3+ times for same user within 60 seconds →     │
│  Fire level 12 alert: "Brute force attack detected"                 │
│  Tag: MITRE ATT&CK T1110.001 — Brute Force: Password Guessing       │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 5 — STORAGE (Wazuh Indexer)                                   │
│                                                                       │
│  Alert written to OpenSearch index                                   │
│  All fields indexed for fast search                                  │
│  Retained for 90 days by default                                     │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 6 — ANALYST VISIBILITY (Wazuh Dashboard)                      │
│                                                                       │
│  SOC Analyst opens https://192.168.1.30                             │
│  Queries Discover: rule.id:60122                                     │
│  Sees alert with: agent.name=DC01, targetUserName=jolaes,           │
│  subStatus=0xc000006a, workstationName=WIN11-WS01                   │
│                                                                       │
│  Time from attack to alert visible: ~10 seconds                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## The Dependency Chain

Every layer depends on the one below it. If any layer breaks, everything above it goes blind:

| Layer | Dependency | What breaks if missing |
|---|---|---|
| GPO Audit Policy | Must be configured on DC01 | Windows never writes the event — nothing to collect |
| Windows Event Log | Must have space and be accessible | Agent has nothing to read |
| Wazuh Agent | Must be installed, running, and connected | Events never leave the endpoint |
| Network (TCP 1514) | Firewall must allow outbound to 192.168.1.30 | Events collected but never forwarded |
| Wazuh Manager | Must be running | Events arrive but aren't processed |
| Wazuh Indexer | Must be running | Alerts processed but not stored |
| Wazuh Dashboard | Must be running | Alerts stored but not visible in UI |

**Troubleshooting principle:** Work bottom-up. If alerts aren't showing in the dashboard, first confirm the event was written to the Windows Event Log, then confirm the agent is forwarding, then confirm the manager processed it.

---

## Why the Event Is on DC01, Not WIN11-WS01

This is a common point of confusion. When jolaes tries to log in from WIN11-WS01 using a wrong password:

1. WIN11-WS01 sends an authentication request to DC01 (Kerberos AS-REQ)
2. DC01 validates (or rejects) the credentials
3. DC01 records the result in *its own* Security Event Log

The workstation (WIN11-WS01) records a *client-side* logon event, but the authoritative authentication event — the one that shows whether the password was right or wrong — lives on the Domain Controller.

This means DC01 is the primary source of authentication telemetry for the entire domain. Every domain logon, regardless of which machine it came from, is recorded on DC01. The `workstationName` field in the 4625 event tells you *where the attempt came from*, but the event itself lives on the DC.

---

## Wazuh Ports Reference

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 1514 | TCP | Agent → Manager | Log forwarding (ongoing) |
| 1515 | TCP | Agent → Manager | Agent registration (one-time) |
| 443/9200 | TCP | Dashboard → Indexer | Dashboard queries OpenSearch |
| 443 | TCP | Browser → Dashboard | Analyst accesses UI |

---

## Key Event IDs in This Lab

| Event ID | Description | Generated By | SIEM Relevance |
|---|---|---|---|
| 4624 | Successful logon | DC01 | Baseline; useful for anomaly detection |
| 4625 | Failed logon | DC01 | Primary brute force / spray indicator |
| 4634 | Logoff | DC01 | Session duration tracking |
| 4740 | Account lockout | DC01 | Brute force confirmation |
| 4768 | Kerberos TGT request | DC01 | Normal auth; anomalies indicate attacks |
| 4769 | Kerberos service ticket | DC01 | Kerberoasting indicator (Day 4) |
| 4688 | Process created | Workstations | Lateral movement, malware execution |
