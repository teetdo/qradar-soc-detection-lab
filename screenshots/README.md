# Screenshot Evidence Gallery

This gallery lists the strongest screenshots already present in this folder.
Screenshots keep their original filenames so they can be traced back to the
collection timeline. Credentials and workstation hostnames must be redacted
before public release.

Use `<LINUX_ACCOUNT>`, `<WINDOWS_ACCOUNT>`, and `<ROGUE_ACCOUNT>` as public
placeholder labels. Public narrative and evidence captions should not expose
real account names even where the original local screenshot contains them.

## Architecture

| Evidence | Screenshot | Verified use |
|---|---|---|
| Lab network topology | [`Sơ đồ kiến trúc lab.png`](Sơ%20đồ%20kiến%20trúc%20lab.png) | Shows the WAN/LAN topology and the isolated lab segments |

## pfSense firewall and containment

| Evidence | Screenshot | Verified use |
|---|---|---|
| pfSense WAN rule for RDP, SSH, and attacker containment | [`Screenshot 2026-11-07 154410.png`](Screenshot%202026-11-07%20154410.png) | Shows `BLOCK_Attacker_Kali_Post_Incident` and the Kali source IP |
| pfSense firewall events received by QRadar | [`Screenshot 2026-11-17 105847.png`](Screenshot%202026-11-17%20105847.png) | Shows live `Firewall - Deny` events on `pfSense-Firewall` |
| Netgate pfSense parser verification | [`Screenshot 2026-11-17 144236.png`](Screenshot%202026-11-17%20144236.png) | Shows firewall deny events with parsed Log Source, category, and Destination Port fields |

## QRadar detection rules

| Evidence | Screenshot | Verified use |
|---|---|---|
| R2b test-stack configuration | [`Screenshot 2026-11-17 141453.png`](Screenshot%202026-11-17%20141453.png) | Documents the CRE rule for distinct Destination Ports |
| R2b rule action and response configuration | [`Screenshot 2026-11-17 142733.png`](Screenshot%202026-11-17%20142733.png) | Shows rule behavior and enablement |
| R3 Phase 1 recon AQL view | [`Screenshot 2026-11-19 225755.png`](Screenshot%202026-11-19%20225755.png) | Shows Source IP `192.168.20.10` and Destination Port data for pfSense recon |

## Windows RDP and post-exploitation

| Evidence | Screenshot | Verified use |
|---|---|---|
| Windows authentication failure events | [`qradar-windows-auth-failures.png`](qradar-windows-auth-failures.png) | Representative Windows authentication events |
| Existing SSH brute-force offense | [`qradar-ssh-brute-force-offense.png`](qradar-ssh-brute-force-offense.png) | Representative QRadar offense view |
| Existing R1 rule view | [`qradar-r1-brute-force-rule.png`](qradar-r1-brute-force-rule.png) | Representative SSH brute-force rule configuration |

## Sysmon extension limitation

| Evidence | Screenshot | Verified use |
|---|---|---|
| Sysmon Content Extension archive | [`Screenshot 2026-11-21 115251.png`](Screenshot%202026-11-21%20115251.png) | Shows the downloaded `SysmonContentExtension.zip` and its package contents |
| JAR-style signature and digest files | [`Screenshot 2026-11-21 115330.png`](Screenshot%202026-11-21%20115330.png) | Shows `META-INF`, RSA/SF signature files, and `MANIFEST.MF` |

## Evidence selection notes

- The folder contains more than 700 screenshots, including setup trials,
  exploratory screenshots, duplicate captures, and console output. The table
  above selects only images that visibly support architecture, detection,
  firewall containment, or a documented technical limitation.
- Before adding a screenshot to a public report, confirm the image does not
  expose credentials, tokens, personal information, private hostnames, or other
  sensitive lab details.
- The original full-resolution screenshots are retained in this folder and
  remain the source of truth. The gallery should not be treated as a claim that
  every screenshot in the folder is publication-ready.
