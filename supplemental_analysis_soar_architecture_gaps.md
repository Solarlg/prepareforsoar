# Supplemental Analysis: SOAR Architecture Gaps and Additions

This note supplements `candidate_solution_cyber_sec_automation.md` and `cyber_sec_automation_system_design_interview.md` with a few relevant concerns that are not covered in depth but matter a lot in a real financial-sector SOAR platform.

## 1. Alert-Centric Design vs. Incident-Centric Operations

Both existing documents mostly reason about a single alert moving through the pipeline. That is a good starting point, but large SOC environments usually need an explicit **incident correlation layer** on top of raw alert processing.

Why this matters:
- One malware outbreak can generate hundreds or thousands of nearly identical alerts across EDR, SIEM, IAM, and vulnerability scanners.
- If the system acts on each alert independently, it can create action storms: duplicate ticket creation, repeated host isolation requests, and noisy analyst approvals.
- Analysts usually work incidents or campaigns, not isolated alert records.

What to add architecturally:
- A correlation service that groups alerts into an `Incident` object using keys like host, user, malware family, campaign ID, or time-window heuristics.
- Suppression and fan-in logic so one incident can drive one approval workflow and one remediation plan across many related alerts.
- Separate state for `Alert`, `Incident`, and `Remediation Action`, rather than using a single state machine for everything.

## 2. Runtime Safety Rails for Automation Blast Radius

The current analysis covers approval workflows, least privilege, and signed playbooks, but it does not explicitly cover **runtime blast-radius controls**. In practice, this is one of the biggest failure modes in security automation.

Why this matters:
- A valid but buggy playbook can quarantine production hosts, disable service accounts, or revoke access for a large number of users very quickly.
- Signed code proves provenance, not correctness.

What to add architecturally:
- A per-playbook kill switch that can disable execution immediately without redeploying the platform.
- Dry-run or shadow mode for new playbooks, where the system records intended actions without performing them.
- Blast-radius guardrails such as "no more than N prod hosts isolated in 10 minutes" or "never disable break-glass accounts automatically."
- Progressive rollout by environment, business unit, or asset criticality tier.
- Compensating-action support for reversible actions, for example restoring a security group rule or re-enabling an account after operator review.

## 3. Evidence Preservation Before Remediation

The current documents discuss audit logs well, but they do not explicitly separate **auditability** from **forensic evidence preservation**.

Why this matters:
- Some remediation actions destroy volatile evidence. Isolating a host, killing a process, deleting a file, rotating credentials, or reimaging a machine can make later investigation harder.
- Financial firms often need both rapid containment and legally defensible investigation records.

What to add architecturally:
- A pre-remediation evidence collection step for certain playbooks, such as process lists, network connections, file hashes, memory capture metadata, or cloud control-plane snapshots.
- Chain-of-custody metadata stored with each evidence artifact: who triggered it, exact timestamps, host identity, tool version, and checksum.
- Policy-driven branching so some actions are blocked until evidence capture succeeds, while others allow containment first and collection second.

## 4. Canonical Schema Governance and Versioning

The candidate solution mentions canonical normalization and OCSF, which is strong, but it does not go deep on **schema evolution**. In a pluggable vendor ecosystem, schema drift becomes an operational problem quickly.

Why this matters:
- Vendors change payloads unexpectedly.
- New internal fields get added over time for analytics, compliance, and automation logic.
- Replay and audit become harder if historical events are interpreted with today's schema assumptions.

What to add architecturally:
- A versioned canonical schema with compatibility rules.
- A schema registry or contract-testing workflow for adapters.
- Adapter-level metrics and DLQ reason codes for malformed payloads or version mismatches.
- Storage of both raw vendor payload and normalized schema version so historical replays are deterministic.

## 5. Priority Isolation and Overload Governance

The existing material does a good job on Kafka, retries, and autoscaling, but it does not explicitly define **priority lanes** or degraded-mode behavior.

Why this matters:
- During a real attack, not all alerts deserve equal compute and queue share.
- A flood of low-value scanner noise can delay P0 containment if all work shares the same execution path.

What to add architecturally:
- Separate topics, partitions, or worker pools for critical vs. non-critical actions.
- Reserved execution capacity for high-severity remediations.
- Explicit degraded mode: continue P0/P1 containment, defer enrichment for low-severity alerts, and shed optional downstream work like secondary notifications.
- Backpressure policies that preserve ingestion durability while protecting downstream systems from overload.

## 6. Regional Boundaries and Duplicate Execution During Failover

The candidate solution mentions multi-AZ and optional multi-region DynamoDB, but the design does not fully address **how regional failover interacts with destructive actions**.

Why this matters:
- A duplicated control plane can issue the same remediation twice during failover if ownership is not fenced correctly.
- Financial-sector environments often have data-residency and jurisdiction constraints that should also apply to automated actions and analyst workflows.

What to add architecturally:
- A clear control-plane ownership model, such as active-passive per region for execution authority, even if ingestion is multi-region.
- Fencing tokens or lease-based leadership for action dispatch so only one region can authorize a destructive remediation at a time.
- Regional execution boundaries where EU alerts are processed and acted upon with EU-resident services and credentials when required.

## Bottom Line

The current analysis is already strong on event-driven design, security hardening, idempotency, and compliance. The biggest missing additions are:

1. incident correlation,
2. runtime automation safety rails,
3. forensic evidence preservation,
4. schema/version governance, and
5. regional execution authority during failover.

Those topics make the design more realistic for a high-risk, heavily regulated SOAR deployment.
