# Automation & QA Developer Skills Assessment

**Candidate:** Sumaiah  
**Role:** Automation & QA Developer  
**Submission Repository:** Public GitHub Repository  
**Loom Walkthrough:** [Watch Loom Video Presentation](https://www.loom.com/share/6a3017d649364622b3c22ed5bd4e477b)

---

## Overview of Deliverables

| Deliverable | File Name | Description |
|---|---|---|
| **Task 1: QA Report (PDF)** | `Task1_QA_Report_Sumaiah.pdf` | Formatted audit report: 7 cataloged bugs + Root-Cause Analysis |
| **Task 1: QA Report (Word)**| `Task1_QA_Report_Sumaiah.docx`| Editable Word (.docx) version adhering to Section 5 template |
| **Task 1: QA Report (MD)**  | `Task1_QA_Report_Sumaiah.md`  | Native Markdown version for direct GitHub browsing |
| **Task 2: n8n Workflow**    | `Task2_Workflow_Sumaiah.json` | API integration workflow (GitHub Search + Releases + Discord) |
| **Task 2: Screenshots**     | `task2_canvas.png`, `task2_execution.png` | (a) Workflow Canvas, (b) Successful Execution showing output data |
| **Bonus: Uptime Monitor**   | `Bonus_UptimeMonitor_Sumaiah.json` | 5-min availability monitor with latency tracking & retry logic |
| **Bonus: Screenshot**       | `bonus_uptime_canvas.png` | Uptime Monitor workflow canvas diagram |

---

## Task 2: Architecture & Technical Design

### 1. APIs Used & Selection Rationale
* **Primary API — GitHub Search REST API (`GET https://api.github.com/search/repositories`):**  
  *Query:* `stars:>10000 language:typescript&sort=stars&order=desc&per_page=10`  
  *Rationale:* Provides rich, dynamic developer metadata (stargazers, forks, open issues, repository descriptions, topics). Developer-focused and production-grade.
* **Secondary API — GitHub Releases API (`GET https://api.github.com/repos/{owner}/{repo}/releases/latest`):**  
  *Rationale:* Enriches each trending repository with actionable release intelligence: latest version tag, release date, and release notes URL.

### 2. Data Transformation Logic
Implemented via the **`Transform & Filter Top 5`** and **`Merge Enriched Release Data`** JavaScript Code nodes:
1. **Sanitization & Filtering:** Strips archived or disabled repositories (`!repo.archived && !repo.disabled`).
2. **Sorting & Slicing:** Sorts repositories by `stargazers_count` descending and isolates the top 5 results.
3. **Derived Metrics:** Calculates engagement ratios (`forks_stars_ratio`) and freshness (`days_since_push`).
4. **Resilient Merging:** Merges secondary release payload; provides defensive defaults (`"No formal release"`) if 404 is encountered.
5. **Conditional Routing:** An **IF Node** routes repositories based on popularity threshold (`stars >= 50,000` = Tier-1 Mega Project vs. High-Growth Emerging Project).

### 3. Error Handling & Silent Failure Prevention
The workflow strictly implements defensive architecture:
* **Continue On Fail:** Configured on both HTTP Request nodes (`onError: continueRegularOutput`) to ensure network blips do not abort execution abruptly.
* **Payload Verification (IF Node):** Verifies `$json.items` exists. If rate-limited (HTTP 403) or unreachable (HTTP 500), execution branches to the **`Send Fallback Error Alert`** node to notify the on-call channel immediately.
* **Global Error Trigger:** An `errorTrigger` node catches uncaught runtime exceptions across the entire workflow, dispatching execution IDs and stack traces to an incident webhook.

### 4. Credentials & Secret Management
No API tokens or webhook URLs are hard-coded in node payloads. Credentials reference n8n's internal credential vault (`githubApi` Header Auth for GitHub PAT tokens, and `notificationCredentials` for webhook channels).

---

## Bonus Task: Uptime Monitor (`demo.realworld.io`)

A dedicated workflow running every 5 minutes (`Schedule Trigger`):
1. **Latency Measurement:** Tracks request start and end timestamps (`Δt = t_end - t0`).
2. **Multi-Condition SLA Check:** Evaluates whether HTTP status is `200 OK` and response time is `< 2500ms`.
3. **Automated Retry Circuit:** If non-200, waits 3 seconds and re-pings the endpoint before alerting.
4. **False-Positive Suppression:** Dispatches high-priority alerts only on confirmed secondary failure; transient flakes are logged silently to a warning log.

---

## Quickstart: How to Import Workflows into n8n

1. Open your self-hosted or n8n Cloud dashboard.
2. Click **Workflows** > **Import from File...**
3. Select `Task2_Workflow_Sumaiah.json` (and `Bonus_UptimeMonitor_Sumaiah.json`).
4. To test manually: Click **Execute Workflow** or issue a curl POST request to the Webhook Trigger node.
