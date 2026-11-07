# 🧪 Testing Results — Phishing Email to Ticket Using n8n

## 📌 Overview

This document records validation results for an n8n phishing-alert triage workflow. The workflow accepts Wazuh-format email alerts, extracts an embedded URL, submits it to VirusTotal, evaluates the returned detection counts, assigns severity, and conditionally creates a Trello incident ticket. A form-based entry path can also render an incident confirmation portal.

**Testing scope:** Two completed test executions are documented below using the user-reported observations. A third non-phishing filtering scenario is described in the project README but is not included in the quantitative results provided here. These tests validate automation behavior in a lab; they do not establish that a real phishing incident or compromise occurred.

## 🎯 Test Objectives

- Validate phishing alert ingestion through the Wazuh test entry path and form simulator.
- Confirm extraction and submission of the URL to VirusTotal.
- Verify the malicious-detection escalation threshold and severity classification.
- Confirm Trello ticket creation for escalated alerts.
- Confirm HTML incident portal rendering when observed.

## ⚙️ Classification Rules

The workflow uses a custom detection-count rule:

| VirusTotal malicious detections | `isMalicious` | Severity | Expected Trello behavior |
|---:|---|---|---|
| 0–2 | `false` | Low | No ticket |
| 3–4 | `true` | Medium | Create ticket |
| 5+ | `true` | High | Create ticket |

**Important:** These are lab-defined decision thresholds, not an official VirusTotal risk rating. Engine detections are not necessarily independent, and a low detection count does not prove a URL is safe.

---

## 🔴 Test 1 — Wazuh Alert Ingestion and Escalation

### Objective

Verify that the Wazuh alert test path accepts a phishing-related alert, extracts the embedded URL, enriches it using VirusTotal, classifies the threat, and creates a Trello ticket when the configured detection threshold is exceeded.

### Observed Results

| Metric | Observed result |
|---|---|
| Alert accepted | **Yes** |
| URL extracted | `http://testsafebrowsing.appspot.com/s/phishing.html` |
| VirusTotal malicious detections | **10** |
| Assigned severity | **High** |
| Escalation branch | **True** |
| Trello ticket created | **Yes** |

### Analysis

The alert was accepted and the phishing test URL was successfully extracted. VirusTotal returned **10 malicious detections**, exceeding both the ticketing threshold of **more than 2** and the High-severity threshold of **5 or more**. The workflow therefore assigned **High** severity and selected the **True** escalation branch.

The observed creation of a Trello card confirms that the automated incident-ticketing step executed for this test. The number of suspicious detections and confirmation portal status were not supplied for Test 1, so neither is reported as verified.

The URL belongs to a designated safe-browsing test endpoint; it is used here as a controlled test indicator rather than evidence of a live phishing campaign. VirusTotal detections may vary between runs.

### Outcome

**PASS —** Alert acceptance, URL extraction, High-severity classification, True-branch escalation, and Trello ticket creation were observed.


---

## 🔴 Test 2 — Form Simulator, Escalation, and Confirmation Portal

### Objective

Verify that a manually submitted Wazuh-format phishing alert follows the shared enrichment pipeline, meets the escalation threshold, creates a Trello ticket, and renders an HTML confirmation portal.

### Observed Results

| Metric | Observed result |
|---|---|
| Alert source | **Form Simulator** |
| URL submitted to VirusTotal | `http://testsafebrowsing.appspot.com/s/phishing.html` |
| VirusTotal malicious detections | **9** |
| VirusTotal suspicious detections | **3** |
| Assigned severity | **High** |
| Escalation branch | **True** |
| Trello ticket created | **Yes** |
| Confirmation portal rendered | **Yes** |

### Analysis

The form-based alert successfully reached VirusTotal analysis. The response contained **9 malicious** and **3 suspicious** detections. The configured verdict uses the **malicious** count for escalation; the suspicious count provides additional context but does not change the current threshold calculation.

Because 9 malicious detections exceed the ticketing threshold and meet the High-severity threshold, the workflow correctly assigned **High** severity and routed the alert through the **True** branch. Trello ticket creation and confirmation portal rendering were both observed.

This test provides evidence that the form-based input path can reach the incident-ticketing and reporting stages. The presence of a rendered confirmation portal verifies the HTML output in this lab; it does not by itself prove that the page was returned to the original form submitter in a production deployment.

### Outcome

**PASS —** Form submission, VirusTotal enrichment, High-severity classification, True-branch escalation, Trello ticket creation, and confirmation portal rendering were observed.


---

## 📊 Cross-Test Comparison

| Validation metric | Test 1 — Wazuh entry | Test 2 — Form simulator |
|---|---|---|
| URL | Safe Browsing phishing test URL | Same test URL |
| Malicious detections | **10** | **9** |
| Suspicious detections | Not recorded | **3** |
| Severity | **High** | **High** |
| Escalation branch | **True** | **True** |
| Trello ticket created | **Yes** | **Yes** |
| Confirmation portal | Not recorded | **Yes** |
| Observed test outcome | **PASS** | **PASS** |

### Key Findings

1. **Both alert entry paths supported escalation.** The Wazuh test path and form simulator each reached a High-severity outcome.
2. **The classification matched the configured rules.** Both 10 and 9 malicious detections exceed the High-severity threshold of 5.
3. **Ticket creation was consistent.** Both True-branch executions resulted in Trello tickets.
4. **The confirmation portal was verified in Test 2.** The HTML incident output rendered after escalation.
5. **Detection counts varied for the same test URL.** Test 1 returned 10 malicious detections, while Test 2 returned 9. This reinforces the need to record execution timestamps and treat external intelligence results as time-dependent.

## ⚠️ Limitations and Additional Validation Needed

- **False-branch behavior is not validated by these two results.** Test a qualifying email alert with 0–2 malicious detections to confirm no Trello ticket is created.
- **Medium severity is untested.** Use a controlled response with 3–4 malicious detections to validate Medium classification and ticket creation.
- **Non-phishing filtering should be documented separately.** Record the parser rejection result, absence of VirusTotal requests, and absence of Trello tickets for the non-phishing test.
- **VirusTotal completion handling needs testing.** A fixed 20-second wait does not guarantee that an asynchronous analysis is complete; add status checking, retries, and timeout handling.
- **API and integration failures remain untested.** Consider malformed JSON, missing URLs, VirusTotal errors or rate limits, and Trello failures.
- **No runtime measurement was supplied.** Do not claim specific time savings until start-to-finish execution times are captured.
- **Test 1 portal status was not recorded.** Avoid treating it as verified.

### Recommended Follow-Up Test Matrix

| Scenario | Malicious count | Expected severity | Expected escalation |
|---|---:|---|---|
| No detections | 0 | Low | False / no ticket |
| Just below escalation threshold | 2 | Low | False / no ticket |
| First escalated count | 3 | Medium | True / ticket |
| Upper Medium boundary | 4 | Medium | True / ticket |
| First High count | 5 | High | True / ticket |
| Non-phishing rule ID/group | N/A | Not scored | Filtered before VirusTotal |

Use **mocked or controlled VirusTotal responses** for exact threshold tests so results are reproducible.

## 🛡️ Security Interpretation

These tests demonstrate automated processing and escalation of phishing-related test alerts. The malicious detection counts are useful triage signals, but they do not independently prove a live phishing attack or successful compromise. An analyst should consider the email's headers, sender authentication, URL context, delivery details, and any user interaction before concluding an incident investigation.

## 🏁 Conclusion

Both documented test executions **passed their observed validation checks**. The workflow accepted phishing alert input through two paths, used VirusTotal detection results to assign **High** severity, executed the **True** escalation branch, and created Trello tickets in both tests. The HTML confirmation portal also rendered successfully in Test 2.

The results support the project as a practical demonstration of **SOAR-based phishing triage and automated incident ticketing**. Testing additional severity boundaries, rejected alerts, and error conditions would strengthen its reliability claims.
