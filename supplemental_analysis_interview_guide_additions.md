# Supplemental Analysis: Interview Guide Additions

This note adds a few interview probes that are not strongly represented in `cyber_sec_automation_system_design_interview.md` but would help distinguish a solid senior candidate from a true staff/architect-level candidate.

## Additional Probe 1: Blast Radius and Safe Automation Rollout

Suggested interviewer question:

"A newly deployed remediation policy accidentally matches a broad class of production servers. How do you prevent the SOAR platform from taking down half the estate before a human notices?"

What an excellent answer should include:
- Dry-run or shadow mode for new playbooks.
- Progressive rollout by environment or asset tier.
- Hard runtime guardrails such as maximum actions per unit time.
- A kill switch that disables a playbook immediately.
- Clear separation between code signing and runtime safety controls.

Why it is useful:
- It tests whether the candidate thinks beyond correctness and into operational safety.
- It exposes whether they understand that the main risk in automation is often an internally caused outage, not just missed remediation.

## Additional Probe 2: Incident Correlation vs. Per-Alert Handling

Suggested interviewer question:

"How would you prevent 2,000 related alerts from generating 2,000 separate tickets, approvals, and remediation attempts during a malware outbreak?"

What an excellent answer should include:
- Correlation of alerts into incidents or campaigns.
- Suppression and fan-in logic.
- Distinct lifecycle models for alerts, incidents, and actions.
- Awareness that analysts and reporting often operate at the incident level.

Why it is useful:
- It reveals whether the candidate is designing for real SOC operations instead of just queue processing.

## Additional Probe 3: Evidence Preservation and Chain of Custody

Suggested interviewer question:

"Before automatically isolating or cleaning a host, how do you preserve evidence needed for later investigation or regulatory review?"

What an excellent answer should include:
- Pre-remediation evidence capture for selected playbooks.
- Checksums, timestamps, and chain-of-custody metadata.
- Policy logic for when containment can happen before evidence capture and when it cannot.
- Clear distinction between searchable audit logs and forensic artifacts.

Why it is useful:
- It tests whether the candidate understands the difference between compliance logging and investigation support.

## Additional Probe 4: Schema Evolution and Vendor Drift

Suggested interviewer question:

"New vendors are added constantly, and existing vendors change webhook payloads without notice. How do you keep normalization stable and replay old data safely?"

What an excellent answer should include:
- Versioned canonical schema.
- Contract testing or schema registry practices.
- Storage of raw payload plus normalized schema version.
- DLQs and observability around adapter failures.

Why it is useful:
- It evaluates practical platform ownership and long-term maintainability.

## Additional Probe 5: Regional Failover Without Double-Executing Actions

Suggested interviewer question:

"If you run the platform across regions for resilience, how do you fail over without blocking remediation or executing the same destructive action twice?"

What an excellent answer should include:
- Clear control-plane ownership for action dispatch.
- Idempotency plus regional fencing or lease-based leadership.
- Understanding that multi-region ingestion is easier than multi-region execution authority.
- Consideration of data residency and credential locality.

Why it is useful:
- It surfaces distributed-systems maturity under high-stakes failure conditions.

## Rubric Additions Worth Considering

The existing rubric is good, but these extra dimensions would sharpen evaluation:

- **Operational Safety:** Can the candidate explain guardrails, kill switches, rollout stages, and blast-radius limits?
- **Data Modeling:** Do they separate alerts, incidents, approvals, and remediation actions cleanly?
- **Forensic Rigor:** Do they preserve evidence, not just logs?
- **Platform Governance:** Do they address schema evolution, playbook lifecycle, and compatibility management?
- **Failure-Domain Thinking:** Do they reason clearly about AZ vs. region failures and duplicate action risks?

## Bottom Line

The current interview pack already covers the major architectural pillars. These additions would make it better at identifying candidates who can safely run a real SOAR platform in production, not just sketch a strong high-level design.
