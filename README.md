
# 🎣 Phishing Email Detection & Automated Incident Ticketing Using n8n

## 📌 Project Overview

This project demonstrates how Security Orchestration, Automation, and Response (SOAR) principles can be used to automate phishing email alert triage, threat intelligence enrichment, and incident ticket creation.

I developed an automated security workflow using **n8n, Wazuh, VirusTotal, and Trello** to process phishing-related security alerts, extract suspicious URLs, analyze them using external threat intelligence, and generate incident tickets when malicious activity is identified.

In a traditional Security Operations Center (SOC), investigating a suspicious email may require analysts to manually review the alert, extract embedded URLs, submit them to a threat intelligence service, interpret detection results, and document the incident.

This workflow automates those repetitive tasks by connecting alert ingestion, email parsing, URL analysis, severity classification, and ticket creation into a single process.

The automation supports both simulated Wazuh alerts and webhook-based ingestion, allowing the workflow to be tested without requiring a continuously active Wazuh environment.

### 🎯 Project Objectives

- Automate the ingestion and processing of phishing-related Wazuh alerts.
- Support both webhook-based alert intake and manual JSON submission.
- Filter unrelated security alerts before performing threat intelligence lookups.
- Extract sender addresses, email subjects, and embedded URLs.
- Integrate VirusTotal for automated URL reputation analysis.
- Interpret malicious detection counts to determine whether escalation is necessary.
- Assign incident identifiers and severity classifications.
- Automatically create Trello investigation tickets for suspicious URLs meeting the escalation threshold.
- Generate an HTML confirmation dashboard with incident details.
- Demonstrate how SOAR automation can reduce repetitive SOC triage tasks.

---

## 🛠️ Technologies & Tools

| Technology | Purpose |
|------------|---------|
| n8n | Security workflow orchestration and automation |
| Wazuh | Security monitoring and alert generation |
| VirusTotal | URL reputation analysis and threat intelligence |
| REST API | Communication with external security services |
| JavaScript | JSON parsing, alert validation, and threat classification |
| JSON | Structured alert processing and API response handling |
| Trello | Automated incident tracking and ticket management |
| HTML | Incident confirmation dashboard |
| MITRE ATT&CK | Classification of phishing-related attack techniques |

### MITRE ATT&CK Mapping

**Tactic:** Initial Access

**Technique:** [T1566.001 — Spearphishing Attachment](https://attack.mitre.org/techniques/T1566/001/)

The sample Wazuh alert includes the MITRE ATT&CK identifier `T1566.001`.

This technique specifically describes phishing involving malicious attachments.

Because this project focuses on extracting and analyzing embedded URLs, [T1566.002 — Spearphishing Link](https://attack.mitre.org/techniques/T1566/002/) is also relevant to the workflow's analysis objectives.

The original alert mapping is preserved, while the distinction between attachment-based and link-based phishing is documented for technical accuracy.

---

## ✨ Key Features

### 1. Dual Alert Intake Methods

The workflow supports two methods of receiving security alerts.

**Production Webhook**

Receives Wazuh security alerts through HTTP POST requests.

**Alert Simulator Form**

Accepts manually submitted Wazuh-format JSON alerts for controlled testing.

Both entry paths feed into the same phishing analysis pipeline.

### 2. Automated Alert Validation

The workflow filters incoming alerts using Wazuh rule groups and rule identifiers.

Only alerts meeting the configured phishing-related criteria proceed to URL analysis.

This prevents unrelated events, such as SSH authentication alerts, from triggering unnecessary VirusTotal requests.

### 3. Email Artifact Extraction

The parser extracts relevant email information, including:

- Sender email address
- Email subject
- Embedded URLs

These fields provide the context needed for threat intelligence enrichment and incident documentation.

### 4. VirusTotal URL Analysis

The workflow submits an extracted URL to VirusTotal for analysis.

After a configured wait period, it retrieves the analysis report and evaluates the returned detection statistics.

### 5. Automated Threat Classification

The workflow uses a configurable malicious-detection threshold to determine whether an alert should be escalated.

| Malicious Detections | Escalation Decision |
|---------------------|---------------------|
| 0–2 | No ticket |
| 3 or more | Create incident ticket |

The threshold is a custom workflow rule rather than an official VirusTotal maliciousness classification.

### 6. Severity Assignment

Alerts are assigned severity labels based on the number of malicious detections.

| Malicious Detections | Severity |
|---------------------|----------|
| 0–2 | LOW |
| 3–4 | MEDIUM |
| 5 or more | HIGH |

### 7. Automated Incident Ticketing

Alerts meeting the escalation threshold generate Trello cards containing structured investigation details.

This provides analysts with a centralized record of the suspicious email and associated URL analysis.

### 8. Incident Confirmation Dashboard

The workflow generates an HTML confirmation page displaying incident metadata, threat detection results, and a link to the generated Trello ticket.

---

## 🧩 Workflow Architecture

The automation consists of 12 nodes organized into six stages:

1. Alert ingestion
2. Alert parsing and validation
3. URL threat intelligence enrichment
4. Threat classification
5. Conditional escalation
6. Incident ticketing and confirmation

### Workflow Diagram

```text
 Wazuh Alert Simulator          Wazuh Alert Ingestion
      (Form Trigger)                (Webhook)
            │                           │
            ▼                           │
    Parse & Wrap Alert                │
            │                           │
            └────────────┬──────────────┘
                         │
                         ▼
                Phishing Email Parser
                         │
                         ▼
                 Submit URL to
                   VirusTotal
                         │
                         ▼
                Wait for Threat
                    Analysis
                         │
                         ▼
                Retrieve VirusTotal
                      Report
                         │
                         ▼
                 Threat Verdict
                     Analysis
                         │
                         ▼
                 Generate Incident
                     Metadata
                         │
                         ▼
                 Threat Escalation
                     Decision
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
        MALICIOUS                NOT ESCALATED
        (>2 detections)          (≤2 detections)
              │                     │
              ▼                     ▼
       Create Trello             End Workflow
       Incident Ticket
              │
              ▼
       Incident Confirmation
              Portal (HTML)
```

<img width="1745" height="628" alt="image" src="https://github.com/user-attachments/assets/a0ac0c5c-f7b2-430d-ba80-e90e9c452758" />

---

## ⚙️ Workflow Implementation

### Step 1: Configure Wazuh Alert Ingestion

**Node:** Wazuh Alert Ingestion

**Type:** Webhook Trigger

The production entry point is designed to receive phishing-related Wazuh alerts through HTTP POST requests.

**Webhook Path:**

```text
/wazuh-phishing
```

When a qualifying security event is forwarded to the webhook, the alert's JSON payload enters the phishing analysis pipeline.

For this lab, a sample alert was available for testing without a continuously active Wazuh environment.

#### Example Alert Details

| Field | Value |
|-------|-------|
| Alert Type | Phishing Email |
| Sender | `security-update@paypal-verification-support.com` |
| MITRE ATT&CK ID | T1566.001 |
| Source | Wazuh Alert |
| Transport | HTTP POST |

The sender address resembles a brand-related security notification, illustrating the type of sender information an analyst may need to examine during phishing investigations.

However, the sender address alone is insufficient to establish whether an email is malicious.
### Step 2: Configure the Alert Simulator

**Node:** Wazuh Alert Simulator Form

**Type:** Form Trigger

The simulator provides an alternative method for executing the workflow.

Instead of waiting for a live Wazuh event, a user can paste a JSON alert into a browser form and submit it directly to the automation.

This enables controlled testing of different alert types and filtering conditions.

#### Form Input

```text
Wazuh Alert JSON
```

The submitted JSON is passed to the Parse & Wrap Alert Payload node for normalization.

---

### Step 3: Parse and Normalize the Alert Payload

**Node:** Parse & Wrap Alert Payload

**Type:** Code

The form submission initially contains the alert as a text string.

This node converts the text into a JavaScript object using:

```javascript
JSON.parse()
```

The parsed alert is then wrapped in a structure matching the webhook payload format.

The standardized structure contains:

```json
{
  "headers": {},
  "params": {},
  "query": {},
  "body": {}
}
```

This normalization allows both entry paths to use the same downstream processing logic.

#### Error Handling

The parser also handles malformed JSON submissions.

For example, if a submitted alert contains invalid JSON syntax, the node catches the parsing error and returns a readable error message.

This prevents malformed form input from being silently treated as valid security data.


### Step 4: Validate and Extract Phishing Indicators

**Node:** Phishing Email Parser

**Type:** Code

This node acts as the workflow's initial security-event filter.

Before extracting email information, it validates whether the incoming Wazuh alert meets the configured phishing-related criteria.

#### Validation Check 1: Rule Groups

The alert must contain at least one of the following rule groups:

```text
postfix
spam
phishing
email
mail
```

These groups identify alerts associated with email processing or phishing-related activity.

#### Validation Check 2: Rule ID

The alert's rule ID must satisfy one of the following conditions:

| Rule ID Range | Purpose |
|---------------|---------|
| 3300–3399 | Configured Postfix-related rule range |
| 100000 or greater | Custom phishing rule range |

These are the validation ranges configured for this project, not a universal definition of all Wazuh phishing alerts.

#### Extracted Fields

When both checks pass, the node extracts:

| Field | Description |
|-------|-------------|
| Sender | Email address associated with the message |
| Subject | Email subject line |
| URLs | Links extracted from the email body |

#### Rejected Alerts

If an alert fails validation, the parser returns a skipped status.

Example:

```json
{
  "skipped": true,
  "reason": "Alert does not match phishing rule criteria"
}
```

The intended behavior is to prevent skipped alerts from proceeding to VirusTotal analysis.

**Implementation Note:** This requires the downstream workflow to explicitly filter skipped items or for the parser to return no items. Returning a `skipped` object alone does not automatically stop subsequent n8n nodes.


### Step 5: Submit the URL to VirusTotal

**Node:** Submit URL to VirusTotal

**Type:** HTTP Request

Once the alert passes validation, the workflow submits the first extracted URL to VirusTotal.

**API Endpoint:**

```http
POST https://www.virustotal.com/api/v3/urls
```

VirusTotal evaluates submitted URLs using its available security analysis engines and intelligence sources.

#### Submission Process

1. Extract the first URL from the parsed email.
2. Submit the URL using an authenticated HTTP POST request.
3. Receive a response containing an analysis identifier and reference.
4. Pass the analysis reference to the next stage.

The response includes a reference such as:

```text
data.links.self
```

This identifies the analysis resource used to retrieve the report.

#### Authentication

The lab uses a preconfigured VirusTotal credential.

For a separate n8n deployment, a valid VirusTotal API credential must be configured.

API keys should be stored securely in n8n credentials rather than embedded directly in workflow code.

### Step 6: Wait for Threat Analysis

**Node:** Wait for Threat Analysis

**Type:** Wait

VirusTotal URL analysis is asynchronous.

After a URL is submitted, VirusTotal may require additional time to complete its analysis.

The workflow includes a Wait node configured to pause execution.

#### Wait Configuration

```text
Wait Duration: 20 seconds
```

This introduces a delay between URL submission and report retrieval.

#### Why the Wait Node Matters

Attempting to retrieve analysis results immediately after submission may return an incomplete or queued analysis.

The Wait node provides additional processing time before the next request.

**Limitation:** A fixed 20-second delay does not guarantee completion. A more robust workflow would check the returned analysis status and retry when necessary.


### Step 7: Retrieve the VirusTotal Analysis Report

**Node:** Retrieve VirusTotal Report

**Type:** HTTP Request

After the wait period, the workflow retrieves the VirusTotal analysis using the reference returned during submission.

The report contains detection statistics that summarize the analysis results.

#### Relevant Detection Categories

| Field | Description |
|-------|-------------|
| `malicious` | Number of engines classifying the URL as malicious |
| `suspicious` | Number of engines classifying the URL as suspicious |
| `harmless` | Number of engines classifying the URL as harmless |
| `undetected` | Number of engines without a detection |

These statistics provide the information used by the workflow's threat classification logic.

#### Example Report Structure

```json
{
  "data": {
    "attributes": {
      "status": "completed",
      "stats": {
        "malicious": 5,
        "suspicious": 2,
        "harmless": 60,
        "undetected": 8
      }
    }
  }
}
```

*Note: This example is illustrative and does not represent a recorded result from the lab.*


### Step 8: Evaluate the Threat Verdict

**Node:** Threat Verdict Analysis

**Type:** Code

The workflow evaluates the number of malicious detections returned by VirusTotal.

#### Escalation Logic

```javascript
const maliciousCount = Number(
  $json.data.attributes.stats.malicious ?? 0
);

const isMalicious = maliciousCount > 2;
```

The logic applies the following decision rule:

| Malicious Detections | isMalicious |
|---------------------|-------------|
| 0 | false |
| 1 | false |
| 2 | false |
| 3 or more | true |

This threshold is intended to reduce unnecessary ticket creation from isolated detections.

However, the number of detections is not a definitive measure of maliciousness. Engines may share intelligence, and a low detection count does not guarantee a URL is safe.

The verdict should therefore be treated as an automated triage decision rather than a conclusive determination.


### Step 9: Generate Incident Metadata

**Node:** Generate Incident Metadata

**Type:** Set

Before escalation, the workflow generates an incident identifier and severity classification.

#### Incident Identifier

The incident ID follows the format:

```text
INC-HX-<unix-timestamp-ms>
```

Example:

```text
INC-HX-1716638412000
```

The timestamp-based identifier provides a convenient way to distinguish incidents in the lab.

However, timestamps alone do not guarantee uniqueness if multiple incidents are processed within the same millisecond.

#### Severity Classification

| Malicious Detections | Severity |
|---------------------|----------|
| 0–2 | LOW |
| 3–4 | MEDIUM |
| 5 or more | HIGH |

The severity label is based on the number of malicious detections.

This provides a simple method for prioritizing incidents within the workflow.


### Step 10: Route Incidents by Threat Verdict

**Node:** Threat Escalation Decision

**Type:** IF

The workflow evaluates the `isMalicious` field.

#### Routing Condition

```javascript
isMalicious === true
```

#### True Branch

When more than two malicious detections are returned:

1. The incident is escalated.
2. A Trello investigation ticket is created.
3. An HTML confirmation page is generated.

#### False Branch

When two or fewer malicious detections are returned:

1. No Trello ticket is created.
2. The workflow ends without escalation.

This conditional routing reduces unnecessary ticket creation for alerts that do not meet the configured detection threshold.


### Step 11: Create an Incident Ticket in Trello

**Node:** Create ITSM Incident Ticket

**Type:** Trello

When the escalation condition is satisfied, the workflow creates a Trello card in the configured incident response queue.

Trello serves as a lightweight incident tracking tool for this lab rather than a dedicated enterprise ITSM platform.

#### Ticket Title Format

```text
INC-HX-XXXX | [PHISHING] <email subject>
```

#### Ticket Description

The generated ticket contains:

- Incident ID
- Sender email address
- Email subject
- Suspicious URL
- VirusTotal detection counts
- Assigned severity
- Response queue

#### Assigned Queue

```text
Email Security Response
```

This creates a structured investigation record that analysts can review and update.

The Trello API returns a card URL, which is passed to the confirmation portal.


### Step 12: Generate the Incident Confirmation Portal

**Node:** Incident Confirmation Portal

**Type:** HTML

The final node generates a formatted incident confirmation page.

The page displays:

| Field | Description |
|-------|-------------|
| Incident ID | Generated incident identifier |
| Severity | Calculated threat severity |
| Malicious Detections | VirusTotal malicious detection count |
| Suspicious Detections | VirusTotal suspicious detection count |
| Trello Ticket | Link to the generated investigation card |

This provides a visual summary of the escalation outcome.

**Implementation Note:** Rendering HTML in an n8n execution output is not necessarily the same as returning a page to the original form submitter. A browser-facing response may require an appropriate form completion configuration or webhook response node.


---

## 🧪 Testing & Validation

The workflow was evaluated using three scenarios designed to test alert processing, phishing detection, and filtering behavior.

### Test 1: Wazuh Webhook Alert

#### Objective

Validate that the workflow can process a Wazuh phishing alert through the webhook entry point.

#### Test Procedure

1. Execute the Wazuh Alert Ingestion test using the prepared alert.
2. Observe the alert parsing stage.
3. Inspect the VirusTotal URL submission.
4. Confirm the configured wait period.
5. Review the retrieved VirusTotal report.
6. Examine the threat verdict.
7. Verify whether the escalation condition is satisfied.
8. Inspect the resulting Trello ticket, if created.

#### Expected Behavior

A qualifying phishing alert with more than two malicious URL detections should proceed through the True branch and generate an incident ticket.

#### Results to Document

| Metric | Observed Result |
|--------|-----------------|
| Alert Accepted | `Yes` |
| URL Extracted | `http://testsafebrowsing.appspot.com/s/phishing.html` |
| VirusTotal Malicious Detections | `10` |
| Assigned Severity | `High` |
| Escalation Branch | `True` |
| Trello Ticket Created | `Yes` |

<img width="462" height="197" alt="image" src="https://github.com/user-attachments/assets/144f4905-f481-4246-81aa-ad81d77f1771" />


### Test 2: Malicious Phishing Alert via Form

#### Objective

Validate that a manually submitted phishing alert can successfully pass through the complete enrichment and escalation pipeline.

#### Test Procedure

1. Open the Wazuh Alert Simulator Form.
2. Paste the provided phishing alert JSON.
3. Submit the form.
4. Observe the alert parsing and URL extraction stages.
5. Confirm the VirusTotal submission.
6. Wait for the configured 20-second pause.
7. Inspect the VirusTotal report.
8. Review the threat verdict.
9. Confirm which escalation branch executes.
10. Inspect the Trello incident ticket.

#### Observed Behavior

During the lab, the Threat Escalation Decision node executed its **True branch**.

This indicates that the workflow's malicious-detection threshold was satisfied.

The exact VirusTotal detection count and assigned severity were not recorded in the provided notes.

#### Findings

The test demonstrated that the form-based alert entry path could feed into the phishing analysis pipeline and reach the escalation decision.

The True branch activation confirmed that the workflow identified the submitted URL as meeting the configured escalation criteria.

#### Results to Document

| Metric | Observed Result |
|--------|-----------------|
| Alert Source | Form Simulator |
| URL Submitted to VirusTotal | `http://testsafebrowsing.appspot.com/s/phishing.html` |
| Malicious Detections | `9` |
| Suspicious Detections | `3` |
| Severity | `High` |
| Escalation Branch | `True` |
| Trello Ticket Created | `Yes` |
| Confirmation Portal Rendered | `Yes` |

<img width="456" height="200" alt="image" src="https://github.com/user-attachments/assets/b345e488-a445-49ee-bfb4-398fa167ca1c" />


### Test 3: Non-Phishing Alert Filtering

#### Objective

Validate that the workflow rejects unrelated security events before submitting URLs to VirusTotal.

#### Test Input

The submitted alert contained a Wazuh rule ID of:

```text
5501
```

The alert did not contain any of the configured mail-related rule groups.

#### Expected Behavior

The alert should fail the phishing parser's validation checks.

As a result:

- No VirusTotal request should be made.
- No threat verdict should be calculated.
- No incident ticket should be created.

#### Observed Behavior

During execution, the workflow stopped at the Phishing Email Parser.

The alert failed the configured rule-group and rule-ID validation checks.

No VirusTotal lookup or Trello ticket creation was triggered.

#### Findings

This test demonstrated that the workflow can distinguish phishing-related alerts from unrelated Wazuh security events.

The filtering logic helps prevent unnecessary external API calls and reduces the likelihood of unrelated events generating phishing investigation tickets.

#### Test Outcome

**PASS — Non-Phishing Alert Rejected**

The workflow correctly prevented an unrelated security alert from proceeding through the phishing enrichment and escalation stages.

<img width="956" height="269" alt="image" src="https://github.com/user-attachments/assets/c2a56fa8-2bcf-4549-8ba6-b2b161646564" />


---

## 🔍 Findings & Security Analysis

### 1. Automated Alert Filtering Reduces Unnecessary Processing

The non-phishing test demonstrated that the workflow can reject security events that do not meet its configured validation criteria.

This prevents unrelated alerts from consuming threat intelligence API requests or generating unnecessary incident tickets.

### 2. Threat Intelligence Provides Additional Context

VirusTotal enrichment allows the workflow to evaluate suspicious URLs using external detection results.

This provides information beyond the original Wazuh alert and supports more informed triage decisions.

### 3. Detection Thresholds Influence Escalation

The workflow requires at least three malicious detections before generating an incident ticket.

This reduces escalation based on isolated detections, although the threshold should be evaluated against real-world false-positive and false-negative rates before production use.

### 4. Conditional Ticketing Supports Incident Prioritization

Only alerts meeting the configured malicious-detection threshold proceed to Trello ticket creation.

This helps keep the incident queue focused on events requiring additional investigation.

### 5. Automated Documentation Improves Consistency

The Trello integration standardizes the information recorded for escalated phishing alerts.

By automatically including relevant email and threat intelligence fields, the workflow reduces repetitive manual documentation.

### 6. Asynchronous API Handling Requires Validation

The 20-second Wait node introduces a delay before report retrieval.

However, the workflow would benefit from explicit analysis-status checking and retry logic to ensure that incomplete reports are not mistakenly interpreted as clean.

---

## 🧠 Skills Demonstrated

| Skill | Application |
|-------|-------------|
| SOAR Automation | Developed an automated phishing alert triage workflow |
| Wazuh Alert Processing | Parsed and validated structured security alerts |
| Threat Intelligence | Integrated VirusTotal URL analysis |
| REST API Integration | Configured authenticated HTTP requests |
| JavaScript | Implemented JSON parsing, validation, and threat classification |
| Alert Filtering | Prevented unrelated security events from reaching enrichment stages |
| Asynchronous Processing | Introduced a wait stage between analysis submission and retrieval |
| Incident Prioritization | Applied detection thresholds and severity classifications |
| Workflow Orchestration | Routed alerts through conditional escalation paths |
| Incident Documentation | Automated Trello ticket generation |
| JSON Processing | Normalized and extracted structured security event data |
| HTML Reporting | Generated a formatted incident confirmation dashboard |

---

## 💡 What I Learned

### Building Automated Phishing Triage Workflows

This project strengthened my understanding of how phishing alert processing can be automated from initial ingestion through threat intelligence analysis and incident ticketing.

### Normalizing Security Data From Multiple Sources

I learned how to standardize form-based and webhook-based alert inputs so they can be processed by a shared automation pipeline.

### Integrating Asynchronous Threat Intelligence APIs

Working with VirusTotal demonstrated the importance of handling analysis submission and result retrieval as separate operations.

### Implementing Alert Validation

The phishing parser reinforced the importance of filtering unrelated security events before performing enrichment.

### Applying Conditional Threat Classification

I gained experience developing logic that translates external detection statistics into escalation decisions.

### Automating Incident Documentation

The Trello integration demonstrated how security enrichment results can be converted into structured investigation records.

---

## 🚀 Future Improvements

### 1. Integrate With a Live Wazuh Environment

Configure Wazuh to automatically forward phishing-related security alerts to the n8n webhook.

This would extend the lab from simulated alert processing to continuous security monitoring.

### 2. Implement VirusTotal Status Polling

Replace the fixed 20-second wait with a polling mechanism that checks whether the analysis status is `completed`.

Add retry limits and timeout handling for queued or delayed analyses.

### 3. Support Multiple URLs Per Email

Expand the workflow to extract and analyze every relevant URL in a message rather than only the first URL.

This would improve coverage when phishing emails contain multiple links.

### 4. Improve Threat Scoring

Expand the threat assessment to incorporate:

- Suspicious detection counts
- URL reputation history
- Domain age and registration context
- Sender authentication results
- Email header anomalies
- Known phishing indicators

### 5. Add Duplicate Incident Detection

Implement deduplication logic to prevent repeated alerts involving the same email or URL from generating unnecessary Trello tickets.

### 6. Improve Error Handling

Add dedicated handling for:

- Invalid JSON submissions
- Missing email fields
- Missing or malformed URLs
- VirusTotal API failures
- API rate limits
- Incomplete analysis reports
- Trello ticket creation failures

### 7. Expand Incident Lifecycle Management

Extend the ticketing integration to support investigation ownership, status tracking, analyst notes, and resolution documentation.

### 8. Strengthen Security Controls

Introduce authenticated webhook intake, input validation, secure credential management, and restricted access to incident data.

---

## 🔐 Security Considerations

This workflow processes potentially malicious URLs and email-related security data.

Important security considerations include:

- Store VirusTotal and Trello API credentials securely in n8n.
- Authenticate incoming webhook requests.
- Validate and sanitize submitted JSON.
- Avoid automatically opening or visiting suspicious URLs.
- Consider data sensitivity before submitting internal or private URLs to external intelligence services.
- Treat VirusTotal detections as supporting evidence rather than definitive proof.
- Sanitize untrusted email subjects, sender addresses, and URLs before rendering them in HTML.
- Restrict access to incident tickets and workflow execution data.
- Use dedicated error handling for incomplete or unavailable threat intelligence results.

---

## 📚 References

- [n8n Documentation](https://docs.n8n.io/)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [VirusTotal API Documentation](https://docs.virustotal.com/reference/overview)
- [Trello Developer Documentation](https://developer.atlassian.com/cloud/trello/)
- [MITRE ATT&CK — Phishing (T1566)](https://attack.mitre.org/techniques/T1566/)

---

## 🏁 Conclusion

This project demonstrates how SOAR automation can streamline phishing alert triage by combining Wazuh security monitoring, VirusTotal threat intelligence, and automated incident ticketing.

By developing a workflow that validates incoming alerts, extracts email indicators, analyzes suspicious URLs, applies threat detection thresholds, and conditionally creates investigation tickets, I gained practical experience with security automation and incident response workflows.

The project strengthened my understanding of API integration, JavaScript-based alert processing, asynchronous analysis, threat classification, and automated incident documentation.

It also established a foundation for more advanced phishing response workflows involving multi-indicator analysis, live SIEM integration, automated correlation, and expanded incident management.
