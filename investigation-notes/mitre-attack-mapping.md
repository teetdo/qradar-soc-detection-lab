# MITRE ATT&CK Mapping

Each technique below is marked **Verified** only where this lab produced direct evidence (a QRadar offense, a saved AQL result, or an audit log entry) for that specific behaviour. Nothing here is marked verified on the basis of theory alone.

| Technique ID | Name | Tactic | Evidence in this lab | Status |
|---|---|---|---|---|
| T1046 | Network Service Discovery | Discovery | R2 experimental AQL count against pfSense firewall logs identified traffic to seven different lab services; R2b final CRE rule counts distinct Destination Ports | ✅ Verified |
| T1110.001 | Brute Force: Password Guessing | Credential Access | R1 (SSH) and R1b (RDP) — threshold-based detection, both positive and negative test cases | ✅ Verified |
| T1078 | Valid Accounts | Defense Evasion / Persistence / Privilege Escalation / Initial Access | R4 — successful authentication event directly following a failure streak, same Source IP | ✅ Verified (for the login event itself) |
| T1087.001 | Account Discovery: Local Account | Discovery | Post-exploitation simulation — account/privilege enumeration commands captured via auditd | ✅ Verified |
| T1136.001 | Create Account: Local Account | Persistence | Rogue account creation (`useradd`/`usermod` on Linux; equivalent on Windows) captured via auditd and Windows Security Event Log, cross-platform category comparison | ✅ Verified |

## R2 and R2b distinction

- **R2 experimental AQL:** used while Destination Port was unparsed and
  remained `0`. It is retained as evidence of the initial workaround and as
  a reconstruction query, but it does not have the same service-discovery
  precision as the final rule.
- **R2b final CRE rule:** created after the Netgate pfSense DSM parsed real
  firewall payloads in QRadar. It counts distinct Destination Ports from the
  same Source IP within five minutes and was verified as Offense #9.
- **Mapping choice:** this lab's netcat activity enumerated selected TCP
  services on a known internal target. That behavior is mapped primarily to
  `T1046 Network Service Discovery`, not `T1595 Active Scanning`, because the
  evidence is service/port discovery on the lab host rather than broad
  external or Internet-facing reconnaissance.

## Not Covered by This Lab

The following techniques are common in SOC detection-engineering portfolios but were **not** attempted here, and are not claimed:

- T1562.001 (Disable or Modify Tools / security software tampering)
- T1070.001 (Indicator Removal: Clear Windows Event Logs)
- T1053.005 (Scheduled Task/Job persistence)
- T1566.002 (Phishing: Spearphishing Link)
- Malware execution / binary analysis of any kind

These represent natural next steps for extending this lab, rather than gaps hidden from this write-up.
