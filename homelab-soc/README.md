# Enterprise SOC Home Lab

A progressively built security operations lab simulating a real corporate IT environment. Covers Active Directory administration, Group Policy hardening, and SIEM deployment with live attack detection.

Built from scratch with no prior IT or cybersecurity experience. Every configuration decision is documented with the reasoning behind it — not just the steps.

---

## Lab Architecture

| Machine | OS | IP | Role |
|---|---|---|---|
| DC01 | Windows Server 2022 | 192.168.1.10 | Domain Controller, DNS |
| WIN11-WS01 | Windows 11 Enterprise | 192.168.1.20 | Domain-joined workstation |
| Ubuntu Server | Ubuntu 24.04 LTS | 192.168.1.30 | Wazuh SIEM (all-in-one) |

**Hypervisor:** VirtualBox 7 — all machines on an isolated internal network (intnet)  
**Domain:** homelab.local

---

## What's Been Built

### [Day 1–2 — Active Directory & Domain Setup](./day-01-active-directory/)

Deployed and promoted a Domain Controller from scratch. Configured DNS, created a tiered user structure (standard users, admin accounts, service accounts), and organized them into Organizational Units following least-privilege principles.

**Key skills demonstrated:** AD DS installation, domain promotion, OU structure, user tiering, DNS configuration

---

### [Day 2 — Group Policy Hardening](./day-02-group-policy/)

Authored GPOs aligned with CIS Windows security benchmarks. Covered password policy, account lockout, audit logging, application restrictions, and security options. Tested policy application via `gpupdate` and verified enforcement on domain workstations.

**Key skills demonstrated:** GPO authoring, CIS benchmark alignment, audit policy configuration, LSDOU policy precedence

---

### [Day 3 — Wazuh SIEM Deployment & Detection Validation](./day-03-wazuh-siem/)

Deployed Wazuh 4.11 all-in-one on Ubuntu Server. Installed agents on both Windows machines. Simulated a password spray attack and traced the detection end-to-end — from audit policy on the endpoint, through the agent, to alert in the SIEM dashboard.

**Key skills demonstrated:** SIEM deployment, agent installation, log pipeline architecture, Event ID analysis, attack simulation and detection validation

---

## Upcoming

| Phase | Description |
|---|---|
| Day 4 — Custom Detection Rules | Write frequency-based Wazuh rules targeting brute force patterns |
| Day 4 — Active Response | Automate IP blocking when brute force threshold is crossed |
| Day 4 — Kali Linux Attack Simulation | Run Hydra, Impacket Kerberoasting, CrackMapExec with full SIEM visibility |
| Day 4 — MITRE ATT&CK Mapping | Map every alert to ATT&CK tactics and techniques |
| Day 5 — Vulnerability Scanning | Run OpenVAS against the lab environment, interpret CVE output |

---

## Diagrams

- [Network Topology](./diagrams/network-topology.md)
- [Active Directory Structure](./diagrams/active-directory-structure.md)
- [Wazuh Detection Pipeline](./diagrams/wazuh-detection-pipeline.md)

---

## Technical Skills Used

Active Directory · Windows Server 2022 · Group Policy · Wazuh SIEM · Ubuntu Server 24.04 · VirtualBox · Linux CLI · Windows PowerShell · Netplan · Event ID analysis

---

*Kayden Java — SDSU Computer Science, B.S. Expected 2029 | kjava9419@sdsu.edu*
