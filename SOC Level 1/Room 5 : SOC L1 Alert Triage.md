
**Triage in a Security Operations Center (SOC)** is the process of reviewing, categorizing, and prioritizing incoming security alerts to determine which represent genuine threats and require immediate investigation or remediation.

Borrowed from emergency medicine, SOC triage acts as the frontline filter against "alert fatigue," ensuring analysts focus on critical incidents rather than false alarms.

---

**Core Stages of SOC Triage**

* **Alert Ingestion & Initial Review (Tier 1):** SIEM/EDR platforms ingest thousands of alerts daily. Analysts review metadata (source IP, destination, timestamp, signature) to verify the triggering event.
* **Validation & Filtering:** Analysts determine whether an alert is a **True Positive** (legitimate malicious activity or policy violation) or a **False Positive** (benign system behavior misidentified as a threat).
* **Context Enrichment:** Analysts gather supporting data—such as threat intelligence feeds, asset criticality, user behavior history, and process logs—to understand the scope.
* **Severity Classification & Prioritization:** Incidents are scored based on potential business impact and threat severity:
* **Low/Informational:** Benign anomalies or low-risk policy deviations; logged or auto-closed.
* **Medium:** Potential policy violations or isolated malware; queued for routine follow-up.
* **High/Critical:** Active compromise, ransomware, lateral movement, or data exfiltration; prioritized immediately.


* **Action & Escalation:** Benign alerts are closed with notes. Validated high-severity threats are escalated to **Tier 2 (Incident Handlers)** or **Tier 3 (Threat Hunters)** for containment and deep-dive forensics.

---

**Common Triage Outcomes**

| Verdict | Meaning | Next Step |
| --- | --- | --- |
| **False Positive** | Harmless event triggered a rule | Close ticket; tune detection rule |
| **True Positive (Benign)** | Real event, but authorized (e.g., admin task) | Close ticket with documentation |
| **True Positive (Malicious)** | Verified threat or attack in progress | Contain asset, escalate to Tier 2/Incident Response |
