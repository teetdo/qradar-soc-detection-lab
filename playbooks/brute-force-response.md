# Playbook - Brute-Force Incident Response

This playbook adapts the incident-response lifecycle described in NIST SP 800-61
to the detections and evidence available in this lab. The operational steps below
are a practical breakdown for this project, not a claim about NIST's phase count.

## 1. Preparation

- Confirm the SSH detection rule [R1](../detection-rules/R1-brute-force-ssh.md)
  and relevant AQL detections are enabled and have been tested with positive and
  negative cases.
- Know how to apply and remove a narrowly scoped firewall block, and how to
  restore the lab's intended network policy.
- Keep account recovery and credential-reset procedures documented for the
  systems in scope.

## 2. Detection and Analysis

- Review the QRadar offense or matching event sequence and collect the relevant
  event timeline before changing systems.
- Check [R1's Source IP limitation](../detection-rules/R1-brute-force-ssh.md):
  the offense field may identify the log-forwarding host rather than the
  originating client. Validate the source against the raw event payload and
  other available telemetry.
- Use [R4](../detection-rules/R4-login-after-failures.aql) to investigate
  whether a successful login followed the failed-login sequence. Treat this as
  an investigative aid and verify the underlying events.
- Assess affected accounts, hosts, time range, and evidence of successful
  access. Apply the organization's severity and escalation criteria.

## 3. Containment

- If the source is confirmed and a block is appropriate, apply a narrowly
  scoped firewall restriction using the lab's established change process.
- Verify the restriction is effective and does not interrupt unrelated
  legitimate traffic. Record the change and its time.

## 4. Eradication

- If unauthorized account changes are confirmed, preserve relevant evidence
  first, then remove or disable the affected account through the system's
  approved administration process.
- Review relevant authentication and account-change logs for additional
  unauthorized changes. This lab does not automate that broader review.

## 5. Recovery

- Reset credentials for confirmed affected accounts and review access that
  depends on those credentials.
- Restore temporary network restrictions when containment is no longer needed,
  and verify normal lab connectivity and logging.
- Monitor subsequent authentication events for recurring activity.

## 6. Post-Incident Activity

- Record the timeline, evidence reviewed, decisions, containment changes, and
  recovery verification.
- Document any detection or collection gaps as follow-up work. For example,
  the unresolved pfSense event-collection issue is described in
  [Lessons Learned](../lessons-learned/troubleshooting-journal.md).
- Update detection logic or response guidance only after validating the change
  against the lab's positive and negative test cases.
