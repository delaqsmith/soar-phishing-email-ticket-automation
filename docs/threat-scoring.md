# 🛡️ Threat Scoring & Escalation Logic

## Overview

This document explains how the **Phishing Email to Ticket Using n8n** workflow interprets VirusTotal URL analysis results, assigns a severity label, and decides whether to create a Trello incident ticket.

The current workflow uses a **detection-count threshold**, not a weighted risk score. The term *threat scoring* here refers to the classification rules used for triage; the number of VirusTotal engines reporting a URL as malicious is **not** a probability that the URL is malicious.

## 🔄 Decision Flow

```text
Validated phishing-related Wazuh alert
                |
                v
       Extract first URL
                |
                v
     Submit URL to VirusTotal
                |
                v
         Wait 20 seconds
                |
                v
      Retrieve analysis report
                |
                v
     Read stats.malicious count
                |
                v
       Assign severity label
                |
                v
       malicious count > 2?
          /             \
        Yes              No
         |                |
   Create Trello      No ticket
   incident ticket   Workflow ends
         |
         v
   Render confirmation
```

> **Important:** The 20-second delay does not guarantee the VirusTotal analysis has completed. The report's analysis status should be checked before interpreting detection statistics. An incomplete report must not be classified as clean.

## 📊 Detection Thresholds

The workflow reads the `malicious` count from the VirusTotal analysis statistics.

| Malicious detections | `isMalicious` | Severity | Trello ticket | Routing |
|---:|:---:|---|:---:|---|
| 0–2 | `false` | LOW | No | False branch |
| 3–4 | `true` | MEDIUM | Yes | True branch |
| 5+ | `true` | HIGH | Yes | True branch |

### Escalation rule

```javascript
const maliciousCount = Number($json.data.attributes.stats.malicious ?? 0);
const isMalicious = maliciousCount > 2;
```

### Severity rule

```javascript
const severity = maliciousCount >= 5
  ? 'HIGH'
  : maliciousCount >= 3
    ? 'MEDIUM'
    : 'LOW';
```

These snippets illustrate the described logic. The exact data path depends on the output of the **Retrieve VirusTotal Report** node.

**Boundary behavior:** A count of exactly **2** does *not* trigger escalation; **3** does. A count of exactly **4** is MEDIUM; **5** is HIGH.

## 🔍 VirusTotal Fields Used

| Field | Meaning | Used for escalation? |
|---|---|---|
| `stats.malicious` | Number of engine verdicts classified as malicious | **Yes** |
| `stats.suspicious` | Number of suspicious verdicts | No, informational |
| `stats.harmless` | Number of harmless verdicts | No, informational |
| `stats.undetected` | Number of undetected verdicts | No, informational |
| `status` | Whether the analysis is complete | Should be checked before classification |

A higher malicious count can justify investigation, but engine results are not necessarily independent. A low count does not prove a URL is safe, and a high count should still be assessed in context.

## 🧩 Prerequisite: Alert Validation

Threat classification occurs **only after** the incoming Wazuh alert passes the phishing parser's configured checks:

1. At least one rule group matches `postfix`, `spam`, `phishing`, `email`, or `mail`.
2. The rule ID falls within **3300–3399** or is **100000 or greater**.
3. A usable URL is extracted from the alert.

These are **project-specific validation rules**, not universal Wazuh phishing definitions. The first extracted URL is submitted for analysis; other URLs in the same message are not evaluated by this version of the workflow.

If validation fails, the alert should be excluded from VirusTotal analysis and ticket creation. Returning `{ "skipped": true }` from a Code node is not by itself sufficient to stop downstream execution; the workflow must filter out the item or use an explicit routing gate.

## 🧪 Testing and Expected Outcomes

The table below uses **illustrative detection counts**, not recorded VirusTotal results.

| Scenario | Malicious detections | Expected severity | Expected ticket |
|---|---:|---|---|
| No detections | 0 | LOW | No |
| Just below threshold | 2 | LOW | No |
| At escalation threshold | 3 | MEDIUM | Yes |
| Upper MEDIUM boundary | 4 | MEDIUM | Yes |
| At HIGH threshold | 5 | HIGH | Yes |
| Many detections | 12 | HIGH | Yes |

### Lab observations supplied so far

- **Malicious phishing alert via form:** The **True** branch executed. The exact VirusTotal counts and severity were not recorded in the project notes.
- **Non-phishing alert:** Rule ID **5501** and no qualifying mail-related rule group. The workflow stopped at the parser; no VirusTotal request or Trello ticket was created.
- **Webhook-path test:** The draft describes the intended end-to-end test, but does not provide a complete set of measured results.

Do not replace these gaps with invented detection counts. Record actual node outputs and screenshots when available.

## 🎫 Incident Metadata and Ticket Creation

For escalated alerts, the workflow generates:

- **Incident ID:** `INC-HX-<Unix timestamp in milliseconds>`
- **Severity:** `MEDIUM` for 3–4 detections, `HIGH` for 5 or more
- **Trello title:** `INC-HX-XXXX | [PHISHING] <email subject>`
- **Ticket details:** sender, subject, analyzed URL, VirusTotal detection counts, severity, and the **Email Security Response** queue

Timestamp-based IDs are convenient in a lab, but do not guarantee uniqueness when events occur in the same millisecond. A UUID or another collision-resistant identifier would be more robust in production.

The **Incident Confirmation Portal** follows ticket creation on the escalation path. Its appearance in an n8n execution does not by itself establish that the HTML was returned to the submitting browser; that depends on the configured response mechanism.

## ⚠️ Limitations and False-Positive/False-Negative Risks

- **Single indicator:** Only `stats.malicious` affects escalation; suspicious verdicts, sender authentication, URL context, and historical reputation are not included.
- **Threshold trade-off:** Requiring at least three malicious detections may suppress some false positives, but may also miss newly created or low-prevalence phishing URLs.
- **Incomplete analyses:** A fixed 20-second wait may yield pending or incomplete results. Missing fields must not silently default to a benign verdict.
- **Single-URL coverage:** The current workflow examines the first extracted URL, potentially missing additional malicious links.
- **Unrelated alert filtering:** The parser must explicitly stop or route skipped events.
- **External data sharing:** URLs submitted to VirusTotal may contain private identifiers or sensitive information; review before submission.
- **Untrusted content:** Email subjects, sender fields, and URLs must be safely encoded when displayed in HTML or ticket descriptions.

## 🚀 Recommended Improvements

1. **Poll for completion:** Check analysis status and retry with a bounded backoff before evaluating `stats`.
2. **Handle errors explicitly:** Separate API failures, missing URLs, invalid responses, and rate limits from LOW-severity verdicts.
3. **Test boundaries:** Validate detection counts of **2, 3, 4, and 5** using controlled mock API responses.
4. **Enrich more context:** Consider `suspicious` counts, sender authentication (SPF/DKIM/DMARC), domain reputation, and other email indicators.
5. **Analyze every relevant URL:** Deduplicate extracted URLs and evaluate each, then determine an aggregate incident verdict.
6. **Avoid duplicate tickets:** Use stable alert or message identifiers and check for existing incidents.
7. **Improve incident IDs:** Use UUIDs or another collision-resistant scheme.
8. **Record audit evidence:** Store sanitized API verdicts, timestamps, routing decisions, and ticket identifiers for reproducible testing.

## 📸 Suggested Evidence

Add screenshots to `screenshots/` and reference them here once available:

- `virustotal-results.png` — completed analysis and detection statistics
- `threat-verdict.png` — `malicious` count, `isMalicious`, and severity
- `escalation-true.png` — ticket creation branch
- `escalation-false.png` — no-ticket branch for a valid low-detection phishing alert
- `non-phishing-filter.png` — rejected Wazuh alert before VirusTotal
- `trello-ticket.png` — generated incident details

## 🔗 Related Documentation

- [Main README](../README.md)
- [Testing Results](testing-results.md) *(add when completed)*
- [VirusTotal API documentation](https://docs.virustotal.com/reference/overview)
- [n8n documentation](https://docs.n8n.io/)

---

**Summary:** The workflow escalates validated phishing alerts when **three or more VirusTotal engines classify the analyzed URL as malicious**. Counts of **3–4** produce **MEDIUM** severity, and **5 or more** produce **HIGH** severity. These thresholds are configurable lab triage rules and should be validated with real incident outcomes before production deployment.
