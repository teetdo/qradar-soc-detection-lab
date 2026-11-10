# QRadar SOC Detection Lab

> A hands-on home lab simulating the full detection lifecycle of a Security Operations Centre — reconnaissance, credential brute-force, cross-platform correlation, and incident response — built on IBM QRadar CE against a segmented firewall topology.

---

## Recruiter TL;DR

This project demonstrates SIEM configuration, custom correlation rule authoring (QRadar CRE), AQL query engineering as a fallback when parsing limitations blocked rule-based detection, cross-platform log collection (Linux auditd + Windows WinCollect/Sysmon), MITRE ATT&CK mapping, and a full detect → investigate → respond → verify cycle — including real attacker-IP blocking, rogue account removal, and credential rotation, not just alert generation.

Three genuine engineering gaps were found during the build (a firewall anti-spoofing rule silently dropping traffic, a DSM that never parsed the field a rule depended on, an RDP brute-force tool failing against NLA) and are documented honestly as a troubleshooting journal rather than edited out of the story — see [`lessons-learned/`](lessons-learned/troubleshooting-journal.md).

Published evidence exports use anonymized account and host labels; private lab IP addresses are retained. The SHA-256 manifest covers these sanitized files.

Start here: [Lab Setup](lab-setup/README.md) · [Detection Rules](detection-rules/) · [Evidence](evidence/) · [Lessons Learned](lessons-learned/troubleshooting-journal.md) · [MITRE ATT&CK Mapping](investigation-notes/mitre-attack-mapping.md)

---

## ⚠️ Ethical & Legal Notice

Every action in this repository took place inside a private, isolated VMware lab on personal hardware. No traffic ever left the lab's internal network segments. Tool usage (Hydra, Nmap/netcat) was limited to systems owned and controlled by the lab operator.

---

## Project Overview

| Property | Detail |
|---|---|
| Focus Area | Blue Team / SOC Analysis / Detection Engineering |
| Lab Type | Local virtualised environment (VMware Workstation) |
| SIEM Platform | IBM QRadar CE 7.3.3 |
| OS Targets | Ubuntu Linux (SSH), Windows 10 (RDP) |
| Network | Segmented WAN/LAN via pfSense firewall — not a flat network |
| Attack Source | Kali Linux (isolated lab network only) |

---

## Lab Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        Host Machine                         │
│                                                               │
│  ┌─────────────┐                                             │
│  │ Kali Linux  │  192.168.20.10          WAN: 192.168.20.1  │
│  │ (Attacker)  │───────────────┐       ┌──────────────────┐  │
│  └─────────────┘               └──────▶│      pfSense      │  │
│                                         │  (Firewall/Router) │  │
│                                         └──────────┬─────────┘  │
│                                     LAN: 192.168.10.254│        │
│                    ┌────────────────────┬────────────┴───┐    │
│                    ▼                    ▼                ▼    │
│            ┌───────────────┐   ┌───────────────┐  ┌───────────┐│
│            │  Ubuntu-Lab   │   │   Win10-Lab   │  │  QRadar CE ││
│            │ 192.168.10.20 │   │ 192.168.10.30 │  │192.168.10.103││
│            └───────────────┘   └───────────────┘  └───────────┘│
└────────────────────────────────────────────────────────────┘
```

Unlike a flat host-only lab, traffic from the attacker segment must traverse a real stateful firewall to reach either victim — the same firewall that, as documented in [Lessons Learned](lessons-learned/troubleshooting-journal.md), silently blocked legitimate traffic via its default anti-spoofing policy and had to be properly diagnosed and fixed.

See [`lab-setup/README.md`](lab-setup/README.md) for full build details.
See the [screenshot gallery](screenshots/README.md) for the lab topology and
QRadar detection evidence.

---

## Tools Used

| Tool | Role |
|---|---|
| **Kali Linux** (Hydra, netcat) | Attack simulation |
| **pfSense** | Firewall / router, WAN-LAN segmentation, source of network-layer logs |
| **IBM QRadar CE** | SIEM — Custom Rules Engine (CRE) + AQL correlation |
| **auditd + rsyslog** | Linux kernel-level audit logging, forwarded via custom `imfile` pipeline |
| **Sysmon + WinCollect** | Windows process telemetry and Security Event Log collection |

---

## Detection Scenarios

| # | Scenario | MITRE ATT&CK | Detection Method | Status |
|---|---|---|---|---|
| 1 | SSH Brute Force (Linux) | T1110.001 | QRadar CRE custom rule — threshold + payload match | ✅ Complete |
| 2 | Port Scan via Firewall Logs | T1595 | AQL (CRE blocked by unparsed port field — see below) | ✅ Complete |
| 3 | Recon → Brute Force Chain | T1595 → T1110.001 | AQL, two-phase correlation | ✅ Complete (timing caveat documented) |
| 4 | Successful Login After Failures | T1078 | AQL, ordered sequence reconstruction | ✅ Complete |
| 5 | RDP Brute Force (Windows) | T1110.001 | QRadar CRE custom rule (category-based, cleanly parsed) | ✅ Complete |
| 6 | Post-Exploitation: Account Creation & Privilege Escalation | T1136.001, T1087.001 | auditd (Linux) / Security Event Log (Windows) cross-platform comparison | ✅ Complete |
| 7 | Incident Response: Containment & Remediation | — | Firewall block, account removal, credential rotation, verified containment | ✅ Complete |

**Why scenario 2 isn't a CRE rule:** Destination Port stayed `0` on every event even after installing the vendor-specific firewall DSM. Rather than block on an unresolved parser issue, detection moved to AQL against the raw event stream — a documented, deliberate engineering decision, not a shortcut. Full root-cause notes in [Lessons Learned](lessons-learned/troubleshooting-journal.md).

---

## Skills Demonstrated

- **SIEM Engineering** — Custom Rules Engine authoring, AQL query writing, DSM/Log Source configuration and troubleshooting
- **Log Pipeline Engineering** — rsyslog `imfile` forwarding, RainerScript syntax, cross-platform log normalization
- **Network Security** — stateful firewall rule design, anti-spoofing policy diagnosis, Layer 2/3 troubleshooting with `ip neigh`/ARP
- **Detection Engineering** — building correlation logic against unparsed/raw payload data when structured fields aren't available
- **Incident Response** — containment (firewall block), eradication (account removal), recovery (credential rotation), verification
- **MITRE ATT&CK Mapping** — technique-level tagging of each detection, with explicit distinction between *verified* and *not yet verified* coverage
- **Technical Troubleshooting** — browser compatibility bugs, firewall silent-drop diagnosis, syntax debugging, resource contention

---

## How to Navigate This Repository

```
qradar-soc-detection-lab-starter/
├── detection-rules/        # QRadar CRE rules and AQL queries
├── evidence/               # Sanitized event exports and SHA-256 manifest
├── investigation-notes/    # MITRE ATT&CK mapping
├── lab-setup/              # Environment build and network design
├── lessons-learned/        # Troubleshooting journal
└── screenshots/            # Lab topology and QRadar evidence
```

---

## Status Legend

| Symbol | Meaning |
|---|---|
| ✅ | Complete — evidence collected and documented |
| 🔄 | In progress |
| 📋 | Planned, not yet started |

---

*Built as part of a cybersecurity internship program. Company-internal process references have been generalised to the public NIST SP 800-61 incident response framework to respect confidentiality.*
