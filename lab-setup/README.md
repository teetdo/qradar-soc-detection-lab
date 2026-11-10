# Lab Setup & Network Design

## Topology

| Host | IP | Role | Specs |
|---|---|---|---|
| Kali Linux | 192.168.20.10 | Attacker | 4 GB RAM |
| pfSense | WAN 192.168.20.1 / LAN 192.168.10.254 | Firewall / Router | 1 GB RAM |
| Ubuntu Victim | 192.168.10.20 | SSH target, auditd/rsyslog source | 4 GB RAM |
| Windows 10 Victim | 192.168.10.30 | RDP target, WinCollect/Sysmon source | — |
| QRadar CE | 192.168.10.103 | SIEM | 10 GB RAM, 4 vCPU, 250 GB disk, v7.3.3 |

Host machine: 16 GB RAM total — running all five VMs simultaneously drove host memory usage to 99% (see [Lessons Learned](../lessons-learned/troubleshooting-journal.md#host-resource-contention)). Only VMs required for the current task were kept powered on.

Download links for the hypervisor, VM images, firewall, and logging tools are
listed in the repository's [Download Resources](../README.md#download-resources)
section. The QRadar CE 7.3.3 OVA link is hosted on Dropbox rather than IBM; verify
the image before importing it and keep this legacy appliance on an isolated
network.

## Why a Segmented Network Instead of a Flat Lab

The attacker segment (192.168.20.0/24) and the victim segment (192.168.10.0/24) are separated by a real stateful firewall rather than sharing a single host-only switch. This was a deliberate choice: it forces every piece of attack traffic to cross an actual security boundary, closer to how traffic reaches an internal host in a real network, and it surfaces real firewall-layer failure modes (see the anti-spoofing incident in Lessons Learned) that a flat network would never expose.

## Log Collection Pipeline

| Source | Transport | Log Source Type on QRadar | Outcome |
|---|---|---|---|
| Ubuntu `/var/log/auth.log` | Syslog UDP 514 | Generic catch-all DSM | Parses payload only — no structured category |
| Ubuntu `auditd` via `/var/log/audit/audit.log` | rsyslog `imfile` → custom ruleset → `omfwd` UDP 514 | Dedicated Linux OS DSM (auto-detected) | Clean parsing — category, username correct |
| pfSense firewall log | Syslog UDP 514 | Netgate pfSense DSM (installed manually) | **Never received events** — root cause undetermined, see Lessons Learned |
| Windows Security Event Log + Sysmon | WinCollect 10.0.2 agent (standalone) | Microsoft Windows Security Event Log DSM | Clean parsing — Source IP and username correct on first attempt |

The contrast between the Linux syslog path (catch-all DSM, misattributed Source IP) and the Windows WinCollect path (dedicated agent, correct parsing from the start) is one of the clearest findings of this lab: **log collection method matters more than log source type** when it comes to field-level parsing accuracy.

## Build Order

1. Deploy pfSense, configure WAN/LAN interfaces, establish routing between segments.
2. Deploy QRadar CE, verify web console access.
3. Deploy Ubuntu and Windows victims, confirm Layer 3 connectivity from Kali through pfSense.
4. Configure Syslog forwarding from Ubuntu and pfSense to QRadar.
5. Install WinCollect on the Windows host, add Security + Sysmon local sources.
6. Build detection logic (see [detection-rules/](../detection-rules/)).
7. Run attack simulations, verify detections, perform response actions.
