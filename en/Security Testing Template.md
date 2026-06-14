# Security Testing Plan & Report (Vulnerability & Penetration Testing)

**Document Control**
- **Project Name:** [Insert Project Name - Must match BRD/FSD]
- **Target Environment:** [e.g., Staging / Pre-Prod / Production]
- **Related FSD Version:** [e.g., 1.0 (Ref: Section 7 Security Requirements)]
- **Testing Window:** [e.g., June 15, 2026 - June 18, 2026]
- **Document Author:** [Insert Name/Title, e.g., Lead Application Security Engineer]
- **Status:** [Planned / In Progress / Remediation / Final Sign-Off]

---

## 1. Introduction
### 1.1 Purpose
The purpose of this document is to outline the strategy, execution, and findings of the Vulnerability Assessment (VA) and Penetration Testing (PT) for the deployed application and database. This ensures that all security controls defined in the FSD are correctly implemented and that the system is resilient against malicious attacks prior to production go-live.

### 1.2 Scope of Testing
**In-Scope Assets:**
- **Web UI (Frontend):** `https://staging.example.com` (React.js build)
- **Core API (Backend):** `https://api.staging.example.com` (Node.js microservice)
- **Background Worker:** Asynchronous processing service (Internal network)
- **Database:** PostgreSQL (Primary & Read-Replicas)
- **Integrations:** Mocked Payment Gateway & Corporate SSO (Okta) SAML endpoints.

**Out-of-Scope:**
- Denial of Service (DoS/DDoS) attacks.
- Physical security of the data centers/cloud provider (AWS/Azure).
- Third-party SSO provider infrastructure (Okta).

---

## 2. Testing Methodology & Tools
> *Guidance: Define the frameworks and tools used to ensure standardized and repeatable testing.*

- **Frameworks:** 
  - OWASP Top 10 (Web Application Security)
  - PTES (Penetration Testing Execution Standard)
  - CIS Benchmarks (for Database and Cloud Infrastructure)
- **Tools Utilized:**
  - **DAST (Dynamic Testing):** Burp Suite Professional, OWASP ZAP
  - **SAST/Dependency:** SonarQube, Snyk, Trivy (Container scanning)
  - **Database/Network:** Nmap, SQLMap, Nessus, pgAudit
  - **API Testing:** Postman, SwaggerSpy

---

## 3. Vulnerability Assessment (VA) Results
> *Guidance: Automated scanning and configuration review results. Focuses on known vulnerabilities, missing patches, and misconfigurations in the app and DB.*

| VA-ID | Asset / Component | Vulnerability Name | Severity (CVSS) | Tool Used | Description & Impact | Remediation Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VA-01** | Docker Containers | Base Image Vulnerabilities | High (7.5) | Trivy | Node.js base image contains CVE-2025-XXXX allowing RCE. | **Open** - DevOps to update base image to `node:20-alpine`. |
| **VA-02** | PostgreSQL DB | Unencrypted Data at Rest | Medium (5.3) | Nessus | AWS RDS encryption parameter was set to `false` in Terraform. | **Fixed** - Terraform updated, DB snapshot restored with AES-256. |
| **VA-03** | Core API | Missing Security Headers | Low (3.1) | OWASP ZAP | `X-Frame-Options` and `Content-Security-Policy` missing. | **Fixed** - Added to Nginx/Ingress configuration. |
| **VA-04** | S3 Bucket | Overly Permissive IAM Role | High (8.1) | Prowler | Background Worker IAM role has `s3:*` instead of specific bucket access. | **Open** - DevOps to restrict IAM policy to least privilege. |

---

## 4. Penetration Testing (PT) Results
> *Guidance: Manual, exploit-driven testing simulating a real-world attacker. Focuses on business logic, authentication bypass, and complex attack chains.*

| PT-ID | FSD Ref | Attack Scenario / Vector | Exploitability | Impact | Findings & Evidence | Remediation Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **PT-01** | FUNC-001 (Auth) | **JWT Algorithm Confusion & Expiry**<br>Attempt to change JWT algorithm to `none` or manipulate expiry claims. | Low | High | System correctly rejects `none` algorithm. However, JWT expiry is set to 24 hours instead of the FSD mandated 1 hour. | **Open** - Dev to update JWT signing config to 3600 seconds. |
| **PT-02** | FUNC-002 (Export) | **Insecure Direct Object Reference (IDOR) on CSV Download**<br>Intercept email download link and change `file_id=101` to `file_id=102`. | High | Critical | Attacker successfully downloaded another user's PII financial report by guessing sequential file IDs. | **Open** - Dev to implement UUIDs for file names and enforce ownership checks in the API. |
| **PT-03** | INT-001 (API) | **SQL Injection via Legacy CRM Sync**<br>Inject malicious payloads into the SOAP XML payload synced from the CRM. | Medium | High | WAF blocked standard SQLi, but a secondary order-by blind SQLi bypassed the filter and extracted DB version. | **Open** - Implement parameterized queries in the SOAP ingestion layer. |
| **PT-04** | N/A (UI) | **Cross-Site Scripting (Stored XSS) in User Profile**<br>Inject `<script>` tags into the "Company Name" field. | High | Medium | Payload executed when an Admin viewed the user list, stealing the Admin's session cookie. | **Open** - Implement React context-aware output encoding and DOMPurify. |

---

## 5. Technical & Non-Technical Considerations

### 5.1 Technical Considerations (Rules of Engagement)
- **Data Privacy:** Testing MUST be performed using synthetic/mock data. No real production PII (Personally Identifiable Information) is to be used or exfiltrated during testing.
- **Destructive Testing:** `DELETE` or `DROP` commands are strictly prohibited. Testers must use `SELECT` or `UPDATE` to prove impact without destroying data.
- **Rate Limiting & WAF:** The Web Application Firewall (WAF) and API rate limiters will remain **ACTIVE** during testing to evaluate their effectiveness. Testers must document any WAF bypasses.
- **Database Snapshots:** A fresh snapshot of the Staging DB must be taken immediately prior to PT-03 (SQLi testing) to allow for rapid restoration if data corruption occurs.

### 5.2 Non-Technical Considerations
- **Legal Authorization ("Get Out of Jail Free" Card):** All penetration testers must carry a signed Letter of Authorization (LoA) from the CISO and Legal department explicitly permitting these activities.
- **SOC / Monitoring Notification:** The Security Operations Center (SOC) and DevOps monitoring teams (Datadog/Splunk) MUST be notified of the testing window. 
  - *Option A (Blind Test):* SOC is NOT notified to test their incident response capabilities.
  - *Option B (Announced Test):* SOC is notified to prevent false-positive panic and unnecessary incident escalation. *(Current Plan: Announced Test)*.
- **Third-Party Coordination:** The Okta and Payment Gateway sandbox environments must be cleared for aggressive testing to avoid getting the corporate IP blacklisted by the vendors.

---

## 6. Roles & Responsibilities (PIC)
> *Guidance: Contact matrix for the testing window. Essential for rapid unblocking and incident management.*

| Role / Responsibility | Name | Contact / Chat | Responsibilities during Testing |
| :--- | :--- | :--- | :--- |
| **Lead Penetration Tester** | [Name] | [Email / Slack] | Executes PT scenarios, documents exploits, writes final report. |
| **AppSec Engineer (Internal)** | [Name] | [Email / Slack] | Reviews VA scans, triages false positives, assists Devs with remediation. |
| **Database Administrator (DBA)** | [Name] | [Email / Slack] | Monitors DB performance during scans, restores snapshots if needed. |
| **DevOps / Cloud Lead** | [Name] | [Email / Slack] | Manages WAF rules, provides test environments, fixes Infra VA findings. |
| **SOC Lead (Monitoring)** | [Name] | [Email / Slack] | Monitors SIEM for tester IPs, validates alert generation. |
| **CISO / Security Sponsor** | [Name] | [Email / Slack] | Final authority on risk acceptance and signing off on the LoA. |

---

## 7. Remediation & Risk Acceptance Plan
> *Guidance: How vulnerabilities will be handled post-testing.*

- **Critical & High Severity:** Must be remediated and re-tested **before** Production Go-Live. No exceptions.
- **Medium Severity:** Must be remediated within 14 days. If Go-Live is urgent, requires a formal Risk Acceptance Form signed by the Product Owner and CISO.
- **Low / Informational:** Logged in the technical debt backlog to be addressed in future sprints.

---

## 8. Final Sign-Off & Clearance
*By signing below, the security and business leaders confirm that the application has undergone rigorous security testing, all Critical/High vulnerabilities have been mitigated, and the residual risk is acceptable for Production deployment.*

| Name | Title / Role | Decision (Clear / Conditional / Blocked) | Signature | Date |
| :--- | :--- | :--- | :--- | :--- |
| [Name] | Lead Penetration Tester | [e.g., Clear] | ____________________ | [Date] |
| [Name] | Application Security Lead | [e.g., Clear] | ____________________ | [Date] |
| [Name] | Product Owner (Risk Acceptor) | [e.g., Clear] | ____________________ | [Date] |
| [Name] | CISO / InfoSec Director | [e.g., Clear] | ____________________ | [Date] |
