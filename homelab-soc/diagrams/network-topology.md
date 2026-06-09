# Network Topology

The lab runs entirely on a VirtualBox isolated internal network (`intnet`). No machine has direct internet access during normal operation. A NAT adapter was temporarily added to Ubuntu Server and DC01 during software installation, then removed.

```
                        ┌─────────────────────────────────────────┐
                        │     VirtualBox Internal Network (intnet) │
                        │              192.168.1.0/24              │
                        │                                          │
                        │  ┌─────────────┐   ┌─────────────────┐  │
                        │  │    DC01     │   │   WIN11-WS01    │  │
                        │  │ 192.168.1.10│   │  192.168.1.20   │  │
                        │  │             │   │                 │  │
                        │  │ Windows     │   │  Windows 11     │  │
                        │  │ Server 2022 │   │  Enterprise     │  │
                        │  │             │   │                 │  │
                        │  │ Roles:      │   │  Roles:         │  │
                        │  │ - AD DS     │   │  - Domain WS    │  │
                        │  │ - DNS       │   │  - Wazuh Agent  │  │
                        │  │ - Wazuh     │   │                 │  │
                        │  │   Agent     │   └─────────────────┘  │
                        │  └─────────────┘                        │
                        │                                          │
                        │        ┌──────────────────┐             │
                        │        │   Ubuntu Server  │             │
                        │        │   192.168.1.30   │             │
                        │        │                  │             │
                        │        │   Ubuntu 24.04   │             │
                        │        │                  │             │
                        │        │   Roles:         │             │
                        │        │   - Wazuh Mgr    │             │
                        │        │   - Wazuh Idx    │             │
                        │        │   - Wazuh Dash   │             │
                        │        └──────────────────┘             │
                        └─────────────────────────────────────────┘
```

## Machine Roles

| Machine | OS | IP | CPU | RAM | Storage |
|---|---|---|---|---|---|
| DC01 | Windows Server 2022 | 192.168.1.10 | 2 vCPU | 2 GB | 60 GB |
| WIN11-WS01 | Windows 11 Enterprise | 192.168.1.20 | 2 vCPU | 4 GB | 60 GB |
| Ubuntu Server | Ubuntu 24.04 LTS | 192.168.1.30 | 2 vCPU | 4 GB | 100 GB |

**Host machine:** Windows desktop, 16 GB RAM, 900 GB free, VirtualBox 7.28

## Network Design Decisions

**Why isolated internal network:** Prevents the attack simulation from affecting any real network. Kali Linux attacks in Day 4 will be fully contained within the lab. This is standard practice for any security lab — you never run attack tools on a shared network.

**Why static IPs:** VirtualBox's internal network has no DHCP server. Even if it did, static IPs are required for server infrastructure — agents are configured to report to the manager by IP, and DNS zones are configured on DC01 by IP. Dynamic addresses would break these dependencies every time a machine reboots.

**Why DNS points to DC01:** All domain-joined machines and the Ubuntu server point to `192.168.1.10` for DNS. DC01 hosts the authoritative DNS zone for `homelab.local`. This is required for Kerberos authentication, domain join, and name resolution across the environment.
