# R1 — SSH Brute Force Detection

**Type:** QRadar Custom Rules Engine (CRE) rule
**MITRE ATT&CK:** T1110.001 (Brute Force: Password Guessing)
**Status:** ✅ Verified — tested positive and negative

## Logic

```
when at least 8 events are seen with the same Source IP in 5 minutes
and when the Event Payload contains 'Failed password'
```

Response: mark the detected event as part of an offense, index the offense by Source IP, and annotate the triggering event with `Brute-force threshold exceeded (>=8 failed logins/5min)`.

## Why payload matching instead of a category condition

Log Activity showed every event from this log source with `Low Level Category = Unknown Generic Log Event` — the generic catch-all DSM was not parsing a structured authentication category. Rather than wait on a DSM fix, the rule was built against the raw payload string `Failed password`, taken verbatim from `/var/log/auth.log`. This is a pragmatic engineering trade-off: less elegant than a category-based condition, but functionally correct and immediately testable.

## Test Results

| Test | Input | Expected | Actual |
|---|---|---|---|
| Negative | 5 wrong passwords (below threshold) | No offense | ✅ No offense created |
| Positive | 15 passwords, 1 correct | Offense created | ✅ Offense created, 19 events |

## Known Limitation

The offense's **Source IP field shows the victim's address** (192.168.10.20, the Ubuntu host), not the attacker's (192.168.20.10, Kali). Root cause: the syslog packet reaching QRadar is physically sent *by the victim* (its own rsyslog daemon forwarding its own log), so QRadar attributes Source IP to the network-level sender rather than parsing the attacker's IP embedded inside the payload text (`...from 192.168.20.10...`). The attacker's real address is present in the raw payload but not promoted to a structured field.

**Planned fix (not implemented in this iteration):** a Custom Event Property using regex extraction against the payload to correctly populate Source IP, then reference that property in the rule instead of the standard field.

## Contrast: Windows Equivalent (R1b)

The Windows-side RDP brute-force rule (built later against WinCollect-sourced events) did **not** need this workaround — Source IP parsed correctly from the start via the dedicated Microsoft Windows Security Event Log DSM, allowing a clean category-based condition instead of a payload match. See the [Windows authentication evidence](../screenshots/qradar-windows-auth-failures.png).
