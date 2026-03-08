# Staff Engineer / Architect System Design Interview
## Theme: High-Level System and Architecture Design
## Domain: Cyber Security Automation (Financial Sector)
## Duration: 60 Minutes

---

## Part 1: The Prompt / Scenario (To be given to the candidate)

**Context:**
You are joining a global financial services firm as a Staff Engineer. The firm operates a massive hybrid infrastructure, including on-premise data centers and multiple public clouds (AWS and Azure). The security operations center (SOC) currently receives over 500,000 security alerts per day from various sources:
*   SIEM (Security Information and Event Management)
*   CSPM (Cloud Security Posture Management)
*   EDR (Endpoint Detection and Response)
*   Vulnerability Scanners
*   Identity and Access Management (IAM) anomaly detectors

**The Problem:**
Currently, Level 1 and Level 2 SOC analysts manually triage these alerts. Due to the volume, alert fatigue is extremely high, and the Mean Time To Remediate (MTTR) for critical vulnerabilities is measured in days, leaving the firm exposed to unacceptable risk. Furthermore, manual interventions are prone to error and lack consistent audit trails required by financial regulators (SEC, FINRA, etc.).

**The Goal:**
Design the architecture for an **Automated Security Orchestration, Automation, and Response (SOAR) Platform**. This platform must ingest alerts from all aforementioned tools, normalize the data, evaluate it against defined security policies (playbooks), and execute automated remediation actions where possible. If an action is too risky to automate fully, it should gather context and route it to an analyst for a "one-click" approval (Human-in-the-loop).

**Examples of automated actions:**
1.  **CSPM Alert:** An S3 bucket is found to be public. **Action:** Automatically apply a restrictive bucket policy to make it private.
2.  **EDR Alert:** Malware detected on an employee's laptop. **Action:** Automatically isolate the machine from the corporate network at the switch/VPN level.
3.  **IAM Alert:** Multiple failed logins followed by a successful login from a new country. **Action:** Automatically disable the user account and force a password reset.

**Your Task:**
Walk me through the system design of this platform from end to end.

---

## Part 2: Interviewer Guide & Expected Solution

### 1. Requirements Gathering (10 mins)
*A strong candidate should not immediately jump into drawing boxes. They should clarify the constraints.*

**Expected Clarifying Questions & (Interviewer Answers):**
*   **Scale/Throughput:** How many alerts? *(Answer: 500k/day baseline, but must handle spikes up to 10k/sec during a coordinated attack or scanner misconfiguration).*
*   **Latency constraints:** How fast does remediation need to happen? *(Answer: P0/Critical alerts must be acted upon within seconds. P3/Low can take minutes).*
*   **Reliability/Availability:** *(Answer: 99.99% - if the security automation platform goes down during an attack, it's catastrophic).*
*   **Audit/Compliance:** *(Answer: Immutable audit logs are strictly required. Every action taken, who/what authorized it, and the outcome must be stored for 7 years).*
*   **Extensibility:** *(Answer: We onboard new security vendors constantly. The architecture must be pluggable).*

### 2. High-Level Architecture Design (20 mins)
*The candidate should propose a decoupled, event-driven architecture.*

**Key Components Expected:**
1.  **Ingestion Layer (API Gateways & Webhooks):** Receives payloads from various security tools.
2.  **Message Broker / Event Bus:** (e.g., Apache Kafka, AWS Kinesis). Essential for decoupling ingestion from processing and handling massive spikes without dropping alerts.
3.  **Normalization & Enrichment Workers:**
    *   *Normalization:* Translates vendor-specific JSON (e.g., CrowdStrike vs. SentinelOne) into a canonical internal "Alert Object" schema.
    *   *Enrichment:* Queries CMDB (Configuration Management Database), HR systems, or Threat Intelligence feeds to add context (e.g., "Is this IP address a known Tor node?", "Does this server process credit card data?").
4.  **Decision / Rule Engine:** (e.g., Open Policy Agent (OPA), Drools, or a custom microservice evaluating DAGs/Playbooks). Evaluates the enriched canonical alert against defined rules to determine the action.
5.  **Execution Engine (Workers):** Microservices responsible for interacting with target systems (AWS API, Active Directory, Cisco Firewalls) to execute the remediation. Must handle rate-limiting, retries, and failures.
6.  **Human-in-the-Loop (HITL) Workflow Service:** Manages state for actions requiring approval. Integrates with Slack/Teams or Jira/ServiceNow to present analysts with context and action buttons.
7.  **Data Stores:**
    *   *State/Workflow DB:* (e.g., PostgreSQL, DynamoDB) Tracks the lifecycle of an alert (New -> Enriched -> Pending Approval -> Remediated).
    *   *Audit/Log Store:* (e.g., Elasticsearch/OpenSearch for searchability + AWS S3 with Object Lock for immutable, WORM-compliant cold storage).

### 3. Deep Dive & Architectural Trade-offs (20 mins)
*As a Staff/Architect, the candidate must demonstrate deep understanding of trade-offs.*

**Probe 1: Idempotency and Race Conditions**
*   *Question:* "What happens if the SIEM sends the same alert 5 times in one minute due to a retry bug? How do you ensure we don't disable a user account 5 times or trigger 5 separate Jira tickets?"
*   *Expected Answer:* The Ingestion/Normalization layer must implement deduplication. Use a distributed cache (Redis) with a TTL to track recent alert hashes. The Execution engine must also implement idempotent API calls to target systems.

**Probe 2: Security of the Automator (The "God Mode" problem)**
*   *Question:* "This system has the credentials to shut down servers, alter firewalls, and disable users. How do you secure the platform itself? If this system is compromised, the attacker owns the bank."
*   *Expected Answer:*
    *   **Least Privilege:** The execution workers shouldn't share one massive IAM role. They should assume narrow roles specifically tailored for the playbook they are executing.
    *   **Secrets Management:** Integration with HashiCorp Vault or AWS Secrets Manager. Secrets should be dynamically generated and short-lived where possible.
    *   **Network Segmentation:** The platform must run in a highly restricted VPC, only allowing outbound connections to specific APIs, with no inbound internet access.
    *   **Code/Playbook Signing:** Playbooks/Rules must be treated as code (GitOps). Any change to a remediation rule requires PR approval and CI/CD deployment. The engine should verify cryptographic signatures of playbooks before executing them.

**Probe 3: Handling Failure in Execution**
*   *Question:* "The Execution Engine tries to isolate a VM in Azure, but the Azure API is currently returning 503s. Walk me through the failure handling."
*   *Expected Answer:*
    *   Implementation of the **Outbox Pattern** or a **Dead Letter Queue (DLQ)**.
    *   Exponential backoff and jitter for retries.
    *   Circuit breakers to prevent hammering a failing downstream API.
    *   If retries are exhausted, the state moves to "Failed Automated Remediation" and immediately pages an on-call analyst via PagerDuty.

### 4. Communication & Leadership (10 mins)
*Assess how the candidate communicates complex ideas.*
*   Did they drive the conversation or wait for prompts?
*   Did they justify their technology choices (e.g., "I chose Kafka over RabbitMQ because we need replayability for audit purposes and high throughput during spikes")?
*   Did they keep the financial regulatory context in mind throughout the design?

---

## Evaluation Rubric

| Criteria | Needs Improvement | Meets Expectations (Senior) | Exceeds Expectations (Staff/Architect) |
| :--- | :--- | :--- | :--- |
| **Requirements** | Jumps straight to design. Misses non-functional requirements. | Asks about scale and latency. Identifies basic constraints. | Anticipates regulatory, audit, and security constraints specific to finance. |
| **Architecture** | Monolithic or tightly coupled design. Synchronous processing. | Decoupled, event-driven architecture using message queues. | clear separation of ingestion, state management, execution, and audit. Uses patterns like Saga or DAGs for playbooks. |
| **Resilience & Scale** | Doesn't address API rate limits or message spikes. | Uses DLQs and basic retries. Understands horizontal scaling. | Implements circuit breakers, idempotency, backpressure, and stateful vs stateless component separation. |
| **Security Design** | Treats the system like a standard CRUD app. | Mentions Vault/Secrets Manager for API keys. | Designs defense-in-depth: JIT access, strict network boundaries, immutable audit trails, GitOps for policy changes. |
