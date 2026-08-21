# TryHackMe SOC Level 1: Complete Architecture Briefing & Reference Guide

---

## 1. SOC Hierarchy & Organizational Governance

A Security Operations Center functions within a layered organizational hierarchy designed to balance strategic business objectives, risk management, and tactical defense.

```
+-------------------------------------------------------------+
|               Executives (CEO, CFO, Board)                  | -> Business risk & continuity
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|           Security Leadership (CISO, CTO, CIO)              | -> Enterprise security strategy
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|          Security Management (SOC Manager, SecOps Lead)     | -> Operations, KPIs & escalation
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|         Technical Execution (L1, L2, Engineers, DFIR)       | -> Alert triage, detection, response
+-------------------------------------------------------------+
```

### Governance Breakdown
* **Executive Layer (CEO/CFO):** Focuses on global corporate objectives, financial risk, regulatory liability, and enterprise uptime.
* **Leadership Layer (CISO/CIO):** Directs enterprise-wide information security programs, risk frameworks, and security budgets.
* **Management Layer (SOC Manager / Red Team Lead):** Oversees departmental operations, shift scheduling, SLA compliance, and executive reporting.
* **Technical Execution Layer:** Technical specialists handling log analysis, threat intelligence, penetration testing, compliance auditing, and pipeline tuning.

---

## 2. Core SOC Operations & Triage Workflow

A functional SOC relies on structured escalation paths between Tier 1, Tier 2, Engineering, and Incident Response to manage alerts systematically.

```
[ Log Sources / Endpoints / Firewalls ]
                  │
                  ▼
      [ SIEM / Log Aggregation ]  <─── Maintained by SOC Engineer
                  │
                  ▼ (Alert Triggered)
      [ Tier 1: SOC L1 Analyst ] ──── False Positive ──► Close Ticket
                  │ (True Positive / Complex Attack)
                  ▼
      [ Tier 2: SOC L2 Analyst ] ──── Deep-Dive Forensic & Host Analysis
                  │ (Active Enterprise Breach)
                  ▼
      [ CSIRT / Incident Responder ] ──► Containment, Eradication & Recovery
```

### Role Breakdown & Responsibilities
* **SOC L1 Analyst (Junior Security Analyst):**
  * **Function:** First line of defense operating 24/7.
  * **Core Tasks:** Continuous queue monitoring, initial alert triage, IOC validation, false positive filtering, and ticket escalation.
  * **Real-World Analogy:** Emergency room triage nurse sorting critical trauma cases from minor illnesses.
* **SOC L2 Analyst (Senior Analyst):**
  * **Function:** Deep forensic investigation and escalation handling.
  * **Core Tasks:** Host-level artifact analysis, memory forensics, phishing payload examination, and multi-stage event correlation.
  * **Real-World Analogy:** Specialized surgeon called to the operating room when standard triage reveals deep trauma.
* **SOC Engineer:**
  * **Function:** Tooling architecture and platform reliability.
  * **Core Tasks:** Maintaining SIEM infrastructure, tuning detection rules (Sigma/YARA), configuring log ingestion pipelines, and ensuring sensor uptime.
  * **Real-World Analogy:** Radar array engineer ensuring monitoring feeds and tracking instruments stay online.
* **Incident Responder (CERT / CSIRT Lead):**
  * **Function:** On-demand crisis management and emergency containment.
  * **Core Tasks:** Responding to major breaches (e.g., active ransomware), network isolation, digital forensics, root cause analysis, and enterprise eradication.
  * **Real-World Analogy:** SWAT / Hazmat unit deployed when the security perimeter has been breached.

---

## 3. Supporting Blue Team Ecosystem

The SOC operates alongside complementary specialized security roles across the enterprise:

| Role | Operational Scope | Interaction with SOC |
| :--- | :--- | :--- |
| **Threat Researcher / Intel Analyst** | Tracks Advanced Persistent Threat (APT) groups, campaign infrastructure, and adversary TTPs. | Supplies IOC feeds and detection signatures (YARA/Sigma) for proactive hunting. |
| **GRC Auditor** | Enforces regulatory frameworks (PCI DSS, ISO 27001, SOC 2, HIPAA). | Audits log retention, access controls, and compliance monitoring policies. |
| **Penetration Tester (Red Team)** | Identifies and exploits system vulnerabilities offensively before attackers do. | Simulates real-world attacks to validate SOC alerting rules and visibility. |
| **AppSec Engineer** | Secures software applications and web codebases (OWASP Top 10 vulnerabilities). | Hardens endpoints and web services, reducing alerts at the WAF level. |
| **DevSecOps** | Integrates automated security scanning (SAST/DAST) into CI/CD pipelines. | Prevents flawed configurations and exposed secrets from reaching production. |
| **AI Researcher** | Researches machine learning models and intelligent automation in security. | Enhances anomaly detection models and automates high-volume log parsing. |

---

## 4. Scenario-to-Role Tactical Attribution

| Incident Scenario | Primary Assigned Role | Technical Rationale |
| :--- | :--- | :--- |
| **"SIEM created an alert about FW-NY-01 firewall brute-force. Who should triage the alert?"** | **Lucas (SOC L1 Analyst)** | Tier 1 analysts perform front-line triage, check authentication failure logs, and verify whether access was gained. |
| **"The HR manager Anna launched a phishing malware. Who should make a deep analysis?"** | **Susan (SOC L2 Analyst)** | Tier 2 analysts investigate escalated threats requiring host process analysis, reverse engineering payloads, and tracking behavior. |
| **"The office in France was hit with ransomware; immediate response is required!"** | **Robert (CERT / CSIRT Lead)** | Critical enterprise-wide incidents require incident responders to isolate subnets, coordinate containment, and direct recovery. |
| **"Our servers storing credit cards require PCI DSS audit. Who can help us here?"** | **Nick (GRC Auditor)** | GRC auditors evaluate adherence to compliance standards, access management policies, and encryption requirements. |
| **"Who can check the new version of tryhackme.thm for vulnerabilities?"** | **Ben (Penetration Tester)** | Offensive security testers actively search for flaws and attempt exploitation prior to public release. |
| **"The SIEM is unavailable due to a storage limit. Who can investigate the issue?"** | **Eugen (SOC Engineer)** | Security engineers maintain platform health, optimize index management, manage log retention, and expand storage pipelines. |
| **"FIN7 threat group actively targets our company. Who can analyze their tactics?"** | **Alice (Threat Researcher)** | Threat researchers dissect adversary tactics, techniques, and procedures (TTPs) and map them to threat models. |

---

## 5. Defensive & Offensive Correlation

* **Reconnaissance & Weaponization:** Threat researchers profile adversary campaigns and supply contextual threat intelligence to the defensive pipeline.
* **Delivery & Exploitation:** Penetration testers simulate attack vectors to ensure perimeter controls and SIEM rules trigger appropriately during early compromise stages.
* **Detection & Triage:** L1 analysts triage initial high-volume alerts from perimeter devices (firewalls, IDS/IPS, WAFs) to confirm true threats.
* **Deep Analysis & Investigation:** L2 analysts examine post-exploitation activity, execution artifacts, and persistence mechanisms across affected endpoints.
* **Containment & Remediation:** CSIRT leads drive incident containment, subnet isolation, and eradication during critical breaches to maintain business continuity.

---

## 6. Acronym Master Reference: Full Meanings & Responsibilities

### Executive & Leadership Roles
* **CEO (Chief Executive Officer):** Directs overall corporate strategy, makes high-level executive decisions, and balances global business objectives and organizational risk.
* **CFO (Chief Financial Officer):** Manages enterprise financial actions, tracks cash flow and budgeting, and assesses financial liabilities arising from operational and cyber risks.
* **CISO (Chief Information Security Officer):** Leads the enterprise-wide information security program, defines cybersecurity policies, manages security budgets, and reports risk posture directly to executive management.
* **CTO (Chief Technology Officer):** Oversees technical architecture, product technology development, and technological infrastructure strategy across the organization.
* **CIO (Chief Information Officer):** Manages internal IT infrastructure, technical resource allocation, and organizational IT service delivery.

### Operations & Engineering Roles
* **SOC (Security Operations Center):** Serves as the centralized defensive unit responsible for continuously monitoring, detecting, analyzing, and responding to cyber threats and security alerts.
* **SOC L1 (Security Operations Center - Level 1 / Tier 1):** Acts as the 24/7 first line of defense; monitors SIEM dashboards, performs initial alert triage, verifies indicators of compromise, filters false positives, and escalates unresolved incidents.
* **SOC L2 (Security Operations Center - Level 2 / Tier 2):** Handles escalated complex alerts, conducts deep-dive host and memory investigations, analyzes malware behaviors, and reconstructs attack chains.
* **CSIRT / CERT (Computer Security Incident Response Team / Computer Emergency Response Team):** Leads rapid-response containment, network isolation, enterprise eradication, and digital forensics during critical, active breaches (such as ransomware outbreaks).
* **GRC (Governance, Risk, and Compliance):** Audits systems against industry frameworks and regulatory mandates, assesses third-party risk, and maintains organizational security policies.
* **AppSec (Application Security):** Evaluates source code, APIs, and web architecture to eliminate application-layer vulnerabilities (e.g., OWASP Top 10) before and after deployment.
* **DevSecOps (Development, Security, and Operations):** Integrates automated security scanning (SAST/DAST, secret detection, dependency audits) directly into continuous integration/continuous deployment (CI/CD) pipelines.

### Technology & Framework Terminology
* **SIEM (Security Information and Event Management):** Ingests, aggregates, normalizes, and correlates log data across the enterprise in real time to generate actionable security alerts.
* **FW (Firewall):** Filters incoming and outgoing network traffic based on predefined security rules to block unauthorized connections and brute-force attempts.
* **PCI DSS (Payment Card Industry Data Security Standard):** Mandatory information security standard that organizations must adhere to when storing, processing, or transmitting credit card and cardholder data.
* **APT (Advanced Persistent Threat):** Stealthy, sophisticated, and well-funded threat group (often nation-state or organized cybercrime) that conducts continuous, targeted malicious campaigns.
* **TTPs (Tactics, Techniques, and Procedures):** The behavioral patterns, operational methods, and technical tools used by adversaries across different stages of a cyberattack.
* **HR (Human Resources):** Corporate department managing personnel; frequently targeted by social engineering and phishing campaigns due to high external email volume.

---

**Key Insight:** Defense in depth relies on distinct operational boundaries — Tier 1 triages the volume, Tier 2 investigates the anomaly, Engineering maintains data pipelines, and Incident Response isolates the breach. Mastering each role's workflow enables a defender to analyze logs with offensive intuition and defensive precision.
