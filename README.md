# CS Command Center

A real-time customer support dashboard built on Google Apps Script.

## The problem

Our CS tech stack is composed of a curated selection of purpose-built tools — Zendesk for tickets, Aircall for phones, Nicereply for CSAT. From a management perspective this creates a fragmented view of operations. There's no single place to assess queue health, SLA compliance, agent workload, and customer satisfaction in real time. No way to deploy resources efficiently or follow up on key events promptly without checking three different apps.

## The solution

A single Google Apps Script file that pulls from all three APIs and renders a consolidated real-time command center inside Google Sheets. Refreshes every 5 minutes. Zero additional cost, no new tools to learn, no vendor to manage.

### What it tracks

**Email (Zendesk)**
- Open ticket count, on-hold count, unassigned count
- SLA compliance — tickets waiting longer than 12 business hours without a first reply
- Per-agent breakdown: assigned, past SLA, longest wait, solved today
- New SAS answering service tickets (belt-and-suspenders detection via tag, subject, and requester)
- Oldest waiting tickets table with clickable links to Zendesk
- Flagged tickets (high priority, warranty, escalated, VIP)

**Phone (Aircall)**
- Answer rate (business hours only, Mon–Fri 6a–5p PST)
- Per-agent inbound/outbound call volume
- Missed calls forwarded to answering service with reason detection
- SMS activity per agent with recent message preview
- Post-call CSAT survey results

**Customer satisfaction (Nicereply)**
- Email CSAT score and percentage (last 24 hours)
- Individual survey responses with score, customer name, ticket link, and timestamp

**Dashboard health indicators**
- Email status: Healthy (0 past SLA) / Watch (1–5) / At Risk (6+)
- Phone status: Healthy (≥75% answer rate) / Watch (60–74%) / At Risk (<60%)

## Tech stack

- **Runtime**: Google Apps Script (V8)
- **UI**: Google Sheets (formatted programmatically — no add-ons)
- **APIs**: Zendesk REST API, Aircall REST API, Nicereply REST API
- **Auth**: Basic HTTP auth for all three, stored in Apps Script Script Properties
- **Scheduling**: Apps Script time-driven trigger (5-minute interval)
- **Dependencies**: None — single file, no build step, no external libraries

## Setup

1. Open a new Google Sheet
2. Go to **Extensions → Apps Script**
3. Paste `cs_command_center_apps_script_v2.js` into `Code.gs` (replace everything)
4. Go to **Project Settings** (gear icon) → **Script Properties** and add:

| Property | Value |
|----------|-------|
| `ZENDESK_TOKEN` | `your_email/token:your_api_token` |
| `AIRCALL_API_ID` | `your_aircall_api_id` |
| `AIRCALL_API_TOKEN` | `your_aircall_api_token` |
| `NICEREPLY_TOKEN` | `your_email:your_nicereply_api_key` |

5. Select `initializeSheet` from the function dropdown, click **Run**
6. Select `setupTrigger` from the function dropdown, click **Run**

## Configuration

All configuration is in the `CONFIG` object at the top of the script:

- `agents` — CS team member names (drives per-agent breakdowns)
- `excludeAgents` — people who use Zendesk but aren't on the CS team (filtered from all stats)
- `excludeSMSLines` — Aircall lines to exclude from SMS tracking (sales reps, etc.)
- `supportNumbers` — Aircall phone numbers for CS support lines
- `answeringServiceNumber` — external answering service number (for SAS forwarding detection)
- `businessHours` — timezone, start/end hour, work days (for answer rate and SLA calculations)
- `thresholds` — green/yellow/red cutoffs for all KPIs

## Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Zendesk   │    │   Aircall   │    │  Nicereply  │
│  REST API   │    │  REST API   │    │  REST API   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────┬───────┴──────────────────┘
                  │
        ┌─────────▼─────────┐
        │  Google Apps Script │
        │  (Code.gs — single │
        │   file, ~3000 LOC) │
        └─────────┬─────────┘
                  │
        ┌─────────▼─────────┐
        │   Google Sheets    │
        │   Dashboard tab    │
        │   (auto-formatted) │
        └───────────────────┘
```

The script runs on a 5-minute trigger. Each refresh fetches fresh data from all three APIs, processes it, and overwrites the Dashboard sheet. A staging sheet is used during writes to prevent the dashboard from flickering mid-update. Raw data is also written to hidden tabs for debugging.
