# Troubleshooting Journal

This is the part of the lab most tutorials skip: the real diagnostic process behind every "it works now" screenshot. Each entry follows the same structure — symptom, false leads ruled out, root cause, fix, and what it taught.

---

## 1. QRadar Rule Wizard "Next" button silently disabled

**Symptom:** Inside the Custom Rules Wizard, the `Next >>` button rendered greyed-out and unresponsive on the very first screen, with no visible error anywhere in the UI.

**Ruled out:** popup blockers, browser zoom level, keyboard focus, mouse click coordinates, stale session cookies.

**Root cause, found via browser DevTools console (F12):**

```
dojo/parser::parse() error TypeError: Cannot read properties of undefined (reading 'firstChild')
Failed to load resource: the server responded with a status of 404 (404)   x3
```

Three 404s against a `null` URL plus a Dojo widget-parsing crash pointed at a JavaScript asset that never loaded. Switching the browser from Chrome to Firefox resolved it immediately — this is a known compatibility gap between QRadar 7.3.3's legacy Dojo-based UI and current Chrome releases, not a configuration fault.

**Fix:** use Firefox for the QRadar web console on this version.

**Takeaway:** when a legacy web application fails in a way that makes no sense from the UI alone, open the browser console before touching any server-side configuration. It took minutes to find the real cause once the console was checked, versus a long chain of guesses beforehand.

---

## 2. Host resource contention across 5 simultaneous VMs

**Symptom:** QRadar became sluggish and intermittently unresponsive; `qradar.error` logged repeated `SarSentinel` warnings for system load and disk I/O exceeding configured thresholds (I/O latency at ~996ms against a 500ms threshold).

**Root cause:** the host machine has 16 GB of physical RAM. Running QRadar (10 GB), Kali (4 GB), Ubuntu (4 GB), and pfSense (1 GB) simultaneously — plus two additional VMs left powered on unnecessarily — pushed Task Manager to show 15.5/15.7 GB (99%) in use, with ~13.7 GB being actively swapped to disk.

**Fix:** only keep the VMs required for the current task powered on. Four core VMs alone (19 GB) already exceed the host's physical RAM, so even a minimal working set requires accepting some swap.

**Takeaway:** resource-contention symptoms on one VM (QRadar's I/O warnings) can have their actual root cause on a completely different layer (the hypervisor host). Always check host-level resource usage before assuming the problem is inside the guest.

---

## 3. pfSense silently dropping all attacker traffic after a reboot

**Symptom:** after a pfSense reboot, every SSH connection and ICMP ping from Kali to the internal hosts timed out — despite the relevant firewall allow-rules being present, enabled, and correctly ordered, and both WAN and LAN interfaces showing "up" with correct IPs on the pfSense dashboard.

**Ruled out (in order):** firewall rule misconfiguration, VMware virtual switch failure (ruled out by checking `ip neigh` on Kali, which showed the pfSense WAN gateway's MAC address as `REACHABLE` — proof Layer 2 connectivity was completely healthy).

**Root cause:** the **"Block private networks and loopback addresses"** option on the pfSense WAN interface was enabled. Because the lab's "WAN" segment (Kali's network) deliberately uses a private RFC1918 range, this anti-spoofing option silently dropped every packet from that range *before* pfSense even evaluated the custom allow-rules — regardless of whether those rules were enabled. Interface tests (ARP, ping to the interface itself) passed because they don't traverse this filter; only inbound WAN traffic did.

**Fix:** uncheck "Block private networks and loopback addresses" under `Interfaces → WAN`, then `Apply Changes`.

**Recurrence:** the same setting re-enabled itself after a later set of VM power cycles and had to be fixed a second time during the Windows RDP branch of the project — now treated as a known gotcha to check first whenever private-IP-as-WAN connectivity breaks unexpectedly.

**Takeaway:** a default security feature (anti-spoofing) can be the correct behaviour on a real network and still be the wrong default for a lab topology that intentionally uses private addressing on both sides of the firewall. Interface-level options that run *before* the rule table are easy to overlook when debugging only at the rule level.

---

## 4. rsyslog RainerScript syntax errors forwarding Linux audit logs

**Symptom:** after writing a config file to forward `auditd` logs via `imfile`, `rsyslogd` failed to start cleanly, logging `invalid character in selector line` and `imfile: on startup file ... does not exist`.

**Root cause, in two parts:**
1. The initial config mixed modern RainerScript syntax (`module(load="imfile")`, `input(type="imfile" ...)`) with the legacy selector/action syntax (`local6.* @host:port`) in the same file — rsyslog's modern parser rejects this mixture.
2. A wrong file path (`/var/log/audit.log` instead of the actual `/var/log/audit/audit.log`).

**Fix:** rewrote the config entirely in RainerScript, routing the input through a named `ruleset` with an explicit `action(type="omfwd" ...)` block instead of a legacy selector line, and corrected the file path. Verified line-by-line structure with `cat -A` (which prints `$` at each true line ending) before every restart, after an earlier attempt via `echo` with embedded `\n` produced a file that *looked* correct to `cat` but parsed incorrectly due to how the shell had interpreted the string.

**Takeaway:** when a config file "looks right" in a plain `cat` but still fails to parse, verify the actual line structure with a tool that shows line-ending characters explicitly, rather than re-reading the same visually-correct output repeatedly.

---

## 5. Firewall DSM installed correctly but never received events

**Symptom:** a dedicated Netgate pfSense DSM was installed on QRadar (two RPM packages, `DSMCommon` then `NetgatePfSense`, in the correct dependency order) and a matching Log Source created — yet the Log Source's status remained `N/A` with `Last Event Time: N/A` indefinitely, and the Destination Port field on related events stayed `0`.

**Status:** root cause **not fully resolved** within this project's timeframe. Installing the correct DSM package is necessary but was not sufficient — the actual routing of events to that Log Source never happened, and the specific reason (Log Source Identifier mismatch, protocol misconfiguration, or a QRadar-side re-indexing delay) was not isolated.

**Workaround:** port-scan detection (R2) and the recon phase of the behavioural chain (R3) were built in AQL against the raw event stream instead of depending on the parsed Destination Port field.

**Takeaway:** installing the "correct" vendor DSM does not guarantee parsing will work immediately — always verify against live data in Log Activity before building detection logic on top of a field you assume is populated. Documenting an unresolved gap honestly, with the workaround that was used instead, is more valuable than pretending the DSM worked.

---

## 6. RDP brute force failing for three independent, stacked reasons

**Symptom:** `hydra -l labuser -P small.txt -t 4 rdp://192.168.10.30` failed every attempt with `[ERROR] freerdp: The connection failed to establish.`

**Root causes, found one at a time:**
1. **NLA (Network Level Authentication)** was enabled on the Windows host. Hydra's RDP module is explicitly marked "experimental" by its own authors and does not reliably negotiate NLA. Confirmed via `Win32_TSGeneralSetting.UserAuthenticationRequired = 1`.
2. After the pfSense block rule from the Linux-side incident response (`BLOCK_Attacker_Kali_Post_Incident`) had been left in place, it was still blocking **all** Kali traffic — including this unrelated RDP test — and had to be temporarily disabled for the Windows scenario.
3. The "Block private networks" anti-spoofing option (see item 3) had re-enabled itself after a VM restart and had to be unchecked again.

**Fix:** disabled NLA via `(Get-WmiObject ... Win32_TSGeneralSetting).SetUserAuthenticationRequired(0)`, confirmed the pfSense rule state, and reduced Hydra's thread count from 4 to 1, which also measurably improved connection reliability against the experimental RDP module.

**Takeaway:** when the same symptom could plausibly come from three unrelated layers (application-level auth negotiation, firewall rule left over from a *different* scenario, and a known-recurring firewall setting), check each independently rather than assuming the most recent change is the cause — in this case, a lab-state artifact (a block rule intentionally created in an earlier scenario) was as significant a contributor as the actual new bug.

---

## 7. Timezone mismatch undermining cross-system timing measurements

**Symptom:** Kali's local system clock was running in a different timezone than QRadar's configured timezone, which would have made any naive "event detected X seconds after attack started" latency calculation across the two systems wrong.

**Fix / discipline applied:** rather than compute or publish any cross-system latency or SLA figure from mismatched timestamps, this was documented as a known limitation, and such measurements were explicitly withheld rather than estimated. (General recommendation: NTP-sync every lab host to a common source before relying on cross-system timing.)

**Takeaway:** a silently wrong number is worse than an explicitly missing one. Not fabricating a result when the underlying data can't support it is itself a documented engineering decision in this project.
