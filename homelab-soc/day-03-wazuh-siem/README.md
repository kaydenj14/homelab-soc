# Day 3: Wazuh SIEM Deployment & Detection Validation

## Objective

Deploy a fully functional SIEM on Ubuntu Server, install agents on all domain endpoints, simulate a real attack (password spraying), and validate the complete detection pipeline from endpoint event generation through to SIEM alert.

---

## Why This Matters

Building an environment is one thing. Being able to *see what's happening in it* is another. A SIEM (Security Information and Event Management) system is how security teams get centralized visibility across an entire infrastructure. Without a SIEM, each machine is an island — you'd have to log into every endpoint individually to check its logs.

Wazuh specifically is used in real production environments and is a common platform in SOC analyst job descriptions. Understanding how to deploy it, configure it, and write queries against it is a directly transferable skill.

---

## Environment

| Machine | Role | IP |
|---|---|---|
| Ubuntu Server 24.04 LTS | Wazuh SIEM — all-in-one deployment | 192.168.1.30 |
| DC01 (Windows Server 2022) | Monitored endpoint — Agent 001 | 192.168.1.10 |
| WIN11-WS01 (Windows 11) | Monitored endpoint — Agent 002 | 192.168.1.20 |

---

## Architecture: Three Components in One

Wazuh's all-in-one deployment runs three distinct services on a single machine:

**Wazuh Manager**  
The brain of the system. Receives raw logs from agents, evaluates them against the ruleset, and generates alerts when rules match. Runs on TCP port 1514 (agent communication) and 1515 (agent registration).

**Wazuh Indexer**  
An OpenSearch (open-source Elasticsearch fork) database. Every event that arrives at the Manager gets stored here with full field indexing, making it searchable in milliseconds across millions of events.

**Wazuh Dashboard**  
A web interface (Kibana fork) running at `https://192.168.1.30`. Provides the Discover search interface, pre-built dashboards for compliance and security events, and alert management.

**Why understand the components separately:** When something breaks (and it will), you need to know which component to troubleshoot. If events aren't appearing in the dashboard, the problem could be the agent not sending, the manager not processing, the indexer not storing, or the dashboard not querying. Each has its own logs and health checks.

---

## Deployment

### 1. Ubuntu Server Configuration

Before deploying Wazuh, the Ubuntu Server needed a static IP configured via Netplan. VirtualBox's internal network doesn't have DHCP, so a static address was required.

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 192.168.1.30/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.10]
```

Nameserver points to DC01 (`192.168.1.10`) so the Ubuntu server can resolve `homelab.local` domain names.

```bash
sudo netplan apply
```

**Why static IP is required:** Agents are configured to report to the manager by IP address. If the manager's IP changes (as it would with DHCP), all agents lose connectivity. Static IPs are standard practice for any server that other machines depend on.

---

### 2. Wazuh All-in-One Installation

```bash
# Download and run the official install script
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

The `-a` flag installs all three components: manager, indexer, and dashboard.

Installation takes approximately 10–15 minutes. The script configures TLS certificates for all internal communication between components, creates the initial admin credentials, and starts all three services.

Post-install, the dashboard credentials are printed to the terminal — save these immediately.

---

### 3. Service Verification

```bash
# Verify all three components are running
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# Check manager is listening for agent connections
sudo ss -tlnp | grep -E '1514|1515'

# Confirm manager log is clean
sudo tail -50 /var/ossec/logs/ossec.log
```

---

### 4. Agent Deployment — DC01

Agents were installed silently on Windows machines using the MSI installer and PowerShell.

```powershell
# Download Wazuh agent MSI (run on DC01)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.11.0-1.msi" `
  -OutFile "wazuh-agent.msi"

# Silent install, pointing agent at the manager
msiexec /i wazuh-agent.msi /q `
  WAZUH_MANAGER="192.168.1.30" `
  WAZUH_AGENT_NAME="DC01"

# Start the agent service
Start-Service -Name "WazuhSvc"
Get-Service -Name "WazuhSvc"
```

Repeat with `WAZUH_AGENT_NAME="WIN11-WS01"` for the workstation.

After starting the agent, it registers with the manager and appears in the Agents section of the dashboard within ~30 seconds.

---

### Active Agents After Deployment

| Agent ID | Name | IP | OS | Status |
|---|---|---|---|---|
| 001 | DC01 | 192.168.1.10 | Windows Server 2022 | Active |
| 002 | WIN11-WS01 | 192.168.1.20 | Windows 11 Enterprise | Active |

---

## Detection Validation: Password Spray Simulation

### The Attack

A password spray attack attempts one password against many accounts — or in this case, multiple wrong passwords against one account. The goal is to generate Event ID 4625 (failed logon) events on the Domain Controller and verify they appear in Wazuh.

**Why run this on WIN11-WS01:** The `runas` command attempts domain authentication, which means the authentication request goes to DC01's KDC. DC01 records the failure. This simulates what a real attacker would do from a compromised workstation.

```cmd
# Run on WIN11-WS01 — enter wrong password each time when prompted
runas /user:homelab\jolaes cmd
runas /user:homelab\jolaes cmd
runas /user:homelab\jolaes cmd
```

After 5 failures, the account lockout policy triggers and Event ID 4740 (account locked out) is generated on DC01.

---

### The Detection Pipeline

The path from attack to SIEM alert:

```
Attack on WIN11-WS01
    ↓
Authentication request sent to DC01 via Kerberos
    ↓
DC01 records Event ID 4625 in Windows Security Event Log
(because Audit Logon Events policy was enabled in GPO)
    ↓
Wazuh Agent (DC01) reads new event from Windows Event Log
    ↓
Agent forwards event to Wazuh Manager (192.168.1.30:1514)
    ↓
Manager evaluates event against ruleset → Rule 60122 fires
    ↓
Alert stored in Wazuh Indexer (OpenSearch)
    ↓
Alert visible in Wazuh Dashboard → Discover
```

---

### Querying in Wazuh Discover

```
# DQL query to find all failed logon events
rule.id:60122

# Or filter by Event ID directly
data.win.system.eventID:4625

# Filter to a specific target user
data.win.eventdata.targetUserName:jolaes
```

---

### Key Fields in a 4625 Alert

| Field | Value Observed | What It Means |
|---|---|---|
| `agent.name` | DC01 | The agent that collected this event |
| `data.win.eventdata.targetUserName` | jolaes | The account the attacker targeted |
| `data.win.eventdata.subStatus` | 0xc000006a | Wrong password (not a non-existent account) |
| `data.win.eventdata.workstationName` | WIN11-WS01 | The machine the attempt came from |
| `data.win.system.eventID` | 4625 | Windows event type — failed logon |
| `rule.level` | 5 | Wazuh severity — medium |

**Why subStatus matters:** The sub-status code tells you *why* the logon failed. `0xc000006a` means the password was wrong — the username exists. `0xc0000064` means the username doesn't exist. An attacker seeing `0xc000006a` knows they have a valid username and can now focus their brute force.

**Why the event is on DC01, not WIN11-WS01:** Kerberos authentication is centralized. The workstation initiates the auth request, but the Domain Controller validates credentials and records the result. This is why DC01 is the primary source of authentication telemetry — it sees every domain logon, regardless of which machine it originated from.

---

## Key Concepts

**Why event correlation matters:** A single Event ID 4625 is noise. A hundred 4625s against the same account in 60 seconds is a brute force attack. The SIEM's job is to connect individual events into a pattern. This is what custom detection rules (Day 4) are built to do.

**Blind spots:** Machines without Wazuh agents are invisible to the SIEM. A standalone Windows machine not joined to the domain — or one with the agent uninstalled — could be compromised with no alerts generated. Agent coverage is a core part of SIEM administration.

**Alert triage vs raw event analysis:** The Wazuh Dashboard shows pre-processed alerts (events that matched a rule). The Discover interface shows raw events. SOC analysts work in both — alerts for initial triage, raw events for deep-dive investigation.

**The audit policy → SIEM dependency:** The Wazuh agent can only collect events that Windows has already written to the Event Log. Windows only writes events for auditing categories that are enabled in Group Policy. This is why audit policy configuration (Day 2) is the prerequisite for SIEM visibility. The chain is: `GPO audit policy → Event Log entry → Agent collection → SIEM alert`.

---

## Troubleshooting Encountered

### Issue 1: Netplan conflict on Ubuntu Server
**Symptom:** Static IP not applying; `netplan apply` failed with a conflict error  
**Cause:** Two netplan config files existed in `/etc/netplan/` — the installer's default and a second one created during setup  
**Fix:** Removed the duplicate file, left only `00-installer-config.yaml` with the static IP configuration  
**Lesson:** Always check `ls /etc/netplan/` before editing — multiple files can silently conflict

### Issue 2: Agent showing as disconnected
**Symptom:** WIN11-WS01 agent registered but showed as "Never connected" in dashboard  
**Cause:** Windows Firewall was blocking outbound TCP 1514 to the Wazuh manager  
**Fix:** Added a Windows Firewall outbound rule allowing TCP 1514 to `192.168.1.30`  
**Lesson:** Agent registration (port 1515) and agent communication (port 1514) are separate. Registration can succeed while communication is blocked.

### Issue 3: Dashboard not accessible via HTTPS
**Symptom:** Browser returned connection refused on `https://192.168.1.30`  
**Cause:** Wazuh Dashboard service wasn't fully started — the indexer takes time to initialize, and the dashboard won't start until the indexer is healthy  
**Fix:** Waited 2–3 minutes after `wazuh-install.sh` completed, then verified all three services with `systemctl status`  
**Lesson:** The three components have a startup dependency order. Wait for the indexer to be healthy before expecting the dashboard to work.
