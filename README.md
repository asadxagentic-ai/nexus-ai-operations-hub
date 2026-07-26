## Overview

Nexus — AI Operations Hub is an automated monitoring workflow that continuously checks key business metrics across Sales, Support, and Finance domains. Running every 5 minutes, it fetches real-time data, detects anomalies using configurable thresholds, leverages an AI agent (GPT-5 mini) to investigate root causes and recommend actions, and delivers structured alerts via email while maintaining a historical log in Google Sheets.

## How It Works

1. **Schedule Trigger** — Fires every 5 minutes to start a new monitoring cycle.
2. **Fetch Metrics** — Three parallel HTTP Request nodes pull current metrics from:
   - Sales API (`/api/metrics/sales`)
   - Support API (`/api/metrics/support`)
   - Payments API (`/api/metrics/payments`)
3. **Detect Anomalies** — A Code node evaluates the fetched data against predefined thresholds:
   - Sales conversion rate < 2% → high severity
   - Sales hourly revenue drop > 15% → critical severity
   - Support ticket backlog > 200 → medium severity
   - Payment failure rate > 5% → high severity
4. **Branch Decision** — An IF node checks if any anomalies were detected (`anomalyCount > 0`).
5. **If Anomalies Found**:
   - **AI Investigator** (LangChain Agent + GPT-5 mini) analyzes the anomaly snapshot and anomalies list, producing a Markdown report with root causes, 2-4 actionable next steps, owning team, and executive impact rating.
   - **Build Alert Payload** formats the AI output with metadata (title, severity, timestamp, anomalies array).
   - **Email Ops Alert** sends an HTML email to the configured recipient.
   - **Log Alert to Sheet** appends a row to Google Sheets for historical tracking.
6. **If No Anomalies** — **Log Healthy Tick** records a "healthy" status with timestamp to the same sheet.

## Nodes & Tools Used

| Node | Type | Purpose |
|------|------|---------|
| Every 5 min | `scheduleTrigger` | Cron-like scheduler (5-minute interval) |
| Fetch Sales Metrics | `httpRequest` | GET sales metrics from API |
| Fetch Support Queue | `httpRequest` | GET support queue metrics from API |
| Fetch Payments | `httpRequest` | GET payment metrics from API |
| Detect Anomalies | `code` | JavaScript threshold evaluation & anomaly aggregation |
| Has Anomalies? | `if` | Branch on `anomalyCount > 0` |
| AI Investigator | `@n8n/n8n-nodes-langchain.agent` | LangChain agent with system prompt for ops analysis |
| GPT-5 mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | OpenAI chat model (temperature 0.2) |
| Build Alert Payload | `set` | Construct alert object for email & sheet |
| Log Healthy Tick | `set` | Construct healthy-status log entry |
| Email Ops Alert | `gmail` | Send HTML alert email via Gmail OAuth2 |
| Log Alert to Sheet | `googleSheets` | Append row to Google Sheets (OAuth2) |
| Sticky Note | `stickyNote` | In-canvas documentation & setup checklist |

## Prerequisites

- **n8n** (self-hosted or cloud) with access to:
  - `n8n-nodes-base` (core nodes)
  - `@n8n/n8n-nodes-langchain` (LangChain nodes)
- **OpenAI API key** with access to `gpt-5-mini` (or substitute model)
- **Gmail OAuth2 credentials** for sending alert emails
- **Google Sheets OAuth2 credentials** with access to the target spreadsheet
- **Three metrics API endpoints** returning JSON with the expected fields:
  - Sales: `conversionRate` (number), `revenueDropPct` (number)
  - Support: `backlog` (number)
  - Finance: `failedPaymentsPct` (number)

## Setup & Usage

1. **Import the workflow** into n8n (Workflows → Import).
2. **Open the workflow** and update the three HTTP Request nodes:
   - Replace `https://example.com/api/metrics/sales` with your actual Sales metrics endpoint
   - Replace `https://example.com/api/metrics/support` with your actual Support metrics endpoint
   - Replace `https://example.com/api/metrics/payments` with your actual Payments metrics endpoint
3. **Configure credentials**:
   - Click **GPT-5 mini** → select/add your OpenAI credential
   - Click **Email Ops Alert** → select/add your Gmail OAuth2 credential
   - Click **Log Alert to Sheet** → select/add your Google Sheets OAuth2 credential, then choose the target spreadsheet and sheet (the workflow expects columns matching the alert payload)
4. **Adjust anomaly thresholds** (optional) in the **Detect Anomalies** Code node to match your business SLAs.
5. **Update the email recipient** in **Email Ops Alert** (`sendTo` field).
6. **Activate the workflow** — it will run automatically every 5 minutes.
7. **Test manually** by clicking **Execute Workflow** to verify end-to-end flow.

## Use Cases

- **SaaS Operations Teams** — Proactive detection of revenue-impacting issues (conversion drops, payment failures) before customers complain.
- **Support Leadership** — Early warning on ticket backlog spikes to trigger staffing adjustments.
- **Finance Controllers** — Real-time visibility into payment failure trends with AI-suggested root causes (e.g., gateway outage, card decline patterns).
- **Engineering / SRE** — Structured, AI-enriched incident context delivered directly to inbox and logged for postmortems.
- **Executive Stakeholders** — Concise, severity-rated summaries with executive impact ratings for quick decision-making.

---

*Built with n8n • Powered by LangChain & OpenAI GPT-5 mini