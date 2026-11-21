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
| pfSense firewall log | Syslog UDP 514 | Netgate pfSense DSM (trained in DSM Editor) | **Verified:** live `filterlog` payload now parses and Destination Port is populated |
| Windows Security Event Log + Sysmon | WinCollect 10.0.2 agent (standalone) | Microsoft Windows Security Event Log DSM | Security events and username/IP parse correctly; Sysmon remains in raw-payload form |

The contrast between the Linux syslog path (catch-all DSM, misattributed Source IP) and the Windows WinCollect path (dedicated agent, correct parsing from the start) is one of the clearest findings of this lab: **log collection method matters more than log source type** when it comes to field-level parsing accuracy.

## pfSense Remote Logging and Netgate DSM

### 1. Configure remote syslog on pfSense

1. In pfSense, open **Status → System Logs → Settings**.
2. Enable **Remote Logging** and set the remote server to `192.168.10.103:514`.
3. Enable `Firewall Events` forwarding.
4. Add a WAN rule to allow Syslog from the pfSense LAN interface to QRadar where
   required by the lab's firewall policy.
5. Generate test traffic and confirm events contain the pfSense `filterlog`
   payload format rather than a generic or empty message.

This lab uses UDP 514. UDP is connectionless, so event receipt must be
confirmed in QRadar rather than assumed from a successful sender restart.

### 2. Install the Netgate pfSense RPMs

Obtain the `DSM-DSMCommon` and `DSM-NetgatePfSense` packages from
[IBM Fix Central](https://www.ibm.com/support/fixcentral?lc=en&lot=153). Select
the package set compatible with QRadar CE 7.3.3 and the Netgate pfSense
firewall version used in the lab. IBM entitlement or support access may be
required.

Copy both IBM DSM packages to the QRadar appliance and install them in the
documented dependency order:

```bash
rpm -Uvh DSM-DSMCommon-*.noarch.rpm
rpm -Uvh DSM-NetgatePfSense-*.noarch.rpm
```

After installation, open **Admin → Deploy Changes**. Verify the Netgate pfSense
DSM is available before creating or editing the firewall Log Source.

### 3. Create the firewall Log Source

Create a Log Source named `pfSense-Firewall` with:

| Setting | Value |
|---|---|
| Log Source Type | Netgate pfSense |
| Protocol | Syslog |
| Enabled | Yes |
| Source IP / Identifier | `192.168.10.254` or the actual pfSense source configuration |
| Target | QRadar UDP 514 |

Confirm in **Log Activity** that new firewall events reach QRadar. Installing
the package alone does not prove that a real payload is routed and parsed by
the correct DSM.

### 4. Resolve the initial Destination Port `0` condition

The initial lab run left `Destination Port` at `0` after the DSM and Log Source
were configured. Before changing detection logic, the following DSM Editor
procedure was used to supply QRadar with a real pfSense payload:

1. Generate traffic from Kali, for example with:

   ```bash
   nc -zv -w 2 192.168.10.20 53
   nc -zv -w 2 192.168.10.20 8080
   nc -zv -w 2 192.168.10.20 3389
   nc -zv -w 2 192.168.10.20 443
   ```

2. Open **Admin → DSM Editor** and locate a real event received by
   `pfSense-Firewall`.
3. Select the Netgate pfSense DSM and test the payload against it.
4. Assign or confirm the sample as a payload for the Log Source.
5. Save and deploy changes.
6. Generate new traffic and verify that the event name, low-level category,
   and Destination Port are populated.

Successful verification produced:

- Event Name: `Firewall - Deny`
- Low Level Category: `Firewall Deny`
- Log Source Status: `Success`
- Destination Port values such as `53`, `8080`, `3389`, and `443`

The root cause was not simply a missing DSM package. The DSM was present, but
a real `filterlog` payload had not been assigned through the parser workflow.
The first checks for this issue should therefore verify Remote Logging,
Log Source configuration, live events, and the payload visible in DSM Editor
before declaring parser incompatibility.

### 5. Detection and validation

After parsing was verified, R2 was rewritten as the CRE rule
`R2b - True Port Scan Detection`. The rule counts distinct Destination Ports
from the same Source IP within five minutes. R3 Phase 1 remains an AQL
fallback that can count distinct ports with `UNIQUECOUNT(destinationport)`;
its expected lab result was `SourceIP=192.168.20.10`,
`DistinctPorts=7`.

## Build Order

1. Deploy pfSense, configure WAN/LAN interfaces, establish routing between segments.
2. Deploy QRadar CE, verify web console access.
3. Deploy Ubuntu and Windows victims, confirm Layer 3 connectivity from Kali through pfSense.
4. Configure Syslog forwarding from Ubuntu and pfSense to QRadar.
5. Install the Netgate pfSense DSM and configure the `pfSense-Firewall` Log Source.
6. Validate live pfSense payload parsing in Log Activity and DSM Editor.
7. Install WinCollect on the Windows host, add Security + Sysmon local sources.
8. Build detection logic (see [detection-rules/](../detection-rules/)).
9. Run attack simulations, verify detections, perform response actions.
