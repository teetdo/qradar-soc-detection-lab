# Screenshots

These screenshots document the isolated lab topology and representative QRadar
detection results.

## Lab architecture

![Lab network architecture](<Sơ đồ kiến trúc lab.png>)

## QRadar detection evidence

### SSH brute-force rule

![QRadar rule for SSH brute-force detection](qradar-r1-brute-force-rule.png)

The rule triggers after at least eight events from the same source IP within
five minutes when the event payload contains `Failed password`.

### Windows authentication failures

![QRadar Windows authentication failure events](qradar-windows-auth-failures.png)

The workstation hostname has been redacted.

### SSH brute-force offense

![QRadar offense showing repeated SSH login failures](qradar-ssh-brute-force-offense.png)

The offense summarizes repeated SSH login failures followed by a successful
login in the lab.

Before publishing additional screenshots, remove or redact credentials,
personal information, internal hostnames, and any other data that should not be
public.
