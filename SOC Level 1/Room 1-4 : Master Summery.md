# Cybersecurity Acronyms: Full Meanings & Core Responsibilities

---

## 1. Executive & Security Leadership Roles

* **CEO (Chief Executive Officer):**
  * **Full Meaning:** Chief Executive Officer[cite: 1]
  * **Responsibility:** Directs overall corporate strategy, makes high-level executive decisions, and balances global business objectives and organizational risk[cite: 1].
* **CFO (Chief Financial Officer):**
  * **Full Meaning:** Chief Financial Officer[cite: 1]
  * **Responsibility:** Manages enterprise financial actions, tracks cash flow and budgeting, and assesses financial liabilities arising from operational and cyber risks[cite: 1].
* **CISO (Chief Information Security Officer):**
  * **Full Meaning:** Chief Information Security Officer[cite: 1]
  * **Responsibility:** Leads the enterprise-wide information security program, defines cybersecurity policies, manages security budgets, and reports risk posture directly to executive management[cite: 1].
* **CTO (Chief Technology Officer):**
  * **Full Meaning:** Chief Technology Officer[cite: 1]
  * **Responsibility:** Oversees technical architecture, product technology development, and technological infrastructure strategy across the organization[cite: 1].
* **CIO (Chief Information Officer):**
  * **Full Meaning:** Chief Information Officer[cite: 1]
  * **Responsibility:** Manages internal IT infrastructure, technical resource allocation, and organizational IT service delivery[cite: 1].

---

## 2. Operations & Engineering Roles

* **SOC (Security Operations Center):**
  * **Full Meaning:** Security Operations Center[cite: 1]
  * **Responsibility:** Serves as the centralized defensive unit responsible for continuously monitoring, detecting, analyzing, and responding to cyber threats and security alerts[cite: 1].
* **SOC L1 (Security Operations Center - Level 1 / Tier 1):**
  * **Full Meaning:** Security Operations Center Level 1 Analyst (Junior Security Analyst)[cite: 1]
  * **Responsibility:** Acts as the 24/7 first line of defense; monitors SIEM dashboards, performs initial alert triage, verifies indicators of compromise, filters false positives, and escalates unresolved incidents[cite: 1].
* **SOC L2 (Security Operations Center - Level 2 / Tier 2):**
  * **Full Meaning:** Security Operations Center Level 2 Analyst (Senior Analyst)[cite: 1]
  * **Responsibility:** Handles escalated complex alerts, conducts deep-dive host and memory investigations, analyzes malware behaviors, and reconstructs attack chains[cite: 1].
* **CSIRT / CERT (Computer Security Incident Response Team / Computer Emergency Response Team):**
  * **Full Meaning:** Computer Security Incident Response Team / Computer Emergency Response Team[cite: 1]
  * **Responsibility:** Leads rapid-response containment, network isolation, enterprise eradication, and digital forensics during critical, active breaches (such as ransomware outbreaks)[cite: 1].
* **GRC (Governance, Risk, and Compliance):**
  * **Full Meaning:** Governance, Risk, and Compliance[cite: 1]
  * **Responsibility:** Audits systems against industry frameworks and regulatory mandates, assesses third-party risk, and maintains organizational security policies[cite: 1].
* **AppSec (Application Security):**
  * **Full Meaning:** Application Security Engineer[cite: 1]
  * **Responsibility:** Evaluates source code, APIs, and web architecture to eliminate application-layer vulnerabilities (e.g., OWASP Top 10) before and after deployment[cite: 1].
* **DevSecOps (Development, Security, and Operations):**
  * **Full Meaning:** Development, Security, and Operations[cite: 1]
  * **Responsibility:** Integrates automated security scanning (SAST/DAST, secret detection, dependency audits) directly into continuous integration/continuous deployment (CI/CD) pipelines[cite: 1].

---

## 3. Technology & Framework Terminology

* **SIEM (Security Information and Event Management):**
  * **Full Meaning:** Security Information and Event Management[cite: 1]
  * **Responsibility / Function:** Ingests, aggregates, normalizes, and correlates log data across the enterprise in real time to generate actionable security alerts[cite: 1].
* **FW (Firewall):**
  * **Full Meaning:** Firewall[cite: 1]
  * **Responsibility / Function:** Filters incoming and outgoing network traffic based on predefined security rules to block unauthorized connections and brute-force attempts[cite: 1].
* **PCI DSS (Payment Card Industry Data Security Standard):**
  * **Full Meaning:** Payment Card Industry Data Security Standard[cite: 1]
  * **Responsibility / Function:** A mandatory information security standard that organizations must adhere to when storing, processing, or transmitting credit card and cardholder data[cite: 1].
* **APT (Advanced Persistent Threat):**
  * **Full Meaning:** Advanced Persistent Threat[cite: 1]
  * **Responsibility / Function:** A stealthy, sophisticated, and well-funded threat group (often nation-state or organized cybercrime) that conducts continuous, targeted malicious campaigns[cite: 1].
* **TTPs (Tactics, Techniques, and Procedures):**
  * **Full Meaning:** Tactics, Techniques, and Procedures[cite: 1]
  * **Responsibility / Function:** The behavioral patterns, operational methods, and technical tools used by adversaries across different stages of a cyberattack[cite: 1].
* **HR (Human Resources):**
  * **Full Meaning:** Human Resources[cite: 1]
  * **Responsibility / Function:** The corporate department managing personnel; frequently targeted by social engineering and phishing campaigns due to high external email volume[cite: 1].

---
# TryHackMe SOC Level 1: Master Summary & Architecture Briefing

---

## 1. SOC Hierarchy & Organizational Governance

A Security Operations Center functions within a layered organizational hierarchy designed to balance strategic business objectives, risk management, and tactical defense[cite: 1].

```
+-------------------------------------------------------------+
|               Executives (CEO, CFO, Board)                  | -> Business risk & continuity[cite: 1]
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|           Security Leadership (CISO, CTO, CIO)              | -> Enterprise security strategy[cite: 1]
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|          Security Management (SOC Manager, SecOps Lead)     | -> Operations, KPIs & escalation[cite: 1]
+-------------------------------------------------------------+
                              │
+-------------------------------------------------------------+
|         Technical Execution (L1, L2, Engineers, DFIR)       | -> Alert triage, detection, response[cite: 1]
+-------------------------------------------------------------+
```

### Governance Breakdown
* **Executive Layer (CEO/CFO):** Focuses on global corporate objectives, financial risk, and enterprise continuity[cite: 1].
* **Leadership Layer (CISO/CIO):** Directs the enterprise-wide information security program, risk frameworks, and high-level security strategies[cite: 1].
* **Management Layer (SOC Manager / Red Team Lead):** Directly oversees departmental operations, manages analyst workflow, tracks response metrics, and reports findings to executives[cite: 1].
* **Technical Execution Layer:** Technical specialists handling hands-on operations, including log analysis, penetration testing, compliance auditing, and pipeline tuning[cite: 1].

---

## 2. Core SOC Operations & Triage Workflow

A functional SOC relies on structured escalation paths between Tier 1, Tier 2, Engineering, and Incident Response to handle threats systematically[cite: 1].

```
[ Log Sources / Endpoints / Firewalls ]
                  │
                  ▼
      [ SIEM / Log Aggregation ]  <─── Maintained by SOC Engineer[cite: 1]
                  │
                  ▼ (Alert Triggered)
      [ Tier 1: SOC L1 Analyst ] ──── False Positive ──► Close Ticket
                  │ (True Positive / Requires In-Depth Work)
                  ▼
      [ Tier 2: SOC L2 Analyst ] ──── Deep-Dive Forensic & Host Analysis[cite: 1]
                  │ (Active Enterprise Breach)
                  ▼
      [ CSIRT / Incident Responder ] ──► Containment, Eradication & Recovery[cite: 1]
```

### Role Breakdown & Responsibilities
* **SOC L1 Analyst (Junior Security Analyst):**
  * **Function:** First line of defense operating 24/7[cite: 1].
  * **Core Tasks:** Continuous queue monitoring, initial alert triage, IOC lookup, false positive filtration, and ticket escalation[cite: 1].
  * **Real-World Analogy:** Emergency room triage nurse sorting critical trauma cases from minor illnesses.
* **SOC L2 Analyst (Senior Analyst):**
  * **Function:** Deep forensic investigation and escalation handling[cite: 1].
  * **Core Tasks:** Investigating complex attacks, performing host-level and memory analysis, analyzing phishing payloads, and correlating multi-stage event logs[cite: 1].
  * **Real-World Analogy:** Specialized surgeon called to the operating room when standard triage reveals deep trauma.
* **SOC Engineer:**
  * **Function:** Tooling architecture and platform reliability[cite: 1].
  * **Core Tasks:** Maintaining SIEM infrastructure, tuning alert rules, managing log storage/pipelines, and ensuring sensor uptime[cite: 1].
  * **Real-World Analogy:** Radar array engineer ensuring monitoring feeds and tracking instruments stay online.
* **Incident Responder (CERT / CSIRT Lead):**
  * **Function:** On-demand crisis management and emergency containment[cite: 1].
  * **Core Tasks:** Responding to major breaches (e.g., active ransomware), network isolation, digital forensics, root cause analysis, and enterprise eradication[cite: 1].
  * **Real-World Analogy:** SWAT / Hazmat unit deployed when the perimeter has been breached.

---

## 3. Supporting Blue Team Ecosystem

The SOC operates alongside complementary specialized security roles across the organization[cite: 1]:

| Role | Operational Scope | Interaction with SOC |
| :--- | :--- | :--- |
| **Threat Researcher / Intel Analyst** | Tracks Advanced Persistent Threat (APT) groups, campaign infrastructure, and adversary TTPs[cite: 1]. | Supplies IOC feeds and detection signatures (YARA/Sigma) for proactive hunting[cite: 1]. |
| **GRC Auditor** | Enforces regulatory frameworks (PCI DSS, ISO 27001, SOC 2, HIPAA)[cite: 1]. | Audits log retention, access controls, and compliance controls[cite: 1]. |
| **Penetration Tester (Red Team)** | Identifies and exploits system vulnerabilities offensively before attackers do[cite: 1]. | Simulates real-world attacks to validate SOC alerting rules and visibility[cite: 1]. |
| **AppSec Engineer** | Secures software applications and web codebases (OWASP Top 10 vulnerabilities)[cite: 1]. | Hardens endpoints and web services, reducing alerts at the WAF level[cite: 1]. |
| **DevSecOps** | Integrates automated security scanning (SAST/DAST) into CI/CD pipelines[cite: 1]. | Prevents flawed configurations and exposed secrets from reaching production[cite: 1]. |
| **AI Researcher** | Researches machine learning models and intelligent automation in security[cite: 1]. | Enhances anomaly detection models and automates high-volume log parsing[cite: 1]. |

---

## 4. Scenario-to-Role Tactical Attribution

| Incident Scenario | Primary Assigned Role | Technical Rationale |
| :--- | :--- | :--- |
| **"SIEM created an alert about FW-NY-01 firewall brute-force. Who should triage the alert?"**[cite: 1] | **Lucas (SOC L1 Analyst)**[cite: 1] | Tier 1 analysts perform front-line triage, check authentication failure logs, and verify whether access was gained[cite: 1]. |
| **"The HR manager Anna launched a phishing malware. Who should make a deep analysis?"**[cite: 1] | **Susan (SOC L2 Analyst)**[cite: 1] | Tier 2 analysts investigate escalated threats requiring host process analysis, reverse engineering payloads, and tracking behavior[cite: 1]. |
| **"The office in France was hit with ransomware; immediate response is required!"**[cite: 1] | **Robert (CERT / CSIRT Lead)**[cite: 1] | Critical enterprise-wide incidents require incident responders to isolate subnets, coordinate containment, and direct recovery[cite: 1]. |
| **"Our servers storing credit cards require PCI DSS audit. Who can help us here?"**[cite: 1] | **Nick (GRC Auditor)**[cite: 1] | GRC auditors evaluate adherence to compliance standards, access management policies, and encryption requirements[cite: 1]. |
| **"Who can check the new version of tryhackme.thm for vulnerabilities?"**[cite: 1] | **Ben (Penetration Tester)**[cite: 1] | Offensive security testers actively search for flaws and attempt exploitation prior to public release[cite: 1]. |
| **"The SIEM is unavailable due to a storage limit. Who can investigate the issue?"**[cite: 1] | **Eugen (SOC Engineer)**[cite: 1] | Security engineers maintain platform health, optimize index management, manage log retention, and expand storage pipelines[cite: 1]. |
| **"FIN7 threat group actively targets our company. Who can analyze their tactics?"**[cite: 1] | **Alice (Threat Researcher)**[cite: 1] | Threat researchers dissect adversary tactics, techniques, and procedures (TTPs) and map them to threat models[cite: 1]. |

---

## 5. Defensive & Offensive Correlation

* **Reconnaissance & Weaponization:** Threat researchers profile adversary campaigns and supply contextual threat intelligence to the defensive pipeline[cite: 1].
* **Delivery & Exploitation:** Penetration testers simulate attack vectors to ensure perimeter controls and SIEM rules trigger appropriately during early compromise stages[cite: 1].
* **Detection & Triage:** L1 analysts triage the initial high-volume alerts from perimeter devices (firewalls, IDS/IPS, WAFs) to confirm true threats[cite: 1].
* **Deep Analysis & Investigation:** L2 analysts examine post-exploitation activity, execution artifacts, and persistence mechanisms across affected endpoints[cite: 1].
* **Containment & Remediation:** CSIRT leads drive incident containment, subnet isolation, and eradication during critical breaches to maintain business continuity[cite: 1].

---

**Key Insight:** Robust defense depends on distinct operational boundaries — Tier 1 handles initial alert triage, Tier 2 investigates complex attacks, Engineering maintains data pipelines, and Incident Response leads crisis containment during active breaches[cite: 1].
**Key Insight:** Security acronyms represent distinct operational layers — leadership defines governance, engineering maintains data ingestion pipelines, and specialized tiers divide triage from containment to eliminate single points of failure[cite: 1].
